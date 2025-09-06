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

stage('Build Image') {
  steps {
    script {
      sh 'set -e; docker build -t myweb:${GIT_COMMIT} -t myweb:latest .'
      env.BUILT_DIGEST = sh(returnStdout: true, script: "docker inspect --format='{{index .RepoDigests 0}}' myweb:latest").trim()
    }
  }


stage('Deploy (only if changed)') {
  steps {
    script {
      def running = sh(returnStdout: true, script: "docker inspect --format='{{.Image}}' myweb 2>/dev/null || true").trim()
      if (running != env.BUILT_DIGEST) {
        sh '''
          docker rm -f myweb || true
          docker run -d --name myweb -p 80:80 myweb:latest
        '''
        echo "Redeployed (image changed)."
      } else {
        echo "Image unchanged → skipping redeploy."
      }
    }
  }
}


  post {
    success { echo "Deployed. http://<EC2-Public-IP>/" }
    failure { echo "Build failed." }
  }
}
