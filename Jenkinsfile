pipeline {
  agent any
  options { timestamps(); ansiColor('xterm') }

  environment {
    FRONTEND_REPO = 'https://github.com/ErezD1/FrontEndTicketProject.git'
    BACKEND_REPO  = 'https://github.com/ErezD1/BackEndTicketProject.git'
    FRONTEND_PORT = '19081'
    BACKEND_PORT  = '19080'
  }

  stages {
    stage('Checkout app repos') {
      steps {
        dir('FrontEndTicketProject') { git url: env.FRONTEND_REPO, branch: 'main' }
        dir('BackEndTicketProject')  { git url: env.BACKEND_REPO,  branch: 'main' }
      }
    }

    stage('Backend: package (skip tests)') {
      steps {
        dir('BackEndTicketProject') {
          sh 'java -version || true'
          sh 'chmod +x mvnw || true'
          sh './mvnw -B -ntp -DskipTests package'
        }
      }
    }

    stage('Build images') {
      steps {
        // IMPORTANT: run from ticket-infra with context ".."
        dir('ticket-infra') {
          sh 'docker build -t tickets-backend:ci  -f Dockerfile.backend ..'
          sh 'docker build -t tickets-frontend:ci -f Dockerfile.frontend ..'
        }
      }
    }

    stage('Spin up test env') {
      steps {
        dir('ticket-infra') {
          sh '''
            set -e
            if docker compose version >/dev/null 2>&1; then DC="docker compose"; else DC="docker-compose"; fi
            $DC down -v || true
            $DC up -d --build
            $DC ps
          '''
        }
      }
    }

    stage('Smoke check') {
      steps {
        sh '''
          set -e
          for i in $(seq 1 60); do
            if curl -sf http://localhost:${FRONTEND_PORT}/ > /dev/null; then
              echo "Frontend is up at http://localhost:${FRONTEND_PORT}"
              exit 0
            fi
            sleep 2
          done
          echo "Frontend didn't start in time"; exit 1
        '''
      }
    }
  }

  post {
    always {
      sh 'if docker compose version >/dev/null 2>&1; then docker compose ps || true; else docker-compose ps || true; fi'
      echo "Try locally: http://localhost:${FRONTEND_PORT}  (frontend),  http://localhost:${BACKEND_PORT} (backend)"
    }
    cleanup {
      sh 'if docker compose version >/dev/null 2>&1; then docker compose down -v || true; else docker-compose down -v || true; fi'
    }
  }
}
