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
                    
                    if (!repoConfig) {
                        error "No repository configuration found for service: ${params.SERVICE_NAME}"
                    }

                    def checkoutResult = checkout([
                        $class: 'GitSCM',
                        branches: [[name: repoConfig.branch]],
                        userRemoteConfigs: [[
                            url: repoConfig.url,
                            credentialsId: 'GithubCredentials'
                        ]]
                    ])


                    // Setează path-urile
                    env.SERVICE_PATH = repoConfig.path
                    env.VALUES_FILE = "${env.SERVICE_PATH}/values-${params.SOURCE_ENV}-${params.BANNER}.yaml"

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


        
        stage('Preparing patch') {
            parallel{
                stage('JSON patch - Deployment') {
                    when {
                        expression { params.DEPLOYMENT == true }
                    }
                    steps {
                        script {
                            echo "Debug - Release Version: ${env.RELEASE_VERSION}"
                                
                            def deployments = kubectl.getResources([
                                resources: 'deployments', 
                                namespace: "${env.TARGET_NAMESPACE}"
                            ])

                            echo "Non filtered deployments are: ${deployments}"

                            env.FILTERED_DEPLOYMENTS = kubectl.filterResourcesByIdentifier([
                                resources: "${deployments}", 
                                identifier: "${env.SERVICE_NAME}-dep-${env.RELEASE_VERSION}"
                            ])

                            echo "Filtered deployments: ${env.FILTERED_DEPLOYMENTS}"
                            
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
                                identifier: "${env.SERVICE_NAME}-${env.RELEASE_VERSION}"
                            ])
                            echo "Filtered target HPA are: ${env.FILTERED_HPA}"

                            env.HPA_PATCH = kubectl.getHPAPatchJsonResponse([
                                    valuesFile: env.VALUES_FILE
                            ])

                            echo "HPA Patch is ${env.HPA_PATCH}"
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
                            
                            env.FILTERED_DEPLOYMENTS.split('\n').each{deployment ->
                                def response = kubectl.checkResourcesDeployment([
                                    namespace: "${env.TARGET_NAMESPACE}",
                                    resourceName: deployment
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

                            env.FILTERED_HPA.split('\n').each{hpa ->
                                def response = kubectl.checkResourcesHPA([
                                    namespace: "${env.TARGET_NAMESPACE}",
                                    resourceName: hpa
                                ])
                                echo "Resources before patch for target hpa ${hpa}: ${response}"
                            }
                        }
                    }
                }
            }
        }


        stage('Patching'){
            parallel{
                stage('Patching the target deployments!'){
                    when {
                        expression { params.ACTION == 'apply' && params.DEPLOYMENT == true}
                    }
                    steps{
                        script{
                            env.FILTERED_DEPLOYMENTS.split('\n').each { deployment -> 

                                def image = sh(
                                    script: """
                                        kubectl get deployment -n ${env.TARGET_NAMESPACE} ${deployment} -o=jsonpath='{.spec.template.spec.containers[0].image}'
                                    """,
                                    returnStdout: true
                                ).trim()

                                echo "Identified image for deployment ${deployment} is ${image}"

                                env.DEP_PATCH = kubectl.getPatchJsonResponseDeployment([
                                    valuesFile: env.VALUES_FILE,
                                    deployment: deployment,
                                    imageContainer: image
                                ])

                                echo "Generated patch for deployment ${deployment} with container name ${deployment.replaceAll('-dep-[0-9.-]+$', '')}"

                                def patchFile = "patch-${deployment}.json"
                                writeFile file: patchFile, text: env.DEP_PATCH
                                echo "Written patch to file: ${patchFile}"

                                kubectl.patchUpdateFileJSON([
                                    namespace: env.TARGET_NAMESPACE,
                                    resourceName: deployment,
                                    resourceType: 'deployment',
                                    patchFile: patchFile
                                ])
                            }
                        }
                    }
                }
            

                stage('Patching the target hpa!'){
                    when {
                        expression { params.ACTION == 'apply' && params.HPA == true}
                    }
                    steps{
                        script{
                            echo "Patching HPA"
                            def patchFile = "patch-hpa.json"
                            
                            writeFile file: patchFile, text: env.HPA_PATCH

                            echo "Patch file: ${patchFile}"

                            env.FILTERED_HPA.split('\n').each{hpa -> 
                                kubectl.patchUpdateFileJSON([
                                namespace: "${env.TARGET_NAMESPACE}",
                                resourceName: hpa,
                                resourceType: 'hpa',
                                patchFile: patchFile
                                ])
                            }
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
                            
                            env.FILTERED_DEPLOYMENTS.split('\n').each{deployment ->
                                def response = kubectl.checkResourcesDeployment([
                                    namespace: "${env.TARGET_NAMESPACE}",
                                    resourceName: deployment
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

                            env.FILTERED_HPA.split('\n').each{hpa ->
                                def response = kubectl.checkResourcesHPA([
                                    namespace: "${env.TARGET_NAMESPACE}",
                                    resourceName: hpa
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
