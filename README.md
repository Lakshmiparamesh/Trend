# Trendify — Production Deployment

A React e-commerce application deployed to AWS EKS using Terraform, Jenkins CI/CD, DockerHub, and Prometheus/Grafana monitoring.

## Architecture Overview

```
GitHub Repo → Jenkins (GitHub Webhook) → DockerHub → EKS (Kubernetes)
                                                          ↓
                                                LoadBalancer (port 80)
                                                          ↓
                                                Nginx container (port 3000)

Monitoring: Prometheus + Grafana (in-cluster)
Infrastructure: Terraform (VPC, IAM, EC2, EKS)
```

## Project Structure

```
Trend/
├── dist/                              # Production-ready React build
│   ├── index.html
│   └── assets/
├── k8s/
│   ├── deployment.yaml                # Kubernetes Deployment (2 replicas)
│   └── service.yaml                   # Kubernetes Service (LoadBalancer)
├── terraform/
│   ├── provider.tf                    # AWS provider configuration
│   ├── variables.tf                   # Input variables
│   ├── vpc.tf                         # VPC, subnets, NAT gateway
│   ├── iam.tf                         # IAM roles (EKS, nodes, Jenkins)
│   ├── security-groups.tf             # Security groups
│   ├── jenkins-ec2.tf                 # Jenkins EC2 instance with user data
│   ├── eks.tf                         # EKS cluster and node group
│   └── outputs.tf                     # Output values
├── monitoring/
│   ├── prometheus-config.yaml         # Prometheus deployment + config
│   └── grafana.yaml                   # Grafana deployment + datasource
├── Dockerfile                         # Nginx container serving React app
├── nginx.conf                         # Nginx config (port 3000, SPA routing)
├── Jenkinsfile                        # Declarative pipeline
├── .dockerignore
├── .gitignore
└── README.md
```

## Prerequisites

- AWS CLI v2 configured with credentials
- Terraform installed (v1.0+)
- Docker installed locally
- kubectl installed
- Git

## Setup Instructions

### Step 1: Create SSH Key Pair for Jenkins EC2

```bash
aws ec2 create-key-pair \
  --key-name trend-jenkins-key \
  --query 'KeyMaterial' \
  --output text \
  --region ap-south-1 > trend-jenkins-key.pem

chmod 400 trend-jenkins-key.pem
```

### Step 2: Provision Infrastructure with Terraform

```bash
cd terraform

terraform init
terraform plan
terraform apply -auto-approve
```

This creates:
- VPC with public/private subnets, NAT gateway
- IAM roles for EKS cluster, node group, and Jenkins
- EC2 instance with Jenkins, Docker, kubectl, AWS CLI pre-installed
- EKS cluster with managed node group (2x t3.medium)

Save the outputs (Jenkins IP, EKS endpoint).

### Step 3: Create DockerHub Repository

1. Log in to [DockerHub](https://hub.docker.com)
2. Create repository: `lakshmiparamesh/trend-app`

### Step 4: Build & Test Docker Image Locally

```bash
docker build -t trend-app .
docker run -p 3000:3000 trend-app
# Visit http://localhost:3000
```

### Step 5: Push Docker Image to DockerHub

```bash
docker login -u lakshmiparamesh
docker tag trend-app:latest lakshmiparamesh/trend-app:latest
docker push lakshmiparamesh/trend-app:latest
```

### Step 6: Configure kubectl for EKS

```bash
aws eks update-kubeconfig --name trend-cluster --region ap-south-1
kubectl get nodes
```

### Step 7: Deploy to EKS

```bash
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml

kubectl get deployments
kubectl get pods
kubectl get svc trend-app-service
```

### Step 8: Setup Jenkins

1. SSH into Jenkins EC2:
   ```bash
   ssh -i trend-jenkins-key.pem ubuntu@<JENKINS_PUBLIC_IP>
   ```

2. Get Jenkins initial admin password:
   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```

3. Open Jenkins UI: `http://<JENKINS_PUBLIC_IP>:8080`

4. Install suggested plugins + additional:
   - Docker Pipeline
   - Pipeline
   - Git
   - GitHub Integration
   - Kubernetes CLI

5. Add DockerHub credentials:
   - Go to Manage Jenkins → Credentials
   - Add credentials (Username/Password)
   - ID: `dockerhub-credentials`
   - Username: `lakshmiparamesh`
   - Password: DockerHub password/token

6. Configure AWS credentials on Jenkins EC2:
   ```bash
   sudo -u jenkins aws configure
   # Enter AWS Access Key, Secret Key, region: ap-south-1
   ```

7. Grant Jenkins EKS access:
   ```bash
   # On your local machine
   kubectl edit configmap aws-auth -n kube-system
   ```
   Add under `mapRoles`:
   ```yaml
   - rolearn: arn:aws:iam::009075573616:role/trend-jenkins-role
     username: jenkins
     groups:
       - system:masters
   ```

### Step 9: Create Jenkins Pipeline

1. New Item → Pipeline
2. Name: `trend-app-pipeline`
3. Pipeline Definition: Pipeline script from SCM
4. SCM: Git
5. Repository URL: `https://github.com/Lakshmiparamesh/Trend.git`
6. Branch: `*/main`
7. Script Path: `Jenkinsfile`

### Step 10: Configure GitHub Webhook

1. Go to GitHub repo → Settings → Webhooks → Add webhook
2. Payload URL: `http://<JENKINS_PUBLIC_IP>:8080/github-webhook/`
3. Content type: `application/json`
4. Events: Just the push event
5. Active: ✅

### Step 11: Deploy Monitoring (Prometheus + Grafana)

```bash
kubectl apply -f monitoring/prometheus-config.yaml
kubectl apply -f monitoring/grafana.yaml

# Get monitoring URLs
kubectl get svc -n monitoring
```

- **Prometheus**: `http://<PROMETHEUS_LB>:9090`
- **Grafana**: `http://<GRAFANA_LB>:3000` (admin / admin123)

Add Kubernetes dashboard in Grafana:
1. Go to Dashboards → Import
2. Dashboard ID: `6417` (Kubernetes Cluster Monitoring)
3. Data source: Prometheus

## CI/CD Pipeline Flow

```
1. Developer pushes code to GitHub (main branch)
2. GitHub webhook triggers Jenkins pipeline
3. Jenkins:
   a. Checks out source code
   b. Builds Docker image
   c. Pushes image to DockerHub
   d. Updates kubeconfig for EKS
   e. Applies Kubernetes manifests
   f. Restarts deployment for latest image
   g. Waits for rollout to complete
4. Application is live on the LoadBalancer URL
```

## Monitoring

- **Prometheus** — Collects metrics from Kubernetes nodes and pods
- **Grafana** — Visualizes metrics with dashboards
  - Default dashboard: Kubernetes Cluster Monitoring (ID 6417)
  - Data source: Prometheus (auto-provisioned)

```bash
# View Prometheus targets
http://<PROMETHEUS_LB>:9090/targets

# View Grafana dashboards
http://<GRAFANA_LB>:3000

# View application logs
kubectl logs -l app=trend-app --tail=100
```

## Deployment Evidence

### Live URLs

| Service       | URL                                                                                          |
|---------------|----------------------------------------------------------------------------------------------|
| Application   | http://a2a3182ced88b4f229dba57e9dd89600-1160890347.ap-south-1.elb.amazonaws.com              |
| Jenkins       | http://43.205.254.170:8080                                                                   |
| Prometheus    | http://a7c8031aa29fd4885b7163ef3519a564-1049899950.ap-south-1.elb.amazonaws.com:9090         |
| Grafana       | http://a5ddf1824bd99469e9c649395791ace9-2055109558.ap-south-1.elb.amazonaws.com:3000         |

### EKS Cluster Status

```
Nodes:
  ip-10-0-10-67   Ready   v1.31.14-eks-8418f54   (Amazon Linux 2023, t3.medium)
  ip-10-0-11-123  Ready   v1.31.14-eks-8418f54   (Amazon Linux 2023, t3.medium)

Pods:
  trend-app-554555b6b6-wbmcx   1/1 Running   (node: ip-10-0-11-123)
  trend-app-554555b6b6-zgdz2   1/1 Running   (node: ip-10-0-10-67)

Services:
  trend-app-service   LoadBalancer   80:30851/TCP
  prometheus-service  LoadBalancer   9090:32760/TCP
  grafana-service     LoadBalancer   3000:31149/TCP
```

### Jenkins Pipeline

- Job: `trend-app-pipeline`
- Build #1: SUCCESS
- Pipeline: Checkout → Docker Build → Push to DockerHub → Update Kubeconfig → Deploy to EKS
- Trigger: GitHub webhook (push events on main branch)

### Terraform Resources (29 total)

- VPC: `vpc-0cb017a638149983a` (10.0.0.0/16)
- 2 public subnets + 2 private subnets
- NAT Gateway, Internet Gateway
- EKS Cluster: `trend-cluster`
- EKS Node Group: 2x t3.medium
- Jenkins EC2: `i-0aa195d125a2fbb50` (t3.medium)
- IAM Roles: eks-cluster, eks-nodes, jenkins

## Key Details

| Item                | Value                                            |
|---------------------|--------------------------------------------------|
| Region              | ap-south-1 (Mumbai)                              |
| DockerHub Image     | lakshmiparamesh/trend-app                        |
| EKS Cluster         | trend-cluster (v1.31)                            |
| App Port            | 3000 (container) → 80 (LoadBalancer)             |
| Replicas            | 2                                                |
| Node Type           | t3.medium                                        |
| Jenkins Instance    | t3.medium (Ubuntu 22.04)                         |
| Infrastructure      | Terraform                                        |
| CI/CD               | Jenkins (declarative pipeline)                   |
| Image Registry      | DockerHub                                        |
| Monitoring          | Prometheus + Grafana                             |
