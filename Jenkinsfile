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
      steps {
        cleanWs()
        checkout scm
      }
    }

    stage('Detect app & Dockerfile') {
      steps {
        sh '''
          set -eu
          PKG="$( [ -f package.json ] && echo package.json || find . -maxdepth 2 -type f -name package.json -print -quit )"
          [ -n "$PKG" ] || { echo "No package.json found"; exit 1; }
          APP_DIR="$(dirname "$PKG")"; [ "$APP_DIR" = "." ] && APP_DIR="."
          DF=""
          [ -f "$APP_DIR/Dockerfile" ] && DF="$APP_DIR/Dockerfile"
          [ -z "$DF" ] && [ -f Dockerfile ] && DF="Dockerfile"
          [ -n "$DF" ] || { echo "No Dockerfile found in $APP_DIR or repo root"; exit 1; }
          printf "APP_DIR=%s\nDOCKERFILE_PATH=%s\n" "$APP_DIR" "$DF" > .envfile
          cat .envfile
        '''
      }
    }

    stage('Detect Docker TLS & IP') {
      steps {
        sh '''
          set -eu

          # Find client certs (compose can mount either layout)
          if [ -f /certs/client/cert.pem ] && [ -f /certs/client/key.pem ] && [ -f /certs/client/ca.pem ]; then
            CERT_DIR="/certs/client"
          elif [ -f /certs/client/client/cert.pem ] && [ -f /certs/client/client/key.pem ] && [ -f /certs/client/client/ca.pem ]; then
            CERT_DIR="/certs/client/client"
          else
            echo "ERROR: Docker client certs not found under /certs/client[/client]"
            ls -l /certs || true; ls -l /certs/client || true; ls -l /certs/client/client || true
            exit 1
          fi

          # Resolve DinD IP on the user-defined network; fallback to hostname
          DIND_IP="$(getent hosts dind | awk '{print $1}' | head -n1 || true)"
          [ -n "$DIND_IP" ] || DIND_IP="dind"

          cat > docker-env.sh <<EOF
export DOCKER_HOST=tcp://$DIND_IP:2376
export DOCKER_TLS_VERIFY=1
export DOCKER_CERT_PATH=$CERT_DIR
export DOCKER_TLS_SERVER_NAME=docker
EOF

          echo "---- docker-env.sh ----"
          cat docker-env.sh
          echo "-----------------------"

          # Sanity
          . ./docker-env.sh
          docker version
        '''
      }
    }

    stage('Install Node modules (Dockerized)') {
      steps {
        sh '''
          set -eu
          . ./docker-env.sh
          . ./.envfile

          # Run Node inside a container; bind-mount workspace
          docker run --rm \
            -v "$PWD":/app -w /app \
            node:20-alpine sh -lc "node -v && npm -v && (npm ci || npm install)"
          echo "✅ npm install done"
        '''
      }
    }

    stage('Unit tests (Dockerized)') {
      steps {
        sh '''
          set -eu
          . ./docker-env.sh

          if grep -q '"test"[[:space:]]*:' package.json; then
            echo "Running tests…"
            docker run --rm \
              -v "$PWD":/app -w /app \
              node:20-alpine sh -lc "npm test"
          else
            echo "No test script defined. Skipping tests."
          fi
        '''
      }
      post {
        always {
          junit allowEmptyResults: true, testResults: '**/junit.xml'
        }
      }
    }

    stage('Quick security check (npm audit)') {
      steps {
        sh '''
          set -eu
          . ./docker-env.sh
          docker run --rm \
            -v "$PWD":/app -w /app \
            node:20-alpine sh -lc "npm audit --audit-level=high || true"
        '''
      }
    }

    stage('Docker build & push') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-credentials',
          usernameVariable: 'DH_USER',
          passwordVariable: 'DH_PASS'
        )]) {
          sh '''
            set -eu
            . ./docker-env.sh
            . ./.envfile

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
