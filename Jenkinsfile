pipeline {
  agent any

  environment {
    DOCKER_IMAGE = "yogithak/go-web-app:${BUILD_NUMBER}"
    DOCKER_CREDENTIALS_ID = 'docker-cred'    // DockerHub credentials ID in Jenkins
    GITHUB_CREDENTIALS_ID = 'github'         // GitHub token secret text in Jenkins
    GIT_REPO_NAME = "go-web-app-devops"
    GIT_USER_NAME = "yogitha-koya"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build and Test Go App') {
      steps {
        sh '''
          go version
          go build -o go-web-app
          go test ./...
        '''
      }
    }

    stage('Build and Push Docker Image') {
      steps {
        script {
          sh 'docker build -t ${DOCKER_IMAGE} .'
          def image = docker.image("${DOCKER_IMAGE}")
          docker.withRegistry('https://index.docker.io/v1/', "${DOCKER_CREDENTIALS_ID}") {
            image.push()
          }
        }
      }
    }

    stage('Update Helm Chart with New Image Tag') {
      steps {
        withCredentials([string(credentialsId: "${GITHUB_CREDENTIALS_ID}", variable: 'GITHUB_TOKEN')]) {
          sh '''
            git config user.email "yogitha.koya@example.com"
            git config user.name "Yogitha Koya"
            sed -i "s/tag: .*/tag: \\"${BUILD_NUMBER}\\"/" helm/go-web-app-chart/values.yaml
            git add helm/go-web-app-chart/values.yaml
            git commit -m "ci: update image tag to ${BUILD_NUMBER}" || echo "No changes to commit"
            git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}.git HEAD:main
          '''
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'go-web-app', fingerprint: true
      cleanWs()
    }
  }
}
