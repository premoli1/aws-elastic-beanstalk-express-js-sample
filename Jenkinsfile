pipeline {
  agent any

  environment {
    // ---- your Docker Hub target ----
    IMAGE_NAME = "premoli126/node-app"
    IMAGE_TAG  = "build-${env.BUILD_NUMBER}"
  }

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '10'))
  }

  stages {
    // 1) Clean checkout
    stage('Checkout (clean)') {
      steps {
        cleanWs()
        checkout scm
      }
    }

    // 2) Quick sanity of repo contents
    stage('Verify workspace') {
      steps {
        sh '''#!/usr/bin/env bash
set -e
echo "== TOP LEVEL =="; ls -la
echo "== LOOK FOR package.json (depth 2) =="
find . -maxdepth 2 -type f -name package.json -print || true
'''
      }
    }

    // 3) Detect app dir + Dockerfile path
    stage('Detect APP_DIR & Dockerfile') {
      steps {
        sh '''#!/usr/bin/env bash
set -e
PKG="$( [ -f package.json ] && echo package.json || find . -maxdepth 2 -type f -name package.json -print -quit )"
[ -n "$PKG" ] || { echo "ERROR: No package.json found in repo root or one-level subfolders"; exit 1; }
APP_DIR="$(dirname "$PKG")"; [ "$APP_DIR" = "." ] && APP_DIR="."
DF=""
[ -f "$APP_DIR/Dockerfile" ] && DF="$APP_DIR/Dockerfile"
[ -z "$DF" ] && [ -f Dockerfile ] && DF="Dockerfile"
echo "APP_DIR=$APP_DIR"     >  .envfile
echo "DOCKERFILE_PATH=$DF"  >> .envfile
cat .envfile
'''
      }
    }

    // 4) Detect Docker TLS certs & pick DinD endpoint (IP or hostname), write docker-env.sh
    stage('Detect Docker TLS & IP') {
      steps {
        sh '''#!/usr/bin/env bash
set -eo pipefail

echo "== Check cert locations =="
ls -l /certs || true
ls -l /certs/client || true
ls -l /certs/client/client || true

# Locate client certs
if [[ -f /certs/client/cert.pem && -f /certs/client/key.pem && -f /certs/client/ca.pem ]]; then
  CERT_DIR="/certs/client"
elif [[ -f /certs/client/client/cert.pem && -f /certs/client/client/key.pem && -f /certs/client/client/ca.pem ]]; then
  CERT_DIR="/certs/client/client"
else
  echo "ERROR: Could not find TLS client certs at /certs/client or /certs/client/client"
  exit 1
fi
echo "Using DOCKER_CERT_PATH=$CERT_DIR"

# Try to resolve DinD IP if getent exists; otherwise fall back to hostname 'docker'
echo "== Resolve DinD =="
DIND_IP=""
if command -v getent >/dev/null 2>&1; then
  DIND_IP="$(getent hosts dind | awk '{print $1}' | head -n1 || true)"
fi

if [[ -n "$DIND_IP" ]]; then
  DOCKER_HOST_VALUE="tcp://${DIND_IP}:2376"
  echo "Resolved dind to IP: ${DIND_IP}"
else
  DOCKER_HOST_VALUE="tcp://docker:2376"
  echo "Could not resolve 'dind' via getent; falling back to hostname 'docker'"
fi

# Write env file used by later stages
cat > docker-env.sh <<EOF
export DOCKER_HOST=${DOCKER_HOST_VALUE}
export DOCKER_TLS_VERIFY=1
export DOCKER_CERT_PATH=${CERT_DIR}
export DOCKER_TLS_SERVER_NAME=docker
set -x
EOF

echo "---- docker-env.sh ----"
cat docker-env.sh
echo "-----------------------"

# Quick client/daemon handshake check
. ./docker-env.sh
docker version
'''
      }
    }

    // 5) Install node modules using a throwaway Node container (fast, no root needed)
    stage('Install Node modules') {
      steps {
        sh '''#!/usr/bin/env bash
set -e
. ./docker-env.sh
source .envfile

# Run npm install inside a container and mount workspace
docker run --rm \
  -v "$WORKSPACE/$APP_DIR":/app -w /app \
  node:20-alpine \
  sh -lc 'node -v && npm -v && (npm ci || npm install)'
echo "✅ npm install done"
'''
      }
    }

    // 6) Unit tests (skip gracefully if no test script)
    stage('Unit tests') {
      steps {
        sh '''#!/usr/bin/env bash
set -e
. ./docker-env.sh
source .envfile

docker run --rm \
  -v "$WORKSPACE/$APP_DIR":/app -w /app \
  node:20-alpine \
  sh -lc 'if npm run | grep -q "^  test"; then npm test; else echo "No tests defined — skipping"; fi'
'''
      }
      post {
        always {
          // No junit.xml by default — harmless to keep for future
          junit allowEmptyResults: true, testResults: '**/junit.xml'
        }
      }
    }

    // 7) Lightweight security check (fast): npm audit (warn-only on High/Critical)
    stage('Security scan (npm audit high+)') {
      steps {
        sh '''#!/usr/bin/env bash
set -e
. ./docker-env.sh
source .envfile

docker run --rm \
  -v "$WORKSPACE/$APP_DIR":/app -w /app \
  node:20-alpine \
  sh -lc 'npm audit --omit=dev --json --audit-level=high || true' \
  | tee npm-audit.json >/dev/null || true

# Keep the report (if any)
[ -f npm-audit.json ] && mv npm-audit.json "$WORKSPACE/.npm-audit.json" || true
'''
      }
      post {
        always {
          archiveArtifacts artifacts: '.npm-audit.json', allowEmptyArchive: true, fingerprint: true
        }
      }
    }

    // 8) Build & push Docker image
    stage('Docker build & push') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-credentials',   // <-- make sure this ID exists in Jenkins
          usernameVariable: 'DH_USER',
          passwordVariable: 'DH_PASS'
        )]) {
          sh '''#!/usr/bin/env bash
set -e
. ./docker-env.sh
source .envfile
[ -n "$DOCKERFILE_PATH" ] || { echo "ERROR: No Dockerfile in $APP_DIR or repo root"; exit 1; }

echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin

docker build -f "$DOCKERFILE_PATH" -t "${IMAGE_NAME}:${IMAGE_TAG}" "$APP_DIR"
docker push "${IMAGE_NAME}:${IMAGE_TAG}"

docker tag  "${IMAGE_NAME}:${IMAGE_TAG}" "${IMAGE_NAME}:latest"
docker push "${IMAGE_NAME}:latest"
'''
        }
      }
    }
  }

  post {
    always {
      cleanWs()
    }
  }
}
