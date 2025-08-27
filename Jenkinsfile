pipeline {
  agent any
  options { skipDefaultCheckout(true) }

  environment {
    FAIL_ON_ISSUES     = 'false'                 // set 'true' untuk enforce Quality Gate & fail on issues
    SONAR_HOST_URL     = 'http://sonarqube:9000' // gunakan DNS container di network jenkins
    SONAR_PROJECT_KEY  = 'babi'
    SONAR_PROJECT_NAME = 'babi'
    DOCKER_NET         = 'jenkins'               // ganti jika nama network berbeda
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

    // Debug: tunjukkan ENV relevan yang masih mengandung 'demo-SAST'
    stage('Debug ENV demo-SAST (scope)') {
      steps {
        sh '''
          set -e
          echo "[debug] ENV relevan (SONAR_|SCANNER_|PROJECT_) yang mengandung 'demo-SAST':"
          printenv | grep -E '^(SONAR_|SCANNER_|PROJECT_)' | grep -n 'demo-SAST' || echo "Tidak ada."
        '''
      }
    }

    // Guard: pastikan tidak ada konfigurasi demo-SAST (kecuali teks di Jenkinsfile)
    stage('Guard: no demo-SAST') {
      steps {
        sh '''
          set -e
          echo "[guard] Cek konfigurasi repo (kecuali Jenkinsfile)…"
          ! grep -R --line-number --exclude-dir=.git --exclude=Jenkinsfile \
             -E '(-Dsonar\\.projectKey=demo-SAST|sonar\\.projectKey[[:space:]]*=[[:space:]]*demo-SAST)' . \
             || { echo "Masih ada konfigurasi demo-SAST di file selain Jenkinsfile!"; exit 1; }

          echo "[guard] Cek ENV yang relevan (SONAR_|SCANNER_|PROJECT_)…"
          BAD_ENV="$(printenv | grep -E '^(SONAR_|SCANNER_|PROJECT_)' | grep -F 'demo-SAST' || true)"
          if [ -n "$BAD_ENV" ]; then
            echo "[guard] Variabel bermasalah:"
            echo "$BAD_ENV"
            exit 1
          fi

          echo "[guard] OK"
        '''
      }
    }

    // === SAST: SonarQube (babi) ===
    stage('SAST - SonarQube (babi)') {
      steps {
        // Safety belt: paksa env Sonar benar walaupun ada ENV liar di Jenkins
        withEnv(['SONAR_PROJECT_KEY=babi','SONAR_PROJECT_NAME=babi']) {
          withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN_RAW')]) {
            sh '''
              set -euo pipefail

              # Trim newline
              SONAR_TOKEN="$(printf %s "$SONAR_TOKEN_RAW" | tr -d '\\r\\n')"

              echo "[assert] SONAR_PROJECT_KEY=$SONAR_PROJECT_KEY"
              test "$SONAR_PROJECT_KEY" = "babi" || { echo "ProjectKey bukan 'babi'!"; exit 1; }

              echo "[preflight] Server reachable?"
              docker run --rm --platform linux/arm64 --network "$DOCKER_NET" curlimages/curl:8.11.1 \
                -sS -o /dev/null -w "HTTP %{http_code}\\n" "$SONAR_HOST_URL/api/server/version"

              echo "[preflight] Token valid?"
              docker run --rm --platform linux/arm64 --network "$DOCKER_NET" -e SONAR_TOKEN="$SONAR_TOKEN" curlimages/curl:8.11.1 \
                -sSf -u "$SONAR_TOKEN:" "$SONAR_HOST_URL/api/authentication/validate" | grep -q '"valid":true'

              echo "[preflight] Token bisa akses project 'babi'?"
              docker run --rm --platform linux/arm64 --network "$DOCKER_NET" -e SONAR_TOKEN="$SONAR_TOKEN" curlimages/curl:8.11.1 \
                -sSf -u "$SONAR_TOKEN:" "$SONAR_HOST_URL/api/projects/search?projects=$SONAR_PROJECT_KEY" \
                | grep -q '"key":"babi"' || { echo "Token TIDAK bisa akses 'babi'"; exit 1; }

              echo "[scan] Pull scanner arm64…"
              docker pull --platform linux/arm64 sonarsource/sonar-scanner-cli:7.2.0.5079

              echo "[scan] Run sonar-scanner…"
              docker run --rm --platform linux/arm64 --network "$DOCKER_NET" \
                -v "$WORKSPACE:/usr/src" -w /usr/src \
                -v "$WORKSPACE/.sonar-cache:/root/.sonar/cache" \
                sonarsource/sonar-scanner-cli:7.2.0.5079 \
                  -X \
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
    }

    // (opsional) tunggu Quality Gate jika FAIL_ON_ISSUES='true'
    stage('SonarQube Quality Gate') {
      when { expression { return env.FAIL_ON_ISSUES == 'true' } }
      steps {
        withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN_RAW')]) {
          sh '''
            set -euo pipefail
            SONAR_TOKEN="$(printf %s "$SONAR_TOKEN_RAW" | tr -d '\\r\\n')"

            TASK_FILE=".scannerwork/report-task.txt"
            test -f "$TASK_FILE"
            CE_TASK_ID="$(grep -E '^ceTaskId=' "$TASK_FILE" | cut -d= -f2)"
            if [ -z "${CE_TASK_ID:-}" ]; then
              CE_TASK_ID="$(grep -E '^ceTaskUrl=' "$TASK_FILE" | sed -n 's/.*id=\\([^&]*\\).*/\\1/p')"
            fi
            echo "[gate] Background task: $CE_TASK_ID"

            for i in $(seq 1 120); do
              RESP="$(docker run --rm --platform linux/arm64 --network "$DOCKER_NET" curlimages/curl:8.11.1 \
                        -sSf -u "$SONAR_TOKEN:" "$SONAR_HOST_URL/api/ce/task?id=$CE_TASK_ID" || true)"
              STATUS="$(printf '%s' "$RESP" | sed -n 's/.*"status":"\\([^"]*\\)".*/\\1/p')"
              echo "[gate] CE status: ${STATUS:-unknown} ($i/120)"
              case "$STATUS" in
                SUCCESS|FAILED|CANCELED) break ;;
              esac
              sleep 3
            done

            [ "$STATUS" = "SUCCESS" ] || { echo "[gate] CE status: $STATUS"; exit 1; }

            QG="$(docker run --rm --platform linux/arm64 --network "$DOCKER_NET" curlimages/curl:8.11.1 \
                  -sSf -u "$SONAR_TOKEN:" \
                  "$SONAR_HOST_URL/api/qualitygates/project_status?projectKey=$SONAR_PROJECT_KEY")"
            printf '%s\n' "$QG" > quality-gate.json
            QSTATUS="$(printf '%s' "$QG" | sed -n 's/.*"status":"\\([^"]*\\)".*/\\1/p')"
            echo "[gate] Quality Gate: $QSTATUS"
            [ "$QSTATUS" = "OK" ] || { echo "Quality Gate failed: $QSTATUS"; exit 1; }
          '''
          archiveArtifacts artifacts: 'quality-gate.json', fingerprint: true
        }
      }
    }

    // === SCA: OWASP Dependency-Check ===
    stage('SCA - Dependency-Check (repo)') {
      agent {
        docker {
          image 'owasp/dependency-check:9.1.0'
          reuseNode true
          args "-v $WORKSPACE/.odc:/usr/share/dependency-check/data -v $WORKSPACE/.odc-temp:/tmp"
        }
      }
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          script {
            sh '''
              set -e
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
              publishHTML(target: [reportDir: 'dependency-check-report', reportFiles: 'dependency-check-report.html', reportName: 'Dependency-Check Report'])
            } else {
              echo "Dependency-Check HTML report not found"
            }
          }
        }
      }
    }

    // === SCA: Trivy (filesystem) ===
    stage('SCA - Trivy (filesystem)') {
      agent {
        docker {
          image 'aquasec/trivy:0.54.1'
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

    // === SAST: Semgrep ===
    stage('SAST - Semgrep') {
      agent {
        docker {
          image 'semgrep/semgrep:1.95.0'
          reuseNode true
          args '--entrypoint='
        }
      }
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          script {
            sh '''
              set +e
              semgrep --version || true
              semgrep scan \
                --config p/ci --config p/owasp-top-ten --config p/docker \
                --exclude 'log/**' --exclude '**/node_modules/**' --exclude '**/dist/**' --exclude '**/build/**' \
                --sarif -o semgrep.sarif \
                --junit-xml --junit-xml-file semgrep-junit.xml \
                --severity error --error
              EC=$?; echo $EC > .semgrep_exit
            '''
            def ec = readFile('.semgrep_exit').trim()
            sh 'ls -lh semgrep.* || true'
            if (env.FAIL_ON_ISSUES == 'true' && ec != '0') {
              error "Fail build (policy) Semgrep exit ${ec}"
            }
          }
        }
      }
      post {
        always {
          archiveArtifacts artifacts: 'semgrep.sarif, semgrep-junit.xml', fingerprint: true, onlyIfSuccessful: false
        }
      }
    }
  }

  post {
    always {
      script {
        if (fileExists('semgrep.sarif'))   { recordIssues(enabledForFailure: true, tools: [sarif(pattern: 'semgrep.sarif',   id: 'Semgrep')]) }
        if (fileExists('trivy-fs.sarif'))  { recordIssues(enabledForFailure: true, tools: [sarif(pattern: 'trivy-fs.sarif', id: 'Trivy FS')]) }
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
