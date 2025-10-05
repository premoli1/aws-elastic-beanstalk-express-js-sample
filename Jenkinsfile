pipeline {
  agent any
  options { timestamps() }

  stages {
    stage('Detect Docker TLS & IP') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail

echo "== Check cert locations =="
ls -l /certs || true
ls -l /certs/client || true
ls -l /certs/client/client || true

# Find client certs
if [[ -f /certs/client/cert.pem && -f /certs/client/key.pem && -f /certs/client/ca.pem ]]; then
  CERT_DIR="/certs/client"
elif [[ -f /certs/client/client/cert.pem && -f /certs/client/client/key.pem && -f /certs/client/client/ca.pem ]]; then
  CERT_DIR="/certs/client/client"
else
  echo "ERROR: Could not find TLS client certs at /certs/client or /certs/client/client"
  exit 1
fi
echo "Using DOCKER_CERT_PATH=$CERT_DIR"

echo "== Resolve dind =="
getent hosts dind || true
DIND_IP="$(getent hosts dind | awk '{print $1}' | head -n1)"
if [[ -z "$DIND_IP" ]]; then
  echo "ERROR: Could not resolve IP for 'dind'"
  exit 1
fi
echo "Detected DinD IP: $DIND_IP"

# Write env file for later steps
cat > docker-env.sh <<EOF
export DOCKER_HOST=tcp://$DIND_IP:2376
export DOCKER_TLS_VERIFY=1
export DOCKER_CERT_PATH=$CERT_DIR
export DOCKER_TLS_SERVER_NAME=docker
set -x
EOF

echo "---- docker-env.sh ----"
cat docker-env.sh
echo "-----------------------"
'''
      }
    }
  }

  post { always { cleanWs() } }
}
