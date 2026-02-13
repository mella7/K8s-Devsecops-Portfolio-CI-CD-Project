# Kubernetes DevSecOps CI/CD Platform

## Overview

This project demonstrates a complete end-to-end DevSecOps pipeline built
around Kubernetes on AWS.

The goal is not just to deploy an application, but to simulate how
modern production systems are built:

-   Infrastructure is provisioned using Infrastructure as Code
-   CI pipelines enforce security and quality gates
-   Container images are scanned before deployment
-   Deployment is handled via GitOps
-   The cluster is fully observable through monitoring tools

This project reflects real-world DevOps practices rather than a simple
lab deployment.

------------------------------------------------------------------------

# Architecture

The platform is structured into four logical layers:

## 1. Infrastructure Layer (Provisioning)

-   Terraform
-   AWS (EC2, EKS, IAM, S3)
-   GitHub Actions

Terraform provisions: - A jumphost EC2 instance - An EKS cluster - IAM
roles and policies - Remote backend (S3 for Terraform state)

All infrastructure changes are triggered through GitHub Actions.

------------------------------------------------------------------------

## 2. Continuous Integration Layer (CI)

-   GitHub Actions
-   SonarCloud
-   Snyk
-   Trivy
-   DockerHub

When code is pushed:

1.  Static code analysis runs (SonarCloud)
2.  Dependency vulnerability scan runs (Snyk)
3.  Docker image is built
4.  Image is scanned (Trivy)
5.  Image is pushed to DockerHub

Only secure and validated images move forward.

------------------------------------------------------------------------

## 3. Continuous Deployment Layer (GitOps)

-   Kubernetes (EKS)
-   ArgoCD
-   AWS Load Balancer Controller

ArgoCD continuously monitors Kubernetes manifests stored in GitHub.

Deployment flow: - Developer updates manifests - ArgoCD detects
changes - Kubernetes cluster reconciles automatically

Manual `kubectl apply` is not used.

Git becomes the single source of truth.

------------------------------------------------------------------------

## 4. Observability Layer

-   Prometheus
-   Grafana

Cluster metrics are collected and visualized:

-   Node CPU & memory
-   Pod health
-   Deployment status
-   Kubernetes control plane metrics

This ensures operational visibility after deployment.

------------------------------------------------------------------------

# Repository Structure

The project uses two separate repositories:

## 1. Infrastructure Repository (iac_code)

Contains: - Terraform configuration - GitHub Actions workflow for
provisioning - Remote backend configuration

Purpose: Automate cloud resource creation.

------------------------------------------------------------------------

## 2. Application Repository (application_code)

Contains: - Application source code - Dockerfile - Kubernetes
manifests - CI workflow for build and security scanning

Purpose: Manage application lifecycle.

------------------------------------------------------------------------

# Step-by-Step Implementation

------------------------------------------------------------------------

# 1. GitHub SSH Setup

Generate SSH key:

``` bash
mkdir -p ~/.ssh
cd ~/.ssh
ssh-keygen -t ed25519 -f key
```

Add the public key to GitHub → Settings → SSH and GPG Keys.

Force Git to use the key:

``` bash
export GIT_SSH_COMMAND="ssh -i ~/.ssh/key"
```

------------------------------------------------------------------------

# 2. Clone Repositories

``` bash
mkdir ~/Projects/devsecops
cd ~/Projects/devsecops

git clone <application_repo>
git clone <iac_repo>
```

If needed, update remote origin:

``` bash
git remote set-url origin <your_repo_url>
```

------------------------------------------------------------------------

# 3. AWS Preparation

## Create IAM User

Create an IAM user for CI/CD testing.

Generate: - AWS_ACCESS_KEY_ID - AWS_SECRET_ACCESS_KEY

Store them in GitHub Secrets.

------------------------------------------------------------------------

## Create S3 Bucket

Create a bucket to store Terraform state remotely.

Add secret:

    BUCKET_TF=<bucket_name>

Remote state ensures: - Safe collaboration - Consistent infrastructure
state - CI compatibility

------------------------------------------------------------------------

## Create EC2 Key Pair

Create a key pair in AWS.

Download the `.pem` file. It will be used to SSH into the jumphost.

------------------------------------------------------------------------

# 4. Install Local Tools

## Terraform

``` bash
sudo apt install terraform
```

## AWS CLI

``` bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip
sudo ./aws/install
```

Configure:

``` bash
aws configure
```

------------------------------------------------------------------------

# 5. Provision Jumphost via GitHub Actions

Push changes in the `iac_code` repository:

``` bash
git commit -am "terraform update"
git push
```

GitHub Actions runs: - terraform init - terraform plan - terraform apply

An EC2 jumphost is created automatically.

Connect:

``` bash
chmod 400 key.pem
ssh -i key.pem ec2-user@<public_ip>
```

------------------------------------------------------------------------

# 6. Configure Jumphost

Update system:

``` bash
sudo dnf update -y
```

Install Docker:

``` bash
sudo dnf install docker -y
sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user
```

Install: - kubectl - eksctl - helm - terraform - trivy

Verify installation:

``` bash
docker --version
kubectl version
eksctl version
```

------------------------------------------------------------------------

# 7. Create EKS Cluster

``` bash
eksctl create cluster \
  --name portfolio-cluster \
  --region us-east-1 \
  --node-type t2.large \
  --nodes 2
```

Update kubeconfig:

``` bash
aws eks update-kubeconfig \
  --region us-east-1 \
  --name portfolio-cluster
```

Verify:

``` bash
kubectl get nodes
```

------------------------------------------------------------------------

# 8. Install AWS Load Balancer Controller

Purpose: Allow Kubernetes to create AWS ALB resources automatically.

Steps: - Create IAM policy - Associate OIDC provider - Create IAM
service account - Install controller via Helm

------------------------------------------------------------------------

# 9. DockerHub Integration

Generate DockerHub access token.

Add GitHub Secrets: - DOCKER_USERNAME - DOCKER_PASSWORD

CI pipeline pushes images securely.

------------------------------------------------------------------------

# 10. Code Quality & Security Tools

## SonarCloud

Performs: - Static code analysis - Maintainability checks - Bug
detection

Secrets: - SONAR_TOKEN - SONAR_ORGANIZATION - SONAR_PROJECT_KEY

------------------------------------------------------------------------

## Snyk

Scans: - Dependency vulnerabilities

Secret: - SNYK_TOKEN

------------------------------------------------------------------------

## Trivy

Scans: - Container image vulnerabilities - OS packages

Pipeline blocks high-severity vulnerabilities.

------------------------------------------------------------------------

# 11. CI Pipeline Flow

When code is pushed:

1.  Checkout repository
2.  Run SonarCloud analysis
3.  Run Snyk dependency scan
4.  Build Docker image
5.  Scan image with Trivy
6.  Push image to DockerHub

Only secure images are published.

------------------------------------------------------------------------

# 12. ArgoCD (GitOps Deployment)

Install ArgoCD:

``` bash
kubectl create namespace argocd

kubectl apply -n argocd \
https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Expose service:

``` bash
kubectl patch svc argocd-server -n argocd \
-p '{"spec": {"type": "LoadBalancer"}}'
```

Retrieve password:

``` bash
kubectl -n argocd get secret argocd-initial-admin-secret \
-o jsonpath="{.data.password}" | base64 -d
```

ArgoCD automatically deploys applications from Git.

------------------------------------------------------------------------

# 13. Monitoring Setup

Add Helm repositories:

``` bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

Install Prometheus:

``` bash
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
```

Install Grafana:

``` bash
helm install grafana grafana/grafana -n monitoring
```

Expose services as LoadBalancer.

Access Grafana dashboard to visualize cluster metrics.

------------------------------------------------------------------------

# End-to-End Flow Summary

    Developer Push
          ↓
    CI Security Checks
          ↓
    Docker Image Built
          ↓
    Image Scanned
          ↓
    Image Pushed
          ↓
    ArgoCD Detects Change
          ↓
    Kubernetes Deploys
          ↓
    Prometheus Monitors
          ↓
    Grafana Visualizes

------------------------------------------------------------------------

# What This Project Demonstrates

This implementation shows practical experience with:

-   Infrastructure as Code
-   Secure CI pipelines
-   Container vulnerability management
-   GitOps deployment
-   Kubernetes operations
-   Cloud networking
-   Cluster observability

It reflects how production-grade cloud-native applications are delivered
and maintained.
