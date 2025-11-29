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
                    sh """
                        mkdir -p .trivy-cache
                        chmod 777 .trivy-cache

                        docker run --rm \\
                        -v /var/run/docker.sock:/var/run/docker.sock \\
                        -v \$(pwd)/.trivy-cache:/var/trivy-cache \\
                        -v \$(pwd)/.trivy-cache:/tmp \\
                        -e TRIVY_CACHE_DIR=/var/trivy-cache \\
                        -e TRIVY_TMP_DIR=/var/trivy-cache \\
                        -e TMPDIR=/tmp \\
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

    stage('Trigger CD Pipeline') {
  steps {
    withCredentials([string(credentialsId: 'jenkins-api-token', variable: 'JENKINS_API_TOKEN')]) {
      sh '''
        set -euo pipefail

        TARGET="https://ec2-13-61-154-102.eu-north-1.compute.amazonaws.com"
        JOB_PATH="${TARGET}/job/gitops-register-app-cd"
        # number of attempts
        RETRIES=3
        SLEEP=5

        attempt=1
        while [ $attempt -le $RETRIES ]; do
          # Try to obtain crumb (if Jenkins has CSRF enabled)
          CRUMB_HEADER=$(curl -sS --connect-timeout 5 --max-time 10 -u clouduser:$JENKINS_API_TOKEN "$JOB_PATH/crumbIssuer/api/xml?xpath=concat(//crumbRequestField,\":\",//crumb)" || true)

          if [ -n "$CRUMB_HEADER" ]; then
            curl -sS -k --connect-timeout 10 --max-time 30 -u clouduser:$JENKINS_API_TOKEN \
              -X POST -H "$CRUMB_HEADER" \
              "${JOB_PATH}/build?token=$JENKINS_API_TOKEN" && break || true
          else
            curl -sS -k --connect-timeout 10 --max-time 30 -u clouduser:$JENKINS_API_TOKEN \
              -X POST "${JOB_PATH}/build?token=$JENKINS_API_TOKEN" && break || true
          fi

          attempt=$((attempt+1))
          sleep $SLEEP
        done

        # If still failing, exit 0 so pipeline won't be marked FAILED due to remote CD unreachability
        true
      '''
    }
  }
}


    post {
        always {
            script {
                sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
                sh "docker rmi ${IMAGE_NAME}:latest || true"
                cleanWs()
            }
        }
    }
}
