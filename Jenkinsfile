pipeline {
  agent any
  options { skipDefaultCheckout(true) }

  environment {
    FAIL_ON_ISSUES     = 'false'

    // ==== SonarQube ====
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

    // ======= Preflight: pastikan token benar (Bearer ke API v2, tanpa pipefail) =======
    stage('Sonar token preflight') {
      steps {
        withCredentials([string(credentialsId: 'sonarqube-token', variable: 'T')]) {
          sh '''
            set -eu
            echo "== Token length & fingerprint =="
            echo -n "$T" | wc -c | awk '{print "len=" $1}'
            printf %s "$T" | sha256sum | awk '{print "sha256=" $1}'

            echo "== Cek API v2 version (401 kalau token salah) =="
            API_VER="$(docker run --rm --network jenkins curlimages/curl -fsS \
              -H "Authorization: Bearer $T" \
              "$SONAR_HOST_URL/api/v2/analysis/version")" || {
                echo "Gagal akses API v2 (auth/network)"; exit 1;
              }
            echo "APIv2: $API_VER"

            echo "== Cek project visible (informational) =="
            docker run --rm --network jenkins curlimages/curl -fsS \
              -H "Authorization: Bearer $T" \
              "$SONAR_HOST_URL/api/projects/search?projects=$SONAR_PROJECT_KEY" \
              -o .sonar_project_search.json || true
            test -s .sonar_project_search.json && cat .sonar_project_search.json || echo "no response (mungkin private)"
          '''
        }
      }
    }

    // ===== SAST: SonarQube (pakai Bearer via env SONAR_TOKEN) =====
    stage('SAST - SonarQube') {
      steps {
        withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
          sh '''
            set -eu
            docker pull sonarsource/sonar-scanner-cli
            docker run --rm --network jenkins \
              -e SONAR_HOST_URL="$SONAR_HOST_URL" \
              -e SONAR_TOKEN="$SONAR_TOKEN" \
              -v "$WORKSPACE:/usr/src" \
              sonarsource/sonar-scanner-cli \
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

    // ===== SCA: OWASP Dependency-Check =====
    stage('SCA - Dependency-Check (repo)') {
      agent {
        docker {
          image 'owasp/dependency-check:latest'
          reuseNode true
          args "-v $WORKSPACE/.odc:/usr/share/dependency-check/data -v $WORKSPACE/.odc-temp:/tmp"
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
              archiveArtifacts artifacts: 'dependency-check-report/dependency-check.log', fingerprint: true
            }
            if (fileExists('dependency-check-report/dependency-check-report.html')) {
              publishHTML(target: [
                reportDir: 'dependency-check-report',
                reportFiles: 'dependency-check-report.html',
                reportName: 'Dependency-Check Report'
              ])
            } else {
              echo "Dependency-Check HTML report not found"
            }
          }
        }
      }
    }

    // ===== SCA: Trivy (filesystem) =====
    stage('SCA - Trivy (filesystem)') {
      agent {
        docker {
          image 'aquasec/trivy:latest'
          reuseNode true
          args "--entrypoint= -v $WORKSPACE/.trivy-cache:/root/.cache/trivy"
        }
      }
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          script {
            sh 'rm -f trivy-fs.txt trivy-fs.sarif || true'
            def ec = sh(returnStatus: true, script: '''
              set +e
              trivy fs --no-progress --exit-code 0 --severity HIGH,CRITICAL . | tee trivy-fs.txt
              trivy fs --no-progress --exit-code 0 --severity HIGH,CRITICAL --format sarif -o trivy-fs.sarif .
            ''')
            echo "Trivy FS scan exit code: ${ec}"
            if (env.FAIL_ON_ISSUES == 'true' && ec != 0) {
              error "Fail build (policy) if Trivy FS exit ${ec}"
            }
            sh 'ls -lh trivy-fs.* || true'
          }
        }
      }
      post {
        always {
          script {
            if (fileExists('trivy-fs.txt'))   { archiveArtifacts artifacts: 'trivy-fs.txt', fingerprint: true }
            if (fileExists('trivy-fs.sarif')) { archiveArtifacts artifacts: 'trivy-fs.sarif', fingerprint: true }
          }
        }
      }
    }

    // ===== SAST: Semgrep (pakai flag JUnit yang benar) =====
    stage('SAST - Semgrep') {
      agent {
        docker {
          image 'semgrep/semgrep:latest'
          reuseNode true
          args '--entrypoint='
        }
      }
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          sh '''
            set -eu
            semgrep --version || true

            # SARIF
            semgrep scan \
              --config p/ci --config p/owasp-top-ten --config p/docker \
              --exclude 'log/**' --exclude '**/node_modules/**' --exclude '**/dist/**' --exclude '**/build/**' \
              --severity error --error \
              --sarif -o semgrep.sarif || true

            # JUnit (flag yang benar)
            semgrep scan \
              --config p/ci --config p/owasp-top-ten --config p/docker \
              --exclude 'log/**' --exclude '**/node_modules/**' --exclude '**/dist/**' --exclude '**/build/**' \
              --severity error --error \
              --junit-xml -o semgrep-junit.xml || true
          '''
        }
      }
      post {
        always {
          script {
            if (fileExists('semgrep.sarif'))     { archiveArtifacts artifacts: 'semgrep.sarif', fingerprint: true }
            if (fileExists('semgrep-junit.xml')) { archiveArtifacts artifacts: 'semgrep-junit.xml', fingerprint: true }
          }
        }
      }
    }
  }

  post {
    always {
      script {
        if (fileExists('semgrep.sarif'))  { recordIssues(enabledForFailure: true, tools: [sarif(pattern: 'semgrep.sarif',   id: 'Semgrep')]) }
        if (fileExists('trivy-fs.sarif')) { recordIssues(enabledForFailure: true, tools: [sarif(pattern: 'trivy-fs.sarif', id: 'Trivy FS')]) }
        if (fileExists('semgrep-junit.xml')) {
          junit allowEmptyResults: false, testResults: 'semgrep-junit.xml'
        } else {
          echo "semgrep-junit.xml not found"
        }
      }
      echo "Scanning All Done. Result: ${currentBuild.currentResult}"
    }
  }
}
