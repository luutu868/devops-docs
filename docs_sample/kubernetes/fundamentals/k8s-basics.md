# Kubernetes Fundamentals

## ☸️ Kubernetes Là Gì?

**Kubernetes (K8s)** là một open-source container orchestration platform được phát triển bởi Google (năm 2014), hiện được maintain bởi Cloud Native Computing Foundation (CNCF).

### **Tại Sao Cần Kubernetes?**

Khi bạn có **1 container** → Dùng Docker là đủ ✅
Khi bạn có **100+ containers** → Cần Kubernetes! 🚀

**Vấn đề Docker không giải quyết được:**
- ❌ Auto-scaling containers
- ❌ Load balancing traffic
- ❌ Self-healing (restart failed containers)
- ❌ Rolling updates & rollbacks
- ❌ Service discovery
- ❌ Secret & config management
- ❌ Multi-host networking

**Kubernetes giải quyết:**
- ✅ Container orchestration
- ✅ Auto-scaling (HPA, VPA, Cluster autoscaler)
- ✅ Self-healing & health checks
- ✅ Declarative configuration
- ✅ Service discovery & load balancing
- ✅ Storage orchestration
- ✅ Automated rollouts & rollbacks
- ✅ Secret & configuration management

## 🏗️ Kubernetes Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    CONTROL PLANE (Master)                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐  │
│  │ API Server   │  │  Scheduler   │  │ Controller Mgr  │  │
│  │ (kube-api)   │  │ (kube-sched) │  │ (kube-ctrl-mgr) │  │
│  └──────────────┘  └──────────────┘  └─────────────────┘  │
│                                                             │
│  ┌────────────────────────────────────────────────────┐    │
│  │              etcd (Key-Value Store)                │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                      WORKER NODES                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Node 1              Node 2              Node 3             │
│  ┌─────────┐        ┌─────────┐        ┌─────────┐         │
│  │ kubelet │        │ kubelet │        │ kubelet │         │
│  │ kube-   │        │ kube-   │        │ kube-   │         │
│  │ proxy   │        │ proxy   │        │ proxy   │         │
│  │         │        │         │        │         │         │
│  │ ┌─────┐ │        │ ┌─────┐ │        │ ┌─────┐ │         │
│  │ │ Pod │ │        │ │ Pod │ │        │ │ Pod │ │         │
│  │ │ Pod │ │        │ │ Pod │ │        │ │ Pod │ │         │
│  │ └─────┘ │        │ └─────┘ │        │ └─────┘ │         │
│  └─────────┘        └─────────┘        └─────────┘         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### **Control Plane Components**

#### **1. API Server (kube-apiserver)**
- RESTful API gateway
- Entry point cho tất cả commands
- Authentication & authorization
- Validates & processes requests

#### **2. etcd**
- Distributed key-value store
- Lưu trữ toàn bộ cluster state
- Single source of truth
- Backup etcd = backup cluster

#### **3. Scheduler (kube-scheduler)**
- Assign Pods to Nodes
- Resource-aware scheduling
- Affinity/anti-affinity rules
- Taints & tolerations

#### **4. Controller Manager (kube-controller-manager)**
- Quản lý các controllers
- Node Controller: Monitor nodes
- Replication Controller: Maintain desired pods
- Endpoints Controller: Populate endpoints
- Service Account Controller: Create default accounts

#### **5. Cloud Controller Manager**
- Tích hợp với cloud providers (AWS, GCP, Azure)
- Node lifecycle
- Load balancers
- Storage volumes

### **Node Components**

#### **1. kubelet**
- Agent chạy trên mỗi node
- Quản lý Pods & containers
- Reports node status to API server
- Executes Pod specs

#### **2. kube-proxy**
- Network proxy
- Manages network rules
- Load balancing for Services
- Forwards traffic to Pods

#### **3. Container Runtime**
- Chạy containers
- Docker, containerd, CRI-O
- Implements Kubernetes CRI (Container Runtime Interface)

## 📦 Core Concepts

### **1. Pod**
```yaml
# Smallest deployable unit
# Can contain 1 or more containers
# Shares network & storage
# Ephemeral (can be destroyed anytime)
```

### **2. ReplicaSet**
```yaml
# Maintains desired number of Pod replicas
# Self-healing
# Usually managed by Deployment
```

### **3. Deployment**
```yaml
# Declarative updates for Pods & ReplicaSets
# Rolling updates
# Rollback capability
# Scaling
```

### **4. Service**
```yaml
# Stable network endpoint for Pods
# Load balancing
# Service discovery
# Types: ClusterIP, NodePort, LoadBalancer
```

### **5. Namespace**
```yaml
# Virtual cluster
# Resource isolation
# Multi-tenancy
# Default namespaces: default, kube-system, kube-public
```

## 🚀 Installation Options

### **Local Development**

#### **Minikube** (Most Popular)
```bash
# Install Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start cluster
minikube start

# Check status
minikube status

# Dashboard
minikube dashboard

# Stop cluster
minikube stop

# Delete cluster
minikube delete
```

#### **Kind (Kubernetes in Docker)**
```bash
# Install Kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Create cluster
kind create cluster

# Create cluster with config
kind create cluster --config kind-config.yaml

# Delete cluster
kind delete cluster
```

#### **Docker Desktop**
```bash
# Enable Kubernetes in Docker Desktop settings
# Easiest for Windows/Mac users
```

### **Production Options**

#### **Managed Kubernetes**
- **AWS EKS** (Elastic Kubernetes Service)
- **Google GKE** (Google Kubernetes Engine)
- **Azure AKS** (Azure Kubernetes Service)
- **DigitalOcean DOKS**
- **Linode LKE**

#### **Self-Managed**
- **kubeadm**: Official tool
- **kops**: Production-grade clusters on AWS
- **Kubespray**: Ansible playbooks
- **Rancher**: Multi-cluster management

## 🔧 kubectl - Kubernetes CLI

### **Installation**

```bash
# Download latest
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Install
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify
kubectl version --client

# Enable autocomplete
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

### **Essential kubectl Commands**

```bash
# Cluster info
kubectl cluster-info
kubectl get nodes
kubectl get componentstatuses
kubectl version

# Get resources
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get all

# Describe resources
kubectl describe pod <pod-name>
kubectl describe node <node-name>

# Create resources
kubectl create -f deployment.yaml
kubectl apply -f deployment.yaml  # Preferred

# Delete resources
kubectl delete pod <pod-name>
kubectl delete -f deployment.yaml

# Logs
kubectl logs <pod-name>
kubectl logs -f <pod-name>  # Follow
kubectl logs <pod-name> -c <container-name>  # Multi-container pod

# Execute commands in pod
kubectl exec <pod-name> -- ls /
kubectl exec -it <pod-name> -- bash

# Port forwarding
kubectl port-forward <pod-name> 8080:80

# Get YAML/JSON
kubectl get pod <pod-name> -o yaml
kubectl get pod <pod-name> -o json

# Edit resources
kubectl edit deployment <deployment-name>

# Scale
kubectl scale deployment <name> --replicas=5

# Rollout
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name>

# Namespaces
kubectl get namespaces
kubectl get pods -n kube-system
kubectl create namespace dev
kubectl config set-context --current --namespace=dev
```

## 📝 First Kubernetes Application

### **Step 1: Create Deployment**

```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```

```bash
# Apply deployment
kubectl apply -f nginx-deployment.yaml

# Check deployment
kubectl get deployments
kubectl get pods
kubectl get rs

# Check details
kubectl describe deployment nginx-deployment
```

### **Step 2: Create Service**

```yaml
# nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  type: NodePort
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080
```

```bash
# Apply service
kubectl apply -f nginx-service.yaml

# Check service
kubectl get services
kubectl describe service nginx-service

# Access service
# If using Minikube:
minikube service nginx-service

# If using NodePort:
curl http://<node-ip>:30080
```

### **Step 3: Scale Application**

```bash
# Scale up
kubectl scale deployment nginx-deployment --replicas=5

# Check pods
kubectl get pods -w  # Watch mode

# Scale down
kubectl scale deployment nginx-deployment --replicas=2
```

### **Step 4: Update Application**

```bash
# Update image
kubectl set image deployment/nginx-deployment nginx=nginx:1.22

# Check rollout status
kubectl rollout status deployment/nginx-deployment

# Check history
kubectl rollout history deployment/nginx-deployment

# Rollback if needed
kubectl rollout undo deployment/nginx-deployment
```

### **Step 5: Cleanup**

```bash
# Delete resources
kubectl delete -f nginx-deployment.yaml
kubectl delete -f nginx-service.yaml

# Or delete by name
kubectl delete deployment nginx-deployment
kubectl delete service nginx-service
```

## 🎯 Kubernetes Configuration Files

### **YAML Structure**

```yaml
apiVersion: apps/v1      # API version
kind: Deployment         # Resource type
metadata:                # Metadata
  name: my-app
  labels:
    app: my-app
spec:                    # Specification
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:              # Pod template
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: my-app:1.0
        ports:
        - containerPort: 8080
```

### **Key Fields**

| Field | Description | Required |
|-------|-------------|----------|
| `apiVersion` | API version | Yes |
| `kind` | Resource type | Yes |
| `metadata` | Resource metadata | Yes |
| `metadata.name` | Resource name | Yes |
| `metadata.labels` | Key-value pairs | No |
| `spec` | Desired state | Yes |

## 🔍 Debugging & Troubleshooting

### **Common Commands**

```bash
# Check pod status
kubectl get pods
kubectl describe pod <pod-name>

# View logs
kubectl logs <pod-name>
kubectl logs <pod-name> --previous  # Previous instance

# Check events
kubectl get events --sort-by=.metadata.creationTimestamp

# Execute in pod
kubectl exec -it <pod-name> -- sh

# Port forward for debugging
kubectl port-forward <pod-name> 8080:80

# Check resource usage
kubectl top nodes
kubectl top pods

# Dry run (test without applying)
kubectl apply -f deployment.yaml --dry-run=client
kubectl apply -f deployment.yaml --dry-run=server

# Explain resources
kubectl explain pods
kubectl explain deployment.spec
```

### **Common Pod States**

| State | Meaning | Action |
|-------|---------|--------|
| `Pending` | Waiting to be scheduled | Check nodes, resources |
| `Running` | Pod is running | ✅ Good |
| `Succeeded` | Completed successfully | ✅ Good for jobs |
| `Failed` | Terminated with error | Check logs |
| `CrashLoopBackOff` | Container keeps crashing | Check logs, fix app |
| `ImagePullBackOff` | Can't pull image | Check image name/registry |
| `ContainerCreating` | Creating container | Wait or check events |

### **Troubleshooting Workflow**

```bash
# 1. Check pod status
kubectl get pods

# 2. Describe pod for events
kubectl describe pod <pod-name>

# 3. Check logs
kubectl logs <pod-name>

# 4. Check events
kubectl get events --field-selector involvedObject.name=<pod-name>

# 5. Execute into pod
kubectl exec -it <pod-name> -- sh

# 6. Check node status
kubectl get nodes
kubectl describe node <node-name>
```

## 🎓 Best Practices

### **1. Use Declarative Configuration**

✅ **Good**: `kubectl apply -f deployment.yaml`
❌ **Bad**: `kubectl create deployment ...`

### **2. Use Namespaces**

```bash
# Separate environments
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace prod
```

### **3. Set Resource Limits**

```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "250m"
  limits:
    memory: "128Mi"
    cpu: "500m"
```

### **4. Use Labels & Selectors**

```yaml
labels:
  app: myapp
  environment: production
  version: v1.0
```

### **5. Health Checks**

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

### **6. Use ConfigMaps & Secrets**

Don't hardcode configs in container images!

### **7. Version Control**

Store all YAML files in Git repository.

## 📊 Kubernetes vs Docker Compose

| Feature | Docker Compose | Kubernetes |
|---------|---------------|------------|
| **Use Case** | Single host, dev | Multi-host, prod |
| **Orchestration** | Basic | Advanced |
| **Scaling** | Manual | Auto-scaling |
| **Self-healing** | No | Yes |
| **Load Balancing** | Basic | Advanced |
| **Rolling Updates** | No | Yes |
| **Service Discovery** | Basic | Advanced |
| **Learning Curve** | Easy | Steep |

## ✅ Quick Reference

```bash
# Essential Commands
kubectl get pods
kubectl describe pod <name>
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- bash
kubectl apply -f <file.yaml>
kubectl delete -f <file.yaml>
kubectl get all
kubectl get events

# Shortcuts
kubectl get po          # pods
kubectl get svc         # services
kubectl get deploy      # deployments
kubectl get rs          # replicasets
kubectl get ns          # namespaces

# Context & Config
kubectl config get-contexts
kubectl config use-context <context-name>
kubectl config set-context --current --namespace=<namespace>
```

## 🎯 Next Steps

1. ✅ Understand Kubernetes architecture
2. ✅ Install kubectl & minikube
3. ✅ Deploy first application
4. ⏭️ Learn about Pods, Deployments, Services (next section)
5. ⏭️ Master Networking & Storage
6. ⏭️ Explore Helm & advanced topics

---

**Tiếp theo**: [Kubernetes Objects (Pods & Deployments) →](../objects/pods-deployments.md)
