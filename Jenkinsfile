```groovy
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'ameesha123/company-website'
        DOCKER_CREDENTIALS = 'dockerhub-credentials'
        DEPLOY_SERVER = 'ubuntu@YOUR_DEPLOYMENT_SERVER_IP'
        SSH_CREDENTIALS = 'deployment-server-ssh'
        CONTAINER_NAME = 'company-website'
        APP_PORT = '80'
    }

    stages {

        stage('BUILD') {
            stages {

                stage('Git Checkout') {
                    steps {
                        git branch: 'main',
                            url: 'https://github.com/Ameesha-vijayan/company-website.git'
                    }
                }

                stage('Docker Image Build') {
                    steps {
                        sh '''
                            docker build \
                            -t ${DOCKER_IMAGE}:${BUILD_NUMBER} \
                            -t ${DOCKER_IMAGE}:latest .
                        '''
                    }
                }

                stage('Docker Hub Login') {
                    steps {
                        withCredentials([
                            usernamePassword(
                                credentialsId: "${DOCKER_CREDENTIALS}",
                                usernameVariable: 'DOCKER_USER',
                                passwordVariable: 'DOCKER_PASS'
                            )
                        ]) {
                            sh '''
                                echo "$DOCKER_PASS" | docker login \
                                -u "$DOCKER_USER" --password-stdin
                            '''
                        }
                    }
                }

                stage('Docker Image Push') {
                    steps {
                        sh '''
                            docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                            docker push ${DOCKER_IMAGE}:latest
                        '''
                    }
                }
            }
        }

        stage('DEPLOY') {
            stages {

                stage('Connect to Deployment Server') {
                    steps {
                        echo 'Connecting to deployment server...'
                    }
                }

                stage('Pull Docker Image') {
                    steps {
                        sshagent(credentials: ["${SSH_CREDENTIALS}"]) {
                            sh '''
                                ssh -o StrictHostKeyChecking=no ${DEPLOY_SERVER} \
                                "docker pull ${DOCKER_IMAGE}:${BUILD_NUMBER}"
                            '''
                        }
                    }
                }

                stage('Stop Old Container') {
                    steps {
                        sshagent(credentials: ["${SSH_CREDENTIALS}"]) {
                            sh '''
                                ssh -o StrictHostKeyChecking=no ${DEPLOY_SERVER} \
                                "docker stop ${CONTAINER_NAME} || true &&
                                 docker rm ${CONTAINER_NAME} || true"
                            '''
                        }
                    }
                }

                stage('Run New Container') {
                    steps {
                        sshagent(credentials: ["${SSH_CREDENTIALS}"]) {
                            sh '''
                                ssh -o StrictHostKeyChecking=no ${DEPLOY_SERVER} \
                                "docker run -d \
                                --name ${CONTAINER_NAME} \
                                -p ${APP_PORT}:80 \
                                ${DOCKER_IMAGE}:${BUILD_NUMBER}"
                            '''
                        }
                    }
                }

                stage('Verify Deployment') {
                    steps {
                        sshagent(credentials: ["${SSH_CREDENTIALS}"]) {
                            sh '''
                                ssh -o StrictHostKeyChecking=no ${DEPLOY_SERVER} \
                                "docker ps --filter name=${CONTAINER_NAME}"
                            '''
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'PIPELINE SUCCESS! Application deployed successfully.'
        }

        failure {
            echo 'PIPELINE FAILED! Check Jenkins Console Output.'
        }
    }
}
```
