pipeline {
    agent any
    tools {
        jdk 'openjdk-11'
        npm 'npm 8.13.2'
        dockerTool 'docker-latest'
        }
    environment {
        POM_VERSION = getVersion()
        JAR_NAME = getJarName()
        AWS_ECR_REGION = 'eu-west-1'
        AWS_ECS_SERVICE = 'ch-dev-user-api-service'
        AWS_ECS_TASK_DEFINITION = 'ch-dev-user-api-taskdefinition'
        AWS_ECS_COMPATIBILITY = 'FARGATE'
        AWS_ECS_NETWORK_MODE = 'awsvpc'
        AWS_ECS_CPU = '256'
        AWS_ECS_MEMORY = '512'
        AWS_ECS_CLUSTER = 'ch-dev'
        AWS_ECS_TASK_DEFINITION_PATH = './ecs/container-definition-update-image.json'
        }
    

    stages{
        stage("Get a project"){
            steps{
                git branch: 'master', url: 'https://github.com/sargarravi/demo.git'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                withCredentials([string(credentialsId: 'AWS_REPOSITORY_URL_SECRET', variable: 'AWS_ECR_URL')]) {
                    script {
                        docker.build("${AWS_ECR_URL}:${POM_VERSION}", "--build-arg PACKAGE_FILE=${PACKAGE_NAME} .")
                        }
                }
            }
        }

        stage('Push Image to ECR') {
            steps {
                withCredentials([string(credentialsId: 'AWS_REPOSITORY_URL_SECRET', variable: 'AWS_ECR_URL')]) {
                    withAWS(region: "${AWS_ECR_REGION}", credentials: 'personal-aws-ecr') {
                        script {
                            def login = ecrLogin()
                            sh('#!/bin/sh -e\n' + "${login}") // hide logging
                            docker.image("${AWS_ECR_URL}:${POM_VERSION}").push()
                        }
                    }
                }
            }
        }

        stage('Deploy in ECS') {
            steps {
                withCredentials([string(credentialsId: 'AWS_EXECUTION_ROL_SECRET', variable: 'AWS_ECS_EXECUTION_ROL'),string(credentialsId: 'AWS_REPOSITORY_URL_SECRET', variable: 'AWS_ECR_URL')]) {
                    script {
                        updateContainerDefinitionJsonWithImageVersion()
                        sh("/usr/local/bin/aws ecs register-task-definition --region ${AWS_ECR_REGION} --family ${AWS_ECS_TASK_DEFINITION} --execution-role-arn ${AWS_ECS_EXECUTION_ROL} --requires-compatibilities ${AWS_ECS_COMPATIBILITY} --network-mode ${AWS_ECS_NETWORK_MODE} --cpu ${AWS_ECS_CPU} --memory ${AWS_ECS_MEMORY} --container-definitions file://${AWS_ECS_TASK_DEFINITION_PATH}")
                        def taskRevision = sh(script: "/usr/local/bin/aws ecs describe-task-definition --task-definition ${AWS_ECS_TASK_DEFINITION} | egrep \"revision\" | tr \"/\" \" \" | awk '{print \$2}' | sed 's/\"\$//'", returnStdout: true)
                        sh("/usr/local/bin/aws ecs update-service --cluster ${AWS_ECS_CLUSTER} --service ${AWS_ECS_SERVICE} --task-definition ${AWS_ECS_TASK_DEFINITION}:${taskRevision}")
                    }
                }
            }
        }

    }
}
