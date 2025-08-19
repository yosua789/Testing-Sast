pipeline {
  agent any
  options { skipDefaultCheckout(true) }

  environment {
    DOCKER_HOST    = "unix:///var/run/docker.sock"
    FAIL_ON_ISSUES = 'false'   // set 'true' utk fail build saat ada vuln di DC
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

    // ===== Build (opsional; aman walau tidak ada pom.xml) =====
    stage('Build with Maven') {
      agent {
        docker {
          image 'maven:3.9.9-eclipse-temurin-11'
          reuseNode true
        }
      }
      steps {
        script {
          def ec = sh(returnStatus: true, script: 'mvn -B clean package -DskipTests')
          if (ec != 0) {
            echo "Maven build skipped/failed (exit ${ec}). Lanjut SCA & container build."
          }
        }
      }
    }

    // ===== SCA: OWASP Dependency-Check =====
    stage('SCA - OWASP Dependency-Check') {
      agent {
        docker {
          image 'owasp/dependency-check:latest'
          reuseNode true
          // cache NVD + temp agar cepat/stabil
          args "-v ${WORKSPACE}/.odc:/usr/share/dependency-check/data -v ${WORKSPACE}/.odc-temp:/tmp"
        }
      }
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          script {
            sh '''
              set -e
              mkdir -p dependency-check-report

              # (opsional) update DB; jangan hentikan pipeline bila gagal
              /usr/share/dependency-check/bin/dependency-check.sh --updateonly || true

              # scan seluruh repo (lebih aman dari ./src saja)
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
              echo "Dependency-Check report tidak ditemukan. Cek dependency-check-report/dependency-check.log"
            }
          }
        }
      }
    }

    // ===== Docker Build (wajib sukses agar Trivy bisa scan image) =====
    stage('Build Docker Image') {
      steps {
        script {
          def e1 = sh(returnStatus: true, script: 'docker version')
          if (e1 != 0) {
            error "Docker daemon tidak siap (exit ${e1})."
          }
          def e2 = sh(returnStatus: true, script: 'docker build -t testing-sast:latest .')
          if (e2 != 0) {
            error "Docker build gagal (exit ${e2}). Periksa Dockerfile sesuai tipe project."
          }
        }
      }
    }

    // ===== SCA: Trivy (scan image) =====
    stage('SCA - Trivy Docker Scan') {
      agent {
        docker {
          image 'aquasec/trivy:latest'
          reuseNode true
          // kosongkan entrypoint + mount docker.sock + cache
          args '--entrypoint="" -v /var/run/docker.sock:/var/run/docker.sock -v ${WORKSPACE}/.trivy-cache:/root/.cache/trivy'
        }
      }
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          script {
            def ec = sh(returnStatus: true, script: '''
              set +e
              trivy --version || true
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
              echo "trivy-report.txt tidak ditemukan. Pastikan image 'testing-sast:latest' berhasil dibuild."
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
