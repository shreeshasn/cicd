pipeline {
  agent any

  environment {
    IMAGE = "shreeshasn/quizmaster"
    IMAGE_TAG = "latest"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Install & Build') {
      steps {
        // Adjust these commands if your build differs
        sh 'npm ci'
        sh 'npm run build || true'
      }
    }

    stage('Prepare build-time secret (temporary)') {
      steps {
        // Create .env.local for build only if the secret exists
        withCredentials([string(credentialsId: 'quiz-api-key', variable: 'GEMINI_SECRET')]) {
          sh '''
            if [ -n "$GEMINI_SECRET" ]; then
              echo "GEMINI_API_KEY=$GEMINI_SECRET" > .env.local
              echo ".env.local created"
            fi
          '''
        }
      }
    }

    stage('Docker Build & Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh '''
            set -e
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
            docker build -t ${IMAGE}:${IMAGE_TAG} .
            docker push ${IMAGE}:${IMAGE_TAG}
          '''
        }
      }
    }
  }

  post {
    success {
      withCredentials([string(credentialsId: 'slack-webhook', variable: 'SLACK_WEBHOOK')]) {
        sh '''
          payload='{"text":"✅ QuizMaster Build SUCCESS — image: '"${IMAGE}:${IMAGE_TAG}"'"}'
          curl -s -X POST -H "Content-type: application/json" --data "$payload" "$SLACK_WEBHOOK" || true
        '''
      }
    }
    failure {
      withCredentials([string(credentialsId: 'slack-webhook', variable: 'SLACK_WEBHOOK')]) {
        sh '''
          payload='{"text":"❌ QuizMaster Build FAILED — check Jenkins console for details (Build #'"${BUILD_NUMBER}"')"}'
          curl -s -X POST -H "Content-type: application/json" --data "$payload" "$SLACK_WEBHOOK" || true
        '''
      }
    }
    cleanup {
      // remove temporary file
      sh 'rm -f .env.local || true'
    }
  }
}
