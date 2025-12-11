pipeline {
  agent any

  environment {
    IMAGE = "shreeshasn/quizmaster"
    IMAGE_TAG = "latest"
  }

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '30'))
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Install & Build') {
      steps {
        script {
          if (isUnix()) {
            sh '''
              node --version || true
              npm --version || true
              npm ci
              npm run build || true
            '''
          } else {
            bat '''
              node --version || echo node not found
              npm --version || echo npm not found
              npm ci
              npm run build || exit /b 0
            '''
          }
        }
      }
    }

    stage('Prepare build-time secret (temporary)') {
      steps {
        withCredentials([string(credentialsId: 'quiz-api-key', variable: 'GEMINI_SECRET')]) {
          script {
            if (isUnix()) {
              // create .env.local only if secret present
              sh '''
                if [ -n "$GEMINI_SECRET" ]; then
                  printf "GEMINI_API_KEY=%s\n" "$GEMINI_SECRET" > .env.local
                  echo ".env.local created"
                else
                  echo "No GEMINI secret provided; skipping .env.local creation"
                fi
              '''
            } else {
              // on Windows, use PowerShell to write .env.local if secret exists
              bat '''
                if not "%GEMINI_SECRET%"=="" (
                  powershell -Command "Set-Content -Path .env.local -Value ('GEMINI_API_KEY=' + $env:GEMINI_SECRET)"
                  echo .env.local created
                ) else (
                  echo No GEMINI secret provided; skipping .env.local creation
                )
              '''
            }
          }
        }
      }
    }

    stage('Docker Build & Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          script {
            if (isUnix()) {
              sh '''
                set -e
                echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                docker build -t $IMAGE:$IMAGE_TAG .
                docker push $IMAGE:$IMAGE_TAG
              '''
            } else {
              // Use PowerShell within a cmd/bat wrapper to access env vars reliably
              bat '''
                powershell -Command "Write-Output $env:DOCKER_PASS | docker login -u $env:DOCKER_USER --password-stdin"
                docker build -t %IMAGE%:%IMAGE_TAG% .
                docker push %IMAGE%:%IMAGE_TAG%
              '''
            }
          }
        }
      }
    }
  }

  post {
    success {
      withCredentials([string(credentialsId: 'slack-webhook', variable: 'SLACK_WEBHOOK')]) {
        script {
          if (isUnix()) {
            sh '''
              payload=$(printf '{"text":"✅ QuizMaster Build SUCCESS — image: %s:%s"}' "$IMAGE" "$IMAGE_TAG")
              curl -s -X POST -H "Content-type: application/json" --data "$payload" "$SLACK_WEBHOOK" || true
            '''
          } else {
            // single-quoted Groovy string prevents Groovy interpolation problems
            bat '''
              powershell -Command "$payload = ConvertTo-Json @{ text = '✅ QuizMaster Build SUCCESS — image: %IMAGE%:%IMAGE_TAG%' }; Invoke-RestMethod -Uri %SLACK_WEBHOOK% -Method Post -Body $payload -ContentType 'application/json'"
            '''
          }
        }
      }
    }

    failure {
      withCredentials([string(credentialsId: 'slack-webhook', variable: 'SLACK_WEBHOOK')]) {
        script {
          if (isUnix()) {
            sh '''
              payload=$(printf '{"text":"❌ QuizMaster Build FAILED — see Jenkins console (Build #%s)"}' "$BUILD_NUMBER")
              curl -s -X POST -H "Content-type: application/json" --data "$payload" "$SLACK_WEBHOOK" || true
            '''
          } else {
            bat '''
              powershell -Command "$payload = ConvertTo-Json @{ text = '❌ QuizMaster Build FAILED — see Jenkins console (Build #%BUILD_NUMBER%)' }; Invoke-RestMethod -Uri %SLACK_WEBHOOK% -Method Post -Body $payload -ContentType 'application/json'"
            '''
          }
        }
      }
    }

    cleanup {
      script {
        if (isUnix()) {
          sh 'rm -f .env.local || true'
        } else {
          bat 'if exist .env.local del .env.local'
        }
      }
    }
  }
}
