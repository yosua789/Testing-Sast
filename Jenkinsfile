pipeline {
  agent any
  options { skipDefaultCheckout(true) }

  environment {
    // infra
    DOCKER_NET             = 'jenkins'
    JENKINS_CONTAINER_NAME = 'jenkins'

    // SonarQube
    SONAR_HOST_URL     = 'http://sonarqube:9000'
    SONAR_PROJECT_KEY  = 'coba'
    SONAR_PROJECT_NAME = 'coba'

    // policy
    FAIL_ON_ISSUES     = 'false'
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

    stage('SAST - SonarQube') {
      steps {
        withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
          sh '''
            set -eux

            # Tentukan cara auth:
            # - sqp_*  => Basic token (sonar.token)
            # - squ_*  => Bearer token (Authorization header)
            EXTRA_AUTH=""
            if echo "$SONAR_TOKEN" | grep -q '^squ_'; then
              # per RFC, spasi di property harus di-escape %20
              EXTRA_AUTH="-Dsonar.http.headers=Authorization=Bearer%20$SONAR_TOKEN"
            else
              EXTRA_AUTH="-Dsonar.token=$SONAR_TOKEN"
            fi

            docker pull sonarsource/sonar-scanner-cli

            docker run --rm \
              --network "$DOCKER_NET" \
              --volumes-from "$JENKINS_CONTAINER_NAME" \
              -w "$WORKSPACE" \
              -e SONAR_HOST_URL="$SONAR_HOST_URL" \
              sonarsource/sonar-scanner-cli \
                -Dsonar.host.url="$SONAR_HOST_URL" \
                $EXTRA_AUTH \
                -Dsonar.projectKey="$SONAR_PROJECT_KEY" \
                -Dsonar.projectName="$SONAR_PROJECT_NAME" \
                -Dsonar.scm.provider=git \
                -Dsonar.sources=. \
                -Dsonar.inclusions="**/*" \
                -Dsonar.exclusions="**/log/**,**/log4/**,**/log_3/**,**/*.test.*,**/node_modules/**,**/dist/**,**/build/**,docker-compose.yaml" \
                -Dsonar.coverage.exclusions="**/*.test.*,**/test/**,**/tests**"
          '''
        }
      }
    }

    stage('SCA - Dependency-Check (repo)') {
      agent {
        docker {
          image 'owasp/dependency-check:latest'
          reuseNode true
          // kosongkan entrypoint; plugin Jenkins otomatis tambahkan --volumes-from <jenkins-cid>
          args "--entrypoint='' -v $WORKSPACE/.odc:/usr/share/dependency-check/data -v $WORKSPACE/.odc-temp:/tmp"
        }
      }
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          script {
            sh '''
              set -eu
              mkdir -p dependency-check-report

              # update DB (toleransi error jaringan)
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

    stage('SCA - Trivy (filesystem)') {
      agent {
        docker {
          image 'aquasec/trivy:latest'
          reuseNode true
          // fix izin cache: taruh di /tmp, map ke workspace agar persist
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
            if (fileExists('semgrep.sarif'))      { archiveArtifacts artifacts: 'semgrep.sarif', fingerprint: false }
            if (fileExists('semgrep-junit.xml'))  {
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
