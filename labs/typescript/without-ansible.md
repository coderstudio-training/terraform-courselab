# Deploying a TypeScript Stack on K3s with Terraform

This guide will walk you through the process of setting up and deploying a full-stack TypeScript application on a local Kubernetes (k3s) cluster, using Terraform for infrastructure management.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Project Structure](#project-structure)
3. [Setting Up the Local Environment](#setting-up-the-local-environment)
4. [Frontend Setup (React with TypeScript)](#frontend-setup-react-with-typescript)
5. [Backend Setup (Node.js with TypeScript)](#backend-setup-nodejs-with-typescript)
6. [Database Setup (PostgreSQL)](#database-setup-postgresql)
7. [Containerization with Docker](#containerization-with-docker)
8. [Infrastructure as Code with Terraform](#infrastructure-as-code-with-terraform)
9. [Deployment Process](#deployment-process)
10. [Testing the Application](#testing-the-application)
11. [Troubleshooting](#troubleshooting)

## Prerequisites

Ensure you have the following installed on your local machine:
- Docker
- Terraform
- kubectl
- Node.js and npm

## Project Structure

Create the following directory structure:

```
project-root/
├── frontend/
├── backend/
├── terraform/
└── scripts/
```

## Setting Up the Local Environment

1. Install k3s:

```bash
curl -sfL https://get.k3s.io | sh -
```

2. Configure kubectl:

```bash
mkdir ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
```

## Frontend Setup (React with TypeScript)

1. Create a new React app with TypeScript:

```bash
npx create-react-app frontend --template typescript
cd frontend
```

2. Install additional dependencies:

```bash
npm install axios
```

3. Replace the contents of `src/App.tsx` with the provided React component code.

## Backend Setup (Node.js with TypeScript)

1. Set up a new Node.js project with TypeScript:

```bash
mkdir backend && cd backend
npm init -y
npm install express pg cors
npm install --save-dev typescript @types/express @types/pg @types/cors ts-node
npx tsc --init
```

2. Update `tsconfig.json` with the provided configuration.

3. Create `src/server.ts` with the provided Node.js server code.

4. Update `package.json` scripts as shown in the original document.

## Database Setup (PostgreSQL)

We'll use a PostgreSQL container in our k3s cluster. The setup will be handled by Terraform in later steps.

## Containerization with Docker

1. Create a `Dockerfile` in the frontend directory with the provided Dockerfile content.

2. Create a `Dockerfile` in the backend directory with the provided Dockerfile content.

3. Build the Docker images:

```bash
cd frontend
docker build -t frontend:latest .
cd ../backend
docker build -t backend:latest .
```

## Infrastructure as Code with Terraform

1. In the `terraform` directory, create `main.tf` with the provided Terraform configuration.

## Deployment Process

1. Navigate to the project root directory.

2. Initialize Terraform:

```bash
cd terraform
terraform init
```

3. Apply the Terraform configuration:

```bash
terraform apply
```

4. Initialize the PostgreSQL database:

```bash
kubectl exec -it -n myapp $(kubectl get pods -n myapp -l app=postgres -o jsonpath='{.items[0].metadata.name}') -- psql -U postgres -d myapp -c "CREATE TABLE IF NOT EXISTS messages (id SERIAL PRIMARY KEY, text TEXT NOT NULL);"
```

## Testing the Application

1. Access the frontend:
   Open a web browser and navigate to `http://localhost:30080`

2. Test the interaction:
   - You should see the React frontend with a message from the server (initially "No messages yet").
   - Enter a new message in the input field and click "Submit".
   - The new message should be displayed after submission.

## Troubleshooting

If you encounter issues:

1. Check pod status:
   ```bash
   kubectl get pods -n myapp
   ```

2. View pod logs:
   ```bash
   kubectl logs -n myapp <pod-name>
   ```

3. Access the backend directly:
   ```bash
   kubectl port-forward -n myapp service/backend 3000:3000
   ```
   Then, in another terminal:
   ```bash
   curl http://localhost:3000/api/message
   ```

4. Check database connectivity:
   ```bash
   kubectl exec -it -n myapp <postgres-pod-name> -- psql -U postgres -d myapp -c "SELECT * FROM messages;"
   ```

5. Review Terraform state:
   ```bash
   cd terraform && terraform show
   ```

6. For TypeScript-specific issues:
   - Check compilation errors:
     ```bash
     cd frontend && npm run build
     cd ../backend && npm run build
     ```
   - Verify TypeScript configs:
     ```bash
     cat tsconfig.json
     ```

Remember, without Ansible, you'll need to perform these steps manually and ensure all prerequisites are installed on your system. This process requires more direct interaction but gives you more control over each step of the deployment.
