pipeline {
    agent any

    parameters {
        string(name: 'VERSION', description: 'Enter the APP VERSION')
    }

    environment {
        AWS_ACCOUNT_ID = "124355663661"
        REGION = "ap-northeast-2"
        REPO_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/sre-training"
        DOCKER_IMAGE = "sre-training-cart:${VERSION}"
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_REGISTRY_CREDENTIALS = 'docker-creds'
    }

    stages {
        stage('Clone') {
            steps {
                echo '🔁 Cloning the GitHub repository...'
                git url: 'https://github.com/Msocial123/fss-Retail-App_kubernetes.git', branch: 'master'
            }
        }

        stage('Docker Build') {
            steps {
                echo "🐳 Building Docker image ${DOCKER_IMAGE}..."
                sh "docker build -t ${DOCKER_IMAGE} ."
            }
        }
        stage('Push to ECR') {
            steps {
                script {
                    withAWS(credentials: 'aws-creds', region: "${REGION}") {
                        echo "📦 Pushing Docker image to AWS ECR..."
                        sh """
                            aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${REPO_URI}
                            docker tag ${DOCKER_IMAGE} ${REPO_URI}:${VERSION}
                            docker push ${REPO_URI}:${VERSION}
                        """
                    }
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${DOCKER_REGISTRY_CREDENTIALS}", passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                    echo "📤 Pushing Docker image to Docker Hub..."
                    sh """
                        docker login -u ${DOCKER_USERNAME} -p ${DOCKER_PASSWORD}
                        docker tag ${DOCKER_IMAGE} muralisocial123/${DOCKER_IMAGE}
                        docker push muralisocial123/${DOCKER_IMAGE}
                    """
                }
            }
        }

        stage('Run Docker Compose') {
            steps {
                echo "🚀 Starting containers using Docker Compose..."
                sh "docker-compose up -d"
            }
            post {
                success {
                    echo "✅ Docker containers started successfully."
                }
                failure {
                    echo "❌ Failed to start Docker containers."
                }
            }
        }

        stage('Cleanup') {
            steps {
                echo "🧹 Cleaning up Docker resources..."
                sh "docker system prune -af"
            }
        }
    }

    post {
        always {
            echo "🧼 Final cleanup..."
            cleanWs()
        }
    }
}