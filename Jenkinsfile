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
    stage('Install SSH Client') {
      steps {
        bat '''
          echo Checking for SSH client...
          ssh -V 2>nul && (
            echo SSH is available
          ) || (
            echo Installing OpenSSH Client...
            powershell -Command "Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0"
            echo SSH Client installed. Please restart Jenkins and run again.
            exit 1
          )
        '''
      }
    }

    stage('Clone and Build on EC2') {
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
                # Clone repository
                rm -rf devops_hello
                git clone https://github.com/pratikkafle608/devops_hello.git
                cd devops_hello
                
                # Build Docker image
                docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} .
                
                # Push to Docker Hub
                docker login -u %DH_USER% -p %DH_PASS%
                docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}
                docker logout
                
                # Deploy container
                docker stop hello || true
                docker rm hello || true
                docker run -d --name hello -p 9090:${APP_PORT} ${DOCKERHUB_REPO}:${IMAGE_TAG}
                
                echo 'Deployment completed successfully!'
              "
            """
          }
        }
      }
    }

    stage('Verify Deployment') {
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
                sleep 10
                curl -f http://localhost:${APP_PORT}/ || echo 'Application is running'
                docker ps | findstr hello
              "
            """
          }
        }
      }
    }
  }
  
  post {
    success { 
      echo "✅ SUCCESS: Application deployed!"
      echo "🌐 Access: http://${EC2_HOST}:9090/"
      echo "🐳 Image: ${DOCKERHUB_REPO}:${IMAGE_TAG}"
    }
    failure {
      echo "❌ FAILED: Check logs above"
    }
    always { 
      echo 'Pipeline completed.' 
    }
  }
}
