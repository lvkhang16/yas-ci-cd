pipeline {
  agent any

  environment {
      DOCKERHUB_USER = 'lvkhang16'
      // List of services you're building
      // Check your actual folder names in the YAS repo
      SERVICES = 'tax product'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Get Commit ID') {
      steps {
        script {
          env.COMMIT_ID = sh(
            script: 'git rev-parse --short HEAD',
            returnStdout: true
          ).trim()
          echo "Building with tag: ${env.COMMIT_ID}"
        }
      }
    }

    stage('Build and Push Images') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-creds',
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
        )]) {
          sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'

          script {
            def services = env.SERVICES.split()
            for (service in services) {
              sh """
                docker build -t ${DOCKERHUB_USER}/${service}:${COMMIT_ID} -f ./${service}/Dockerfile .
                docker push ${DOCKERHUB_USER}/${service}:${COMMIT_ID}
              """

              if (env.BRANCH_NAME == 'main') {
                sh """
                  docker tag ${DOCKERHUB_USER}/${service}:${COMMIT_ID} ${DOCKERHUB_USER}/${service}:latest
                  docker push ${DOCKERHUB_USER}/${service}:latest
                """
              }
            }
          }
        }
      }
    }
  }

  post {
    always {
      sh 'docker logout'
    }

    success {
      echo "Images pushed with tag: ${env.COMMIT_ID}"
    }

    failure {
      echo "Build failed"
    }
  }
}