pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "laravel-app:latest"
        LOCALSTACK_ENDPOINT = "http://localstack:4566"
    }

    stages {
        stage('Checkout') {
            steps {
                // Use the 'github' credentials ID
                git(
                    url: 'https://github.com/mas7/my-laravel-app.git',
                    credentialsId: 'github',
                    branch: 'main'
                )
            }
        }

        stage('Build') {
            steps {
                script {
                    docker.build(DOCKER_IMAGE)
                }
            }
        }

        stage('Run Tests') {
            steps {
                sh 'docker run --rm -v $(pwd):/var/www/html laravel-app php artisan test'
            }
        }

        stage('Push to Local Docker Registry') {
            steps {
                script {
                    docker.image(DOCKER_IMAGE).push()
                }
            }
        }

        stage('Deploy to LocalStack') {
            steps {
                script {
                    // Example: Push Docker image to LocalStack ECR
                    sh """
                        aws --endpoint-url=$LOCALSTACK_ENDPOINT ecr create-repository --repository-name laravel-app || true
                        aws --endpoint-url=$LOCALSTACK_ENDPOINT ecr get-login-password | docker login --username AWS --password-stdin localhost:4566
                        docker tag $DOCKER_IMAGE:latest localhost:4566/laravel-app:latest
                        docker push localhost:4566/laravel-app:latest
                    """
                }
            }
        }

        stage('Deploy to ECS on LocalStack') {
            steps {
                script {
                    // Example: Create ECS cluster and service in LocalStack
                    sh """
                        aws --endpoint-url=$LOCALSTACK_ENDPOINT ecs create-cluster --cluster-name laravel-cluster || true

                        aws --endpoint-url=$LOCALSTACK_ENDPOINT ecs register-task-definition \
                            --family laravel-task \
                            --container-definitions '[
                                {
                                    "name": "laravel-app",
                                    "image": "localhost:4566/laravel-app:latest",
                                    "essential": true,
                                    "portMappings": [
                                        {
                                            "containerPort": 80,
                                            "hostPort": 8000
                                        }
                                    ]
                                }
                            ]' || true

                        aws --endpoint-url=$LOCALSTACK_ENDPOINT ecs create-service \
                            --cluster laravel-cluster \
                            --service-name laravel-service \
                            --task-definition laravel-task \
                            --desired-count 1
                    """
                }
            }
        }
    }

    post {
        always {
            cleanWs() // Ensure Workspace Cleanup Plugin is installed
        }
    }
}
