pipeline {
  agent any
  options { skipDefaultCheckout(true) }

  environment {
    FAIL_ON_ISSUES = 'false'
  }

  stages {
    stage('Clean Workspace') {
      steps { cleanWs() }
    }

    stage('Checkout') {
      steps {
        checkout([$class: 'GitSCM',
          branches: [[name: '*/main']],
          extensions: [[$class: 'CloneOption', shallow: false, noTags: false]],
          userRemoteConfigs: [[url: 'https://github.com/yosua789/Testing-Sast.git']]
        ])
      }
    }

    // ===== SCA: OWASP Dependency-Check =====
    stage('SCA - Dependency-Check (repo)') {
      agent {
        docker {
          image 'owasp/dependency-check:latest'
          reuseNode true
          args "-v ${WORKSPACE}/.odc:/usr/share/dependency-check/data -v ${WORKSPACE}/.odc-temp:/tmp"
        }
      }
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          script {
            sh '''
              set -e
              mkdir -p dependency-check-report

              # update DB (if fail, keep continue)
              /usr/share/dependency-check/bin/dependency-check.sh --updateonly || true
              
              set +e
              /usr/share/dependency-check/bin/dependency-check.sh \
                --project "Testing-Sast" \
                --scan . \
                --format ALL \
                --out dependency-check-report \
                --log dependency-check-report/dependency-check.log \
                --failOnCVSS 11
              echo $? > .dc_exit
            '''
            def rc = readFile('.dc_exit').trim()
            echo "Dependency-Check exit code: ${rc}"
            if (env.FAIL_ON_ISSUES == 'true' && rc != '0') {
              error "Fail build (policy) karena Dependency-Check exit ${rc}"
            }
          }
        }
      }
      post {
        always {
          script {
            if (fileExists('dependency-check-report/dependency-check.log')) {
              archiveArtifacts artifacts: 'dependency-check-report/dependency-check.log', fingerprint: true
            }
            if (fileExists('dependency-check-report/dependency-check-report.html')) {
              publishHTML(target: [
                reportDir: 'dependency-check-report',
                reportFiles: 'dependency-check-report.html',
                reportName: 'Dependency-Check Report'
              ])
            } else {
              echo "Dependency-Check HTML report tidak ditemukan. Cek dependency-check-report/dependency-check.log"
            }
          }
        }
      }
    }

    // ===== SCA: Trivy (scan system container) =====
    stage('SCA - Trivy (filesystem)') {
      agent {
        docker {
          image 'aquasec/trivy:latest'
          reuseNode true
          args '--entrypoint="" -v ${WORKSPACE}/.trivy-cache:/root/.cache/trivy'
        }
      }
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          script {
            sh 'rm -f trivy-fs.txt trivy-fs.sarif || true'
            def ec = sh(returnStatus: true, script: '''
              set +e
              trivy fs --no-progress --exit-code 0 \
                --severity HIGH,CRITICAL . | tee trivy-fs.txt

              trivy fs --no-progress --exit-code 0 \
                --severity HIGH,CRITICAL --format sarif -o trivy-fs.sarif .
            ''')
            echo "Trivy FS scan exit code: ${ec}"
            if (env.FAIL_ON_ISSUES == 'true' && ec != 0) {
              error "Fail build (policy) if Trivy FS exit ${ec}"
            }
          }
        }
      }
      post {
        always {
          script {
            if (fileExists('trivy-fs.txt')) {
              archiveArtifacts artifacts: 'trivy-fs.txt', fingerprint: true
            } else {
              echo "trivy-fs.txt tidak ditemukan."
            }
            if (fileExists('trivy-fs.sarif')) {
              archiveArtifacts artifacts: 'trivy-fs.sarif', fingerprint: true
            }
          }
        }
      }
    }
  }

  post {
    always {
      echo "Scanning All Done. Result: ${currentBuild.currentResult}"
    }
  }
}
