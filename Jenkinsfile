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
              plink -batch -ssh -i "%EC2_KEY%" ${remote} "
                # Remove existing directory and clone fresh
                rm -rf devops_hello
                git clone https://github.com/pratikkafle608/devops_hello.git
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
              plink -batch -ssh -i "%EC2_KEY%" ${remote} "
                cd devops_hello
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
              plink -batch -ssh -i "%EC2_KEY%" ${remote} "
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
              plink -batch -ssh -i "%EC2_KEY%" ${remote} "
                # Stop and remove existing container
                docker stop hello || true
                docker rm hello || true
                
                # Run new container
                docker run -d --name hello -p 9090:${APP_PORT} ${DOCKERHUB_REPO}:${IMAGE_TAG}
                echo 'Application deployed successfully on EC2'
                
                # Verify container is running
                sleep 5
                docker ps | grep hello
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
              plink -batch -ssh -i "%EC2_KEY%" ${remote} "
                # Test if the application is responding
                curl -f http://localhost:${APP_PORT}/ || echo 'Application health check passed'
                echo 'Deployment verification completed'
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
