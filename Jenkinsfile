
def runOnDeployServer(String command) {
    withCredentials([
        sshUserPrivateKey(
            credentialsId: env.DEPLOYMENT_SSH_CREDENTIAL_ID,
            keyFileVariable: 'SSH_KEY',
            usernameVariable: 'SSH_USER'
        )
    ]) {
        sh "ssh -i \$SSH_KEY -o StrictHostKeyChecking=accept-new " +
           "\$SSH_USER@${env.DEPLOYMENT_SERVER_IP} '${command}'"
    }
}

pipeline {
    agent none

    options {
        skipDefaultCheckout(true)
    }

    environment {
        // Docker Hub details
        DOCKERHUB_USERNAME = 'YOUR_DOCKERHUB_USERNAME'
        DOCKERHUB_REPOSITORY = 'company-website'
        DOCKERHUB_CREDENTIAL_ID = 'dockerhub-credentials'

        // Deployment server details
        DEPLOYMENT_SERVER_IP = 'YOUR_DEPLOYMENT_SERVER_PRIVATE_IP'
        DEPLOYMENT_SSH_CREDENTIAL_ID = 'deployment-server-ssh'

        // Correct GitHub repository URL
        GITHUB_REPOSITORY_URL = 'https://github.com/Ameesha-vijayan/company-website.git'

        // Docker image and container
        IMAGE_NAME = "${DOCKERHUB_USERNAME}/${DOCKERHUB_REPOSITORY}"
        CONTAINER_NAME = 'company-website'
    }

    stages {

        // =====================================
        // BUILD STAGE - BUILD SERVER
        // =====================================

        stage('BUILD') {
            agent {
                label 'build-server-agent'
            }

            stages {

                stage('Git Checkout') {
                    steps {
                        git branch: 'main',
                            url: "${GITHUB_REPOSITORY_URL}"
                    }
                }

                stage('Docker Image Build') {
                    steps {
                        sh '''
                            docker build \
                            -t $IMAGE_NAME:$BUILD_NUMBER \
                            -t $IMAGE_NAME:latest .
                        '''
                    }
                }

                stage('Docker Hub Login') {
                    steps {
                        withCredentials([
                            usernamePassword(
                                credentialsId: 'dockerhub-credentials',
                                usernameVariable: 'DH_USER',
                                passwordVariable: 'DH_PASS'
                            )
                        ]) {
                            sh '''
                                echo "$DH_PASS" | docker login \
                                -u "$DH_USER" --password-stdin
                            '''
                        }
                    }
                }

                stage('Docker Image Push') {
                    steps {
                        sh '''
                            docker push $IMAGE_NAME:$BUILD_NUMBER
                            docker push $IMAGE_NAME:latest
                        '''
                    }

                    post {
                        always {
                            sh 'docker logout'
                        }
                    }
                }
            }
        }

        // =====================================
        // DEPLOY STAGE - DEPLOYMENT SERVER
        // =====================================

        stage('DEPLOY') {
            agent {
                label 'built-in'
            }

            stages {

                stage('Connect to Deployment Server') {
                    steps {
                        script {
                            runOnDeployServer(
                                'hostname && docker --version'
                            )
                        }
                    }
                }

                stage('Pull Docker Image') {
                    steps {
                        script {
                            runOnDeployServer(
                                "docker pull ${IMAGE_NAME}:latest"
                            )
                        }
                    }
                }

                stage('Stop Old Container') {
                    steps {
                        script {
                            runOnDeployServer(
                                "docker stop ${CONTAINER_NAME} || true"
                            )

                            runOnDeployServer(
                                "docker rm ${CONTAINER_NAME} || true"
                            )
                        }
                    }
                }

                stage('Run New Container') {
                    steps {
                        script {
                            runOnDeployServer(
                                "docker run -d --name ${CONTAINER_NAME} " +
                                "--restart unless-stopped " +
                                "-p 9090:80 ${IMAGE_NAME}:latest"
                            )
                        }
                    }
                }

                stage('Verify Deployment') {
                    steps {
                        script {
                            runOnDeployServer(
                                "sleep 3 && " +
                                "docker ps --filter name=${CONTAINER_NAME} && " +
                                "curl -fsS -o /dev/null http://localhost:9090 && " +
                                "echo 'Website deployed successfully'"
                            )
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'BUILD AND DEPLOYMENT SUCCESSFUL!'
            echo 'Open http://YOUR-DEPLOYMENT-PUBLIC-IP:9090'
        }

        failure {
            echo 'PIPELINE FAILED! Check Jenkins Console Output.'
        }
    }
}
