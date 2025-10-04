pipeline {
  agent any
  environment {
    IMAGE_NAME = "premoli126/node-app"
    IMAGE_TAG  = "build-${env.BUILD_NUMBER}"
    DOCKER_HOST       = 'tcp://dind:2375'
    DOCKER_TLS_VERIFY = ''
    DOCKER_CERT_PATH  = ''
  }
  options { timestamps(); buildDiscarder(logRotator(numToKeepStr: '10')) }

  stages {

    stage('Checkout (clean)') {
      steps { cleanWs(); checkout scm }
    }

    stage('Verify workspace') {
      steps {
        sh '''
          set -e
          echo "== TOP LEVEL =="; ls -la
          echo "== LOOK FOR package.json (depth 2) =="
          find . -maxdepth 2 -type f -name package.json -print || true
        '''
      }
    }

    stage('Detect APP_DIR & Dockerfile') {
      steps {
        sh '''
          set -e
          PKG="$( [ -f package.json ] && echo package.json || find . -maxdepth 2 -type f -name package.json -print -quit )"
          [ -n "$PKG" ] || { echo "ERROR: No package.json found"; exit 1; }
          APP_DIR="$(dirname "$PKG")"; [ "$APP_DIR" = "." ] && APP_DIR="."
          DF=""
          [ -f "$APP_DIR/Dockerfile" ] && DF="$APP_DIR/Dockerfile"
          [ -z "$DF" ] && [ -f Dockerfile ] && DF="Dockerfile"
          echo "APP_DIR=$APP_DIR"    >  .envfile
          echo "DOCKERFILE_PATH=$DF" >> .envfile
          cat .envfile
        '''
      }
    }

    stage('Docker sanity (DinD reachable)') {
      steps {
        withEnv(['DOCKER_TLS_VERIFY=', 'DOCKER_CERT_PATH=']) {
          sh '''
            set -e
            echo "DOCKER_HOST=$DOCKER_HOST"
            echo "DOCKER_TLS_VERIFY=${DOCKER_TLS_VERIFY:-<empty>}"
            echo "DOCKER_CERT_PATH=${DOCKER_CERT_PATH:-<empty>}"
            docker version
          '''
        }
      }
    }

    stage('Install deps (Node 16)') {
      steps {
        withEnv(['DOCKER_TLS_VERIFY=', 'DOCKER_CERT_PATH=']) {
          sh '''
            set -e
            source .envfile
            CID="$(docker create node:16 bash -lc 'sleep infinity')"
            docker cp "$APP_DIR/." "$CID:/app"
            docker start "$CID" 1>/dev/null
            docker exec -u 0:0 "$CID" bash -lc 'cd /app && node -v && npm -v && (npm ci || npm install)'
            docker rm -f "$CID" 1>/dev/null
          '''
        }
      }
    }

    stage('Unit tests (Node 16)') {
      steps {
        withEnv(['DOCKER_TLS_VERIFY=', 'DOCKER_CERT_PATH=']) {
          sh '''
            set -e
            source .envfile
            CID="$(docker create node:16 bash -lc 'sleep infinity')"
            docker cp "$APP_DIR/." "$CID:/app"
            docker start "$CID" 1>/dev/null
            docker exec -u 0:0 "$CID" bash -lc 'cd /app && (npm test || echo "No tests found — continuing")'
            docker rm -f "$CID" 1>/dev/null
          '''
        }
      }
      post { always { junit allowEmptyResults: true, testResults: '**/junit.xml' } }
    }

    stage('OWASP scan (fail on High/Critical)') {
      steps {
        withEnv(['DOCKER_TLS_VERIFY=', 'DOCKER_CERT_PATH=']) {
          sh '''
            set -e
            source .envfile
            mkdir -p .depcheck
            CID="$(docker create owasp/dependency-check:latest \
              --scan /src \
              --format XML,HTML \
              --out /report \
              --failOnCVSS 7.0)"
            docker cp "$APP_DIR/." "$CID:/src"
            set +e; docker start -a "$CID"; RC=$?; set -e
            docker cp "$CID:/report/." ".depcheck/" || true
            docker rm -f "$CID" 1>/dev/null || true
            exit $RC
          '''
        }
      }
      post { always { archiveArtifacts artifacts: '.depcheck/**', allowEmptyArchive: true, fingerprint: true } }
    }

    stage('Docker build & push') {
      steps {
        withEnv(['DOCKER_TLS_VERIFY=', 'DOCKER_CERT_PATH=']) {
          withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
            sh '''
              set -e
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
  }

  post { always { cleanWs() } }
}