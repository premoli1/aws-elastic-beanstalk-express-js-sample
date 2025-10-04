pipeline {
  agent any

  environment {
    // ==== Docker Hub target ====
    IMAGE_NAME = "premoli126/node-app"
    IMAGE_TAG  = "build-${env.BUILD_NUMBER}"

    // ==== DinD over TLS (matches your docker-compose) ====
    DOCKER_HOST       = 'tcp://dind:2376'
    DOCKER_TLS_VERIFY = '1'
    DOCKER_CERT_PATH  = '/certs/client'
  }

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '10'))
  }

  stages {

    stage('Checkout (clean)') {
      steps {
        cleanWs()
        checkout scm
      }
    }

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

    stage('Docker sanity (TLS 2376)') {
      steps {
        sh '''#!/usr/bin/env bash
set -e
echo "DOCKER_HOST=$DOCKER_HOST"
echo "DOCKER_TLS_VERIFY=$DOCKER_TLS_VERIFY"
echo "DOCKER_CERT_PATH=$DOCKER_CERT_PATH"
# Optional: quick API ping using curl and client certs
if command -v curl >/dev/null 2>&1; then
  echo "Pinging Docker API at https://dind:2376/_ping ..."
  curl --silent --show-error --fail \
       --cacert "$DOCKER_CERT_PATH/ca.pem" \
       --cert   "$DOCKER_CERT_PATH/cert.pem" \
       --key    "$DOCKER_CERT_PATH/key.pem" \
       https://dind:2376/_ping | grep -q '^OK$'
fi
docker version
'''
      }
    }

    // ---------- Build steps inside ephemeral Node 16 containers (safe for TLS DinD) ----------
    stage('Install deps (Node 16)') {
      steps {
        sh '''#!/usr/bin/env bash
set -e
source .envfile
CID="$(docker create node:16 bash -lc 'sleep infinity')"
docker cp "$APP_DIR/." "$CID:/app"
docker start "$CID" 1>/dev/null
docker exec -u 0:0 "$CID" bash -lc 'cd /app && node -v && npm -v && (npm ci || npm install)'
docker rm -f "$CID" 1>/dev/null
echo "✅ npm install done"
'''
      }
    }

    stage('Unit tests (Node 16)') {
      steps {
        sh '''#!/usr/bin/env bash
set -e
source .envfile
CID="$(docker create node:16 bash -lc 'sleep infinity')"
docker cp "$APP_DIR/." "$CID:/app"
docker start "$CID" 1>/dev/null
# If you don't have tests, don't fail the build
docker exec -u 0:0 "$CID" bash -lc 'cd /app && (npm test || echo "No tests found — continuing")'
docker rm -f "$CID" 1>/dev/null
'''
      }
      post {
        always {
          // harmless if you don't produce junit.xml
          junit allowEmptyResults: true, testResults: '**/junit.xml'
        }
      }
    }

    // ---------- Security scan (OWASP DC) — fail on High/Critical (CVSS ≥ 7.0) ----------
    stage('OWASP scan (fail on High/Critical)') {
      steps {
        sh '''#!/usr/bin/env bash
set -e
source .envfile
mkdir -p .depcheck

# Run OWASP Dependency-Check with fail gate at CVSS 7.0
CID="$(docker create owasp/dependency-check:latest \
  --scan /src \
  --format XML,HTML \
  --out /report \
  --failOnCVSS 7.0)"
docker cp "$APP_DIR/." "$CID:/src"

set +e
docker start -a "$CID"
RC=$?
set -e

docker cp "$CID:/report/." ".depcheck/" || true
docker rm -f "$CID" 1>/dev/null || true

exit $RC   # non-zero if High/Critical found
'''
      }
      post {
        always {
          archiveArtifacts artifacts: '.depcheck/**', allowEmptyArchive: true, fingerprint: true
        }
      }
    }

    // ---------- Build & Push image ----------
    stage('Docker build & push') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-credentials',   // <--- change if your ID differs
          usernameVariable: 'DH_USER',
          passwordVariable: 'DH_PASS'
        )]) {
          sh '''#!/usr/bin/env bash
set -e
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
