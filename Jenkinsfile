pipeline {
  agent any
  options { timestamps(); ansiColor('xterm'); skipDefaultCheckout(false) }

  environment {
    // Host port to avoid Jenkins' 8080
    BACKEND_PORT = '19080'
    // In-container port Spring Boot will listen on
    APP_PORT     = '8090'
  }

  stages {
    stage('Show layout (debug)') {
      steps {
        sh 'pwd && ls -la'
        sh 'java -version || true'
      }
    }

    stage('Backend: package (skip tests)') {
      steps {
        sh 'chmod +x mvnw || true'
        sh './mvnw -B -ntp -DskipTests package'
      }
    }

    stage('Build Docker image') {
      steps {
        sh 'docker build -t tickets-backend:ci -f Dockerfile.backend .'
      }
    }

    stage('Run with Compose') {
      steps {
        sh '''
          set -e
          if docker compose version >/dev/null 2>&1; then DC="docker compose"; else DC="docker-compose"; fi
          $DC down -v || true
          BACKEND_PORT=''' + '${BACKEND_PORT}' + ''' APP_PORT=''' + '${APP_PORT}' + ''' $DC up -d --build
          $DC ps
        '''
      }
    }

    stage('Smoke check') {
      steps {
        sh '''
          set -e
          echo "Waiting for app on http://localhost:${BACKEND_PORT} ..."
          for i in $(seq 1 60); do
            if curl -sf "http://localhost:${BACKEND_PORT}/" > /dev/null; then
              echo "✅ App is up: http://localhost:${BACKEND_PORT}"
              exit 0
            fi
            sleep 2
          done
          echo "❌ App did not become ready in time"
          exit 1
        '''
      }
    }
  }

  post {
    always {
      sh 'if docker compose version >/dev/null 2>&1; then docker compose ps || true; else docker-compose ps || true; fi'
      echo "Open:  http://localhost:${BACKEND_PORT}"
    }
    cleanup {
      sh 'if docker compose version >/dev/null 2>&1; then docker compose down -v || true; else docker-compose down -v || true; fi'
    }
  }
}
