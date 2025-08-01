@Library('shared-library-matei-github') _

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
            choices: ['asm-graphql-svc', 'hybris-svc', 'kiosk-svc'],
            description: 'Select the name of the service in which you want to modify resources'
        )
        choice(
            name: 'IS_RELEASE',
            choices: ['true', 'false'],
            description: 'Choose true if you desire the target service to be promoted to "release" from "candidate"'
        )
    }


    environment {
        SOURCE_NAMESPACE = "${BANNER}-${SOURCE_ENV}-space"
        TARGET_NAMESPACE = "${BANNER}-${TARGET_ENV}-space"
    }

    stages{

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

        stage('Get Release Version') {
            steps {
                script {
                    env.RELEASE_VERSION = kubectl.getReleaseVersion([
                        namespace: "${SOURCE_NAMESPACE}",
                        resourceName: "${SERVICE_NAME}",
                        resourceType: 'dr'
                    ])
                    echo "Version of release is ${RELEASE_VERSION}!"
                }
            }
        }

        stage('Generating patch update in JSON form'){
            steps{
                script{
                    env.JSON_RESPONSE = kubectl.getPatchJsonResponse([
                        namespace: "${SOURCE_NAMESPACE}",
                        resourceName: "${SERVICE_NAME}",
                        resourceType: 'deployment',
                        releaseVersion: "${env.RELEASE_VERSION}"
                    ])
                    echo "JSON RESPONSE ${env.JSON_RESPONSE}"
                }
            }
        }

        stage('Get all the deployments and HPA from ${TARGET_NAMESPACE}') {
            parallel {
                stage('Identifying deployments from target namespace ${TARGET_NAMESPACE} associated with ${SERVICE_NAME}-${RELEASE_VERSION}') {
                    steps {
                        script {
                            echo "Debug - Release Version: ${env.RELEASE_VERSION}"
                            echo "Debug - Service Name: ${env.SERVICE_NAME}"
                            
                            env.DEPLOYMENTS=kubectl.getResources([
                                resources: 'deployments', 
                                namespace: "${env.TARGET_NAMESPACE}"
                            ])

                            env.FILTERED_DEPLOYMENTS=kubectl.filterResourcesByVersion([
                                resources: "${env.DEPLOYMENTS}", 
                                version: "${env.RELEASE_VERSION}"
                            ])

                            echo "Filtered deployments: ${env.FILTERED_DEPLOYMENTS}"
                        }
                    }
                }

                stage('Identifying HPA from target namespace ${TARGET_NAMESPACE} associated with ${SERVICE_NAME}-${RELEASE_VERSION}') {
                    steps {
                        script {
                            env.HPA=kubectl.getResources([
                                resources: 'hpa', 
                                namespace: "${env.TARGET_NAMESPACE}"
                            ])

                            env.FILTERED_HPA=kubectl.filterResourcesByVersion([
                                resources: "${env.HPA}", 
                                version: "${env.SERVICE_NAME}"
                            ])

                            echo "Filtered HPA are: ${env.FILTERED_HPA}"
                        }
                    }
                }
            }
        }

        stage('Debug - Check Resources Before') {
            when {
                expression { env.IS_RELEASE == 'true' }
            }
            steps {
                script {
                    echo "=== RESOURCES BEFORE PATCH ==="
                    env.FILTERED_DEPLOYMENTS.split('\n').each { deployment ->
                        def resources = kubectl.checkResources([
                            namespace: "${env.TARGET_NAMESPACE}",
                            resourceName: deployment,
                            resourceType: 'deployment'
                        ])
                        echo "Resources for deployment ${deployment}: ${resources}"
                    }
                }
            }
        }

        stage('Promoting the candidate (adding resources to ${env.TARGET_NAMESPACE})'){
            when {
                        expression { env.IS_RELEASE == 'true' }
            }
            steps{
                script{
                    echo "Allocating more resources to deployments"
                    env.FILTERED_DEPLOYMENTS.split('\n').each { deployment ->
                    
                        writeFile file: "patch-${deployment}.json", text: env.JSON_RESPONSE

                        def resources = kubectl.patchUpdateFileJSON([
                            namespace: "${env.TARGET_NAMESPACE}",
                            resourceName: deployment,
                            resourceType: 'deployment',
                            patchFile: "patch-${deployment}.json"
                        ])
                        echo "Resources for deployment ${deployment}: ${resources}"
                    }

                    echo "Changing HPA associated"
                    env.FILTERED_HPA.split('\n').each { hpa ->
                        def resources = kubectl.getSpecificResource([
                            namespace: "${env.TARGET_NAMESPACE}",
                            resourceName: hpa,
                            resourceType: 'hpa'
                        ])
                        echo "Details about hpa ${hpa}: ${resources}"
                    }
                }
            }
        }

        stage('Debug - Check Resources After') {
            when {
                expression { env.IS_RELEASE == 'true' }
            }
            steps {
                script {
                    echo "=== RESOURCES AFTER PATCH ==="
                    env.FILTERED_DEPLOYMENTS.split('\n').each { deployment ->
                        def resources = kubectl.checkResources([
                            namespace: env.TARGET_NAMESPACE,
                            resourceName: deployment,
                            resourceType: 'deployment'
                        ])
                        echo "Resources for deployment ${deployment}: ${resources}"
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
