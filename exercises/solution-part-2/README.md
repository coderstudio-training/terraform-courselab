# K3s Terraform and Ansible Solutions - 2024 Edition

This document provides solutions to the K3s Terraform and Ansible Exercises - 2024 Edition. These solutions build upon the original local K3s setup and incorporate the latest best practices.

## Exercise 1 Solution: Enhance Backend Deployment

1. Update Terraform configuration (`main.tf`):

```hcl
resource "kubernetes_deployment" "backend" {
  metadata {
    name      = "backend"
    namespace = kubernetes_namespace.myapp.metadata[0].name
  }
  spec {
    replicas = 3
    strategy {
      type = "RollingUpdate"
      rolling_update {
        max_surge       = "25%"
        max_unavailable = "25%"
      }
    }
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
          resources {
            limits = {
              cpu    = "0.5"
              memory = "512Mi"
            }
            requests = {
              cpu    = "250m"
              memory = "50Mi"
            }
          }
          readiness_probe {
            http_get {
              path = "/health"
              port = 3000
            }
            initial_delay_seconds = 10
            period_seconds        = 5
          }
          liveness_probe {
            http_get {
              path = "/health"
              port = 3000
            }
            initial_delay_seconds = 15
            period_seconds        = 10
          }
        }
      }
    }
  }
}
```

2. Add to Ansible playbook (`playbook.yml`):

```yaml
- name: Create HorizontalPodAutoscaler for backend
  kubernetes:
    definition:
      apiVersion: autoscaling/v2
      kind: HorizontalPodAutoscaler
      metadata:
        name: backend-hpa
        namespace: myapp
      spec:
        scaleTargetRef:
          apiVersion: apps/v1
          kind: Deployment
          name: backend
        minReplicas: 2
        maxReplicas: 10
        metrics:
        - type: Resource
          resource:
            name: cpu
            target:
              type: Utilization
              averageUtilization: 50
```

## Exercise 2 Solution: Implement Secure Ingress

1. Add to Terraform configuration (`main.tf`):

```hcl
resource "helm_release" "nginx_ingress" {
  name       = "nginx-ingress"
  repository = "https://kubernetes.github.io/ingress-nginx"
  chart      = "ingress-nginx"
  namespace  = "ingress-nginx"
  create_namespace = true
}

resource "kubernetes_ingress_v1" "myapp_ingress" {
  metadata {
    name      = "myapp-ingress"
    namespace = kubernetes_namespace.myapp.metadata[0].name
    annotations = {
      "kubernetes.io/ingress.class" = "nginx"
      "cert-manager.io/cluster-issuer" = "selfsigned-issuer"
    }
  }
  spec {
    tls {
      hosts       = ["myapp.local"]
      secret_name = "myapp-tls"
    }
    rule {
      host = "myapp.local"
      http {
        path {
          path = "/api"
          path_type = "Prefix"
          backend {
            service {
              name = kubernetes_service.backend.metadata[0].name
              port {
                number = 3000
              }
            }
          }
        }
        path {
          path = "/"
          path_type = "Prefix"
          backend {
            service {
              name = kubernetes_service.frontend.metadata[0].name
              port {
                number = 80
              }
            }
          }
        }
      }
    }
  }
}
```

2. Add to Ansible playbook (`playbook.yml`):

```yaml
- name: Generate self-signed certificate
  openssl_certificate:
    path: /tmp/myapp.crt
    privatekey_path: /tmp/myapp.key
    provider: selfsigned

- name: Create TLS secret
  kubernetes:
    definition:
      apiVersion: v1
      kind: Secret
      metadata:
        name: myapp-tls
        namespace: myapp
      type: kubernetes.io/tls
      data:
        tls.crt: "{{ lookup('file', '/tmp/myapp.crt') | b64encode }}"
        tls.key: "{{ lookup('file', '/tmp/myapp.key') | b64encode }}"

- name: Add hostfile entry
  lineinfile:
    path: /etc/hosts
    line: "127.0.0.1 myapp.local"
    state: present
```

## Exercise 3 Solution: Enhance Database Security

1. Add to Terraform configuration (`main.tf`):

```hcl
resource "kubernetes_secret" "db_credentials" {
  metadata {
    name      = "db-credentials"
    namespace = kubernetes_namespace.myapp.metadata[0].name
  }

  data = {
    username = "myapp"
    password = random_password.db_password.result
  }
}

resource "random_password" "db_password" {
  length  = 16
  special = false
}

resource "kubernetes_network_policy" "db_network_policy" {
  metadata {
    name      = "db-network-policy"
    namespace = kubernetes_namespace.myapp.metadata[0].name
  }

  spec {
    pod_selector {
      match_labels = {
        app = "postgres"
      }
    }

    ingress {
      from {
        pod_selector {
          match_labels = {
            app = "backend"
          }
        }
      }
    }

    policy_types = ["Ingress"]
  }
}
```

2. Update Ansible playbook (`playbook.yml`):

```yaml
- name: Deploy PostgreSQL with secrets
  kubernetes:
    definition:
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: postgres
        namespace: myapp
      spec:
        # ... existing spec ...
        template:
          # ... existing template ...
          spec:
            containers:
            - name: postgres
              # ... existing container config ...
              env:
              - name: POSTGRES_USER
                valueFrom:
                  secretKeyRef:
                    name: db-credentials
                    key: username
              - name: POSTGRES_PASSWORD
                valueFrom:
                  secretKeyRef:
                    name: db-credentials
                    key: password

- name: Set up database backup
  kubernetes:
    definition:
      apiVersion: batch/v1
      kind: CronJob
      metadata:
        name: db-backup
        namespace: myapp
      spec:
        schedule: "0 1 * * *"
        jobTemplate:
          spec:
            template:
              spec:
                containers:
                - name: db-backup
                  image: postgres:13
                  command:
                  - /bin/sh
                  - -c
                  - pg_dump -h postgres -U $POSTGRES_USER $POSTGRES_DB > /backup/db_backup_$(date +%Y%m%d).sql
                  env:
                  - name: POSTGRES_USER
                    valueFrom:
                      secretKeyRef:
                        name: db-credentials
                        key: username
                  - name: POSTGRES_PASSWORD
                    valueFrom:
                      secretKeyRef:
                        name: db-credentials
                        key: password
                  - name: POSTGRES_DB
                    value: myapp
                volumes:
                - name: backup
                  persistentVolumeClaim:
                    claimName: backup-pvc
```

## Exercise 4 Solution: Implement GitOps with ArgoCD and GitLab CI

1. Add ArgoCD installation to Terraform configuration (`main.tf`):

```hcl
resource "helm_release" "argocd" {
  name             = "argocd"
  repository       = "https://argoproj.github.io/argo-helm"
  chart            = "argo-cd"
  namespace        = "argocd"
  create_namespace = true
  version          = "3.35.4"  # Use the latest stable version

  set {
    name  = "server.service.type"
    value = "LoadBalancer"
  }
}

resource "kubernetes_secret" "gitlab_access" {
  metadata {
    name      = "gitlab-access"
    namespace = "argocd"
  }

  data = {
    username = var.gitlab_username
    password = var.gitlab_password
  }

  type = "Opaque"
}
```

2. Add to Ansible playbook (`playbook.yml`) to configure ArgoCD:

```yaml
- name: Wait for ArgoCD to be ready
  kubernetes:
    api_version: v1
    kind: Pod
    namespace: argocd
    label_selectors:
      - app.kubernetes.io/name=argocd-server
    wait: yes
    wait_condition:
      type: Ready

- name: Configure ArgoCD with GitLab repository
  kubernetes:
    definition:
      apiVersion: argoproj.io/v1alpha1
      kind: Application
      metadata:
        name: myapp
        namespace: argocd
      spec:
        project: default
        source:
          repoURL: 'https://gitlab.com/your-username/your-repo.git'
          targetRevision: HEAD
          path: k8s  # Assuming your K8s manifests are in a 'k8s' directory
        destination:
          server: 'https://kubernetes.default.svc'
          namespace: myapp
        syncPolicy:
          automated:
            prune: true
            selfHeal: true

- name: Configure ArgoCD to use GitLab credentials
  kubernetes:
    definition:
      apiVersion: v1
      kind: Secret
      metadata:
        name: gitlab-repo-creds
        namespace: argocd
        labels:
          argocd.argoproj.io/secret-type: repository
      stringData:
        type: git
        url: 'https://gitlab.com/your-username/your-repo.git'
        username: '{{ gitlab_username }}'
        password: '{{ gitlab_password }}'
```

3. Create a `.gitlab-ci.yml` file in your project repository:

```yaml
stages:
  - build
  - test
  - update_manifests

build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker build -t myapp:$CI_COMMIT_SHA .
    - docker push myregistry.com/myapp:$CI_COMMIT_SHA

test:
  stage: test
  image: node:20-alpine
  script:
    - npm install
    - npm test

update_manifests:
  stage: update_manifests
  image: alpine/git
  script:
    - git config user.email "ci@example.com"
    - git config user.name "GitLab CI"
    - git checkout -b update-manifests-$CI_COMMIT_SHA
    - sed -i "s|image: myapp:.*|image: myapp:$CI_COMMIT_SHA|g" k8s/deployment.yaml
    - git add k8s/deployment.yaml
    - git commit -m "Update image tag to $CI_COMMIT_SHA"
    - git push origin update-manifests-$CI_COMMIT_SHA
  only:
    - main
```

This solution does the following:

1. Uses Terraform to install ArgoCD in the K3s cluster and create a secret for GitLab credentials.
2. Uses Ansible to configure ArgoCD with the GitLab repository and set up automated sync.
3. Provides a GitLab CI configuration that:
   - Builds and tests the application
   - Updates the Kubernetes manifests in the GitLab repository when changes are made to the application code

Note: You'll need to replace placeholders like `your-username`, `your-repo`, and `myregistry.com` with your actual GitLab and container registry details.

Remember to secure your GitLab credentials and consider using GitLab CI/CD variables for sensitive information in your pipeline.

## Exercise 5 Solution: Monitoring and Logging

1. Add to Terraform configuration (`main.tf`):

```hcl
resource "helm_release" "prometheus" {
  name       = "prometheus"
  repository = "https://prometheus-community.github.io/helm-charts"
  chart      = "kube-prometheus-stack"
  namespace  = "monitoring"
  create_namespace = true
}

resource "helm_release" "loki" {
  name       = "loki"
  repository = "https://grafana.github.io/helm-charts"
  chart      = "loki-stack"
  namespace  = "monitoring"
  create_namespace = true
}
```

2. Add to Ansible playbook (`playbook.yml`):

```yaml
- name: Configure Grafana dashboards
  kubernetes:
    definition:
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: grafana-dashboards
        namespace: monitoring
      data:
        my-dashboard.json: |
          {
            "dashboard": {
              "id": null,
              "title": "My Dashboard",
              "panels": [
                {
                  "title": "Pod CPU Usage",
                  "type": "graph",
                  "datasource": "Prometheus",
                  "targets": [
                    {
                      "expr": "sum(rate(container_cpu_usage_seconds_total{namespace=\"myapp\"}[5m])) by (pod)",
                      "legendFormat": "{{pod}}"
                    }
                  ]
                }
              ]
            }
          }

- name: Configure Prometheus alerts
  kubernetes:
    definition:
      apiVersion: monitoring.coreos.com/v1
      kind: PrometheusRule
      metadata:
        name: myapp-alerts
        namespace: monitoring
      spec:
        groups:
        - name: myapp
          rules:
          - alert: HighCPUUsage
            expr: sum(rate(container_cpu_usage_seconds_total{namespace="myapp"}[5m])) by (pod) > 0.8
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: High CPU usage detected
              description: Pod {{ $labels.pod }} has high CPU usage
```

These solutions provide a comprehensive update to the original K3s setup, incorporating modern Kubernetes features, improved security practices, and advanced deployment techniques using the latest versions of Terraform and Ansible.
