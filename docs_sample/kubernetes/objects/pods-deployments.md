# Kubernetes Objects: Pods & Deployments

## 🎯 Kubernetes Objects Overview

Kubernetes objects là **persistent entities** trong Kubernetes system. Chúng represent state của cluster:
- Containers nào đang chạy
- Resources available cho applications
- Policies cho applications (restart, upgrades, fault-tolerance)

## 📦 Pods

### **Pod Là Gì?**

**Pod** là smallest deployable unit trong Kubernetes. Đây là wrapper around one hoặc more containers.

```
┌──────────────────────────────────┐
│          Pod                     │
│  ┌────────────────────────────┐  │
│  │  Container 1 (Main App)    │  │
│  └────────────────────────────┘  │
│  ┌────────────────────────────┐  │
│  │  Container  2 (Sidecar)    │  │
│  └────────────────────────────┘  │
│                                  │
│  Shared:                         │
│  - Network (localhost)           │
│  - Storage (Volumes)             │
│  - IPC namespace                 │
└──────────────────────────────────┘
```

### **Đặc Điểm của Pod**

✅ **Ephemeral**: Pods can be created/destroyed anytime
✅ **Unique IP**: Mỗi Pod có IP address riêng
✅ **Shared Network**: Containers trong Pod dùng chung network namespace
✅ **Shared Storage**: Containers có thể share volumes
✅ **Atomic Unit**: Scale bằng cách tạo nhiều Pods, không scale containers trong Pod

### **Single vs Multi-Container Pods**

#### **Single Container Pod** (Phổ biến nhất - 95%)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
```

#### **Multi-Container Pod** (Sidecar Pattern)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
spec:
  containers:
  # Main application
  - name: app
    image: myapp:1.0
    ports:
    - containerPort: 8080
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/app
  
  # Sidecar: Log shipper
  - name: log-shipper
    image: fluent/fluentd:latest
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/app
      readOnly: true
  
  volumes:
  - name: shared-logs
    emptyDir: {}
```

### **Pod Lifecycle**

```
┌──────────┐
│ Pending  │ --> Pulling image, scheduling
└────┬─────┘
     ↓
┌──────────┐
│ Running  │ --> At least 1 container running
└────┬─────┘
     ↓
┌──────────┐     ┌───────────┐
│Succeeded │ or  │  Failed   │
└──────────┘     └───────────┘
```

**Pod Phases:**

| Phase | Description |
|-------|-------------|
| `Pending` | Pod accepted but not running yet |
| `Running` | Pod bound to node, containers created |
| `Succeeded` | All containers terminated successfully |
| `Failed` | At least one container failed |
| `Unknown` | Cannot determine Pod state |

### **Container States**

| State | Description |
|-------|-------------|
| `Waiting` | Not running, pulling image or waiting |
| `Running` | Container executing |
| `Terminated` | Execution completed or failed |

### **Pod Commands**

```bash
# Create Pod
kubectl run nginx --image=nginx
kubectl apply -f pod.yaml

# List Pods
kubectl get pods
kubectl get pods -o wide  # More details (IP, Node)
kubectl get pods -w       # Watch mode

# Describe Pod
kubectl describe pod nginx

# Get Pod logs
kubectl logs nginx
kubectl logs nginx -f     # Follow logs
kubectl logs nginx --previous  # Previous container logs

# Execute in Pod
kubectl exec nginx -- ls /
kubectl exec -it nginx -- bash

# Delete Pod
kubectl delete pod nginx
kubectl delete -f pod.yaml

# Port forward
kubectl port-forward nginx 8080:80

# Copy files
kubectl cp nginx:/etc/nginx/nginx.conf ./nginx.conf
kubectl cp ./index.html nginx:/usr/share/nginx/html/

# Get Pod YAML
kubectl get pod nginx -o yaml
kubectl get pod nginx -o json
```

### **Pod Manifest Anatomy**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: default
  labels:
    app: my-app
    version: v1
    environment: production
  annotations:
    description: "My production app"
    
spec:
  # Containers
  containers:
  - name: app
    image: myapp:1.0
    imagePullPolicy: IfNotPresent  # Always, Never, IfNotPresent
    
    # Ports
    ports:
    - containerPort: 8080
      protocol: TCP
      name: http
    
    # Environment Variables
    env:
    - name: ENV
      value: "production"
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    
    # Resources
    resources:
      requests:
        memory: "128Mi"
        cpu: "500m"
      limits:
        memory: "256Mi"
        cpu: "1000m"
    
    # Health Checks
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
    
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
    
    # Volumes
    volumeMounts:
    - name: config
      mountPath: /etc/config
    - name: data
      mountPath: /data
  
  # Init Containers (run before main containers)
  initContainers:
  - name: init-db
    image: busybox
    command: ['sh', '-c', 'until nc -z db 5432; do echo waiting for db; sleep 2; done']
  
  # Volumes
  volumes:
  - name: config
    configMap:
      name: app-config
  - name: data
    persistentVolumeClaim:
      claimName: app-data
  
  # Restart Policy
  restartPolicy: Always  # Always, OnFailure, Never
  
  # Node Selection
  nodeSelector:
    disktype: ssd
  
  # Tolerations
  tolerations:
  - key: "key1"
    operator: "Equal"
    value: "value1"
    effect: "NoSchedule"
```

### **Init Containers**

Init containers chạy **trước** main containers. Use cases:
- Wait for service dependencies
- Setup configuration files
- Database migrations
- Security scans

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c', 'until nslookup db.default.svc.cluster.local; do echo waiting; sleep 2; done']
  
  - name: init-config
    image: busybox
    command: ['sh', '-c', 'echo "Config setup" > /work-dir/config.txt']
    volumeMounts:
    - name: workdir
      mountPath: /work-dir
  
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: workdir
      mountPath: /config
  
  volumes:
  - name: workdir
    emptyDir: {}
```

## 🚀 Deployments

### **Deployment Là Gì?**

**Deployment** provides declarative updates for Pods and ReplicaSets.

**Vấn đề với bare Pods:**
- ❌ Không tự phục hồi nếu crash
- ❌ Không scale được
- ❌ Không có rolling updates
- ❌ Không có rollback

**Deployment giải quyết:**
- ✅ Maintain desired number of replicas
- ✅ Self-healing
- ✅ Rolling updates
- ✅ Rollback
- ✅ Scaling
- ✅ Pause/Resume

### **Deployment Architecture**

```
┌─────────────────────────────────────┐
│         Deployment                  │
│  (Manages ReplicaSets)              │
└────────────┬────────────────────────┘
             │
    ┌────────┴────────┐
    │                 │
┌───▼────┐      ┌────▼───┐
│  RS v1 │      │  RS v2 │
│ (old)  │      │ (new)  │
└───┬────┘      └────┬───┘
    │                │
    └────────┬───────┘
             │
    ┌────────┴────────┐
    │        │        │
┌───▼──┐ ┌──▼───┐ ┌──▼───┐
│ Pod1 │ │ Pod2 │ │ Pod3 │
└──────┘ └──────┘ └──────┘
```

### **Basic Deployment**

```yaml
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
```

```bash
# Create Deployment
kubectl apply -f deployment.yaml

# Check Deployment
kubectl get deployments
kubectl get deploy

# Check ReplicaSets
kubectl get replicasets
kubectl get rs

# Check Pods
kubectl get pods

# All in one
kubectl get deploy,rs,pods
```

### **Deployment Operations**

#### **1. Scaling**

```bash
# Scale using kubectl
kubectl scale deployment nginx-deployment --replicas=5

# Scale using YAML
kubectl apply -f deployment.yaml  # After changing replicas in YAML

# Autoscaling
kubectl autoscale deployment nginx-deployment --min=3 --max=10 --cpu-percent=80
```

#### **2. Rolling Updates**

```bash
# Update image
kubectl set image deployment/nginx-deployment nginx=nginx:1.22

# Edit deployment
kubectl edit deployment nginx-deployment

# Apply updated YAML
kubectl apply -f deployment.yaml

# Check rollout status
kubectl rollout status deployment/nginx-deployment

# Watch Pods being updated
kubectl get pods -w
```

**Rolling Update Strategy:**

```yaml
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max new pods during update
      maxUnavailable: 1  # Max unavailable pods during update
```

**Update Process:**
```
Initial:   [v1] [v1] [v1]
Step 1:    [v1] [v1] [v1] [v2]  (maxSurge: +1)
Step 2:    [v1] [v1] [v2] [v2]  (replace 1)
Step 3:    [v1] [v2] [v2] [v2]  (replace 1)
Final:     [v2] [v2] [v2]
```

#### **3. Rollback**

```bash
# Check rollout history
kubectl rollout history deployment/nginx-deployment

# Check specific revision
kubectl rollout history deployment/nginx-deployment --revision=2

# Rollback to previous version
kubectl rollout undo deployment/nginx-deployment

# Rollback to specific revision
kubectl rollout undo deployment/nginx-deployment --to-revision=2

# Pause rollout
kubectl rollout pause deployment/nginx-deployment

# Resume rollout
kubectl rollout resume deployment/nginx-deployment
```

#### **4. Restart Deployment**

```bash
# Restart all Pods (rolling restart)
kubectl rollout restart deployment/nginx-deployment
```

### **Deployment Strategies**

#### **1. RollingUpdate (Default)**

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
```

**Pros:**
- ✅ Zero downtime
- ✅ Gradual traffic shift
- ✅ Easy rollback

**Cons:**
- ❌ Both versions running simultaneously
- ❌ Slower deployment

#### **2. Recreate**

```yaml
spec:
  strategy:
    type: Recreate
```

**Process:**
1. Terminate all old Pods
2. Create new Pods

**Pros:**
- ✅ Simple
- ✅ Only one version at a time

**Cons:**
- ❌ Downtime during update
- ❌ Not suitable for production

### **Complete Deployment Example**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: production
  labels:
    app: webapp
    version: v2
spec:
  replicas: 5
  
  # Selector
  selector:
    matchLabels:
      app: webapp
  
  # Update strategy
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 1
  
  # Min ready seconds
  minReadySeconds: 10
  
  # Revision history limit
  revisionHistoryLimit: 10
  
  # Pod template
  template:
    metadata:
      labels:
        app: webapp
        version: v2
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    
    spec:
      # Init container
      initContainers:
      - name: migration
        image: myapp-migrations:v2
        command: ["./migrate.sh"]
        env:
        - name: DB_HOST
          value: "postgres.default.svc.cluster.local"
      
      # Main containers
      containers:
      - name: webapp
        image: myapp:v2
        imagePullPolicy: IfNotPresent
        
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
        - containerPort: 9090
          name: metrics
        
        env:
        - name: ENV
          value: "production"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        
        resources:
          requests:
            memory: "256Mi"
            cpu: "500m"
          limits:
            memory: "512Mi"
            cpu: "1000m"
        
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          successThreshold: 1
          failureThreshold: 3
        
        volumeMounts:
        - name: config
          mountPath: /app/config
        - name: cache
          mountPath: /app/cache
      
      volumes:
      - name: config
        configMap:
          name: webapp-config
      - name: cache
        emptyDir:
          sizeLimit: 1Gi
      
      # Node selection
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: disktype
                operator: In
                values:
                - ssd
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - webapp
              topologyKey: kubernetes.io/hostname
      
      # Tolerations
      tolerations:
      - key: "app"
        operator: "Equal"
        value: "webapp"
        effect: "NoSchedule"
```

## 🎓 Best Practices

### **1. Always Use Deployments**

❌ **Bad**: Create bare Pods
```bash
kubectl run nginx --image=nginx
```

✅ **Good**: Create Deployments
```bash
kubectl create deployment nginx --image=nginx
```

### **2. Set Resource Limits**

```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "250m"
  limits:
    memory: "256Mi"
    cpu: "500m"
```

### **3. Use Health Checks**

```yaml
livenessProbe:   # Restart if unhealthy
  httpGet:
    path: /health
    port: 8080

readinessProbe:  # Remove from service if not ready
  httpGet:
    path: /ready
    port: 8080
```

### **4. Use Rolling Updates**

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0  # Zero downtime
```

### **5. Set Revision History**

```yaml
revisionHistoryLimit: 10  # Keep last 10 revisions
```

### **6. Use Labels Properly**

```yaml
labels:
  app: myapp
  version: v1
  environment: production
  team: backend
```

## 🚨 Troubleshooting

### **Pod Issues**

```bash
# Pod not starting
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl get events

# CrashLoopBackOff
kubectl logs <pod-name> --previous

# ImagePullBackOff
kubectl describe pod <pod-name>  # Check image name

# Pending
kubectl describe pod <pod-name>  # Check resources, node selector
```

### **Deployment Issues**

```bash
# Deployment not ready
kubectl get deployment
kubectl describe deployment <name>

# Rollout stuck
kubectl rollout status deployment/<name>
kubectl get rs
kubectl get pods

# Too many old ReplicaSets
kubectl get rs
kubectl delete rs <old-rs-name>
```

## ✅ Quick Reference

```bash
# Pods
kubectl run nginx --image=nginx
kubectl get pods
kubectl describe pod <name>
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- bash
kubectl delete pod <name>

# Deployments
kubectl create deployment nginx --image=nginx
kubectl get deployments
kubectl scale deployment <name> --replicas=3
kubectl set image deployment/<name> nginx=nginx:1.22
kubectl rollout status deployment/<name>
kubectl rollout undo deployment/<name>
kubectl delete deployment <name>
```

---

**Tiếp theo**: [Kubernetes Networking →](../networking/services-ingress.md)
