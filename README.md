# 🚀 Jenkins DevOps Course Project Repository

Welcome to the **course-jenkins-project** — a comprehensive hands-on DevOps lab repository designed to demonstrate modern CI/CD workflows using **Jenkins**, **Docker**, **AWS Lambda**, and **Kubernetes (EKS)**.

This project is a refined fork of [`kodekloudhub/course-jenkins-project`](https://github.com/kodekloudhub/course-jenkins-project), with advanced extensions and real-world production-grade configurations contributed by [momagdyy97](https://github.com/momagdyy97).

---

## 📁 Repository Structure

Each subdirectory in this repo represents a **progressive DevOps concept or deployment strategy**. Below is a summary of what's inside:

### `01-jenkins-basics/`
> ✅ **Goal**: Introduce Jenkins freestyle and pipeline jobs  
> 📦 Contains:  
> - Simple Jenkinsfile pipelines  
> - Hello-world Python/Node apps  
> - Job parameterization and environment variables

---

### `02-single-server-deployment/`
> ✅ **Goal**: Deploy applications directly to a target Linux server (via SSH or local runner)  
> 📦 Contains:  
> - Jenkinsfile for SCP/SSH-based deployment  
> - Systemd service setup  
> - Bash automation scripts

---

### `03-lambda-deployment/`
> ✅ **Goal**: Automate deployment of AWS Lambda functions using Jenkins  
> 📦 Contains:  
> - Python Lambda function code  
> - Jenkinsfile integrating AWS CLI and IAM roles  
> - SAM-based deployment approach (optional)

---

### `04-docker/`
> ✅ **Goal**: Containerize and deploy applications using Docker  
> 📦 Contains:  
> - Flask app source code  
> - Dockerfile with multistage builds  
> - Jenkinsfile for build + test + push to Docker Hub  
> - Uses `pytest` and `k6` for testing phases

---

### `05-kubernetes/`
> ✅ **Goal**: Deploy containerized apps to Kubernetes (AWS EKS)  
> 📦 Contains:  
> - Pod and Deployment YAMLs  
> - Ingress & Service manifests  
> - Jenkins pipeline deploying to EKS (`prod` context)  
> - AWS IAM integration and cluster context configuration

---

### `06-advanced-project/` *(WIP or archived)*
> 🔧 **Goal**: Placeholder for advanced full-stack or microservices deployments  
> 📦 Contains:  
> - Legacy test projects  
> - Templates for Jenkins multi-branch pipelines

---

## 🔐 Secrets & Credentials (managed via Jenkins)
- Docker Hub Credentials (`docker-creds`)
- AWS Access Key / Secret (`aws-access-key`, `aws-secret-key`)
- Kubeconfig (`kubeconfig-credentials-id`)

> All secrets are referenced securely using Jenkins credentials store.

---

## ✅ Prerequisites

- AWS CLI & IAM setup with EKS permissions
- Docker Hub account for pushing container images
- Jenkins master with required plugins:
  - Pipeline
  - Git
  - Docker Pipeline
  - Kubernetes CLI
  - Credentials Binding

---

## 🧪 Sample Commands

```bash
# Create EKS Cluster (prod)
eksctl create cluster --name prod --region eu-west-2 \
  --nodegroup-name prod-nodes --node-type t3.medium \
  --nodes 2 --nodes-min 1 --nodes-max 3 --managed

# Update kubeconfig
aws eks update-kubeconfig --region eu-west-2 --name prod

# Verify connection
kubectl get nodes

👤 Maintained by
Mohamed Magdy
AWS DevOps & Cloud Engineer

📄 License
This repository is based on KodeKloud's Jenkins course, extended under fair use for learning, customization, and deployment practice. Original copyright belongs to the respective authors.
