pipeline {
    agent none
    environment {
        AWS_REGION   = "us-east-1"
        ECR_REGISTRY = "992382545251.dkr.ecr.us-east-1.amazonaws.com"
        APP_NAME     = "calc-app"
        PROD_HOST    = "ubuntu@54.144.113.195"
    }

    stages {
        stage('CI for Pull Requests') {
            when { changeRequest() }
            agent { docker { image 'python:3.11' } }
            stages {
                stage('Build') {
                    steps {
                        script {
                            env.IMAGE_TAG = "pr-${CHANGE_ID}-${BUILD_NUMBER}"
                            sh "docker build -t ${ECR_REGISTRY}/${APP_NAME}:${IMAGE_TAG} ."
                        }
                    }
                }
                stage('Test') {
                    steps {
                        sh 'pytest --junitxml=tests/test-results/results.xml'
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
                            aws ecr get-login-password --region $AWS_REGION \
                              | docker login --username AWS --password-stdin $ECR_REGISTRY
                        '''
                    }
                }
                stage('Push Image to ECR') {
                    steps {
                        sh "docker push ${ECR_REGISTRY}/${APP_NAME}:${IMAGE_TAG}"
                    }
                }
            }
        }

        stage('CD for Main') {
            when { branch 'main' }
            agent { docker { image 'python:3.11' } }
            stages {
                stage('Build') {
                    steps {
                        script {
                            env.IMAGE_TAG = "latest"
                            sh "docker build -t ${ECR_REGISTRY}/${APP_NAME}:${IMAGE_TAG} ."
                        }
                    }
                }
                stage('Test') {
                    steps {
                        sh 'pytest --junitxml=tests/test-results/results.xml'
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
                            aws ecr get-login-password --region $AWS_REGION \
                              | docker login --username AWS --password-stdin $ECR_REGISTRY
                        '''
                    }
                }
                stage('Push Image to ECR') {
                    steps {
                        sh "docker push ${ECR_REGISTRY}/${APP_NAME}:${IMAGE_TAG}"
                    }
                }
                stage('Deploy & Health Check') {
                    steps {
                        sshagent(['prod-ec2-key']) {
                            sh '''
                                ssh -o StrictHostKeyChecking=no $PROD_HOST "
                                  aws ecr get-login-password --region $AWS_REGION \
                                    | docker login --username AWS --password-stdin $ECR_REGISTRY &&
                                  docker pull ${ECR_REGISTRY}/${APP_NAME}:${IMAGE_TAG} &&
                                  docker stop ${APP_NAME} || true &&
                                  docker rm ${APP_NAME} || true &&
                                  docker run -d --name ${APP_NAME} -p 80:5000 ${ECR_REGISTRY}/${APP_NAME}:${IMAGE_TAG}
                                "
                            '''
                        }
                        retry(5) {
                            sh "curl --fail http://${PROD_HOST#*@}/health || exit 1"
                            sleep 10
                        }
                    }
                }
            }
        }
    }
}

