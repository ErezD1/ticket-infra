// ticket-infra/Jenkinsfile
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
        // Run tests in a Maven container; results land in the mounted folder
        sh '''
          docker run --rm -v "$PWD/BackEndTicketProject":/app -w /app \
            maven:3.9-eclipse-temurin-21 mvn -B -ntp test
        '''
      }
      post {
        always {
          junit allowEmptyResults: true, testResults: 'BackEndTicketProject/**/target/surefire-reports/*.xml'
        }
      }
    }

    stage('Frontend: CI build (optional tests)') {
      steps {
        sh '''
          docker run --rm -v "$PWD/FrontEndTicketProject":/web -w /web node:20 \
            sh -lc "npm ci && npm run build"
        '''
      }
      post {
        always {
          archiveArtifacts artifacts: 'FrontEndTicketProject/dist/**/*', allowEmptyArchive: true
        }
      }
    }

    stage('Build images') {
      steps {
        dir('ticket-infra') {
          sh 'docker build -t tickets-backend:ci -f Dockerfile.backend ..'
          sh 'docker build -t tickets-frontend:ci -f Dockerfile.frontend ..'
        }
      }
    }

    stage('Spin up test env') {
      steps {
        dir('ticket-infra') {
          sh 'docker compose down -v || true'
          sh 'docker compose up -d --build'
        }
      }
    }

    stage('Smoke check') {
      steps {
        // Check frontend is serving and proxy works
        sh '''
          for i in $(seq 1 60); do
            if curl -sf http://localhost:8081/ > /dev/null; then exit 0; fi
            sleep 2
          done
          echo "Frontend not responding in time"; exit 1
        '''
        // Optional API poke through the proxy (adjust path if needed)
        sh 'curl -sf http://localhost:8081/api || true'
      }
    }
  }

  post {
    always {
      dir('ticket-infra') {
        sh 'docker compose ps || true'
      }
    }
    cleanup {
      dir('ticket-infra') {
        sh 'docker compose down -v || true'
      }
    }
  }
}
