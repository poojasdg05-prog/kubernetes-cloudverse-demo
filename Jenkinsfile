pipeline {

    agent any

    environment {
        AWS_REGION     = 'us-east-1'
        AWS_ACCOUNT_ID = '914834315117'
        EKS_CLUSTER    = 'cloudverse-cluster'
        NAMESPACE      = 'cloudverse'

        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

        SERVICES = 'ui api-gateway auth-service user-service product-service order-service cart-service notification-service analytics-service search-service'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {

        stage('Checkout') {
            steps {
                echo '========== CHECKOUT =========='

                checkout scm

                sh '''
                    echo "Commit:"
                    git rev-parse --short HEAD

                    echo "Workspace:"
                    pwd

                    ls -la
                '''
            }
        }

        stage('Verify Tools') {
            steps {
                echo '========== VERIFY TOOLS =========='

                sh '''
                    set -e

                    git --version
                    docker --version
                    aws --version
                    kubectl version --client
                '''
            }
        }

        stage('Verify AWS') {
            steps {
                echo '========== VERIFY AWS =========='

                sh '''
                    set -e

                    aws sts get-caller-identity
                '''
            }
        }

        stage('Configure EKS') {
            steps {
                echo '========== CONFIGURE EKS =========='

                sh '''
                    set -e

                    aws eks update-kubeconfig \
                      --region "$AWS_REGION" \
                      --name "$EKS_CLUSTER"

                    kubectl get nodes
                '''
            }
        }

        stage('Create Namespace') {
            steps {
                echo '========== CREATE NAMESPACE =========='

                sh '''
                    set -e

                    kubectl create namespace "$NAMESPACE" \
                      --dry-run=client \
                      -o yaml | kubectl apply -f -

                    kubectl get namespace "$NAMESPACE"
                '''
            }
        }

        stage('ECR Login') {
            steps {
                echo '========== ECR LOGIN =========='

                sh '''
                    set -e

                    aws ecr get-login-password \
                      --region "$AWS_REGION" |
                    docker login \
                      --username AWS \
                      --password-stdin "$ECR_REGISTRY"
                '''
            }
        }

        stage('Prepare ECR') {
            steps {
                echo '========== PREPARE ECR =========='

                sh '''
                    set -e

                    for SERVICE in $SERVICES
                    do
                        REPOSITORY="cloudverse/$SERVICE"

                        echo "Checking $REPOSITORY"

                        aws ecr describe-repositories \
                          --repository-names "$REPOSITORY" \
                          --region "$AWS_REGION" \
                          >/dev/null 2>&1 ||
                        aws ecr create-repository \
                          --repository-name "$REPOSITORY" \
                          --region "$AWS_REGION"
                    done
                '''
            }
        }

        stage('Build Images') {
            steps {
                echo '========== BUILD IMAGES =========='

                sh '''
                    set -e

                    for SERVICE in $SERVICES
                    do
                        echo "================================"
                        echo "BUILDING $SERVICE"
                        echo "================================"

                        docker build \
                          -t "$SERVICE:$BUILD_NUMBER" \
                          "cloudverse/services/$SERVICE"
                    done
                '''
            }
        }

        stage('Tag Images') {
            steps {
                echo '========== TAG IMAGES =========='

                sh '''
                    set -e

                    for SERVICE in $SERVICES
                    do
                        docker tag \
                          "$SERVICE:$BUILD_NUMBER" \
                          "$ECR_REGISTRY/cloudverse/$SERVICE:$BUILD_NUMBER"
                    done
                '''
            }
        }

        stage('Push Images') {
            steps {
                echo '========== PUSH IMAGES =========='

                sh '''
                    set -e

                    for SERVICE in $SERVICES
                    do
                        echo "Pushing $SERVICE:$BUILD_NUMBER"

                        docker push \
                          "$ECR_REGISTRY/cloudverse/$SERVICE:$BUILD_NUMBER"
                    done
                '''
            }
        }

        stage('Deploy Kubernetes Manifests') {
            steps {
                echo '========== DEPLOY MANIFESTS =========='

                sh '''
                    set -e

                    kubectl apply \
                      -f cloudverse/k8s-manifests/ \
                      -n "$NAMESPACE"
                '''
            }
        }

        stage('Update Application Images') {
            steps {
                echo '========== UPDATE IMAGES =========='

                sh '''
                    set -e

                    for SERVICE in $SERVICES
                    do
                        echo "Updating deployment: $SERVICE"

                        kubectl set image \
                          deployment/"$SERVICE" \
                          "$SERVICE"="$ECR_REGISTRY/cloudverse/$SERVICE:$BUILD_NUMBER" \
                          -n "$NAMESPACE"
                    done
                '''
            }
        }

        stage('Wait for Rollout') {
            steps {
                echo '========== WAIT FOR ROLLOUT =========='

                sh '''
                    set -e

                    for SERVICE in $SERVICES
                    do
                        echo "Waiting for $SERVICE"

                        kubectl rollout status \
                          deployment/"$SERVICE" \
                          -n "$NAMESPACE" \
                          --timeout=300s
                    done
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                echo '========== VERIFY DEPLOYMENT =========='

                sh '''
                    echo ""
                    echo "========== PODS =========="
                    kubectl get pods -n "$NAMESPACE" -o wide

                    echo ""
                    echo "========== DEPLOYMENTS =========="
                    kubectl get deployments -n "$NAMESPACE"

                    echo ""
                    echo "========== SERVICES =========="
                    kubectl get svc -n "$NAMESPACE"

                    echo ""
                    echo "========== INGRESS =========="
                    kubectl get ingress -n "$NAMESPACE"

                    echo ""
                    echo "========== CURRENT IMAGES =========="
                    kubectl get deployments \
                      -n "$NAMESPACE" \
                      -o custom-columns=DEPLOYMENT:.metadata.name,IMAGE:.spec.template.spec.containers[*].image
                '''
            }
        }
    }

    post {

        success {
            echo '======================================'
            echo ' CLOUDVERSE CI/CD SUCCESSFUL'
            echo '======================================'
        }

        failure {
            echo '======================================'
            echo ' CLOUDVERSE CI/CD FAILED'
            echo '======================================'

            sh '''
                kubectl get pods -n "$NAMESPACE" || true
                kubectl get events -n "$NAMESPACE" \
                  --sort-by=.lastTimestamp | tail -30 || true
            '''
        }

        always {
            echo 'Cleaning Docker build cache'

            sh '''
                docker image prune -f || true
            '''
        }
    }
}
