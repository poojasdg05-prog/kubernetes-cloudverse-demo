pipeline {

    agent any

    environment {
        AWS_REGION  = 'us-east-1'
        CLUSTER_NAME = 'cloudverse-cluster'
        NAMESPACE    = 'cloudverse'

        AWS_ACCOUNT_ID = '914834315117'

        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

        UI_REPO = 'cloudverse/ui-service'
        UI_IMAGE = "${ECR_REGISTRY}/${UI_REPO}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('AWS Identity') {
            steps {
                sh '''
                    echo "Checking AWS identity..."
                    aws sts get-caller-identity
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "Building UI Docker image..."

                    docker build \
                        -t ui-service:${BUILD_NUMBER} \
                        cloudverse/services/ui-service
                '''
            }
        }

        stage('Login to ECR') {
            steps {
                sh '''
                    echo "Logging into Amazon ECR..."

                    aws ecr get-login-password \
                        --region ${AWS_REGION} | \
                    docker login \
                        --username AWS \
                        --password-stdin ${ECR_REGISTRY}
                '''
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh '''
                    echo "Tagging image..."

                    docker tag \
                        ui-service:${BUILD_NUMBER} \
                        ${UI_IMAGE}:${BUILD_NUMBER}

                    echo "Pushing image to ECR..."

                    docker push \
                        ${UI_IMAGE}:${BUILD_NUMBER}
                '''
            }
        }

        stage('Connect to EKS') {
            steps {
                sh '''
                    echo "Updating kubeconfig..."

                    aws eks update-kubeconfig \
                        --region ${AWS_REGION} \
                        --name ${CLUSTER_NAME}

                    echo "Checking Kubernetes access..."

                    kubectl get nodes

                    echo "Checking CloudVerse namespace..."

                    kubectl get pods -n ${NAMESPACE}
                '''
            }
        }

        stage('Deploy UI') {
            steps {
                sh '''
                    echo "Updating UI deployment..."

                    kubectl set image deployment/ui-service \
                        ui-service=${UI_IMAGE}:${BUILD_NUMBER} \
                        -n ${NAMESPACE}

                    echo "Deployment image updated successfully."

                    kubectl get deployment ui-service \
                        -n ${NAMESPACE} \
                        -o jsonpath='{.spec.template.spec.containers[0].image}{"\\n"}'
                '''
            }
        }

        stage('Wait for Rollout') {
            steps {
                sh '''
                    echo "Waiting for UI rollout..."

                    kubectl rollout status \
                        deployment/ui-service \
                        -n ${NAMESPACE} \
                        --timeout=180s
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "Checking deployment..."

                    kubectl get deployment ui-service \
                        -n ${NAMESPACE}

                    echo "Checking UI pods..."

                    kubectl get pods \
                        -n ${NAMESPACE} \
                        -l app=ui-service \
                        -o wide

                    echo "Checking all CloudVerse pods..."

                    kubectl get pods -n ${NAMESPACE}
                '''
            }
        }
    }

    post {

        success {
            echo '''
            ==========================================
            CloudVerse UI Deployment Successful
            ==========================================
            '''
        }

        failure {
            echo '''
            ==========================================
            CloudVerse UI Deployment Failed
            ==========================================
            '''

            sh '''
                echo "Deployment status:"
                kubectl get deployment ui-service -n ${NAMESPACE} || true

                echo "Pod status:"
                kubectl get pods -n ${NAMESPACE} || true

                echo "Recent events:"
                kubectl get events \
                    -n ${NAMESPACE} \
                    --sort-by=.lastTimestamp | tail -30 || true
            '''
        }
    }
}
