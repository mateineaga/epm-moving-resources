@Library('shared-library-matei-github')

pipeline {
    agent any 

    parameters {
        choice choices: ['ab', 'dll', 'mi', 'ms', 'alb'], description: 'Select source banner to change resources from', name: 'SOURCE_BANNER'
        choice choices: ['ab', 'dll', 'mi', 'ms', 'alb'], description: 'Select target banner to change resources to', name: 'TARGET_BANNER'
        choice choices: ['test_service_1', 'test_service_2', 'test_service_3'], description: 'Select the name of the service in which you want to modify resources', name: 'SERVICE_NAME'
        choice choices: ['true', 'false'], description: 'Choose true if you desire the target service to be promoted to "release" from "candidate"', name: 'IS_RELEASE'
    }


    stage ('Checkout source') {
        steps {
            git url: 'https://github.com/mateineaga/epm-moving-resources.git'
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
        }`
    }

}