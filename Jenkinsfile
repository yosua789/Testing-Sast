pipeline {
    agent any

    environment {
        DOCKER_HOST = "unix:///var/run/docker.sock"
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    doGenerateSubmoduleConfigurations: false,
                    extensions: [[$class: 'CloneOption', noTags: false, shallow: false]],
                    userRemoteConfigs: [[url: 'https://github.com/yosua789/Testing-Sast.git']]
                ])
            }
        }

        stage('Build with Maven') {
            agent {
                docker {
                    image 'maven:3.8.8-openjdk-11'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
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
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
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
