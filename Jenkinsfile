pipeline {
  agent any
  options { skipDefaultCheckout(true) }

  environment {
    FAIL_ON_ISSUES     = 'false'

    // SonarQube
    SONAR_HOST_URL     = 'http://sonarqube:9000'
    SONAR_PROJECT_KEY  = 'central dashboard monitoring'
    SONAR_PROJECT_NAME = 'central dashboard monitoring'
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

    // === Build (compile Java so SonarJava can analyze bytecode) ===
    stage('Build') {
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          sh '''
            set -eux

            # If no Maven/Gradle, create a tiny pom with required deps (servlet/slf4j/esapi/javassist)
            if [ ! -f pom.xml ] && [ ! -f build.gradle ] && [ ! -f gradlew ]; then
              cat > pom.xml <<'POM'
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>tmp.bootstrap</groupId>
  <artifactId>testing-sast-bootstrap</artifactId>
  <version>1.0.0</version>
  <properties>
    <maven.compiler.source>8</maven.compiler.source>
    <maven.compiler.target>8</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>
  <dependencies>
    <dependency><groupId>javax.servlet</groupId><artifactId>javax.servlet-api</artifactId><version>4.0.1</version><scope>provided</scope></dependency>
    <dependency><groupId>org.slf4j</groupId><artifactId>slf4j-api</artifactId><version>1.7.36</version></dependency>
    <dependency><groupId>org.owasp.esapi</groupId><artifactId>esapi</artifactId><version>2.5.0.0</version></dependency>
    <dependency><groupId>org.javassist</groupId><artifactId>javassist</artifactId><version>3.29.2-GA</version></dependency>
  </dependencies>
  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.11.0</version>
        <configuration><release>8</release></configuration>
      </plugin>
    </plugins>
  </build>
</project>
POM
            fi

            if [ -f pom.xml ]; then
              echo "Building with Maven"
              docker run --rm --network jenkins --volumes-from jenkins -w "$WORKSPACE" \
                maven:3-eclipse-temurin-17 mvn -B -DskipTests=true clean compile test-compile
            elif [ -f build.gradle ] || [ -f gradlew ]; then
              echo "Building with Gradle"
              docker run --rm --network jenkins --volumes-from jenkins -w "$WORKSPACE" \
                gradle:8.10.2-jdk17 bash -lc './gradlew clean build -x test || gradle clean build -x test'
            else
              echo "Fallback: plain javac (best-effort)"
              mkdir -p target/classes target/test-classes
              docker run --rm --network jenkins --volumes-from jenkins -w "$WORKSPACE" \
                eclipse-temurin:17-jdk bash -lc 'find src -type f -name "*.java" > .java-list || true; if [ -s .java-list ]; then javac -d target/classes @.java-list || true; fi'
            fi

            echo "Sample compiled classes:"
            find target -name "*.class" | head -n 20 || true
          '''
        }
      }
    }

    // === SAST: SonarQube (pakai container SonarScanner ephemeral) ===
    stage('SAST - SonarQube') {
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          withCredentials([string(credentialsId: 'sonarqube-token', variable: 'T')]) {
            sh '''
              set -eux

              # Token handling:
              # - "squ_..." user token via ENV SONAR_TOKEN
              # - "sqp_..." project analysis token via -Dsonar.token
              AUTH_ENV=""; AUTH_PROP=""
              if echo "$T" | grep -q '^squ_'; then AUTH_ENV="-e SONAR_TOKEN=$T"; else AUTH_PROP="-Dsonar.token=$T"; fi

              docker pull sonarsource/sonar-scanner-cli

              # Compute Java flags if we have bytecode
              EXTRA_JAVA_FLAGS=""
              [ -d target/classes ] && EXTRA_JAVA_FLAGS="$EXTRA_JAVA_FLAGS -Dsonar.java.binaries=target/classes"
              [ -d target/test-classes ] && EXTRA_JAVA_FLAGS="$EXTRA_JAVA_FLAGS -Dsonar.java.test.binaries=target/test-classes"

              # If Java sources exist but no classes, exclude *.java to avoid analyzer crash
              if find src -name "*.java" 2>/dev/null | grep -q . && [ ! -d target/classes ]; then
                echo "WARNING: Java sources present but no compiled classes; excluding **/*.java for this run."
                EXTRA_JAVA_FLAGS="$EXTRA_JAVA_FLAGS -Dsonar.exclusions=**/*.java"
              fi

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
                  ${EXTRA_JAVA_FLAGS}
              rc=$?
              set -e
              echo $rc > .sonar_exit
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
              # Update DB (kalau lama/timeout, biarin lanjut)
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
              junit allowEmptyResults: true, testResults: 'semgrep-junit.xml', skipPublishingChecks: true, skipMarkingBuildUnstable: true
            } else {
              echo "semgrep-junit.xml not found"
            }
            if (env.FAIL_ON_ISSUES != 'true' && currentBuild.result == 'UNSTABLE') {
              currentBuild.result = 'SUCCESS'
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
