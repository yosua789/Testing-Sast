pipeline {
    agent any

    environment {
        DOCKER_HOST = "unix:///var/run/docker.sock"
    }

    stages {
        // Checkout di node host, bukan container
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // Build Maven pakai Docker container
        stage('Build with Maven') {
            agent {
                docker {
                    image 'maven:3.8.8-openjdk-11'
                    args '-v /var/run/docker.sock:/var/run/docker.sock -v $WORKSPACE:$WORKSPACE -w $WORKSPACE'
                }
            }
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        // Dependency check dengan container
        stage('SCA - Dependency Check') {
            agent {
                docker {
                    image 'owasp/dependency-check:latest'
                    args '-v /var/run/docker.sock:/var/run/docker.sock -v $WORKSPACE:$WORKSPACE -w $WORKSPACE'
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

        // Build Docker image
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t testing-sast:latest .'
            }
        }

        // Trivy scan
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
