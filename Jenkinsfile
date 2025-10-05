pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        DOCKER_HOST           = "tcp://docker:2376"
        DOCKER_TLS_VERIFY     = "1"
        DOCKER_CERT_PATH      = "/certs/client/"
    }

    stages {
        stage('Install System Dependencies') {
            steps {
                // No root on controller: skip apt-get
                echo 'Skipping apt-get (no root). Using Dockerized tools instead.'
            }
        }

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Install Node Modules') {
            steps {
                // Run npm inside a Node container
                sh '''
                  set -e
                  docker run --rm \
                    -v "$PWD":/app -w /app \
                    node:20-alpine \
                    sh -lc 'node -v && npm -v && (npm ci || npm install)'
                '''
            }
        }

        stage('Run Unit Tests') {
            steps {
                // Run tests inside the same Node container
                sh '''
                  set -e
                  docker run --rm \
                    -v "$PWD":/app -w /app \
                    node:20-alpine \
                    sh -lc 'if npm run | grep -q "^ *test"; then npm test; else echo "No test script defined. Skipping tests."; fi'
                '''
            }
        }

        stage('Security Scan') {
            steps {
                // Use Snyk’s Docker image; requires a Snyk token credential you already added
                withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
                    sh '''
                      set -e
                      docker run --rm \
                        -e SNYK_TOKEN="$SNYK_TOKEN" \
                        -v "$PWD":/app -w /app \
                        snyk/snyk:alpine \
                        snyk test --severity-threshold=high || true
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t premoli126/node-app:latest .'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                sh '''
                  echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin
                  docker push premoli126/node-app:latest
                '''
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: '**/test-results/*.xml', allowEmptyArchive: true
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check logs for details.'
        }
    }
}
