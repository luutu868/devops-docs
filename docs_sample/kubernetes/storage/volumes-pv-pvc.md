# Kubernetes Storage: Volumes, PV & PVC

## 💾 Kubernetes Storage Overview

Container filesystems là **ephemeral** (tạm thời). Khi container restart → **data bị mất**.

**Problems:**
- ❌ Data loss khi container restart
- ❌ Không share data giữa containers trong Pod
- ❌ Persistent storage cho databases

**Solutions:**
- ✅ **Volumes**: Share data trong Pod
- ✅ **PersistentVolumes (PV)**: Cluster storage
- ✅ **PersistentVolumeClaims (PVC)**: Request storage

```
┌────────────────────────────────────┐
│           Pod                      │
│  ┌──────────────────────────────┐  │
│  │  Container 1                 │  │
│  │  /data ───┐                  │  │
│  └───────────┼──────────────────┘  │
│              │                     │
│  ┌───────────▼──────────────────┐  │
│  │      Volume (emptyDir)       │  │
│  └───────────┬──────────────────┘  │
│              │                     │
│  ┌───────────▼──────────────────┐  │
│  │  Container 2                 │  │
│  │  /data                       │  │
│  └──────────────────────────────┘  │
└────────────────────────────────────┘
```

## 📁 Volumes

### **Volume Types**

| Type | Description | Use Case |
|------|-------------|----------|
| **emptyDir** | Temporary, Pod lifetime | Cache, scratch space |
| **hostPath** | Mount from Node | Development, system access |
| **configMap** | Config files | Application configuration |
| **secret** | Sensitive data | Passwords, certificates |
| **persistentVolumeClaim** | Request PV | Database data |
| **nfs** | Network File System | Shared storage |
| **awsElasticBlockStore** | AWS EBS | AWS persistent storage |
| **gcePersistentDisk** | GCP Persistent Disk | GCP persistent storage |
| **azureDisk** | Azure Disk | Azure persistent storage |

### **1. emptyDir**

Empty directory tạo khi Pod start, **deleted khi Pod dies**.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-volume-pod
spec:
  containers:
  # Writer container
  - name: writer
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true; do date >> /data/log.txt; sleep 5; done"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  
  # Reader container
  - name: reader
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "tail -f /data/log.txt"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  
  volumes:
  - name: shared-data
    emptyDir: {}  # Default: use Node disk
```

**emptyDir with Memory:**

```yaml
volumes:
- name: cache
  emptyDir:
    medium: Memory  # Use RAM instead of disk
    sizeLimit: 1Gi
```

**Use cases:**
- Cache data
- Scratch space
- Share data between containers trong Pod

### **2. hostPath**

Mount directory from **Node filesystem**.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: host-data
      mountPath: /usr/share/nginx/html
  
  volumes:
  - name: host-data
    hostPath:
      path: /data/website  # Path on Node
      type: DirectoryOrCreate
```

**hostPath types:**

| Type | Description |
|------|-------------|
| `DirectoryOrCreate` | Create if not exists |
| `Directory` | Must exist |
| `File` | Must be file |
| `FileOrCreate` | Create file if not exists |

**⚠️ Security Risk:**
- Pod có thể access Node filesystem
- Chỉ dùng cho development
- **KHÔNG dùng trong production**

### **3. ConfigMap Volume**

Mount ConfigMap as files.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  config.yaml: |
    database:
      host: db.example.com
      port: 5432
    cache:
      enabled: true
  app.properties: |
    app.name=MyApp
    app.version=1.0
---
apiVersion: v1
kind: Pod
metadata:
  name: configmap-pod
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: config
      mountPath: /etc/config
      readOnly: true
  
  volumes:
  - name: config
    configMap:
      name: app-config
```

**Result:**
```
/etc/config/
├── config.yaml
└── app.properties
```

**Mount specific keys:**

```yaml
volumes:
- name: config
  configMap:
    name: app-config
    items:
    - key: config.yaml
      path: config.yaml  # Only mount this file
```

### **4. Secret Volume**

Mount Secrets as files (base64 decoded).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4=  # base64: admin
  password: cGFzc3dvcmQ=  # base64: password
---
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: secrets
      mountPath: /etc/secrets
      readOnly: true
  
  volumes:
  - name: secrets
    secret:
      secretName: db-credentials
```

**Result:**
```bash
$ cat /etc/secrets/username
admin

$ cat /etc/secrets/password
password
```

**Set permissions:**

```yaml
volumes:
- name: secrets
  secret:
    secretName: db-credentials
    defaultMode: 0400  # Read-only for owner
```

### **5. PersistentVolumeClaim**

Request storage from PersistentVolume (see below).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: data
      mountPath: /data
  
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc  # Reference PVC
```

## 🗄️ PersistentVolumes (PV)

**PersistentVolume** là **cluster resource** representing physical storage.

### **PV Lifecycle**

```
┌──────────────┐
│  Available   │  Chưa được claim
└──────┬───────┘
       ↓ (Bound by PVC)
┌──────────────┐
│    Bound     │  Đang được sử dụng
└──────┬───────┘
       ↓ (PVC deleted)
┌──────────────┐
│   Released   │  PVC deleted, data vẫn còn
└──────┬───────┘
       ↓ (Reclaim policy)
┌──────────────┐      ┌──────────────┐
│   Available  │  or  │   Failed     │
└──────────────┘      └──────────────┘
```

### **Basic PV**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-1
spec:
  capacity:
    storage: 10Gi
  
  accessModes:
  - ReadWriteOnce  # RWO
  
  persistentVolumeReclaimPolicy: Retain
  
  storageClassName: standard
  
  hostPath:
    path: /mnt/data  # For development only
```

### **Access Modes**

| Mode | Abbreviation | Description |
|------|--------------|-------------|
| **ReadWriteOnce** | RWO | Single Node, read-write |
| **ReadOnlyMany** | ROX | Multiple Nodes, read-only |
| **ReadWriteMany** | RWX | Multiple Nodes, read-write |
| **ReadWriteOncePod** | RWOP | Single Pod, read-write |

**Support Matrix:**

| Storage Type | RWO | ROX | RWX |
|--------------|-----|-----|-----|
| AWS EBS | ✅ | ❌ | ❌ |
| GCP PD | ✅ | ✅ | ❌ |
| Azure Disk | ✅ | ❌ | ❌ |
| NFS | ✅ | ✅ | ✅ |
| CephFS | ✅ | ✅ | ✅ |

### **Reclaim Policies**

| Policy | Description |
|--------|-------------|
| **Retain** | Keep data after PVC deletion (manual cleanup) |
| **Delete** | Delete PV and underlying storage automatically |
| **Recycle** | Wipe data (`rm -rf`), make available again (deprecated) |

### **Storage Classes**

Different storage types with different performance:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: fast-pv
spec:
  storageClassName: fast-ssd
  capacity:
    storage: 50Gi
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /mnt/ssd
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: slow-pv
spec:
  storageClassName: slow-hdd
  capacity:
    storage: 100Gi
  accessModes:
  - ReadWriteMany
  hostPath:
    path: /mnt/hdd
```

### **Cloud Provider PV Examples**

#### **AWS EBS**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: aws-ebs-pv
spec:
  capacity:
    storage: 20Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: gp2
  awsElasticBlockStore:
    volumeID: vol-0123456789abcdef0  # AWS EBS volume ID
    fsType: ext4
```

#### **GCP Persistent Disk**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: gcp-pd-pv
spec:
  capacity:
    storage: 20Gi
  accessModes:
  - ReadWriteOnce
  storageClassName: standard
  gcePersistentDisk:
    pdName: my-disk  # GCP disk name
    fsType: ext4
```

#### **Azure Disk**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: azure-disk-pv
spec:
  capacity:
    storage: 20Gi
  accessModes:
  - ReadWriteOnce
  storageClassName: managed-premium
  azureDisk:
    diskName: myDisk
    diskURI: /subscriptions/.../resourceGroups/.../providers/Microsoft.Compute/disks/myDisk
    kind: Managed
```

#### **NFS**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
  - ReadWriteMany  # NFS supports RWX
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: nfs-server.example.com
    path: /exports/data
```

## 📝 PersistentVolumeClaims (PVC)

**PVC** là **request for storage** by user.

### **Basic PVC**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
```

### **PV & PVC Binding**

Kubernetes **automatically binds** PVC to matching PV:

**Matching criteria:**
1. Storage class same
2. Access mode compatible
3. Size sufficient

```yaml
# PV with 10Gi
---
# PVC requests 5Gi
# → Kubernetes binds PVC to this PV
# → PVC gets 10Gi (not 5Gi, uses entire PV)
```

### **Using PVC in Pod**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard
---
apiVersion: v1
kind: Pod
metadata:
  name: mysql-pod
spec:
  containers:
  - name: mysql
    image: mysql:8.0
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: password
    volumeMounts:
    - name: mysql-data
      mountPath: /var/lib/mysql
  
  volumes:
  - name: mysql-data
    persistentVolumeClaim:
      claimName: mysql-pvc
```

### **PVC in StatefulSet**

StatefulSets use **VolumeClaimTemplates** to automatically create PVCs:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 10Gi
```

**Result:**
```
mysql-0 → data-mysql-0 (PVC) → pv-1
mysql-1 → data-mysql-1 (PVC) → pv-2
mysql-2 → data-mysql-2 (PVC) → pv-3
```

## 🎯 StorageClass

**StorageClass** provides **dynamic provisioning** of PVs.

### **Without StorageClass (Static Provisioning)**

```
Admin → Manually create PV → User create PVC → Bind
```

### **With StorageClass (Dynamic Provisioning)**

```
User create PVC → StorageClass automatically creates PV → Bind
```

### **Basic StorageClass**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs  # Cloud provider provisioner
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### **Common Provisioners**

| Provider | Provisioner |
|----------|-------------|
| AWS EBS | `kubernetes.io/aws-ebs` |
| GCP PD | `kubernetes.io/gce-pd` |
| Azure Disk | `kubernetes.io/azure-disk` |
| Azure File | `kubernetes.io/azure-file` |
| NFS | `nfs.csi.k8s.io` |
| Local | `kubernetes.io/no-provisioner` |

### **AWS EBS StorageClass**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: aws-fast
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3         # gp2, gp3, io1, io2
  iopsPerGB: "10"
  fsType: ext4
  encrypted: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true
```

### **GCP Persistent Disk StorageClass**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gcp-fast
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd  # pd-standard, pd-ssd
  replication-type: regional-pd
reclaimPolicy: Delete
allowVolumeExpansion: true
```

### **Azure StorageClass**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-fast
provisioner: kubernetes.io/azure-disk
parameters:
  storageaccounttype: Premium_LRS  # Standard_LRS, Premium_LRS
  kind: Managed
reclaimPolicy: Delete
allowVolumeExpansion: true
```

### **Local StorageClass**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer  # Important for local
```

### **Dynamic Provisioning Example**

```yaml
# 1. Create StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
---
# 2. Create PVC (automatically creates PV)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: fast  # Reference StorageClass
  resources:
    requests:
      storage: 10Gi
# Kubernetes automatically creates PV with 10Gi AWS EBS gp3 volume
---
# 3. Use in Pod
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: dynamic-pvc
```

### **Default StorageClass**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
```

**Usage:**
```yaml
# PVC without storageClassName uses default
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  # storageClassName: omitted → uses default
```

### **Volume Expansion**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: expandable
provisioner: kubernetes.io/aws-ebs
allowVolumeExpansion: true  # Enable expansion
```

**Expand volume:**

```bash
# Edit PVC, increase size
kubectl edit pvc my-pvc

# Change: storage: 10Gi → storage: 20Gi
# PV automatically expands (for supported storage types)
```

## 📊 Storage Comparison

### **Static vs Dynamic Provisioning**

| Aspect | Static | Dynamic |
|--------|--------|---------|
| **PV Creation** | Manual by admin | Automatic by StorageClass |
| **Flexibility** | Limited | High |
| **Use Case** | Fixed storage | On-demand storage |
| **Cost** | Pre-allocated | Pay as you use |

### **Volume Types Comparison**

| Type | Persistent | Shared | Use Case |
|------|-----------|--------|----------|
| **emptyDir** | ❌ | Within Pod | Cache, scratch |
| **hostPath** | ⚠️ | No | Development only |
| **ConfigMap/Secret** | ✅ | Read-only | Configuration |
| **PVC** | ✅ | Depends on accessMode | Production data |
| **NFS** | ✅ | Yes (RWX) | Shared files |

## 🎓 Best Practices

### **1. Always Use PVC in Production**

❌ **Bad**: hostPath, emptyDir for important data
```yaml
volumes:
- name: data
  hostPath:
    path: /data  # Lost if Pod moves to different Node
```

✅ **Good**: PVC with reliable storage
```yaml
volumes:
- name: data
  persistentVolumeClaim:
    claimName: my-pvc
```

### **2. Use Dynamic Provisioning**

✅ **Create StorageClasses for different tiers:**
```yaml
# Fast for databases
fast-ssd: 3000 IOPS, gp3

# Standard for apps
standard: gp2

# Slow for backups
slow-hdd: st1
```

### **3. Set Resource Requests**

```yaml
resources:
  requests:
    storage: 10Gi  # Request what you need
```

### **4. Use Appropriate Access Mode**

```yaml
# Single Pod/Node: RWO
accessModes: [ReadWriteOnce]

# Shared read: ROX
accessModes: [ReadOnlyMany]

# Shared read-write (expensive): RWX
accessModes: [ReadWriteMany]
```

### **5. Set Reclaim Policy Carefully**

```yaml
# Production data: Retain
reclaimPolicy: Retain

# Development: Delete
reclaimPolicy: Delete
```

### **6. Enable Volume Expansion**

```yaml
allowVolumeExpansion: true  # Allow growing storage
```

### **7. Use StatefulSets for Databases**

```yaml
# Automatic PVC management
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    # ...
```

## 🚨 Troubleshooting

### **PVC Pending**

```bash
# Check PVC status
kubectl get pvc

# Describe PVC
kubectl describe pvc my-pvc

# Common issues:
# - No matching PV available
# - StorageClass not found
# - Insufficient capacity
```

**Solutions:**

```bash
# Check available PVs
kubectl get pv

# Check StorageClass
kubectl get storageclass

# Create matching PV
kubectl apply -f pv.yaml
```

### **PV Not Binding**

```bash
# Check PV details
kubectl describe pv my-pv

# Check:
# - Access mode matches
# - Storage size sufficient
# - StorageClass matches
# - PV status is Available
```

### **Volume Mount Permission Denied**

```yaml
# Set security context
spec:
  securityContext:
    fsGroup: 1000  # Volume group ownership
  containers:
  - name: app
    securityContext:
      runAsUser: 1000
      runAsGroup: 1000
```

### **Data Not Persisting**

```bash
# Check volume type
kubectl get pod <name> -o yaml | grep -A 10 volumes:

# emptyDir does NOT persist after Pod deletion
# Use PVC for persistent storage
```

## ✅ Quick Reference

```bash
# PV
kubectl get pv
kubectl describe pv <name>
kubectl delete pv <name>

# PVC
kubectl get pvc
kubectl describe pvc <name>
kubectl delete pvc <name>

# StorageClass
kubectl get storageclass
kubectl get sc
kubectl describe sc <name>

# Check volume usage
kubectl exec <pod> -- df -h

# Expand PVC
kubectl edit pvc <name>  # Increase storage size
```

---

**Tiếp theo**: [Helm & Operators →](../advanced/helm-operators.md)
