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

    // ===== Preflight: cek token & project =====
    stage('Sonar token preflight') {
      steps {
        withCredentials([string(credentialsId: 'sonarqube-token', variable: 'T')]) {
          sh '''
            set -eux
            echo -n "$T" | wc -c | awk '{print "Token length:", $1}'
            echo -n "$T" | sha256sum | awk '{print "SHA256(JenkinsCred)="$1}'

            # versi API v2 -> harus 200 dan keluar angka
            docker run --rm --network jenkins curlimages/curl -fsS \
              -H "Authorization: Bearer $T" \
              "$SONAR_HOST_URL/api/v2/analysis/version" \
              | xargs echo "APIv2 version:"

            # cari project -> harus ketemu
            docker run --rm --network jenkins curlimages/curl -fsS \
              -H "Authorization: Bearer $T" \
              "$SONAR_HOST_URL/api/projects/search?projects=$SONAR_PROJECT_KEY" \
              | tee .sonar_project_search.json >/dev/null
            grep -q '"key":"'$SONAR_PROJECT_KEY'"' .sonar_project_search.json
          '''
        }
      }
    }

    // ===== SAST: SonarQube =====
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
          script {
            def rc = readFile('.dc_exit').trim()
            echo "Dependency-Check exit code: ${rc}"
            if (env.FAIL_ON_ISSUES == 'true' && rc != '0') {
              error "Fail build (policy) DC exit ${rc}"
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

    // ===== SCA: Trivy FS =====
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
          sh '''
            set +e
            rm -f trivy-fs.txt trivy-fs.sarif || true
            trivy fs --no-progress --exit-code 0 --severity HIGH,CRITICAL . | tee trivy-fs.txt
            trivy fs --no-progress --exit-code 0 --severity HIGH,CRITICAL --format sarif -o trivy-fs.sarif .
            echo $? > .trivy_exit
          '''
          script {
            echo "Trivy FS scan exit code: " + readFile('.trivy_exit').trim()
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

    // ===== SAST: Semgrep (SARIF & JUnit) =====
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
            set +e
            semgrep --version || true

            # Run 1: SARIF
            semgrep scan \
              --config p/ci --config p/owasp-top-ten --config p/docker \
              --exclude 'log/**' --exclude '**/node_modules/**' --exclude '**/dist/**' --exclude '**/build/**' \
              --sarif --output semgrep.sarif \
              --severity error --error
            EC1=$?

            # Run 2: JUnit XML
            semgrep scan \
              --config p/ci --config p/owasp-top-ten --config p/docker \
              --exclude 'log/**' --exclude '**/node_modules/**' --exclude '**/dist/**' --exclude '**/build/**' \
              --junit-xml --output semgrep-junit.xml \
              --severity error --error
            EC2=$?

            echo $EC1 > .semgrep_ec1
            echo $EC2 > .semgrep_ec2

            ls -lh semgrep.* || true
          '''
          script {
            def ec1 = readFile('.semgrep_ec1').trim()
            def ec2 = readFile('.semgrep_ec2').trim()
            echo "Semgrep exit codes: SARIF=${ec1}, JUNIT=${ec2}"
            if (env.FAIL_ON_ISSUES == 'true' && (ec1 != '0' || ec2 != '0')) {
              error "Fail build (policy) Semgrep exit SARIF=${ec1} JUNIT=${ec2}"
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
