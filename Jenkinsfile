pipeline {
  agent any
  environment {
    AWS_REGION = 'us-east-1'
    
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('PR CI') {
      when { changeRequest() }
      agent none
      stages {

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
                pip install pytest
                pytest -q --maxfail=1 --disable-warnings
              '
            '''
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

      } // end inner stages
    } // end PR CI wrapper
  } // end stages
}
