pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        CLUSTER_NAME = 'cloudverse-cluster'
        ECR_REPO = 'cloudverse/product-service'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('AWS Identity') {
            steps {
                sh 'aws sts get-caller-identity'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                docker build \
                  -t product-service:${BUILD_NUMBER} \
                  cloudverse/services/product-service
                '''
            }
        }

        stage('Login to ECR') {
            steps {
                sh '''
                ACCOUNT_ID=$(aws sts get-caller-identity \
                  --query Account --output text)

                aws ecr get-login-password \
                  --region ${AWS_REGION} |
                docker login \
                  --username AWS \
                  --password-stdin \
                  ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                '''
            }
        }

        stage('Push Image') {
            steps {
                sh '''
                ACCOUNT_ID=$(aws sts get-caller-identity \
                  --query Account --output text)

                IMAGE=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${BUILD_NUMBER}

                docker tag product-service:${BUILD_NUMBER} $IMAGE
                docker push $IMAGE
                '''
            }
        }

        stage('Connect EKS') {
            steps {
                sh '''
                aws eks update-kubeconfig \
                  --region ${AWS_REGION} \
                  --name ${CLUSTER_NAME}

                kubectl get nodes
                '''
            }
        }
    }
}
