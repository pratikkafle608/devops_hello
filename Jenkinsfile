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
    stage('Test EC2 Connection') {
      steps {
        withCredentials([sshUserPrivateKey(
          credentialsId: env.EC2_CREDS,
          keyFileVariable: 'EC2_KEY',
          usernameVariable: 'EC2_USER_FROM_CREDS'
        )]) {
          script {
            def user = (env.EC2_USER?.trim()) ?: EC2_USER_FROM_CREDS
            def remote = "${user}@${env.EC2_HOST}"

            // Test if native SSH is available
            bat """
              ssh -V 2>nul && (
                echo Native SSH available
                ssh -o StrictHostKeyChecking=no -i "%EC2_KEY%" ${remote} "echo 'SSH connection successful'"
              ) || (
                echo Native SSH not available, checking for plink...
                where plink 2>nul && (
                  echo Plink available
                  plink -batch -ssh -i "%EC2_KEY%" ${remote} "echo 'Plink connection successful'"
                ) || (
                  echo "ERROR: Neither SSH nor Plink available. Please install OpenSSH or Plink."
                  exit 1
                )
              )
            """
          }
        }
      }
    }

    stage('Clone Repository on EC2') {
      steps {
        withCredentials([sshUserPrivateKey(
          credentialsId: env.EC2_CREDS,
          keyFileVariable: 'EC2_KEY',
          usernameVariable: 'EC2_USER_FROM_CREDS'
        )]) {
          script {
            def user = (env.EC2_USER?.trim()) ?: EC2_USER_FROM_CREDS
            def remote = "${user}@${env.EC2_HOST}"

            bat """
              ssh -o StrictHostKeyChecking=no -i "%EC2_KEY%" ${remote} "
                # Remove existing directory and clone fresh
                sudo rm -rf /home/ubuntu/devops_hello
                git clone https://github.com/pratikkafle608/devops_hello.git /home/ubuntu/devops_hello
                echo 'Repository cloned successfully on EC2'
              "
            """
          }
        }
      }
    }

    stage('Build Docker Image on EC2') {
      steps {
        withCredentials([sshUserPrivateKey(
          credentialsId: env.EC2_CREDS,
          keyFileVariable: 'EC2_KEY',
          usernameVariable: 'EC2_USER_FROM_CREDS'
        )]) {
          script {
            def user = (env.EC2_USER?.trim()) ?: EC2_USER_FROM_CREDS
            def remote = "${user}@${env.EC2_HOST}"

            bat """
              ssh -o StrictHostKeyChecking=no -i "%EC2_KEY%" ${remote} "
                cd /home/ubuntu/devops_hello
                docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} .
                echo 'Docker image built successfully on EC2'
              "
            """
          }
        }
      }
    }

    stage('Push Docker Image from EC2') {
      steps {
        withCredentials([
          sshUserPrivateKey(credentialsId: env.EC2_CREDS, keyFileVariable: 'EC2_KEY', usernameVariable: 'EC2_USER_FROM_CREDS'),
          usernamePassword(credentialsId: env.DOCKERHUB_CRED, usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')
        ]) {
          script {
            def user = (env.EC2_USER?.trim()) ?: EC2_USER_FROM_CREDS
            def remote = "${user}@${env.EC2_HOST}"

            bat """
              ssh -o StrictHostKeyChecking=no -i "%EC2_KEY%" ${remote} "
                docker login -u %DH_USER% -p %DH_PASS%
                docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}
                docker logout
                echo 'Docker image pushed successfully to Docker Hub'
              "
            """
          }
        }
      }
    }
    
    stage('Deploy on EC2') {
      steps {
        withCredentials([sshUserPrivateKey(
          credentialsId: env.EC2_CREDS,
          keyFileVariable: 'EC2_KEY',
          usernameVariable: 'EC2_USER_FROM_CREDS'
        )]) {
          script {
            def user = (env.EC2_USER?.trim()) ?: EC2_USER_FROM_CREDS
            def remote = "${user}@${env.EC2_HOST}"

            bat """
              ssh -o StrictHostKeyChecking=no -i "%EC2_KEY%" ${remote} "
                # Stop and remove existing container
                docker stop hello || true
                docker rm hello || true
                
                # Run new container
                docker run -d --name hello -p 9090:${APP_PORT} ${DOCKERHUB_REPO}:${IMAGE_TAG}
                echo 'Application deployed successfully on EC2'
              "
            """
          }
        }
      }
    }
  }
  
  post {
    success { 
      echo "✅ SUCCESS: Application deployed successfully!"
      echo "🌐 Access your application at: http://${EC2_HOST}:9090/"
      echo "🐳 Docker Image: ${DOCKERHUB_REPO}:${IMAGE_TAG}"
    }
    failure {
      echo "❌ FAILED: Pipeline execution failed"
      echo "Check the console output above for details"
    }
    always { 
      echo 'Pipeline execution completed.' 
    }
  }
}
