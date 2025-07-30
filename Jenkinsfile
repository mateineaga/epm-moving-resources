@Library('shared-library-matei-github') _

pipeline {
    agent any 

    parameters {
        choice choices: ['ab', 'dll', 'mi', 'ms', 'alb'], description: 'Select source banner to change resources from', name: 'SOURCE_BANNER'
        choice choices: ['ab', 'dll', 'mi', 'ms', 'alb'], description: 'Select target banner to change resources to', name: 'TARGET_BANNER'
        choice choices: ['dev1', 'dev2', 'dev3', 'qa1', 'qa2', 'qa3'], description: 'Select environment', name: 'ENV'
        choice choices: ['asm-graphql-svc', 'hybris-svc', 'kiosk-svc'], description: 'Select the name of the service in which you want to modify resources', name: 'SERVICE_NAME'
        choice choices: ['true', 'false'], description: 'Choose true if you desire the target service to be promoted to "release" from "candidate"', name: 'IS_RELEASE'
    }

    environment {
        SOURCE_NAMESPACE = "${SOURCE_BANNER}-${ENV}-space"
        TARGET_NAMESPACE = "${TARGET_BANNER}-${ENV}-space"
    }

    stages{
        stage ('Checkout source') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/mateineaga/epm-moving-resources.git',
                    credentialsId: 'GithubCredentials'
                // helloWorld()
            }
        }

        stage('Checking parameters'){
            steps{
                echo "Source banner is: ${SOURCE_BANNER}"
                echo "Target banner is: ${TARGET_BANNER}"
                echo "Service banner is: ${SERVICE_NAME}"
                echo "Is release?: ${IS_RELEASE}"
                echo "Source NS: ${SOURCE_NAMESPACE}"
                echo "Target NS: ${TARGET_NAMESPACE}"
            }
        }

        stage('Getting k8s environment'){
            steps{
                script{
                    sh 'kubectl get pods'
                    sh 'kubectl get services'
                    sh 'kubectl get deployments'
                }
            }
        }
    }
    

}