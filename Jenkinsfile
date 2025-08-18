pipeline {
  agent any

  environment {
    SNYK_SEVERITY  = 'high'
    FAIL_ON_ISSUES = 'false'
  }

  options { timestamps() }

  stages {
    stage('Checkout') { steps { checkout scm } }

    stage('SCA: Snyk (console only)') {
      steps {
        withCredentials([string(credentialsId: 'SNYK_TOKEN', variable: 'SNYK_TOKEN')]) {
          sh """
            set +e
            docker run --rm \
              -e SNYK_TOKEN=$SNYK_TOKEN \
              -v "\$PWD:/work" -w /work \
              snyk/snyk:latest snyk test \
                --all-projects \
                --exclude=**/node_modules/**,**/.venv/**,**/vendor/**,**/build/**,**/dist/** \
                --severity-threshold=${SNYK_SEVERITY}
            SNYK_EXIT=\$?
            echo "Snyk exit code: \$SNYK_EXIT"
            if [ "\${FAIL_ON_ISSUES}" = "true" ]; then exit \$SNYK_EXIT; else exit 0; fi
          """
        }
      }
    }

    stage('SCA: SonarQube') {
      steps {
        withCredentials([string(credentialsId: 'sonarqube-token', variable: 'sonarqube-token')]) {
          sh """
            docker run --rm \
              -e SONAR_HOST_URL=http://localhost:9010 \
              -e SONAR_LOGIN=$ sonarqube-token \
              -v "\$PWD:/usr/src" \
              sonarsource/sonar-scanner-cli
          """
        }
      }
    }
  }
}
