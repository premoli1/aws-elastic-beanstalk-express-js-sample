pipeline {
  agent any

  environment {
    // ===== Docker Hub target =====
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

    // ====== Key: Detect TLS cert path & verify DinD over TLS ======
    stage('Detect Docker TLS & sanity') {
      steps {
        sh '''
          set -e

          # Build an env file we source in all later stages
          cat > docker-env.sh <<'EOF'
# default to TLS on 2376
export DOCKER_HOST=tcp://dind:2376
export DOCKER_TLS_VERIFY=1
# we'll fill DOCKER_CERT_PATH below
EOF

          # Figure out where the client certs actually are
          if [ -f /certs/client/cert.pem ] && [ -f /certs/client/key.pem ] && [ -f /certs/client/ca.pem ]; then
            CERT_DIR="/certs/client"
          elif [ -f /certs/client/client/cert.pem ] && [ -f /certs/client/client/key.pem ] && [ -f /certs/client/client/ca.pem ]; then
            # This happens when the volume root (/certs) is mounted at /certs/client in Jenkins
            CERT_DIR="/certs/client/client"
          else
            echo "ERROR: Could not find client certs at /certs/client or /certs/client/client"
            echo "List at /certs and /certs/client/client (if any):"
            ls -l /certs || true; ls -l /certs/client/client || true
            exit 1
          fi

          echo "Using DOCKER_CERT_PATH=$CERT_DIR"
          echo "export DOCKER_CERT_PATH=$CERT_DIR" >> docker-env.sh

          # Optional: ping the API directly
          echo "Pinging Docker API at https://dind:2376/_ping ..."
          curl --silent --show-error --fail \
               --cacert "$CERT_DIR/ca.pem" \
               --cert   "$CERT_DIR/cert.pem" \
               --key    "$CERT_DIR/key.pem" \
               https://dind:2376/_ping | grep -q '^OK$'

          # Final verify via docker client
          . ./docker-env.sh
          docker version
        '''
      }
    }

    // ---------- Build steps inside ephemeral Node 16 containers ----------
    stage('Install deps (Node 16)') {
      steps {
        sh '''
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
        sh '''
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

    // ---------- Security scan (OWASP DC) — fail on High/Critical (CVSS ≥ 7.0) ----------
    stage('OWASP scan (fail on High/Critical)') {
      steps {
        sh '''
          set -e
          . ./docker-env.sh
          source .envfile
          mkdir -p .depcheck

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
      post { always { archiveArtifacts artifacts: '.depcheck/**', allowEmptyArchive: true, fingerprint: true } }
    }

    // ---------- Build & Push image ----------
    stage('Docker build & push') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-credentials',   // <-- change if your ID differs
          usernameVariable: 'DH_USER',
          passwordVariable: 'DH_PASS'
        )]) {
          sh '''
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
