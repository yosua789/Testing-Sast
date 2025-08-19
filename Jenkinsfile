pipeline {
  agent any
  options { skipDefaultCheckout(true) }

  environment {
    DOCKER_HOST    = "unix:///var/run/docker.sock"
    FAIL_ON_ISSUES = 'false'   // set 'true' untuk menegakkan kegagalan saat ada vuln
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
          image 'maven:3.9.9-eclipse-temurin-11'   // Java 11; ganti ke -17 bila perlu
          reuseNode true
        }
      }
      steps {
        script {
          // Aman walau tidak ada pom.xml (hanya log & lanjut)
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
          // cache NVD + temp agar cepat dan stabil
          args "-v ${WORKSPACE}/.odc:/usr/share/dependency-check/data -v ${WORKSPACE}/.odc-temp:/tmp"
        }
      }
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          script {
            sh '''
              set -e
              mkdir -p dependency-check-report

              # (Opsional) Update DB dulu; jika gagal, jangan hentikan pipeline
              /usr/share/dependency-check/bin/dependency-check.sh --updateonly || true

              # Scan; --failOnCVSS 11 = jangan fail karena temuan (hanya crash yang membuat non-zero)
              set +e
              /usr/share/dependency-check/bin/dependency-check.sh \
                --project "Testing-Sast" \
                --scan ./src \
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
            // Arsip log untuk debugging
            if (fileExists('dependency-check-report/dependency-check.log')) {
              archiveArtifacts artifacts: 'dependency-check-report/dependency-check.log', fingerprint: true
            }
            // Publish HTML hanya jika ada
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

    stage('Build Docker Image') {
      steps {
        script {
          def ec1 = sh(returnStatus: true, script: 'docker version')
          if (ec1 != 0) {
            echo "Docker tidak siap (exit ${ec1}). Skip build image & lanjut."
          } else {
            def ec2 = sh(returnStatus: true, script: 'docker build -t testing-sast:latest .')
            if (ec2 != 0) {
              echo "Docker build gagal (exit ${ec2}). Trivy mungkin tidak menemukan image."
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
          // pakai shell sebagai entrypoint + akses docker.sock + cache
          args '--entrypoint=/bin/sh -v /var/run/docker.sock:/var/run/docker.sock -v ${WORKSPACE}/.trivy-cache:/root/.cache/trivy'
        }
      }
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          script {
            def ec = sh(returnStatus: true, script: '''
              set -e
              trivy --version || true
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
              echo "trivy-report.txt tidak ditemukan, kemungkinan Trivy belum berjalan benar."
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
