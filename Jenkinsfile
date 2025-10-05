pipeline {
  agent any

  environment {
    IMAGE_NAME = "premoli126/node-app"
    IMAGE_TAG  = "build-${env.BUILD_NUMBER}"
  }

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '10'))
  }

  stages {

    stage('Checkout Code') {
      steps {
        cleanWs()
        checkout scm
      }
    }

    // Detect cert path, resolve DinD IP, and prepare Docker env file
    stage('Detect Docker TLS & sanity') {
      steps {
        sh '''#!/usr/bin/env bash
set -e

# 1) Find client certs inside Jenkins container
if [ -f /certs/client/cert.pem ] && [ -f /certs/client/key.pem ] && [ -f /certs/client/ca.pem ]; then
  CERT_DIR="/certs/client"
elif [ -f /certs/client/client/cert.pem ] && [ -f /certs/client/client/key.pem ] && [ -f /certs/client/client/ca.pem ]; then
  CERT_DIR="/certs/client/client"
else
  echo "ERROR: Could not find client certs at /certs/client or /certs/client/client"
  ls -l /certs || true; ls -l /certs/client || true; ls -l /certs/client/client || true
  exit 1
fi
echo "Using DOCKER_CERT_PATH=$CERT_DIR"

# 2) Resolve DinD container IP (service name from your docker-compose is 'dind')
DIND_IP="$(getent hosts dind | awk '{print $1}' | head -n1)"
[ -n "$DIND_IP" ] || { echo "ERROR: Could not resolve IP for 'dind'"; exit 1; }
echo "Detected DinD IP: $DIND_IP"

# 3) Write env file used by later stages (use IP, but pin TLS server name to 'docker' which is in cert SANs)
cat > docker-env.sh <<EOF
export DOCKER_HOST=tcp://$DIND_IP:2376
export DOCKER_TLS_VERIFY=1
export DOCKER_CERT_PATH=$CERT_DIR
export DOCKER_TLS_SERVER_NAME=docker
EOF

echo "---- docker-env.sh ----"; cat docker-env.sh; echo "-----------------------"

# 4) Sanity check Docker connectivity
. ./docker-env.sh
docker version
'''
      }
    }

    stage('Install Node Modules') {
      steps {
        sh '''#!/usr/bin/env bash
set -e
. ./docker-env.sh
# Use a throwaway Node container to run npm install in the workspace
docker run --rm \
  -v "$PWD":/app -w /app \
  node:20-alpine sh -lc 'node -v && npm -v && (npm ci || npm install)'
echo "✅ npm install done"
'''
      }
    }

    stage('Run Unit Tests') {
      steps {
        sh '''#!/usr/bin/env bash
set -e
. ./docker-env.sh
docker run --rm \
  -v "$PWD":/app -w /app \
  node:20-alpine sh -lc 'if npm run | grep -q "^  test"; then npm test; else echo "No test script defined. Skipping tests."; fi'
'''
      }
      post {
        always {
          // No junit produced by the sample app; keep allowEmptyResults to avoid noise
          junit allowEmptyResults: true, testResults: '**/junit.xml'
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        sh '''#!/usr/bin/env bash
set -e
. ./docker-env.sh
docker build -t "${IMAGE_NAME}:${IMAGE_TAG}" -t "${IMAGE_NAME}:latest" .
'''
      }
    }

    stage('Push to Docker Hub') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
          sh '''#!/usr/bin/env bash
set -e
. ./docker-env.sh
echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
docker push "${IMAGE_NAME}:${IMAGE_TAG}"
docker push "${IMAGE_NAME}:latest"
'''
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: '.depcheck/**', allowEmptyArchive: true, fingerprint: true
      cleanWs()
    }
  }
}
