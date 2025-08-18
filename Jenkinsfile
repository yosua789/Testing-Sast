pipeline {
  agent any

  stages {
    stage('SCA Scan') {
      steps {
        sh '''
          dependency-check.sh \
            --project MyApp \
            --scan . \
            --format HTML \
            --out dependency-check-report
        '''
      }
    }
    stage('Publish Report') {
      steps {
        publishHTML([allowMissing: false,
          alwaysLinkToLastBuild: true,
          keepAll: true,
          reportDir: 'dependency-check-report',
          reportFiles: 'dependency-check-report.html',
          reportName: 'Dependency Check Report'])
      }
    }
  }
}
