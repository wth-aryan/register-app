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
        JENKINS_API_TOKEN = credentials("JENKINS_API_TOKEN")
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
                    sh '''
                        docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy image ${IMAGE_NAME}:latest \
                        --no-progress \
                        --scanners vuln \
                        --exit-code 0 \
                        --severity HIGH,CRITICAL \
                        --format table
                    '''
                }
            }
        }

       stage('Trigger CD Pipeline') {
    steps {
        withCredentials([string(credentialsId: 'JENKINS_API_TOKEN', variable: 'TOKEN')]) {
            sh '''
                set -euo pipefail

                TARGET="http://ec2-13-61-154-102.eu-north-1.compute.amazonaws.com:8080"
                JOB_PATH="${TARGET}/job/gitops-register-app-cd"
                RETRIES=3
                SLEEP=5

                attempt=1
                while [ $attempt -le $RETRIES ]; do
                    CRUMB_HEADER=$(curl -sS --connect-timeout 5 --max-time 10 -u aryan:${TOKEN} "${JOB_PATH}/crumbIssuer/api/xml?xpath=concat(//crumbRequestField,\\\":\\\",//crumb)" || true)

                    if [ -n "$CRUMB_HEADER" ]; then
                        curl -sS --connect-timeout 10 --max-time 30 -u aryan:${TOKEN} \
                            -X POST -H "$CRUMB_HEADER" \
                            "${JOB_PATH}/build?token=${TOKEN}" && break || true
                    else
                        curl -sS --connect-timeout 10 --max-time 30 -u aryan:${TOKEN} \
                            -X POST "${JOB_PATH}/build?token=${TOKEN}" && break || true
                    fi

                    attempt=$((attempt+1))
                    sleep $SLEEP
                done

                true
            '''
        }
    }
}

    post {
        always {
            script {
                sh 'docker system prune -f || true'
                sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
                sh "docker rmi ${IMAGE_NAME}:latest || true"
                cleanWs()
            }
        }
    }
}
