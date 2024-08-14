# Local K3s Tutorial with Terraform

This tutorial will guide you through setting up a local Kubernetes environment using k3s and Terraform. We'll deploy a full-stack application consisting of a React frontend, Node.js backend, and PostgreSQL database.

## Table of Contents

1. [Prerequisites](./)
2. [Project Structure](#project-structure)
3. [Setting Up the Local Environment](#setting-up-the-local-environment)
4. [Creating the Application Components](#creating-the-application-components)
5. [Containerizing the Applications](#containerizing-the-applications)
6. [Configuring Terraform](#configuring-terraform)
7. [Deployment Process](#deployment-process)
8. [Testing the Application](#testing-the-application)
9. [Troubleshooting](#troubleshooting)

## Project Structure

Create the following directory structure for your project:

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

2. Set up kubectl to use k3s:

```bash
mkdir ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
```

## Creating the Application Components

### Frontend (React)

1. Navigate to the `frontend` directory and create a new React app:

```bash
cd frontend
npx create-react-app .
```

2. Replace the contents of `src/App.js` with:

```jsx
import React, { useState, useEffect } from "react";
import axios from "axios";

function App() {
  const [message, setMessage] = useState("");
  const [inputText, setInputText] = useState("");

  useEffect(() => {
    fetchMessage();
  }, []);

  const fetchMessage = async () => {
    try {
      const response = await axios.get("http://localhost:3000/api/message");
      setMessage(response.data.message);
    } catch (error) {
      console.error("Error fetching message:", error);
    }
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      await axios.post("http://localhost:3000/api/message", {
        text: inputText,
      });
      fetchMessage();
      setInputText("");
    } catch (error) {
      console.error("Error submitting message:", error);
    }
  };

  return (
    <div>
      <h1>K3s React App</h1>
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

### Backend (Node.js)

1. Navigate to the `backend` directory and initialize a new Node.js project:

```bash
cd ../backend
npm init -y
npm install express pg cors
```

2. Create `server.js` with the following content:

```javascript
const express = require("express");
const { Pool } = require("pg");
const cors = require("cors");

const app = express();
app.use(cors());
app.use(express.json());

const pool = new Pool({
  user: "postgres",
  host: "postgres",
  database: "myapp",
  password: "password",
  port: 5432,
});

app.get("/api/message", async (req, res) => {
  try {
    const result = await pool.query(
      "SELECT text FROM messages ORDER BY id DESC LIMIT 1",
    );
    res.json({ message: result.rows[0]?.text || "No messages yet" });
  } catch (error) {
    console.error("Error fetching message:", error);
    res.status(500).json({ error: "Internal server error" });
  }
});

app.post("/api/message", async (req, res) => {
  try {
    const { text } = req.body;
    await pool.query("INSERT INTO messages (text) VALUES ($1)", [text]);
    res.status(201).json({ message: "Message saved successfully" });
  } catch (error) {
    console.error("Error saving message:", error);
    res.status(500).json({ error: "Internal server error" });
  }
});

const port = 3000;
app.listen(port, () => {
  console.log(`Server running on port ${port}`);
});
```

## Containerizing the Applications

1. In the `frontend` directory, create a `Dockerfile`:

```dockerfile
FROM node:14 as build
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

2. In the `backend` directory, create a `Dockerfile`:

```dockerfile
FROM node:14
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

3. Build the Docker images:

```bash
cd ../frontend
docker build -t frontend:latest .

cd ../backend
docker build -t backend:latest .
```

## Configuring Terraform

1. Navigate to the `terraform` directory and create `main.tf`:

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

## Deployment Process

1. Navigate to the `terraform` directory.

2. Initialize Terraform:

```bash
terraform init
```

3. Apply the Terraform configuration:

```bash
terraform apply
```

4. After Terraform has finished applying the configuration, initialize the PostgreSQL database:

```bash
kubectl run -n myapp postgres-init --rm --tty -i --restart='Never' \
  --image postgres:13 --env="PGPASSWORD=password" \
  -- psql -h postgres -U postgres -d myapp -c "CREATE TABLE IF NOT EXISTS messages (id SERIAL PRIMARY KEY, text TEXT NOT NULL);"
```

## Testing the Application

1. Access the frontend:
   Open a web browser and navigate to `http://localhost:30080`

2. Test the interaction:
   - You should see the React frontend with a message from the server (initially "No messages yet").
   - Enter a new message in the input field and click "Submit".
   - The new message should be displayed after submission.

## Troubleshooting

If you encounter issues, try these troubleshooting steps:

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
   terraform show
   ```

6. If you need to make changes, you can modify the Terraform configuration and run:
   ```bash
   terraform plan
   terraform apply
   ```

This revised tutorial provides a step-by-step guide to setting up a local Kubernetes environment using k3s and Terraform, without the use of Ansible. The manual steps replace the automation that Ansible provided in the original version.
