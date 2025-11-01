# Django Kubernetes Deployment Guide

This guide will help you deploy the Django application to a local Kubernetes cluster using Helm charts and ArgoCD.

## Prerequisites

Install the following tools on your macOS/Linux machine:

- **Docker Desktop**: Container runtime
- **kubectl**: Kubernetes command-line tool
- **Kind**: Local Kubernetes cluster
- **Helm**: Kubernetes package manager

### Installation Commands (macOS)

```bash
# Install Homebrew (if not already installed)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install tools and check version
brew install kubectl 
kubectl version --client
brew install kind
kind --version
brew install helm
helm version
###PLUS
brew install argocd

# Create a project directory
mkdir -p ~/django-k8s-project
#Create kind yaml file under project directory
cat > kind-config.yaml 
This Kind YAML file defines a customized Kubernetes cluster setup in Docker, specifying node roles, labels, and port mappings for testing or development.

# Install Docker Desktop from https://www.docker.com/products/docker-desktop
```

## Quick Start

### 1. Create Kubernetes Cluster

Create a multi-node Kind cluster with port mappings:

```bash
# Navigate to project directory
cd django-k8s-project

# Create cluster using the provided config
kind create cluster --name django-cluster --config kind-config.yaml
```

**Why?** This creates a 3-node Kubernetes cluster (1 control-plane, 2 workers) that mimics a production environment.
# Check commands 
kubectl get nodes
kubectl cluster-info
kubectl get nodes -o wide
docker ps

### 2. Install Ingress Controller

```bash
# Install NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Wait for it to be ready (Optional)
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s

kubectl get pods -A
```
**Why?** Ingress allows external HTTP/HTTPS access to services inside the cluster.

### 3. Build Docker Image
Update docker file with correct path

```bash
cd django-app

# Build the Django application image
docker build -t django-app:v1 .

#Check image(Optional)
docker images

# Load image into Kind cluster
kind load docker-image django-app:v1 --name django-cluster
```

**Why?** Kind clusters don't have access to local Docker images by default. We need to explicitly load them.

### 4. Create Helm Chart Structure
cd django-k8s-project

# Create Helm chart directory structure and Helm chart files
mkdir -p helm-chart/templates

```bash
cat > helm-chart/Chart.yaml
cat > helm-chart/values.yaml
cat > helm-chart/templates/secret.yaml
cat > helm-chart/templates/configmap.yaml
cat > helm-chart/templates/migration-job.yaml
cat > helm-chart/templates/deployment.yaml
cat > helm-chart/templates/service.yaml
cat > helm-chart/templates/ingress.yaml
```

### 5. Deploy PostgreSQL Database

```bash
# Add Bitnami Helm repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm dependency update

# Install PostgreSQL
helm install django-postgres bitnami/postgresql \
  --set auth.username=django \
  --set auth.password=django123 \
  --set auth.database=djangodb \
  --set primary.persistence.size=1Gi

kubectl get pods -w (optional)
```

**Why?** The Django application requires PostgreSQL. We use Helm to deploy it with persistent storage.

### 6. Deploy Django Application

```bash
cd ../helm-chart

# Install Django application
helm install django-app .
```

**Why?** This deploys:
- ConfigMap (environment variables)
- Secret (sensitive data like SECRET_KEY)
- Migration Job (runs `python manage.py migrate`)
- Deployment (3 Django pods for high availability)
- Service (internal load balancer)
- Ingress (external access routing)

### 7. Wait for Pods to be Ready

```bash
# Watch pod status
kubectl get pods -w
```

Wait until all pods show `Running` status:
- `django-app-xxxxx` (3 pods)
- `django-postgres-postgresql-0` (1 pod)
- `django-migrate-xxxxx` (should be `Completed`)

Press `Ctrl+C` to stop watching.

### 8. Add DNS Entry

Edit your hosts file to resolve `django.local`:

```bash
sudo nano /etc/hosts
```
Add this line at the end:
```
127.0.0.1 django.local
```
kubectl get ingress

**Why?** This tells your computer that `django.local` should point to localhost.

### 9. Expose Ingress (Local Workaround)

Due to Kind limitations, we need to manually forward the Ingress port:

```bash
# In a new terminal, run this and keep it open
sudo kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 80:80
```

**Why?** Kind doesn't support LoadBalancer services natively. In production (AWS/GCP/Azure), this step wouldn't be needed.

### 10. Access the Application

Open your browser and navigate to:

- **API**: http://django.local
- **Admin Panel**: http://django.local/admin/

You should see the Django application running!

### 11. Create Admin User

```bash
# Get a Django pod name
kubectl get pods | grep django-app

# Execute into the pod (replace POD_NAME with actual name)
kubectl exec -it <POD_NAME> -- bash

# Inside the pod, create superuser
python manage.py createsuperuser

kubectl exec -it django-app-8464c96c-ckt7r -- bash -c "python manage.py createsuperuser"

# Follow prompts:
# Username: admin
# Email: admin@example.com
# Password: admin123
kubectl exec -it django-app-8464c96c-ckt7r -- bash -c "python manage.py shell -c 'from django.contrib.auth import get_user_model; print(get_user_model().objects.filter(is_superuser=True))'"
# Exit the pod
exit
```

Now you can log in to http://django.local/admin/ with these credentials.

## Architecture Overview

```
┌─────────────────┐
│    Browser      │
└────────┬────────┘
         │ http://django.local
         ▼
┌─────────────────┐
│  Ingress (NGINX)│ ← Routes traffic based on hostname
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Django Service  │ ← Load balances across pods
└────────┬────────┘
         │
    ┌────┴────┬────────┐
    ▼         ▼        ▼
┌────────┐ ┌────────┐ ┌────────┐
│Django-1│ │Django-2│ │Django-3│ ← 3 replicas for HA
└────┬───┘ └────┬───┘ └────┬───┘
     │          │          │
     └──────────┴──────────┘
                │
                ▼
         ┌─────────────┐
         │ PostgreSQL  │ ← Stateful database
         └─────────────┘
```

## Key Components Explained

### Helm Chart Structure

```
helm-chart/
├── Chart.yaml              # Chart metadata
├── values.yaml             # Configuration values
└── templates/
    ├── configmap.yaml      # Environment variables
    ├── secret.yaml         # Sensitive data (SECRET_KEY, passwords)
    ├── migration-job.yaml  # Database migration (runs before deploy)
    ├── deployment.yaml     # Django pods (3 replicas)
    ├── service.yaml        # Internal load balancer
    └── ingress.yaml        # External HTTP routing
```

### Environment Variables

The application requires these environment variables (managed by ConfigMap/Secret):

- `SECRET_KEY`: Django secret key for cryptographic signing
- `DEBUG`: Debug mode (false in production)
- `DATABASE_NAME`: PostgreSQL database name
- `DATABASE_USER`: PostgreSQL username
- `DATABASE_PASSWORD`: PostgreSQL password
- `DATABASE_HOST`: PostgreSQL service DNS name
- `DATABASE_PORT`: PostgreSQL port (5432)

### Migration Job

The migration job runs `python manage.py migrate` before the application starts. This ensures the database schema is up-to-date. It uses Kubernetes Job with an init container that waits for PostgreSQL to be ready.

## Troubleshooting

### Check Pod Status

```bash
# List all pods
kubectl get pods

# Describe a pod for details
kubectl describe pod <POD_NAME>

# View pod logs
kubectl logs <POD_NAME>

# View logs for all Django pods
kubectl logs -l app=django --tail=50
```

### Check Database Connection

```bash
# Connect to PostgreSQL
kubectl exec -it django-postgres-postgresql-0 -- psql -U django -d djangodb

# Inside psql, list tables
\dt

# Exit
\q
```

### Check Ingress

```bash
# Get ingress details
kubectl get ingress
kubectl describe ingress django-ingress

# Check Ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller
```

### Common Issues

**"Empty reply from server"**
- Make sure port-forward is running: `sudo kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 80:80`
- Check if `/etc/hosts` contains `127.0.0.1 django.local`

**Pods stuck in "Pending"**
- Check events: `kubectl describe pod <POD_NAME>`
- Might be resource constraints or image pull issues

**Migration job failed**
- Check logs: `kubectl logs job/django-migrate`
- Ensure PostgreSQL is running: `kubectl get pods | grep postgres`

## Upgrading the Application

When you make changes to the Django code:

```bash
# 1. Rebuild Docker image with new tag
cd django-app
docker build -t django-app:v2 .

# 2. Load new image into Kind
kind load docker-image django-app:v2 --name django-cluster

# 3. Update values.yaml
cd ../helm-chart
# Edit values.yaml and change: image: django-app:v2

# 4. Upgrade Helm release
helm upgrade django-app .

# 5. Watch rollout
kubectl rollout status deployment/django-app
```

## ArgoCD Setup (GitOps)

### Install ArgoCD

```bash
# Create namespace
kubectl get ns (optional check existing namespaces)
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods
kubectl wait --namespace argocd \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/name=argocd-server \
  --timeout=180s
```
### Build and push Django Docker image
```bash
docker build -t yagmurulas/django-app:v1 .
docker login
docker push yagmurulas/django-app:v1

# Note: Update values.yml with new repository - image name and push your code to repo

### Access ArgoCD UI

```bash
# Port forward ArgoCD server
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open browser: https://localhost:8081

**Get admin password:**
```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Login with username: `admin` and the password from above.

### Create ArgoCD Application
In CLI:
1.  Create argocd manifest yaml file 
nano argocd-django-app.yaml 
2.  Deploy
kubectl apply -f argocd-django-app.yaml

In ArgoCD UI:
1. Click **"+ NEW APP"**
2. Fill in:
   - Application Name: `django-app`
   - Project: `default`
   - Sync Policy: `Automatic` (or Manual for testing)
   - Repository URL: Your Git repository URL
   - Revision: `HEAD` or `main`
   - Path: `helm-chart`
   - Cluster URL: `https://kubernetes.default.svc`
   - Namespace: `default`
3. Click **CREATE**

ArgoCD will now monitor your Git repository and automatically deploy changes!

4. For SYNC
argocd app sync django-app

5. Create ingress for argocd
file - argocd-ingress.yaml

add domain to /etc/hosts 
127.0.0.1   argocd.localdev.me

domain - argocd.localdev.me

6. Port forwording for both argocd and django application
sudo kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 80:80 443:443

## Cleanup

To remove everything:

```bash
# Uninstall Helm releases
helm uninstall django-app
helm uninstall django-postgres

# Delete Kind cluster
kind delete cluster --name django-cluster
```

## Production Considerations

This local setup has limitations. For production deployment:

### 1. Use Managed Database
- **AWS RDS**, **Google Cloud SQL**, or **Azure Database for PostgreSQL**
- Automated backups and point-in-time recovery
- Multi-AZ deployment for high availability
- Automated failover and read replicas

### 2. Use Cloud Load Balancer
- Ingress will get a real external IP automatically
- No need for port-forwarding
- Supports SSL/TLS termination

### 3. Implement Secrets Management
- **HashiCorp Vault**, **AWS Secrets Manager**, or **Azure Key Vault**
- Never commit secrets to Git
- Rotate secrets regularly

### 4. Add SSL/TLS
- Install **cert-manager** for automatic SSL certificates
- Use **Let's Encrypt** for free certificates
- Configure Ingress with TLS

### 5. Configure Monitoring
- **Prometheus** for metrics collection
- **Grafana** for dashboards
- **AlertManager** for alerts
- Monitor pod health, database performance, request latency

### 6. Implement Logging
- **ELK Stack** (Elasticsearch, Logstash, Kibana)
- Or **Loki** + **Grafana**
- Centralized log aggregation from all pods

### 7. Set Up CI/CD
- **GitHub Actions**, **GitLab CI**, or **Jenkins**
- Automated testing on pull requests
- Automated image building and pushing to registry
- ArgoCD for continuous deployment

### 8. Configure Autoscaling
- **Horizontal Pod Autoscaler (HPA)** based on CPU/memory
- **Vertical Pod Autoscaler (VPA)** for right-sizing
- **Cluster Autoscaler** for node scaling

### 9. Implement Security Best Practices
- **Network Policies** to restrict pod-to-pod communication
- **RBAC** (Role-Based Access Control) for user permissions
- **Pod Security Standards** to prevent privileged containers
- Regular security scanning of container images
- Use **OPA** (Open Policy Agent) for policy enforcement

### 10. Database Backup Strategy
- Automated daily backups
- Backup retention policy (e.g., 30 days)
- Test restore procedures regularly
- Store backups in different region/zone

### 11. Multi-Environment Setup
- Separate clusters/namespaces for dev, staging, production
- Environment-specific values files
- Progressive rollout strategy (canary/blue-green deployments)

### 12. Resource Management
- Set resource requests and limits for all pods
- Use **ResourceQuotas** to prevent resource exhaustion
- Monitor resource usage and optimize

## Useful Commands Reference

```bash
# Cluster management
kind get clusters
kind delete cluster --name django-cluster
kubectl cluster-info
kubectl get nodes

# Pod management
kubectl get pods
kubectl get pods -A  # All namespaces
kubectl describe pod <POD_NAME>
kubectl logs <POD_NAME>
kubectl logs -f <POD_NAME>  # Follow logs
kubectl exec -it <POD_NAME> -- bash

# Service and Ingress
kubectl get svc
kubectl get ingress
kubectl describe ingress <INGRESS_NAME>

# Helm
helm list
helm status <RELEASE_NAME>
helm upgrade <RELEASE_NAME> .
helm uninstall <RELEASE_NAME>
helm rollback <RELEASE_NAME> <REVISION>

# Debug
kubectl get events --sort-by='.lastTimestamp'
kubectl top pods  # Resource usage
kubectl top nodes
```