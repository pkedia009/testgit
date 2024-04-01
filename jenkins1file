def setAWSCredentials = {
    // Use withCredentials to securely retrieve AWS account ID
    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws_cred']]) {
        env.AWS_ACCOUNT_ID = sh(script: 'aws sts get-caller-identity --query "Account" --output text', returnStdout: true).trim()
    }
}

def setAWSDefaultRegion = {
    // Use withCredentials to securely retrieve AWS default region
    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws_cred']]) {
        env.AWS_DEFAULT_REGION = sh(script: 'aws configure get region', returnStdout: true).trim()
    }
}

setAWSCredentials()
setAWSDefaultRegion()

pipeline {
    agent any
    
    environment {
        // Define AWS credentials and region
       AWS_ACCOUNT_ID = "${env.AWS_ACCOUNT_ID}"
    AWS_DEFAULT_REGION = "${env.AWS_DEFAULT_REGION}"
       
        // Define the path to your Dockerfile
        DOCKERFILE_PATH = '/var/lib/jenkins/workspace/eks'
        IMAGE_REPO_NAME = "test_eks"
        IMAGE_TAG = "v1"
        REPOSITORY_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}"
    }
 
    stages {
        stage('Create ECR Repository') {
            steps {
                script {
                    // Create ECR repository
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws_cred']]) {
                        sh "aws ecr create-repository --repository-name ${IMAGE_REPO_NAME} --region ${AWS_DEFAULT_REGION}"
                    }
                }
            }
        }
        
        stage('Cloning Git') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/develop']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: '', url: 'https://github.com/pkedia009/eks_test.git']]])     
            }
        }
        
        stage('Logging into AWS ECR') {
            steps {
                script {
                    // Get the AWS credentials from Jenkins credentials
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws_cred']]) {
                        // Use the AWS CLI to retrieve an authentication token to use for Docker login
                        def ecrLoginCmd = "aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${REPOSITORY_URI}"
                        sh ecrLoginCmd
                    }
                }
            }
        }

        stage('Building image') {
            steps {
                script {
                    dockerImage = docker.build "${IMAGE_REPO_NAME}:${IMAGE_TAG}"
                }
            }
        }
        
        stage('Pushing to ECR') {
            steps {
                script {
                    sh "docker tag ${IMAGE_REPO_NAME}:${IMAGE_TAG} ${REPOSITORY_URI}:${IMAGE_TAG}"
                    sh "docker push ${REPOSITORY_URI}:${IMAGE_TAG}"
                }
            }
        }
        
        stage('Helm Deploy') {
            steps {
                script {
                    // Deploy using Helm
                    sh "helm upgrade first --install mychart --namespace helm-deployment --set image.tag=${IMAGE_TAG}"
                }
            }
        }
    }
}
