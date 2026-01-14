pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        AWS_ACCOUNT_ID = '206501439294'
        ECR_REPO = 'ecstest'
        IMAGE_TAG = "${BUILD_NUMBER}"
        ECR_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
        ECS_CLUSTER = 'ecsdemotest'
        ECS_SERVICE = 'frontendnginx-service'
        CONTAINER_NAME = 'nginxfrontend'
    }

    tools {
        maven 'M3'        // Jenkins → Global Tool Config
        jdk 'Java11'
    }

    stages {

        stage('Checkout Source') {
            steps {
                git branch: 'main',
                    url: 'hhttps://github.com/Amansh07/ecs-hmdm.git'
            }
        }

        stage('Build WAR using Maven') {
            steps {
                sh '''
                mvn clean package -DskipTests
                ls -lh target
                '''
            }
        }

        stage('Docker Build & Push to ECR') {
            steps {
                sh '''
                aws ecr get-login-password --region ${AWS_REGION} \
                | docker login --username AWS --password-stdin ${ECR_URI}

                docker build -t ${ECR_REPO}:${IMAGE_TAG} .
                docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_URI}:${IMAGE_TAG}
                docker push ${ECR_URI}:${IMAGE_TAG}
                '''
            }
        }

        stage('Update ECS Task Definition') {
            steps {
                sh '''
                TD_ARN=$(aws ecs describe-services \
                  --cluster ${ECS_CLUSTER} \
                  --services ${ECS_SERVICE} \
                  --query "services[0].taskDefinition" \
                  --output text)

                aws ecs describe-task-definition \
                  --task-definition "$TD_ARN" \
                  --query 'taskDefinition' \
                  --output json > td.json

                jq '.containerDefinitions |=
                    map(if .name=="'${CONTAINER_NAME}'"
                    then .image="'${ECR_URI}':'${IMAGE_TAG}'"
                    else . end)
                    | del(.revision,.status,.registeredAt,.compatibilities,.requiresAttributes)'
                    td.json > td-new.json

                NEW_TD_ARN=$(aws ecs register-task-definition \
                  --cli-input-json file://td-new.json \
                  --query 'taskDefinition.taskDefinitionArn' \
                  --output text)

                aws ecs update-service \
                  --cluster ${ECS_CLUSTER} \
                  --service ${ECS_SERVICE} \
                  --task-definition "$NEW_TD_ARN" \
                  --force-new-deployment
                '''
            }
        }
    }

    post {
        success {
            echo "✅ WAR deployed successfully to ECS"
        }
        failure {
            echo "❌ Deployment failed"
        }
    }
}

