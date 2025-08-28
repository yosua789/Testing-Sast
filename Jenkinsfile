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
      script {
        sh '''
          set +e

          # 1) Output SARIF
          semgrep scan \
            --config p/ci --config p/owasp-top-ten --config p/docker \
            --exclude 'log/**' --exclude '**/node_modules/**' --exclude '**/dist/**' --exclude '**/build/**' \
            --severity ERROR --error \
            --sarif -o semgrep.sarif
          EC1=$?

          # 2) Output JUnit XML
          semgrep scan \
            --config p/ci --config p/owasp-top-ten --config p/docker \
            --exclude 'log/**' --exclude '**/node_modules/**' --exclude '**/dist/**' --exclude '**/build/**' \
            --severity ERROR --error \
            --junit-xml -o semgrep-junit.xml
          EC2=$?

          if [ $EC1 -ne 0 ] || [ $EC2 -ne 0 ]; then echo 1 > .semgrep_exit; else echo 0 > .semgrep_exit; fi
        '''
        def ec = readFile('.semgrep_exit').trim()
        sh 'ls -lh semgrep.sarif semgrep-junit.xml || true'
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
