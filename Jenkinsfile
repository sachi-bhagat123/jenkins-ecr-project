
pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        AWS_ACCOUNT_ID = '489952456454'
        ECR_REPOSITORY = 'jenkins-python-app'
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_NAME = "${ECR_REGISTRY}/${ECR_REPOSITORY}"
        TEST_SERVER = '3.110.29.185'
        SSH_USER = 'ubuntu'
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Downloading source code from GitHub'

                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    python3 -m venv venv
                    venv/bin/pip install -r requirements.txt
                '''
            }
        }

        stage('Run Tests') {
            steps {
                sh '''
                    venv/bin/python -m pytest -v
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    env.IMAGE_TAG = "${BUILD_NUMBER}"
                }

                sh '''
                    docker build \
                    -t ${IMAGE_NAME}:${IMAGE_TAG} .
                '''
            }
        }

        stage('Authenticate with AWS ECR') {
            steps {
                sh '''
                    aws ecr get-login-password \
                    --region ${AWS_REGION} | \
                    docker login \
                    --username AWS \
                    --password-stdin ${ECR_REGISTRY}
                '''
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh '''
                    docker push ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }

        stage('Deploy to Test Environment') {
            steps {
                sshagent(credentials: ['test-ec2-ssh']) {

                    sh '''
                        ssh ${SSH_USER}@${TEST_SERVER} "
                            aws ecr get-login-password \
                            --region ${AWS_REGION} |
                            docker login \
                            --username AWS \
                            --password-stdin ${ECR_REGISTRY}

                            docker pull ${IMAGE_NAME}:${IMAGE_TAG}

                            docker rm -f python-app || true

                            docker run -d \
                            --name python-app \
                            --restart unless-stopped \
                            -p 5000:5000 \
                            ${IMAGE_NAME}:${IMAGE_TAG}
                        "
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'CI/CD Pipeline completed successfully!'
        }

        failure {
            echo 'CI/CD Pipeline failed. Check console output.'
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}
