# Session Summary: E-commerce Infrastructure Setup

## 🎯 **Session Overview**
**Date**: July 6, 2024  
**Duration**: Complete infrastructure setup and GitHub repository migration  
**Goal**: Set up complete e-commerce application infrastructure with Terraform, AWS, and CI/CD

---

## 📋 **Tasks Completed**

### 1. **Environment Setup & Tool Installation**

#### **Terraform Installation**
- **Status**: ✅ Already installed (v1.12.2)
- **Verification Command**: `terraform -v`
- **Purpose**: Infrastructure as Code (IaC) for AWS resource management

#### **AWS CLI Installation**
- **Status**: ✅ Already installed (v2.27.45)
- **Verification Command**: `aws --version`
- **Purpose**: AWS service interaction and credential management

#### **AWS Credentials Configuration**
- **Command**: `aws configure`
- **Credentials Set**:
  - Access Key ID: `[REDACTED]`
  - Secret Access Key: `[REDACTED]`
  - Default Region: `us-east-2`
  - Output Format: `json`
- **Storage**: `~/.aws/credentials` (local file)
- **Security**: ✅ Credentials safely stored locally
- **Purpose**: Authentication for AWS services

### 2. **Terraform Infrastructure Configuration**

#### **S3 Backend Setup**
- **File Modified**: `terraform/terraform.tf`
- **Original Configuration**: `eu-west-1` region
- **Final Configuration**: `us-east-2` region
- **S3 Bucket**: `terraform-s3-backend-tws-hackathon-us-east-2`
- **Purpose**: Remote state storage for Terraform

#### **Region Migration**
- **File Modified**: `terraform/provider.tf`
- **Changes Made**:
  - Region: `eu-west-1` → `us-east-2`
  - Availability Zones: `eu-west-1a, eu-west-1b` → `us-east-2a, us-east-2b`
- **Purpose**: Align infrastructure with preferred AWS region

#### **SSH Key Generation**
- **Command**: `ssh-keygen -t rsa -b 4096 -f terra-key -N ""`
- **Files Created**:
  - `terraform/terra-key` (private key)
  - `terraform/terra-key.pub` (public key)
- **Purpose**: EC2 instance access and AWS key pair creation

#### **Terraform Initialization**
- **Commands**:
  - `terraform init -reconfigure` (multiple times for region changes)
  - `terraform plan` (verification)
- **Purpose**: Initialize Terraform workspace and validate configuration

### 3. **GitHub Repository Management**

#### **Repository Creation**
- **Tool Used**: GitHub CLI (`gh`)
- **Repository URL**: `https://github.com/wnaveed5/tws-e-commerce-app_hackathon`
- **Commands**:
  - `gh repo create tws-e-commerce-app_hackathon --public --description "E-commerce application with Terraform infrastructure and Kubernetes deployment" --source=. --remote=origin --push`
- **Purpose**: Create new repository under user's account

#### **Branch Management**
- **Original Branch**: `master`
- **Migration Attempt**: `master` → `main` → `master`
- **Commands**:
  - `git branch -m master main`
  - `git push -u origin main`
  - `gh repo edit --default-branch main`
  - `git push origin --delete master`
  - `git branch -m main master`
  - `gh repo edit --default-branch master`
  - `git push origin --delete main`
- **Final State**: `master` branch as default
- **Purpose**: Modernize branch naming (reverted to maintain consistency)

#### **Remote Configuration**
- **Original Remote**: `https://github.com/devopsdock0125/tws-e-commerce-app_hackathon`
- **New Remote**: `https://github.com/wnaveed5/tws-e-commerce-app_hackathon`
- **Commands**:
  - `git remote set-url origin https://github.com/wnaveed5/tws-e-commerce-app_hackathon.git`
- **Purpose**: Transfer repository ownership

### 4. **Code Management & Version Control**

#### **Git Operations**
- **Status Check**: `git status`
- **Staging**: `git add .`
- **Commit**: `git commit -m "Update infrastructure: migrate to us-east-2 region and configure S3 backend"`
- **Push**: `git push origin master`
- **Files Modified**:
  - `Jenkinsfile`
  - `terraform/provider.tf`
  - `terraform/terraform.tf`

### 5. **Load Balancer Controller Installation**

**Note**: All commands in this section are executed on the bastion host (Ubuntu 24.04.2 LTS) via SSH connection.

**Why We Need AWS Load Balancer Controller:**
- **Automatic ALB Creation**: Automatically creates Application Load Balancers (ALBs) when you create Kubernetes Ingress resources
- **External Access**: Provides external access to your applications running in the EKS cluster
- **SSL/TLS Termination**: Handles SSL certificate management and termination at the load balancer level
- **Path-Based Routing**: Routes traffic to different services based on URL paths
- **Health Checks**: Performs health checks on your application pods and routes traffic only to healthy instances
- **Auto Scaling Integration**: Works seamlessly with Kubernetes Horizontal Pod Autoscaler (HPA)
- **Cost Optimization**: Automatically creates and deletes ALBs based on Ingress resources, avoiding unnecessary costs

#### **Download IAM Policy Document:**
```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.13.0/docs/install/iam_policy.json
```

#### **Create AWS Load Balancer Controller IAM Policy:**
```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

#### **Install eksctl (required for IAM service account creation):**
```bash
# Download and install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

# Move to a directory in your PATH
sudo mv /tmp/eksctl /usr/local/bin

# Verify installation
eksctl version
```

#### **Create IAM Service Account for Load Balancer Controller:**
```bash
eksctl create iamserviceaccount \
  --cluster=tws-eks-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::081055084897:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --region us-east-2 \
  --approve
```
**Note**: eksctl uses CloudFormation templates to create and manage IAM service accounts. This ensures proper integration between EKS and IAM, allowing the load balancer controller to assume the necessary IAM role for managing AWS resources.

#### **Add EKS Helm Repository:**
```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
```
**Purpose**: The EKS Helm repository contains the official AWS Load Balancer Controller Helm chart. Adding this repository allows us to install the controller using Helm, which provides a standardized way to deploy and manage Kubernetes applications with proper versioning and configuration management.

#### **Install AWS Load Balancer Controller via Helm:**
```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=tws-eks-cluster \
  --set region=us-east-2 \
  --set vpcId=vpc-0be3037b2732504a4 \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --version 1.13.0
```

#### **Verify Load Balancer Controller Deployment:**
```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```
**Expected Output**: Shows deployment status with pods starting up (READY: 0/2 initially, then 2/2 when fully ready)

**Monitoring Commands**:
```bash
# Check pod status
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller

# Check logs if needed
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

### **EBS CSI Driver Setup (Next Step)**

**Note**: All commands in this section are executed on the bastion host (Ubuntu 24.04.2 LTS) via SSH connection.

**Why We Need EBS CSI Driver:**
- **Persistent Storage**: Allows applications to use persistent volumes for data storage
- **Database Storage**: Essential for databases like MongoDB that need persistent data
- **File Storage**: Provides reliable storage for application files and uploads
- **Stateful Applications**: Required for applications that need to maintain state across pod restarts
- **Data Persistence**: Ensures data survives pod crashes, deployments, and node failures

**Steps Required:**
1. **Create IAM Role for EBS CSI Driver** (using eksctl):
   ```bash
   eksctl create iamserviceaccount \
     --name ebs-csi-controller-sa \
     --namespace kube-system \
     --cluster tws-eks-cluster \
     --role-name AmazonEKS_EBS_CSI_DriverRole \
     --role-only \
     --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
     --approve
   ```

2. **Add EBS CSI Driver Helm Repository:**
   ```bash
   helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
   helm repo update
   ```
   **Purpose**: The AWS EBS CSI Driver Helm repository contains the official driver chart for managing EBS volumes in Kubernetes. Adding this repository allows us to install the driver using Helm for proper versioning and configuration management.

3. **Install EBS CSI Driver via Helm:**
   ```bash
   helm upgrade --install aws-ebs-csi-driver \
     --namespace kube-system \
     aws-ebs-csi-driver/aws-ebs-csi-driver
   ```
   **Purpose**: Installs the AWS EBS CSI Driver in the cluster. The `--upgrade --install` flag ensures it will install if not present or upgrade if already installed. This driver enables Kubernetes to provision, attach, and manage EBS volumes for persistent storage.

4. **Verify EBS CSI Driver Deployment:**
   ```bash
   kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
   ```
   **Expected Output**: Shows EBS CSI Driver pods status (should show controller and node pods running)

5. **Deploy Sample Application** to test persistent storage

**Reference**: [AWS EKS EBS CSI Driver Documentation](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html#eksctl_store_app_data)

### **ArgoCD Setup (Next Step)**

**Note**: All commands in this section are executed on the bastion host (Ubuntu 24.04.2 LTS) via SSH connection.

**Why We Need ArgoCD:**
- **GitOps Workflow**: Implements GitOps principles where Git is the single source of truth
- **Continuous Deployment**: Automatically deploys applications when Git repositories change
- **Application Management**: Provides a web UI and CLI for managing Kubernetes applications
- **Multi-Environment**: Supports managing multiple environments (dev, staging, prod)
- **Rollback Capability**: Easy rollback to previous application versions
- **Health Monitoring**: Monitors application health and sync status
- **Declarative**: Uses declarative manifests for application definitions

**Steps Required:**
1. **Create ArgoCD Namespace:**
   ```bash
   kubectl create namespace argocd
   ```
   **Purpose**: Creates a dedicated namespace for ArgoCD components, providing isolation and organization for the GitOps platform.

2. **Install ArgoCD**
3. **Configure ArgoCD**

#### **Get and Configure ArgoCD Values File:**
```bash
helm show values argo/argo-cd > argocd-values.yaml
```

**Edit the values file with the following configuration:**

**Option 1: With Domain and SSL (Production):**
```yaml
global:
  domain: argocd.example.com

configs:
  params:
    server.insecure: true

server:
  ingress:
    enabled: true
    controller: aws
    ingressClassName: alb
    annotations:
      alb.ingress.kubernetes.io/scheme: internet-facing
      alb.ingress.kubernetes.io/certificate-arn: <your-cert-arn>
      alb.ingress.kubernetes.io/group.name: easyshop-app-lb
      alb.ingress.kubernetes.io/target-type: ip
      alb.ingress.kubernetes.io/backend-protocol: HTTP
      alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80}, {"HTTPS":443}]'
      alb.ingress.kubernetes.io/ssl-redirect: '443'
    hostname: argocd.devopsdock.site
    aws:
      serviceType: ClusterIP
      backendProtocolVersion: GRP
```

**Option 2: Without Domain (Development/Testing):**
```yaml
global:
  domain: argocd.example.com

configs:
  params:
    server.insecure: true

server:
  ingress:
    enabled: true
    controller: aws
    ingressClassName: alb
    annotations:
      alb.ingress.kubernetes.io/scheme: internet-facing
      alb.ingress.kubernetes.io/group.name: easyshop-app-lb
      alb.ingress.kubernetes.io/target-type: ip
      alb.ingress.kubernetes.io/backend-protocol: HTTP
      alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80}]'
    aws:
      serviceType: ClusterIP
      backendProtocolVersion: GRP
```

**Configuration Details:**
- **Domain**: Sets the global domain for ArgoCD
- **Insecure Mode**: Enables insecure server mode for development
- **Ingress**: Configures AWS ALB ingress for external access
- **SSL/TLS**: Sets up HTTPS with certificate and redirect (Option 1 only)
- **Load Balancer**: Uses the same ALB group as the main application
- **Service Type**: ClusterIP for internal communication
- **HTTP Only**: Option 2 uses HTTP only without SSL certificate requirement

4. **Deploy Applications via ArgoCD**

### **ArgoCD Application Setup for E-commerce App**

**Repository Structure:**
- The Kubernetes manifests for the e-commerce application are located in the `kubernetes` directory at the root of the repository: [https://github.com/wnaveed5/tws-e-commerce-app_hackathon/tree/master/kubernetes](https://github.com/wnaveed5/tws-e-commerce-app_hackathon/tree/master/kubernetes)

**ArgoCD Application Spec:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ecommerce
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/wnaveed5/tws-e-commerce-app_hackathon.git
    targetRevision: master
    path: kubernetes
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Key Points:**
- `repoURL` must point to the correct GitHub repository.
- `targetRevision` should match the branch containing your manifests (e.g., `master`).
- `path` must be set to `kubernetes` to match the directory in the repo where the manifests are stored.

**Why:**
- ArgoCD uses these values to locate and deploy your Kubernetes manifests. If the path is incorrect or does not exist, ArgoCD will fail to generate manifests and deployment will not work.

**Result:**
- With the correct `repoURL`, `targetRevision`, and `path`, ArgoCD can successfully deploy the e-commerce application from the specified GitHub repository.

### **Metrics Server Installation**

**Command:**
```bash
helm upgrade --install metrics-server metrics-server/metrics-server
```

**Why We Need Metrics Server:**
- **Resource Monitoring:** Metrics Server collects resource usage data (CPU, memory) from each node and pod in the cluster.
- **Horizontal Pod Autoscaling:** Enables Kubernetes features like Horizontal Pod Autoscaler (HPA) to automatically scale pods based on real-time resource usage.
- **kubectl top:** Allows you to use `kubectl top` to view live resource usage for nodes and pods.
- **Cluster Health:** Provides essential metrics for monitoring and troubleshooting cluster health and performance.
- **Foundation for Observability:** Serves as a foundation for more advanced monitoring and alerting tools.

**Result:**
- With Metrics Server installed, you can monitor resource usage, enable autoscaling, and improve the observability of your Kubernetes workloads.

### **Prometheus & Grafana Monitoring Stack Installation**

**Step 1: Create the Monitoring Namespace**
```bash
kubectl create namespace monitoring
```

**Step 2: Add the Prometheus Community Helm Repository**
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

**Step 3: Install kube-prometheus-stack**
```bash
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring
```

**Why We Need kube-prometheus-stack:**
- **Comprehensive Monitoring:** Installs Prometheus for metrics collection and Grafana for visualization in one step.
- **Kubernetes Integration:** Automatically discovers and monitors Kubernetes components, nodes, and pods.
- **Dashboards:** Provides pre-built Grafana dashboards for cluster, node, and application metrics.
- **Alerting:** Includes Alertmanager for sending notifications based on metric thresholds.
- **Extensibility:** Easily add custom metrics, dashboards, and alert rules.

**Reference:**  
- [kube-prometheus-stack on ArtifactHub](https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack)

**Result:**  
- With kube-prometheus-stack installed, you gain full observability into your Kubernetes cluster, including resource usage, application health, and alerting capabilities.

**Key ALB Ingress Annotations for Grafana and Prometheus:**
- `alb.ingress.kubernetes.io/scheme: internet-facing` — Makes the ALB accessible from the internet.
- `alb.ingress.kubernetes.io/group.name: easyshop-app-lb` — Groups multiple ingresses to share the same ALB.
- `alb.ingress.kubernetes.io/target-type: ip` — Routes traffic directly to pod IPs (required for EKS).
- `alb.ingress.kubernetes.io/backend-protocol: HTTP` — Specifies the backend protocol for the target group.
- `alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80}]'` — Configures the ALB to listen on HTTP port 80.

These annotations ensure that both Grafana and Prometheus are exposed securely and efficiently via the AWS Application Load Balancer (ALB), making them accessible from outside the cluster.

### **Why We Use Ingress Controllers (e.g., for Grafana)**

- **External Access:** Ingress controllers allow you to expose internal Kubernetes services (like Grafana) to the outside world via HTTP/HTTPS.
- **Load Balancing:** They provide load balancing for incoming traffic to your services.
- **Path and Host Routing:** Ingress controllers can route traffic based on URL paths or hostnames, enabling multiple services to share a single load balancer.
- **TLS/SSL Termination:** They handle SSL certificates and HTTPS termination, providing secure access to your services.
- **Centralized Access Management:** Ingress controllers offer a single point to manage access, authentication, and security policies for all your exposed services.
- **Cost Efficiency:** By sharing a single load balancer (like AWS ALB) among multiple services, you reduce cloud costs compared to exposing each service with its own load balancer.
- **Flexibility:** Easily update routing rules, add new services, or change security settings without redeploying your applications.

**In the context of Grafana:**
- Using an ingress controller (like AWS ALB) allows you to securely and efficiently access the Grafana dashboard from outside the cluster, with support for HTTPS, authentication, and custom routing as needed.

---

## 🏗️ **Infrastructure Components**

### **Terraform Resources (65 total)**
1. **VPC & Networking**
   - VPC: `10.0.0.0/16` in `us-east-2`
   - Public Subnets: `10.0.1.0/24`, `10.0.2.0/24`
   - Private Subnets: `10.0.3.0/24`, `10.0.4.0/24`
   - Internet Gateway & NAT Gateway
   - Route Tables & Security Groups

2. **EKS Cluster**
   - Cluster Name: `tws-eks-cluster`
   - Version: `1.31`
   - Node Group: `tws-demo-ng` (t3.large, SPOT instances)
   - Scaling: 1-3 nodes

3. **EC2 Instance (Bastion Host)**
   - Purpose: Secure gateway server for infrastructure access
   - AMI: Ubuntu 24.04
   - Security Group: Ports 22, 80, 443, 8080
   - User Data: `install_tools.sh`
   - SSH Access: `terra-key` private key
   - Role: Command center for EKS cluster management and application deployment

4. **Security & IAM**
   - KMS encryption keys
   - IAM roles and policies
   - Security groups for cluster and nodes

### **Application Components**
1. **Next.js E-commerce App**
   - Framework: Next.js
   - Docker: Multi-stage build
   - Port: 3000

2. **CI/CD Pipeline**
   - Jenkins Pipeline (Jenkinsfile)
   - Docker image building
   - Security scanning (Trivy)
   - Kubernetes deployment

3. **Containerization**
   - Main App: `wnaveed5/easyshop-app`
   - Migration: `wnaveed5/easyshop-migration`
   - Dockerfiles: `Dockerfile`, `Dockerfile.dev`, `scripts/Dockerfile.migration`

---

## 🔧 **Key Files & Their Purposes**

### **Infrastructure Files**
- `terraform/terraform.tf`: S3 backend configuration
- `terraform/provider.tf`: AWS provider and region settings
- `terraform/vpc.tf`: VPC and networking resources
- `terraform/eks.tf`: EKS cluster configuration
- `terraform/ec2.tf`: EC2 instance and security groups
- `terraform/outputs.tf`: Terraform outputs

### **Application Files**
- `Dockerfile`: Production Docker image
- `Dockerfile.dev`: Development Docker image
- `package.json`: Node.js dependencies
- `next.config.js`: Next.js configuration

### **CI/CD Files**
- `Jenkinsfile`: Jenkins pipeline definition
- `kubernetes/`: Kubernetes manifests
- `scripts/install_tools.sh`: EC2 instance setup script
- `vars/update_k8s_manifests.groovy`: Jenkins shared library function for K8s manifest updates

### **Documentation**
- `README.md`: Project documentation
- `about.md`: Project information

---

## 🚨 **Issues Encountered & Resolutions**

### **1. S3 Backend Region Mismatch**
- **Issue**: S3 bucket in `eu-west-1`, backend configured for `us-east-2`
- **Resolution**: Created new S3 bucket in `us-east-2`

### **2. SSH Key Missing**
- **Issue**: Terraform plan failed due to missing `terra-key.pub`
- **Resolution**: Generated new SSH key pair

### **3. Jenkins Shared Library**
- **Issue**: Jenkinsfile references non-existent shared library
- **Resolution**: ✅ Created Jenkins shared libraries repository
- **Repository**: [https://github.com/wnaveed5/jenkins-shared-libraries](https://github.com/wnaveed5/jenkins-shared-libraries)
- **Key File**: `vars/update_k8s_manifests.groovy`
- **Status**: ✅ Resolved - Jenkins pipeline can now load shared library

#### **update_k8s_manifests.groovy Function Details:**
- **Purpose**: Automates Kubernetes manifest updates with new Docker image tags
- **Functionality**:
  - Updates deployment manifests in `kubernetes/` directory
  - Replaces old image tags with new build tags (e.g., `wnaveed5/easyshop-app:123`)
  - Commits updated manifests back to repository using Git credentials
- **Pipeline Integration**: Called in "Update Kubernetes Manifests" stage
- **Parameters**:
  - `imageTag`: Build number from environment
  - `manifestsPath`: 'kubernetes' directory
  - `gitCredentials`: 'github-credentials'
  - `gitUserName`: 'Jenkins CI'
  - `gitUserEmail`: 'wnaveed5@gmail.com'
- **Benefits**: Enables GitOps workflow, automated manifest updates, traceable changes

### **4. Git Remote Conflicts**
- **Issue**: Remote origin already exists when creating new repository
- **Resolution**: Used `git remote set-url` to update existing remote

### **5. EKS Node Group Scaling Issue**
- **Observation**: Only one node was present in the EKS cluster, though three were expected.
- **Terraform Configuration**:  
  - `min_size = 1`  
  - `max_size = 3`  
  - `desired_size = 1` (was the reason for only one node)
- **Action Taken**:  
  - Updated `desired_size` to 3 in `terraform/eks.tf`.
  - Ran `terraform apply` to scale the node group.
- **Result**:  
  - No new nodes appeared after apply; possible drift or Terraform not updating the node group as expected.
- **Next Steps**:  
  - Verify node group status in AWS.
  - Check current nodes in the cluster using `kubectl get nodes` or AWS CLI.

### **6. Bastion Host: EKS Access Troubleshooting & Resolution**
- **Logged into Bastion Host (Ubuntu 24.04.2 LTS)**
- **Configured AWS CLI credentials** using `aws configure` (Access Key, Secret Key, Region: `us-east-2`, Output: `json`)
- **Updated kubeconfig** for EKS with:
  ```bash
  aws eks update-kubeconfig --region us-east-2 --name tws-eks-cluster
  ```
- **Initial `kubectl get nodes` failed** with authentication errors:  
  _"the server has asked for the client to provide credentials"_
- **Created EKS access entry** for IAM user:
  ```bash
  aws eks create-access-entry \
    --cluster-name tws-eks-cluster \
    --principal-arn arn:aws:iam::081055084897:user/itadmin \
    --region us-east-2
  ```
- **Attempted to create access policy** (wrong command: `create-access-policy`), received error and corrected to:
  ```bash
  aws eks associate-access-policy \
    --cluster-name tws-eks-cluster \
    --principal-arn arn:aws:iam::081055084897:user/itadmin \
    --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
    --access-scope type=cluster \
    --region us-east-2
  ```
- **Verified policy association** with:
  ```bash
  aws eks list-associated-access-policies \
    --cluster-name tws-eks-cluster \
    --principal-arn arn:aws:iam::081055084897:user/itadmin \
    --region us-east-2
  ```
- **Confirmed EKS access:**  
  - `kubectl get nodes` now returns the expected node(s):
    ```
    NAME                                       STATUS   ROLES    AGE   VERSION
    ip-10-0-1-230.us-east-2.compute.internal   Ready    <none>   52m   v1.31.7-eks-473151a
    ```

---

## 📊 **Commands Executed Summary**

### **Terraform Commands**
```bash
terraform -v                    # Verify installation
terraform init -reconfigure     # Initialize with new backend
terraform plan                  # Validate configuration
```

### **AWS Commands**
```bash
aws configure                   # Set up credentials
aws configure set region us-east-2  # Set default region
aws s3api create-bucket         # Create S3 backend bucket
```

### **AWS Credentials Management**
- **Access Key ID**: `[REDACTED]`
- **Secret Access Key**: `[REDACTED]`
- **Storage Location**: `~/.aws/credentials`
- **Security Note**: ✅ **Always keep credentials in local file** - AWS Console won't show secret key again after creation
- **Best Practices**:
  - Download CSV file when first creating keys
  - Store securely in `~/.aws/credentials`
  - Set file permissions: `chmod 600 ~/.aws/credentials`
  - Never commit to Git or share publicly
  - Rotate keys every 90 days
  - Use environment variables as alternative: `export AWS_SECRET_ACCESS_KEY=...`

### **Git Commands**
```bash
git remote -v                   # Check remotes
git remote set-url origin       # Update remote URL
git add .                       # Stage changes
git commit -m "message"         # Commit changes
git push origin master          # Push to remote
git branch -m                   # Rename branches
```

### **GitHub CLI Commands**
```bash
gh repo create                  # Create repository
gh repo edit --default-branch   # Change default branch
```

### **SSH Commands**
```bash
ssh-keygen -t rsa -b 4096 -f terra-key -N ""  # Generate SSH keys
```

---

## 🎯 **Next Steps Required**

### **Immediate Actions**
1. **✅ Jenkins Pipeline Fixed**
   - Created Jenkins shared libraries repository
   - Repository: [https://github.com/wnaveed5/jenkins-shared-libraries](https://github.com/wnaveed5/jenkins-shared-libraries)
   - Key file: `vars/update_k8s_manifests.groovy`
   - Pipeline can now load shared library successfully

2. **Deploy Infrastructure**
   - Run `terraform apply`
   - Verify all resources created
   - Test EKS cluster connectivity

3. **Application Deployment**
   - Build and push Docker images
   - Deploy to Kubernetes
   - Verify application functionality

### **Bastion Host Access & Management**
- **Purpose**: Secure gateway server for accessing private infrastructure
- **Security Architecture**:
  - Bastion host in public subnet (internet accessible)
  - EKS cluster in private subnets (protected)
  - All access to private resources through bastion host
- **SSH Access**: Using `terra-key` private key
- **Key Operations from Bastion Host**:
  - Configure kubectl: `aws eks update-kubeconfig --region us-east-2 --name tws-eks-cluster`
  - Verify cluster: `