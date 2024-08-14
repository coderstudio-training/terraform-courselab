# K3s and Terraform Exercise Solutions

This document provides solutions to the exercises from the K3s and Terraform Exercises. Remember, these are example solutions, and there may be multiple valid ways to solve each exercise.

## Exercise 1 Solution: Modify the Frontend

1. Update `frontend/src/App.js`:

```jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

function App() {
  // ... existing code ...

  const clearAllMessages = async () => {
    try {
      await axios.delete('http://localhost:3000/api/messages');
      setMessage('No messages');
    } catch (error) {
      console.error('Error clearing messages:', error);
    }
  };

  return (
    <div>
      {/* ... existing JSX ... */}
      <button onClick={clearAllMessages}>Clear All Messages</button>
    </div>
  );
}

export default App;
```

2. Update `backend/server.js`:

```javascript
// ... existing code ...

app.delete('/api/messages', async (req, res) => {
  try {
    await pool.query('TRUNCATE TABLE messages');
    res.status(200).json({ message: 'All messages cleared' });
  } catch (error) {
    console.error('Error clearing messages:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// ... rest of the code ...
```

3. No changes to Terraform configuration are necessary for this exercise.

## Exercise 2 Solution: Add Environment Variables

1. Modify `backend/server.js`:

```javascript
const pool = new Pool({
  user: process.env.DB_USER || 'postgres',
  host: process.env.DB_HOST || 'postgres',
  database: process.env.DB_NAME || 'myapp',
  password: process.env.DB_PASSWORD || 'password',
  port: parseInt(process.env.DB_PORT || '5432'),
});
```

2. Update Terraform configuration in `main.tf`:

```hcl
resource "kubernetes_deployment" "backend" {
  # ... existing configuration ...

  spec {
    # ... existing spec ...

    template {
      # ... existing template ...

      spec {
        container {
          # ... existing container config ...

          env {
            name  = "DB_USER"
            value = "postgres"
          }
          env {
            name  = "DB_HOST"
            value = "postgres"
          }
          env {
            name  = "DB_NAME"
            value = "myapp"
          }
          env {
            name  = "DB_PASSWORD"
            value = "password"  # In a real scenario, use a secret here
          }
          env {
            name  = "DB_PORT"
            value = "5432"
          }
        }
      }
    }
  }
}
```

## Exercise 3 Solution: Implement Health Checks

1. Add a health endpoint to `backend/server.js`:

```javascript
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'healthy' });
});
```

2. Update Terraform configuration in `main.tf`:

```hcl
resource "kubernetes_deployment" "backend" {
  # ... existing configuration ...

  spec {
    # ... existing spec ...

    template {
      # ... existing template ...

      spec {
        container {
          # ... existing container config ...

          liveness_probe {
            http_get {
              path = "/health"
              port = 3000
            }
            initial_delay_seconds = 10
            period_seconds        = 5
          }

          readiness_probe {
            http_get {
              path = "/health"
              port = 3000
            }
            initial_delay_seconds = 5
            period_seconds        = 5
          }
        }
      }
    }
  }
}
```

## Exercise 4 Solution: Scale the Backend

1. Update Terraform configuration in `main.tf`:

```hcl
variable "backend_replicas" {
  description = "Number of backend replicas"
  type        = number
  default     = 1
}

resource "kubernetes_deployment" "backend" {
  # ... existing configuration ...

  spec {
    replicas = var.backend_replicas

    # ... rest of the spec ...
  }
}
```

2. To scale, you can now use:

```bash
terraform apply -var="backend_replicas=3"
```

## Exercise 5 Solution: Add Redis Caching

1. Add Redis deployment and service to `main.tf`:

```hcl
resource "kubernetes_deployment" "redis" {
  metadata {
    name      = "redis"
    namespace = kubernetes_namespace.myapp.metadata[0].name
  }
  spec {
    replicas = 1
    selector {
      match_labels = {
        app = "redis"
      }
    }
    template {
      metadata {
        labels = {
          app = "redis"
        }
      }
      spec {
        container {
          image = "redis:6"
          name  = "redis"
          port {
            container_port = 6379
          }
        }
      }
    }
  }
}

resource "kubernetes_service" "redis" {
  metadata {
    name      = "redis"
    namespace = kubernetes_namespace.myapp.metadata[0].name
  }
  spec {
    selector = {
      app = kubernetes_deployment.redis.spec[0].template[0].metadata[0].labels.app
    }
    port {
      port        = 6379
      target_port = 6379
    }
  }
}
```

2. Modify `backend/server.js` (you'll need to install the `redis` npm package):

```javascript
const redis = require('redis');
const { promisify } = require('util');

const redisClient = redis.createClient({
  host: process.env.REDIS_HOST || 'redis',
  port: process.env.REDIS_PORT || 6379,
});

const getAsync = promisify(redisClient.get).bind(redisClient);
const setAsync = promisify(redisClient.set).bind(redisClient);

app.get('/api/message', async (req, res) => {
  try {
    const cachedMessage = await getAsync('latest_message');
    if (cachedMessage) {
      return res.json({ message: cachedMessage });
    }

    const result = await pool.query('SELECT text FROM messages ORDER BY id DESC LIMIT 1');
    const message = result.rows[0]?.text || 'No messages yet';

    await setAsync('latest_message', message, 'EX', 60);  // Cache for 60 seconds
    res.json({ message });
  } catch (error) {
    console.error('Error fetching message:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Update post method to invalidate cache when new message is added
app.post('/api/message', async (req, res) => {
  try {
    const { text } = req.body;
    await pool.query('INSERT INTO messages (text) VALUES ($1)', [text]);
    await redisClient.del('latest_message');  // Invalidate cache
    res.status(201).json({ message: 'Message saved successfully' });
  } catch (error) {
    console.error('Error saving message:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});
```

3. Update Terraform configuration for backend in `main.tf`:

```hcl
resource "kubernetes_deployment" "backend" {
  # ... existing configuration ...

  spec {
    # ... existing spec ...

    template {
      # ... existing template ...

      spec {
        container {
          # ... existing container config ...

          env {
            name  = "REDIS_HOST"
            value = "redis"
          }
          env {
            name  = "REDIS_PORT"
            value = "6379"
          }
        }
      }
    }
  }
}
```

## Bonus Exercise Solution: Implement Ingress

1. Install Ingress controller (this step is manual and depends on your K3s setup).

2. Add Ingress resource to `main.tf`:

```hcl
resource "kubernetes_ingress" "myapp_ingress" {
  metadata {
    name      = "myapp-ingress"
    namespace = kubernetes_namespace.myapp.metadata[0].name
    annotations = {
      "kubernetes.io/ingress.class" = "nginx"
    }
  }

  spec {
    rule {
      http {
        path {
          path = "/api"
          backend {
            service_name = kubernetes_service.backend.metadata[0].name
            service_port = 3000
          }
        }

        path {
          path = "/"
          backend {
            service_name = kubernetes_service.frontend.metadata[0].name
            service_port = 80
          }
        }
      }
    }
  }
}
```

3. Update frontend to use relative path for API calls:

In `frontend/src/App.js`, change all API calls from `http://localhost:3000/api/...` to `/api/...`.

Remember to rebuild your Docker images and reapply your Terraform configuration after making these changes.
