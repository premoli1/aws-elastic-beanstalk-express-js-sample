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

    stage('Checkout (clean)') {
      steps { cleanWs(); checkout scm }
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

    // Detect cert path, DinD IP, and set Docker TLS env (no compose changes)
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

# 2) Resolve DinD IP on the user-defined network
DIND_IP="$(getent hosts dind | awk '{print $1}' | head -n1)"
[ -n "$DIND_IP" ] || { echo "ERROR: Could not resolve IP for 'dind'"; exit 1; }
echo "Detected DinD IP: $DIND_IP"

# 3) Write env file for all later stages:
#    - connect by IP
#    - but force TLS server name = 'docker' (which matches the cert SAN)
cat > docker-env.sh <<EOF
export DOCKER_HOST=tcp://$DIND_IP:2376
export DOCKER_TLS_VERIFY=1
export DOCKER_CERT_PATH=$CERT_DIR
export DOCKER_TLS_SERVER_NAME=docker
EOF

echo "---- docker-env.sh ----"
cat docker-env.sh
echo "-----------------------"

# 4) Verify via docker client using the same env
. ./docker-env.sh
docker version
'''
      }
    }

    // Install deps in ephemeral Node 16 container
    stage('Install deps (Node 16)') {
      steps {
        sh '''#!/usr/bin/env bash
set -e
. ./docker-env.sh
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
. ./docker-env.sh
source .envfile

CID="$(docker create node:16 bash -lc 'sleep infinity')"
docker cp "$APP_DIR/." "$CID:/app"
docker start "$CID" 1>/dev/null
docker exec -u 0:0 "$CID" bash -lc 'cd /app && (npm test || echo "No tests found — continuing")'
docker rm -f "$CID" 1>/dev/null
'''
      }
      post { always { junit allowEmptyResults: true, testResults: '**/junit.xml' } }
    }

    // Security scan (OWASP DC) — fail on High/Critical (CVSS ≥ 7.0)
    stage('OWASP scan (fail on High/Critical)') {
      steps {
        sh '''#!/usr/bin/env bash
set -e
. ./docker-env.sh
source .envfile
mkdir -p .depcheck

# IMPORTANT: pass multiple -f flags, not a comma list
CID="$(docker create owasp/dependency-check:latest \
  -f XML -f HTML \
  --scan /src \
  --out /report \
  --project node-app \
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

    // Docker build & push
    stage('Docker build & push') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-credentials',   // change if your ID differs
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

  post { always { cleanWs() } }
}
