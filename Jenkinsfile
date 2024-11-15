pipeline {
    agent none

    stages {
        stage('Build and Test') {
            agent {
                docker {
                    image 'docker:latest'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            environment {
                DOCKER_IMAGE = "laravel-app:latest"
                LOCALSTACK_ENDPOINT = "http://localstack:4566"
            }
            steps {
                checkout scm

                sh 'docker build -t $DOCKER_IMAGE .'

                sh 'docker run --rm -v $(pwd):/var/www/html $DOCKER_IMAGE php artisan test'
            }
        }

        stage('Push to Local Docker Registry') {
            agent any
            steps {
                script {
                    docker.image("laravel-app:latest").push()
                }
            }
        }

        stage('Deploy to LocalStack') {
            agent any
            environment {
                LOCALSTACK_ENDPOINT = "http://localstack:4566"
            }
            steps {
                script {
                    sh """
                        aws --endpoint-url=$LOCALSTACK_ENDPOINT ecr create-repository --repository-name laravel-app || true
                        aws --endpoint-url=$LOCALSTACK_ENDPOINT ecr get-login-password | docker login --username AWS --password-stdin localhost:4566
                        docker tag laravel-app:latest localhost:4566/laravel-app:latest
                        docker push localhost:4566/laravel-app:latest
                    """
                }
            }
        }

        stage('Deploy to ECS on LocalStack') {
            agent any
            environment {
                LOCALSTACK_ENDPOINT = "http://localstack:4566"
            }
            steps {
                script {
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
            cleanWs()
        }
    }
}
