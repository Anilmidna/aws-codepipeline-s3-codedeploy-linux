pipeline {
  agent any

  // Trigger: keep webhook OR polling (not both)
  triggers {
    githubPush()
    // pollSCM('H/2 * * * *')
  }

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  // Toggle so you can enable Sonar only when ready
  parameters {
    booleanParam(name: 'RUN_SONAR', defaultValue: false, description: 'Run SonarQube analysis & enforce Quality Gate')
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    // --- NEW: Static analysis stage (runs only when RUN_SONAR=true) ---
    stage('SonarQube Analysis') {
      when { expression { params.RUN_SONAR } }
      steps {
        // 'sonar-local' must match the name you configure in Manage Jenkins → System → SonarQube servers
        withSonarQubeEnv('sonar-local') {
          script {
            // Tool name must match Manage Jenkins → Tools → SonarQube Scanner installations
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

    // --- NEW: Wait for Quality Gate result (only when RUN_SONAR=true) ---
    stage('Quality Gate') {
      when { expression { params.RUN_SONAR } }
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
      echo "Deployed successfully. Visit http://$JENKINS_URL for Jenkins and http://<EC2-Public-IP>/ for the site."
    }
    failure {
      echo "Build failed. Check console output."
    }
  }
}
