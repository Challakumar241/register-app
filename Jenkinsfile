pipeline {
    agent { label 'Jenkins-Agent' }

    tools {
        jdk 'java17'         // Matches Global Tool Configuration
        maven 'maven3'       // Matches Global Tool Configuration
    }

    environment {
        APP_NAME = "register-app-pipeline"
        RELEASE = "1.0.0"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
        IMAGE_NAME = "challakumar241/${APP_NAME}" // Adjust username as needed
        JENKINS_API_TOKEN = credentials("JENKINS_API_TOKEN")
    }

    stages {
        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Checkout from SCM") {
            steps {
                git branch: 'main', credentialsId: 'github', url: 'https://github.com/Challakumar241/register-app'
            }
        }

        stage("Build Application") {
            steps {
                sh 'mvn clean package'
            }
        }

        stage("Test Application") {
            steps {
                sh 'mvn test'
            }
        }

        stage("SonarQube Analysis") {
            steps {
                script {
                    withSonarQubeEnv(credentialsId: 'jenkins-sonarqube-token') {
                        sh 'mvn sonar:sonar'
                    }
                }
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'jenkins-sonarqube-token'
                }
            }
        }

        stage("Build & Push Docker Image") {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh '''
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                            export DOCKER_BUILDKIT=1
                            docker build -t "$DOCKER_USER/$APP_NAME:$IMAGE_TAG" .
                            docker tag "$DOCKER_USER/$APP_NAME:$IMAGE_TAG" "$DOCKER_USER/$APP_NAME:latest"
                            docker push "$DOCKER_USER/$APP_NAME:$IMAGE_TAG"
                            docker push "$DOCKER_USER/$APP_NAME:latest"
                        '''
                    }
                }
            }
        }

        stage("Trivy Scan") {
            steps {
                sh '''
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy image "$IMAGE_NAME:latest" \
                        --no-progress --scanners vuln --exit-code 0 \
                        --severity HIGH,CRITICAL --format table || true
                '''
            }
        }

        stage("Cleanup Docker Artifacts") {
            steps {
                sh '''
                    docker rmi "$IMAGE_NAME:$IMAGE_TAG" || true
                    docker rmi "$IMAGE_NAME:latest" || true
                '''
            }
        }

        stage("Trigger CD Pipeline") {
            steps {
                script {
                    sh """
                        curl -v -k --user clouduser:${JENKINS_API_TOKEN} -X POST \\
                        -H 'cache-control: no-cache' \\
                        -H 'content-type: application/x-www-form-urlencoded' \\
                        --data 'IMAGE_TAG=${IMAGE_TAG}' \\
                        'http://ec2-54-234-152-100.compute-1.amazonaws.com:8080/job/gitops-register-app-cd/buildWithParameters?token=gitops-token'
                    """
                }
            }
        }
    }

    post {
        failure {
            emailext(
                body: '${SCRIPT, template="groovy-html.template"}',
                subject: "${env.JOB_NAME} - Build #${env.BUILD_NUMBER} - Failed",
                mimeType: 'text/html',
                to: "challakumar241@gmail.com"
            )
        }
        success {
            emailext(
                body: '${SCRIPT, template="groovy-html.template"}',
                subject: "${env.JOB_NAME} - Build #${env.BUILD_NUMBER} - Successful",
                mimeType: 'text/html',
                to: "challakumar241@gmail.com"
            )
        }
    }
}
