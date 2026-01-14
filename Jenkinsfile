
pipeline {
  agent any

  environment {
    AWS_REGION     = 'us-east-1'
    AWS_ACCOUNT_ID = '206501439294'
    ECR_REPO       = 'ecstest'
    IMAGE_TAG      = "${BUILD_NUMBER}"
    ECR_REGISTRY   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
    ECR_URI        = "${ECR_REGISTRY}/${ECR_REPO}"
    ECS_CLUSTER    = 'ecsdemotest'
    ECS_SERVICE    = 'frontendnginx-service'
    CONTAINER_NAME = 'nginxfrontend'
  }

  stages {

    stage('Checkout') {
      steps {
        git branch: 'main', url: 'https://github.com/Amansh07/ecs-hmdm.git'
      }
    }

    stage('Verify Java & Maven') {
      steps {
        sh '''
          set -euo pipefail
          java -version
          mvn -version
          aws --version
          docker --version
          jq --version || { echo "jq not found. Please install jq on the agent."; exit 1; }
        '''
      }
    }

    stage('Build WAR') {
      steps {
        sh '''
          set -euo pipefail
          mvn clean package -DskipTests
          echo "Built artifacts:"
          find . -name "launcher.war" -type f -print || echo "launcher.war not found"
        '''
      }
    }

    stage('Docker Build & Push') {
      steps {
        sh '''
          set -euo pipefail

          # Login to ECR (non-interactive)
          aws ecr get-login-password --region "${AWS_REGION}" \
            | docker login --username AWS --password-stdin "${ECR_REGISTRY}"

          # Ensure repository exists (idempotent)
          aws ecr describe-repositories \
            --repository-names "${ECR_REPO}" \
            --region "${AWS_REGION}" >/dev/null 2>&1 \
            || aws ecr create-repository \
                 --repository-name "${ECR_REPO}" \
                 --region "${AWS_REGION}"

          # Build, tag, push
          docker build -t "${ECR_REPO}:${IMAGE_TAG}" .
          docker tag "${ECR_REPO}:${IMAGE_TAG}" "${ECR_URI}:${IMAGE_TAG}"
          docker push "${ECR_URI}:${IMAGE_TAG}"

          # Optional: print image digest
          aws ecr describe-images \
            --repository-name "${ECR_REPO}" \
            --image-ids imageTag="${IMAGE_TAG}" \
            --region "${AWS_REGION}" \
            --query 'imageDetails[0].imageDigest' --output text || true
        '''
      }
    }

    stage('Deploy to ECS') {
      steps {
        sh '''
          set -euo pipefail

          # Get current task definition ARN used by the service
          TD_ARN=$(aws ecs describe-services \
            --cluster "${ECS_CLUSTER}" \
            --services "${ECS_SERVICE}" \
            --region "${AWS_REGION}" \
            --query "services[0].taskDefinition" \
            --output text)

          # Fetch full task definition JSON
          aws ecs describe-task-definition \
            --task-definition "${TD_ARN}" \
            --region "${AWS_REGION}" \
            --query 'taskDefinition' \
            --output json > td.json

          # Update image for target container; clean fields disallowed on register
          jq '.containerDefinitions |=
                map(if .name=="'"${CONTAINER_NAME}"'"
                    then .image="'"${ECR_URI}:${IMAGE_TAG}"'"
                    else . end)
              | del(.revision,.status,.registeredAt,.compatibilities,.requiresAttributes,.taskDefinitionArn)' \
            td.json > td-new.json

          # Register new task definition
          NEW_TD_ARN=$(aws ecs register-task-definition \
            --cli-input-json file://td-new.json \
            --region "${AWS_REGION}" \
            --query 'taskDefinition.taskDefinitionArn' \
            --output text)

          echo "Registered new task definition: ${NEW_TD_ARN}"

          # Update service to use new task definition
          aws ecs update-service \
            --cluster "${ECS_CLUSTER}" \
            --service "${ECS_SERVICE}" \
            --region "${AWS_REGION}" \
            --task-definition "${NEW_TD_ARN}" \
            --force-new-deployment

          # Optional: wait until service is stable
          aws ecs wait services-stable \
            --cluster "${ECS_CLUSTER}" \
            --services "${ECS_SERVICE}" \
            --region "${AWS_REGION}"
        '''
      }
    }
  }
}
