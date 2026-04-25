pipeline {
  agent any

  environment {
      DOCKERHUB_USER = 'lvkhang16'
      // List of services you're building
      // Check your actual folder names in the YAS repo
      SERVICES = 'tax product'
      CHANGED_SERVICES = ''
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

    stage('Detect Changed Services') {
      steps {
        script {
          def allServices = (env.SERVICES ?: '').tokenize(' ')
          def changedOutput = sh(
            script: '''
              set -e
              if [ -n "$GIT_PREVIOUS_SUCCESSFUL_COMMIT" ] && git rev-parse --verify "$GIT_PREVIOUS_SUCCESSFUL_COMMIT" >/dev/null 2>&1; then
                git diff --name-only "$GIT_PREVIOUS_SUCCESSFUL_COMMIT" HEAD
              elif git rev-parse --verify HEAD~1 >/dev/null 2>&1; then
                git diff --name-only HEAD~1 HEAD
              else
                git ls-tree --name-only -r HEAD
              fi
            ''',
            returnStdout: true
          ).trim()

          def changedFiles = []
          if (changedOutput) {
            for (line in changedOutput.split('\n')) {
              def filePath = line.trim()
              if (filePath) {
                changedFiles << filePath
              }
            }
          }

          def changedServices = []
          for (service in allServices) {
            boolean serviceChanged = false
            for (filePath in changedFiles) {
              if (filePath == service || filePath.startsWith("${service}/")) {
                serviceChanged = true
                break
              }
            }
            if (serviceChanged) {
              changedServices << service
            }
          }
          def changedServicesValue = changedServices ? changedServices.join(' ') : ''

          env.CHANGED_SERVICES = changedServicesValue
          echo "Changed files: ${changedFiles}"

          if (changedServices.size() > 0) {
            echo "Services to build: ${changedServicesValue}"
          } else {
            echo 'No matching service changes found in SERVICES list.'
          }
        }
      }
    }

    stage('Build and Push Images') {
      when {
        expression {
          return env.CHANGED_SERVICES?.trim()
        }
      }
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-creds',
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
        )]) {
          sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'

          script {
            def services = (env.CHANGED_SERVICES ?: '').tokenize(' ')
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
      sh 'docker logout || true'
    }

    success {
      script {
        if (env.CHANGED_SERVICES?.trim()) {
          echo "Images pushed with tag: ${env.COMMIT_ID}"
          echo "Built services: ${env.CHANGED_SERVICES}"
        } else {
          echo 'No listed service changes detected. Skipped image build and push.'
        }
      }
    }

    failure {
      echo "Build failed"
    }
  }
}