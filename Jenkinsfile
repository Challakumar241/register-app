pipeline {
    agent { label 'Jenkins-Agent' }

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        RELEASE = "1.0.0"
        DOCKER_USER = "challakumar241"
        IMAGE_NAME = "${DOCKER_USER}/challakumar241" // Your DockerHub repo
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
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
                sh "mvn clean package"
            }
        }

        stage("Test Application") {
            steps {
                sh "mvn test"
            }
        }

        stage("SonarQube Analysis") {
            steps {
                script {
                    withSonarQubeEnv(credentialsId: 'jenkins-sonarqube-token') { 
                        sh "mvn sonar:sonar"
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
                dir('.') {
                    script {
                        def versionTag = "${IMAGE_NAME}:${BUILD_NUMBER}"
                        def latestTag = "${IMAGE_NAME}:latest"

                        // Build Docker image with both tags
                        sh "docker build -t ${versionTag} ."
                        sh "docker tag ${versionTag} ${latestTag}"

                        // Push to DockerHub
                        docker.withRegistry('https://index.docker.io/v1/', 'dockerhub') {
                            docker.image(versionTag).push()
                            docker.image(latestTag).push()
                        }
                    }
                }
            }
        }

        stage("Trivy Scan") {
            steps {
                script {
                    sh '''
                        docker run -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy image challakumar241/challakumar241:latest \
                        --no-progress --scanners vuln --exit-code 0 \
                        --severity HIGH,CRITICAL --format table
                    '''
                }
            }
        }

        stage("Cleanup Artifacts") {
            steps {
                script {
                    sh "docker rmi ${IMAGE_NAME}:${BUILD_NUMBER} || true"
                    sh "docker rmi ${IMAGE_NAME}:latest || true"
                }
            }
        }
    }
}

