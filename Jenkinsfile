pipeline {
  agent any
  options { timestamps(); ansiColor('xterm'); buildDiscarder(logRotator(numToKeepStr: '20')) }
  environment {
    AWS_REGION   = 'us-east-1'
    AWS_ACCOUNT  = '992382545251'
    ECR_REPO     = 'ci-cd-platform'
    APP_NAME     = 'platform-app'
    COMMIT_SHORT = "${env.GIT_COMMIT?.take(7)}"
  }
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
          python --version
          pip install --no-cache-dir --upgrade pip
          if [ -f requirements.txt ]; then pip install --no-cache-dir -r requirements.txt; fi
          pip install --no-cache-dir pytest
          pytest -q --maxfail=1 --disable-warnings --junitxml=test-report.xml
        '''
      }
      post {
        always {
          junit allowEmptyResults: true, testResults: 'test-report.xml'
          archiveArtifacts artifacts: 'test-report.xml', onlyIfSuccessful: false
        }
      }
    }

    stage('Docker Build') {
      when { branch 'master' }
      steps {
        script {
          env.IMAGE_TAG = "master-${env.BUILD_NUMBER}-${env.COMMIT_SHORT}"
        }
        sh 'docker build -t ${APP_NAME}:${IMAGE_TAG} .'
      }
    }

    stage('Login & Create ECR (if needed)') {
      when { branch 'master' }
      steps {
        sh '''
          aws ecr describe-repositories --repository-names ${ECR_REPO} --region ${AWS_REGION} >/dev/null 2>&1 || \
            aws ecr create-repository --repository-names ${ECR_REPO} --region ${AWS_REGION}
          aws ecr get-login-password --region ${AWS_REGION} | \
            docker login --username AWS --password-stdin ${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com
        '''
      }
    }

    stage('Tag & Push') {
      when { branch 'master' }
      steps {
        sh '''
          ECR=${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}
          docker tag ${APP_NAME}:${IMAGE_TAG} $ECR:${IMAGE_TAG}
          docker push $ECR:${IMAGE_TAG}
          docker tag $ECR:${IMAGE_TAG} $ECR:latest
          docker push $ECR:latest
        '''
      }
    }
  }
  post { always { echo "Branch=${env.BRANCH_NAME} ImageTag=${env.IMAGE_TAG}" } }
}
