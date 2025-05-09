# 🧠 Kubernetes + Jenkins CI/CD + AWS EKS Orchestration Guide

This document outlines the architecture, configuration, and deployment process for container orchestration using **Kubernetes**, **AWS Elastic Kubernetes Service (EKS)**, and **Jenkins CI/CD**.

---

## 🧭 Container Orchestrators

Container orchestrators are the **brain of modern containerized infrastructure**. Their responsibilities include:

- 🚀 Deploying containers across available nodes
- ⚖️ Load-balancing requests to container instances
- 🔗 Managing inter-container communication
- ♻️ Restarting failed containers automatically
- 📦 Rescheduling containers if a host fails

---

## ⚙️ Kubernetes Architecture

Kubernetes is an open-source container orchestrator designed to automate deployment, scaling, and operations of application containers.

### Node Types:
- **Control Plane Nodes**  
  Responsible for cluster management and orchestration.
  
- **Worker Nodes**  
  Execute container workloads and run Kubernetes objects like Pods.

---

## ☁️ AWS Elastic Kubernetes Service (EKS)

**EKS** is Amazon’s managed Kubernetes service. It offloads control plane management to AWS, while allowing flexibility in node provisioning.

### Key Benefits:
- High availability across multiple **Availability Zones**
- Integrated with **IAM**, **Elastic Load Balancer**, and **ECR**
- Auto-scaling and control plane load management
- Optionally use **Fargate** to eliminate node management

---

## 📦 Kubernetes Pods

Pods are the **smallest deployable unit in Kubernetes** — typically one or more containers sharing storage/network and executed together.

```yaml
# pods.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  labels:
    name: myapp
spec:
  containers:
    - name: myapp
      image: momousa1997/flask-app:latest
      resources:
        limits:
          memory: "128Mi"
          cpu: "500m"
      ports:
        - containerPort: 5000
Apply:

bash
kubectl apply -f pods.yaml
📦 Kubernetes Deployments
Deployments manage Pods and ReplicaSets, enabling:

Auto-scaling

Self-healing (pod restarts)

Rolling updates and rollbacks

yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: momousa1997/flask-app:latest
          resources:
            limits:
              memory: "128Mi"
              cpu: "500m"
          ports:
            - containerPort: 5000
Apply:

bash
kubectl apply -f deployment.yaml
📁 Kubeconfig Explained
kubeconfig is the config file used by kubectl to connect to Kubernetes clusters.

Default location:
bash
Copy code
$HOME/.kube/config
Use cases:
Set as KUBECONFIG environment variable

Switch context with:

bash
Copy code
kubectl config use-context cluster1
kubectl config use-context cluster2
Example structure:
yaml
Copy code
apiVersion: v1
clusters:
- cluster:
    server: https://192.168.59.123:8443
  name: cluster1
contexts:
- context:
    cluster: cluster1
    user: john
  name: john@cluster1
current-context: john@cluster1
users:
- name: john
  user:
    client-certificate: path/to/cert
    client-key: path/to/key
🔁 Jenkins CI/CD Pipeline for EKS Deployment
This pipeline:

Installs Python dependencies

Runs tests

Builds + pushes Docker images

Deploys to staging and prod EKS clusters

groovy
pipeline {
    agent any

    environment {
        IMAGE_NAME = 'momousa1997/flask-app'
        IMAGE_TAG = "${IMAGE_NAME}:${env.BUILD_NUMBER}"
        PATH = "/var/lib/jenkins/.local/bin:$PATH"
        KUBECONFIG = credentials('kubeconfig-credentials-id')
        AWS_ACCESS_KEY_ID = credentials('aws-access-key')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')
    }

    stages {
        stage('Setup') {
            steps {
                dir('04-docker') {
                    sh '''
                        pip install --user --upgrade pip
                        pip install --user -r requirements.txt
                    '''
                }
                sh '''
                    chmod 644 $KUBECONFIG
                '''
            }
        }

        stage('Test') {
            steps {
                dir('04-docker') {
                    sh '~/.local/bin/pytest'
                }
            }
        }

        stage('Login to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-creds', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                    sh 'echo $PASSWORD | docker login -u $USERNAME --password-stdin'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                dir('04-docker') {
                    sh 'docker build -t $IMAGE_TAG .'
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                sh 'docker push $IMAGE_TAG'
            }
        }

        stage('Deploy to Staging') {
            steps {
                sh '''
                    kubectl config use-context arn:aws:eks:eu-west-2:568645574857:cluster/staging
                    kubectl set image deployment/flask-app flask-app=$IMAGE_TAG
                '''
            }
        }

        stage('Acceptance Test') {
            steps {
                script {
                    def service = sh(
                        script: "kubectl get svc flask-app-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}:{.spec.ports[0].port}'",
                        returnStdout: true
                    ).trim()
                    echo "Service URL: ${service}"
                    sh "k6 run -e SERVICE=${service} acceptance-test.js"
                }
            }
        }

        stage('Deploy to Prod') {
            steps {
                sh '''
                    kubectl config use-context arn:aws:eks:eu-west-2:568645574857:cluster/prod
                    kubectl set image deployment/flask-app flask-app=$IMAGE_TAG
                '''
            }
        }
    }
}
🧪 Useful Commands Cheat Sheet
bash
# Create EKS cluster
eksctl create cluster \
  --name prod \
  --region eu-west-2 \
  --nodegroup-name prod-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed

# Addons
eksctl create addon --name vpc-cni --cluster prod --region eu-west-2 --force
eksctl create addon --name kube-proxy --cluster prod --region eu-west-2 --force
eksctl create addon --name coredns --cluster prod --region eu-west-2 --force

# Update kubeconfig
aws eks update-kubeconfig --region eu-west-2 --name prod

# Cluster context
kubectl config use-context arn:aws:eks:eu-west-2:568645574857:cluster/prod
kubectl get nodes
kubectl get pods -A
🔐 IAM Access for kubectl via ConfigMap Patch
To grant an IAM user admin access to EKS:

yaml
# aws-auth-patch.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::568645574857:role/eksctl-prod-nodegroup-prod-nodes-NodeInstanceRole-XYZ
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
  mapUsers: |
    - userarn: arn:aws:iam::568645574857:user/mmagdy
      username: mmagdy
      groups:
        - system:masters
Apply:
kubectl apply -f aws-auth-patch.yaml

when you want to add credentials -> for kubeconfig -> go to c -> users -> .lube -> config -> under id add kubeconfig-credentials-id

kubectl create namespace staging

kubectl create namespace production

Objective: Learn about Kubernetes pod

Draft a Kubernetes pod definition in a YAML file, specifying the container nginx and exposing it on port 8080.
Deploy the pod to your Kubernetes cluster using Kubectl.

Deploy the pod using the below manifest file.

apiVersion: v1
kind: Pod
metadata:
  name: webapp
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 8080

Objective: Learn how to implement downtime and no downtime deployment strategies

Edit the /root/deployment_production.yaml manifest file to use rolling update strategy with the below specifications and deploy it.

maxUnavailable: 1
maxSurge: 1
Edit the /root/deployment_staging.yaml manifest file to use the Recreate strategy and deploy it.

Include the RollingUpdate strategy in the /root/deployment_production.yaml manifest as specified below and then deploy it.

 selector:
    matchLabels:
      app: flask-app
  strategy:                         #added
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1                   #end
  template:
    metadata:
Include the Recreate strategy in the deployment_staging.yaml manifest as specified below and then deploy it.

selector:
    matchLabels:
      app: flask-app
  strategy:                        #added
   type: Recreate                  #end
  template:
    metadata:
      labels:
        app: flask-app

Objective: Learn how to deploy container app to the EKS cluster using Jenkins CI/CD pipeline

Add your EKS cluster credentials to Jenkins as a secret file named kubeconfig for access

Add a new stage named Deploy to EKS Prod env to the deployToProd pipeline to deploy the application to EKS using kubectl commands, with the manifest file deployment_production.yaml already present in the repo.

repo- https://github.com/kodekloudhub/jenkins-project.git

Run the Jenkins pipeline to automate the deployment process

Jenkins Credentials:
User: admin
Password: Adm!n321

Install AWS Credentials plugin
Create a new credential of kind aws credentials and name it aws.
Create a secret file type credential named kubeconfig for access
Append the stage to the deployToProd job to deploy the application to the EKS cluster
pipeline {
    agent any
        environment {
        KUBECONFIG = credentials('kubeconfig')
    }

    stages {
        stage('Checkout') {
            steps {
                sh 'git clone https://github.com/kodekloudhub/jenkins-project.git'
                sh "ls -ltr"
            }
        }
        stage('Approve Default Image') {
            steps {
                script {
                    env.PARAMETER_VALUE = 'sanjeevkt720/jenkins-flask-app:v5'
                    echo 'Using default Docker image: ${env.PARAMETER_VALUE}'
                }
            }
        }
        stage('Deploy to EKS Prod env')     #added
            {
            steps
                {
                script 
                    {   
                        withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY', credentialsId: 'aws')]) {
                        def deploymentExists = sh(script: "kubectl get deployment flask-app-deployment-prod -n prod  -o json", returnStatus: true)
                            if (deploymentExists == 0) 
                                {
                                    sh "kubectl set image -n prod deployment/flask-app-deployment-prod flask-app=${env.PARAMETER_VALUE} --record"
                                    echo "Kubernetes deployment updated successfully"
                                } 
                            else 
                                {
                                    sh "kubectl apply -f jenkins-project/deployment_production.yaml"
                                    echo "Kubernetes deployment created successfully"
                                }
                        }
                    }
                }
            }
    }
}

