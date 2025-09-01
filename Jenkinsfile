pipeline {
  agent any
  options { skipDefaultCheckout(true) }

  environment {
    FAIL_ON_ISSUES     = 'false'

    // SonarQube
    SONAR_HOST_URL     = 'http://sonarqube:9000'
    SONAR_PROJECT_KEY  = 'coba'
    SONAR_PROJECT_NAME = 'coba'
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
        sh 'echo "WS: $WORKSPACE" && ls -la'
      }
    }

    // === SAST: SonarQube ===
    stage('SAST - SonarQube') {
      steps {
        withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
          sh '''
            set -eux
            docker pull sonarsource/sonar-scanner-cli
            docker run --rm --network jenkins \
              -e SONAR_HOST_URL="$SONAR_HOST_URL" \
              -e SONAR_TOKEN="$SONAR_TOKEN" \
              -v "$WORKSPACE:/usr/src" \
              sonarsource/sonar-scanner-cli \
                -Dsonar.host.url="$SONAR_HOST_URL" \
                -Dsonar.token="$SONAR_TOKEN" \
                -Dsonar.projectKey="$SONAR_PROJECT_KEY" \
                -Dsonar.projectName="$SONAR_PROJECT_NAME" \
                -Dsonar.sources=. \
                -Dsonar.inclusions="**/*" \
                -Dsonar.exclusions="**/log/**,**/log4/**,**/log_3/**,**/*.test.*,**/node_modules/**,**/dist/**,**/build/**,docker-compose.yaml" \
                -Dsonar.coverage.exclusions="**/*.test.*,**/test/**,**/tests**"
          '''
        }
      }
    }

    // === SCA: OWASP Dependency-Check (repo) ===
    stage('SCA - Dependency-Check (repo)') {
      agent {
        docker {
          image 'owasp/dependency-check:latest'
          reuseNode true
          // WAJIB: kosongkan entrypoint supaya Jenkins bisa menjalankan `cat`
          args "--entrypoint='' -v $WORKSPACE/.odc:/usr/share/dependency-check/data -v $WORKSPACE/.odc-temp:/tmp"
        }
      }
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          script {
            sh '''
              set -eu
              mkdir -p dependency-check-report
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
              error "Fail build (policy) Dependency-Check exit ${rc}"
            }
          }
        }
      }
      post {
        always {
          script {
            if (fileExists('dependency-check-report/dependency-check.log')) {
              archiveArtifacts artifacts: 'dependency-check-report/**', fingerprint: false, onlyIfSuccessful: false
            } else {
              echo "Dependency-Check report not found"
            }
          }
        }
      }
    }

    // === SCA: Trivy (filesystem) ===
    stage('SCA - Trivy (filesystem)') {
      agent {
        docker {
          image 'aquasec/trivy:latest'
          reuseNode true
          // FIX: cache permission (user 1000). Pakai cache di /tmp & mount ke workspace
          args "--entrypoint='' -e HOME=/tmp -e XDG_CACHE_HOME=/tmp/trivy-cache -v $WORKSPACE/.trivy-cache:/tmp/trivy-cache"
        }
      }
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          script {
            sh '''
              rm -f trivy-fs.txt trivy-fs.sarif || true
              set +e
              trivy fs --cache-dir /tmp/trivy-cache --no-progress --exit-code 0 --severity HIGH,CRITICAL . | tee trivy-fs.txt
              trivy fs --cache-dir /tmp/trivy-cache --no-progress --exit-code 0 --severity HIGH,CRITICAL --format sarif -o trivy-fs.sarif .
              echo $? > .trivy_exit
            '''
            def ec = readFile('.trivy_exit').trim()
            echo "Trivy FS scan exit code: ${ec}"
            if (env.FAIL_ON_ISSUES == 'true' && ec != '0') {
              error "Fail build (policy) Trivy FS exit ${ec}"
            }
            sh 'ls -lh trivy-fs.* || true'
          }
        }
      }
      post {
        always {
          script {
            if (fileExists('trivy-fs.txt'))   { archiveArtifacts artifacts: 'trivy-fs.txt',  fingerprint: false }
            if (fileExists('trivy-fs.sarif')) { archiveArtifacts artifacts: 'trivy-fs.sarif', fingerprint: false }
          }
        }
      }
    }

    // === SAST: Semgrep ===
    stage('SAST - Semgrep') {
      agent {
        docker {
          image 'semgrep/semgrep:latest'
          reuseNode true
          args "--entrypoint=''"
        }
      }
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          script {
            sh '''
              set +e
              semgrep --version || true

              # SARIF
              semgrep scan \
                --config p/ci --config p/owasp-top-ten --config p/docker \
                --exclude 'log/**' --exclude '**/node_modules/**' --exclude '**/dist/**' --exclude '**/build/**' \
                --severity ERROR --error \
                --sarif --output semgrep.sarif .

              # JUnit
              semgrep scan \
                --config p/ci --config p/owasp-top-ten --config p/docker \
                --exclude 'log/**' --exclude '**/node_modules/**' --exclude '**/dist/**' --exclude '**/build/**' \
                --severity ERROR --error \
                --junit-xml --output semgrep-junit.xml .

              echo $? > .semgrep_exit
            '''
            def ec = readFile('.semgrep_exit').trim()
            if (env.FAIL_ON_ISSUES == 'true' && ec != '0') {
              error "Fail build (policy) Semgrep exit ${ec}"
            }
            sh 'ls -lh semgrep.* || true'
          }
        }
      }
      post {
        always {
          script {
            if (fileExists('semgrep.sarif'))  { archiveArtifacts artifacts: 'semgrep.sarif',    fingerprint: false }
            if (fileExists('semgrep-junit.xml')) {
              archiveArtifacts artifacts: 'semgrep-junit.xml', fingerprint: false
              junit allowEmptyResults: false, testResults: 'semgrep-junit.xml'
            } else {
              echo "semgrep-junit.xml not found"
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
