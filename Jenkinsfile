pipeline {
  agent any
  options { skipDefaultCheckout(true) }

  environment {
    DOCKER_HOST    = "unix:///var/run/docker.sock"
    FAIL_ON_ISSUES = 'false'   // set 'true' kalau mau fail build saat ada vuln
  }

  stages {
    stage('Clean Workspace') {
      steps { cleanWs() }
    }

    stage('Checkout Repository') {
      steps {
        checkout([$class: 'GitSCM',
          branches: [[name: '*/main']],
          extensions: [[$class: 'CloneOption', shallow: false, noTags: false]],
          userRemoteConfigs: [[url: 'https://github.com/yosua789/Testing-Sast.git']]
        ])
      }
    }

    stage('Build with Maven') {
      agent {
        docker {
          image 'maven:3.9.9-eclipse-temurin-11'
          reuseNode true
        }
      }
      steps {
        script {
          // Aman meski tidak ada pom.xml
          def ec = sh(returnStatus: true, script: 'mvn -B clean package -DskipTests')
          if (ec != 0) {
            echo "Maven build skipped/failed (exit ${ec}). Lanjut ke SCA."
          }
        }
      }
    }

    stage('SCA - OWASP Dependency-Check') {
      agent {
        docker {
          image 'owasp/dependency-check:latest'
          reuseNode true
          args "-v ${WORKSPACE}/.odc:/usr/share/dependency-check/data"
        }
      }
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          script {
            sh 'mkdir -p dependency-check-report || true'
            def cmd = '''
              set +e
              /usr/share/dependency-check/bin/dependency-check.sh \
                --project "Testing-Sast" \
                --scan ./src \
                --format HTML \
                --out dependency-check-report
            '''
            def ec = sh(returnStatus: true, script: cmd)
            echo "Dependency-Check exit code: ${ec}"

            if (env.FAIL_ON_ISSUES == 'true' && ec != 0) {
              error "Fail build (policy) karena Dependency-Check exit ${ec}"
            }
          }
        }
      }
      post {
        always {
          script {
            def reportPath = 'dependency-check-report/dependency-check-report.html'
            if (fileExists(reportPath)) {
              publishHTML(target: [
                reportDir: 'dependency-check-report',
                reportFiles: 'dependency-check-report.html',
                reportName: 'Dependency-Check Report'
              ])
            } else {
              echo "Dependency-Check report tidak ditemukan, skip publishHTML."
            }
          }
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          // Kalau Docker belum siap, jangan jatuhkan build
          def ec1 = sh(returnStatus: true, script: 'docker version')
          if (ec1 != 0) {
            echo "Docker tidak siap (exit ${ec1}). Skip build image & lanjut."
          } else {
            def ec2 = sh(returnStatus: true, script: 'docker build -t testing-sast:latest .')
            if (ec2 != 0) {
              echo "Docker build gagal (exit ${ec2}). Lanjut ke Trivy stage (bisa skip otomatis)."
            }
          }
        }
      }
    }

    stage('SCA - Trivy Docker Scan') {
      agent {
        docker {
          image 'aquasec/trivy:latest'
          reuseNode true
          args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
      }
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          script {
            // Scan image; tetap 0 agar uji coba lanjut
            def ec = sh(returnStatus: true, script: '''
              set +e
              trivy image --no-progress --exit-code 0 \
                --severity HIGH,CRITICAL testing-sast:latest | tee trivy-report.txt
            ''')
            echo "Trivy scan exit code: ${ec}"
          }
        }
      }
      post {
        always {
          script {
            if (fileExists('trivy-report.txt')) {
              archiveArtifacts artifacts: 'trivy-report.txt', fingerprint: true
            } else {
              echo "trivy-report.txt tidak ditemukan, skip archive."
            }
          }
        }
      }
    }
  }

  post {
    always {
      echo "Pipeline selesai bro, result: ${currentBuild.currentResult}"
    }
  }
}
