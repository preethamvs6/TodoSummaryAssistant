
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        BACKEND_IMAGE  = 'YOUR_DOCKERHUB_USERNAME/todo-backend'
        FRONTEND_IMAGE = 'YOUR_DOCKERHUB_USERNAME/todo-frontend'
        DEPLOY_HOST    = '127.0.0.1'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Test Backend') {
            steps {
                dir('Backend/todo-summary-assistant') {
                    sh 'chmod +x mvnw'
                    sh './mvnw clean test'
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    docker build -t ${BACKEND_IMAGE}:${BUILD_NUMBER} \
                        Backend/todo-summary-assistant
                    docker build -t ${FRONTEND_IMAGE}:${BUILD_NUMBER} \
                        Frontend/todo
                '''
            }
        }

        stage('Push Images') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login \
                            -u "$DOCKER_USER" --password-stdin

                        docker push ${BACKEND_IMAGE}:${BUILD_NUMBER}
                        docker push ${FRONTEND_IMAGE}:${BUILD_NUMBER}

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy on EC2') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'app-db-creds',
                        usernameVariable: 'DB_USERNAME',
                        passwordVariable: 'DB_PASSWORD'
                    )
                ]) {
                    sh '''
                        docker pull ${BACKEND_IMAGE}:${BUILD_NUMBER}
                        docker pull ${FRONTEND_IMAGE}:${BUILD_NUMBER}

                        docker rm -f todo-backend-new 2>/dev/null || true

                        docker run -d \
                            --name todo-backend-new \
                            --restart unless-stopped \
                            -p 127.0.0.1:8080:8080 \
                            -e DB_URL="$DB_URL" \
                            -e DB_USERNAME="$DB_USERNAME" \
                            -e DB_PASSWORD="$DB_PASSWORD" \
                            ${BACKEND_IMAGE}:${BUILD_NUMBER}

                        docker run --rm --network host \
                            ${BACKEND_IMAGE}:${BUILD_NUMBER} \
                            sh -c 'exit 0' || true

                        curl --retry 12 --retry-delay 5 \
                            --retry-connrefused \
                            http://127.0.0.1:8080/
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed. Review the stage logs before retrying.'
        }
    }
}

