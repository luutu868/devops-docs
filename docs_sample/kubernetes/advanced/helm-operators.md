# Helm & Kubernetes Operators

## Helm - The Kubernetes Package Manager

### **Helm Là Gì?**

**Helm** là package manager cho Kubernetes (giống như apt/yum cho Linux, npm cho Node.js).

**Problems without Helm:**
- Many YAML files to manage (Deployment, Service, ConfigMap, Ingress...)
- Hard to version and track changes
- Difficult to reuse configurations
- No templating or parameterization

**Helm solves:**
- **Charts**: Package of Kubernetes resources
- **Templates**: Parameterize YAML files
- **Releases**: Versioned deployments
- **Repositories**: Share and reuse charts
- **Rollback**: Easy version management

### **Helm Architecture**

```
┌─────────────────────────────────────┐
│         Helm CLI                    │
│   (helm install/upgrade/delete)     │
└────────────┬────────────────────────┘
             ↓
┌────────────────────────────────────┐
│       Helm Chart                   │
│  ┌──────────────────────────────┐  │
│  │  Chart.yaml (metadata)       │  │
│  │  values.yaml (config)        │  │
│  │  templates/ (K8s YAML)       │  │
│  └──────────────────────────────┘  │
└────────────┬────────────────────────┘
             ↓
┌────────────────────────────────────┐
│      Kubernetes Cluster            │
│  Deployment, Service, Ingress...   │
└────────────────────────────────────┘
```

### **Install Helm**

```bash
# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# macOS
brew install helm

# Windows
choco install kubernetes-helm

# Verify
helm version
```

## 🚢 Using Helm Charts

### **Basic Commands**

```bash
# Search charts
helm search hub wordpress
helm search repo nginx

# Add repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add stable https://charts.helm.sh/stable
helm repo update

# List repos
helm repo list

# Search in repo
helm search repo bitnami/nginx

# Install chart
helm install myapp bitnami/nginx

# List releases
helm list

# Uninstall
helm uninstall myapp

# Get values
helm show values bitnami/nginx
```

### **Installing Applications**

#### **1. Install NGINX**

```bash
# Add Bitnami repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Install NGINX
helm install my-nginx bitnami/nginx

# Check installation
kubectl get all
```

#### **2. Install WordPress**

```bash
# Install WordPress with MySQL
helm install my-wordpress bitnami/wordpress

# Get WordPress URL
kubectl get svc my-wordpress
```

#### **3. Install with Custom Values**

```bash
# Get default values
helm show values bitnami/nginx > values.yaml

# Edit values.yaml
# Change replicas, resources, etc.

# Install with custom values
helm install my-nginx bitnami/nginx -f values.yaml

# Or override specific values
helm install my-nginx bitnami/nginx \
  --set replicaCount=3 \
  --set service.type=LoadBalancer
```

### **Helm Release Management**

```bash
# List releases
helm list
helm list --all-namespaces

# Get release info
helm status my-nginx
helm get values my-nginx
helm get manifest my-nginx

# Upgrade release
helm upgrade my-nginx bitnami/nginx -f values.yaml

# Rollback
helm rollback my-nginx 1  # Rollback to revision 1

# History
helm history my-nginx

# Uninstall
helm uninstall my-nginx

# Uninstall but keep history
helm uninstall my-nginx --keep-history
```

## 🛠️ Creating Helm Charts

### **Chart Structure**

```
mychart/
├── Chart.yaml          # Chart metadata
├── values.yaml         # Default configuration values
├── charts/             # Dependency charts
├── templates/          # Kubernetes YAML templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── _helpers.tpl   # Template helpers
│   └── NOTES.txt      # Post-install notes
└── .helmignore        # Files to ignore
```

### **Create New Chart**

```bash
# Create chart
helm create myapp

# Check chart structure
tree myapp/

# Lint chart
helm lint myapp/

# Dry run (test without installing)
helm install myapp ./myapp --dry-run --debug

# Install local chart
helm install myapp ./myapp

# Package chart
helm package myapp/
# Creates: myapp-0.1.0.tgz
```

### **Chart.yaml**

```yaml
apiVersion: v2
name: myapp
description: A Helm chart for my application
type: application
version: 0.1.0        # Chart version
appVersion: "1.0.0"   # Application version

keywords:
  - web
  - application
home: https://myapp.com
sources:
  - https://github.com/mycompany/myapp
maintainers:
  - name: John Doe
    email: john@example.com
icon: https://myapp.com/logo.png

dependencies:
  - name: mysql
    version: 9.3.0
    repository: https://charts.bitnami.com/bitnami
```

### **values.yaml**

```yaml
# Default configuration values

replicaCount: 3

image:
  repository: myapp
  tag: "1.0.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: myapp-tls
      hosts:
        - myapp.example.com

resources:
  limits:
    cpu: 1000m
    memory: 512Mi
  requests:
    cpu: 500m
    memory: 256Mi

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

nodeSelector: {}
tolerations: []
affinity: {}
```

### **templates/deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: {{ .Values.service.targetPort }}
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /health
            port: http
        readinessProbe:
          httpGet:
            path: /ready
            port: http
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
```

### **templates/_helpers.tpl**

```yaml
{{/*
Expand the name of the chart.
*/}}
{{- define "myapp.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "myapp.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "myapp.labels" -}}
helm.sh/chart: {{ include "myapp.chart" . }}
{{ include "myapp.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "myapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### **Helm Template Functions**

```yaml
# Values access
{{ .Values.replicaCount }}
{{ .Values.image.repository }}

# Chart metadata
{{ .Chart.Name }}
{{ .Chart.Version }}
{{ .Release.Name }}
{{ .Release.Namespace }}

# Conditionals
{{- if .Values.ingress.enabled }}
# Create ingress
{{- end }}

# Loops
{{- range .Values.env }}
- name: {{ .name }}
  value: {{ .value }}
{{- end }}

# Default values
{{ .Values.image.tag | default .Chart.AppVersion }}

# String functions
{{ .Values.name | upper }}
{{ .Values.name | lower }}
{{ .Values.name | quote }}

# Include templates
{{ include "myapp.fullname" . }}

# Indentation
{{- toYaml .Values.resources | nindent 12 }}
```

### **Testing Chart**

```bash
# Lint
helm lint ./myapp

# Dry run (see generated YAML)
helm install myapp ./myapp --dry-run --debug

# Template (render templates)
helm template myapp ./myapp

# Install with custom values
helm install myapp ./myapp \
  --set replicaCount=5 \
  --set image.tag=2.0.0

# Upgrade
helm upgrade myapp ./myapp -f prod-values.yaml

# Test release
helm test myapp
```

## 🤖 Kubernetes Operators

### **Operator Là Gì?**

**Operator** là application-specific controller extending Kubernetes API.

**Concepts:**
- **Custom Resources (CR)**: New resource types
- **Custom Resource Definitions (CRD)**: Schema for CRs
- **Controller**: Logic to manage CRs

```
┌──────────────────────────────────────┐
│       Kubernetes API                 │
│                                      │
│  Built-in:                           │
│  - Pod, Deployment, Service          │
│                                      │
│  Custom (via CRD):                   │
│  - MySQL, PostgreSQL, Redis          │
└──────────┬───────────────────────────┘
           ↓
┌──────────────────────────────────────┐
│        Operator                      │
│  (Controller logic)                  │
│  - Watch CRs                         │
│  - Reconcile desired state           │
│  - Handle operations                 │
└──────────┬───────────────────────────┘
           ↓
┌──────────────────────────────────────┐
│    Kubernetes Resources              │
│  StatefulSet, Service, ConfigMap...  │
└──────────────────────────────────────┘
```

### **Why Operators?**

**Problem:**
- Complex applications need operational knowledge
- Backup, restore, upgrade, failover
- Manual operations are error-prone

**Solution:**
- Encode operational knowledge in code
- Automate Day 1 (install) and Day 2 (operate) tasks

### **Operator Examples**

| Operator | Application | Features |
|----------|-------------|----------|
| **Prometheus Operator** | Prometheus | Auto-config, service discovery |
| **MySQL Operator** | MySQL | Backup, restore, HA |
| **PostgreSQL Operator** | PostgreSQL | Replication, failover |
| **MongoDB Operator** | MongoDB | Sharding, backup |
| **Elasticsearch Operator** | Elasticsearch | Cluster management |
| **Istio Operator** | Istio | Service mesh install |

### **Operator Maturity Model**

```
Level 5: Auto Pilot
└─ Full automation, self-healing, auto-tuning

Level 4: Deep Insights
└─ Metrics, alerts, logging, tracing

Level 3: Full Lifecycle
└─ Backup, restore, upgrades, scaling

Level 2: Seamless Upgrades
└─ Rolling updates, version management

Level 1: Basic Install
└─ Automated deployment
```

## 🔧 Using Operators

### **Install Operator Lifecycle Manager (OLM)**

```bash
# Install OLM
curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.25.0/install.sh | bash -s v0.25.0

# Check OLM
kubectl get csv -A
kubectl get installplan -A
```

### **Install Operators Using OLM**

```bash
# Search operators
kubectl get packagemanifests | grep prometheus

# Install Prometheus Operator
kubectl create -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: prometheus
  namespace: operators
spec:
  channel: beta
  name: prometheus
  source: operatorhubio-catalog
  sourceNamespace: olm
EOF

# Check installation
kubectl get csv -n operators
kubectl get pods -n operators
```

### **Example: MySQL Operator**

#### **1. Install MySQL Operator**

```bash
# Add Helm repo
helm repo add mysql-operator https://mysql.github.io/mysql-operator/
helm repo update

# Install operator
helm install mysql-operator mysql-operator/mysql-operator -n mysql-operator --create-namespace

# Check operator
kubectl get pods -n mysql-operator
```

#### **2. Create MySQL Cluster**

```yaml
apiVersion: mysql.oracle.com/v2
kind: InnoDBCluster
metadata:
  name: mysql-cluster
spec:
  secretName: mysql-root-password
  tlsUseSelfSigned: true
  instances: 3
  router:
    instances: 1
  datadirVolumeClaimTemplate:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 10Gi
```

```bash
# Create secret
kubectl create secret generic mysql-root-password \
  --from-literal=rootPassword=MyPassword123

# Create cluster
kubectl apply -f mysql-cluster.yaml

# Check cluster
kubectl get innodbcluster
kubectl get pods

# Connect to MySQL
kubectl run mysql-client --image=mysql:8.0 -it --rm -- \
  mysql -h mysql-cluster -uroot -pMyPassword123
```

#### **3. Operations**

```bash
# Backup
kubectl apply -f - <<EOF
apiVersion: mysql.oracle.com/v2
kind: MySQLBackup
metadata:
  name: backup-1
spec:
  clusterName: mysql-cluster
  backupProfileName: default
EOF

# Scale
kubectl patch innodbcluster mysql-cluster --type merge \
  -p '{"spec":{"instances":5}}'

# Upgrade
kubectl patch innodbcluster mysql-cluster --type merge \
  -p '{"spec":{"version":"8.0.34"}}'
```

### **Example: Prometheus Operator**

#### **1. Install Using Helm**

```bash
# Add Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace

# Check components
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

#### **2. Access Prometheus**

```bash
# Port forward Prometheus
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090

# Open browser: http://localhost:9090

# Port forward Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# Open browser: http://localhost:3000
# Default: admin / prom-operator
```

#### **3. Create ServiceMonitor**

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-monitor
  namespace: default
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

```bash
# Apply ServiceMonitor
kubectl apply -f servicemonitor.yaml

# Prometheus automatically discovers and scrapes metrics
```

## 📚 Popular Operators

### **1. Cert-Manager**

Automate TLS certificate management.

```bash
# Install
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Create ClusterIssuer (Let's Encrypt)
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF

# Use in Ingress
annotations:
  cert-manager.io/cluster-issuer: "letsencrypt-prod"
```

### **2. Velero (Backup)**

Backup and restore Kubernetes resources.

```bash
# Install CLI
brew install velero

# Install Velero server
velero install \
  --provider aws \
  --bucket my-backup-bucket \
  --secret-file ./credentials-velero

# Backup
velero backup create my-backup

# Restore
velero restore create --from-backup my-backup

# Schedule backups
velero schedule create daily --schedule="0 2 * * *"
```

### **3. ArgoCD**

GitOps continuous delivery.

```bash
# Install
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get admin password
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d

# Login
argocd login localhost:8080

# Create application
argocd app create myapp \
  --repo https://github.com/myorg/myrepo \
  --path k8s \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```

## 🎓 Best Practices

### **Helm Best Practices**

1. **Use values.yaml for configuration**
```yaml
# values.yaml - all configurable values
# values-dev.yaml - dev overrides
# values-prod.yaml - prod overrides
```

2. **Version your charts**
```yaml
# Chart.yaml
version: 1.2.3  # Increment on changes
appVersion: "2.0.0"
```

3. **Lint before release**
```bash
helm lint ./mychart
helm template ./mychart --debug
```

4. **Use dependencies**
```yaml
# Chart.yaml
dependencies:
  - name: mysql
    version: 9.x.x
    repository: https://charts.bitnami.com/bitnami
```

5. **Document your chart**
```markdown
# README.md
## Installation
## Configuration
## Upgrading
```

### **Operator Best Practices**

1. **Start with existing operators**
   - Check OperatorHub.io
   - Don't reinvent the wheel

2. **Understand operator capabilities**
   - What Day 2 operations are supported?
   - Auto-scaling? Backup? Restore?

3. **Test operators in dev first**
   - Operators have cluster-wide permissions
   - Test thoroughly before production

4. **Monitor operator health**
```bash
kubectl get pods -n <operator-namespace>
kubectl logs -n <operator-namespace> <operator-pod>
```

## 🚨 Troubleshooting

### **Helm Issues**

```bash
# Release stuck
helm uninstall <release> --wait

# Values not applying
helm get values <release>
helm upgrade <release> ./chart -f values.yaml --debug

# Chart errors
helm lint ./chart
helm template ./chart --debug

# List all releases
helm list --all-namespaces

# Rollback failed upgrade
helm rollback <release> <revision>
```

### **Operator Issues**

```bash
# Operator not starting
kubectl get pods -n <op-namespace>
kubectl logs -n <op-namespace> <op-pod>

# CRD not found
kubectl get crds
kubectl describe crd <crd-name>

# Custom resource not reconciling
kubectl describe <resource-type> <name>
kubectl get events --sort-by='.lastTimestamp'

# Check operator logs
kubectl logs -n <op-namespace> <op-pod> -f
```

## Quick Reference

```bash
# Helm
helm repo add <name> <url>
helm search repo <keyword>
helm install <release> <chart>
helm upgrade <release> <chart>
helm rollback <release> <revision>
helm uninstall <release>
helm list

# Operators
kubectl get crds
kubectl get <crd-type>
kubectl describe <crd-type> <name>
kubectl logs -n <op-ns> <op-pod>

# OLM
kubectl get csv -A
kubectl get operators
kubectl get packagemanifests
```

---

**Tiếp theo**: [Managed Kubernetes →](../managed/eks-gke-aks.md)
