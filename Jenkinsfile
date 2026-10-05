pipeline {

    agent any

    environment {
        AWS_REGION = 'us-east-1'
        ECR_PREFIX = 'cloudverse'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Verify AWS') {
            steps {
                sh '''
                    echo "Checking AWS identity..."
                    aws sts get-caller-identity
                '''
            }
        }

        stage('ECR Login') {
            steps {
                sh '''
                    AWS_ACCOUNT_ID=$(aws sts get-caller-identity \
                      --query Account \
                      --output text)

                    aws ecr get-login-password \
                      --region ${AWS_REGION} | \
                    docker login \
                      --username AWS \
                      --password-stdin \
                      ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                '''
            }
        }

        stage('Build All Images') {
            steps {
                sh '''
                    SERVICES="ui api-gateway auth-service user-service product-service order-service cart-service notification-service analytics-service search-service"

                    for SERVICE in $SERVICES
                    do

                      echo "========================================="
                      echo "BUILDING $SERVICE"
                      echo "========================================="

                      docker build \
                        -t $SERVICE:${BUILD_NUMBER} \
                        cloudverse/services/$SERVICE

                    done
                '''
            }
        }

        stage('Tag and Push All Images') {
            steps {
                sh '''
                    AWS_ACCOUNT_ID=$(aws sts get-caller-identity \
                      --query Account \
                      --output text)

                    ECR_REGISTRY=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                    SERVICES="ui api-gateway auth-service user-service product-service order-service cart-service notification-service analytics-service search-service"

                    for SERVICE in $SERVICES
                    do

                      echo "========================================="
                      echo "PUSHING $SERVICE"
                      echo "========================================="

                      IMAGE=${ECR_REGISTRY}/${ECR_PREFIX}/${SERVICE}:${BUILD_NUMBER}

                      docker tag \
                        $SERVICE:${BUILD_NUMBER} \
                        $IMAGE

                      docker push $IMAGE

                    done
                '''
            }
        }

        stage('Verify Images') {
            steps {
                sh '''
                    SERVICES="ui api-gateway auth-service user-service product-service order-service cart-service notification-service analytics-service search-service"

                    for SERVICE in $SERVICES
                    do

                      echo "-----------------------------------------"
                      echo "ECR IMAGE: $SERVICE"
                      echo "-----------------------------------------"

                      aws ecr describe-images \
                        --repository-name ${ECR_PREFIX}/$SERVICE \
                        --region ${AWS_REGION} \
                        --query 'imageDetails[].imageTags' \
                        --output table

                    done
                '''
            }
        }
    }

    post {

        success {
            echo '========================================='
            echo 'ALL CLOUDVERSE IMAGES BUILT SUCCESSFULLY'
            echo '========================================='
        }

        failure {
            echo '========================================='
            echo 'CLOUDVERSE BUILD FAILED'
            echo
