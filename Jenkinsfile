pipeline{

    agent any

    environment{
        //set env vars
        AWS_ID = credentials('AWS_ID')
        ECR_REGION = 'us-east-1'
        APP_ENV = 'dev'
        SERVICE_NAME = 'user-ms'
        APP_NAME = 'my-user-microservice'
        ORGANIZATION = 'Aline-Financial-MY'
        PROJECT_NAME = 'aline-user-microservice-my'
    }

    stages{
        stage('Checkout'){
            steps{
                //get branch
                git branch: 'dev', url: 'https://github.com/markyates7748/aline-user-microservice-my.git'
                sh'git submodule init'
                sh'git submodule update'
            }
        }
        stage('SonarQube Analysis'){
            steps{
                //test with SonarQube
                withSonarQubeEnv('SonarQubeServer'){
                    sh'mvn clean test sonar:sonar'
                }
            }
        }
        stage('QualityGate'){
            steps{
                //wait for quality response
                timeout(time: 10, unit: 'MINUTES'){
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage('Package'){
            steps{
                //package project
                sh'mvn package -DskipTests'
            }
        }
        stage('Build Image'){
            environment{
                COMMIT_HASH = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
            }
            steps{
                //build docker image
                sh'docker build . -t ${APP_NAME}/${APP_ENV}/${SERVICE_NAME}:${COMMIT_HASH}'
                sh'docker tag ${APP_NAME}/${APP_ENV}/${SERVICE_NAME}:${COMMIT_HASH} ${AWS_ID}.dkr.ecr.${ECR_REGION}.amazonaws.com/${APP_NAME}:${COMMIT_HASH}'
                sh'docker tag ${APP_NAME}/${APP_ENV}/${SERVICE_NAME}:${COMMIT_HASH} ${AWS_ID}.dkr.ecr.${ECR_REGION}.amazonaws.com/${APP_NAME}:latest'
            }
        }
        stage('Push Image'){
            environment{
                COMMIT_HASH = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
            }
            steps{
                //push image to cloud
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: "cloudshark-Mark", accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]){
                    sh'aws ecr get-login-password --region $ECR_REGION | docker login --username AWS --password-stdin $AWS_ID.dkr.ecr.$ECR_REGION.amazonaws.com'
                    sh'docker push ${AWS_ID}.dkr.ecr.${ECR_REGION}.amazonaws.com/${APP_NAME}:${COMMIT_HASH}'
                    sh'docker push ${AWS_ID}.dkr.ecr.${ECR_REGION}.amazonaws.com/${APP_NAME}:latest'
                }
            }
        }
        //deploy stage
    }

    post{
        always{
            //clean up
            sh'mvn clean'
            sh'docker image prune -a -f'
        }
    }

}
