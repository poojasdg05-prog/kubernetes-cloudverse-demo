# 🚀 CloudVerse Deployment Using Argo CD GitOps

> **Complete Step-by-Step Argo CD Implementation Guide for the CloudVerse Kubernetes Project**

---

## 📌 Table of Contents

1. [Project Overview](#-project-overview)
2. [What is Argo CD?](#-what-is-argo-cd)
3. [Why Do We Need Argo CD?](#-why-do-we-need-argo-cd)
4. [Argo CD Use Cases](#-argo-cd-use-cases)
5. [Advantages of Argo CD](#-advantages-of-argo-cd)
6. [CI vs CD](#-ci-vs-cd)
7. [Architecture Before and After Argo CD](#-architecture-before-and-after-argo-cd)
8. [CloudVerse Repository Structure](#-cloudverse-repository-structure)
9. [Prerequisites](#-prerequisites)
10. [Step 1 — Connect to EKS](#-step-1--connect-to-eks)
11. [Step 2 — Install Argo CD](#-step-2--install-argo-cd)
12. [Step 3 — Understand Argo CD Components](#-step-3--understand-argo-cd-components)
13. [Step 4 — Verify Argo CD](#-step-4--verify-argo-cd)
14. [Step 5 — Access Argo CD UI](#-step-5--access-argo-cd-ui)
15. [Step 6 — Get Admin Password](#-step-6--get-admin-password)
16. [Step 7 — Create Argo CD Directory](#-step-7--create-argo-cd-directory)
17. [Step 8 — Create Argo CD Application](#-step-8--create-argo-cd-application)
18. [Understanding the Application Manifest](#-understanding-the-application-manifest)
19. [Step 9 — Push Argo CD Configuration to Git](#-step-9--push-argo-cd-configuration-to-git)
20. [Step 10 — Bootstrap the Application](#-step-10--bootstrap-the-application)
21. [Step 11 — Verify Argo CD UI](#-step-11--verify-argo-cd-ui)
22. [Step 12 — Perform First GitOps Deployment](#-step-12--perform-first-gitops-deployment)
23. [Step 13 — Update Kubernetes Manifest](#-step-13--update-kubernetes-manifest)
24. [Step 14 — Watch Automatic Deployment](#-step-14--watch-automatic-deployment)
25. [Step 15 — Test Self-Healing](#-step-15--test-self-healing)
26. [Step 16 — Test Rollback](#-step-16--test-rollback)
27. [Argo CD Sync States](#-argo-cd-sync-states)
28. [Argo CD Health States](#-argo-cd-health-states)
29. [Production Architecture](#-production-architecture)
30. [CI/CD Architecture](#-cicd-architecture)
31. [Secrets Management](#-secrets-management)
32. [Troubleshooting](#-troubleshooting)
33. [Interview Explanation](#-interview-explanation)
34. [Complete Execution Flow](#-complete-execution-flow)

---

# 🌐 Project Overview

CloudVerse is a Kubernetes-based microservices application deployed on Amazon EKS.

The application contains multiple services such as:

* UI
* API Gateway
* Authentication Service
* User Service
* Product Service
* Order Service
* Cart Service
* Notification Service
* Analytics Service
* Search Service
* PostgreSQL Database
* AWS ALB Ingress
* Horizontal Pod Autoscaler
* Persistent Storage
* Kubernetes Secrets
* Kubernetes Services
* Deployments
* ReplicaSets
* Pods

The application is already deployed using Kubernetes YAML manifests.

The next goal is to implement:

> **GitOps Continuous Delivery using Argo CD.**

---

# 🔵 What is Argo CD?

**Argo CD is a declarative GitOps Continuous Delivery tool for Kubernetes.**

Instead of manually deploying Kubernetes resources using commands such as:

```bash
kubectl apply -f deployment.yaml
```

Argo CD continuously monitors Kubernetes configuration stored in Git.

The basic idea is:

```text
Git Repository
      │
      │ Desired Kubernetes State
      ▼
   Argo CD
      │
      │ Compare + Synchronize
      ▼
Kubernetes / EKS
```

Git becomes the:

> **Source of Truth**

If Git says:

```yaml
replicas: 3
```

then the desired state of the application is three replicas.

If someone manually changes the cluster to:

```text
replicas: 5
```

Argo CD can detect this configuration drift and, when self-healing is enabled, restore the state defined in Git.

---

# 🎯 Why Do We Need Argo CD?

Without Argo CD, application deployment commonly looks like:

```text
Developer
    │
    ▼
Change application
    │
    ▼
Build Docker Image
    │
    ▼
Push Image → ECR
    │
    ▼
Modify Kubernetes YAML
    │
    ▼
kubectl apply
    │
    ▼
EKS
```

This means deployment depends on someone manually interacting with the Kubernetes cluster.

For example:

```bash
kubectl apply -f 07-product-service.yaml
```

With Argo CD:

```text
Developer
    │
    │ git push
    ▼
GitHub
    │
    │ Argo CD monitors Git
    ▼
Argo CD
    │
    │ Automatic Sync
    ▼
Amazon EKS
    │
    ▼
Kubernetes Rolling Update
```

Therefore, after GitOps is implemented, normal deployment changes should happen through:

```bash
git add .
git commit -m "Update application"
git push
```

rather than manually changing production using `kubectl apply`.

---

# 💼 Argo CD Use Cases

Argo CD is useful for:

### 1. Continuous Delivery

Automatically deploy Kubernetes configuration stored in Git.

### 2. GitOps

Git becomes the central source of truth for application configuration.

### 3. Configuration Drift Detection

Argo CD identifies differences between:

```text
Git Desired State

       VS

Kubernetes Actual State
```

### 4. Self-Healing

If someone manually changes a Git-managed Kubernetes resource, Argo CD can restore the configuration defined in Git.

### 5. Multi-Cluster Management

Argo CD can manage applications across multiple Kubernetes clusters.

Example:

```text
                  Argo CD
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
     DEV EKS       QA EKS      PROD EKS
```

### 6. Environment Management

Argo CD can be used for:

```text
DEV
QA
UAT
STAGING
PRODUCTION
```

### 7. Rollback

Git history can be used to restore a previous application configuration.

### 8. Kubernetes Visualization

Argo CD provides a UI showing:

```text
Application
    │
Deployment
    │
ReplicaSet
    │
Pods
    │
Services
    │
Ingress
```

---

# ⭐ Advantages of Argo CD

| Feature           | Benefit                             |
| ----------------- | ----------------------------------- |
| GitOps            | Git becomes source of truth         |
| Automatic Sync    | Automatically deploy Git changes    |
| Self-Healing      | Correct manual configuration drift  |
| Drift Detection   | Detect Git vs cluster differences   |
| Pruning           | Remove resources deleted from Git   |
| Rollback          | Restore previous Git configuration  |
| Web UI            | Visualize Kubernetes applications   |
| Auditability      | Git records configuration changes   |
| Multi-Cluster     | Manage multiple Kubernetes clusters |
| Helm Support      | Deploy Helm charts                  |
| Kustomize Support | Manage environment overlays         |
| RBAC              | Control deployment access           |

---

# 🔄 CI vs CD

One important concept is:

> **Argo CD normally handles the CD/GitOps deployment side.**

CI can be handled by:

* Jenkins
* GitHub Actions
* GitLab CI
* AWS CodePipeline
* Other CI platforms

---

## CI Flow

```text
Developer
    │
    ▼
Git Push
    │
    ▼
Jenkins / GitHub Actions
    │
    ├── Compile
    ├── Unit Test
    ├── Security Scan
    ├── Docker Build
    └── Docker Push
              │
              ▼
          Amazon ECR
```

---

## CD Flow

```text
Git Manifest Updated
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
        ▼
Kubernetes Deployment
```

Therefore:

```text
CI
│
├── Build
├── Test
├── Docker Build
└── Push Image

CD
│
├── Monitor Git
├── Compare Desired State
├── Synchronize
├── Deploy
└── Monitor Health
```

---

# 🏗 Architecture Before and After Argo CD

## Before Argo CD

```text
GitHub
   │
   │ git clone / pull
   ▼
EC2 / Admin Machine
   │
   │ kubectl apply
   ▼
Amazon EKS
   │
   ├── UI
   ├── API Gateway
   ├── Microservices
   └── PostgreSQL
```

The administrator is responsible for applying Kubernetes YAML.

---

## After Argo CD

```text
Developer
    │
    │ git push
    ▼
GitHub Repository
    │
    │ watched by
    ▼
┌─────────────────────┐
│       Argo CD       │
│                     │
│ Git vs Kubernetes   │
└──────────┬──────────┘
           │
           │ Sync
           ▼
      Amazon EKS
           │
     ┌─────┼──────────┐
     ▼     ▼          ▼
    UI  API Gateway  Services
                      │
                      ▼
                  PostgreSQL
```

---

# 📁 CloudVerse Repository Structure

Repository:

```text
kubernetes-cloudverse-demo/
│
├── setup-cloudverse.sh
├── commands.sh
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

Argo CD will monitor:

```text
cloudverse/k8s-manifests/
```

Argo CD does **not** need to monitor the application source-code directories for this deployment.

---

# 📋 Prerequisites

Before starting, make sure the following are available:

```text
✔ AWS CLI
✔ kubectl
✔ Git
✔ Docker
✔ Amazon EKS cluster
✔ CloudVerse application
✔ GitHub repository
✔ ECR repositories
✔ kubectl access to EKS
```

---

# 🚀 Step 1 — Connect to EKS

Configure kubeconfig:

```bash
aws eks update-kubeconfig \
  --region us-east-1 \
  --name cloudverse-cluster
```

Verify the cluster:

```bash
kubectl get nodes
```

Check CloudVerse:

```bash
kubectl get pods -n cloudverse
```

Check all resources:

```bash
kubectl get all -n cloudverse
```

At this stage:

```text
CloudVerse
    │
    ▼
Already Running
```

There is no need to delete the application just to introduce Argo CD.

---

# 🚀 Step 2 — Install Argo CD

Create the Argo CD namespace:

```bash
kubectl create namespace argocd
```

Verify:

```bash
kubectl get namespaces
```

Expected:

```text
argocd
cloudverse
default
kube-system
```

Install Argo CD:

```bash
kubectl apply -n argocd \
  --server-side \
  --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

> **Production Note:** Pin a tested Argo CD release version in production instead of permanently following a moving `stable` manifest.

---

# 🧩 Step 3 — Understand Argo CD Components

Check:

```bash
kubectl get all -n argocd
```

Important Argo CD components include:

```text
argocd-server
argocd-repo-server
argocd-application-controller
argocd-applicationset-controller
argocd-dex-server
argocd-redis
```

---

## 🔹 Repo Server

The Repo Server reads configuration from Git.

```text
GitHub
   │
   ▼
Repo Server
   │
   ▼
Kubernetes Manifests
```

Its responsibility is to retrieve and generate the desired manifests.

---

## 🔹 Application Controller

The Application Controller continuously compares:

```text
Desired State
     │
     │ Git
     ▼

     VS

Actual State
     │
     │ Kubernetes
     ▼
```

Example:

Git:

```yaml
replicas: 2
```

Cluster:

```text
replicas: 5
```

Argo CD can identify:

```text
OUT OF SYNC
```

---

## 🔹 Argo CD Server

Provides:

```text
Web UI
API
CLI connectivity
```

Users interact with Argo CD through this component.

---

## 🔹 ApplicationSet Controller

ApplicationSet helps generate multiple Argo CD Applications.

It becomes especially useful when managing:

```text
Multiple Applications
Multiple Clusters
Multiple Environments
```

---

# 🚀 Step 4 — Verify Argo CD

Check Pods:

```bash
kubectl get pods -n argocd
```

Watch them:

```bash
kubectl get pods -n argocd -w
```

Wait until required Pods are:

```text
Running
```

Exit watch mode:

```text
Ctrl + C
```

---

# 🌐 Step 5 — Access Argo CD UI

For the lab environment, use port forwarding:

```bash
kubectl port-forward \
  svc/argocd-server \
  -n argocd \
  8080:443
```

If accessing from the same machine:

```text
https://localhost:8080
```

If the cluster is being administered from a remote EC2 instance, use an appropriate secure access method to reach the forwarded port.

Avoid unnecessarily exposing the Argo CD server directly to the public internet.

---

# 🔑 Step 6 — Get Admin Password

Username:

```text
admin
```

Get the initial password:

```bash
kubectl -n argocd get secret \
  argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

Then:

```bash
echo
```

Login using:

```text
Username : admin
Password : <generated-password>
```

For a long-lived environment, change the initial password and remove the initial admin secret after configuring proper access.

---

# 📂 Step 7 — Create Argo CD Directory

Go to the repository:

```bash
cd kubernetes-cloudverse-demo
```

Create:

```bash
mkdir -p argocd
```

The repository now becomes:

```text
kubernetes-cloudverse-demo/
│
├── cloudverse/
│   ├── services/
│   └── k8s-manifests/
│
├── argocd/
│   └── cloudverse-application.yaml
│
├── commands.sh
└── setup-cloudverse.sh
```

---

# 🚀 Step 8 — Create Argo CD Application

Create:

```bash
vi argocd/cloudverse-application.yaml
```

Add:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: cloudverse
  namespace: argocd

spec:

  project: default

  source:
    repoURL: https://github.com/Prashanthsb007/kubernetes-cloudverse-demo.git
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

---

# 🧠 Understanding the Application Manifest

## Application CRD

```yaml
kind: Application
```

`Application` is an Argo CD Custom Resource.

It tells Argo CD:

> Manage the application described by this Git source and Kubernetes destination.

---

## Git Repository

```yaml
repoURL: https://github.com/Prashanthsb007/kubernetes-cloudverse-demo.git
```

Argo CD reads application configuration from this repository.

---

## Branch

```yaml
targetRevision: main
```

Argo CD monitors:

```text
main
```

---

## Path

```yaml
path: cloudverse/k8s-manifests
```

Argo CD monitors:

```text
cloudverse/k8s-manifests/
```

It does not deploy everything in the repository.

---

## Destination Cluster

```yaml
server: https://kubernetes.default.svc
```

This means:

> Deploy into the same Kubernetes cluster where Argo CD is running.

Therefore, when Argo CD and CloudVerse are in the same EKS cluster, there is no need to register that same cluster again using `argocd cluster add`.

---

## Destination Namespace

```yaml
namespace: cloudverse
```

Application workloads are deployed into:

```text
cloudverse
```

---

# 🔄 Automatic Synchronization

```yaml
automated:
  prune: true
  selfHeal: true
```

This enables automatic GitOps reconciliation.

---

## 🩹 selfHeal

Suppose Git contains:

```yaml
replicas: 2
```

Someone manually executes:

```bash
kubectl scale deployment product-service \
  --replicas=5 \
  -n cloudverse
```

Now:

```text
Git       = 2
Kubernetes = 5
```

Argo CD detects drift.

With:

```yaml
selfHeal: true
```

Argo CD can reconcile Kubernetes back toward the desired state in Git.

> **Important:** If an HPA controls replicas for a Deployment, replica count is also being managed dynamically. Do not use an HPA-controlled replica field as your main self-healing demonstration without accounting for that ownership.

---

## 🗑 prune

Suppose Git contains:

```text
product-service
order-service
cart-service
```

A Git-managed resource is then removed from the desired manifests.

With:

```yaml
prune: true
```

Argo CD can remove the corresponding resource from Kubernetes during synchronization.

### ⚠️ Warning

Be careful with pruning in production.

Deleting configuration from Git may cause deletion of the live Kubernetes resource.

---

## CreateNamespace

```yaml
syncOptions:
  - CreateNamespace=true
```

This allows Argo CD to create the destination namespace when it does not already exist.

---

# 🚀 Step 9 — Push Argo CD Configuration to Git

Check:

```bash
git status
```

Add:

```bash
git add argocd/cloudverse-application.yaml
```

Commit:

```bash
git commit -m "Add Argo CD deployment for CloudVerse"
```

Push:

```bash
git push origin main
```

---

# 🚀 Step 10 — Bootstrap the Application

This is the initial bootstrap operation:

```bash
kubectl apply -f argocd/cloudverse-application.yaml
```

Check:

```bash
kubectl get applications -n argocd
```

Expected eventually:

```text
NAME         SYNC STATUS   HEALTH STATUS
cloudverse   Synced        Healthy
```

If it initially shows:

```text
OutOfSync
```

inspect it:

```bash
kubectl describe application cloudverse -n argocd
```

Also check:

```bash
kubectl get pods -n cloudverse
```

---

# 🌳 Step 11 — Verify Argo CD UI

Open Argo CD.

The application should appear similar to:

```text
┌─────────────────────────────┐
│         CLOUDVERSE          │
│                             │
│ Sync Status : Synced        │
│ Health      : Healthy       │
└─────────────────────────────┘
```

Opening the application displays the Kubernetes resource hierarchy.

Conceptually:

```text
                    CloudVerse
                        │
       ┌────────────────┼─────────────────┐
       │                │                 │
       ▼                ▼                 ▼
   PostgreSQL       API Gateway           UI
       │                │                 │
       ▼                ▼                 ▼
      PVC            Service           Service
                        │
                        ▼
                  Microservices
                        │
        ┌───────────────┼────────────────┐
        ▼               ▼                ▼
      Auth            Product           Order
        │               │                │
       Pods            Pods             Pods

                        │
                        ▼
                      Ingress
                        │
                        ▼
                       ALB
```

---

# 🚀 Step 12 — Perform First GitOps Deployment

Now perform a real deployment.

Suppose:

```text
product-service:v1
```

is currently running.

We will deploy:

```text
product-service:v2
```

using GitOps.

---

## Build v2

Move to the project:

```bash
cd cloudverse
```

Build:

```bash
docker build \
  -t cloudverse-product-service:v2 \
  ./services/product-service
```

Get AWS Account ID:

```bash
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity \
  --query Account \
  --output text)
```

Verify:

```bash
echo $AWS_ACCOUNT_ID
```

---

## Tag Image

```bash
docker tag \
  cloudverse-product-service:v2 \
  $AWS_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/cloudverse/product-service:v2
```

---

## Login to ECR

```bash
aws ecr get-login-password \
  --region us-east-1 | \
docker login \
  --username AWS \
  --password-stdin \
  $AWS_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com
```

---

## Push Image

```bash
docker push \
  $AWS_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/cloudverse/product-service:v2
```

Now:

```text
Amazon ECR
│
├── product-service:v1
└── product-service:v2
```

---

# 🚀 Step 13 — Update Kubernetes Manifest

Open the product service manifest:

```bash
vi k8s-manifests/07-product-service.yaml
```

Change:

```yaml
image: <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/cloudverse/product-service:v1
```

to:

```yaml
image: <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/cloudverse/product-service:v2
```

Save it.

---

## ❌ DO NOT DO THIS

Do not execute:

```bash
kubectl apply -f k8s-manifests/07-product-service.yaml
```

Why?

Because now:

> **Git + Argo CD should perform the deployment.**

---

## Commit the Change

```bash
git add k8s-manifests/07-product-service.yaml
```

```bash
git commit -m "Upgrade product service from v1 to v2"
```

```bash
git push origin main
```

Now stop.

Do not manually modify the Deployment.

---

# 🔄 Step 14 — Watch Automatic Deployment

The following process occurs:

```text
Developer
    │
    │ git push
    ▼
GitHub
    │
    ▼
Argo CD Detects Commit
    │
    ▼
Compare Git vs EKS
    │
    ▼
OUT OF SYNC
    │
    ▼
Automatic Sync
    │
    ▼
Deployment Updated
    │
    ▼
Kubernetes Creates New ReplicaSet
    │
    ▼
v2 Pods Start
    │
    ▼
Readiness Probe Passes
    │
    ▼
Old v1 Pods Removed
    │
    ▼
SYNCED + HEALTHY
```

---

## Watch Pods

```bash
kubectl get pods -n cloudverse -w
```

---

## Watch Rolling Update

```bash
kubectl rollout status \
  deployment/product-service \
  -n cloudverse
```

---

## Check Product Pods

```bash
kubectl get pods \
  -n cloudverse \
  -l app=product-service
```

---

## Check Running Image

```bash
kubectl get deployment product-service \
  -n cloudverse \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Expected:

```text
.../cloudverse/product-service:v2
```

🎉 The application was deployed through GitOps.

---

# 🩹 Step 15 — Test Self-Healing

The goal is to prove:

```text
Git = Source of Truth
```

Use a Git-managed field that is not simultaneously controlled by another Kubernetes controller.

For example, if a Deployment is not HPA-managed, change its replica count manually:

```bash
kubectl scale deployment <deployment-name> \
  --replicas=5 \
  -n cloudverse
```

Check:

```bash
kubectl get deployment <deployment-name> \
  -n cloudverse
```

Argo CD should detect:

```text
Git Desired State

        VS

Cluster Actual State
```

With self-healing enabled:

```text
Manual Change
     │
     ▼
Configuration Drift
     │
     ▼
Argo CD Detects Difference
     │
     ▼
Self-Healing
     │
     ▼
Git Desired State Restored
```

### HPA Important Note

If a Deployment has an HPA, the HPA may legitimately change `spec.replicas`.

Therefore, for a clean self-healing demonstration, use another Git-managed field or a Deployment that is not controlled by HPA.

---

# ⏪ Step 16 — Test Rollback

Suppose:

```text
v1 = Stable

v2 = Problem
```

Check Git history:

```bash
git log --oneline
```

Example:

```text
abc123 Upgrade product service from v1 to v2
xyz789 Previous working configuration
```

Revert:

```bash
git revert abc123
```

Push:

```bash
git push origin main
```

Argo CD detects:

```text
Git

product-service:v1

       │
       ▼
Argo CD
       │
       ▼
EKS
       │
       ▼
Rolling Update
       │
       ▼
v1 Restored
```

This provides a Git-tracked rollback.

---

# 🔄 Argo CD Sync States

Argo CD applications commonly show states such as:

## 🟢 Synced

```text
Git Desired State
       =
Kubernetes Actual State
```

Everything matches.

---

## 🟠 OutOfSync

```text
Git Desired State
       ≠
Kubernetes Actual State
```

Possible causes:

```text
New Git commit
Manual Kubernetes change
Resource missing
Manifest changed
Image version changed
```

---

# ❤️ Argo CD Health States

Health status can include:

```text
Healthy
Progressing
Degraded
Missing
Suspended
Unknown
```

### Healthy

Application resources are functioning normally.

### Progressing

A Deployment may still be rolling out.

### Degraded

A resource is not reaching the expected healthy condition.

For example:

```text
Deployment
    │
    ▼
Pod
    │
    ▼
CrashLoopBackOff
```

Argo CD may report the application/resource as degraded.

---

# 🏭 Production Architecture

A real organization may have:

```text
DEV
QA
UAT
PROD
```

A GitOps repository can evolve toward:

```text
gitops/
│
├── dev/
│   └── cloudverse/
│
├── qa/
│   └── cloudverse/
│
├── uat/
│   └── cloudverse/
│
└── prod/
    └── cloudverse/
```

Architecture:

```text
                         Git
                          │
                          ▼
                       Argo CD
                          │
          ┌───────────────┼───────────────┐
          │               │               │
          ▼               ▼               ▼
       DEV EKS          QA EKS         PROD EKS
          │               │               │
          ▼               ▼               ▼
      CloudVerse      CloudVerse       CloudVerse
```

Another design is to use separate Argo CD installations per environment or security boundary.

The correct production design depends on organizational isolation and security requirements.

---

# 🔁 CI/CD Architecture

The final project can eventually look like:

```text
Developer
    │
    │ git push
    ▼
GitHub
    │
    ▼
┌─────────────────────────────┐
│       CI PIPELINE           │
│                             │
│ Jenkins / GitHub Actions    │
│                             │
│ 1. Checkout                 │
│ 2. Test                     │
│ 3. Docker Build             │
│ 4. Security Scan            │
│ 5. Push Image               │
└─────────────┬───────────────┘
              │
              ▼
          Amazon ECR
              │
              ▼
      Update Image Tag
              │
              ▼
        Git Repository
              │
              ▼
┌─────────────────────────────┐
│           ARGO CD           │
│                             │
│ Compare Git vs Kubernetes   │
│ Sync                        │
│ Self-Heal                   │
│ Monitor Health              │
└─────────────┬───────────────┘
              │
              ▼
          Amazon EKS
              │
              ▼
       Rolling Update
              │
              ▼
        New Version
```

Therefore:

```text
CI                     CD
────────────────       ────────────────

Jenkins                Argo CD
GitHub Actions         GitOps
Docker Build           Synchronization
Testing                Deployment
Security Scan          Drift Detection
Push → ECR             Self-Healing
```

---

# 🔐 Secrets Management

For a learning project, Kubernetes Secret manifests may be stored in the repository.

For production, avoid storing plaintext credentials such as:

```yaml
stringData:
  POSTGRES_PASSWORD: my-password
```

A better architecture is:

```text
AWS Secrets Manager
        │
        ▼
External Secrets Operator
        │
        ▼
Kubernetes Secret
        │
        ▼
Application Pod
```

Git then contains configuration describing how to retrieve the secret rather than exposing the actual password.

---

# 🛠 Troubleshooting

## Application OutOfSync

Check:

```bash
kubectl get applications -n argocd
```

Describe:

```bash
kubectl describe application cloudverse -n argocd
```

---

## Check Argo CD Pods

```bash
kubectl get pods -n argocd
```

---

## Check Application Controller Logs

```bash
kubectl logs \
  -n argocd \
  deployment/argocd-application-controller
```

Depending on the installed Argo CD version/resource type, first check:

```bash
kubectl get deployment,statefulset -n argocd
```

and retrieve logs from the actual controller resource or Pod.

---

## Check Repo Server Logs

```bash
kubectl logs \
  -n argocd \
  deployment/argocd-repo-server
```

---

## Check Argo CD Server Logs

```bash
kubectl logs \
  -n argocd \
  deployment/argocd-server
```

---

## Check CloudVerse Pods

```bash
kubectl get pods -n cloudverse
```

---

## Describe Problem Pod

```bash
kubectl describe pod <pod-name> \
  -n cloudverse
```

---

## Check Pod Logs

```bash
kubectl logs <pod-name> \
  -n cloudverse
```

---

## Check Events

```bash
kubectl get events \
  -n cloudverse \
  --sort-by=.metadata.creationTimestamp
```

---

## Check Deployment

```bash
kubectl get deployment \
  -n cloudverse
```

---

## Check Services

```bash
kubectl get svc \
  -n cloudverse
```

---

## Check Ingress

```bash
kubectl get ingress \
  -n cloudverse
```

---

## Check Application Details

```bash
kubectl get application cloudverse \
  -n argocd \
  -o yaml
```

---

# 🎤 Interview Explanation

A concise interview explanation of this implementation is:

> We deployed our CloudVerse microservices application on Amazon EKS and implemented GitOps-based Continuous Delivery using Argo CD.
>
> Kubernetes manifests are maintained in Git, which acts as our source of truth. Argo CD continuously compares the desired state stored in Git with the actual state running in EKS.
>
> Whenever a Kubernetes manifest is changed and pushed to Git, Argo CD detects the difference and synchronizes the required changes to the cluster. We enabled automated synchronization, pruning, and self-healing so that Git-managed resources remain aligned with the desired configuration.
>
> Application images are stored in Amazon ECR. When a new application image is built, its image tag is updated in the Kubernetes manifests and committed to Git. Argo CD detects the new desired state and Kubernetes performs the rolling deployment.
>
> This provides automated deployment, configuration drift detection, Git-based auditing, self-healing, easier rollback, and centralized visibility of Kubernetes application health.

---

# 🧠 Important Interview Question

### Why not directly use `kubectl apply` from Jenkins?

A traditional pipeline can do:

```text
Jenkins
   │
   │ kubectl apply
   ▼
EKS
```

With GitOps:

```text
Jenkins
   │
   ├── Build
   ├── Test
   └── Push Image
          │
          ▼
         ECR

Git Desired State
       │
       ▼
     Argo CD
       │
       ▼
      EKS
```

This separates:

```text
BUILD responsibility

from

DEPLOYMENT reconciliation
```

Argo CD continuously verifies that the live cluster remains aligned with Git instead of performing only a one-time deployment command.

---

# 🧠 Another Important Interview Question

### What happens if somebody manually modifies Kubernetes?

Example:

```bash
kubectl edit deployment product-service -n cloudverse
```

Now:

```text
Git
 │
 │ Desired State
 ▼

Different From

Kubernetes
 │
 │ Actual State
 ▼
```

Argo CD detects:

```text
OUT OF SYNC
```

With:

```yaml
selfHeal: true
```

Argo CD can restore the Git-defined configuration.

---

# 🧠 What Happens When Git Changes?

Suppose:

```text
product-service:v1
```

changes to:

```text
product-service:v2
```

Complete internal flow:

```text
1. Developer changes manifest
            │
            ▼
2. git commit
            │
            ▼
3. git push
            │
            ▼
4. GitHub main branch updated
            │
            ▼
5. Argo CD observes/reconciles repository state
            │
            ▼
6. Argo CD compares Git with EKS
            │
            ▼
7. Application becomes OutOfSync
            │
            ▼
8. Automated Sync starts
            │
            ▼
9. Deployment spec updated
            │
            ▼
10. Kubernetes Deployment Controller
            │
            ▼
11. New ReplicaSet created
            │
            ▼
12. v2 Pod starts
            │
            ▼
13. Startup/Readiness checks
            │
            ▼
14. Pod becomes Ready
            │
            ▼
15. Old v1 Pod terminated
            │
            ▼
16. Rolling Update completed
            │
            ▼
17. Argo CD reports Synced
            │
            ▼
18. Application reports Healthy
```

---

# 🏆 Complete Execution Flow

Use this as the final implementation checklist.

```text
Existing CloudVerse EKS Cluster
              │
              ▼
1. Verify EKS Nodes
              │
              ▼
2. Verify CloudVerse Application
              │
              ▼
3. Create argocd Namespace
              │
              ▼
4. Install Argo CD
              │
              ▼
5. Verify Argo CD Pods
              │
              ▼
6. Access Argo CD UI
              │
              ▼
7. Retrieve Admin Password
              │
              ▼
8. Create argocd/ Directory
              │
              ▼
9. Create cloudverse-application.yaml
              │
              ▼
10. Push Argo CD Config → Git
              │
              ▼
11. Bootstrap Application CR
              │
              ▼
12. Verify Synced + Healthy
              │
              ▼
13. Build product-service:v2
              │
              ▼
14. Push v2 → Amazon ECR
              │
              ▼
15. Change Manifest v1 → v2
              │
              ▼
16. git commit
              │
              ▼
17. git push
              │
              ▼
       ❌ NO kubectl apply
              │
              ▼
18. Argo CD Detects Difference
              │
              ▼
19. Application → OutOfSync
              │
              ▼
20. Automatic Sync
              │
              ▼
21. Kubernetes RollingUpdate
              │
              ▼
22. New ReplicaSet
              │
              ▼
23. New v2 Pods
              │
              ▼
24. Readiness Successful
              │
              ▼
25. Old v1 Pods Terminated
              │
              ▼
26. Application → Synced
              │
              ▼
27. Application → Healthy
              │
              ▼
28. Test Configuration Drift
              │
              ▼
29. Verify Self-Healing
              │
              ▼
30. Test Git Revert / Rollback
              │
              ▼
       🎉 GitOps Complete
```

---

# 🎯 Final Architecture

```text
                         DEVELOPER
                             │
                             │ git push
                             ▼
                    ┌────────────────┐
                    │     GitHub     │
                    │ Source of Truth│
                    └───────┬────────┘
                            │
                  ┌─────────┴──────────┐
                  │                    │
                  ▼                    ▼
             CI PIPELINE            ARGO CD
        Jenkins/GitHub Actions        │
                  │                   │
                  │ Build             │ Monitor
                  │ Test              │ Compare
                  │ Docker Build      │ Sync
                  │                   │ Self-Heal
                  ▼                   │
             Amazon ECR               │
                  │                   │
                  └────────┐          │
                           │          │
                           ▼          ▼
                         AMAZON EKS
                             │
             ┌───────────────┼────────────────┐
             │               │                │
             ▼               ▼                ▼
            UI          API Gateway      Microservices
                                              │
                                              ▼
                                          PostgreSQL
```

---

# ✅ Final Result

After completing this implementation:

```text
✔ CloudVerse runs on Amazon EKS

✔ Docker images are stored in Amazon ECR

✔ Kubernetes configuration is stored in Git

✔ Git acts as the source of truth

✔ Argo CD monitors the Git repository

✔ Argo CD detects configuration differences

✔ Argo CD automatically synchronizes approved Git changes

✔ Kubernetes performs rolling deployments

✔ Argo CD detects configuration drift

✔ Self-healing can restore Git-managed configuration

✔ Pruning can remove resources no longer defined in Git

✔ Git provides deployment history and rollback capability

✔ Argo CD UI provides application health and resource visibility
```

---

# 🎉 CloudVerse + Argo CD GitOps Implementation Complete

```text
                    Git
                     │
                     ▼
                  Argo CD
                     │
             GitOps Reconciliation
                     │
                     ▼
                 Amazon EKS
                     │
                     ▼
                  CloudVerse
```

> **The key GitOps principle:**
>
> **Do not treat the live cluster as the source of truth.**
>
> **Change the desired state in Git → commit → push → let Argo CD reconcile Kubernetes.**

---
