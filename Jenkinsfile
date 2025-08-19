pipeline {
  agent any
  options { skipDefaultCheckout(true) }

  stages {
    stage('Checkout') {
      steps {
        checkout([$class: 'GitSCM',
          branches: [[name: '*/main']],
          extensions: [[$class: 'CloneOption', shallow: false, noTags: false]],
          userRemoteConfigs: [[url: 'https://github.com/yosua789/Testing-Sast.git']]
        ])
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
        sh '''
          /usr/share/dependency-check/bin/dependency-check.sh \
            --project "Testing-Sast" \
            --scan ./src \
            --format HTML \
            --out dependency-check-report
        '''
      }
      post {
        always {
          publishHTML(target: [
            reportDir: 'dependency-check-report',
            reportFiles: 'dependency-check-report.html',
            reportName: 'Dependency-Check Report'
          ])
        }
      }
    }

    stage('Build Docker Image') {
      steps {
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
        sh 'trivy image --no-progress --exit-code 0 --severity HIGH,CRITICAL testing-sast:latest | tee trivy-report.txt'
      }
      post {
        always { archiveArtifacts artifacts: 'trivy-report.txt', fingerprint: true }
      }
    }
  }
}
