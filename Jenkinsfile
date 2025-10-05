pipeline {
  agent any

  environment {
    IMAGE_NAME = "premoli126/node-app"
    IMAGE_TAG  = "build-${env.BUILD_NUMBER}"

    // Docker Hub (Username/Password credentials with ID below)
    DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')

    // Optional Snyk token (Secret text). If not present, Snyk stage skips.
    SNYK_TOKEN = credentials('snyk-token')
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

    stage('Detect Docker TLS & IP') {
      steps {
        sh '''#!/bin/sh
set -eu

echo "== Check cert locations =="
ls -l /certs || true
ls -l /certs/client || true
ls -l /certs/client/client || true

# Find client certs in mounted volume
CERT_DIR="/certs/client/client"
if [ ! -f "$CERT_DIR/cert.pem" ] || [ ! -f "$CERT_DIR/key.pem" ] || [ ! -f "$CERT_DIR/ca.pem" ]; then
  CERT_DIR="/certs/client"
fi
if [ ! -f "$CERT_DIR/cert.pem" ] || [ ! -f "$CERT_DIR/key.pem" ] || [ ! -f "$CERT_DIR/ca.pem" ]; then
  echo "ERROR: Could not find Docker client certs at /certs/client or /certs/client/client"
  exit 2
fi
echo "Using DOCKER_CERT_PATH=$CERT_DIR"

# Resolve DinD IP (compose service name 'dind')
echo "== Resolve dind =="
DIND_IP="$(getent hosts dind | awk '{print $1}' | head -n1 || true)"
if [ -z "$DIND_IP" ]; then
  echo "ERROR: Could not resolve IP for 'dind' on the 'ci' network."
  exit 2
fi

# Env for later stages
cat > docker-env.sh <<EOF
export DOCKER_HOST=tcp://$DIND_IP:2376
export DOCKER_TLS_VERIFY=1
export DOCKER_CERT_PATH=$CERT_DIR
export DOCKER_TLS_SERVER_NAME=docker
EOF

echo "---- docker-env.sh ----"
cat docker-env.sh
echo "-----------------------"

# Verify daemon connectivity
. ./docker-env.sh
docker version
'''
      }
    }

    stage('Install Node Modules') {
      steps {
        sh '''#!/bin/sh
set -eu
. ./docker-env.sh

docker run --rm \
  -e "npm_config_loglevel=warn" \
  -v "$PWD":/app -w /app \
  node:20-alpine \
  sh -lc 'node -v && npm -v && (npm ci || npm install)'
'''
      }
    }

    stage('Run Unit Tests') {
      steps {
        sh '''#!/bin/sh
set -eu
. ./docker-env.sh

HAS_TEST="$(node -e "try{console.log(!!require('./package.json').scripts.test)}catch(e){console.log(false)}")"
if [ "$HAS_TEST" = "true" ]; then
  docker run --rm -v "$PWD":/app -w /app node:20-alpine sh -lc 'npm test'
else
  echo "No test script defined. Skipping tests."
fi
'''
      }
      post {
        always {
          // harmless if no reports exist
          junit allowEmptyResults: true, testResults: '**/junit*.xml'
        }
      }
    }

    stage('Security Scan (Snyk - optional)') {
      when { expression { return env.SNYK_TOKEN && env.SNYK_TOKEN.trim() } }
      steps {
        sh '''#!/bin/sh
set -eu
. ./docker-env.sh

docker run --rm \
  -e SNYK_TOKEN="$SNYK_TOKEN" \
  -v "$PWD":/app -w /app \
  snyk/snyk-cli:stable \
  snyk test --org=premoli1 --severity-threshold=high || true
'''
      }
    }

    stage('Build Docker Image') {
      steps {
        sh '''#!/bin/sh
set -eu
. ./docker-env.sh

docker build -t "${IMAGE_NAME}:${IMAGE_TAG}" .
docker tag  "${IMAGE_NAME}:${IMAGE_TAG}" "${IMAGE_NAME}:latest"
'''
      }
    }

    stage('Push to Docker Hub') {
      steps {
        sh '''#!/bin/sh
set -eu
. ./docker-env.sh

echo "$DOCKERHUB_CREDENTIALS_PSW" | docker login -u "$DOCKERHUB_CREDENTIALS_USR" --password-stdin
docker push "${IMAGE_NAME}:${IMAGE_TAG}"
docker push "${IMAGE_NAME}:latest"
'''
      }
    }

    // Final stage so workspace steps run IN an agent context
    stage('Finalize / Cleanup') {
      steps {
        echo 'Finalizing...'
      }
      post {
        always {
          // If you produce artifacts elsewhere, archive here (inside a stage).
          // archiveArtifacts artifacts: '.depcheck/**', allowEmptyArchive: true, fingerprint: true
          cleanWs()
        }
      }
    }
  }
}
