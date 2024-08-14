# K3s Terraform and Ansible Exercises

These exercises are designed to build upon the local K3s setup using Terraform and Ansible


## Exercise 1: Enhance Backend Deployment

Modify the backend deployment to use the latest Kubernetes features for improved resilience and scalability.

1. Update the Terraform configuration to use a Deployment with a RollingUpdate strategy.
2. Implement readiness and liveness probes in the backend container specification.
3. Add resource requests and limits to the backend container.
4. Use the Ansible kubernetes module to create a HorizontalPodAutoscaler for the backend.

## Exercise 2: Implement Secure Ingress

Add an Ingress resource to route external traffic to your services securely.

1. Use Terraform to deploy an Ingress controller (e.g., Nginx Ingress Controller).
2. Create an Ingress resource in Terraform to route traffic to your frontend and backend services.
3. Implement TLS termination using a self-signed certificate (use Ansible to generate the certificate).
4. Update the Ansible playbook to configure any necessary DNS settings for local testing.

## Exercise 3: Enhance Database Security

Improve the security of the PostgreSQL database deployment.

1. Use Terraform to create a Kubernetes Secret for the database credentials.
2. Modify the Ansible playbook to use this Secret in the PostgreSQL deployment.
3. Implement a NetworkPolicy using Terraform to restrict database access to only the backend service.
4. Use Ansible to configure regular database backups to a persistent volume.

## Exercise 4: Implement GitOps with ArgoCD and GitLab CI

Set up a GitOps workflow using ArgoCD for continuous deployment and GitLab CI for continuous integration.

1. Use Terraform to install ArgoCD in the K3s cluster.
2. Create a GitLab repository to store your Kubernetes manifests.
3. Use Ansible to configure ArgoCD to sync your GitLab repository with the cluster.
4. Implement a GitLab CI pipeline that:
   a. Builds and tests your application code
   b. Updates the Kubernetes manifests in the GitLab repository when changes are made to your application code
5. Configure ArgoCD to automatically sync changes from the GitLab repository to the cluster.


## Exercise 5: Monitoring and Logging

Set up a basic monitoring and logging stack.

1. Use Terraform to deploy Prometheus and Grafana for monitoring.
2. Implement Loki and Promtail for log aggregation using Terraform.
3. Create an Ansible playbook to configure custom dashboards in Grafana.
4. Implement alerts in Prometheus and use Ansible to configure alert routing.

Remember to test your changes thoroughly and ensure that both Terraform and Ansible remain idempotent throughout these exercises.

---
Feeling stuck? Click [here](/exercises/solution-part-2/README.md) to see the solution
