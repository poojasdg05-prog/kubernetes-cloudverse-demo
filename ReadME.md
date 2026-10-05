# 🚀 CloudVerse — Jenkins CI/CD + Amazon EKS + Argo CD + Prometheus + Grafana

> **Complete End-to-End DevOps Implementation Guide**

---

# 📌 Project Overview

CloudVerse is a Kubernetes-based microservices application running on **Amazon EKS**.

This implementation adds a complete DevOps workflow using:

- 🐙 GitHub — Source Code Management
- ⚙️ Jenkins — Continuous Integration
- 🐳 Docker — Containerization
- 📦 Amazon ECR — Container Registry
- ☸️ Amazon EKS — Kubernetes Platform
- 🔄 Argo CD — GitOps Continuous Delivery
- 📊 Prometheus — Metrics Collection
- 📈 Grafana — Monitoring and Visualization

---

# 🏗️ Final Architecture

```text
                        Developer
                            │
                         git push
                            │
                            ▼
                         GitHub
                            │
                            ▼
                    ┌───────────────┐
                    │    Jenkins    │
                    │      CI       │
                    └───────┬───────┘
                            │
             ┌──────────────┼──────────────┐
             │              │              │
             ▼              ▼              ▼
           Build           Test        Docker Build
                                           │
                                           ▼
                                      Amazon ECR
                                           │
                                 Update Image Tags
                                           │
                                           ▼
                                         GitHub
                                           │
                                           ▼
                                      ┌─────────┐
                                      │ Argo CD │
                                      └────┬────┘
                                           │
                                           ▼
                                      Amazon EKS
                                           │
                    ┌──────────────────────┼──────────────────────┐
                    │                      │                      │
                    ▼                      ▼                      ▼
                   UI                 API Gateway           Microservices
                                                                 │
                                                                 ▼
                                                             PostgreSQL

                                           │
                                           ▼
                                      Prometheus
                                           │
                                           ▼
                                        Grafana
                                           │
                                           ▼
                                      Dashboards
```

---

# 🧩 CloudVerse Microservices

The application contains:

| # | Service | Purpose |
|---|---|---|
| 1 | `ui` | Frontend |
| 2 | `api-gateway` | API Gateway |
| 3 | `auth-service` | Authentication |
| 4 | `user-service` | User Management |
| 5 | `product-service` | Product Management |
| 6 | `order-service` | Order Management |
| 7 | `cart-service` | Shopping Cart |
| 8 | `notification-service` | Notifications |
| 9 | `analytics-service` | Analytics |
| 10 | `search-service` | Search |

PostgreSQL is deployed using Kubernetes manifests and therefore does **not** require a custom Jenkins Docker build.

---

# 📁 Repository Structure

```text
kubernetes-cloudverse-demo/
│
├── Jenkinsfile
│
├── README.md
├── commands.sh
├── setup-cloudverse.sh
│
├── argocd/
│   └── cloudverse-application.yaml
│
└── cloudverse/
    │
    ├── services/
    │   ├── ui/
    │   ├── api-gateway/
    │   ├── auth-service/
    │   ├── user-service/
    │   ├── product-service/
    │   ├── order-service/
    │   ├── cart-service/
    │   ├── notification-service/
    │   ├── analytics-service/
    │   └── search-service/
    │
    └── k8s-manifests/
        ├── 00-namespace.yaml
        ├── 01-postgres-pv-pvc.yaml
        ├── 02-postgres-secret.yaml
        ├── 03-postgres-deployment.yaml
        ├── 04-postgres-service.yaml
        ├── 05-auth-service.yaml
        ├── ...
        ├── 13-api-gateway.yaml
        ├── 14-ui-service.yaml
        ├── 15-ingress.yaml
        └── 16-hpa.yaml
```

---

# 🔄 CI vs CD

## Jenkins — Continuous Integration

Jenkins performs:

```text
Git Checkout
     │
     ▼
Validate AWS
     │
     ▼
ECR Login
     │
     ▼
Build Docker Images
     │
     ▼
Tag Images
     │
     ▼
Push Images
     │
     ▼
Amazon ECR
```

## Argo CD — Continuous Delivery

Argo CD performs:

```text
GitHub Kubernetes Manifests
          │
          ▼
       Argo CD
          │
      Compare State
          │
          ▼
       Amazon EKS
          │
          ▼
   CloudVerse Application
```

---

# ============================================================
# ☸️ PART 1 — CREATE AMAZON EKS CLUSTER
# ============================================================

If the previous CloudVerse cluster was destroyed, create a new cluster.

Example:

```bash
eksctl create cluster \
  --name cloudverse-jenkins-cluster \
  --region us-east-1 \
  --version 1.36 \
  --nodegroup-name my-node \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 3 \
  --nodes-max 6 \
  --managed
```

Cluster creation may take several minutes.

---

# 🔍 Verify Cluster

```bash
eksctl get cluster --region us-east-1
```

Configure kubeconfig:

```bash
aws eks update-kubeconfig \
  --region us-east-1 \
  --name cloudverse-jenkins-cluster
```

Verify:

```bash
kubectl get nodes
```

Expected:

```text
NAME                         STATUS   ROLES    AGE
ip-xxx.ec2.internal          Ready    <none>   ...
ip-xxx.ec2.internal          Ready    <none>   ...
ip-xxx.ec2.internal          Ready    <none>   ...
```

All nodes must become:

```text
Ready
```

---

# ============================================================
# 💾 PART 2 — INSTALL EBS CSI DRIVER
# ============================================================

Associate IAM OIDC provider:

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster cloudverse-jenkins-cluster \
  --region us-east-1 \
  --approve
```

Create IAM role:

```bash
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster cloudverse-jenkins-cluster \
  --region us-east-1 \
  --role-name AmazonEKS_EBS_CSI_DriverRole_CloudVerse \
  --role-only \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve
```

Get role ARN:

```bash
ROLE_ARN=$(aws iam get-role \
  --role-name AmazonEKS_EBS_CSI_DriverRole_CloudVerse \
  --query 'Role.Arn' \
  --output text)

echo $ROLE_ARN
```

Install EBS CSI:

```bash
eksctl create addon \
  --cluster cloudverse-jenkins-cluster \
  --region us-east-1 \
  --name aws-ebs-csi-driver \
  --service-account-role-arn $ROLE_ARN \
  --force
```

Verify:

```bash
kubectl get pods -n kube-system | grep ebs
```

---

# ============================================================
# ⚙️ PART 3 — CREATE JENKINS EC2 SERVER
# ============================================================

Create a separate EC2 instance for Jenkins.

Recommended configuration:

```text
Name          : jenkins-server
OS            : Amazon Linux 2023
Instance Type : t3.medium
Storage       : 30 GB
```

Security Group:

```text
22    → Your IP
8080  → Your IP
```

Do not permanently expose Jenkins port `8080` to the entire Internet.

Attach an IAM role that provides the permissions required for:

```text
ECR
EKS
STS
```

Use least-privilege IAM permissions for production environments.

---

# ============================================================
# ☕ PART 4 — INSTALL JENKINS
# ============================================================

SSH into Jenkins EC2.

```bash
ssh -i aws.pem ec2-user@<JENKINS-IP>
```

Update packages:

```bash
sudo dnf update -y
```

Install Java:

```bash
sudo dnf install java-21-amazon-corretto -y
```

Verify:

```bash
java -version
```

Add Jenkins repository:

```bash
sudo wget -O /etc/yum.repos.d/jenkins.repo \
https://pkg.jenkins.io/rpm-stable/jenkins.repo
```

Import Jenkins signing key:

```bash
sudo rpm --import \
https://pkg.jenkins.io/rpm-stable/jenkins.io-2026.key
```

Install Jenkins:

```bash
sudo dnf install jenkins -y
```

Start Jenkins:

```bash
sudo systemctl enable --now jenkins
```

Verify:

```bash
sudo systemctl status jenkins
```

---

# 🌐 PART 5 — ACCESS JENKINS

Open:

```text
http://<JENKINS-IP>:8080
```

Get initial password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Paste the password into Jenkins.

Choose:

```text
Install suggested plugins
```

Create the Jenkins administrator account.

---

# 🔌 PART 6 — JENKINS PLUGINS

Navigate:

```text
Manage Jenkins
      ↓
Plugins
      ↓
Available Plugins
```

Install:

```text
Git
GitHub
Pipeline
Docker Pipeline
Credentials Binding
```

Restart Jenkins if requested.

---

# ============================================================
# 🐳 PART 7 — INSTALL DOCKER
# ============================================================

Install Docker:

```bash
sudo dnf install docker -y
```

Start Docker:

```bash
sudo systemctl enable --now docker
```

Verify:

```bash
docker --version
```

---

# 🔐 IMPORTANT — DOCKER PERMISSION FIX

If this error occurs:

```text
permission denied while trying to connect to the Docker daemon socket
unix:///var/run/docker.sock
```

add both users to the Docker group:

```bash
sudo usermod -aG docker ec2-user
sudo usermod -aG docker jenkins
```

Check:

```bash
getent group docker
```

Expected:

```text
docker:x:xxx:ec2-user,jenkins
```

Log out:

```bash
exit
```

SSH back into EC2.

Then:

```bash
groups
```

`docker` should appear.

Test:

```bash
docker ps
```

Test Jenkins:

```bash
sudo -u jenkins docker ps
```

Restart Jenkins:

```bash
sudo systemctl restart jenkins
```

Both commands should work without Docker socket permission errors.

---

# ============================================================
# 🛠️ PART 8 — INSTALL REQUIRED TOOLS
# ============================================================

## Git

```bash
sudo dnf install git -y
```

Verify:

```bash
git --version
```

---

## AWS CLI

Check:

```bash
aws --version
```

If unavailable:

```bash
sudo dnf install unzip -y

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" \
-o "awscliv2.zip"

unzip awscliv2.zip

sudo ./aws/install
```

Verify:

```bash
aws --version
```

Test IAM role:

```bash
aws sts get-caller-identity
```

---

## kubectl

Install a kubectl version compatible with the EKS cluster.

Verify:

```bash
kubectl version --client
```

Configure EKS:

```bash
aws eks update-kubeconfig \
  --region us-east-1 \
  --name cloudverse-jenkins-cluster
```

Verify:

```bash
kubectl get nodes
```

---

## Helm

Install Helm and verify:

```bash
helm version
```

---

# ============================================================
# 🐙 PART 9 — CLONE CLOUDVERSE
# ============================================================

Clone the repository:

```bash
cd /tmp

git clone https://github.com/Renuka-dot/kubernetes-cloudverse-demo.git

cd kubernetes-cloudverse-demo
```

Check:

```bash
ls
```

Check services:

```bash
ls cloudverse/services
```

Expected:

```text
ui
api-gateway
auth-service
user-service
product-service
order-service
cart-service
notification-service
analytics-service
search-service
```

---

# 🐳 Verify Dockerfiles

```bash
find cloudverse/services \
  -maxdepth 2 \
  -name Dockerfile \
  -print
```

Every application service should have a usable Dockerfile before starting Jenkins builds.

---

# ============================================================
# 📦 PART 10 — CREATE ECR REPOSITORIES FOR ALL SERVICES
# ============================================================

Define services:

```bash
SERVICES="ui api-gateway auth-service user-service product-service order-service cart-service notification-service analytics-service search-service"
```

Create ECR repositories:

```bash
for SERVICE in $SERVICES
do

  echo "Checking cloudverse/$SERVICE"

  aws ecr describe-repositories \
    --repository-names cloudverse/$SERVICE \
    --region us-east-1 >/dev/null 2>&1 || \
  aws ecr create-repository \
    --repository-name cloudverse/$SERVICE \
    --region us-east-1

done
```

Verify:

```bash
aws ecr describe-repositories \
  --region us-east-1 \
  --query 'repositories[?starts_with(repositoryName, `cloudverse/`)].repositoryName' \
  --output table
```

Expected repositories:

```text
cloudverse/ui
cloudverse/api-gateway
cloudverse/auth-service
cloudverse/user-service
cloudverse/product-service
cloudverse/order-service
cloudverse/cart-service
cloudverse/notification-service
cloudverse/analytics-service
cloudverse/search-service
```

---

# ============================================================
# 🔑 PART 11 — TEST ECR LOGIN
# ============================================================

Get AWS account:

```bash
AWS_ACCOUNT_ID=$(aws sts get-caller-identity \
  --query Account \
  --output text)

echo $AWS_ACCOUNT_ID
```

Login:

```bash
aws ecr get-login-password \
  --region us-east-1 | \
docker login \
  --username AWS \
  --password-stdin \
  ${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com
```

Expected:

```text
Login Succeeded
```

Do not permanently store an ECR Docker password. Jenkins can generate a temporary ECR login token during every pipeline execution.

---

# ============================================================
# 🧪 PART 12 — TEST BUILD ALL CLOUDVERSE SERVICES
# ============================================================

Before Jenkins automation, test all Dockerfiles.

```bash
cd /tmp/kubernetes-cloudverse-demo
```

Run:

```bash
SERVICES="ui api-gateway auth-service user-service product-service order-service cart-service notification-service analytics-service search-service"

for SERVICE in $SERVICES
do

  echo "=========================================="
  echo "BUILDING: $SERVICE"
  echo "=========================================="

  docker build \
    -t $SERVICE:test \
    cloudverse/services/$SERVICE || exit 1

done
```

Verify:

```bash
docker images
```

This confirms that Jenkins should be able to use the same Docker build contexts.

---

# ============================================================
# ⚙️ PART 13 — JENKINS PIPELINE FOR ALL 10 SERVICES
# ============================================================

The Jenkins pipeline now builds:

```text
ui
api-gateway
auth-service
user-service
product-service
order-service
cart-service
notification-service
analytics-service
search-service
```

Pipeline flow:

```text
                       Jenkins
                          │
                          ▼
                      Checkout
                          │
                          ▼
                    Verify AWS
                          │
                          ▼
                     ECR Login
                          │
                          ▼
               Build ALL 10 Images
                          │
                          ▼
                Tag ALL 10 Images
                          │
                          ▼
               Push ALL 10 → ECR
                          │
                          ▼
                    Verify Images
```

---

# 📄 Jenkinsfile

Create/edit:

```bash
vi Jenkinsfile
```

Use:

```groovy
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
                    echo "========================================="
                    echo "VERIFYING AWS IDENTITY"
                    echo "========================================="

                    aws sts get-caller-identity
                '''
            }
        }

        stage('ECR Login') {
            steps {
                sh '''
                    echo "========================================="
                    echo "LOGGING INTO AMAZON ECR"
                    echo "========================================="

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
                      echo "BUILDING: $SERVICE"
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
                      echo "PUSHING: $SERVICE"
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

        stage('Verify ECR Images') {
            steps {
                sh '''
                    SERVICES="ui api-gateway auth-service user-service product-service order-service cart-service notification-service analytics-service search-service"

                    for SERVICE in $SERVICES
                    do

                      echo "======================================"
                      echo "ECR IMAGE: $SERVICE"
                      echo "======================================"

                      aws ecr describe-images \
                        --repository-name ${ECR_PREFIX}/$SERVICE \
                        --region ${AWS_REGION} \
                        --query 'imageDetails[].imageTags[]' \
                        --output text

                    done
                '''
            }
        }
    }

    post {

        success {
            echo '========================================='
            echo 'CLOUDVERSE CI PIPELINE SUCCESSFUL'
            echo 'ALL 10 IMAGES BUILT AND PUSHED TO ECR'
            echo '========================================='
        }

        failure {
            echo '========================================='
            echo 'CLOUDVERSE PIPELINE FAILED'
            echo 'CHECK JENKINS CONSOLE OUTPUT'
            echo '========================================='
        }

        always {
            sh '''
                echo "Cleaning unused Docker build data..."
                docker image prune -f || true
            '''
        }
    }
}
```

---

# 🏷️ Jenkins Build Numbers

Jenkins automatically provides:

```text
BUILD_NUMBER
```

For example:

```text
Jenkins Build #8
```

creates:

```text
cloudverse/ui:8
cloudverse/api-gateway:8
cloudverse/auth-service:8
cloudverse/user-service:8
cloudverse/product-service:8
cloudverse/order-service:8
cloudverse/cart-service:8
cloudverse/notification-service:8
cloudverse/analytics-service:8
cloudverse/search-service:8
```

This gives every CI build a traceable image version.

---

# ============================================================
# 🐙 PART 14 — PUSH JENKINSFILE TO GITHUB
# ============================================================

Check:

```bash
git status
```

Add:

```bash
git add Jenkinsfile
```

Commit:

```bash
git commit -m "Add Jenkins CI pipeline for all CloudVerse services"
```

Push:

```bash
git push origin main
```

Check:

```bash
git log --oneline -5
```

---

# ============================================================
# ⚙️ PART 15 — CREATE JENKINS PIPELINE JOB
# ============================================================

Jenkins:

```text
Dashboard
    ↓
New Item
    ↓
cloudverse-pipeline
    ↓
Pipeline
```

Configure:

```text
Definition:

Pipeline script from SCM
```

SCM:

```text
Git
```

Repository:

```text
https://github.com/Renuka-dot/kubernetes-cloudverse-demo.git
```

Branch:

```text
*/main
```

Script Path:

```text
Jenkinsfile
```

Save.

---

# 🚀 PART 16 — RUN PIPELINE

Click:

```text
Build Now
```

Open:

```text
Build History
      ↓
Build Number
      ↓
Console Output
```

Expected pipeline:

```text
Checkout
    ↓
Verify AWS
    ↓
ECR Login
    ↓
Build All Images
    │
    ├── ui
    ├── api-gateway
    ├── auth-service
    ├── user-service
    ├── product-service
    ├── order-service
    ├── cart-service
    ├── notification-service
    ├── analytics-service
    └── search-service
    ↓
Tag Images
    ↓
Push All Images
    ↓
Verify ECR
    ↓
SUCCESS
```

---

# ============================================================
# 🔍 PART 17 — VERIFY ALL IMAGES IN ECR
# ============================================================

Run:

```bash
SERVICES="ui api-gateway auth-service user-service product-service order-service cart-service notification-service analytics-service search-service"

for SERVICE in $SERVICES
do

  echo "======================================"
  echo "ECR IMAGE: $SERVICE"
  echo "======================================"

  aws ecr describe-images \
    --repository-name cloudverse/$SERVICE \
    --region us-east-1 \
    --query 'imageDetails[].imageTags[]' \
    --output text

done
```

Example:

```text
======================================
ECR IMAGE: ui
======================================
8

======================================
ECR IMAGE: api-gateway
======================================
8

======================================
ECR IMAGE: auth-service
======================================
8

...

======================================
ECR IMAGE: search-service
======================================
8
```

At this point:

```text
GitHub
   │
   ▼
Jenkins
   │
   ├── Build 10 Images
   │
   └── Push 10 Images
   ▼
Amazon ECR

        ✅ CI COMPLETE
```

---

# ============================================================
# 🔄 PART 18 — CONTINUOUS DELIVERY WITH ARGO CD
# ============================================================

The recommended architecture separates CI and CD.

```text
Jenkins = CI
Argo CD = CD
```

Jenkins should:

```text
Build
Test
Create Docker Images
Push Images → ECR
Update Git Image Tags
```

Argo CD should:

```text
Monitor Git
Detect Manifest Changes
Synchronize Kubernetes
Deploy to EKS
Self-Heal Drift
```

Final flow:

```text
Developer
    │
    │ git push
    ▼
GitHub
    │
    ▼
Jenkins
    │
    ├── Build
    ├── Docker Build
    └── Push → ECR
             │
             ▼
      Update Manifest Tag
             │
             ▼
           GitHub
             │
             ▼
          Argo CD
             │
             ▼
         Amazon EKS
```

---

# ============================================================
# 🔵 PART 19 — INSTALL ARGO CD
# ============================================================

Create namespace:

```bash
kubectl create namespace argocd
```

Install:

```bash
kubectl apply \
  -n argocd \
  --server-side \
  --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

For production, pin a tested Argo CD version instead of permanently tracking `stable`.

Verify:

```bash
kubectl get pods -n argocd
```

Wait until the required pods are:

```text
Running
```

---

# 🔑 Get Argo CD Password

```bash
kubectl -n argocd get secret \
  argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

echo
```

Username:

```text
admin
```

---

# 🌐 Access Argo CD

For a lab:

```bash
kubectl port-forward \
  svc/argocd-server \
  -n argocd \
  8080:443
```

Use a secure tunnel if Kubernetes is being administered from a remote EC2 server.

---

# ============================================================
# 📄 PART 20 — ARGO CD APPLICATION
# ============================================================

Use:

```text
argocd/cloudverse-application.yaml
```

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: cloudverse
  namespace: argocd

spec:

  project: default

  source:
    repoURL: https://github.com/Renuka-dot/kubernetes-cloudverse-demo.git
    targetRevision: main
    path: cloudverse/k8s-manifests

  destination:
    server: https://kubernetes.default.svc
    namespace: cloudverse

  syncPolicy:

    automated:
      prune: true
      selfHeal: true

    syncOptions:
      - CreateNamespace=true
```

Apply:

```bash
kubectl apply \
  -f argocd/cloudverse-application.yaml
```

Verify:

```bash
kubectl get applications -n argocd
```

---

# 🔍 Verify CloudVerse Deployment

```bash
kubectl get pods -n cloudverse
```

Check deployments:

```bash
kubectl get deployments -n cloudverse
```

Check services:

```bash
kubectl get svc -n cloudverse
```

Check ingress:

```bash
kubectl get ingress -n cloudverse
```

Check everything:

```bash
kubectl get all -n cloudverse
```

---

# 🔎 Troubleshoot Pods

If a pod is not running:

```bash
kubectl get pods -n cloudverse
```

Describe:

```bash
kubectl describe pod <POD-NAME> \
  -n cloudverse
```

Logs:

```bash
kubectl logs <POD-NAME> \
  -n cloudverse
```

For multi-container pods:

```bash
kubectl get pod <POD-NAME> \
  -n cloudverse \
  -o jsonpath='{.spec.containers[*].name}'
```

Then:

```bash
kubectl logs <POD-NAME> \
  -c <CONTAINER-NAME> \
  -n cloudverse
```

---

# ============================================================
# 📊 PART 21 — PROMETHEUS + GRAFANA
# ============================================================

Now add observability.

Architecture:

```text
                         Amazon EKS
                             │
            ┌────────────────┼────────────────┐
            │                │                │
            ▼                ▼                ▼
          Nodes             Pods        Kubernetes API
            │                │                │
            └────────────────┼────────────────┘
                             │
                             ▼
                       Prometheus
                             │
                       Store Metrics
                             │
                             ▼
                          Grafana
                             │
                             ▼
                         Dashboards
```

---

# 📦 PART 22 — INSTALL kube-prometheus-stack

Create namespace:

```bash
kubectl create namespace monitoring
```

Add Helm repository:

```bash
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts
```

Update:

```bash
helm repo update
```

Install:

```bash
helm install monitoring \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring
```

Verify Helm:

```bash
helm list -n monitoring
```

Check pods:

```bash
kubectl get pods -n monitoring
```

Expected components include:

```text
Prometheus
Grafana
Alertmanager
kube-state-metrics
node-exporter
Prometheus Operator
```

---

# 🔍 PART 23 — VERIFY PROMETHEUS

Check services:

```bash
kubectl get svc -n monitoring
```

Check Prometheus:

```bash
kubectl get prometheus -n monitoring
```

Check targets/resources:

```bash
kubectl get servicemonitor -A
```

---

# 🌐 PART 24 — ACCESS PROMETHEUS

Port forward:

```bash
kubectl port-forward \
  -n monitoring \
  svc/monitoring-kube-prometheus-prometheus \
  9090:9090
```

Access locally or through a secure SSH tunnel:

```text
http://localhost:9090
```

Useful PromQL:

```promql
up
```

Node CPU:

```promql
rate(node_cpu_seconds_total[5m])
```

Pod CPU:

```promql
rate(container_cpu_usage_seconds_total[5m])
```

Memory:

```promql
container_memory_working_set_bytes
```

---

# ============================================================
# 📈 PART 25 — ACCESS GRAFANA
# ============================================================

Check service:

```bash
kubectl get svc \
  -n monitoring \
  | grep grafana
```

Port forward:

```bash
kubectl port-forward \
  svc/monitoring-grafana \
  3000:80 \
  -n monitoring
```

Access:

```text
http://localhost:3000
```

---

# 🔑 Grafana Credentials

Username:

```text
admin
```

Get password:

```bash
kubectl get secret \
  monitoring-grafana \
  -n monitoring \
  -o jsonpath="{.data.admin-password}" \
  | base64 --decode

echo
```

---

# 📊 PART 26 — GRAFANA DASHBOARDS

The monitoring stack provides Kubernetes dashboards for areas such as:

```text
Cluster
Nodes
Namespaces
Pods
Workloads
CPU
Memory
Networking
Persistent Volumes
Kubernetes API
```

Monitor CloudVerse namespace:

```text
namespace = cloudverse
```

Important metrics:

```text
CPU Usage
Memory Usage
Pod Status
Pod Restarts
Deployment Replicas
Network Traffic
PVC Usage
Node Health
API Server Health
```

---

# 📊 CloudVerse Monitoring Architecture

```text
CloudVerse
    │
    ├── UI
    ├── API Gateway
    ├── Auth
    ├── User
    ├── Product
    ├── Order
    ├── Cart
    ├── Notification
    ├── Analytics
    └── Search
          │
          ▼
       Kubernetes
          │
          ▼
      Prometheus
          │
          ▼
        Grafana
          │
          ▼
      Dashboards
```

---

# ============================================================
# 🚨 PART 27 — MONITOR POD RESTARTS
# ============================================================

CLI:

```bash
kubectl get pods \
  -n cloudverse
```

Prometheus query:

```promql
kube_pod_container_status_restarts_total{namespace="cloudverse"}
```

---

# 🚨 Monitor Unavailable Replicas

```promql
kube_deployment_status_replicas_unavailable{namespace="cloudverse"}
```

---

# 📈 Monitor CPU

```promql
sum(
  rate(
    container_cpu_usage_seconds_total{
      namespace="cloudverse",
      container!=""
    }[5m]
  )
)
```

---

# 📈 Monitor Memory

```promql
sum(
  container_memory_working_set_bytes{
    namespace="cloudverse",
    container!=""
  }
)
```

---

# ============================================================
# 🔎 PART 28 — TROUBLESHOOTING
# ============================================================

## Docker Permission Denied

Error:

```text
permission denied while trying to connect to the Docker daemon socket
```

Fix:

```bash
sudo usermod -aG docker ec2-user
sudo usermod -aG docker jenkins
```

Log out/in and restart Jenkins:

```bash
sudo systemctl restart jenkins
```

Test:

```bash
docker ps

sudo -u jenkins docker ps
```

---

## Jenkins Cannot Access AWS

Test:

```bash
sudo -u jenkins aws sts get-caller-identity
```

Check the IAM role attached to the Jenkins EC2 instance.

---

## Jenkins Cannot Access EKS

Test:

```bash
sudo -u jenkins aws eks describe-cluster \
  --name cloudverse-jenkins-cluster \
  --region us-east-1
```

The Jenkins IAM role must have the required AWS permissions and appropriate EKS cluster access if Jenkins is directly interacting with Kubernetes.

When Argo CD handles CD, Jenkins does not need broad Kubernetes administrator permissions merely to build and push images.

---

## ImagePullBackOff

Check:

```bash
kubectl describe pod <POD> \
  -n cloudverse
```

Verify image:

```bash
kubectl get pod <POD> \
  -n cloudverse \
  -o jsonpath='{.spec.containers[*].image}'
```

Verify ECR:

```bash
aws ecr describe-images \
  --repository-name cloudverse/<SERVICE> \
  --region us-east-1
```

---

## CrashLoopBackOff

```bash
kubectl logs <POD> \
  -n cloudverse
```

Previous container:

```bash
kubectl logs <POD> \
  -n cloudverse \
  --previous
```

Describe:

```bash
kubectl describe pod <POD> \
  -n cloudverse
```

---

## Pending Pods

```bash
kubectl describe pod <POD> \
  -n cloudverse
```

Check:

```bash
kubectl get events \
  -n cloudverse \
  --sort-by=.lastTimestamp
```

Possible causes:

```text
Insufficient CPU
Insufficient Memory
PVC Pending
Node Selector
Taints/Tolerations
Maximum Pods Per Node
EBS CSI problems
```

---

## Prometheus Pods Not Running

```bash
kubectl get pods \
  -n monitoring
```

Describe:

```bash
kubectl describe pod <POD> \
  -n monitoring
```

Check events:

```bash
kubectl get events \
  -n monitoring \
  --sort-by=.lastTimestamp
```

---

## Grafana Not Accessible

Check:

```bash
kubectl get svc \
  -n monitoring
```

Restart port-forward:

```bash
kubectl port-forward \
  svc/monitoring-grafana \
  3000:80 \
  -n monitoring
```

---

# ============================================================
# 🧹 PART 29 — DOCKER DISK CLEANUP
# ============================================================

Jenkins builds many Docker images, so periodically check:

```bash
docker system df
```

Remove unused images:

```bash
docker image prune -f
```

Remove unused build cache:

```bash
docker builder prune -f
```

Be careful with:

```bash
docker system prune
```

because it can remove additional unused Docker resources.

---

# ============================================================
# 🔐 PART 30 — SECURITY BEST PRACTICES
# ============================================================

For a production-style implementation:

- Do not store AWS access keys directly inside `Jenkinsfile`.
- Prefer an EC2 IAM role for Jenkins.
- Apply least-privilege IAM permissions.
- Do not expose Jenkins `8080` publicly.
- Do not expose Grafana publicly without authentication/security controls.
- Do not expose Prometheus publicly.
- Do not commit Kubernetes secrets in plaintext.
- Do not use `latest` as the only production image tag.
- Use immutable image tags such as Jenkins build numbers or Git commit SHA.
- Use ECR image scanning.
- Protect the Git `main` branch.
- Use Jenkins credentials for Git write authentication.
- Restrict who can trigger production deployments.
- Pin tested component versions in production.

---

# ============================================================
# 🧹 PART 31 — CLEANUP
# ============================================================

## Delete Monitoring

```bash
helm uninstall monitoring \
  -n monitoring
```

Delete namespace:

```bash
kubectl delete namespace monitoring
```

---

## Delete Argo CD

```bash
kubectl delete \
  -f argocd/cloudverse-application.yaml
```

Then:

```bash
kubectl delete namespace argocd
```

---

## Delete CloudVerse

```bash
kubectl delete namespace cloudverse
```

---

## Delete EKS Cluster

```bash
eksctl delete cluster \
  --name cloudverse-jenkins-cluster \
  --region us-east-1
```

Verify:

```bash
eksctl get cluster \
  --region us-east-1
```

---

# ============================================================
# 🎯 COMPLETE END-TO-END FLOW
# ============================================================

```text
Developer Changes Code
          │
          ▼
       Git Push
          │
          ▼
        GitHub
          │
          ▼
        Jenkins
          │
          ├── Checkout
          ├── Validate AWS
          ├── Build UI
          ├── Build API Gateway
          ├── Build Auth
          ├── Build User
          ├── Build Product
          ├── Build Order
          ├── Build Cart
          ├── Build Notification
          ├── Build Analytics
          └── Build Search
                    │
                    ▼
                 Docker
                    │
                    ▼
                Amazon ECR
                    │
                    ▼
             Manifest Image Tag
                    │
                    ▼
                  GitHub
                    │
                    ▼
                  Argo CD
                    │
                    ▼
                Amazon EKS
                    │
        ┌───────────┼───────────┐
        │           │           │
        ▼           ▼           ▼
       UI       API Gateway  Services
                                │
                                ▼
                            PostgreSQL

                    │
                    ▼
                Prometheus
                    │
                    ▼
                  Grafana
                    │
                    ▼
                Dashboards
```

---

# 🏆 Technologies Used

| Technology | Purpose |
|---|---|
| Git | Version Control |
| GitHub | Source Repository |
| Jenkins | Continuous Integration |
| Docker | Containerization |
| Amazon ECR | Docker Registry |
| Amazon EKS | Kubernetes |
| Kubernetes | Container Orchestration |
| Argo CD | GitOps Continuous Delivery |
| Helm | Kubernetes Package Manager |
| Prometheus | Monitoring |
| Grafana | Visualization |
| AWS EBS CSI | Persistent Storage |
| AWS IAM | Authentication & Authorization |

---

# 🎤 Interview Explanation

> **CloudVerse is a microservices application deployed on Amazon EKS. I implemented Jenkins as the Continuous Integration platform. When application code changes are pushed to GitHub, Jenkins checks out the repository, builds Docker images for all ten CloudVerse application services, tags the images using the Jenkins build number and pushes them to individual Amazon ECR repositories.**
>
> **For Continuous Delivery, Argo CD follows the GitOps model. Kubernetes manifests are maintained in Git, and Argo CD continuously compares the desired state stored in Git with the actual state in EKS. When the manifests change, Argo CD synchronizes the changes to the cluster.**
>
> **For observability, I deployed the kube-prometheus-stack using Helm. Prometheus collects Kubernetes cluster, node, workload and pod metrics, while Grafana provides dashboards for monitoring CPU, memory, pod restarts, deployment replicas, networking and overall cluster health.**
>
> **This provides an end-to-end DevOps workflow from source code to container build, registry, GitOps deployment and production-style monitoring.**

---

# ⭐ Final Result

After completing this project:

```text
GitHub
   │
   ▼
Jenkins CI
   │
   ▼
Docker
   │
   ▼
Amazon ECR
   │
   ▼
GitOps Repository
   │
   ▼
Argo CD
   │
   ▼
Amazon EKS
   │
   ├── CloudVerse Microservices
   └── PostgreSQL
          │
          ▼
      Prometheus
          │
          ▼
        Grafana
          │
          ▼
      Monitoring
```

## 🚀 CloudVerse — Complete CI/CD + GitOps + Observability Project

**Jenkins + Docker + Amazon ECR + Amazon EKS + Argo CD + Prometheus + Grafana**
