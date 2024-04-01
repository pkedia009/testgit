pipeline {
    agent any
    
    environment {
        // Define the path to your Dockerfile
        DOCKERFILE_PATH = '/var/lib/jenkins/workspace/eks'
        // Define the AWS ECR repository URL
        AWS_ECR_REPO_URL = '533267099239.dkr.ecr.us-east-1.amazonaws.com/test_eks'
    }
    
    stages {
        stage('Cloning Git') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/develop']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: '', url: 'https://github.com/pkedia009/eks_test.git']]])     
            }
        }
        
        stage('Building Docker image') {
            steps {
                script {
                    // Build the Docker image using the specified Dockerfile path
                    def dockerImage = docker.build("-f ${DOCKERFILE_PATH}/Dockerfile .")
                    // Tag the Docker image with the build number
                    dockerImage.tag("${BUILD_NUMBER}")
                }
            }
        }
        
        stage('Pushing to ECR') {
            when {
                branch 'develop'
            }
            steps {
                echo '=== Pushing Docker Image ==='
                script {
                    // Get the Git commit hash
                    def GIT_COMMIT_HASH = sh(script: "git log -n 1 --pretty=format:'%H'", returnStdout: true).trim()
                    def SHORT_COMMIT = GIT_COMMIT_HASH.take(7)
                    withCredentials([aws(credentials: 'aws_cred', region: 'us-east-1')]) {
                sh "aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin $AWS_ECR_REPO_URL"
                
                // Push the Docker image to AWS ECR
                docker.withRegistry("${AWS_ECR_REPO_URL}", 'ecr:us-east-1') {
                    dockerImage.push("${AWS_ECR_REPO_URL}:${BUILD_NUMBER}")
                }
                }
            }
        }}
        
        stage ('Helm Deploy') {
            steps {
                script {
                    // Deploy using Helm
                    sh "helm upgrade first --install mychart --namespace helm-deployment --set image.tag=$BUILD_NUMBER"
                }
            }
        }
    }
}
