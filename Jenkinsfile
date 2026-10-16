pipeline {
    agent any

    environment {
        DOCKER_USER = 'yourdockerhubuser'
        EC2_HOST    = 'your-ec2-ip'
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
                withCredentials([sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'KEY', usernameVariable: 'USER')]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no -i \$KEY \${USER}@\${EC2_HOST} "
                            docker pull ${DOCKER_USER}/todo-backend:${BUILD_NUMBER}
                            docker pull ${DOCKER_USER}/todo-frontend:${BUILD_NUMBER}
                            docker rm -f todo-backend todo-frontend || true
                            docker run -d --name todo-backend -p 8080:8080 ${DOCKER_USER}/todo-backend:${BUILD_NUMBER}
                            docker run -d --name todo-frontend -p 80:8080 ${DOCKER_USER}/todo-frontend:${BUILD_NUMBER}
                        "
                    """
                }
            }
        }

        stage('5. Health Check') {
            steps {
                sh "curl --retry 5 --retry-delay 5 http://${EC2_HOST}:8080/actuator/health"
            }
        }
    }
}
