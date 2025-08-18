pipeline {
    agent any

    tools {
        maven 'Maven3'
        jdk 'Java11'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/yosua789/Testing-Sast.git'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('SCA - Dependency Check') {
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
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
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
                sh '''
                  trivy image --exit-code 0 --severity HIGH,CRITICAL testing-sast:latest > trivy-report.txt
                '''
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
            echo "Pipeline selesai - hasil scan tersedia di artifacts."
        }
    }
}
