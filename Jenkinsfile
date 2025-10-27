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
    // ✅ 1. Check if SSH client is available
    stage('Install SSH Client') {
      steps {
        bat '''
        echo Checking for SSH client...
        ssh -V >nul 2>&1 && (
            echo SSH is available
        ) || (
            echo Installing OpenSSH Client...
            "C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\powershell.exe" -Command "Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0"
            echo SSH Client installed. Please restart Jenkins and run again.
            exit 1
        )
        '''
      }
    }

    // ✅ 2. Proceed with checkout
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    // ✅ 3. Build Docker image
    stage('Build Docker image') {
      steps {
        script {
          if (isUnix()) {
            sh "docker version"
            sh "docker build -t ${env.DOCKERHUB_REPO}:${env.IMAGE_TAG} ."
          } else {
            bat "docker version"
            bat "docker build -t ${env.DOCKERHUB_REPO}:${env.IMAGE_TAG} ."
          }
        }
      }
    }

    // ✅ 4. Push Docker image
    stage('Push Docker image') {
      steps {
        script {
          withCredentials([usernamePassword(credentialsId: env.DOCKERHUB_CRED, usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
            if (isUnix()) {
              sh """
                echo "\$DH_PASS" | docker login -u "\$DH_USER" --password-stdin
                docker push ${env.DOCKERHUB_REPO}:${env.IMAGE_TAG}
              """
            } else {
              bat """
                docker login -u %DH_USER% -p %DH_PASS%
                docker push ${env.DOCKERHUB_REPO}:${env.IMAGE_TAG}
              """
            }
          }
        }
      }
    }

    // ✅ 5. Deploy on EC2
    stage('Deploy on EC2 (pull & run)') {
      when { expression { return env.EC2_HOST?.trim() } }
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: env.EC2_CREDS,
                                  keyFileVariable: 'EC2_KEY',
                                  usernameVariable: 'EC2_USER_FROM_CREDS')]) {
          script {
            def user = (env.EC2_USER?.trim()) ?: EC2_USER_FROM_CREDS
            def remote = "${user}@${env.EC2_HOST}"

            if (isUnix()) {
              sh "chmod 600 \"$EC2_KEY\" && ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i \"$EC2_KEY\" ${remote} 'echo connected'"
              sh """
                ssh -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i "$EC2_KEY" ${remote} \\
                'docker pull ${DOCKERHUB_REPO}:${IMAGE_TAG} && \\
                 docker rm -f hello || true && \\
                 docker run -d --name hello -p 80:${APP_PORT} ${DOCKERHUB_REPO}:${IMAGE_TAG}'
              """
            } else {
              bat """
                set "KEY=%EC2_KEY%"
                for /f "tokens=* delims=" %%I in ('whoami') do set "WHO=%%I"
                icacls "%KEY%" /inheritance:r
                icacls "%KEY%" /grant:r "%WHO%:F"
                icacls "%KEY%" /remove "BUILTIN\\Users" "NT AUTHORITY\\Authenticated Users" "Everyone" 1>nul 2>nul

                ssh -T -o BatchMode=yes -o ConnectTimeout=15 -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i "%EC2_KEY%" ${remote} "docker pull ${DOCKERHUB_REPO}:${IMAGE_TAG} && (docker rm -f hello || true) && docker run -d --name hello -p 9090:${APP_PORT} ${DOCKERHUB_REPO}:${IMAGE_TAG}"
              """
            }
          }
        }
      }
    }
  }

  post {
    success {
      echo "✅ Deployed successfully: http://${EC2_HOST}:9090/"
    }
    always {
      echo 'Pipeline finished.'
    }
  }
}
