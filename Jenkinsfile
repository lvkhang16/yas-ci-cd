def changedServices = []

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

          def detectedServices = []
          for (service in allServices) {
            boolean serviceChanged = false
            for (filePath in changedFiles) {
              if (filePath == service || filePath.startsWith("${service}/")) {
                serviceChanged = true
                break
              }
            }
            if (serviceChanged) {
              detectedServices << service
            }
          }
          changedServices = detectedServices
          def changedServicesValue = changedServices ? changedServices.join(' ') : ''
          echo "Changed files: ${changedFiles}"

          if (changedServices.size() > 0) {
            echo "Services to build: ${changedServicesValue}"
          } else {
            echo 'No matching service changes found in SERVICES list.'
          }
        }
      }
    }

    stage('Build Service JARs') {
      steps {
        script {
          if (changedServices.isEmpty()) {
            echo 'No changed services to build JARs for. Skipping Maven build.'
            return
          }

          for (service in changedServices) {
            if (!fileExists("${service}/pom.xml")) {
              echo "Skipping ${service}: pom.xml not found."
              continue
            }

            if (fileExists("${service}/mvnw") && fileExists("${service}/.mvn/wrapper/maven-wrapper.properties")) {
              sh """
                cd ./${service}
                chmod +x ./mvnw
                ./mvnw -B clean package -DskipTests
              """
            } else {
              sh "mvn -f ./${service}/pom.xml -B clean package -DskipTests"
            }
          }
        }
      }
    }

    stage('Build and Push Images') {
      steps {
        script {
          if (changedServices.isEmpty()) {
            echo 'No changed services to build. Skipping image build and push.'
            return
          }

          withCredentials([usernamePassword(
            credentialsId: 'dockerhub-creds',
            usernameVariable: 'DOCKER_USER',
            passwordVariable: 'DOCKER_PASS'
          )]) {
            sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'

            for (service in changedServices) {
              if (!fileExists("${service}/Dockerfile")) {
                echo "Skipping ${service}: Dockerfile not found."
                continue
              }

              sh """
                docker build -t ${DOCKERHUB_USER}/${service}:${COMMIT_ID} -f ./${service}/Dockerfile ./${service}
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
        def changedServicesValue = changedServices ? changedServices.join(' ') : ''
        if (changedServicesValue) {
          echo "Images pushed with tag: ${env.COMMIT_ID}"
          echo "Built services: ${changedServicesValue}"
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