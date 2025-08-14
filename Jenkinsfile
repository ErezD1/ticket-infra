pipeline {
  agent any
  options { timestamps(); ansiColor('xterm') }

  environment {
    // Adjust these if your folder names are different:
    BACKEND_DIR  = 'backend'         // e.g. 'server', 'spring', etc.
    FRONTEND_DIR = 'frontend'        // e.g. 'ui', 'app', etc.

    // Host ports (keep Jenkins’ 8080 free)
    BACKEND_PORT  = '19080'
    FRONTEND_PORT = '19081'
  }

  stages {
    stage('Show layout (debug)') {
      steps {
        sh 'pwd && ls -la'
        sh 'echo "Backend dir: $BACKEND_DIR  | Frontend dir: $FRONTEND_DIR"'
        sh 'ls -la "$BACKEND_DIR" || true'
        sh 'ls -la "$FRONTEND_DIR" || true'
      }
    }

    // Optional: build the backend once outside Docker just to fail fast on obvious issues
    stage('Backend: package (skip tests)') {
      steps {
        dir("${env.BACKEND_DIR}") {
          sh 'chmod +x mvnw || true'
          sh './mvnw -B -ntp -DskipTests package || true'
        }
      }
    }

    stage('Build images') {
      steps {
        // Pass the subdir names as build args so Dockerfiles can COPY correctly
        sh 'docker build -t tickets-backend:ci  --build-arg BACKEND_DIR="$BACKEND_DIR"  -f Dockerfile.backend .'
        sh 'docker build -t tickets-frontend:ci --build-arg FRONTEND_DIR="$FRONTEND_DIR" -f Dockerfile.frontend .'
      }
    }

    stage('Compose up') {
      steps {
        sh '''
          set -e
          export BACKEND_DIR FRONTEND_DIR BACKEND_PORT FRONTEND_PORT
          if docker compose version >/dev/null 2>&1; then DC="docker compose"; else DC="docker-compose"; fi
          $DC down -v || true
          $DC up -d --build
          $DC ps
        '''
      }
    }

    stage('Smoke check') {
      steps {
        sh '''
          set -e
          for i in $(seq 1 60); do
            if curl -sf "http://localhost:${FRONTEND_PORT}/" > /dev/null; then
              echo "✅ Frontend is up: http://localhost:${FRONTEND_PORT}"
              exit 0
            fi
            sleep 2
          done
          echo "❌ Frontend did not start in time"
          exit 1
        '''
      }
    }
  }

  post {
    always {
      sh 'if docker compose version >/dev/null 2>&1; then docker compose ps || true; else docker-compose ps || true; fi'
      echo "Open: http://localhost:${FRONTEND_PORT} (frontend) | http://localhost:${BACKEND_PORT} (backend)"
    }
    cleanup {
      sh 'if docker compose version >/dev/null 2>&1; then docker compose down -v || true; else docker-compose down -v || true; fi'
    }
  }
}
