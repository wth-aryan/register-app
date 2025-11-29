pipeline {
    agent { label 'Jenkins-Agent' }
    tools {
        jdk 'java17'
        maven 'Maven3'
    }

    environment {
        APP_NAME = "register-app-pipeline"
        RELEASE = "1.0.0"
        DOCKER_USER = "wtharyan"
        DOCKER_PASS = 'dockerhub'
        IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
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
                git branch: 'main', credentialsId: 'github', url: 'https://github.com/wth-aryan/register-app'
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
                        sh "mvn org.sonarsource.scanner.maven:sonar-maven-plugin:3.11.0.3922:sonar"
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: false, credentialsId: 'jenkins-sonarqube-token'
            }
        }

        stage("Build & Push Docker Image") {
            steps {
                script {
                    def dockerImage = docker.build("${env.IMAGE_NAME}:${env.IMAGE_TAG}")
                    docker.withRegistry('', env.DOCKER_PASS) {
                        dockerImage.push("${env.IMAGE_TAG}")
                        dockerImage.push("latest")
                    }
                }
            }
        }

        stage("Trivy Scan") {
            steps {
                script {
                    sh """
                        # Create cache dir
                        mkdir -p .trivy-cache
                        chmod 777 .trivy-cache

                        # Run Trivy Scan
                        # Added -e TMPDIR to force downloads to the mounted volume
                        docker run --rm \\
                        -v /var/run/docker.sock:/var/run/docker.sock \\
                        -v \$(pwd)/.trivy-cache:/var/trivy-cache \\
                        -e TRIVY_CACHE_DIR=/var/trivy-cache \\
                        -e TRIVY_TMP_DIR=/var/trivy-cache \\
                        -e TMPDIR=/var/trivy-cache \\
                        aquasec/trivy image ${IMAGE_NAME}:latest \\
                        --no-progress \\
                        --scanners vuln \\
                        --exit-code 0 \\
                        --severity HIGH,CRITICAL \\
                        --format table
                    """
                }
            }
        }
        stage ('Cleanup Artifacts') {
            steps {
                script {
                    sh "docker rmi ${env.IMAGE_NAME}:${env.IMAGE_TAG} || true"
                    sh "docker rmi ${env.IMAGE_NAME}:latest || true"
                }
            }
        }

        stage('Trigger CD Pipeline') {
            steps {
                script {
                    sh """
                        curl -sS -k \
                          --user clouduser:${JENKINS_API_TOKEN} \
                          -X POST \
                          -H 'Cache-Control: no-cache' \
                          -H 'Content-Type: application/x-www-form-urlencoded' \
                          "https://ec2-13-61-154-102.eu-north-1.compute.amazonaws.com/job/gitops-register-app-cd/build?token=${JENKINS_API_TOKEN}"
                    """
                }
            }
        }
    }
}
