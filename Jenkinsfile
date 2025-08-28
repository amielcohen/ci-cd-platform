pipeline {
  agent any
  environment {
    AWS_REGION = 'us-east-1'
    APP_REPO   = 'ci-cd-platform'      
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('PR CI') {
      when { changeRequest() }           
      agent none
      stages {

        stage('Ensure ECR Repo Exists') {
          agent { docker { image 'amazon/aws-cli:2' } }
          steps {
            sh '''
              set -e
              aws ecr describe-repositories --repository-names "$APP_REPO" --region "$AWS_REGION" >/dev/null 2>&1 || \
              aws ecr create-repository --repository-name "$APP_REPO" --region "$AWS_REGION" >/dev/null
            '''
          }
        }

        stage('Build Image (PR)') {
          agent { docker { image 'docker:24.0-cli'; args '-v /var/run/docker.sock:/var/run/docker.sock' } }
          steps {
            sh '''
              set -e
              apk add --no-cache python3 py3-pip >/dev/null
              pip3 install --no-cache-dir awscli >/dev/null

              ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
              ECR_REGISTRY=$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
              IMAGE=$ECR_REGISTRY/$APP_REPO:pr-$CHANGE_ID-$BUILD_NUMBER

              aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY
              docker build -t $IMAGE .
              echo $IMAGE > .image_ref
              echo "Built image: $IMAGE"
            '''
          }
        }

        stage('Test (PR)') {
          agent { docker { image 'docker:24.0-cli'; args '-v /var/run/docker.sock:/var/run/docker.sock' } }
          steps {
            sh '''
              set -e
              docker run --rm -v "$PWD":/workspace -w /workspace python:3.11 /bin/sh -lc '
                set -e
                pip install --upgrade pip
                if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
                pip install pytest pytest-cov
                # הפקה של דוח JUnit כדי ש-Jenkins יציג תוצאות
                pytest -q --maxfail=1 --disable-warnings --junitxml=test-results.xml --cov=. --cov-report=term
              '
            '''
          }
          post {
            always {
              // מציג תוצאות בדשבורד ושומר אותן
              junit allowEmptyResults: true, testResults: 'test-results.xml'
              archiveArtifacts artifacts: 'test-results.xml', fingerprint: true, allowEmptyArchive: true
            }
          }
        }

        stage('Push to ECR (PR)') {
          agent { docker { image 'docker:24.0-cli'; args '-v /var/run/docker.sock:/var/run/docker.sock' } }
          steps {
            sh '''
              set -e
              IMAGE=$(cat .image_ref)
              ECR_REGISTRY=$(echo $IMAGE | cut -d/ -f1)
              aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY
              docker push $IMAGE
              echo "Pushed: $IMAGE"
            '''
          }
        }

      } // inner stages
    } // PR CI wrapper
  } // stages
}
