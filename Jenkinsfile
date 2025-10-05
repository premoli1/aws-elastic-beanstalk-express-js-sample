pipeline {
  agent any
  options { timestamps() }

  stages {
    stage('Checkout') {
      steps { cleanWs(); checkout scm }
    }
    stage('List workspace') {
      steps {
        sh '''
          set -e
          echo "== TOP LEVEL =="; ls -la
          echo "== package.json? =="; ls -l package.json || true
        '''
      }
    }
  }

  post { always { cleanWs() } }
}
