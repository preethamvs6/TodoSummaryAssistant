pipeline {
    agent any

    environment {
        DOCKER_USER = 'preethamvs6'
        EC2_HOST    = '13.235.68.52'
    }

    stages {
        stage('1. Test') {
            steps {
                dir('Backend/todo-summary-assistant') {
                    sh 'chmod +x mvnw && ./mvnw clean test'
                }
            }
        }

        stage('2. Build') {
            steps {
                sh "docker build -t ${DOCKER_USER}/todo-backend:${BUILD_NUMBER} -t ${DOCKER_USER}/todo-backend:latest Backend/todo-summary-assistant"
                sh "docker build -t ${DOCKER_USER}/todo-frontend:${BUILD_NUMBER} -t ${DOCKER_USER}/todo-frontend:latest Frontend/todo"
            }
        }

        stage('3. Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'U', passwordVariable: 'P')]) {
                    sh 'echo "$P" | docker login -u "$U" --password-stdin'
                    sh "docker push ${DOCKER_USER}/todo-backend:${BUILD_NUMBER}"
                    sh "docker push ${DOCKER_USER}/todo-backend:latest"
                    sh "docker push ${DOCKER_USER}/todo-frontend:${BUILD_NUMBER}"
                    sh "docker push ${DOCKER_USER}/todo-frontend:latest"
                }
            }
        }

        stage('4. Deploy') {
            steps {
                sh '''
                    docker network create todo-net || true
                    docker rm -f todo-frontend backend || true
                    docker run -d --name backend --network todo-net -p 8081:8080 \
                        -e SPRING_PROFILES_ACTIVE=prod \
                        ${DOCKER_USER}/todo-backend:${BUILD_NUMBER}
                    docker run -d --name todo-frontend --network todo-net -p 80:8080 \
                        ${DOCKER_USER}/todo-frontend:${BUILD_NUMBER}
                '''
            }
        }

        stage('5. Health Check') {
            steps {
                sh '''
                    sleep 10
                    curl --retry 10 --retry-delay 3 http://localhost:8081/actuator/health || true
                    curl --retry 5 --retry-delay 2 http://localhost:80/health
                '''
            }
        }
    }
}

