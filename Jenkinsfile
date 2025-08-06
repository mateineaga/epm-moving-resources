@Library('shared-library-matei-github') _
import jenkins.model.Jenkins
import hudson.plugins.copyartifact.BuildSelector

pipeline {
    agent {
        kubernetes {
            yaml '''
            apiVersion: v1
            kind: Pod
            metadata:
              labels:
                jenkins: agent
            spec:
              containers:
                - name: kubectl
                  image: jenkins-slave:latest
                  imagePullPolicy: IfNotPresent
                  command:
                  - cat
                  tty: true
              serviceAccountName: jenkins
            '''
            defaultContainer 'kubectl'
            namespace 'jenkins'
        }
    }


    parameters {
        choice(
            name: 'ACTION',
            choices: ['apply', 'revert'],
            description: 'Choose action: apply changes or revert to previous state'
        )
        booleanParam(name: 'DEPLOYMENT', defaultValue: true, description: 'Select if you want to apply the action on deployment')
        booleanParam(name: 'HPA', defaultValue: true, description: 'Select if you want to apply the action on HPA')
        choice(
            name: 'BANNER',
            choices: ['ab', 'dll', 'mi', 'ms', 'alb'],
            description: 'Select source banner to change resources from'
        )
        choice(
            name: 'SOURCE_ENV',
            choices: ['dev1', 'dev2', 'dev3', 'qa1', 'qa2', 'qa3', 'perf', 'prod'],
            description: 'Select environment'
        )
        choice(
            name: 'TARGET_ENV',
            choices: ['qa1', 'dev1', 'dev2', 'dev3', 'qa2', 'qa3', 'perf'],
            description: 'Select target environment'
        )
        choice(
            name: 'SERVICE_NAME',
            choices: ['graphql', 'store', 'bloomreach-authoring',  'bloomreach-delivery', 'hybris' ], // graphql, store, bloomreach(authoring, delivery)
            description: 'Select the name of the service in which you want to modify resources'
        )
        booleanParam(name: 'IS_RELEASE', defaultValue: true, description: 'Choose true if you desire the "release" or "candidate" service')
    }

    // TODO: check eu-digital-fe-stores (next), eu-digital-graphql, eu-digital-bloomreach-cms - for helm charts to see names of services convention + find additional prefixes for each service (asm, kiosk, ..etc)
    // TODO 2 : checkout repo specific for every service https://github.com/RoyalAholdDelhaize/eu-digital-bloomreach-cms/blob/develop/pipeline/bloomreach-service/values-dev1-ab.yaml to get patch update json
    
    environment {
        SOURCE_NAMESPACE = "${BANNER}-${SOURCE_ENV}-space"
        TARGET_NAMESPACE = "${BANNER}-${TARGET_ENV}-space"
    }

    stages{
        stage('Initialize') {
            steps {
                script {
                    env.SERVICE_REPOS = getServiceRepos()
                }
            }
        }

        stage('Checking parameters'){
            steps{
                echo "Banner is: ${BANNER}"
                echo "Service name is: ${SERVICE_NAME}"
                echo "Is release?: ${IS_RELEASE}"
                echo "Source NS: ${env.SOURCE_NAMESPACE}"
                echo "Target NS: ${env.TARGET_NAMESPACE}"
            }
        }

        stage('Validate Parameters') {
            steps {
                script {
                    if (params.SOURCE_ENV == params.TARGET_ENV) {
                        error "Source and Target environments cannot be the same!"
                    }
                }
            }
        }

        stage('Checkout Code') {
            steps {
                script {
                    def serviceRepos = getServiceRepos()
                    def repoConfig = serviceRepos.get(params.SERVICE_NAME)

                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: repoConfig.branch]],
                        userRemoteConfigs: [[
                            url: repoConfig.url,
                            credentialsId: 'GithubCredentials'
                        ]]
                    ])

                    env.SERVICE_PATH = repoConfig.path
                    env.VALUES_FILE = "values-${params.SOURCE_ENV}-${params.BANNER}.yaml"
                }
            }
        }
        

        stage('Get Release Version') {
            steps {
                script {
                    env.RELEASE_VERSION = kubectl.getReleaseVersion([
                        namespace: "${SOURCE_NAMESPACE}",
                        resourceName: "${SERVICE_NAME}",
                        release: params.IS_RELEASE
                    ])
                    echo "Version of release is ${env.RELEASE_VERSION}!"
                }
            }
        }


        // TODO: De modificat sa preia valori din github, nu de la alte pods
        stage('Preparing patch') {
            parallel{
                stage('JSON patch - Deployment') {
                    when {
                        expression { params.DEPLOYMENT == true }
                    }
                    steps {
                        script {
                            echo "Debug - Release Version: ${env.RELEASE_VERSION}"
                                
                            env.DEPLOYMENTS = kubectl.getResources([
                                resources: 'deployments', 
                                namespace: "${env.TARGET_NAMESPACE}"
                            ])

                            echo "Non filtered deployments are: ${env.DEPLOYMENTS}"

                            env.FILTERED_DEPLOYMENTS = kubectl.filterResourcesByIdentifier([
                                resources: "${env.DEPLOYMENTS}", 
                                identifier: "${env.RELEASE_VERSION}"
                            ])

                            echo "Filtered deployments: ${env.FILTERED_DEPLOYMENTS}"

                            env.DEP_PATCH = kubectl.getPatchJsonResponseDeployment(
                                valuesFile: env.VALUES_FILE
                            )

                            echo "Patch which will be applied to deployments: ${env.DEP_PATCH}"
                            
                        }
                    }
                }

                stage('JSON patch - HPA') {
                    when {
                        expression { params.HPA == true }
                    }
                    steps {
                        script {
                            env.HPA=kubectl.getResources([
                                resources: 'hpa', 
                                namespace: "${env.TARGET_NAMESPACE}"
                            ])

                            echo "Non filtered target HPA are: ${env.HPA}"

                            env.FILTERED_HPA=kubectl.filterResourcesByIdentifier([
                                resources: "${env.HPA}", 
                                identifier: "${env.SERVICE_NAME.replace("-svc","")}-${env.RELEASE_VERSION}"
                            ])

                            echo "Filtered target HPA are: ${env.FILTERED_HPA}"

                            // env.SOURCE_HPA=kubectl.getResources([
                            //     resources: 'hpa', 
                            //     namespace: "${env.SOURCE_NAMESPACE}"
                            // ])

                            // echo "All source HPA to extract specs from are: ${env.SOURCE_HPA}"

                            // env.SOURCE_FILTERED_HPA=kubectl.filterResourcesByIdentifier([
                            //     resources: "${env.SOURCE_HPA}", 
                            //     identifier: "${env.SERVICE_NAME.replace("-svc","")}-${env.RELEASE_VERSION}"
                            // ])

                            // echo "Filtered source HPA to extract specs from are: ${env.SOURCE_FILTERED_HPA}"

                            env.HPA_PATCH = kubectl.getHPAPatchJsonResponse(
                                valuesFile: env.VALUES_FILE
                            )

                            echo "Patch which will be applied to HPA: ${env.HPA_PATCH}"
                        }
                    }
                }
            }
        }
        
        
        // De eliminat backup

        stage('Backup Current State') {
            parallel {
                stage('Backing up for target deployment') {
                    when {
                        expression { params.ACTION == 'apply' && params.DEPLOYMENT == true}
                    }
                    steps {
                        script {
                            env.FLITERED_DEPLOYMENTS.each{deployment ->
                                def backup = kubectl.backupResource([
                                    namespace: env.TARGET_NAMESPACE,
                                    resourceName: deployment,
                                    resourceType: 'deployment',
                                    backupPrefix: 'backup',
                                    releaseVersion: env.RELEASE_VERSION,
                                    serviceName: env.SERVICE_NAME
                                ])
                                
                                writeFile file: backup.fileName, text: backup.content
                                archiveArtifacts artifacts: backup.fileName
                                
                                echo "Deployment backup saved as artifact: ${backup.fileName}"
                            }
                        }
                    }
                }
            
                stage('Backing up for target hpa') {
                    when {
                        expression { params.ACTION == 'apply' && params.IS_RELEASE == true && params.HPA == true}
                    }
                    steps {
                        script {
                                env.FLITERED_HPA.each{hpa ->
                                    def backup = kubectl.backupResource([
                                        namespace: env.TARGET_NAMESPACE,
                                        resourceName: hpa,
                                        resourceType: 'hpa',
                                        backupPrefix: 'backup-hpa'
                                    ])
                                    
                                    writeFile file: backup.fileName, text: backup.content
                                    archiveArtifacts artifacts: backup.fileName
                                    
                                    echo "HPA backup saved as artifact: ${backup.fileName}"
                                }
                            }
                        }
                    }
            }
        }
        

        stage('Debug - Check Resources Before') {
            parallel {
                stage('Check Deployment Resources') {
                    when {
                        expression { params.DEPLOYMENT == true }
                    }
                    steps {
                        script {
                            echo "=== RESOURCES BEFORE APPLY/PATCH - DEPLOYMENTS ==="
                            
                            env.FLITERED_DEPLOYMENTS.each{deployment ->
                                def response = kubectl.checkResourcesDeployment([
                                    namespace: "${env.TARGET_NAMESPACE}",
                                    resourceName: deployment,
                                    resourceType: 'deployment'
                                ])
                                echo "Resources before patch for target deployment ${deployment}: ${response}"
                            }
                            
                        }
                    }
                }

                stage('Check HPA Resources') {
                    when {
                        expression { params.HPA == true }
                    }
                    steps {
                        script {
                            echo "=== RESOURCES BEFORE APPLY/PATCH - HPA ==="

                            env.FLITERED_HPA.each{hpa ->
                                def response = kubectl.checkResourcesDeployment([
                                    namespace: "${env.TARGET_NAMESPACE}",
                                    resourceName: hpa,
                                    resourceType: 'hpa'
                                ])
                                echo "Resources before patch for target hpa ${hpa}: ${response}"
                            }
                        }
                    }
                }
            }
        }


        // stage('Promoting the candidate'){
        //     parallel{
        //         stage('Patching the target deployment!'){
        //             when {
        //                 expression { params.ACTION == 'apply' && params.IS_RELEASE == true && params.DEPLOYMENT == true}
        //             }
        //             steps{
        //                 script{
        //                     echo "Allocating more resources to deployments"

        //                     writeFile file: "patch-${env.FILTERED_DEPLOYMENTS.trim()}.json", text: env.DEPLOYMENT_JSON_RESPONSE

        //                     kubectl.patchUpdateFileJSON([
        //                         namespace: "${env.TARGET_NAMESPACE}",
        //                         resourceName: env.FILTERED_DEPLOYMENTS.trim(),
        //                         resourceType: 'deployment',
        //                         patchFile: "patch-${env.FILTERED_DEPLOYMENTS.trim()}.json"
        //                     ])
        //                 }
        //             }
        //         }
            

        //         stage('Patching the target hpa!'){
        //             when {
        //                 expression { params.ACTION == 'apply' && params.IS_RELEASE == true && params.HPA == true}
        //             }
        //             steps{
        //                 script{
        //                     echo "Changing HPA associated"
        //                     def hpa = env.FILTERED_HPA.trim()
        //                     def patchFile = "patch-${hpa}.json"
                            
        //                     writeFile file: patchFile, text: env.HPA_JSON_RESPONSE

        //                     kubectl.patchUpdateFileJSON([
        //                         namespace: "${env.TARGET_NAMESPACE}",
        //                         resourceName: hpa,
        //                         resourceType: 'hpa',
        //                         patchFile: patchFile

        //                     ])

        //                 }
        //             }
        //         }
        //     }
            
        // }

        stage('Reverting the resources to initial values.'){
            when {
                expression { params.ACTION == 'revert' }
            }
            parallel{
                stage('Reverting the target deployment'){
                    when {
                        expression { params.DEPLOYMENT == true}
                    }
                    steps{
                        script{
                            kubectl.revertResource([
                                namespace: env.TARGET_NAMESPACE,
                                resourceName: env.FILTERED_DEPLOYMENTS.trim(),
                                resourceType: 'deployment',
                                jobName: env.JOB_NAME,
                                backupPrefix: 'backup'
                            ])
                        }
                    }
                }

                stage('Reverting the target hpa'){
                    when {
                        expression { params.HPA == true}
                    }
                    steps{
                        script{
                            kubectl.revertResource([
                                namespace: env.TARGET_NAMESPACE,
                                resourceName: env.FILTERED_HPA.trim(),
                                resourceType: 'hpa',
                                jobName: env.JOB_NAME,
                                backupPrefix: 'backup-hpa'
                            ])
                        }
                    }
                }
            }
        }
            
        

        stage('Debug - Check Resources after') {
            parallel {
                stage('Check Deployment Resources') {
                    when {
                        expression { params.DEPLOYMENT == true }
                    }
                    steps {
                        script {
                            echo "=== RESOURCES AFTER APPLY/PATCH - DEPLOYMENTS ==="
                            
                            env.FLITERED_DEPLOYMENTS.each{deployment ->
                                def response = kubectl.checkResourcesDeployment([
                                    namespace: "${env.TARGET_NAMESPACE}",
                                    resourceName: deployment,
                                    resourceType: 'deployment'
                                ])
                                echo "Resources AFTER patch for target deployment ${deployment}: ${response}"
                            }
                            
                        }
                    }
                }

                stage('Check HPA Resources') {
                    when {
                        expression { params.HPA == true }
                    }
                    steps {
                        script {
                            echo "=== RESOURCES AFTER APPLY/PATCH - HPA ==="

                            env.FLITERED_HPA.each{hpa ->
                                def response = kubectl.checkResourcesDeployment([
                                    namespace: "${env.TARGET_NAMESPACE}",
                                    resourceName: hpa,
                                    resourceType: 'hpa'
                                ])
                                echo "Resources AFTER patch for target hpa ${hpa}: ${response}"
                            }
                        }
                    }
                }
            }
        }
        
    }

    post {
        success {
            echo "Successfully completed pipeline"
        }
        failure {
            echo "Pipeline failed"
        }
        always {
            cleanWs deleteDirs: true
        }
    }
}
