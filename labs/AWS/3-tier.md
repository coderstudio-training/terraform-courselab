# Deploying a 3-Tier Web Application

This comprehensive tutorial will guide you through the process of deploying and configuring a 3-tier web application using Docker, Kubernetes, Terraform, and Ansible. We'll cover each step in detail, providing explanations, code samples, and best practices.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Application Architecture](#application-architecture)
3. [Containerizing the Application with Docker](#containerizing-the-application-with-docker)
4. [Setting up the Infrastructure with Terraform](#setting-up-the-infrastructure-with-terraform)
5. [Configuring the Environment with Ansible](#configuring-the-environment-with-ansible)
6. [Deploying the Application to Kubernetes](#deploying-the-application-to-kubernetes)
7. [Monitoring and Logging](#monitoring-and-logging)
8. [CI/CD Pipeline](#cicd-pipeline)
9. [Security Considerations](#security-considerations)
10. [Conclusion and Next Steps](#conclusion-and-next-steps)

## Prerequisites

Before starting this tutorial, ensure you have the following:

- A local development environment with:
  - Docker (v20.10 or later)
  - kubectl (v1.21 or later)
  - Terraform (v1.0 or later)
  - Ansible (v2.9 or later)
- An AWS account with appropriate permissions
- AWS CLI configured with your credentials
- Basic knowledge of JavaScript (for frontend), Python (for backend), and SQL
- Git for version control

## Application Architecture

Our 3-tier web application consists of:

1. Frontend: A React.js application
2. Backend: A Python Flask API
3. Database: MySQL database

Here's a high-level architecture diagram:

```
[Frontend (React)] <--> [Backend (Flask)] <--> [Database (MySQL)]
```

## Containerizing the Application with Docker

### Frontend Dockerfile

Create a `Dockerfile` in your frontend directory:

```dockerfile
# Use an official Node runtime as the base image
FROM node:20-alpine-alpine

# Set the working directory in the container
WORKDIR /app

# Copy package.json and package-lock.json
COPY package*.json ./

# Install dependencies
RUN npm ci

# Copy the rest of the application code
COPY . .

# Build the application
RUN npm run build

# Use nginx to serve the static files
FROM nginx:alpine
COPY --from=0 /app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

This Dockerfile uses a multi-stage build to create a smaller final image. It builds the React application and then serves it using nginx.

### Backend Dockerfile

Create a `Dockerfile` in your backend directory:

```dockerfile
# Use an official Python runtime as the base image
FROM python:3.9-slim-buster

# Set the working directory in the container
WORKDIR /app

# Copy the requirements file
COPY requirements.txt .

# Install dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy the rest of the application code
COPY . .

# Expose the port the app runs on
EXPOSE 5000

# Command to run the application
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
```

This Dockerfile sets up a Python environment, installs dependencies, and uses gunicorn to run the Flask application.

### Building and Pushing Docker Images

Build the Docker images:

```bash
docker build -t your-dockerhub-username/frontend:v1 ./frontend
docker build -t your-dockerhub-username/backend:v1 ./backend
```

Push the images to Docker Hub (or your preferred container registry):

```bash
docker push your-dockerhub-username/frontend:v1
docker push your-dockerhub-username/backend:v1
```

## Setting up the Infrastructure with Terraform

Create a `main.tf` file to define your AWS infrastructure:

```hcl
provider "aws" {
  region = "us-west-2"
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true

  tags = {
    Name = "main-vpc"
  }
}

# Public Subnet
resource "aws_subnet" "public" {
  count                   = 2
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.${count.index + 1}.0/24"
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "public-subnet-${count.index + 1}"
  }
}

# Private Subnet
resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 101}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "private-subnet-${count.index + 1}"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "main-igw"
  }
}

# EKS Cluster
resource "aws_eks_cluster" "main" {
  name     = "main-cluster"
  role_arn = aws_iam_role.eks_cluster.arn

  vpc_config {
    subnet_ids = aws_subnet.public[*].id
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_cluster_AmazonEKSClusterPolicy,
  ]
}

# EKS Node Group
resource "aws_eks_node_group" "main" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "main-node-group"
  node_role_arn   = aws_iam_role.eks_node_group.arn
  subnet_ids      = aws_subnet.private[*].id

  scaling_config {
    desired_size = 2
    max_size     = 3
    min_size     = 1
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_node_group_AmazonEKSWorkerNodePolicy,
    aws_iam_role_policy_attachment.eks_node_group_AmazonEKS_CNI_Policy,
    aws_iam_role_policy_attachment.eks_node_group_AmazonEC2ContainerRegistryReadOnly,
  ]
}

# ... (IAM roles and policy attachments for EKS cluster and node group)
```

This Terraform configuration sets up a VPC with public and private subnets, an internet gateway, and an EKS cluster with a node group.

Apply the Terraform configuration:

```bash
terraform init
terraform apply
```

## Configuring the Environment with Ansible

Create an Ansible playbook (`configure.yml`) to set up the necessary software and configurations:

```yaml
---
- hosts: all
  become: yes
  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
      when: ansible_os_family == "Debian"

    - name: Install Docker
      package:
        name: docker
        state: present

    - name: Install kubectl
      get_url:
        url: https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubectl
        dest: /usr/local/bin/kubectl
        mode: '0755'

    - name: Install AWS CLI
      pip:
        name: awscli
        state: present

    - name: Configure AWS CLI
      command: aws configure set {{ item.key }} {{ item.value }}
      loop:
        - { key: 'aws_access_key_id', value: '{{ aws_access_key }}' }
        - { key: 'aws_secret_access_key', value: '{{ aws_secret_key }}' }
        - { key: 'region', value: '{{ aws_region }}' }
      no_log: true

    - name: Update kubeconfig
      command: aws eks get-token --cluster-name main-cluster | kubectl apply -f -

    # Add more tasks as needed
```

Create an inventory file (`inventory.ini`) with your target hosts:

```ini
[eks_nodes]
node1 ansible_host=<EC2_INSTANCE_IP_1>
node2 ansible_host=<EC2_INSTANCE_IP_2>

[eks_nodes:vars]
ansible_user=ec2-user
ansible_ssh_private_key_file=/path/to/your/ssh/key.pem
```

Run the Ansible playbook:

```bash
ansible-playbook -i inventory.ini configure.yml --extra-vars "aws_access_key=<YOUR_ACCESS_KEY> aws_secret_key=<YOUR_SECRET_KEY> aws_region=us-west-2"
```

## Deploying the Application to Kubernetes

Create Kubernetes manifests for each tier of your application.

### Frontend Deployment and Service (frontend.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: your-dockerhub-username/frontend:v1
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
```

### Backend Deployment and Service (backend.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: your-dockerhub-username/backend:v1
        ports:
        - containerPort: 5000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: database-url
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
    - protocol: TCP
      port: 5000
      targetPort: 5000
```

### Database Deployment and Service (database.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: database
        image: mysql:5.7
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: root-password
        - name: MYSQL_DATABASE
          value: myapp
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 1
            memory: 2Gi
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
---
apiVersion: v1
kind: Service
metadata:
  name: database-service
spec:
  selector:
    app: database
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306
```

Create a Secret for database credentials:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secrets
type: Opaque
data:
  root-password: <base64-encoded-password>
  database-url: <base64-encoded-database-url>
```

Apply the Kubernetes manifests:

```bash
kubectl apply -f frontend.yaml
kubectl apply -f backend.yaml
kubectl apply -f database.yaml
kubectl apply -f db-secrets.yaml
```

## Monitoring and Logging

Set up Prometheus and Grafana for monitoring:

1. Install Helm:

```bash
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```

2. Add the Prometheus community Helm repository:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

3. Install Prometheus and Grafana:

```bash
helm install prometheus prometheus-community/kube-prometheus-stack
```

4. Access Grafana dashboard:

```bash
kubectl port-forward deployment/prometheus-grafana 3000
```

Visit `http://localhost:3000` and log in with the default credentials (admin/prom-operator).

## CI/CD Pipeline

Here's a sample GitLab CI/CD pipeline (`.gitlab-ci.yml`):

```yaml
stages:
  - build
  - test
  - deploy

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: ""

build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $CI_REGISTRY_IMAGE/frontend:$CI_COMMIT_SHA ./frontend
    - docker push $CI_REGISTRY_IMAGE/frontend:$CI_COMMIT_SHA
    - docker build -t $CI_REGISTRY_IMAGE/backend:$CI_COMMIT_SHA ./backend
    - docker push $CI_REGISTRY_IMAGE/backend:$CI_COMMIT_SHA

test:
  stage: test
  image: python:3.9
  script:
    - cd backend
    - pip install -r requirements.txt
    - python -m pytest

deploy:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl set image deployment/frontend frontend=$CI_REGISTRY_IMAGE/frontend:$CI_COMMIT_SHA
    - kubectl set image deployment/backend backend=$CI_REGISTRY_IMAGE/backend:$CI_COMMIT_SHA
  only:
    - main
```

## Security Considerations

1. Network Policies:

Create a `network-policy.yaml` file:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress

---

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
  policyTypes:
