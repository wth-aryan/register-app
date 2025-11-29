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
        IMAGE_NAME = "${DOCKER_USER}" + "/" + "${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
    }
    
    stages{
        stage("Cleanup Workspace"){
            steps {
                cleanWs()
            }
        }

        stage("Checkout from SCM"){
            steps {
                git branch: 'main', credentialsId: 'github', url: 'https://github.com/wth-aryan/register-app'
            }
        }

        stage("Build Application"){
            steps {
                sh "mvn clean package"
            }
        }

        stage("Test Application"){
            steps {
                sh "mvn test"
            }
        }
        
        stage("SonarQube Analysis"){
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
                    docker_image = docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                    docker.withRegistry('', "${DOCKER_PASS}") {
                        docker_image.push("${IMAGE_TAG}")
                        docker_image.push("latest")
                    }
                }
            }
        }

        stage("Trivy Scan") {
            steps {
                script {
                    sh ("docker run -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image ${IMAGE_NAME}:latest --no-progress --scanners vuln --exit-code 0 --severity HIGH,CRITICAL --format table")
                }
            }
        }

        stage ('Cleanup Artifacts') {
            steps {
                script {
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
                    sh "docker rmi ${IMAGE_NAME}:latest || true"
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
