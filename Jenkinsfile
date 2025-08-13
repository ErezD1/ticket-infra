pipeline {
  agent any
  options {
    timestamps()
    ansiColor('xterm')
    skipDefaultCheckout(true)
  }

  environment {
    FRONTEND_REPO = 'https://github.com/ErezD1/FrontEndTicketProject.git'
    BACKEND_REPO  = 'https://github.com/ErezD1/BackEndTicketProject.git'
    FRONTEND_PORT = '19081'
    BACKEND_PORT  = '19080'
    DB_HOST_PORT  = '33306'
  }

  stages {

    stage('Checkout') {
      steps {
        dir('FrontEndTicketProject') { git url: env.FRONTEND_REPO, branch: 'main' }
        dir('BackEndTicketProject')  { git url: env.BACKEND_REPO,  branch: 'main' }
        sh 'mkdir -p ticket-infra'
      }
    }

    stage('Prepare infra files') {
      steps {
        dir('ticket-infra') {
          // Dockerfile.backend
          sh '''
cat > Dockerfile.backend <<'EOF'
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /src
COPY BackEndTicketProject/pom.xml .
COPY BackEndTicketProject/src ./src
RUN mvn -B -DskipTests package

FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /src/target/*.jar /app/app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","/app/app.jar"]
EOF
'''

          // Dockerfile.frontend
          sh '''
cat > Dockerfile.frontend <<'EOF'
FROM node:20 AS build
WORKDIR /web
COPY FrontEndTicketProject/package*.json ./
RUN npm ci
COPY FrontEndTicketProject/ .
RUN npm run build

FROM nginx:alpine
COPY ticket-infra/nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /web/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx","-g","daemon off;"]
EOF
'''

          // nginx.conf
          sh '''
cat > nginx.conf <<'EOF'
server {
  listen 80;
  root /usr/share/nginx/html;
  index index.html;

  location / { try_files $uri /index.html; }

  # proxy API calls to Spring Boot service
  location /api/ {
    proxy_pass http://backend:8080/api/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }
}
EOF
'''

          // docker-compose.yml
          sh """
cat > docker-compose.yml <<'EOF'
services:
  db:
    image: mysql:8.0
    container_name: tickets-mysql
    environment:
      MYSQL_DATABASE: tickets
      MYSQL_USER: tickets
      MYSQL_PASSWORD: tickets
      MYSQL_ROOT_PASSWORD: root
    command: ["--default-authentication-plugin=mysql_native_password"]
    healthcheck:
      test: ["CMD-SHELL","mysqladmin ping -h localhost -proot --silent"]
      interval: 5s
      timeout: 3s
      retries: 20
    ports:
      - "${DB_HOST_PORT}:3306"
    volumes:
      - mysql_data:/var/lib/mysql

  backend:
    build:
      context: ..
      dockerfile: ticket-infra/Dockerfile.backend
    container_name: tickets-backend
    depends_on:
      db:
        condition: service_healthy
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://db:3306/tickets?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC
      SPRING_DATASOURCE_USERNAME: tickets
      SPRING_DATASOURCE_PASSWORD: tickets
    ports:
      - "${BACKEND_PORT}:8080"

  frontend:
    build:
      context: ..
      dockerfile: ticket-infra/Dockerfile.frontend
    container_name: tickets-frontend
    depends_on: [backend]
    ports:
      - "${FRONTEND_PORT}:80"

volumes:
  mysql_data:
EOF
"""
        }
      }
    }

    stage('Backend: Package (skip tests)') {
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
          for i in $(seq 1 90); do
            if curl -sf "http://localhost:${FRONTEND_PORT}/" > /dev/null; then
              echo "Frontend is up at http://localhost:${FRONTEND_PORT}"
              exit 0
            fi
            sleep 2
          done
          echo "Frontend didn't start in time"
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
      echo "Open:  http://localhost:${FRONTEND_PORT}  (frontend)  |  http://localhost:${BACKEND_PORT}  (backend)  |  MySQL: localhost:${DB_HOST_PORT}"
    }
    cleanup {
      dir('ticket-infra') {
        sh 'if docker compose version >/dev/null 2>&1; then docker compose down -v || true; else docker-compose down -v || true; fi'
      }
    }
  }
}
