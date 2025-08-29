pipeline {
  agent any
  options { skipDefaultCheckout(true); timestamps() }

  environment {
    FAIL_ON_ISSUES     = 'false'

    // --- SonarQube ---
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

    // =========================
    // Preflight: token & akses
    // =========================
    stage('Sonar token preflight') {
      steps {
        withCredentials([string(credentialsId: 'sonarqube-token', variable: 'T')]) {
          sh '''
            set -euxo pipefail

            # Info token (cek panjang & fingerprint supaya yakin ini token yang bener)
            echo -n "$T" | wc -c | awk '{print "Token length:", $1}'
            echo -n "$T" | sha256sum | awk '{print "SHA256(JenkinsCred)="$1}'

            # 1) Basic auth validate (ini pasti support Basic)
            docker run --rm --network jenkins curlimages/curl -fsS \
              -u "$T:" "$SONAR_HOST_URL/api/authentication/validate" \
              | tee .validate.json >/dev/null
            grep -q '"valid":true' .validate.json

            # 2) Bearer ke API v2 (HEADER DIKUTIP!)
            docker run --rm --network jenkins curlimages/curl -fsS \
              -H "Authorization: Bearer $T" \
              "$SONAR_HOST_URL/api/v2/analysis/version" \
              | xargs echo "APIv2 version:"

            # 3) Token lihat project
            docker run --rm --network jenkins curlimages/curl -fsS \
              -H "Authorization: Bearer $T" \
              "$SONAR_HOST_URL/api/projects/search?projects=$SONAR_PROJECT_KEY" \
              | tee .proj.json >/dev/null
            grep -q "\\\"key\\\":\\\"$SONAR_PROJECT_KEY\\\"" .proj.json
          '''
        }
      }
    }

    // =========================
    // SAST: SonarQube
    // =========================
    stage('SAST - SonarQube') {
      steps {
        withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
          sh '''
            set -euxo pipefail
            docker pull sonarsource/sonar-scanner-cli

            # NOTE: pakai env SONAR_HOST_URL & SONAR_TOKEN saja (tanpa -Dsonar.token biar gak override warning)
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

    // =========================
    // SCA: OWASP Dependency-Check
    // =========================
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
          sh '''
            set -euxo pipefail
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
          script {
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
                reportDir : 'dependency-check-report',
                reportFiles: 'dependency-check-report.html',
                reportName: 'Dependency-Check Report'
              ])
            } else {
              echo "Dependency-Check HTML report tidak ditemukan."
            }
          }
        }
      }
    }

    // =========================
    // SCA: Trivy (filesystem)
    // =========================
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
          sh 'rm -f trivy-fs.txt trivy-fs.sarif || true'
          script {
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

    // =========================
    // SAST: Semgrep (fix JUnit output)
    // =========================
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
            set -euxo pipefail
            semgrep --version || true

            # Output SARIF -> file
            set +e
            semgrep scan \
              --config p/ci --config p/owasp-top-ten --config p/docker \
              --exclude 'log/**' --exclude '**/node_modules/**' --exclude '**/dist/**' --exclude '**/build/**' \
              --sarif --output semgrep.sarif \
              --severity error --error
            ec1=$?

            # Output JUnit -> file (pakai --junit-xml + --output)
            semgrep scan \
              --config p/ci --config p/owasp-top-ten --config p/docker \
              --exclude 'log/**' --exclude '**/node_modules/**' --exclude '**/dist/**' --exclude '**/build/**' \
              --junit-xml --output semgrep-junit.xml \
              --severity error --error
            ec2=$?

            # Simpan exit code gabungan (biar bisa policy gate opsional)
            test $ec1 -eq 0 -a $ec2 -eq 0; echo $? > .semgrep_exit
            ls -lh semgrep.* || true
          '''
          script {
            def ec = readFile('.semgrep_exit').trim()
            if (env.FAIL_ON_ISSUES == 'true' && ec != '0') {
              error "Fail build (policy) karena Semgrep exit ${ec}"
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
        if (fileExists('semgrep.sarif'))  { recordIssues(enabledForFailure: true, tools: [sarif(pattern: 'semgrep.sarif',  id: 'Semgrep')]) }
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
