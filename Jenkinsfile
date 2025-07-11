pipeline {
  agent {
    docker {
      image 'golang:1.22-alpine'     // Use official Go image
      args '-u root -v /var/run/docker.sock:/var/run/docker.sock'  // To enable Docker CLI inside container
    }
  }

  environment {
    DOCKER_IMAGE = "yogithak/go-web-app:${BUILD_NUMBER}"
    DOCKER_CREDENTIALS_ID = 'docker-cred'    // Jenkins credentials ID for DockerHub
    GITHUB_CREDENTIALS_ID = 'github'         // Jenkins secret text (GitHub PAT)
    GIT_REPO_NAME = "go-web-app-devops"
    GIT_USER_NAME = "yogitha-koya"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Install Docker CLI') {
      steps {
        sh '''
          apk add --no-cache docker-cli git bash
        '''
      }
    }

    stage('Build and Test Go App') {
      steps {
        sh '''
          go version
          go build -buildvcs=false -o go-web-app
          go test ./...
        '''
      }
    }

    stage('Scan Docker Image with Docker Scout') {
      steps {
        withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDENTIALS_ID}", usernameVariable: 'DOCKER_HUB_USERNAME', passwordVariable: 'DOCKER_HUB_PASSWORD')]) {
          script {
            sh '''
              echo "Installing Docker Scout CLI..."
              apk add --no-cache curl
              curl -sSfL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh | sh -s -- -b /usr/local/bin
              
              echo 'Docker login...'
              docker login -u "$DOCKER_HUB_USERNAME" -p "$DOCKER_HUB_PASSWORD"
              
              echo "Building Docker image for scan..."
              docker build -t ${DOCKER_IMAGE} .

              echo "Scanning image with Docker Scout..."
              docker-scout quickview ${DOCKER_IMAGE} || echo "Scan completed with warnings"
            '''
          }
        }
      }
    }  
    stage('Push Docker Image to Docker Hub') {
      steps {
        script {
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

            echo "Current directory: \$(pwd)"
            ls -al
            git config --global --add safe.directory /var/lib/jenkins/workspace/go-web-app-pipeline
            
            sed -i "s/tag: .*/tag: \\"${BUILD_NUMBER}\\"/" helm/go-web-app-chart/values.yaml
            git config user.email "yogithak@gmail.com"
            git config user.name "Yogitha K"
            
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
