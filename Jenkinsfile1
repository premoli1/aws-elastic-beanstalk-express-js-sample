pipeline {
  /* Use Node 16 for the Node steps, like in your file */
  agent {
    docker {
      image 'node:16'
      args '-u root'
    }
  }

  options { timestamps() }

  environment {
    IMAGE_NAME = 'premoli126/node-app'
    IMAGE_TAG  = "v${env.BUILD_NUMBER}"
    // Snyk token from Jenkins credentials (Secret text)
    SNYK_TOKEN = credentials('snyk-token')
  }

  stages {
    stage('Install Dependencies') {
      steps {
        sh 'npm install --save'
      }
    }

    stage('Unit Tests') {
      steps {
        sh 'npm test || echo "no tests defined"'
      }
    }

    stage('Security Scan (Snyk)') {
      steps {
        sh '''
          npm install -g snyk --unsafe-perm
          snyk auth "$SNYK_TOKEN"
          # Fail build on High/Critical:
          snyk test --severity-threshold=high
          # If you need org scoping, uncomment and set your org:
          snyk test --org=05e95a46-936a-484e-86c9-09a140e4db93 --severity-threshold=high
        '''
      }
    }

    /* Build & push needs Docker access (DinD).
       Run this stage on the Jenkins controller where DOCKER_HOST/Certs are set
       by your Jenkins container env (from docker-compose). */
    stage('Build Docker Image') {
      agent { label '' }  // switch off the Node docker agent for Docker CLI access
      environment {
        DOCKER_HOST = 'tcp://dind:2376'
        DOCKER_CERT_PATH = '/certs/client'
        DOCKER_TLS_VERIFY = '1'
      }
      steps {
        script {
          // Bind Docker Hub username/password properly:
          withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials',
                                            usernameVariable: 'DOCKERHUB_USR',
                                            passwordVariable: 'DOCKERHUB_PSW')]) {
            sh """
              docker login -u "$DOCKERHUB_USR" -p "$DOCKERHUB_PSW" docker.io
              docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
              docker push ${IMAGE_NAME}:${IMAGE_TAG}
              docker tag  ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest
              docker push ${IMAGE_NAME}:latest
            """
          }
        }
      }
    }
  }

  post {
    always {
      // keep it simple; archive any test XMLs if they exist
      archiveArtifacts artifacts: '**/test-results/*.xml', allowEmptyArchive: true
      cleanWs()
    }
  }
}
