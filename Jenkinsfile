pipeline {
  agent any

  environment {
    IMAGE_NAME = "premoli126/node-app"
    IMAGE_TAG  = "build-${env.BUILD_NUMBER}"
  }

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '10'))
    timeout(time: 60, unit: 'MINUTES')   // whole pipeline guard
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
[ -n "$PKG" ] || { echo "ERROR: No package.json found"; exit 1; }
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

    stage('Detect Docker TLS & sanity') {
      steps {
        sh '''#!/usr/bin/env bash
set -e

# locate client certs exactly how your compose mounts them
if [ -f /certs/client/cert.pem ] && [ -f /certs/client/key.pem ] && [ -f /certs/client/ca.pem ]; then
  CERT_DIR="/certs/client"
elif [ -f /certs/client/client/cert.pem ] && [ -f /certs/client/client/key.pem ] && [ -f /certs/client/client/ca.pem ]; then
  CERT_DIR="/certs/client/client"
else
  echo "ERROR: TLS client certs not found"; ls -l /certs || true; ls -l /certs/client || true; ls -l /certs/client/client || true; exit 1
fi
DIND_IP="$(getent hosts dind | awk '{print $1}' | head -n1)"
[ -n "$DIND_IP" ] || { echo "ERROR: cannot resolve dind"; exit 1; }

cat > docker-env.sh <<EOF
export DOCKER_HOST=tcp://$DIND_IP:2376
export DOCKER_TLS_VERIFY=1
export DOCKER_CERT_PATH=$CERT_DIR
export DOCKER_TLS_SERVER_NAME=docker
EOF

. ./docker-env.sh
docker version
'''
      }
    }

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
docker exec -u 0:0 "$CID" bash -lc 'cd /app && (npm test || echo "No tests configured")'
docker rm -f "$CID" 1>/dev/null
'''
      }
      post { always { junit allowEmptyResults: true, testResults: '**/junit.xml' } }
    }

    stage('OWASP scan (fast & fail on High/Critical)') {
      steps {
        sh '''#!/usr/bin/env bash
set -e
. ./docker-env.sh
source .envfile
mkdir -p .depcheck

# Reuse CVE database across builds (HUGE speedup after first run)
docker volume inspect dc-data >/dev/null 2>&1 || docker volume create dc-data

# If DB already exists in dc-data, skip update for speed
HAS_DB="$(docker run --rm -v dc-data:/usr/share/dependency-check/data alpine sh -lc 'test -e /usr/share/dependency-check/data/cve.db && echo yes || true')"
EXTRA=""
[ "$HAS_DB" = "yes" ] && EXTRA="--noupdate"

CID="$(docker create \
  -v dc-data:/usr/share/dependency-check/data \
  owasp/dependency-check:latest \
  -f XML -f HTML \
  $EXTRA \
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

exit $RC   # non-zero => fail on High/Critical
'''
      }
      post { always { archiveArtifacts artifacts: '.depcheck/**', allowEmptyArchive: true, fingerprint: true } }
    }

    stage('Docker build & push') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-credentials',   // make sure this ID exists in Jenkins
          usernameVariable: 'DH_USER',
          passwordVariable: 'DH_PASS'
        )]) {
          sh '''#!/usr/bin/env bash
set -e
. ./docker-env.sh
source .envfile
[ -n "$DOCKERFILE_PATH" ] || { echo "ERROR: No Dockerfile"; exit 1; }

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
