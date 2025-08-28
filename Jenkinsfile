pipeline {
  agent any
  options {
    timestamps()
    ansiColor('xterm')
  }
  stages {
    stage('Checkout') {
      steps { checkout scm }
    }
    stage('Smoke') {
      steps {
        echo "Branch: ${env.BRANCH_NAME}"
        echo "PR ID: ${env.CHANGE_ID ?: 'none'}"
        sh 'echo Hello from Jenkins && uname -a'
      }
    }
  }
  post {
    always {
      echo "Done. Build: ${env.BUILD_TAG}"
    }
  }
}

