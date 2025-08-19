pipeline {
    agent any

    environment {
        DOCKER_HOST = "tcp://docker:2375"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/yosua789/Testing-Sast.git'
            }
        }

        stage('Build') {
            agent {
                docker { 
                    image 'maven:3.8.8-openjdk-11'
                    args '-e DOCKER_HOST=tcp://docker:2375'
                }
            }
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('SCA - Dependency Check') {
            agent {
                docker {
                    image 'owasp/dependency-check:latest'
                    args '-e DOCKER_HOST=tcp://docker:2375'
                }
            }
            steps {
                sh '''
                  dependency-check.sh \
                    --project Testing-Sast \
                    --scan ./src/main \
                    --format HTML \
                    --out dependency-check-report
                '''
            }
            post {
                always {
                    publishHTML(target: [
                        reportDir: 'dependency-check-report',
                        reportFiles: 'dependency-check-report.html',
                        reportName: 'Dependency-Check Report'
                    ])
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t testing-sast:latest .'
            }
        }

        stage('SCA - Trivy Docker Scan') {
            steps {
                sh 'trivy image --exit-code 0 --severity HIGH,CRITICAL testing-sast:latest > trivy-report.txt'
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.txt', fingerprint: true
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline done"
        }
    }
}
