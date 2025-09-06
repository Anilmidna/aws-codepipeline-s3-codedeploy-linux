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
    script {
      sh 'set -e; docker build -t myweb:${GIT_COMMIT} -t myweb:latest .'
      // Use the local image ID (reliable even if RepoDigests is empty)
      env.BUILT_IMAGE_ID = sh(
        returnStdout: true,
        script: "docker inspect --format='{{.Id}}' myweb:latest"
      ).trim()
    }
  }
}


stage('Deploy Container') {
  steps {
    script {
      // Running container's image ID (empty if container doesn't exist yet)
      def runningId = sh(
        returnStdout: true,
        script: "docker inspect --format='{{.Image}}' myweb 2>/dev/null || true"
      ).trim()

      if (runningId != env.BUILT_IMAGE_ID) {
        echo "Image changed → redeploying"
        sh '''
          set -e
          docker rm -f myweb || true
          docker run -d --name myweb -p 80:80 myweb:latest
        '''
      } else {
        echo "Image unchanged → skipping redeploy"
      }
    }
  }
}


  post {
    success { echo "Deployed. http://<EC2-Public-IP>/" }
    failure { echo "Build failed." }
  }
}
