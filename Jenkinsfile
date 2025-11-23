pipeline {
    agent { label 'Jenkins-Agent' }

    tools {
        jdk 'java17'
        maven 'Maven3'
    }

    stages {

        stage('Cleanup Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from SCM') {
            steps {
                git branch: 'main', credentialsId: 'github', url: 'https://github.com/wth-aryan/register-app'
            }
        }

        stage('Build Application') {
            steps {
                sh "mvn clean package"
            }
        }

        stage('Test Application') {
            steps {
                sh "mvn test"
            }
        }

        stage("SonarQube Analysis") {
            steps {
                script {
                    withSonarQubeEnv(credentialsId: 'jenkins-sonarqube-token') {
                        sh 'mvn sonar:sonar -Dsonar.projectKey=register-app -Dsonar.host.url=http://<YOUR-IP>:9000'
                    }
                }
            }
        }

        stage("Quality Gate") {
            steps {
                timeout(time: 3, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
    }
}
