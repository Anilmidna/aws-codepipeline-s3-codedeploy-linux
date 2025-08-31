pipeline {
  agent any

  // Use either webhook trigger OR polling (keep only one of these two 'triggers' blocks)
  triggers {
    // For GitHub webhooks (recommended)
    githubPush()
    // Fallback (comment above line and uncomment this if you can't open 8080 to GitHub):
    // pollSCM('H/2 * * * *') // check every ~30 mins
  }

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }
    stage('Build Docker Image') {
      steps {
        sh '''
          set -e
          docker build -t myweb:${BUILD_NUMBER} .
          docker tag myweb:${BUILD_NUMBER} myweb:latest
        '''
      }
    }
    stage('Deploy Container') {
      steps {
        sh '''
          set -e
          # Stop & remove any previous container named 'myweb'
          if [ "$(docker ps -aq -f name=myweb)" ]; then
            docker rm -f myweb || true
          fi
          # Run new container on host port 80
          docker run -d --name myweb -p 80:80 myweb:latest
        '''
      }
    }
  }
  post {
    success {
      echo "Deployed successfully. Visit http://$JENKINS_URL to check Jenkins and http://<EC2-Public-IP>/ for site."
    }
    failure {
      echo "Build failed. Check console output."
    }
  }
}
