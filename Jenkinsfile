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
      env:
        - name: GIT_SSL_NO_VERIFY
          value: "true"
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
            choices: ['dev1', 'dev2', 'dev3', 'qa1', 'qa2', 'qa3', 'uat', 'perf', 'prod'],
            description: 'Select environment'
        )
        choice(
            name: 'TARGET_ENV',
            choices: ['qa1', 'dev1', 'dev2', 'dev3', 'qa2', 'qa3', 'uat', 'perf'],
            description: 'Select target environment'
        )
        choice(
            name: 'SERVICE_NAME',
            choices: ['store', 'graphql', 'bloomreach'], 
            description: 'Select the name of the service in which you want to modify resources'
        )
        booleanParam(name: 'IS_RELEASE', defaultValue: true, description: 'Choose true if you desire the "release" or "candidate" service')
    }

    
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
                    if ((params.SOURCE_ENV == 'perf' || params.TARGET_ENV == 'perf') && params.BANNER != 'dll') {
                        error "Performance environment can only be used with 'dll' banner! Current banner is '${params.BANNER}'"
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
            parallel {
                stage('JSON patch - Deployment') {
                    when {
                        expression { params.DEPLOYMENT == true }
                    }
                    steps {
                        script {
                            env.DEPLOYMENTS = kubectl.getResource([
                                resources: 'deployments', 
                                namespace: "${env.TARGET_NAMESPACE}",
                                options: "--no-headers -o=custom-columns=NAME:.metadata.name"
                            ])

                            // Definește pattern-urile pentru toate tipurile posibile de deployment-uri
                            def deploymentPatterns = []
                            
                            // Pattern-uri de bază
                            deploymentPatterns.add("${env.SERVICE_NAME}-dep-${env.RELEASE_VERSION}")
                            deploymentPatterns.add("${env.SERVICE_NAME}-dashboard-dep-${env.RELEASE_VERSION}")

                            if (params.SERVICE_NAME == 'store') {
                                deploymentPatterns.add("instore-wms-${env.RELEASE_VERSION}")
                            }
                            
                            // Pattern-uri specifice pentru bloomreach (vor fi ignorate pentru alte servicii)
                            if (params.SERVICE_NAME == 'bloomreach') {
                                deploymentPatterns.add("${env.SERVICE_NAME}-authoring-dep-${env.RELEASE_VERSION}")
                                deploymentPatterns.add("${env.SERVICE_NAME}-delivery-dep-${env.RELEASE_VERSION}")
                            }

                            def allDeployments = []
                            deploymentPatterns.each { pattern ->
                                def found = kubectl.filterResourcesByIdentifier([
                                    resources: "${env.DEPLOYMENTS}",
                                    identifier: pattern
                                ])
                                if (found) {
                                    allDeployments.add(found)
                                }
                            }

                            env.FILTERED_DEPLOYMENTS = allDeployments
                                .findAll { it != "" }
                                .join('\n')
                                .trim()
                            
                            if (!env.FILTERED_DEPLOYMENTS) {
                                echo "Warning: No deployments found matching patterns for ${env.SERVICE_NAME}"
                            } else {
                                echo "Found deployments:"
                                env.FILTERED_DEPLOYMENTS.split('\n').each { deployment ->
                                    echo "- ${deployment}"
                                }
                            }
                        }
                    }
                }

                stage('JSON patch - HPA') {
                    when {
                        expression { params.HPA == true }
                    }
                    steps {
                        script {
                            env.HPA = kubectl.getResource([
                                resources: 'hpa', 
                                namespace: "${env.TARGET_NAMESPACE}",
                                options: "--no-headers -o=custom-columns=NAME:.metadata.name"
                            ])

                            def hpaPatterns = []

                            if (params.SERVICE_NAME == 'bloomreach') {
                                hpaPatterns.addAll([
                                    "bloomreach-authoring-scaledobject-${env.RELEASE_VERSION}",
                                    "bloomreach-delivery-scaledobject-${env.RELEASE_VERSION}",
                                    "bloomreach-authoring-hpa-${env.RELEASE_VERSION}",
                                    "bloomreach-delivery-hpa-${env.RELEASE_VERSION}"
                                ])
                            } else {
                                hpaPatterns.addAll([
                                    "${env.SERVICE_NAME}-${env.RELEASE_VERSION}",
                                    "${env.SERVICE_NAME}-hpa-${env.RELEASE_VERSION}",
                                    "${env.SERVICE_NAME}-dashboard-hpa-${env.RELEASE_VERSION}",
                                    "${env.SERVICE_NAME}-dashboard-${env.RELEASE_VERSION}",
                                    "${env.SERVICE_NAME}-scaledobject-${env.RELEASE_VERSION}"
                                ])
                            }

                            def allHPAs = []
                            hpaPatterns.each { pattern ->
                                def found = kubectl.filterResourcesByIdentifier([
                                    resources: "${env.HPA}",
                                    identifier: pattern
                                ])
                                if (found) {
                                    allHPAs.add(found)
                                }
                            }

                            env.FILTERED_HPA = allHPAs
                                .findAll { it != "" }
                                .join('\n')
                                .trim()

                            echo "Found HPAs: ${env.FILTERED_HPA}"
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
                                def response = kubectl.getResource([
                                    namespace: "${env.TARGET_NAMESPACE}",
                                    resources: "deployment/${deployment}",
                                    options: "-o=jsonpath='{.spec.template.spec.containers[0].resources}' | jq '.'"
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
                                def response = kubectl.getResource([
                                    namespace: "${env.TARGET_NAMESPACE}",
                                    resources: "hpa/${hpa}",
                                    options: "-o=jsonpath='{.spec}' | jq '.'"
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
                        expression { params.DEPLOYMENT == true}
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

                                if (env.DEP_PATCH) {
                                    def patchFile = "patch-${deployment}.json"
                                    writeFile file: patchFile, text: env.DEP_PATCH
                                    echo "Written patch to file: ${patchFile}"

                                    kubectl.patchUpdateFile([
                                        namespace: env.TARGET_NAMESPACE,
                                        resourceName: deployment,
                                        resourceType: 'deployment',
                                        patchFile: patchFile
                                    ])
                                } else {
                                    echo "Skipping patch for ${deployment} - no valid resources configuration available"
                                }
                            }
                        }
                    }
                }
            

                stage('Patching the target hpa!') {
                    when {
                        expression { params.HPA == true }
                    }
                    steps {
                        script {
                            echo "Patching HPA"
                            env.FILTERED_HPA.split('\n').each { hpa -> 
                                env.HPA_PATCH = kubectl.getHPAPatchJsonResponse([
                                    valuesFile: env.VALUES_FILE,
                                    resourceName: hpa 
                                ])

                                if (env.HPA_PATCH) {
                                    def patchFile = "patch-${hpa}.json"
                                    writeFile file: patchFile, text: env.HPA_PATCH
                                    echo "Patch file for ${hpa}: ${patchFile}"

                                    kubectl.patchUpdateFile([
                                        namespace: "${env.TARGET_NAMESPACE}",
                                        resourceName: hpa,
                                        resourceType: 'hpa',
                                        patchFile: patchFile
                                    ])
                                } else {
                                    echo "Skipping HPA patch for ${hpa} - no configuration available"
                                }
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
                                def response = kubectl.getResource([
                                    namespace: "${env.TARGET_NAMESPACE}",
                                    resources: "deployment/${deployment}",
                                    options: "-o=jsonpath='{.spec.template.spec.containers[0].resources}' | jq '.'"
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
                                def response = kubectl.getResource([
                                    namespace: "${env.TARGET_NAMESPACE}",
                                    resources: "hpa/${hpa}",
                                    options: "-o=jsonpath='{.spec}' | jq '.'"
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
