# Trendify — Execution Output Document

## Project: TrendStore (Final Project - 2)
- **Tags**: AWS, Docker, DevOps
- **GitHub**: https://github.com/Lakshmiparamesh/Trend
- **Region**: ap-south-1 (Mumbai)

---

## 1. Terraform Infrastructure Provisioning

### Command: `terraform init`
```
Initializing the backend...
Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.x.x...
Terraform has been successfully initialized!
```

### Command: `terraform plan -out=tfplan`
```
Plan: 29 to add, 0 to change, 0 to destroy.
```

### Command: `terraform apply tfplan`
```
aws_iam_role.eks_nodes: Creating...
aws_iam_role.eks_cluster: Creating...
aws_iam_role.jenkins: Creating...
aws_eip.nat: Creating...
aws_vpc.main: Creating...
aws_eip.nat: Creation complete after 1s [id=eipalloc-0888e1e515c94b285]
aws_iam_role.eks_cluster: Creation complete after 2s [id=trend-eks-cluster-role]
aws_iam_role.jenkins: Creation complete after 2s [id=trend-jenkins-role]
aws_iam_role.eks_nodes: Creation complete after 2s [id=trend-eks-node-role]
aws_iam_instance_profile.jenkins: Creation complete after 7s [id=trend-jenkins-profile]
aws_vpc.main: Creation complete after 12s [id=vpc-0cb017a638149983a]
aws_internet_gateway.main: Creation complete after 0s [id=igw-01bd8a3f8a3905471]
aws_subnet.public[0]: Creation complete after 11s [id=subnet-0e4d6839a7e838d44]
aws_subnet.public[1]: Creation complete after 11s [id=subnet-04cfd62960911aad0]
aws_subnet.private[0]: Creation complete after 1s [id=subnet-0b61b80bddaad61c1]
aws_subnet.private[1]: Creation complete after 1s [id=subnet-02480fc15f0964258]
aws_security_group.jenkins: Creation complete after 2s [id=sg-0becca33e437e2d5d]
aws_security_group.eks_cluster: Creation complete after 2s [id=sg-0772c89e8857177d7]
aws_nat_gateway.main: Creation complete after 1m23s [id=nat-07b0cd35577d326a1]
aws_instance.jenkins: Creation complete after 12s [id=i-0aa195d125a2fbb50]
aws_eks_cluster.main: Creation complete after 8m22s [id=trend-cluster]
aws_eks_node_group.main: Creation complete after 1m48s [id=trend-cluster:trend-nodes]

Apply complete! Resources: 29 added, 0 changed, 0 destroyed.

Outputs:

eks_cluster_endpoint = "https://081F99A3DD3DB6A4112A168DA1C09E02.gr7.ap-south-1.eks.amazonaws.com"
eks_cluster_name = "trend-cluster"
eks_cluster_security_group_id = "sg-0772c89e8857177d7"
jenkins_public_ip = "43.205.254.170"
jenkins_url = "http://43.205.254.170:8080"
vpc_id = "vpc-0cb017a638149983a"
```

**29 Resources Created:**
- VPC (vpc-0cb017a638149983a) with CIDR 10.0.0.0/16
- 2 Public Subnets + 2 Private Subnets
- Internet Gateway (igw-01bd8a3f8a3905471)
- NAT Gateway (nat-07b0cd35577d326a1)
- Elastic IP (eipalloc-0888e1e515c94b285)
- Route Tables (public + private)
- Security Groups (Jenkins + EKS)
- IAM Roles (EKS Cluster, EKS Nodes, Jenkins) with policies
- Jenkins EC2 Instance (i-0aa195d125a2fbb50, t3.medium)
- EKS Cluster (trend-cluster, v1.31)
- EKS Node Group (trend-nodes, 2x t3.medium)

---

## 2. Docker Build and Push to DockerHub

### Command: `docker build -t lakshmiparamesh/trend-app .`
```
[+] Building 8.2s (9/9) FINISHED
 => [internal] load build definition from Dockerfile
 => [internal] load .dockerignore
 => [1/3] FROM nginx:1.25-alpine
 => [2/3] RUN rm -rf /usr/share/nginx/html/*
 => [3/3] COPY dist/ /usr/share/nginx/html/
 => [4/4] COPY nginx.conf /etc/nginx/conf.d/default.conf
 => exporting to image
 => => naming to docker.io/lakshmiparamesh/trend-app:latest
```

### Command: `docker push lakshmiparamesh/trend-app:latest`
```
The push refers to repository [docker.io/lakshmiparamesh/trend-app]
latest: digest: sha256:... size: 1986
```

**DockerHub Repository**: lakshmiparamesh/trend-app

---

## 3. EKS Cluster - kubectl Configuration

### Command: `aws eks update-kubeconfig --name trend-cluster --region ap-south-1`
```
Added new context arn:aws:eks:ap-south-1:009075573616:cluster/trend-cluster to C:\Users\SIS210\.kube\config
```

### Command: `kubectl get nodes -o wide`
```
NAME                                         STATUS   ROLES    AGE   VERSION                INTERNAL-IP   OS-IMAGE                        CONTAINER-RUNTIME
ip-10-0-10-67.ap-south-1.compute.internal    Ready    <none>   52m   v1.31.14-eks-8418f54   10.0.10.67    Amazon Linux 2023.12.20261102   containerd://2.3.7
ip-10-0-11-123.ap-south-1.compute.internal   Ready    <none>   52m   v1.31.14-eks-8418f54   10.0.11.123   Amazon Linux 2023.12.20261102   containerd://2.3.7
```

---

## 4. Kubernetes Deployment to EKS

### Command: `kubectl apply -f k8s/deployment.yaml`
```
deployment.apps/trend-app created
```

### Command: `kubectl apply -f k8s/service.yaml`
```
service/trend-app-service created
```

### Command: `kubectl get pods -o wide`
```
NAME                         READY   STATUS    RESTARTS   AGE    IP            NODE
trend-app-554555b6b6-wbmcx   1/1     Running   0          97s    10.0.11.209   ip-10-0-11-123.ap-south-1.compute.internal
trend-app-554555b6b6-zgdz2   1/1     Running   0          118s   10.0.10.73    ip-10-0-10-67.ap-south-1.compute.internal
```

### Command: `kubectl get svc`
```
NAME                TYPE           CLUSTER-IP      EXTERNAL-IP                                                                PORT(S)        AGE
kubernetes          ClusterIP      172.20.0.1      <none>                                                                     443/TCP        56m
trend-app-service   LoadBalancer   172.20.94.101   a2a3182ced88b4f229dba57e9dd89600-1160890347.ap-south-1.elb.amazonaws.com   80:30851/TCP   35m
```

**Application URL**: http://a2a3182ced88b4f229dba57e9dd89600-1160890347.ap-south-1.elb.amazonaws.com

---

## 5. AWS Auth ConfigMap (Grant Jenkins EKS Access)

### Command: `kubectl apply -f k8s/aws-auth-patch.yaml`
```
configmap/aws-auth configured
```

aws-auth-patch.yaml grants:
- `trend-eks-node-role` → system:bootstrappers, system:nodes
- `trend-jenkins-role` → system:masters (allows Jenkins to deploy)

---

## 6. Jenkins Setup on EC2

### Jenkins EC2 Instance
- **IP**: 43.205.254.170
- **Instance ID**: i-0aa195d125a2fbb50
- **Type**: t3.medium (Ubuntu 22.04)
- **URL**: http://43.205.254.170:8080

### SSH into Jenkins EC2
```
ssh -i trend-jenkins-key.pem ubuntu@43.205.254.170
```

### Jenkins Installation (via WAR file)
```
Jenkins version: 2.580.2
Java version: openjdk-21-jre
Docker: installed and configured
kubectl: installed
AWS CLI: installed with IAM instance profile
```

### Jenkins Plugins Installed
- Docker Pipeline
- Pipeline
- Git
- GitHub Integration
- Kubernetes CLI

### Jenkins Credentials Added
- ID: `dockerhub-credentials` (Username/Password for DockerHub)

### Jenkins Pipeline Job Created
- Name: `trend-app-pipeline`
- Type: Pipeline (from SCM)
- SCM: Git → https://github.com/Lakshmiparamesh/Trend.git
- Branch: */main
- Script Path: Jenkinsfile

---

## 7. Jenkins Pipeline Build #1 — SUCCESS

### Build Details
```
Build #1 (Nov 12, 2026, 6:20:30 AM)
Started by user Lakshmi P
Result: SUCCESS
Duration: 1 min 16 sec

Revision: 32b1969a122bd3ee590be04b6c37d79e84494610
Repository: https://github.com/Lakshmiparamesh/Trend.git
Branch: refs/remotes/origin/main
```

### Pipeline Stages Executed
```
Stage 1: Checkout          ✅ SUCCESS
Stage 2: Docker Build      ✅ SUCCESS  (built lakshmiparamesh/trend-app:1)
Stage 3: Docker Push       ✅ SUCCESS  (pushed to DockerHub)
Stage 4: Update Kubeconfig ✅ SUCCESS  (connected to trend-cluster)
Stage 5: Deploy to EKS     ✅ SUCCESS  (kubectl apply + rollout restart)
```

### Build #2 — Auto-triggered by GitHub Webhook
```
Build #2 triggered automatically after git push (GitHub webhook)
This confirms the CI/CD pipeline auto-trigger is working.
```

---

## 8. GitHub Webhook Configuration

### Webhook Created
```
Webhook ID: 710720626
Payload URL: http://43.205.254.170:8080/github-webhook/
Content Type: application/json
Events: push
Active: true
```

Webhook auto-triggers Jenkins pipeline on every push to main branch.

---

## 9. Monitoring — Prometheus + Grafana

### Command: `kubectl apply -f monitoring/prometheus-config.yaml`
```
namespace/monitoring created
configmap/prometheus-config created
deployment.apps/prometheus created
service/prometheus-service created
```

### Command: `kubectl apply -f monitoring/grafana.yaml`
```
deployment.apps/grafana created
configmap/grafana-datasources created
service/grafana-service created
```

### Command: `kubectl get all -n monitoring`
```
NAME                              READY   STATUS    RESTARTS   AGE
pod/grafana-9458574b6-hf7xw       1/1     Running   0          50m
pod/prometheus-79c49fbffb-htvjv   1/1     Running   0          50m

NAME                         TYPE           CLUSTER-IP      EXTERNAL-IP                                                                PORT(S)          AGE
service/grafana-service      LoadBalancer   172.20.44.205   a5ddf1824bd99469e9c649395791ace9-2055109558.ap-south-1.elb.amazonaws.com   3000:31149/TCP   50m
service/prometheus-service   LoadBalancer   172.20.31.134   a7c8031aa29fd4885b7163ef3519a564-1049899950.ap-south-1.elb.amazonaws.com   9090:32760/TCP   50m

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/grafana      1/1     1            1           50m
deployment.apps/prometheus   1/1     1            1           50m
```

**Prometheus URL**: http://a7c8031aa29fd4885b7163ef3519a564-1049899950.ap-south-1.elb.amazonaws.com:9090
**Grafana URL**: http://a5ddf1824bd99469e9c649395791ace9-2055109558.ap-south-1.elb.amazonaws.com:3000 (admin / admin123)

Grafana auto-provisioned with Prometheus data source (shown as COMPLETE on dashboard).

---

## 10. Screenshots

| Screenshot | Description |
|------------|-------------|
| screenshots/01-app-live.jpg | Trendify app running on EKS LoadBalancer |
| screenshots/02-jenkins-pipeline.jpg | Jenkins pipeline with Build #1 SUCCESS and Build #2 auto-triggered |
| screenshots/03-jenkins-build1-success.jpg | Build #1 details — SUCCESS, 1m16s, all stages passed |
| screenshots/04-prometheus.jpg | Prometheus targets page running on EKS |
| screenshots/05-grafana-dashboard.jpg | Grafana dashboard with Prometheus data source configured |

---

## Summary

| Component | Tool | Status |
|-----------|------|--------|
| Infrastructure | Terraform | 29 resources provisioned (VPC, EKS, EC2, IAM) |
| Containerization | Docker + DockerHub | Image built and pushed (lakshmiparamesh/trend-app) |
| Orchestration | AWS EKS (Kubernetes v1.31) | 2 nodes, 2 pods, LoadBalancer service |
| CI/CD | Jenkins (Declarative Pipeline) | Build #1 SUCCESS, auto-trigger via webhook |
| Monitoring | Prometheus + Grafana | Deployed in monitoring namespace |
| Version Control | GitHub | https://github.com/Lakshmiparamesh/Trend |
