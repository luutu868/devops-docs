# Kubernetes Networking: Services & Ingress

## Kubernetes Networking Overview

Kubernetes networking giải quyết 4 vấn đề chính:
1. **Container-to-Container**: Trong Pod (localhost)
2. **Pod-to-Pod**: Across nodes (flat network)
3. **Pod-to-Service**: Load balancing
4. **External-to-Service**: Ingress/LoadBalancer

```
┌─────────────────────────────────────────────────────────┐
│                   External Traffic                      │
│                          ↓                              │
│                    ┌──────────┐                         │
│                    │ Ingress  │                         │
│                    └────┬─────┘                         │
│                         ↓                               │
│         ┌───────────────┴────────────────┐              │
│         ↓                                ↓              │
│    ┌─────────┐                      ┌─────────┐        │
│    │Service A│                      │Service B│        │
│    └────┬────┘                      └────┬────┘        │
│         ↓                                ↓              │
│    ┌────┴────┐                      ┌────┴────┐        │
│    │  Pod    │                      │  Pod    │        │
│    └─────────┘                      └─────────┘        │
└─────────────────────────────────────────────────────────┘
```

## Services

### **Service Là Gì?**

**Service** là abstraction layer cung cấp stable network endpoint cho set of Pods.

**Vấn đề:**
- Pods có IP động (ephemeral)
- Pods có thể restart/scale → IP thay đổi
- Làm sao để connect to Pods reliably?

**Giải pháp:**
- Service provides **stable IP** and **DNS name**
- Load balances traffic across Pods
- Selects Pods using **labels**

### **Service Types**

| Type | Description | Use Case |
|------|-------------|----------|
| **ClusterIP** | Internal IP (default) | Internal communication |
| **NodePort** | Expose on Node port | Development/Testing |
| **LoadBalancer** | Cloud LB (AWS ELB, GCP LB) | Production external access|
| **ExternalName** | DNS CNAME mapping | External service proxy |

### **1. ClusterIP (Default)**

Expose Service on **internal cluster IP**. Only accessible within cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP  # Default, có thể bỏ
  selector:
    app: backend  # Select Pods with this label
  ports:
  - protocol: TCP
    port: 80        # Service port
    targetPort: 8080  # Pod port
```

```bash
# Create Service
kubectl apply -f service.yaml

# Get Services
kubectl get services
kubectl get svc

# Describe Service
kubectl describe svc backend-service

# Test from inside cluster
kubectl run test --image=busybox -it --rm -- wget -O- backend-service
```

**How it works:**
```
┌──────────────────────────────────┐
│  ClusterIP: 10.96.0.10:80       │
│       (stable endpoint)          │
└───────────┬──────────────────────┘
            │ Load balance
    ┌───────┴────────┬────────┐
    ↓                ↓        ↓
┌────────┐      ┌────────┐ ┌────────┐
│ Pod:80 │      │ Pod:80 │ │ Pod:80 │
└────────┘      └────────┘ └────────┘
```

### **2. NodePort**

Expose Service on **each Node's IP** at a static port (30000-32767).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-nodeport
spec:
  type: NodePort
  selector:
    app: webapp
  ports:
  - protocol: TCP
    port: 80          # Service port
    targetPort: 8080  # Pod port
    nodePort: 30080   # Node port (optional, auto-assigned if omitted)
```

**Access:**
```bash
# From outside cluster
curl http://<NodeIP>:30080

# Get NodePort
kubectl get svc webapp-nodeport
```

**How it works:**
```
External Request
     ↓
┌────────────────────────┐
│   Node IP:30080        │
└───────┬────────────────┘
        ↓
┌────────────────────────┐
│   ClusterIP:80         │
└───────┬────────────────┘
        ↓
    Load balance
    ┌───┴───┬───────┐
    ↓       ↓       ↓
  Pod:80  Pod:80  Pod:80
```

**Pros & Cons:**

**Pros:**
- Simple to setup
- Good for development

**Cons:**
- Need to know Node IPs
- Port range limited (30000-32767)
- Not suitable for production
- No load balancing across Nodes

### **3. LoadBalancer**

Create **external load balancer** (AWS ELB, GCP LB, Azure LB).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-lb
spec:
  type: LoadBalancer
  selector:
    app: webapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```

```bash
# Get external IP
kubectl get svc webapp-lb

# Output:
# NAME        TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)
# webapp-lb   LoadBalancer   10.96.0.100     35.184.23.45     80:31234/TCP
```

**How it works:**
```
Internet
    ↓
┌────────────────────────┐
│  Cloud Load Balancer   │
│  (AWS ELB/NLB)         │
└───────┬────────────────┘
        ↓
┌────────────────────────┐
│   NodePort :31234      │
└───────┬────────────────┘
        ↓
┌────────────────────────┐
│   ClusterIP:80         │
└───────┬────────────────┘
        ↓ Load balance
    ┌───┴───┬───────┐
    ↓       ↓       ↓
  Pod:80  Pod:80  Pod:80
```

**Pros & Cons:**

**Pros:**
- Full load balancing
- Automatic DNS
- Production-ready

**Cons:**
- Costs money (cloud LB)
- Each Service = 1 load balancer (expensive)
- Use Ingress instead for HTTP(S)

### **4. ExternalName**

Map Service to **external DNS name**.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: database.example.com
```

```bash
# Pods can now connect to "external-db" instead of "database.example.com"
kubectl run test --image=busybox -it --rm -- nslookup external-db
```

### **Service Discovery**

#### **1. Environment Variables**

Kubernetes automatically creates env vars for Services:

```bash
# If you have service named "backend-service"
BACKEND_SERVICE_SERVICE_HOST=10.96.0.10
BACKEND_SERVICE_SERVICE_PORT=80
```

```yaml
containers:
- name: frontend
  image: frontend:1.0
  env:
  - name: BACKEND_URL
    value: "http://backend-service:80"
```

#### **2. DNS (Recommended)**

Kubernetes DNS (CoreDNS) provides automatic DNS records:

```bash
# Same namespace
http://backend-service

# Different namespace
http://backend-service.production.svc.cluster.local

# Format: <service-name>.<namespace>.svc.cluster.local
```

```yaml
containers:
- name: frontend
  image: frontend:1.0
  env:
  - name: BACKEND_URL
    value: "http://backend-service"  # DNS resolution
```

### **Advanced Service Features**

#### **Session Affinity**

```yaml
spec:
  sessionAffinity: ClientIP  # Stick to same Pod
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600
```

#### **Headless Service**

Direct access to Pod IPs (no load balancing):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cassandra
spec:
  clusterIP: None  # Headless
  selector:
    app: cassandra
  ports:
  - port: 9042
```

**Use cases:**
- StatefulSets
- Databases (need direct Pod access)
- Service meshes

#### **Multi-Port Services**

```yaml
spec:
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 8080
  - name: https
    protocol: TCP
    port: 443
    targetPort: 8443
  - name: metrics
    protocol: TCP
    port: 9090
    targetPort: 9090
```

#### **Named Ports**

```yaml
# In Pod/Deployment
spec:
  containers:
  - name: webapp
    ports:
    - name: http
      containerPort: 8080
    - name: metrics
      containerPort: 9090

---
# In Service
spec:
  ports:
  - name: http
    port: 80
    targetPort: http  # Reference by name
  - name: metrics
    port: 9090
    targetPort: metrics
```

### **Service Endpoints**

Service automatically creates Endpoints for matching Pods:

```bash
# Check Endpoints
kubectl get endpoints backend-service

# Describe Endpoints
kubectl describe endpoints backend-service

# Output:
# Subsets:
#   Addresses:    10.244.1.5,10.244.2.7,10.244.3.8
#   Ports:        8080/TCP
```

### **Service Without Selector**

Manual Endpoints (for external services):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-api
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
---
apiVersion: v1
kind: Endpoints
metadata:
  name: external-api  # Same name as Service
subsets:
- addresses:
  - ip: 203.0.113.5  # External IP
  ports:
  - port: 8080
```

## 🌍 Ingress

### **Ingress Là Gì?**

**Ingress** manages **external HTTP(S) access** to Services. Think of it as "Layer 7 Load Balancer".

**Problem with Services:**
- LoadBalancer Service = 1 cloud LB (expensive!)
- Need many LBs for many apps

**Solution:**
- 1 Ingress = 1 LB for **many Services**
- HTTP(S) routing based on **host/path**
- SSL/TLS termination
- Name-based virtual hosting

### **Ingress vs Service LoadBalancer**

**Without Ingress:**
```
app1.com → LoadBalancer 1 → Service A
app2.com → LoadBalancer 2 → Service B
app3.com → LoadBalancer 3 → Service C
(3 cloud load balancers = $$$$)
```

**With Ingress:**
```
                   ┌─→ Service A
app1.com ─┐        │
app2.com ─┼─→ Ingress ┼─→ Service B
app3.com ─┘        │
                   └─→ Service C
(1 cloud load balancer = $)
```

### **Ingress Components**

1. **Ingress Resource**: Rules for routing
2. **Ingress Controller**: Implements rules (NGINX, Traefik, HAProxy)

### **Popular Ingress Controllers**

| Controller | Description | Best For |
|-----------|-------------|----------|
| **NGINX Ingress** | Most popular | General purpose |
| **Traefik** | Cloud-native, easy config | Microservices |
| **HAProxy** | High performance | Enterprise |
| **Kong** | API Gateway features | API management |
| **Istio** | Service mesh | Advanced traffic management |
| **AWS ALB** | AWS Application LB | AWS EKS |
| **GCE** | GCP HTTP(S) LB | GCP GKE |

### **Install NGINX Ingress Controller**

```bash
# Using Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx

# Using kubectl
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml

# Check installation
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

### **Basic Ingress**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp-service
            port:
              number: 80
```

```bash
# Create Ingress
kubectl apply -f ingress.yaml

# Get Ingress
kubectl get ingress

# Describe Ingress
kubectl describe ingress webapp-ingress

# Test (add to /etc/hosts first)
curl http://myapp.example.com
```

### **Path-Based Routing**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-ingress
spec:
  rules:
  - host: mycompany.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
      
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
      
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 8081
```

**Routing:**
```
mycompany.com/api/*    → api-service:8080
mycompany.com/web/*    → web-service:80
mycompany.com/admin/*  → admin-service:8081
```

### **Host-Based Routing**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-ingress
spec:
  rules:
  # App 1
  - host: app1.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
  
  # App 2
  - host: app2.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
  
  # App 3
  - host: app3.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app3-service
            port:
              number: 80
```

### **HTTPS/TLS Ingress**

#### **1. Create TLS Secret**

```bash
# Self-signed cert (development)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=myapp.example.com/O=myapp"

# Create Secret
kubectl create secret tls myapp-tls \
  --cert=tls.crt \
  --key=tls.key
```

#### **2. Use TLS in Ingress**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: https-ingress
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls  # TLS Secret name
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp-service
            port:
              number: 80
```

**Multiple domains:**

```yaml
spec:
  tls:
  - hosts:
    - app1.example.com
    secretName: app1-tls
  - hosts:
    - app2.example.com
    secretName: app2-tls
  rules:
  - host: app1.example.com
    # ...
  - host: app2.example.com
    # ...
```

### **Ingress Annotations**

Annotations control Ingress Controller behavior:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: annotated-ingress
  annotations:
    # Rewrite paths
    nginx.ingress.kubernetes.io/rewrite-target: /
    
    # SSL redirect
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "10"
    
    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
    
    # Basic Auth
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    
    # Whitelist IPs
    nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8,192.168.0.0/16"
    
    # Timeout
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    
    # Client body size
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    
    # Custom headers
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Custom-Header: value";
spec:
  # ...
```

### **Path Rewriting**

```yaml
# Original: myapp.com/api/v1/users
# Backend:  api-service/users

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /api/v1(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

### **Default Backend**

Fallback for non-matching requests (404 handler):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-with-default
spec:
  defaultBackend:
    service:
      name: default-backend
      port:
        number: 80
  rules:
  - host: myapp.com
    # ...
```

### **Complete Ingress Example**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: production-ingress
  namespace: production
  annotations:
    # SSL
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "50"
    
    # Timeouts
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    
    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-credentials: "true"
    
    # Security
    nginx.ingress.kubernetes.io/force-ssl-redirect "true"
    nginx.ingress.kubernetes.io/hsts: "true"
    nginx.ingress.kubernetes.io/hsts-max-age: "31536000"
    
spec:
  ingressClassName: nginx
  
  tls:
  - hosts:
    - myapp.com
    - www.myapp.com
    secretName: myapp-tls
  
  rules:
  # Main domain
  - host: myapp.com
    http:
      paths:
      # Frontend
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
      
      # API
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
      
      # Admin
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 8081
      
      # Metrics (internal only)
      - path: /metrics
        pathType: Prefix
        backend:
          service:
            name: metrics-service
            port:
              number: 9090
  
  # WWW redirect
  - host: www.myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

## Network Policies

Control traffic flow between Pods (firewall rules).

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow-frontend
spec:
  podSelector:
    matchLabels:
      app: api  # Apply to API pods
  
  policyTypes:
  - Ingress
  - Egress
  
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend  # Only allow from frontend
    ports:
    - protocol: TCP
      port: 8080
  
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database  # Only allow to database
    ports:
    - protocol: TCP
      port: 5432
```

## 🎓 Best Practices

### **1. Use Ingress for HTTP(S)**

**Bad**: Multiple LoadBalancers
```yaml
# Expensive! Each app = 1 cloud LB
type: LoadBalancer  # App 1
type: LoadBalancer  # App 2
type: LoadBalancer  # App 3
```

**Good**: One Ingress for all apps
```yaml
# 1 cloud LB for all apps
kind: Ingress
```

### **2. Always Use TLS**

```yaml
spec:
  tls:
  - hosts:
    - myapp.com
    secretName: myapp-tls
```

### **3. Use DNS Names**

```yaml
# In application config
BACKEND_URL: http://backend-service  # Not 10.96.0.10
```

### **4. Set Resource Limits on Services**

```yaml
# In Deployment
resources:
  limits:
    memory: "512Mi"
    cpu: "1000m"
```

### **5. Use Health Checks**

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
```

## 🚨 Troubleshooting

### **Service Issues**

```bash
# Service not accessible
kubectl get svc
kubectl describe svc <name>
kubectl get endpoints <name>  # Check if Pods selected

# No endpoints
kubectl get pods --show-labels
# Check if Pod labels match Service selector

# DNS not working
kubectl run test --image=busybox -it --rm -- nslookup backend-service
kubectl get svc -n kube-system  # Check CoreDNS
```

### **Ingress Issues**

```bash
# Ingress not working
kubectl get ingress
kubectl describe ingress <name>

# Check Ingress Controller
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx <ingress-controller-pod>

# Get Ingress Controller service
kubectl get svc -n ingress-nginx

# Check Ingress rules
kubectl get ingress <name> -o yaml

# Test DNS
nslookup myapp.example.com

# Check TLS
openssl s_client -connect myapp.example.com:443 -servername myapp.example.com
```

## Quick Reference

```bash
# Services
kubectl expose deployment nginx --port=80 --target-port=8080
kubectl get svc
kubectl describe svc <name>
kubectl get endpoints <name>

# Ingress
kubectl get ingress
kubectl describe ingress <name>
kubectl get ingress <name> -o yaml

# Network Policies
kubectl get networkpolicies
kubectl describe netpol <name>

# DNS
kubectl run test --image=busybox -it --rm -- nslookup <service-name>
```

---

**Tiếp theo**: [Kubernetes Storage →](../storage/volumes-pv-pvc.md)
