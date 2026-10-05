pipeline {

    agent any

    environment {
        AWS_REGION = 'us-east-1'
        ECR_PREFIX = 'cloudverse'
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out CloudVerse repository...'
                checkout scm
            }
        }

        stage('Verify Environment') {
            steps {
                sh '''
                    echo "===== AWS VERSION ====="
                    aws --version

                    echo "===== DOCKER VERSION ====="
                    docker --version

                    echo "===== KUBECTL VERSION ====="
                    kubectl version --client

                    echo "===== AWS IDENTITY ====="
                    aws sts get-caller-identity
                '''
            }
        }

        stage('ECR Login') {
            steps {
                sh '''
                    echo "Logging into Amazon ECR..."

                    AWS_ACCOUNT_ID=$(aws sts get-caller-identity \
                        --query Account \
                        --output text)

                    aws ecr get-login-password \
                        --region ${AWS_REGION} | \
                    docker login \
                        --username AWS \
                        --password-stdin \
                        ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                    echo "ECR login successful"
                '''
            }
        }

        stage('Build All Services') {
            steps {
                sh '''
                    set -e

                    SERVICES="ui api-gateway auth-service user-service product-service order-service cart-service notification-service analytics-service search-service"

                    for SERVICE in $SERVICES
                    do
                        echo "=============================================="
                        echo "BUILDING SERVICE: $SERVICE"
                        echo "=============================================="

                        docker build \
                            -t ${SERVICE}:${BUILD_NUMBER} \
                            cloudverse/services/${SERVICE}

                        echo "Successfully built ${SERVICE}:${BUILD_NUMBER}"
                        echo ""
                    done
                '''
            }
        }

        stage('Tag Images') {
            steps {
                sh '''
                    set -e

                    AWS_ACCOUNT_ID=$(aws sts get-caller-identity \
                        --query Account \
                        --output text)

                    ECR_REGISTRY=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                    SERVICES="ui api-gateway auth-service user-service product-service order-service cart-service notification-service analytics-service search-service"

                    for SERVICE in $SERVICES
                    do
                        echo "=============================================="
                        echo "TAGGING SERVICE: $SERVICE"
                        echo "=============================================="

                        docker tag \
                            ${SERVICE}:${BUILD_NUMBER} \
                            ${ECR_REGISTRY}/${ECR_PREFIX}/${SERVICE}:${BUILD_NUMBER}

                    done
                '''
            }
        }

        stage('Push Images to ECR') {
            steps {
                sh '''
                    set -e

                    AWS_ACCOUNT_ID=$(aws sts get-caller-identity \
                        --query Account \
                        --output text)

                    ECR_REGISTRY=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                    SERVICES="ui api-gateway auth-service user-service product-service order-service cart-service notification-service analytics-service search-service"

                    for SERVICE in $SERVICES
                    do
                        echo "=============================================="
                        echo "PUSHING SERVICE: $SERVICE"
                        echo "=============================================="

                        docker push \
                            ${ECR_REGISTRY}/${ECR_PREFIX}/${SERVICE}:${BUILD_NUMBER}

                        echo "Successfully pushed ${SERVICE}:${BUILD_NUMBER}"
                        echo ""
                    done
                '''
            }
        }

        stage('Verify ECR Images') {
            steps {
                sh '''
                    set -e

                    SERVICES="ui api-gateway auth-service user-service product-service order-service cart-service notification-service analytics-service search-service"

                    for SERVICE in $SERVICES
                    do
                        echo "=============================================="
                        echo "VERIFYING: $SERVICE"
                        echo "=============================================="

                        aws ecr describe-images \
                            --repository-name ${ECR_PREFIX}/${SERVICE} \
                            --region ${AWS_REGION} \
                            --query "imageDetails[?contains(imageTags, '${BUILD_NUMBER}')].imageTags" \
                            --output table

                    done
                '''
            }
        }
    }

    post {

        success {
            echo '''
            ==============================================
            CLOUDVERSE CI PIPELINE SUCCESSFUL
            ==============================================
            All services were:
            1. Checked out
            2. Docker built
            3. Tagged
            4. Pushed to Amazon ECR
            5. Verified
            ==============================================
            '''
        }

        failure {
            echo '''
            ==============================================
            CLOUDVERSE CI PIPELINE FAILED
            ==============================================
            Check the Jenkins Console Output.
            ==============================================
            '''
        }

        always {
            sh '''
                echo "Cleaning unused Docker images..."
                docker image prune -f || true
            '''
        }
    }
}


