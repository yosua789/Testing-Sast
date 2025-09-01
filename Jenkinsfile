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

    // ===== NEW: Build Java (creates classes for Sonar) =====
    stage('Build') {
      steps {
        script {
          sh '''
            set -eux

            # Detect build tool
            if [ -f pom.xml ]; then
              echo "Detected Maven build"
              docker run --rm --network jenkins \
                --volumes-from jenkins -w "$WORKSPACE" \
                maven:3-eclipse-temurin-17 \
                mvn -B -DskipTests=false clean test
              BUILD_TOOL=maven
            elif [ -f build.gradle ] || ls *.gradle >/dev/null 2>&1; then
              echo "Detected Gradle build"
              docker run --rm --network jenkins \
                --volumes-from jenkins -w "$WORKSPACE" \
                gradle:8.10.2-jdk17 \
                bash -lc "./gradlew clean test || gradle clean test"
              BUILD_TOOL=gradle
            else
              echo "No Maven/Gradle detected. Compiling with javac..."
              mkdir -p target/classes target/test-classes
              docker run --rm --network jenkins \
                --volumes-from jenkins -w "$WORKSPACE" \
                eclipse-temurin:17-jdk \
                bash -lc 'find src -type f -name "*.java" > .java-list || true; \
                          if [ -s .java-list ]; then javac -d target/classes @.java-list; else echo "No Java sources found"; fi'
              BUILD_TOOL=javac
            fi

            echo "Build tool = ${BUILD_TOOL}"

            echo "Sample of compiled classes (if any):"
            find target -name "*.class" | head -n 10 || true
          '''
        }
      }
    }

    // === SAST: SonarQube (Scanner CLI in container) ===
    stage('SAST - SonarQube') {
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          withCredentials([string(credentialsId: 'sonarqube-token', variable: 'T')]) {
            sh '''
              set -eux

              # Auth handling: user token (squ_) via SONAR_TOKEN env; project token (sqp_) via -Dsonar.token
              AUTH_ENV=""; AUTH_PROP=""
              if echo "$T" | grep -q '^squ_'; then
                AUTH_ENV="-e SONAR_TOKEN=$T"
              else
                AUTH_PROP="-Dsonar.token=$T"
              fi

              docker pull sonarsource/sonar-scanner-cli

              # Compute binary dirs if present
              JAVA_BIN_DIR=""; JAVA_TEST_BIN_DIR=""
              if [ -d target/classes ]; then JAVA_BIN_DIR="-Dsonar.java.binaries=target/classes"; fi
              if [ -d target/test-classes ]; then JAVA_TEST_BIN_DIR="-Dsonar.java.test.binaries=target/test-classes"; fi

              # If there are .java files but no classes, add a helpful message and still run (will fail, but stage is non-blocking)
              if find . -name "*.java" | grep -q . && [ ! -d target/classes ]; then
                echo "WARNING: Java sources detected but no compiled classes found. SonarJava may fail."
              fi

              # Run scanner
              set +e
              docker run --rm --network jenkins \
                --volumes-from jenkins \
                -w "$WORKSPACE" \
                -e SONAR_HOST_URL="$SONAR_HOST_URL" \
                $AUTH_ENV \
                sonarsource/sonar-scanner-cli \
                  -Dsonar.host.url="$SONAR_HOST_URL" \
                  $AUTH_PROP \
                  -Dsonar.projectKey="$SONAR_PROJECT_KEY" \
                  -Dsonar.projectName="$SONAR_PROJECT_NAME" \
                  -Dsonar.scm.provider=git \
                  -Dsonar.sources=. \
                  -Dsonar.inclusions="**/*" \
                  -Dsonar.exclusions="**/log/**,**/log4/**,**/log_3/**,**/*.test.*,**/node_modules/**,**/dist/**,**/build/**,docker-compose.yaml" \
                  -Dsonar.coverage.exclusions="**/*.test.*,**/test/**,**/tests**" \
                  ${JAVA_BIN_DIR} \
                  ${JAVA_TEST_BIN_DIR}
              rc=$?
              set -e
              echo $rc > .sonar_exit
              exit 0
            '''
            script {
              def rc = readFile('.sonar_exit').trim()
              echo "SonarScanner exit code: ${rc}"
              if (env.FAIL_ON_ISSUES == 'true' && rc != '0') {
                error "Fail build (policy) SonarScanner exit ${rc}"
              }
            }
          }
        }
      }
    }

    // === SCA: OWASP Dependency-Check (repo) ===
    stage('SCA - Dependency-Check (repo)') {
      agent {
        docker {
          image 'owasp/dependency-check:latest'
          reuseNode true
          args "--entrypoint=''"
        }
      }
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          script {
            sh '''
              set -eux
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
            if (fileExists('dependency-check-report')) {
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
            if (fileExists('trivy-fs.txt'))   { archiveArtifacts artifacts: 'trivy-fs.txt',   fingerprint: false }
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
            if (fileExists('semgrep.sarif'))      { archiveArtifacts artifacts: 'semgrep.sarif', fingerprint: false }
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
