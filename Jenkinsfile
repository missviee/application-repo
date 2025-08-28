pipeline {
    agent {
        docker { image 'docker:latest' 
                 args '-v /var/run/docker.sock:/var/run/docker.sock' }
    }
    environment {
        AWS_REGION = 'us-east-1'
        ECR_REGISTRY = '992382545251.dkr.ecr.us-east-1.amazonaws.com'
        APP_NAME = 'calculator-app'
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    def imageTag
                    if (env.CHANGE_ID) {
                        imageTag = "pr-${env.CHANGE_ID}-${env.BUILD_NUMBER}"
                    } else if (env.BRANCH_NAME == 'main') {
                        imageTag = "latest"
                    } else {
                        imageTag = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
                    }
                    env.IMAGE_TAG = imageTag
                    sh "docker build -t ${ECR_REGISTRY}/${APP_NAME}:${IMAGE_TAG} ."
                }
            }
        }
        stage('Test') {
            steps {
                sh 'pytest tests/' 
            }
            post {
                always {
                    junit 'tests/test-results/*.xml'
                }
            }
        }
        stage('Login to AWS ECR') {
            steps {
                sh '''
                    aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY
                '''
            }
        }
        stage('Push Image to ECR') {
            steps {
                sh "docker push ${ECR_REGISTRY}/${APP_NAME}:${IMAGE_TAG}"
            }
        }
        stage('Deploy & Health Check') {
            when {
                branch 'main'
            }
            steps {
                script {
                    sh '''
                    ssh -o StrictHostKeyChecking=no ubuntu@your-production-ec2-ip '
                      docker pull ${ECR_REGISTRY}/${APP_NAME}:latest &&
                      docker stop ${APP_NAME} || true &&
                      docker rm ${APP_NAME} || true &&
                      docker run -d --name ${APP_NAME} -p 80:80 ${ECR_REGISTRY}/${APP_NAME}:latest
                    '
                    '''
                    retry(5) {
                        sh 'curl --fail http://your-production-ec2-ip/health || exit 1'
                        sleep 10
                    }
                }
            }
        }
    }
}

