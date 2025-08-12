pipeline {
  agent any
  options { timestamps(); ansiColor('xterm') }

  environment {
    FRONTEND_REPO = 'https://github.com/ErezD1/FrontEndTicketProject.git'
    BACKEND_REPO  = 'https://github.com/ErezD1/BackEndTicketProject.git'
  }

  stages {
    stage('Checkout') {
      steps {
        dir('FrontEndTicketProject') { git url: env.FRONTEND_REPO, branch: 'main' }
        dir('BackEndTicketProject')  { git url: env.BACKEND_REPO,  branch: 'main' }
      }
    }

    stage('Backend: Unit tests') {
      steps {
        dir('BackEndTicketProject') {
          sh 'java -version || true'         // debug: show the JRE that Jenkins already has
          sh 'chmod +x mvnw || true'
          sh './mvnw -B -ntp test'
        }
      }
      post {
        always {
          junit allowEmptyResults: true, testResults: 'BackEndTicketProject/**/target/surefire-reports/*.xml'
        }
      }
    }

    // NOTE: We skip a standalone Node build step so you don't need the NodeJS plugin.
    // The frontend will be built inside its Docker image in the next stage.

    stage('Build images') {
      steps {
        // Build with context ".." so Dockerfiles inside ticket-infra can COPY from sibling folders
        dir('.') {
          sh 'docker build -t tickets-backend:ci  -f ticket-infra/Dockerfile.backend ..'
          sh 'docker build -t tickets-frontend:ci -f ticket-infra/Dockerfile.frontend ..'
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
          # use the same compose alias as above
          if docker compose version >/dev/null 2>&1; then DC="docker compose"; else DC="docker-compose"; fi

          # Wait for the frontend container to be up (compose sets container_name: tickets-frontend)
          for i in $(seq 1 60); do
            if docker ps --format '{{.Names}}' | grep -q '^tickets-frontend$'; then break; fi
            sleep 2
          done

          # Try up to 60s for HTTP 200 on the served index via the container network namespace
          for i in $(seq 1 60); do
            if docker run --rm --network=container:tickets-frontend curlimages/curl -sf http://localhost/ > /dev/null; then
              echo "Frontend is serving content."
              exit 0
            fi
            sleep 2
          done

          echo "Frontend did not become ready in time."
          exit 1
        '''
      }
    }
  }

  post {
    always {
      dir('ticket-infra') {
        sh 'if docker compose version >/dev/null 2>&1; then docker compose ps || true; else docker-compose ps || true; fi'
      }
    }
    cleanup {
      dir('ticket-infra') {
        sh 'if docker compose version >/dev/null 2>&1; then docker compose down -v || true; else docker-compose down -v || true; fi'
      }
    }
  }
}
