# Deploying a TypeScript Stack on K3s with Terraform and Ansible

This guide will walk you through the process of setting up and deploying a full-stack TypeScript application on a local Kubernetes (k3s) cluster, using Terraform for infrastructure management and Ansible for automation.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Project Structure](#project-structure)
3. [Setting Up the Local Environment](#setting-up-the-local-environment)
4. [Frontend Setup (React with TypeScript)](#frontend-setup-react-with-typescript)
5. [Backend Setup (Node.js with TypeScript)](#backend-setup-nodejs-with-typescript)
6. [Database Setup (PostgreSQL)](#database-setup-postgresql)
7. [Containerization with Docker](#containerization-with-docker)
8. [Infrastructure as Code with Terraform](#infrastructure-as-code-with-terraform)
9. [Automation with Ansible](#automation-with-ansible)
10. [Deployment Process](#deployment-process)
11. [Testing the Application](#testing-the-application)
12. [Troubleshooting](#troubleshooting)

## Prerequisites

Ensure you have the following installed on your local machine:
- Docker
- Terraform
- Ansible
- kubectl
- Node.js and npm

## Project Structure

Create the following directory structure:

```
project-root/
├── frontend/
├── backend/
├── terraform/
├── ansible/
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

3. Replace the contents of `src/App.tsx` with:

```tsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

interface Message {
  text: string;
}

function App() {
  const [message, setMessage] = useState<string>('');
  const [inputText, setInputText] = useState<string>('');

  useEffect(() => {
    fetchMessage();
  }, []);

  const fetchMessage = async () => {
    try {
      const response = await axios.get<Message>('http://localhost:3000/api/message');
      setMessage(response.data.text);
    } catch (error) {
      console.error('Error fetching message:', error);
    }
  };

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    try {
      await axios.post<Message>('http://localhost:3000/api/message', { text: inputText });
      fetchMessage();
      setInputText('');
    } catch (error) {
      console.error('Error submitting message:', error);
    }
  };

  return (
    <div>
      <h1>K3s React App with TypeScript</h1>
      <p>Message from server: {message}</p>
      <form onSubmit={handleSubmit}>
        <input
          type="text"
          value={inputText}
          onChange={(e) => setInputText(e.target.value)}
          placeholder="Enter a new message"
        />
        <button type="submit">Submit</button>
      </form>
    </div>
  );
}

export default App;
```

## Backend Setup (Node.js with TypeScript)

1. Set up a new Node.js project with TypeScript:

```bash
mkdir backend && cd backend
npm init -y
npm install express pg cors
npm install --save-dev typescript @types/express @types/pg @types/cors ts-node
npx tsc --init
```

2. Update `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "es6",
    "module": "commonjs",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true
  },
  "include": ["src/**/*"]
}
```

3. Create `src/server.ts`:

```typescript
import express, { Request, Response } from 'express';
import { Pool } from 'pg';
import cors from 'cors';

const app = express();
app.use(cors());
app.use(express.json());

const pool = new Pool({
  user: 'postgres',
  host: 'postgres',
  database: 'myapp',
  password: 'password',
  port: 5432,
});

app.get('/api/message', async (req: Request, res: Response) => {
  try {
    const result = await pool.query('SELECT text FROM messages ORDER BY id DESC LIMIT 1');
    res.json({ text: result.rows[0]?.text || 'No messages yet' });
  } catch (error) {
    console.error('Error fetching message:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

app.post('/api/message', async (req: Request, res: Response) => {
  try {
    const { text } = req.body;
    await pool.query('INSERT INTO messages (text) VALUES ($1)', [text]);
    res.status(201).json({ message: 'Message saved successfully' });
  } catch (error) {
    console.error('Error saving message:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

const port = 3000;
app.listen(port, () => {
  console.log(`Server running on port ${port}`);
});
```

4. Update `package.json` scripts:

```json
"scripts": {
  "start": "node dist/server.js",
  "build": "tsc",
  "dev": "ts-node src/server.ts"
}
```

## Database Setup (PostgreSQL)

We'll use a PostgreSQL container in our k3s cluster. The setup will be handled by Terraform and Ansible in later steps.

## Containerization with Docker

1. Create a `Dockerfile` in the frontend directory:

```dockerfile
FROM node:20-alpine as build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

2. Create a `Dockerfile` in the backend directory:

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build
EXPOSE 3000
CMD ["npm", "start"]
```

## Infrastructure as Code with Terraform

1. In the `terraform` directory, create `main.tf`:

```hcl
terraform {
  required_providers {
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
  }
}

provider "kubernetes" {
  config_path = "~/.kube/config"
}

resource "kubernetes_namespace" "myapp" {
  metadata {
    name = "myapp"
  }
}

resource "kubernetes_deployment" "postgres" {
  metadata {
    name      = "postgres"
    namespace = kubernetes_namespace.myapp.metadata[0].name
  }
  spec {
    replicas = 1
    selector {
      match_labels = {
        app = "postgres"
      }
    }
    template {
      metadata {
        labels = {
          app = "postgres"
        }
      }
      spec {
        container {
          image = "postgres:13"
          name  = "postgres"
          env {
            name  = "POSTGRES_DB"
            value = "myapp"
          }
          env {
            name  = "POSTGRES_PASSWORD"
            value = "password"
          }
          port {
            container_port = 5432
          }
        }
      }
    }
  }
}

resource "kubernetes_service" "postgres" {
  metadata {
    name      = "postgres"
    namespace = kubernetes_namespace.myapp.metadata[0].name
  }
  spec {
    selector = {
      app = kubernetes_deployment.postgres.spec[0].template[0].metadata[0].labels.app
    }
    port {
      port        = 5432
      target_port = 5432
    }
  }
}

resource "kubernetes_deployment" "backend" {
  metadata {
    name      = "backend"
    namespace = kubernetes_namespace.myapp.metadata[0].name
  }
  spec {
    replicas = 1
    selector {
      match_labels = {
        app = "backend"
      }
    }
    template {
      metadata {
        labels = {
          app = "backend"
        }
      }
      spec {
        container {
          image = "backend:latest"
          name  = "backend"
          image_pull_policy = "Never"
          port {
            container_port = 3000
          }
        }
      }
    }
  }
}

resource "kubernetes_service" "backend" {
  metadata {
    name      = "backend"
    namespace = kubernetes_namespace.myapp.metadata[0].name
  }
  spec {
    selector = {
      app = kubernetes_deployment.backend.spec[0].template[0].metadata[0].labels.app
    }
    port {
      port        = 3000
      target_port = 3000
    }
  }
}

resource "kubernetes_deployment" "frontend" {
  metadata {
    name      = "frontend"
    namespace = kubernetes_namespace.myapp.metadata[0].name
  }
  spec {
    replicas = 1
    selector {
      match_labels = {
        app = "frontend"
      }
    }
    template {
      metadata {
        labels = {
          app = "frontend"
        }
      }
      spec {
        container {
          image = "frontend:latest"
          name  = "frontend"
          image_pull_policy = "Never"
          port {
            container_port = 80
          }
        }
      }
    }
  }
}

resource "kubernetes_service" "frontend" {
  metadata {
    name      = "frontend"
    namespace = kubernetes_namespace.myapp.metadata[0].name
  }
  spec {
    type = "NodePort"
    selector = {
      app = kubernetes_deployment.frontend.spec[0].template[0].metadata[0].labels.app
    }
    port {
      port        = 80
      target_port = 80
      node_port   = 30080
    }
  }
}
```

## Automation with Ansible

1. In the `ansible` directory, create `playbook.yml`:

```yaml
---
- hosts: localhost
  connection: local
  become: yes
  tasks:
    - name: Ensure Docker is installed
      apt:
        name: docker.io
        state: present
      when: ansible_os_family == "Debian"

    - name: Ensure k3s is installed
      shell: curl -sfL https://get.k3s.io | sh -
      args:
        creates: /usr/local/bin/k3s

    - name: Copy k3s config
      copy:
        src: /etc/rancher/k3s/k3s.yaml
        dest: ~/.kube/config
        remote_src: yes
        owner: "{{ ansible_user_id }}"
        group: "{{ ansible_user_id }}"
        mode: '0600'

    - name: Build frontend Docker image
      docker_image:
        name: frontend
        build:
          path: ../frontend
          args:
            NODE_ENV: production
        source: build
        force_source: yes

    - name: Build backend Docker image
      docker_image:
        name: backend
        build:
          path: ../backend
          args:
            NODE_ENV: production
        source: build
        force_source: yes

    - name: Apply Terraform configuration
      terraform:
        project_path: ../terraform
        state: present

    - name: Initialize PostgreSQL database
      kubernetes:
        definition:
          apiVersion: v1
          kind: Pod
          metadata:
            name: postgres-init
            namespace: myapp
          spec:
            containers:
            - name: postgres-init
              image: postgres:13
              command: ["/bin/sh", "-c"]
              args:
                - psql -h postgres -U postgres -d myapp -c "CREATE TABLE IF NOT EXISTS messages (id SERIAL PRIMARY KEY, text TEXT NOT NULL);"
              env:
                - name: PGPASSWORD
                  value: password
      register: postgres_init

    - name: Wait for PostgreSQL initialization
      kubernetes:
        api_version: v1
        kind: Pod
        name: postgres-init
        namespace: myapp
        wait: yes
        wait_condition:
          type: Complete
          status: "True"
      when: postgres_init.changed
```

## Deployment Process

1. Navigate to the project root directory.

2. Run the Ansible playbook:

```bash
ansible-playbook -i localhost, ansible/playbook.yml
```

This will:
- Ensure Docker and k3s are installed
- Build Docker images for frontend and backend
- Apply the Terraform configuration
- Initialize the PostgreSQL database

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

6. Check Ansible execution:
   ```bash
   ansible-playbook -i localhost, ansible/playbook.yml --check --diff
   ```

7. For TypeScript-specific issues:
   - Check compilation errors:
     ```bash
     cd frontend && npm run build
     cd ../backend && npm run build
     ```
   - Verify TypeScript configs:
     ```bash
     cat tsconfig.json
     ```
