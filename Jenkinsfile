pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = 'pratikkafle/hello-final' 
        IMAGE_TAG      = new Date().format('yyyyMMdd-HHmm')
        DOCKERHUB_CRED = 'dockerhub-creds'
        EC2_HOST       = 'ec2-18-118-241-253.us-east-2.compute.amazonaws.com'
        EC2_USER       = 'kpratik'
        EC2_CREDS      = 'ec2-creds'                
        APP_PORT       = '8080'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/pratikkafle608/devops_hello.git',
                        credentialsId: 'github-https-token'
                    ]]
                ])
            }
        }

        stage('Build Docker image') {
            steps {
                script {
                    if (isUnix()) {
                        sh """
                            docker build --no-cache -t ${env.DOCKERHUB_REPO}:${env.IMAGE_TAG} .
                            docker tag ${env.DOCKERHUB_REPO}:${env.IMAGE_TAG} ${env.DOCKERHUB_REPO}:latest
                        """
                    } else {
                        bat """
                            docker build --no-cache -t ${env.DOCKERHUB_REPO}:${env.IMAGE_TAG} .
                            docker tag ${env.DOCKERHUB_REPO}:${env.IMAGE_TAG} ${env.DOCKERHUB_REPO}:latest
                        """
                    }
                }
            }
        }

        stage('Push Docker image') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: env.DOCKERHUB_CRED, usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                        if (isUnix()) {
                            sh """
                                echo "\$DH_PASS" | docker login -u "\$DH_USER" --password-stdin
                                docker push ${env.DOCKERHUB_REPO}:${env.IMAGE_TAG}
                                docker push ${env.DOCKERHUB_REPO}:latest
                            """
                        } else {
                            bat """
                                echo %DH_PASS% | docker login -u %DH_USER% --password-stdin
                                docker push ${env.DOCKERHUB_REPO}:${env.IMAGE_TAG}
                                docker push ${env.DOCKERHUB_REPO}:latest
                            """
                        }
                    }
                }
            }
        }
        
        stage('Deploy on EC2') {
            when { expression { return env.EC2_HOST?.trim() } }
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: env.EC2_CREDS,
                                          keyFileVariable: 'EC2_KEY',
                                          usernameVariable: 'EC2_USER_FROM_CREDS')]) {
                    script {
                        def user = (env.EC2_USER?.trim()) ?: EC2_USER_FROM_CREDS
                        def remote = "${user}@${env.EC2_HOST}"

                        if (isUnix()) {
                            sh "chmod 600 \"$EC2_KEY\""
                            sh """
                                ssh -o StrictHostKeyChecking=no -i "$EC2_KEY" ${remote} '
                                    docker pull ${DOCKERHUB_REPO}:${IMAGE_TAG}
                                    docker stop hello || true
                                    docker rm hello || true
                                    docker run -d --name hello -p 9090:${APP_PORT} ${DOCKERHUB_REPO}:${IMAGE_TAG}
                                '
                            """
                        } else {
                            bat """
                                plink -batch -ssh -i "%EC2_KEY%" ${remote} "
                                    docker pull ${DOCKERHUB_REPO}:${IMAGE_TAG}
                                    docker stop hello || true
                                    docker rm hello || true
                                    docker run -d --name hello -p 9090:${APP_PORT} ${DOCKERHUB_REPO}:${IMAGE_TAG}
                                "
                            """
                        }
                    }
                }
            }
        }
    }
    
    post {
        success { 
            echo "Successfully deployed: http://${EC2_HOST}:9090/"
        }
        failure {
            echo "Pipeline failed - check logs above"
        }
        always { 
            echo 'Pipeline execution completed.'
        }
    }
}
