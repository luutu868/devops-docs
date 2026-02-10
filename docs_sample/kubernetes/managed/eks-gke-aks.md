# Managed Kubernetes: EKS, GKE, AKS

## ☁️ Managed Kubernetes Overview

**Managed Kubernetes** services run Kubernetes control plane for you.

### **Self-Managed vs Managed**

| Aspect | Self-Managed | Managed |
|--------|--------------|---------|
| **Control Plane** | You manage | Cloud manages |
| **Upgrades** | Manual | Automated/Assisted |
| **HA** | You configure | Built-in |
| **Cost** | Server costs only | Service + servers |
| **Expertise** | High required | Lower required |
| **Flexibility** | Full control | Some limitations |

### **Popular Managed Services**

| Provider | Service | Market Share |
|----------|---------|--------------|
| **Amazon** | EKS (Elastic Kubernetes Service) | ~45% |
| **Google** | GKE (Google Kubernetes Engine) | ~25% |
| **Microsoft** | AKS (Azure Kubernetes Service) | ~20% |
| **Others** | DigitalOcean, Linode, etc. | ~10% |

### **Comparison Matrix**

| Feature | AWS EKS | GCP GKE | Azure AKS |
|---------|---------|---------|-----------|
| **Control Plane Cost** | $0.10/hour ($73/month) | Free | Free |
| **Node Cost** | EC2 pricing | GCE pricing | VM pricing |
| **Kubernetes Version** | Usually 1-2 behind | Latest + older | Latest + 2 older |
| **Auto-scaling** | ✅ Cluster Autoscaler | ✅ Native | ✅ Cluster Autoscaler |
| **Networking** | AWS VPC CNI | GCP VPC-native | Azure CNI/Kubenet |
| **Load Balancer** | ELB/ALB/NLB | Cloud Load Balancing | Azure LB |
| **Storage** | EBS, EFS, FSx | Persistent Disk | Azure Disk/Files |
| **Registry** | ECR | GCR/Artifact Registry | ACR |
| **Serverless** | Fargate | Autopilot | AKS Virtual Nodes |
| **Service Mesh** | App Mesh | Anthos Service Mesh | Open Service Mesh |
| **GitOps** | ✅ | ✅ Anthos Config Mgmt | ✅ GitOps with Flux |

## 🟠 AWS EKS (Elastic Kubernetes Service)

### **EKS Architecture**

```
┌──────────────────────────────────────────────┐
│          AWS EKS Control Plane               │
│         (Managed by AWS)                     │
│  ┌────────────────────────────────────────┐  │
│  │  API Server, etcd, Controller Manager │  │
│  │  Scheduler, Cloud Controller Manager  │  │
│  └────────────────────────────────────────┘  │
└──────────────┬───────────────────────────────┘
               ↓
┌──────────────────────────────────────────────┐
│          Data Plane (Your VPC)               │
│  ┌─────────────┐  ┌─────────────┐           │
│  │  Worker     │  │  Worker     │           │
│  │  Node (EC2) │  │  Node (EC2) │           │
│  └─────────────┘  └─────────────┘           │
│                                              │
│  Or:                                         │
│  ┌─────────────────────────────────┐        │
│  │   Fargate Pods (Serverless)     │        │
│  └─────────────────────────────────┘        │
└──────────────────────────────────────────────┘
```

### **Prerequisites**

```bash
# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Configure AWS
aws configure
# Enter: Access Key, Secret Key, Region, Output format

# Install eksctl (EKS CLI)
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Verify
aws --version
eksctl version
kubectl version --client
```

### **Create EKS Cluster**

#### **Method 1: Using eksctl (Recommended)**

```bash
# Basic cluster
eksctl create cluster \
  --name my-cluster \
  --region us-west-2 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 1 \
  --nodes-max 4 \
  --managed

# Get kubeconfig
aws eks update-kubeconfig --region us-west-2 --name my-cluster

# Verify
kubectl get nodes
```

#### **Method 2: Using eksctl Config File**

```yaml
# cluster.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: production-cluster
  region: us-west-2
  version: "1.28"

vpc:
  cidr: 10.0.0.0/16
  nat:
    gateway: HighlyAvailable  # NAT Gateway per AZ

iam:
  withOIDC: true  # Enable IAM Roles for Service Accounts

managedNodeGroups:
  - name: general-workers
    instanceType: t3.medium
    minSize: 2
    maxSize: 10
    desiredCapacity: 3
    volumeSize: 50
    ssh:
      allow: true
      publicKeyName: my-key
    labels:
      role: general
    tags:
      nodegroup-role: general
    iam:
      withAddonPolicies:
        autoScaler: true
        ebs: true
        efs: true
        albIngress: true
        cloudWatch: true

  - name: cpu-workers
    instanceType: c5.2xlarge
    minSize: 0
    maxSize: 5
    desiredCapacity: 0
    labels:
      role: cpu-intensive
    taints:
      - key: workload
        value: cpu-intensive
        effect: NoSchedule

cloudWatch:
  clusterLogging:
    enableTypes: ["api", "audit", "authenticator", "controllerManager", "scheduler"]
```

```bash
# Create cluster
eksctl create cluster -f cluster.yaml

# This takes 15-20 minutes
```

### **EKS with Fargate**

Run pods **without managing nodes** (serverless).

```yaml
# fargate-cluster.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: fargate-cluster
  region: us-west-2

fargateProfiles:
  - name: fp-default
    selectors:
      - namespace: default
        labels:
          fargate: "yes"
  
  - name: fp-kube-system
    selectors:
      - namespace: kube-system
```

```bash
# Create cluster
eksctl create cluster -f fargate-cluster.yaml

# Deploy to Fargate
kubectl run nginx --image=nginx -l fargate=yes

# Pod runs on Fargate (no nodes)
kubectl get pods -o wide
```

### **EKS Add-ons**

#### **1. AWS Load Balancer Controller**

```bash
# Install
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

#### **2. EBS CSI Driver (Storage)**

```bash
# Install add-on
eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster my-cluster \
  --service-account-role-arn arn:aws:iam::ACCOUNT_ID:role/AmazonEKS_EBS_CSI_DriverRole \
  --force
```

#### **3. Cluster Autoscaler**

```bash
# Install
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml

# Edit deployment
kubectl edit deployment cluster-autoscaler -n kube-system
# Add: --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/<cluster-name>
```

### **EKS Best Practices**

```bash
# 1. Use managed node groups
managedNodeGroups: []

# 2. Enable OIDC for IAM roles
iam:
  withOIDC: true

# 3. Enable control plane logging
cloudWatch:
  clusterLogging:
    enableTypes: ["all"]

# 4. Use private subnets for nodes
vpc:
  subnets:
    private: {}

# 5. Enable encryption
secretsEncryption:
  keyARN: arn:aws:kms:...
```

## 🔵 GCP GKE (Google Kubernetes Engine)

### **GKE Architecture**

```
┌──────────────────────────────────────────────┐
│          GKE Control Plane                   │
│         (Managed by Google)                  │
│  ┌────────────────────────────────────────┐  │
│  │  Regional Control Plane (HA)          │  │
│  │  API, etcd, Controller Manager        │  │
│  └────────────────────────────────────────┘  │
└──────────────┬───────────────────────────────┘
               ↓
┌──────────────────────────────────────────────┐
│          Node Pools (GCE VMs)                │
│  ┌─────────────┐  ┌─────────────┐           │
│  │  Worker     │  │  Worker     │           │
│  │  Node       │  │  Node       │           │
│  └─────────────┘  └─────────────┘           │
└──────────────────────────────────────────────┘
```

### **Prerequisites**

```bash
# Install gcloud CLI
curl https://sdk.cloud.google.com | bash
exec -l $SHELL

# Initialize gcloud
gcloud init

# Install kubectl
gcloud components install kubectl

# Install gke-gcloud-auth-plugin
gcloud components install gke-gcloud-auth-plugin

# Set project
gcloud config set project PROJECT_ID

# Verify
gcloud --version
kubectl version --client
```

### **Create GKE Cluster**

#### **Standard Cluster**

```bash
# Basic cluster
gcloud container clusters create my-cluster \
  --zone us-central1-a \
  --num-nodes 3 \
  --machine-type n1-standard-2 \
  --disk-size 50 \
  --enable-autoscaling \
  --min-nodes 1 \
  --max-nodes 5 \
  --enable-autorepair \
  --enable-autoupgrade

# Get credentials
gcloud container clusters get-credentials my-cluster --zone us-central1-a

# Verify
kubectl get nodes
```

#### **Regional Cluster (HA)**

```bash
# Multi-zone cluster
gcloud container clusters create production-cluster \
  --region us-central1 \
  --num-nodes 1 \
  --machine-type n1-standard-4 \
  --enable-autoscaling \
  --min-nodes 1 \
  --max-nodes 10 \
  --enable-autorepair \
  --enable-autoupgrade \
  --enable-ip-alias \
  --network "default" \
  --subnetwork "default" \
  --enable-stackdriver-kubernetes \
  --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver \
  --workload-pool=PROJECT_ID.svc.id.goog \
  --enable-shielded-nodes
```

#### **GKE Autopilot (Fully Managed)**

```bash
# Autopilot - Google manages nodes completely
gcloud container clusters create-auto autopilot-cluster \
  --region us-central1

# No node management needed
# Pay per Pod resource usage
```

**Autopilot vs Standard:**

| Feature | Standard | Autopilot |
|---------|----------|-----------|
| **Node Management** | You manage | Google manages |
| **Pricing** | Per node | Per Pod resource |
| **Control** | Full | Limited |
| **Security** | You configure | Hardened by default |
| **Scaling** | Manual/Auto | Automatic |

### **GKE Node Pools**

```bash
# Create additional node pool
gcloud container node-pools create gpu-pool \
  --cluster my-cluster \
  --zone us-central1-a \
  --machine-type n1-standard-4 \
  --num-nodes 0 \
  --enable-autoscaling \
  --min-nodes 0 \
  --max-nodes 5 \
  --accelerator type=nvidia-tesla-t4,count=1

# List node pools
gcloud container node-pools list --cluster my-cluster --zone us-central1-a

# Delete node pool
gcloud container node-pools delete gpu-pool --cluster my-cluster --zone us-central1-a
```

### **GKE Features**

#### **1. Workload Identity**

Map Kubernetes ServiceAccount to Google Cloud Service Account:

```bash
# Enable Workload Identity
gcloud container clusters update my-cluster \
  --workload-pool=PROJECT_ID.svc.id.goog

# Create service account
gcloud iam service-accounts create gcs-reader

# Bind to K8s ServiceAccount
kubectl create serviceaccount gcs-reader

gcloud iam service-accounts add-iam-policy-binding \
  gcs-reader@PROJECT_ID.iam.gserviceaccount.com \
  --member "serviceAccount:PROJECT_ID.svc.id.goog[default/gcs-reader]" \
  --role roles/storage.objectViewer

# Use in Pod
kubectl annotate serviceaccount gcs-reader \
  iam.gke.io/gcp-service-account=gcs-reader@PROJECT_ID.iam.gserviceaccount.com
```

#### **2. GKE Ingress (GCLB)**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    kubernetes.io/ingress.class: "gce"
    kubernetes.io/ingress.global-static-ip-name: "web-static-ip"
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

#### **3. Cloud Logging & Monitoring**

```bash
# Enable
gcloud container clusters update my-cluster \
  --enable-cloud-logging \
  --enable-cloud-monitoring \
  --logging=SYSTEM,WORKLOAD \
  --monitoring=SYSTEM
```

### **GKE Best Practices**

```bash
# 1. Use regional clusters (HA)
--region us-central1

# 2. Enable Workload Identity
--workload-pool=PROJECT_ID.svc.id.goog

# 3. Enable Binary Authorization
--enable-binauthz

# 4. Use Shielded GKE Nodes
--enable-shielded-nodes

# 5. Enable automatic upgrades
--enable-autoupgrade --enable-autorepair

# 6. Use VPC-native clusters
--enable-ip-alias
```

## 🔷 Azure AKS (Azure Kubernetes Service)

### **AKS Architecture**

```
┌──────────────────────────────────────────────┐
│          AKS Control Plane                   │
│         (Managed by Azure)                   │
│  ┌────────────────────────────────────────┐  │
│  │  API Server, etcd, Controllers         │  │
│  └────────────────────────────────────────┘  │
└──────────────┬───────────────────────────────┘
               ↓
┌──────────────────────────────────────────────┐
│          Node Pools (Azure VMs)              │
│  ┌─────────────┐  ┌─────────────┐           │
│  │  System     │  │  User        │           │
│  │  Node Pool  │  │  Node Pool   │           │
│  └─────────────┘  └─────────────┘           │
└──────────────────────────────────────────────┘
```

### **Prerequisites**

```bash
# Install Azure CLI
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Login
az login

# Install kubectl
az aks install-cli

# Verify
az --version
kubectl version --client
```

### **Create AKS Cluster**

#### **Basic Cluster**

```bash
# Create resource group
az group create \
  --name myResourceGroup \
  --location eastus

# Create AKS cluster
az aks create \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --node-count 3 \
  --node-vm-size Standard_D2s_v3 \
  --enable-addons monitoring \
  --enable-managed-identity \
  --generate-ssh-keys

# Get credentials
az aks get-credentials \
  --resource-group myResourceGroup \
  --name myAKSCluster

# Verify
kubectl get nodes
```

#### **Production Cluster**

```bash
# Advanced cluster
az aks create \
  --resource-group myResourceGroup \
  --name production-cluster \
  --location eastus \
  --node-count 3 \
  --node-vm-size Standard_D4s_v3 \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 10 \
  --network-plugin azure \
  --network-policy azure \
  --service-cidr 10.0.0.0/16 \
  --dns-service-ip 10.0.0.10 \
  --docker-bridge-address 172.17.0.1/16 \
  --vnet-subnet-id /subscriptions/.../subnets/aks-subnet \
  --enable-managed-identity \
  --enable-addons monitoring,azure-policy \
  --enable-azure-rbac \
  --enable-pod-identity \
  --zones 1 2 3
```

### **AKS Node Pools**

```bash
# Add node pool
az aks nodepool add \
  --resource-group myResourceGroup \
  --cluster-name myAKSCluster \
  --name gpupool \
  --node-count 1 \
  --node-vm-size Standard_NC6 \
  --enable-cluster-autoscaler \
  --min-count 0 \
  --max-count 3 \
  --node-taints sku=gpu:NoSchedule

# List node pools
az aks nodepool list \
  --resource-group myResourceGroup \
  --cluster-name myAKSCluster

# Scale node pool
az aks nodepool scale \
  --resource-group myResourceGroup \
  --cluster-name myAKSCluster \
  --name agentpool \
  --node-count 5

# Delete node pool
az aks nodepool delete \
  --resource-group myResourceGroup \
  --cluster-name myAKSCluster \
  --name gpupool
```

### **AKS Features**

#### **1. Azure AD Integration**

```bash
# Enable Azure AD
az aks update \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --enable-aad \
  --enable-azure-rbac
```

#### **2. Azure Container Registry (ACR) Integration**

```bash
# Create ACR
az acr create --resource-group myResourceGroup --name myACR --sku Standard

# Attach ACR to AKS
az aks update \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --attach-acr myACR

# Push image
az acr build --registry myACR --image myapp:v1 .

# Use in Pod
image: myacr.azurecr.io/myapp:v1
```

#### **3. Azure Monitor for Containers**

```bash
# Enable monitoring
az aks enable-addons \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --addons monitoring

# View logs in Azure Portal → Monitor → Containers
```

#### **4. Virtual Nodes (ACI)**

```bash
# Enable virtual nodes (serverless)
az aks enable-addons \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --addons virtual-node \
  --subnet-name virtual-node-subnet

# Deploy to virtual node
kubectl label namespace default kubernetes.azure.com/mode=virtual
```

### **AKS Best Practices**

```bash
# 1. Use managed identity
--enable-managed-identity

# 2. Enable cluster autoscaler
--enable-cluster-autoscaler

# 3. Use Azure CNI for networking
--network-plugin azure

# 4. Enable monitoring
--enable-addons monitoring

# 5. Use multiple node pools
az aks nodepool add ...

# 6. Enable availability zones
--zones 1 2 3

# 7. Integrate with ACR
--attach-acr <acr-name>
```

## 📊 Cost Optimization

### **General Tips**

1. **Use autoscaling**
```bash
# Scale down when idle
--enable-cluster-autoscaler --min-count 1
```

2. **Use spot/preemptible instances**
```bash
# AWS: Spot instances (70% discount)
# GCP: Preemptible VMs (80% discount)
# Azure: Spot VMs (90% discount)
```

3. **Right-size nodes**
```bash
# Don't use oversized instances
# Monitor resource usage, adjust accordingly
```

4. **Use serverless for burst workloads**
```bash
# AWS Fargate, GKE Autopilot, AKS Virtual Nodes
```

5. **Set resource limits**
```yaml
resources:
  limits:
    cpu: 500m
    memory: 512Mi
```

6. **Delete unused resources**
```bash
# Remove idle node pools, clusters
```

## ✅ Quick Reference

### **AWS EKS**

```bash
eksctl create cluster -f cluster.yaml
aws eks update-kubeconfig --name <cluster>
eksctl delete cluster --name <cluster>
```

### **GCP GKE**

```bash
gcloud container clusters create <cluster>
gcloud container clusters get-credentials <cluster>
gcloud container clusters delete <cluster>
```

### **Azure AKS**

```bash
az aks create -g <rg> -n <cluster>
az aks get-credentials -g <rg> -n <cluster>
az aks delete -g <rg> -n <cluster>
```

---

**🎉 Hoàn thành phần Kubernetes!**

**Bạn đã học:**
- ✅ Kubernetes Fundamentals
- ✅ Pods & Deployments
- ✅ Services & Ingress
- ✅ Volumes & Storage
- ✅ Helm & Operators
- ✅ Managed Kubernetes (EKS, GKE, AKS)

**Tiếp theo**: [CI/CD Fundamentals →](../../cicd/fundamentals/cicd-basics.md)
