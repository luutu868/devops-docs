# ArgoCD & GitOps

## GitOps Overview

**GitOps** là practice sử dụng **Git** làm single source of truth cho infrastructure và applications.

**Core Principles:**
1. **Declarative**: System state được define trong Git
2. **Versioned**: All changes tracked in Git history  
3. **Automated**: Changes automatically applied
4. **Continuously reconciled**: Desired state vs actual state

```
┌──────────────────────────────────────────┐
│          Git Repository                  │
│     (Source of Truth)                    │
│  - Kubernetes manifests                  │
│  - Helm charts                           │
│  - Kustomize files                       │
└────────────┬─────────────────────────────┘
             ↓
┌──────────────────────────────────────────┐
│         GitOps Operator                  │
│    (ArgoCD, Flux, etc.)                  │
│  - Monitor Git for changes               │
│  - Apply to cluster                      │
│  - Keep in sync                          │
└────────────┬─────────────────────────────┘
             ↓
┌──────────────────────────────────────────┐
│      Kubernetes Cluster                  │
│  - Actual running state                  │
└──────────────────────────────────────────┘
```

**Traditional vs GitOps:**

| Aspect | Traditional | GitOps |
|--------|-------------|--------|
| **Deployment** | kubectl apply, helm install | Git commit/PR |
| **Source of Truth** | Cluster state | Git repository |
| **Changes** | Imperative | Declarative |
| **Rollback** | Manual | Git revert |
| **Audit** | Logs | Git history |
| **Access** | kubectl access | Git permissions |

## 🐙 ArgoCD Overview

**ArgoCD** là declarative, GitOps continuous delivery tool for Kubernetes.

**Key Features:**
- Automated deployment
- Health status monitoring
- Automated sync
- Rollback capability
- SSO integration
- Multi-cluster support
- Web UI + CLI

## Installation

### **Method 1: kubectl**

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Check installation
kubectl get pods -n argocd

# Wait for pods to be ready
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
```

### **Method 2: Helm**

```bash
# Add Helm repo
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Install ArgoCD
helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --set server.service.type=LoadBalancer

# Check installation
helm list -n argocd
kubectl get pods -n argocd
```

### **Access ArgoCD UI**

```bash
# Method 1: Port forward
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Access: https://localhost:8080

# Method 2: LoadBalancer (if supported)
kubectl get svc argocd-server -n argocd

# Method 3: Ingress
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  ingressClassName: nginx
  rules:
  - host: argocd.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 443
  tls:
  - hosts:
    - argocd.example.com
    secretName: argocd-tls
EOF
```

### **Get Admin Password**

```bash
# Get initial password
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d

# Login: admin / <password>

# Change password (recommended)
argocd account update-password
```

## 🖥️ ArgoCD CLI

### **Install CLI**

```bash
# Linux
curl -sSL -o argocd-linux-amd64 \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

# macOS
brew install argocd

# Verify
argocd version
```

### **Login via CLI**

```bash
# Login
argocd login localhost:8080 \
  --username admin \
  --password <password> \
  --insecure

# Or with port-forward
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
argocd login localhost:8080 --insecure
```

## 📱 Creating Applications

### **Method 1: ArgoCD UI**

1. **New App** → Fill form:
   - Application Name: myapp
   - Project: default
   - Sync Policy: Automatic
   - Repository URL: https://github.com/user/repo
   - Path: k8s/
   - Cluster: https://kubernetes.default.svc
   - Namespace: default

2. **Create** → Application syncs automatically

### **Method 2: ArgoCD CLI**

```bash
# Create application
argocd app create myapp \
  --repo https://github.com/user/repo.git \
  --path k8s \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default

# Sync application
argocd app sync myapp

# Get app details
argocd app get myapp

# List apps
argocd app list

# Delete app
argocd app delete myapp
```

### **Method 3: Kubernetes Manifest**

```yaml
# application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  
  source:
    repoURL: https://github.com/user/repo.git
    targetRevision: HEAD
    path: k8s
  
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f application.yaml
```

## 🔧 Application Configuration

### **Basic Application**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-app
  namespace: argocd
spec:
  project: default
  
  source:
    repoURL: https://github.com/myorg/myapp.git
    targetRevision: main
    path: manifests/
  
  destination:
    server: https://kubernetes.default.svc
    namespace: nginx
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### **Helm Application**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-helm
  namespace: argocd
spec:
  project: default
  
  source:
    repoURL: https://github.com/myorg/helm-charts.git
    targetRevision: main
    path: charts/nginx
    helm:
      releaseName: nginx
      values: |
        replicaCount: 3
        service:
          type: LoadBalancer
      # Or use values file
      valueFiles:
        - values-prod.yaml
      # Override specific values
      parameters:
        - name: image.tag
          value: "1.21"
  
  destination:
    server: https://kubernetes.default.svc
    namespace: nginx
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### **Kustomize Application**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-kustomize
  namespace: argocd
spec:
  project: default
  
  source:
    repoURL: https://github.com/myorg/myapp.git
    targetRevision: main
    path: overlays/production
    kustomize:
      namePrefix: prod-
      nameSuffix: -v2
      images:
        - myapp:latest=myapp:1.2.3
      commonLabels:
        env: production
  
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### **Multi-Source Application**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: multi-source-app
  namespace: argocd
spec:
  project: default
  
  sources:
    - repoURL: https://github.com/myorg/app.git
      targetRevision: main
      path: manifests/
    
    - repoURL: https://charts.helm.sh/stable
      chart: redis
      targetRevision: 17.0.0
      helm:
        values: |
          auth:
            enabled: false
  
  destination:
    server: https://kubernetes.default.svc
    namespace: default
```

## Sync Policies

### **Automatic Sync**

```yaml
syncPolicy:
  automated:
    prune: true      # Delete resources not in Git
    selfHeal: true   # Revert manual changes
    allowEmpty: false
  
  syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - PruneLast=true
  
  retry:
    limit: 5
    backoff:
      duration: 5s
      factor: 2
      maxDuration: 3m
```

### **Manual Sync**

```yaml
syncPolicy:
  syncOptions:
    - CreateNamespace=true
```

```bash
# Sync manually
argocd app sync myapp

# Sync specific resource
argocd app sync myapp --resource=Deployment:default:nginx
```

### **Sync Windows**

```yaml
syncPolicy:
  syncOptions:
    - CreateNamespace=true
  
  # Only sync during specific windows
  syncWindows:
    - kind: allow
      schedule: "0 2 * * *"  # Daily at 2 AM
      duration: 1h
      applications:
        - myapp
      namespaces:
        - production
```

## Projects

**Projects** provide logical grouping of applications.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
spec:
  description: Production applications
  
  # Source repositories
  sourceRepos:
    - https://github.com/myorg/*
  
  # Destination clusters and namespaces
  destinations:
    - namespace: production
      server: https://kubernetes.default.svc
    - namespace: staging
      server: https://kubernetes.default.svc
  
  # Allowed resource types
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
  
  # Denied resource types
  namespaceResourceBlacklist:
    - group: ''
      kind: ResourceQuota
    - group: ''
      kind: LimitRange
  
  # Sync windows
  syncWindows:
    - kind: allow
      schedule: "0 2 * * *"
      duration: 1h
```

## 🔐 RBAC & Security

### **Create Read-Only User**

```yaml
# argocd-cm ConfigMap
data:
  accounts.readonly: apiKey, login
  accounts.readonly.enabled: "true"

# argocd-rbac-cm ConfigMap
policy.csv: |
  p, role:readonly, applications, get, */*, allow
  p, role:readonly, logs, get, */*, allow
  g, readonly, role:readonly

policy.default: role:readonly
```

```bash
# Update password
argocd account update-password --account readonly
````

### **SSO Integration (OAuth/OIDC)**

```yaml
# argocd-cm ConfigMap
data:
  url: https://argocd.example.com
  
  dex.config: |
    connectors:
      - type: github
        id: github
        name: GitHub
        config:
          clientID: $github-client-id
          clientSecret: $github-client-secret
          orgs:
            - name: myorg
```

## 🔔 Notifications

### **Install Notifications**

```bash
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj-labs/argocd-notifications/stable/manifests/install.yaml
```

### **Configure Slack**

```yaml
# argocd-notifications-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token
  
  template.app-deployed: |
    message: |
      Application {{.app.metadata.name}} is now running new version.
    slack:
      attachments: |
        [{
          "title": "{{ .app.metadata.name}}",
          "title_link":"{{.context.argocdUrl}}/applications/{{.app.metadata.name}}",
          "color": "#18be52",
          "fields": [
          {
            "title": "Sync Status",
            "value": "{{.app.status.sync.status}}",
            "short": true
          }]
        }]
  
  trigger.on-deployed: |
    - when: app.status.operationState.phase in ['Succeeded']
      send: [app-deployed]

---
# Application with notifications
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  annotations:
    notifications.argoproj.io/subscribe.on-deployed.slack: my-channel
```

## Monitoring & Health

### **Health Checks**

ArgoCD automatically checks resource health:

```yaml
# Custom health check
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  # ...
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
  
  # Custom health assessment
  healthCheck:
    timeout: 60
    retries: 3
```

### **Metrics**

```bash
# Enable Prometheus metrics
kubectl port-forward svc/argocd-metrics -n argocd 8082:8082

# Metrics endpoint: http://localhost:8082/metrics
```

**Key Metrics:**
- `argocd_app_info`: Application info
- `argocd_app_sync_total`: Sync attempts
- `argocd_app_k8s_request_total`: K8s API requests

## 🎓 Best Practices

### **1. Repository Structure**

```
myapp/
├── base/                    # Base manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── dev/                 # Dev environment
│   │   ├── kustomization.yaml
│   │   └── patches.yaml
│   ├── staging/             # Staging environment
│   │   ├── kustomization.yaml
│   │   └── patches.yaml
│   └── production/          # Production environment
│       ├── kustomization.yaml
│       └── patches.yaml
└── argocd/
    ├── dev-app.yaml
    ├── staging-app.yaml
    └── prod-app.yaml
```

### **2. Use Projects**

```yaml
# Separate projects for environments
- development-project
- staging-project
- production-project
```

### **3. Enable Auto-Sync with Caution**

```yaml
# Development: Auto-sync
syncPolicy:
  automated:
    prune: true
    selfHeal: true

# Production: Manual sync
syncPoli cy:
  syncOptions:
    - CreateNamespace=true
  # No automated sync
```

### **4. Use App of Apps Pattern**

```yaml
# apps/root-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/argocd-apps.git
    path: apps/
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

# This app creates other apps:
# apps/app1.yaml, apps/app2.yaml, etc.
```

### **5. Use Sync Waves**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
  annotations:
    argocd.argoproj.io/sync-wave: "0"  # Create namespace first
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: config
  annotations:
    argocd.argoproj.io/sync-wave: "1"  # Then ConfigMap
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  annotations:
    argocd.argoproj.io/sync-wave: "2"  # Finally Deployment
```

## GitOps Workflow

### **Complete GitOps Pipeline**

```
Developer → Git Push → CI Pipeline → Update Manifests → Git Commit
                                                              ↓
                                                         ArgoCD Sync
                                                              ↓
                                                      Kubernetes Cluster
```

### **Example: Image Update**

```bash
# CI pipeline updates image tag in Git
git clone https://github.com/myorg/k8s-manifests.git
cd k8s-manifests

# Update image tag
sed -i 's|image: myapp:.*|image: myapp:1.2.3|' deployment.yaml

# Commit and push
git add deployment.yaml
git commit -m "Update myapp to 1.2.3"
git push

# ArgoCD detects change and syncs automatically
# (if auto-sync enabled)
```

## 🚨 Troubleshooting

### **Common Issues**

```bash
# App OutOfSync
argocd app get myapp
argocd app diff myapp
argocd app sync myapp

# App stuck syncing
kubectl get application myapp -n argocd -o yaml
kubectl logs -n argocd deployment/argocd-application-controller

# Refresh app
argocd app get myapp --refresh

# Hard refresh
argocd app get myapp --hard-refresh
```

## Quick Reference

```bash
# Applications
argocd app create <name> --repo <url> --path <path> --dest-server <server>
argocd app list
argocd app get <name>
argocd app sync <name>
argocd app delete <name>

# Projects
argocd proj list
argocd proj create <name>

# Clusters
argocd cluster list
argocd cluster add <context>

# Login
argocd login <server>
argocd logout <server>
```

---

**Tiếp theo**: [CI/CD Best Practices →](../best-practices/cicd-best-practices.md)
