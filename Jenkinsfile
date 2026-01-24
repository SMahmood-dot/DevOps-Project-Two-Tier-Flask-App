pipeline {
  agent any

  options { skipDefaultCheckout(true) }

  environment {
    AWS_REGION = "us-east-2"
    ECR_REPO   = "flask-app"
  }

  stages {

    stage("Checkout") {
      steps {
        checkout scm
      }
    }

stage("DEBUG: show Jenkinsfile + HEAD") {
  steps {
    sh '''
      set -e
      echo "----- Jenkinsfile (first 120 lines) -----"
      sed -n '1,120p' Jenkinsfile
      echo "----- git HEAD -----"
      git rev-parse HEAD
      git log -1 --oneline
    '''
  }
}

stage("Resolve ECR Image URI") {
  steps {
    sh '''
      set -e
      AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
      ECR_REGISTRY="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"

      IMAGE_TAG=$(git rev-parse --short=8 HEAD)
      IMAGE_URI="$ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG"

      echo "AWS_REGION=$AWS_REGION" > aws.env
      echo "ECR_REGISTRY=$ECR_REGISTRY" >> aws.env
      echo "IMAGE_URI=$IMAGE_URI" >> aws.env

      cat aws.env
    '''
  }
}


    stage("ECR Login") {
      steps {
        sh '''
          set -e
          . ./aws.env
          aws ecr get-login-password --region "$AWS_REGION" | docker login --username AWS --password-stdin "$ECR_REGISTRY"
        '''
      }
    }

    stage("Build Docker Image") {
      steps {
        sh '''
          set -e
          . ./aws.env
          docker build -t "$IMAGE_URI" .
        '''
      }
    }

    stage("Push Image to ECR") {
      steps {
        sh '''
          set -e
          . ./aws.env
          docker push "$IMAGE_URI"
        '''
      }
    }

    stage("Deploy from ECR (docker compose)") {
      steps {
        sh '''
          set -e
          . ./aws.env
          export ECR_IMAGE="$IMAGE_URI"

          docker compose down || true
          docker compose up -d

          docker ps
        '''
      }
    }

    stage("Health Check") {
      steps {
        sh '''
          set -e
          sleep 6
          curl -f http://localhost:5000/ || (docker compose logs --tail=200 && exit 1)
        '''
      }
    }

  }
}
