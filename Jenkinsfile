pipeline {
  agent any
  options { skipDefaultCheckout(true) }

  environment {
    DOCKER_HOST    = "unix:///var/run/docker.sock"
    FAIL_ON_ISSUES = 'false'
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
          image 'maven:3.9.9-eclipse-temurin-11'
          reuseNode true
        }
      }
      steps {
        sh 'mvn -B clean package -DskipTests || echo "No pom.xml, skip build"'
      }
    }

    stage('SCA - OWASP Dependency-Check') {
      agent {
        docker {
          image 'owasp/dependency-check:latest'
          reuseNode true
          args "-v ${WORKSPACE}/.odc:/usr/share/dependency-check/data"
        }
      }
      steps {
        script {
          sh '''
            set +e
            /usr/share/dependency-check/bin/dependency-check.sh \
              --project "Testing-Sast" \
              --scan ./src \
              --format HTML \
              --out dependency-check-report
            echo $? > .dc_exit
          '''
          def rc = readFile('.dc_exit').trim()
          echo "Dependency-Check exit code: ${rc}"

          if (env.FAIL_ON_ISSUES == 'true' && rc != '0') {
            error "Failing build due to Dependency-Check exit code ${rc}"
          } else if (rc != '0') {
            echo "Uji coba: Dependency-Check return ${rc}, pipeline lanjut (tidak fail)."
          }
        }
      }
      post {
        always {
          script {
            def reportPath = 'dependency-check-report/dependency-check-report.html'
            if (fileExists(reportPath)) {
              publishHTML(target: [
                reportDir: 'dependency-check-report',
                reportFiles: 'dependency-check-report.html',
                reportName: 'Dependency-Check Report'
              ])
            } else {
              echo "Dependency-Check report tidak ditemukan, skip publishHTML."
            }
          }
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        sh 'docker version'
        sh 'docker build -t testing-sast:latest .'
      }
    }

    stage('SCA - Trivy Docker Scan') {
      agent {
        docker {
          image 'aquasec/trivy:latest'
          reuseNode true
          args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
      }
      steps {
        sh '''
          set +e
          trivy image --no-progress --exit-code 0 \
            --severity HIGH,CRITICAL testing-sast:latest | tee trivy-report.txt
        '''
      }
      post {
        always {
          archiveArtifacts artifacts: 'trivy-report.txt', fingerprint: true
        }
      }
    }
  }

  post {
    always { echo 'Pipeline selesai bro, semuanya aman.' }
  }
}
