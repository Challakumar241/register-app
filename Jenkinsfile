pipeline {
    agent { label 'Jenkins-Agent' }

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        RELEASE = "1.0.0"
        DOCKER_REPO = "challakumar241/challakumar241"
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
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build and Push Docker Image') {
            steps {
                dir('java-maven-sonar-argocd-helm-k8s/spring-boot-app') {
                    script {
                        def dockerImage = "${DOCKER_REPO}:${RELEASE}-${BUILD_NUMBER}"
                        sh "docker build -t ${dockerImage} ."
                        docker.withRegistry('https://index.docker.io/v1/', 'docker-cred') {
                            docker.image(dockerImage).push()
                            docker.image(dockerImage).push('latest')
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
                    aquasec/trivy image ${DOCKER_REPO}:latest \
                    --no-progress --scanners vuln --exit-code 0 \
                    --severity HIGH,CRITICAL --format table
                    '''
                }
            }
        }

        stage("Cleanup Artifacts") {
            steps {
                script {
                    sh "docker rmi ${DOCKER_REPO}:${RELEASE}-${BUILD_NUMBER} || true"
                    sh "docker rmi ${DOCKER_REPO}:latest || true"
                }
            }
        }
    }
}
