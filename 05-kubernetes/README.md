**-------------------------------------------------------**
**Container Orchestrators**
- They are the brains of a containerized environments.
**-------------------------------------------------------**
**Reponsibilities**
- Deploying containers across all available servers.
- Load-balancing requests to containers.
- Providing container to container connectivity.
- Restarting failed containers.
- Move containers when hosts fails.
**-------------------------------------------------------**
**Kubernetes Architecture**
- Kubernetes is an open-source container orchestrator.
- Kubernetes cluster has two types of nodes:-
    - Control-Plane nodes - managers of the cluster
        - Watch over cluster and make sure cluster is kept in a safe working state.
    - Worker-Nodes - responsible for actually running the containerized workloads.
**-------------------------------------------------------**
- AWS ELASTIC KUBERNETES SERVICES is managed Kubernetes service.
- EKS manages the control-plane for you.
- Users are still responsible for managing worker nodes.
    - Unless you descide to use Fargate (AWS will manage the nodes).
Benefits:-
- Runs & Scales control-plane across multiple availability zones.
- Scales control-plane instances based on load.
- Can integrate with other AWS services.
- IAM for authentication.
- Elastic Load Balancer.
- ECR for container images.
**-------------------------------------------------------**
- Kubernetes Pods:- They are the smallest deployable units of Kubernetes. In its most basic definition, a Pod is a single running instance of a process in a cluster, possibly compromising many containers managed as a single unit by Kubernetes.
--------------
pods.yml
--------------
apiVersion:v1
kind: Pod
metadata:
  name:myapp
  labels:
    name:myapp
spec:
  containers:
  - name:myapp
    image: momousa1997/flask-app:latest
    resources:
      limits:
        memory:"128Mi"
        cpu: "500m"
      ports:
       - containerPort:5000
- kubectl apply -f pod.yml
**-------------------------------------------------------**
Kubernetes: Deployments
- They watch over pods and restart failed pods.
- They scale number of pod instances.
- Provides upgraded and rollback functionality
-----------------
deployment.yaml
-----------------
apiVersion: apps/v1
kind: Deployment
metadata:
  name:myapp
spec:
  replicas:3
  selector:
    matchLabels:
      app:myapp
  template:
    metadata:
      labels:
        app:myapp
    spec:
      containers:
      - name:myapp
        image: momousa1997/flask-app:latest
        resources:
          limits:
            memory:"128Mi"
            cpu: "500m"
          ports:
          - containerPort:5000
- kubectl apply -f deployment.yaml
**-------------------------------------------------------**
kubeconfig file:- It provides all of the configuration and information necessary to connect to and authenticate with various kubernetes clusters that you and organization utilize. -> by default will look for the file first to find information about the cluster.
---------------------------------------------------------------------------------------------------------------------------------------
- $HOME/.kube/config/ 
- We can add it as environment variable -> KUBECONFIG
**---------------------------------------------------------------------------------------------------------------------------------------**
apiVersion: v1
clusters:
- cluster:
    server: https://192.168.54.19:8443
  name: cluster2
- cluster:
    server: https://192.168.59.123:8443
  name: cluster1
contexts: // Where you bind user to the cluster
- context:
    cluster: cluster1
    user: john
  name: john@cluster1
- context:
    cluster: cluster2
    user: mike
  name: mike@cluster2
  current-context: cluster1
kind: config
preferences: {}
users:
- name: john
  user:
    client-certificate:
    client-key:
- name: mike
  user:
    client-certificate:
    client-key
**---------------------------------------------------------------------------------------------------------------------------------------**
Kubernetes: Kubeconfig
Directory: $HOME/.kube/config
Enironment Variable: KUBECONFIG
Command Line Argument: Kubectl --kubeconfig <path-to-file>

kubectl config use-context cluster1
kubectl config use-context cluster2
**---------------------------------------------------------------------------------------------------------------------------------------**
Jenkins pipeline:-
------------------
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
                        echo "Setting up Python environment"
                        pip install --user --upgrade pip
                        pip install --user -r requirements.txt
                    '''
                }
                sh '''
                    echo "Setting KUBECONFIG permissions"
                    ls -la $KUBECONFIG
                    chmod 644 $KUBECONFIG
                    ls -la $KUBECONFIG
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
                echo 'Docker Hub login successful'
            }
        }

        stage('Build Docker Image') {
            steps {
                dir('04-docker') {
                    sh '''
                        docker build -t $IMAGE_TAG .
                        docker image ls
                    '''
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                sh 'docker push $IMAGE_TAG'
                echo 'Docker image pushed successfully'
            }
        }

        stage('Deploy to Staging') {
            steps {
                sh '''
                    echo "Deploying to STAGING cluster..."
                    kubectl config use-context arn:aws:eks:eu-west-2:568645574857:cluster/staging
                    kubectl config current-context
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
                    echo "Deploying to PROD cluster..."
                    kubectl config use-context arn:aws:eks:eu-west-2:568645574857:cluster/prod
                    kubectl config current-context
                    kubectl set image deployment/flask-app flask-app=$IMAGE_TAG
                '''
            }
        }
    }
}
------------------------------------------------------------------------------------------------------
Important Commands:-
- eksctl create cluster \
  --name prod \
  --region eu-west-2 \
  --nodegroup-name prod-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed
- eksctl create addon --name vpc-cni --cluster prod --region eu-west-2 --force
- aws eks update-kubeconfig --region eu-west-2 --name prod
- kubectl config current-context
- kubectl config use-context arn:aws:eks:eu-west-2:568645574857:cluster/prod
- kubectl get nodes
- kubectl get pods -A
- Install Core Add-ons (like VPC CNI, CoreDNS, etc):
eksctl create addon --name vpc-cni --cluster prod --region eu-west-2 --force
eksctl create addon --name kube-proxy --cluster prod --region eu-west-2 --force
eksctl create addon --name coredns --cluster prod --region eu-west-2 --force
- Patch IAM access (if not done):
If you want your IAM user (e.g., mmagdy) to run kubectl and administer the cluster:
# aws-auth-patch.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::568645574857:role/eksctl-prod-nodegroup-prod-nodes-NodeInstanceRole-<suffix>
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
  mapUsers: |
    - userarn: arn:aws:iam::568645574857:user/mmagdy
      username: mmagdy
      groups:
        - system:masters
--------------------------------------------------------

