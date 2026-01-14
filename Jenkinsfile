
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

          # Build the project
          mvn clean package -DskipTests

          echo "Locating built WAR(s):"
          WAR_PATH="$(find . -name "*.war" -type f | head -n1 || true)"
          if [ -z "${WAR_PATH}" ]; then
            echo "ERROR: No WAR produced by the build. Check your pom.xml packaging and module path."
            exit 1
          fi
          echo "Found WAR: ${WAR_PATH}"

          # Ensure Docker build context has a 'target' dir with the WAR at a known path
          mkdir -p target
          cp -f "${WAR_PATH}" target/ROOT.war
          ls -l target
        '''
      }
    }

    stage('Docker Build & Push') {
      steps {
        sh '''
          set -euo pipefail

          # Resolve docker command; fallback to sudo if socket permission is denied
          DOCKER="docker"
          if ! ${DOCKER} info >/dev/null 2>&1; then
            echo "docker info failed; retrying with sudo docker"
            DOCKER="sudo docker"
          fi

          echo "Logging into ECR registry: ${ECR_REGISTRY}"
          aws ecr get-login-password --region "${AWS_REGION}" \
            | ${DOCKER} login --username AWS --password-stdin "${ECR_REGISTRY}"

          # Ensure repository exists (idempotent)
          aws ecr describe-repositories \
            --repository-names "${ECR_REPO}" \
            --region "${AWS_REGION}" >/dev/null 2>&1 \
            || aws ecr create-repository \
                 --repository-name "${ECR_REPO}" \
                 --region "${AWS_REGION}"

          # Build, tag, push
          ${DOCKER} build -t "${ECR_REPO}:${IMAGE_TAG}" .
          ${DOCKER} tag "${ECR_REPO}:${IMAGE_TAG}" "${ECR_URI}:${IMAGE_TAG}"
          ${DOCKER} push "${ECR_URI}:${IMAGE_TAG}"

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

          if [ -z "${TD_ARN}" ] || [ "${TD_ARN}" = "None" ]; then
            echo "ERROR: Could not resolve current task definition ARN for service ${ECS_SERVICE}."
            exit 1
          fi
          echo "Current task definition: ${TD_ARN}"

          # Fetch full task definition JSON
          aws ecs describe-task-definition \
            --task-definition "${TD_ARN}" \
            --region "${AWS_REGION}" \
            --query 'taskDefinition' \
            --output json > td.json

          # Update image for the target container; remove fields not accepted by register-task-definition
          jq '
            (.containerDefinitions) |=
              map(if .name=="'"${CONTAINER_NAME}"'"
                  then .image="'"${ECR_URI}:${IMAGE_TAG}"'"
                  else . end)
            | del(
                .revision,
                .status,
                .registeredAt,
                .taskDefinitionArn,
                .requiresAttributes,
                .compatibilities,
                .registeredBy
              )
          ' td.json > td-new.json

          # Sanity check before register
          test -s td-new.json || { echo "ERROR: td-new.json is empty."; exit 1; }
          echo "Preview of td-new.json:"
          jq . td-new.json | sed -n '1,120p'

          # Register new task definition
          NEW_TD_ARN=$(aws ecs register-task-definition \
            --cli-input-json file://td-new.json \
            --region "${AWS_REGION}" \
            --query 'taskDefinition.taskDefinitionArn' \
            --output text)

          if [ -z "${NEW_TD_ARN}" ] || [ "${NEW_TD_ARN}" = "None" ]; then
            echo "ERROR: Failed to register new task definition."
            exit 1
          fi
          echo "Registered new task definition: ${NEW_TD_ARN}"

          # Update service to use new task definition
          aws ecs update-service \
            --cluster "${ECS_CLUSTER}" \
            --service "${ECS_SERVICE}" \
            --region "${AWS_REGION}" \
            --task-definition "${NEW_TD_ARN}" \
            --force-new-deployment

          # Wait until service is stable
          aws ecs wait services-stable \
            --cluster "${ECS_CLUSTER}" \
            --services "${ECS_SERVICE}" \
            --region "${AWS_REGION}"

          echo "Deployment complete and service is stable."
        '''
      }
    }
  }
}
