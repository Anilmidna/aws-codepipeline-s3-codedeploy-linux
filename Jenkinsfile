pipeline {
  agent any
  triggers { githubPush() }
  options { timestamps(); disableConcurrentBuilds() }

  stages {
    stage('Checkout') { steps { checkout scm } }

    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv('sonar-local') {
          script {
            def scannerHome = tool 'SonarScanner'
            sh """
              set -e
              "${scannerHome}/bin/sonar-scanner" \
                -Dsonar.projectKey=myweb \
                -Dsonar.sources=. \
                -Dsonar.sourceEncoding=UTF-8
            """
          }
        }
      }
    }

    stage('Quality Gate') {
      steps {
        timeout(time: 3, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
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
          if [ "$(docker ps -aq -f name=myweb)" ]; then
            docker rm -f myweb || true
          fi
          docker run -d --name myweb -p 80:80 myweb:latest
        '''
      }
    }
  }

  post {
    success { echo "Deployed. http://<EC2-Public-IP>/" }
    failure { echo "Build failed." }
  }
}
