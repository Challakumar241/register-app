pipeline {
  agent any

  tools {
    jdk 'Java17'
    maven 'Maven3'
  }

  environment {
    DOCKER_IMAGE = "challakumar241/challakumar241:${BUILD_NUMBER}"
    GIT_REPO_NAME = "register-app"
    GIT_USER_NAME = "Challakumar241"
  }

  stages {
    stage('Cleanup Workspace') {
      steps {
        cleanWs()
      }
    }

    stage('Checkout') {
      steps {
        git branch: 'main', url: "https://github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}.git", credentialsId: 'github'
      }
    }

    stage('Build and Test') {
      steps {
        sh 'mvn clean package'
        sh 'mvn test'
      }
    }

    stage('Build and Push Docker Image') {
      steps {
        script {
          def imageTag = "challakumar241/challakumar241:${BUILD_NUMBER}"
          def latestTag = "challakumar241/challakumar241:latest"

          sh "docker build -t ${imageTag} ."
          sh "docker tag ${imageTag} ${latestTag}"

          docker.withRegistry('https://index.docker.io/v1/', 'dockerhub') {
            docker.image(imageTag).push()
            docker.image(latestTag).push()
          }
        }
      }
    }

    stage('Trivy Security Scan') {
      steps {
        sh """
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
          aquasec/trivy image challakumar241/challakumar241:latest \
          --no-progress --scanners vuln --exit-code 0 \
          --severity HIGH,CRITICAL --format table
        """
      }
    }

    stage('Cleanup Docker Images') {
      steps {
        sh "docker rmi challakumar241/challakumar241:${BUILD_NUMBER} || true"
        sh "docker rmi challakumar241/challakumar241:latest || true"
      }
    }
  }
}
