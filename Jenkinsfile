pipeline {
    agent any  // Runs on any available agent (e.g., your Windows Jenkins node)

    environment {
        DOCKER_IMAGE = 'pratikkafle/hello-final'
        DOCKER_TAG = "${env.BUILD_NUMBER}-${new Date().format('yyyyMMdd-HHmm')}"  // e.g., 1-20231027-1001
        DOCKER_CREDENTIALS_ID = 'docker-hub-credentials'  // Jenkins credential ID for Docker Hub login
    }

    stages {
        stage('Declarative: Checkout SCM') {
            steps {
                checkout scm  // Checks out your repo
            }
        }

        stage('Check Docker') {
            steps {
                script {
                    try {
                        if (isUnix()) {
                            sh 'docker version'
                        } else {
                            bat 'docker version'
                        }
                        echo 'Docker daemon is running and accessible.'
                    } catch (Exception e) {
                        echo "Docker is not accessible. Ensure Docker Desktop is running on the Jenkins machine. Error: ${e.getMessage()}"
                        error('Aborting pipeline: Docker daemon not available.')
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    try {
                        def buildCmd = "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
                        if (isUnix()) {
                            sh buildCmd
                        } else {
                            bat buildCmd
                        }
                        echo "Docker image built successfully: ${DOCKER_IMAGE}:${DOCKER_TAG}"
                    } catch (Exception e) {
                        echo "Failed to build Docker image. Error: ${e.getMessage()}"
                        error('Build failed.')
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    try {
                        withCredentials([usernamePassword(credentialsId: DOCKER_CREDENTIALS_ID, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                            def loginCmd = "docker login -u ${DOCKER_USER} -p ${DOCKER_PASS}"
                            def pushCmd = "docker push ${DOCKER_IMAGE}:${DOCKER_TAG}"
                            if (isUnix()) {
                                sh loginCmd
                                sh pushCmd
                            } else {
                                bat loginCmd
                                bat pushCmd
                            }
                        }
                        echo "Docker image pushed successfully to Docker Hub."
                    } catch (Exception e) {
                        echo "Failed to push Docker image. Check credentials and network. Error: ${e.getMessage()}"
                        error('Push failed.')
                    }
                }
            }
        }

        stage('Deploy on EC2') {
            steps {
                script {
                    try {
                        // Assumes SSH access to EC2. Replace with your EC2 details.
                        def ec2Host = 'ec2-user@your-ec2-public-ip'  // e.g., ec2-user@54.123.45.67
                        def deployCmd = """
                            ssh -o StrictHostKeyChecking=no ${ec2Host} '
                                docker pull ${DOCKER_IMAGE}:${DOCKER_TAG} &&
                                docker run -d -p 80:80 ${DOCKER_IMAGE}:${DOCKER_TAG}
                            '
                        """
                        if (isUnix()) {
                            sh deployCmd
                        } else {
                            // For Windows, you might need an SSH tool like OpenSSH or PuTTY. Adjust if needed.
                            bat "ssh -o StrictHostKeyChecking=no ${ec2Host} \"docker pull ${DOCKER_IMAGE}:${DOCKER_TAG} && docker run -d -p 80:80 ${DOCKER_IMAGE}:${DOCKER_TAG}\""
                        }
                        echo "Deployment to EC2 completed successfully."
                    } catch (Exception e) {
                        echo "Failed to deploy on EC2. Check SSH keys, EC2 access, and Docker on EC2. Error: ${e.getMessage()}"
                        error('Deployment failed.')
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline finished. Cleaning up...'
            script {
                try {
                    if (isUnix()) {
                        sh 'docker system prune -f'  // Remove dangling images/containers
                    } else {
                        bat 'docker system prune -f'
                    }
                } catch (Exception e) {
                    echo "Cleanup failed: ${e.getMessage()}"
                }
            }
        }
        success {
            echo '✅ Pipeline succeeded!'
        }
        failure {
            echo '❌ Pipeline failed - check logs above.'
        }
    }
}
