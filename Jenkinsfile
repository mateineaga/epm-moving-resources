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
            choices: ['asm-graphql-svc', 'hybris-svc', 'kiosk-svc'],
            description: 'Select the name of the service in which you want to modify resources'
        )
        booleanParam(name: 'IS_RELEASE', defaultValue: true, description: 'Choose true if you desire the target service to be promoted to "release" from "candidate"')
        
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

        stage('Get all the deployments and HPA from target namespace') {
            parallel{
                stage('Identifying deployments ') {
                    when {
                            expression { params.DEPLOYMENT == true }
                    }
                    steps {
                        script {
                            
                            echo "Debug - Release Version: ${env.RELEASE_VERSION}"
                                
                            env.DEPLOYMENTS=kubectl.getResources([
                                resources: 'deployments', 
                                namespace: "${env.TARGET_NAMESPACE}"
                            ])

                            echo "Non filtered deployments are: ${env.DEPLOYMENTS}"

                            env.FILTERED_DEPLOYMENTS=kubectl.filterResourcesByIdentifier([
                                resources: "${env.DEPLOYMENTS}", 
                                identifier: "${env.RELEASE_VERSION}"
                            ])

                            echo "Filtered deployments: ${env.FILTERED_DEPLOYMENTS}"
                        }
                    }
                }

                stage('Identifying HPA ') {
                    when {
                        expression { params.HPA == true }
                    }
                    steps {
                        script {
                            env.HPA=kubectl.getResources([
                                resources: 'hpa', 
                                namespace: "${env.TARGET_NAMESPACE}"
                            ])

                            echo "Non filtered HPA are: ${env.HPA}"

                            env.FILTERED_HPA=kubectl.filterResourcesByIdentifier([
                                resources: "${env.HPA}", 
                                identifier: "${env.SERVICE_NAME.replace("-svc","")}-${env.RELEASE_VERSION}"
                            ])

                            echo "Filtered HPA are: ${env.FILTERED_HPA}"
                        }
                    }
                }
            }
        }
        
        

        stage('Getting JSON files for HPA and deployments'){
            parallel{
                stage('Generating patch update in JSON form, from source deployment!'){
                    when {
                        expression { params.ACTION == 'apply' && params.IS_RELEASE == true && params.DEPLOYMENT == true }
                    }
                    steps{
                        script{
                            env.DEPLOYMENT_JSON_RESPONSE = kubectl.getPatchJsonResponseDeployment([
                                namespace: "${SOURCE_NAMESPACE}",
                                resourceName: "${SERVICE_NAME}".replace("-svc","-dep"),
                                releaseVersion: "${env.RELEASE_VERSION}"
                            ])
                            echo "JSON RESPONSE ${env.DEPLOYMENT_JSON_RESPONSE}"
                        }
                    }
                }

                stage('Generating patch update in JSON form, from source hpa!'){
                    when {
                        expression { params.ACTION == 'apply' && params.IS_RELEASE == true && params.HPA == true}
                    }
                    steps{
                        script{
                            env.HPA_JSON_RESPONSE = kubectl.getHPAPatchJsonResponse([
                                namespace: "${SOURCE_NAMESPACE}",
                                resourceName: env.FILTERED_HPA.trim()
                            ])
                            echo "JSON RESPONSE for source hpa ${env.HPA_JSON_RESPONSE}"
                        }
                    }
                }
            }
        }

        stage('Backup Current State') {
            parallel{
                stage('Backing up for target deployment'){
                    when {
                        expression { params.ACTION == 'apply' && params.IS_RELEASE == true && params.DEPLOYMENT == true}
                    }

                    steps {
                        script {
                            echo "=== BACKING UP CURRENT STATE ==="
                            env.FILTERED_DEPLOYMENTS.split('\n').each { deployment ->
                                def timestamp = new Date().format('yyyyMMdd-HHmmss')
                                def backupFileName = "backup-${deployment}-${timestamp}.json"

                                def jsonResponse = kubectl.getPatchJsonResponseDeployment([
                                    namespace: "${SOURCE_NAMESPACE}",
                                    resourceName: "${SERVICE_NAME}".replace("-svc","-dep"),
                                    resourceType: 'deployment',
                                    releaseVersion: "${env.RELEASE_VERSION}"
                                ])
                                
                                // Salvăm backup-ul ca artifact
                                writeFile file: backupFileName, text: jsonResponse
                                archiveArtifacts artifacts: backupFileName
                                
                                echo "Deployment Backup saved as artifact: ${backupFileName}"

                            }
                        }
                    }
                }
            
                stage('Backing up for target hpa'){
                    when {
                        expression { params.ACTION == 'apply' && params.IS_RELEASE == true && params.HPA == true}
                    }
                    steps{
                        script{
                            def hpa = env.FILTERED_HPA.trim()
                            def timestamp = new Date().format('yyyyMMdd-HHmmss')
                            def hpaBackupFileName = "backup-hpa-${hpa}-${timestamp}.json"

                            def hpaJsonResponse = kubectl.getHPAPatchJsonResponse([
                                namespace: "${TARGET_NAMESPACE}",
                                resourceName: hpa
                            ])

                            writeFile file: hpaBackupFileName, text: hpaJsonResponse
                            archiveArtifacts artifacts: hpaBackupFileName

                            echo "HPA Backup saved as artifact: ${hpaBackupFileName}"
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

                stage('Check HPA Resources') {
                    when {
                        expression { params.HPA == true }
                    }
                    steps {
                        script {
                            echo "=== RESOURCES BEFORE APPLY/PATCH - HPA ==="
                            env.HPA_JSON_RESPONSE = kubectl.checkResources([
                                namespace: "${env.TARGET_NAMESPACE}",
                                resourceName: env.FILTERED_HPA.trim(),
                                resourceType: 'hpa'
                            ])

                            echo "Resources for HPA ${env.FILTERED_HPA.trim()}: ${env.HPA_JSON_RESPONSE}"
                        }
                    }
                }
            }
        }

        stage('Promoting the candidate'){
            parallel{
                stage('Patching the target deployment!'){
                    when {
                        expression { params.ACTION == 'apply' && params.IS_RELEASE == true && params.DEPLOYMENT == true}
                    }
                    steps{
                        script{
                            echo "Allocating more resources to deployments"
                            env.FILTERED_DEPLOYMENTS.split('\n').each { deployment ->
                            
                                writeFile file: "patch-${deployment}.json", text: env.DEPLOYMENT_JSON_RESPONSE

                                def resources = kubectl.patchUpdateFileJSON([
                                    namespace: "${env.TARGET_NAMESPACE}",
                                    resourceName: deployment,
                                    resourceType: 'deployment',
                                    patchFile: "patch-${deployment}.json"
                                ])
                                echo "Resources for deployment ${deployment}: ${resources}"
                            }
                        }
                    }
                }
            

                stage('Patching the target hpa!'){
                    when {
                        expression { params.ACTION == 'apply' && params.IS_RELEASE == true && params.DEPLOYMENT == true}
                    }
                    steps{
                        script{
                            echo "Changing HPA associated"
                            def hpa = env.FILTERED_HPA.trim()
                            writeFile file: "patch-${hpa}.json", text: env.HPA_JSON_RESPONSE

                            def hpaResources = kubectl.patchUpdateFileJSON([
                                namespace: "${env.TARGET_NAMESPACE}",
                                resourceName: hpa,
                                resourceType: 'hpa',
                                patchFile: "patch-${hpa}.json"

                            ])
                            echo "Resources for HPA ${hpa}: ${hpaResources}"
                        }
                    }
                }
            }
            
        }

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
                            echo "=== REVERTING TO PREVIOUS STATE ==="
                        
                            // Pentru fiecare deployment
                            env.FILTERED_DEPLOYMENTS.split('\n').each { deployment ->
                                // Găsim backup-ul corespunzător din artefacte
                                def artifactPattern = "backup-${deployment}-*.json"
                                
                                // Copiem artefactul din build-ul anterior folosind lastSuccessful
                                copyArtifacts(
                                    projectName: env.JOB_NAME,
                                    selector: lastSuccessful(),
                                    filter: artifactPattern
                                )
                                
                                // Verificăm dacă există fișierul de backup
                                def backupFile = sh(
                                    script: "ls -1 ${artifactPattern} | head -1",
                                    returnStdout: true
                                ).trim()
                                
                                if (backupFile) {
                                    echo "Found backup for ${deployment}: ${backupFile}"
                                    
                                    // Citim conținutul backup-ului
                                    def revertPatch = readFile(backupFile)
                                    
                                    // Creăm fișierul de patch pentru revert
                                    def revertFileName = "revert-${deployment}.json"
                                    writeFile file: revertFileName, text: revertPatch
                                    
                                    // Aplicăm patch-ul
                                    kubectl.patchUpdateFileJSON([
                                        namespace: env.TARGET_NAMESPACE,
                                        resourceName: deployment,
                                        resourceType: 'deployment',
                                        patchFile: revertFileName
                                    ])
                                    
                                    echo "Reverted deployment: ${deployment}"
                                } else {
                                    error "No backup files found for deployment: ${deployment}"
                                }
                            }
                        }
                        
                    }
                }

                stage('Reverting the target hpa'){
                    when {
                        expression { params.HPA == true}
                    }
                    steps{
                        script{
                            def hpa = env.FILTERED_HPA.trim()
                            def hpaArtifactPattern = "backup-hpa-${hpa}-*.json"

                            copyArtifacts(
                                projectName: env.JOB_NAME,
                                selector: lastSuccessful(),
                                filter: hpaArtifactPattern
                            )

                            def hpaBackupFile = sh(
                                script: "ls -1 ${hpaArtifactPattern} | head -1",
                                returnStdout: true
                            ).trim()

                            if (hpaBackupFile) {
                                echo "Found backup for HPA ${hpa}: ${hpaBackupFile}"
                                def hpaRevertPatch = readFile(hpaBackupFile)
                                def hpaRevertFileName = "revert-hpa-${hpa}.json"
                                writeFile file: hpaRevertFileName, text: hpaRevertPatch

                                kubectl.patchUpdateFileJSON([
                                    namespace: env.TARGET_NAMESPACE,
                                    resourceName: hpa,
                                    resourceType: 'hpa',
                                    patchFile: hpaRevertFileName
                                ])

                                echo "Reverted HPA: ${hpa}"
                            }
                        }
                    }
                }
            }
        }
            
        

        stage('Debug - Check Resources After') {
            // when {
            //     expression { params.IS_RELEASE == true }
            // }
            parallel{
                stage('Debug for target deployment'){
                    when {
                        expression { params.DEPLOYMENT == true}
                    }
                    steps{
                        script{
                            echo " RESOURCES AFTER PATCH - DEPLOYMENT"
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

                stage('Debug for target hpa'){
                    when {
                        expression { params.HPA == true}
                    }
                    steps{
                        script{
                            echo "RESOURCES AFTER PATCH - HPA"
                            def hpa = env.FILTERED_HPA.trim()
                            def hpaResources = kubectl.checkResources([
                                namespace: env.TARGET_NAMESPACE,
                                resourceName: hpa,
                                resourceType: 'hpa'
                            ])
                            echo "Resources for HPA ${hpa}: ${hpaResources}"
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
