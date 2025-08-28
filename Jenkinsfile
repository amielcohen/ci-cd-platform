pipeline {
  agent any

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        sh 'echo "Checked out $BRANCH_NAME"'
      }
    }

    stage('CI tests (PR)') {
      when { changeRequest() }
      agent { docker { image 'python:3.11' } } 
      steps {
        sh '''
          set -e
          python --version
          pip install --upgrade pip
          if [ -f requirements.txt ]; then
            pip install -r requirements.txt
          fi
          pip install pytest
          pytest -q --maxfail=1 --disable-warnings --junitxml=test-report.xml
        '''
      }
      post {
        always {
          junit 'test-report.xml'
          archiveArtifacts artifacts: 'test-report.xml', onlyIfSuccessful: false
        }
      }
    }

    stage('CD on master (placeholder)') {
      when { branch 'master' }  
      steps {
        sh 'echo "Deploy placeholder on master (נממש בחלק D)"'
      }
    }
  }

  post {
    always {
      echo "Pipeline finished for branch: ${env.BRANCH_NAME}"
    }
  }
}
