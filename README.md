# 3-Tier NodeJS CRUD App — DevSecOps End-to-End Project

> **DevOps Shack** | @devopsshack | 2026  
> Full-stack user management app deployed on AWS EKS with an Enterprise DevSecOps CI/CD pipeline using GitHub Actions.

---

## What This Project Is

A production-grade, 3-tier web application used as the base for a complete DevSecOps pipeline demonstration. The app is a simple User Management CRUD system — the real learning is in how it is built, secured, containerised, and deployed end-to-end without manual steps.

**Stack:**
- **Frontend** — React 17, Webpack, Axios
- **Backend** — Node.js, Express 4, MySQL
- **Container** — Docker (multi-stage, non-root user)
- **Orchestration** — Kubernetes on AWS EKS
- **CI/CD** — GitHub Actions (15 jobs, 5 security scanners)
- **Ingress** — AWS ALB via AWS Load Balancer Controller

---

## Project Structure

```
3-Tier-NodeJS-End-To-End-Project/
├── client/                        # React frontend
│   ├── src/
│   │   ├── App.js                 # Main app component
│   │   ├── components/            # UsersList, UserItem
│   │   └── api/users.js           # Axios API calls
│   ├── public/                    # Static assets
│   ├── package.json               # React + Webpack deps
│   └── webpack.config.js
├── server/                        # Node.js backend
│   ├── server.js                  # Entry point (port 5000)
│   ├── routes/users.js            # REST API routes
│   ├── controllers/               # userController.js
│   ├── models/userModel.js        # MySQL queries
│   └── config/db.js               # DB connection
├── k8s-manifests/
│   ├── app-deployment.yaml        # EKS Deployment (SHA-tagged image)
│   ├── app-svc.yaml               # ClusterIP Service
│   └── app-ingress.yaml           # ALB Ingress (awscdaypune.in)
├── .github/workflows/
│   └── nodejs-cicd.yml            # 15-job DevSecOps pipeline
├── Dockerfile                     # Multi-stage, non-root build
├── docker-compose.yml             # Local dev with MySQL
└── README.md
```

---

## Application Features

- Add users (name, email, role: Admin/User)
- View all users in a list
- Edit user details
- Delete users
- Responsive UI with smooth animations

---

## Pipeline — Step-by-Step Guide

> Full pipeline documentation, deep-dive PDF, and supporting assets are available in the Google Drive folder below.

### [Pipeline Resources & Guide — Google Drive](https://drive.google.com/drive/folders/1C3c9sUNMvZtpKuG9gBuxQk6DvEkHgALh?usp=sharing)

The Drive folder contains:
- DevSecOps CI Pipeline Deep Dive PDF (31 pages, all 15 jobs explained)
- GitHub Actions workflow YAML
- Kubernetes manifests
- IAM policy JSON for ALB Controller
- Supporting diagrams and cheatsheets

---

## Pipeline Overview — 15 Jobs, 10 Waves

| Wave | Jobs | Purpose |
|------|------|---------|
| 1 | `gitleaks-scan` | Secret scanning — blocks everything if it fails |
| 2 | `checkov-terraform`, `checkov-k8s`, `checkov-dockerfile`, `client-trivy-fs`, `server-trivy-fs` | IaC + dependency vulnerability scans (parallel) |
| 3 | `client-lint`, `server-lint` | Code quality gate |
| 4 | `client-test`, `server-test` | Unit tests |
| 5 | `client-build` | Frontend compilation |
| 6 | `docker-build` | Docker image assembly |
| 7 | `image-trivy-scan` | Container image vulnerability scan |
| 8 | `docker-push` | Push to Docker Hub (main branch only) |
| 9 | `update-k8s-manifest` | GitOps — update image SHA in YAML, commit back |
| 10 | `deploy-to-k8s` | Deploy to EKS via OIDC (no static credentials) |

**Security scanners used:** Gitleaks · Checkov (Terraform, K8s, Dockerfile) · Trivy FS (client + server) · Trivy Image

---

## Step-by-Step Setup

### Step 1 — Prerequisites

- AWS CLI configured (`aws configure`)
- `kubectl` installed
- `eksctl` installed
- `helm` installed
- Docker installed
- GitHub repository forked or cloned

### Step 2 — Create EKS Cluster

```bash
eksctl create cluster \
  --name my-cluster \
  --region ap-south-1 \
  --nodegroup-name standard-nodes \
  --node-type t3.medium \
  --nodes 2
```

### Step 3 — Tag Public Subnets for ALB

```powershell
# Auto-tag all public subnets (PowerShell)
(aws ec2 describe-subnets --region ap-south-1 `
  --filters "Name=tag:alpha.eksctl.io/cluster-name,Values=my-cluster" `
  --query "Subnets[?MapPublicIpOnLaunch==``true``].SubnetId" `
  --output text).Split() | ForEach-Object {
  aws ec2 create-tags --region ap-south-1 --resources $_ `
    --tags Key=kubernetes.io/role/elb,Value=1 `
           Key=kubernetes.io/cluster/my-cluster,Value=shared
}
```

### Step 4 — Install AWS Load Balancer Controller

```bash
# Create IAM policy
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

# Associate OIDC provider
eksctl utils associate-iam-oidc-provider \
  --region ap-south-1 --cluster my-cluster --approve

# Create service account with IAM binding
eksctl create iamserviceaccount \
  --cluster=my-cluster --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts --region ap-south-1 --approve

# Install via Helm
helm repo add eks https://aws.github.io/eks-charts && helm repo update eks
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --version 1.14.0
```

### Step 5 — Create Kubernetes Namespace + Secrets

```bash
kubectl create namespace project

kubectl create secret generic mysql-secret -n project \
  --from-literal=MYSQL_DATABASE=test_db \
  --from-literal=MYSQL_USER=appuser \
  --from-literal=MYSQL_PASSWORD=apppass \
  --from-literal=DATABASE_URL=mysql://appuser:apppass@mysql:3306/test_db
```

### Step 6 — Configure GitHub Secrets

Go to your repo **Settings > Secrets and variables > Actions** and add:

| Secret | Value |
|--------|-------|
| `DOCKERHUB_USERNAME` | Your Docker Hub username |
| `DOCKERHUB_TOKEN` | Docker Hub access token (not password) |
| `AWS_ROLE_TO_ASSUME` | `arn:aws:iam::<ACCOUNT_ID>:role/GitHubActionsEKSDeployRole` |

> `GITHUB_TOKEN` is auto-provided — no action needed.

### Step 7 — Set Up GitHub OIDC IAM Role

Create an IAM role in AWS with the following trust policy (replace `<GITHUB_ORG>` and `<REPO_NAME>`):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
        "token.actions.githubusercontent.com:sub": "repo:<GITHUB_ORG>/<REPO_NAME>:ref:refs/heads/main"
      }
    }
  }]
}
```

Attach policies: `AmazonEKSClusterPolicy`, `AmazonEKSWorkerNodePolicy`, `AmazonEC2ContainerRegistryReadOnly`.

### Step 8 — Push to Main to Trigger Pipeline

```bash
git add .
git commit -m "feat: initial commit"
git push origin main
```

The pipeline runs automatically. On push to `main`, all 15 jobs execute including Docker push, GitOps manifest update, and EKS deploy.

---

## Running Locally (Docker Compose)

```bash
# Clone the repo
git clone https://github.com/<your-org>/3-Tier-NodeJS-End-To-End-Project.git
cd 3-Tier-NodeJS-End-To-End-Project

# Start MySQL + App
docker compose up --build

# Access the app
open http://localhost:5000
```

Docker Compose spins up:
- MySQL 8.0 with health check
- Node.js app (builds client + server, serves on port 5000)

---

## Kubernetes Manifests

### Deployment (`k8s-manifests/app-deployment.yaml`)

- Image tagged with `GITHUB_SHA` — immutable, every commit gets a unique tag
- Secrets pulled from `mysql-secret` Kubernetes Secret
- `imagePullSecrets` references `regcred` for Docker Hub authentication

### Ingress (`k8s-manifests/app-ingress.yaml`)

- ALB scheme: `internet-facing`
- Target type: `ip` (pod-level routing)
- HTTPS on port 443 with ACM certificate
- HTTP → HTTPS redirect
- Host: `awscdaypune.in`

---

## Security Features

| Feature | Tool | What It Catches |
|---------|------|-----------------|
| Secret scanning | Gitleaks | Hardcoded API keys, tokens, passwords in code/git history |
| IaC scanning | Checkov | Terraform, K8s, Dockerfile misconfigurations |
| Dependency CVEs | Trivy FS | Known CVEs in npm packages (client + server) |
| Image CVEs | Trivy Image | OS-level CVEs in Docker base image layers |
| No static credentials | GitHub OIDC | Eliminates long-lived AWS access keys entirely |
| Non-root container | Dockerfile | `appuser` group — no root process in production |

---

## Environment Variables

| Variable | Source | Description |
|----------|--------|-------------|
| `DB_HOST` | Hardcoded `mysql` | MySQL service hostname in K8s |
| `DB_NAME` | `mysql-secret` | Database name (`test_db`) |
| `DB_USER` | `mysql-secret` | DB username |
| `DB_PASSWORD` | `mysql-secret` | DB password |
| `DATABASE_URL` | `mysql-secret` | Full connection URL |
| `NODE_ENV` | Dockerfile | Set to `production` in container |

---

## Tech Versions

| Component | Version |
|-----------|---------|
| Node.js | 20 (Alpine) |
| React | 17.0.2 |
| Express | 4.17.1 |
| MySQL (client) | 2.18.1 |
| MySQL (server) | 8.0 |
| Webpack | 5.x |
| ALB Controller | 1.14.0 |
| eksctl | 0.224.0 |

---

## License

This project is licensed under the MIT License.

> **NOTE:** This application should not be used for commercial purposes by anyone other than DevOps Shack.

---

*DevOps Shack | @devopsshack | Building DevOps education for everyone*
