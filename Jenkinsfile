pipeline {
  agent any

  environment {
    DOCKERHUB_REPO = 'pratikkafle/hello-final' 
    IMAGE_TAG      =  new Date().format('yyyyMMdd-HHmm')
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
          branches: [[name: '*/master']],
          extensions: [],
          userRemoteConfigs: [[
            url: 'https://github.com/pratikkafle608/devops_hello.git',
            credentialsId: 'github-https-token'
          ]]
        ])
      }
    }

    stage('Check Docker') {
      steps {
        script {
          echo "Checking Docker availability..."
          try {
            bat '''
              echo Testing Docker connection...
              docker version
              echo ✓ Docker is accessible
              echo.
              echo Docker system info:
             docker images
            '''
          } catch (Exception e) {
            error """
            ❌ DOCKER NOT ACCESSIBLE!
            
            Possible fixes:
            1. Make sure Docker Desktop is RUNNING on the Jenkins server
            2. Jenkins service might need to run as the same user as Docker Desktop
            3. Check if Docker is using Windows Containers mode
            
            Error: ${e.message}
            """
          }
        }
      }
    }

    stage('Build Docker image') {
      steps {
        script {
          bat """
            echo Building Docker image...
            docker build -t ${env.DOCKERHUB_REPO}:${env.IMAGE_TAG} .
            echo ✓ Image built successfully
          """
        }
      }
    }

    stage('Push Docker image') {
      steps {
        script {
          withCredentials([usernamePassword(credentialsId: env.DOCKERHUB_CRED, usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
            bat """
              echo Logging into Docker Hub...
              docker login -u %DH_USER% -p %DH_PASS%
              echo Pushing image to Docker Hub...
              docker push ${env.DOCKERHUB_REPO}:${env.IMAGE_TAG}
              echo ✓ Image pushed successfully
            """
          }
        }
      }
    }
    
    stage('Deploy on EC2') {
      when { 
        expression { return env.EC2_HOST?.trim() } 
      }
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
              set "KEY=%EC2_KEY%"
              for /f "tokens=* delims=" %%I in ('whoami') do set "WHO=%%I"
              icacls "%KEY%" /inheritance:r
              icacls "%KEY%" /grant:r "%WHO%:F"
              
              echo Deploying to EC2...
              ssh -T -o BatchMode=yes -o ConnectTimeout=15 -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i "%EC2_KEY%" ${remote} "
                docker pull ${DOCKERHUB_REPO}:${IMAGE_TAG} && 
                (docker rm -f hello || true) && 
                docker run -d --name hello -p 9090:${APP_PORT} ${DOCKERHUB_REPO}:${IMAGE_TAG}
              "
              echo ✓ Deployment completed
            """
          }
        }
      }
    }
  }
  
  post {
    success { 
      echo "✅ Pipeline SUCCESS - Application deployed: http://${EC2_HOST}:9090/" 
    }
    failure { 
      echo "❌ Pipeline FAILED - Check the stages above for specific errors" 
    }
    always { 
      bat 'docker system prune -f || echo "Cleanup skipped"'
      echo 'Pipeline execution completed.' 
    }
  }
}
