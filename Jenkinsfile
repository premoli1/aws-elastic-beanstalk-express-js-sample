pipeline {
    agent any
    
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        DOCKER_HOST = "tcp://docker:2376"
        DOCKER_TLS_VERIFY = "1"
        DOCKER_CERT_PATH = "/certs/client"
    }
    stages {
        stage('Install System Dependencies') {
            steps {
                sh '''
                    apt-get update
                    apt-get install -y git docker.io curl gnupg lsb-release
                '''
            }
        }
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }
        stage('Install Node Modules') {
            steps {
                sh 'npm install --save'
            }
        }
        stage('Run Unit Tests') {
            steps {
                sh '''
                    if npm run | grep -q "test"; then
                        npm test
                    else
                        echo "No test script defined. Skipping tests."
                    fi
                '''
            }
        }
        stage('Security Scan') {
            steps {
                sh 'npm install -g snyk'
                sh 'snyk test --org=premoli1 --severity-threshold=high || true'
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

