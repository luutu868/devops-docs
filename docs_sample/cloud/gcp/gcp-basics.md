# GCP Basics - Google Cloud Platform

## Giới Thiệu GCP

Google Cloud Platform (GCP) là bộ dịch vụ điện toán đám mây của Google, tận dụng cùng infrastructure mà Google sử dụng cho sản phẩm của mình như Google Search, Gmail, YouTube.

**Ưu điểm GCP:**
```yaml
- Google's infrastructure: Same infrastructure as Google products
- Performance: Global fiber network, low latency
- Big Data & ML: Leading AI/ML services (TensorFlow, BigQuery)
- Kubernetes: GKE - Best managed Kubernetes service
- Pricing: Per-second billing, sustained use discounts
- Open source: Strong commitment to open source technologies
```

## GCP Global Infrastructure

### Regions và Zones

```
Region (us-central1)
├── Zone A (us-central1-a)
│   └── Data Centers
├── Zone B (us-central1-b)
│   └── Data Centers
├── Zone C (us-central1-c)
│   └── Data Centers
└── Zone F (us-central1-f)
    └── Data Centers
```

**Regions:**
- 38+ regions globally
- Multiple zones per region (3-4 zones typically)
- Low latency within region (<5ms between zones)

**Zones:**
- Isolated locations within region
- Independent power, cooling, networking
- Deploy across zones for high availability

**Network:**
- Private global fiber network
- Premium tier: Google's network
- Standard tier: Public internet

### Region Selection

```yaml
Factors:
  Latency:
    - Geographic proximity to users
    - Premium tier for best performance
  
  Cost:
    - Iowa (us-central1) typically cheapest
    - Asia/Australia more expensive
    - Network egress costs vary
  
  Compliance:
    - Data residency requirements
    - GDPR (European regions)
    - Regional regulations
  
  Services:
    - Most services in all regions
    - Some ML/AI services limited
    - Check regional availability
```

## IAM - Identity and Access Management

### GCP IAM Concepts

**Identity:**
```
Google Account (user@gmail.com)
├── Used by individuals
└── Full access with proper permissions

Service Account (sa@project.iam.gserviceaccount.com)
├── Used by applications/VMs
├── Can be impersonated
└── JSON key file for authentication

Google Group (group@googlegroups.com)
├── Collection of users
└── Assign permissions to group

Cloud Identity Domain (user@company.com)
├── Organization's managed identities
└── G Suite / Cloud Identity
```

**Resource Hierarchy:**
```
Organization (company.com)
├── Folder: Production
│   ├── Project: web-app-prod
│   │   ├── Compute Engine VMs
│   │   ├── Cloud Storage buckets
│   │   └── Cloud SQL databases
│   └── Project: api-prod
└── Folder: Development
    ├── Project: web-app-dev
    └── Project: api-dev

Permissions inherit down the hierarchy
```

**Roles:**
```yaml
Primitive Roles (Not Recommended):
  Owner: Full access including billing
  Editor: Modify/delete resources
  Viewer: Read-only access

Predefined Roles (Recommended):
  compute.admin: Full Compute Engine access
  storage.objectViewer: Read GCS objects
  cloudsql.admin: Manage Cloud SQL
  container.developer: Deploy to GKE

Custom Roles:
  Define specific permissions
  Organization or project level
  Example: dataViewer with specific permissions
```

### IAM Policy

```json
{
  "bindings": [
    {
      "role": "roles/storage.objectViewer",
      "members": [
        "user:alice@example.com",
        "serviceAccount:my-sa@project.iam.gserviceaccount.com",
        "group:developers@example.com"
      ]
    },
    {
      "role": "roles/storage.admin",
      "members": [
        "user:admin@example.com"
      ],
      "condition": {
        "title": "Restrict to specific bucket",
        "expression": "resource.name.startsWith('projects/_/buckets/prod-')"
      }
    }
  ]
}
```

### IAM Best Practices

**1. Use Service Accounts:**
```bash
# Create service account
gcloud iam service-accounts create my-app-sa \
  --display-name="My Application Service Account"

# Grant role to service account
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:my-app-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"

# Create key (avoid if possible, use Workload Identity instead)
gcloud iam service-accounts keys create key.json \
  --iam-account=my-app-sa@PROJECT_ID.iam.gserviceaccount.com
```

**2. Least Privilege:**
```bash
# Bad - Too broad
roles/editor

# Good - Specific predefined role
roles/compute.instanceAdmin.v1

# Better - Custom role with exact permissions
gcloud iam roles create vmStarter \
  --project=PROJECT_ID \
  --title="VM Starter" \
  --description="Can only start VMs" \
  --permissions=compute.instances.start,compute.instances.get \
  --stage=GA
```

**3. Organization Policies:**
```yaml
Constraints:
  compute.restrictVpcPeering:
    - Restrict which VPCs can be peered
  
  compute.vmExternalIpAccess:
    - Deny external IPs on VMs
  
  storage.uniformBucketLevelAccess:
    - Enforce uniform bucket access
  
  iam.disableServiceAccountKeyCreation:
    - Prevent SA key creation
```

### gcloud CLI Commands

```bash
# Authenticate
gcloud auth login
gcloud auth application-default login

# Set project
gcloud config set project PROJECT_ID

# List projects
gcloud projects list

# Get IAM policy
gcloud projects get-iam-policy PROJECT_ID

# Add IAM binding
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="user:alice@example.com" \
  --role="roles/viewer"

# Remove IAM binding
gcloud projects remove-iam-policy-binding PROJECT_ID \
  --member="user:alice@example.com" \
  --role="roles/viewer"

# List service accounts
gcloud iam service-accounts list

# Impersonate service account
gcloud compute instances list \
  --impersonate-service-account=my-sa@PROJECT_ID.iam.gserviceaccount.com
```

## Compute Engine

### Machine Types

```yaml
General Purpose:
  E2: Cost-optimized, shared CPU
    - e2-micro, e2-small, e2-medium
    - e2-standard-2, e2-standard-4
  
  N2: Balanced, latest generation
    - n2-standard-2, n2-standard-4
    - n2-highmem-2, n2-highcpu-2
  
  N2D: AMD EPYC processors
    - n2d-standard-2, n2d-highmem-4

Compute Optimized:
  C2: High-performance computing
    - c2-standard-4, c2-standard-8
  
  C2D: AMD EPYC, cost-effective
    - c2d-standard-4, c2d-highcpu-8

Memory Optimized:
  M2: Ultra high memory (12 TB)
    - m2-ultramem-208, m2-megamem-416

Accelerator Optimized:
  A2: NVIDIA A100 GPUs
    - a2-highgpu-1g (1 GPU)
    - a2-highgpu-8g (8 GPUs)
```

### Custom Machine Types

```bash
# Create custom machine type
gcloud compute instances create my-vm \
  --custom-cpu=4 \
  --custom-memory=16GB \
  --zone=us-central1-a

# With extended memory (more than 6.5GB per vCPU)
gcloud compute instances create my-vm \
  --custom-cpu=4 \
  --custom-memory=32GB \
  --custom-extensions \
  --zone=us-central1-a
```

### Pricing Models

**1. On-Demand:**
```yaml
Pay as you go
No commitment
Per-second billing (minimum 1 minute)

Example:
  n1-standard-1: $0.0475/hour
  Per-second: $0.000013/second
```

**2. Committed Use Discounts:**
```yaml
1 or 3 year commitment
Up to 57% discount
Automatic application

Memory-optimized: Up to 70% discount
```

**3. Preemptible VMs:**
```yaml
Up to 80% discount
Can be terminated anytime
Maximum 24 hour runtime
No live migration

Use cases:
  - Batch processing
  - Fault-tolerant workloads
  - CI/CD runners
```

**4. Spot VMs:**
```yaml
Similar to Preemptible
No maximum runtime
Dynamic pricing
Can be terminated with 30-second warning
```

**5. Sustained Use Discounts:**
```yaml
Automatic discount
Based on monthly usage
Up to 30% discount
No commitment needed
```

### Create Compute Engine VM

```bash
# Create basic VM
gcloud compute instances create my-vm \
  --zone=us-central1-a \
  --machine-type=e2-medium \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud

# Create VM with startup script
gcloud compute instances create web-server \
  --zone=us-central1-a \
  --machine-type=e2-micro \
  --tags=http-server,https-server \
  --metadata=startup-script='#!/bin/bash
    apt-get update
    apt-get install -y nginx
    systemctl start nginx
    echo "Hello from $(hostname)" > /var/www/html/index.html'

# Create preemptible VM
gcloud compute instances create preempt-vm \
  --zone=us-central1-a \
  --machine-type=e2-medium \
  --preemptible

# Create spot VM
gcloud compute instances create spot-vm \
  --zone=us-central1-a \
  --machine-type=e2-medium \
  --provisioning-model=SPOT

# List instances
gcloud compute instances list

# SSH to instance
gcloud compute ssh my-vm --zone=us-central1-a

# Stop instance
gcloud compute instances stop my-vm --zone=us-central1-a

# Start instance
gcloud compute instances start my-vm --zone=us-central1-a

# Delete instance
gcloud compute instances delete my-vm --zone=us-central1-a
```

### Firewall Rules

```bash
# Create firewall rule to allow HTTP
gcloud compute firewall-rules create allow-http \
  --allow=tcp:80 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=http-server

# Allow HTTPS
gcloud compute firewall-rules create allow-https \
  --allow=tcp:443 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=https-server

# Allow SSH from specific IP
gcloud compute firewall-rules create allow-ssh \
  --allow=tcp:22 \
  --source-ranges=203.0.113.0/24

# List firewall rules
gcloud compute firewall-rules list

# Delete firewall rule
gcloud compute firewall-rules delete allow-http
```

## Cloud Storage (GCS)

### Storage Classes

```yaml
Standard:
  Use: Hot data, frequently accessed
  Availability: 99.95% (multi-region), 99.9% (region)
  Min Storage Duration: None
  Retrieval Cost: None

Nearline:
  Use: Infrequently accessed (once per month)
  Availability: 99.9% (multi-region), 99.0% (region)
  Min Storage Duration: 30 days
  Retrieval Cost: $0.01/GB

Coldline:
  Use: Rarely accessed (once per quarter)
  Availability: 99.9% (multi-region), 99.0% (region)
  Min Storage Duration: 90 days
  Retrieval Cost: $0.02/GB

Archive:
  Use: Long-term archive (once per year)
  Availability: 99.9% (multi-region), 99.0% (region)
  Min Storage Duration: 365 days
  Retrieval Cost: $0.05/GB
```

### Bucket Locations

```yaml
Multi-Region:
  - US, EU, ASIA
  - Highest availability
  - Higher cost
  - Use: Serving global users

Dual-Region:
  - 2 specific regions (e.g., us-east1 + us-west1)
  - Geo-redundancy
  - Balance cost & availability

Region:
  - Single region (e.g., us-central1)
  - Lowest latency
  - Lower cost
  - Use: Regional workloads
```

### gsutil Commands

```bash
# Create bucket
gsutil mb gs://my-unique-bucket-name
gsutil mb -l us-central1 gs://my-regional-bucket
gsutil mb -c NEARLINE gs://my-nearline-bucket

# Upload file
gsutil cp myfile.txt gs://my-bucket/

# Upload directory
gsutil -m cp -r mydir gs://my-bucket/

# Download file
gsutil cp gs://my-bucket/myfile.txt ./

# List buckets
gsutil ls

# List objects
gsutil ls gs://my-bucket/
gsutil ls -r gs://my-bucket/**

# Sync directory
gsutil rsync -r mydir gs://my-bucket/mydir

# Delete object
gsutil rm gs://my-bucket/myfile.txt

# Delete all objects
gsutil -m rm -r gs://my-bucket/**

# Make object public
gsutil acl ch -u AllUsers:R gs://my-bucket/myfile.txt

# Set bucket lifecycle
cat > lifecycle.json << 'JSON'
{
  "lifecycle": {
    "rule": [
      {
        "action": {"type": "SetStorageClass", "storageClass": "NEARLINE"},
        "condition": {"age": 30}
      },
      {
        "action": {"type": "Delete"},
        "condition": {"age": 365}
      }
    ]
  }
}
JSON

gsutil lifecycle set lifecycle.json gs://my-bucket

# Enable versioning
gsutil versioning set on gs://my-bucket

# Set CORS
cat > cors.json << 'JSON'
[
  {
    "origin": ["https://example.com"],
    "method": ["GET", "PUT", "POST"],
    "responseHeader": ["Content-Type"],
    "maxAgeSeconds": 3600
  }
]
JSON

gsutil cors set cors.json gs://my-bucket
```

### IAM for Cloud Storage

```bash
# Grant role to user
gsutil iam ch user:alice@example.com:objectViewer gs://my-bucket

# Grant role to service account
gsutil iam ch serviceAccount:my-sa@project.iam.gserviceaccount.com:objectAdmin gs://my-bucket

# Grant role to all users (public)
gsutil iam ch allUsers:objectViewer gs://my-bucket

# View IAM policy
gsutil iam get gs://my-bucket

# Remove role
gsutil iam ch -d user:alice@example.com:objectViewer gs://my-bucket
```

## Cloud SQL

### Available Databases

```yaml
MySQL:
  Versions: 5.7, 8.0
  Use: Web applications, general purpose
  HA: Regional only

PostgreSQL:
  Versions: 12, 13, 14, 15
  Use: Complex queries, ACID compliance
  HA: Regional and cross-region (preview)

SQL Server:
  Versions: 2017, 2019
  Use: .NET applications, Windows workloads
  HA: Regional only
```

### High Availability

```yaml
Regional HA:
  - Primary and standby in different zones
  - Synchronous replication
  - Automatic failover (<60 seconds)
  - Same region only
  - RPO: Zero data loss
  - RTO: ~60 seconds

Read Replicas:
  - Asynchronous replication
  - Read scaling
  - Can be in different regions
  - Manual promotion
  - Cross-region disaster recovery
```

### Create Cloud SQL Instance

```bash
# Create MySQL instance
gcloud sql instances create my-sql-instance \
  --database-version=MYSQL_8_0 \
  --tier=db-n1-standard-2 \
  --region=us-central1 \
  --root-password=MySecurePassword123! \
  --storage-type=SSD \
  --storage-size=20GB \
  --storage-auto-increase \
  --backup \
  --backup-start-time=03:00 \
  --enable-bin-log \
  --availability-type=REGIONAL

# Create PostgreSQL instance
gcloud sql instances create my-pg-instance \
  --database-version=POSTGRES_15 \
  --tier=db-custom-2-8192 \
  --region=us-central1 \
  --availability-type=REGIONAL

# List instances
gcloud sql instances list

# Create database
gcloud sql databases create mydb \
  --instance=my-sql-instance

# Create user
gcloud sql users create alice \
  --instance=my-sql-instance \
  --password=SecurePassword123!

# Connect to instance
gcloud sql connect my-sql-instance --user=root

# Create read replica
gcloud sql instances create my-sql-replica \
  --master-instance-name=my-sql-instance \
  --tier=db-n1-standard-1 \
  --region=us-east1

# Create backup
gcloud sql backups create \
  --instance=my-sql-instance \
  --description="Manual backup"

# List backups
gcloud sql backups list --instance=my-sql-instance

# Restore from backup
gcloud sql backups restore BACKUP_ID \
  --instance=my-sql-instance

# Delete instance
gcloud sql instances delete my-sql-instance
```

### Connection Methods

```yaml
Public IP:
  - Authorized networks
  - Add your IP to whitelist
  - Less secure
  
Private IP:
  - VPC peering
  - Private connection
  - More secure
  - Recommended

Cloud SQL Proxy:
  - Secure tunnel
  - No firewall rules needed
  - IAM authentication
  - Best for development
  
Example:
  cloud_sql_proxy -instances=PROJECT:REGION:INSTANCE=tcp:3306
  mysql -h 127.0.0.1 -u root -p
```

## GKE - Google Kubernetes Engine

### GKE Cluster Types

```yaml
Standard Cluster:
  - Manual node management
  - More control
  - Node pools with different machine types
  - Fine-grained configuration

Autopilot Cluster:
  - Fully managed nodes
  - Automatic scaling
  - Per-pod billing
  - Opinionated configuration
  - Recommended for most users
```

### Create GKE Cluster

```bash
# Create autopilot cluster (recommended)
gcloud container clusters create-auto my-autopilot-cluster \
  --region=us-central1

# Create standard cluster
gcloud container clusters create my-cluster \
  --zone=us-central1-a \
  --machine-type=e2-medium \
  --num-nodes=3 \
  --disk-size=20GB \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=5 \
  --enable-autorepair \
  --enable-autoupgrade \
  --release-channel=regular

# Regional cluster (multi-zonal)
gcloud container clusters create my-regional-cluster \
  --region=us-central1 \
  --num-nodes=1 \
  --machine-type=e2-small

# With preemptible nodes
gcloud container clusters create my-cluster \
  --zone=us-central1-a \
  --machine-type=e2-medium \
  --num-nodes=3 \
  --preemptible

# Get credentials
gcloud container clusters get-credentials my-cluster \
  --zone=us-central1-a

# List clusters
gcloud container clusters list

# Resize cluster
gcloud container clusters resize my-cluster \
  --num-nodes=5 \
  --zone=us-central1-a

# Upgrade cluster
gcloud container clusters upgrade my-cluster \
  --master \
  --zone=us-central1-a

# Delete cluster
gcloud container clusters delete my-cluster \
  --zone=us-central1-a
```

### Workload Identity

```bash
# Enable Workload Identity on cluster
gcloud container clusters update my-cluster \
  --workload-pool=PROJECT_ID.svc.id.goog \
  --zone=us-central1-a

# Create namespace
kubectl create namespace my-app

# Create Kubernetes service account
kubectl create serviceaccount my-ksa -n my-app

# Create Google service account
gcloud iam service-accounts create my-gsa

# Grant IAM role to GSA
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:my-gsa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"

# Bind KSA to GSA
gcloud iam service-accounts add-iam-policy-binding \
  my-gsa@PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="serviceAccount:PROJECT_ID.svc.id.goog[my-app/my-ksa]"

# Annotate KSA
kubectl annotate serviceaccount my-ksa \
  -n my-app \
  iam.gke.io/gcp-service-account=my-gsa@PROJECT_ID.iam.gserviceaccount.com

# Use in pod
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  serviceAccountName: my-ksa
  containers:
  - name: app
    image: google/cloud-sdk:slim
```

## VPC - Virtual Private Cloud

### VPC Network Types

```yaml
Auto Mode:
  - Automatically created subnets (one per region)
  - Predefined IP ranges (10.128.0.0/9)
  - Easy to start
  - Less control

Custom Mode:
  - Manually create subnets
  - Custom IP ranges
  - More control
  - Recommended for production
```

### Create VPC

```bash
# Create custom VPC
gcloud compute networks create my-vpc \
  --subnet-mode=custom

# Create subnet
gcloud compute networks subnets create my-subnet \
  --network=my-vpc \
  --region=us-central1 \
  --range=10.0.1.0/24

# Create subnet with secondary range (for GKE pods)
gcloud compute networks subnets create gke-subnet \
  --network=my-vpc \
  --region=us-central1 \
  --range=10.0.2.0/24 \
  --secondary-range=pods=10.4.0.0/14,services=10.0.32.0/20

# List VPCs
gcloud compute networks list

# List subnets
gcloud compute networks subnets list --network=my-vpc

# Delete VPC (must delete all resources first)
gcloud compute networks delete my-vpc
```

### Firewall Rules

```bash
# Default deny all ingress
# Default allow all egress

# Allow SSH from IAP
gcloud compute firewall-rules create allow-ssh-iap \
  --network=my-vpc \
  --allow=tcp:22 \
  --source-ranges=35.235.240.0/20 \
  --description="Allow SSH from IAP"

# Allow internal traffic
gcloud compute firewall-rules create allow-internal \
  --network=my-vpc \
  --allow=tcp,udp,icmp \
  --source-ranges=10.0.0.0/8

# Allow HTTP from everywhere with tag
gcloud compute firewall-rules create allow-http \
  --network=my-vpc \
  --allow=tcp:80 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=web-server
```

## GCP Best Practices

### Cost Optimization

```yaml
1. Committed Use Discounts:
   - 1 or 3 year commitment
   - Up to 57% savings on Compute Engine
   - Automatically applied

2. Sustained Use Discounts:
   - Automatic discounts
   - Based on monthly usage
   - Up to 30% savings
   - No commitment needed

3. Preemptible/Spot VMs:
   - Up to 80% savings
   - Batch processing, CI/CD
   - Fault-tolerant workloads

4. Right-Sizing:
   - Use recommender API
   - Start small, scale up
   - Custom machine types
   - Monitor with Cloud Monitoring

5. Storage Lifecycle:
   - Nearline for 30+ days
   - Coldline for 90+ days
   - Archive for 365+ days
   - Auto-delete old data

6. Network Egress:
   - Use Cloud CDN
   - Multi-region buckets for global access
   - Minimize data transfer between regions

7. Cost Monitoring:
   - Enable billing export
   - Use BigQuery for analysis
   - Set budget alerts
   - Label resources for cost allocation
```

### Security Best Practices

```yaml
1. Identity:
   - Use organization policies
   - Service accounts for applications
   - Workload Identity for GKE
   - Avoid service account keys

2. Network:
   - VPC Service Controls
   - Private Google Access
   - Cloud Armor for DDoS protection
   - VPC Flow Logs

3. Data:
   - Customer-managed encryption keys (CMEK)
   - Encrypt data at rest and in transit
   - Cloud KMS for key management
   - Secret Manager for secrets

4. Monitoring:
   - Cloud Logging
   - Cloud Monitoring
   - Security Command Center
   - Cloud Audit Logs

5. Compliance:
   - Policy Intelligence
   - Access Transparency
   - Config Connector
   - Forseti Security (open source)
```

### High Availability Patterns

```yaml
Multi-Zone:
  Compute:
    - Regional managed instance groups
    - Spread VMs across zones
    - Load balancer across zones
  
  Storage:
    - Regional persistent disks
    - Multi-region Cloud Storage
  
  Database:
    - Cloud SQL regional HA
    - Spanner (global by default)

Multi-Region:
  Global Load Balancer:
    - Anycast IP
    - Route to nearest region
    - Automatic failover
  
  Cloud CDN:
    - Cache content globally
    - Reduce latency
    - Offload backend
  
  Data Replication:
    - Multi-region buckets
    - Cloud Spanner (global)
    - Cloud SQL cross-region replicas
```

## DevOps on GCP

### 1. CI/CD with Cloud Build

**Cloud Build Pipeline:**
```yaml
# cloudbuild.yaml
steps:
  # Install dependencies
  - name: 'python:3.8-slim'
    entrypoint: pip
    args: ['install', '-r', 'requirements.txt', '--user']

  # Run tests
  - name: 'python:3.8-slim'
    entrypoint: python
    args: ['-m', 'pytest', 'tests/', '--junitxml=test_results.xml']
    env:
      - 'PYTHONPATH=/workspace'

  # Build container
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', 'gcr.io/$PROJECT_ID/devops-app:$COMMIT_SHA', '.']

  # Push container
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', 'gcr.io/$PROJECT_ID/devops-app:$COMMIT_SHA']

  # Deploy to Cloud Run
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: gcloud
    args:
      - 'run'
      - 'deploy'
      - 'devops-service'
      - '--image=gcr.io/$PROJECT_ID/devops-app:$COMMIT_SHA'
      - '--region=us-central1'
      - '--platform=managed'
      - '--allow-unauthenticated'

# Store test results
artifacts:
  objects:
    location: 'gs://$PROJECT_ID-cloudbuild-artifacts/'
    paths: ['test_results.xml']
```

**GitHub Actions with GCP:**
```yaml
name: Deploy to GCP
on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Authenticate to Google Cloud
      uses: google-github-actions/auth@v0
      with:
        credentials_json: ${{ secrets.GCP_SA_KEY }}
    
    - name: Set up Cloud SDK
      uses: google-github-actions/setup-gcloud@v0
    
    - name: Build and Push Docker Image
      run: |
        gcloud builds submit --tag gcr.io/${{ secrets.GCP_PROJECT_ID }}/devops-app:$GITHUB_SHA
    
    - name: Deploy to Cloud Run
      run: |
        gcloud run deploy devops-service \
          --image gcr.io/${{ secrets.GCP_PROJECT_ID }}/devops-app:$GITHUB_SHA \
          --region us-central1 \
          --platform managed \
          --allow-unauthenticated
```

### 2. Infrastructure as Code

**Deployment Manager:**
```yaml
# deployment-manager.yaml
resources:
  - name: devops-vpc
    type: compute.v1.network
    properties:
      routingConfig:
        routingMode: REGIONAL
      autoCreateSubnetworks: false

  - name: devops-subnet
    type: compute.v1.subnetwork
    properties:
      ipCidrRange: 10.0.0.0/24
      network: $(ref.devops-vpc.selfLink)
      region: us-central1

  - name: devops-vm
    type: compute.v1.instance
    properties:
      zone: us-central1-a
      machineType: zones/us-central1-a/machineTypes/n1-standard-1
      disks:
        - deviceName: boot
          type: PERSISTENT
          boot: true
          autoDelete: true
          initializeParams:
            sourceImage: projects/debian-cloud/global/images/family/debian-10
      networkInterfaces:
        - network: $(ref.devops-vpc.selfLink)
          subnetwork: $(ref.devops-subnet.selfLink)
          accessConfigs:
            - name: External NAT
              type: ONE_TO_ONE_NAT
```

**Terraform for GCP:**
```hcl
terraform {
  required_providers {
    google = {
      source = "hashicorp/google"
      version = "~> 4.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

resource "google_compute_network" "vpc" {
  name                    = "devops-vpc"
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "subnet" {
  name          = "devops-subnet"
  ip_cidr_range = "10.0.0.0/24"
  region        = var.region
  network       = google_compute_network.vpc.id
}

resource "google_container_cluster" "gke" {
  name     = "devops-cluster"
  location = var.region

  remove_default_node_pool = true
  initial_node_count       = 1

  network    = google_compute_network.vpc.name
  subnetwork = google_compute_subnetwork.subnet.name
}

resource "google_container_node_pool" "nodes" {
  name       = "devops-node-pool"
  cluster    = google_container_cluster.gke.id
  node_count = 2

  node_config {
    machine_type = "e2-medium"
    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]
  }
}
```

### 3. Monitoring & Observability

**Cloud Monitoring (Stackdriver):**
```yaml
# Cloud Monitoring Dashboard
dashboard:
  displayName: DevOps Dashboard
  gridLayout:
    columns: 2
    widgets:
      - title: CPU Utilization
        xyChart:
          dataSets:
            - timeSeriesQuery:
                timeSeriesFilter:
                  filter: metric.type="compute.googleapis.com/instance/cpu/utilization"
                  resource.type="gce_instance"
                unitOverride: "1"
          timeshiftDuration: 0s
          yAxis:
            label: CPU Utilization
            scale: LINEAR

      - title: Memory Usage
        xyChart:
          dataSets:
            - timeSeriesQuery:
                timeSeriesFilter:
                  filter: metric.type="compute.googleapis.com/instance/memory/balloon/ram_used"
                  resource.type="gce_instance"
          yAxis:
            label: Memory (bytes)
            scale: LINEAR
```

**Cloud Trace:**
```yaml
# Enable Cloud Trace for applications
# In your application code:

from google.cloud import trace

def traced_function():
    with trace.Client().trace() as span:
        span.add_label('operation', 'database_query')
        # Your code here
        return result
```

### 4. Container Services

**Google Kubernetes Engine (GKE):**
```yaml
# GKE Cluster with Autopilot
apiVersion: container.googleapis.com/v1
kind: Cluster
metadata:
  name: devops-cluster
spec:
  location: us-central1
  enableAutopilot: true
  network: projects/my-project/global/networks/devops-vpc
  subnetwork: projects/my-project/regions/us-central1/subnetworks/devops-subnet

---
# Kubernetes Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: devops-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: devops-app
  template:
    metadata:
      labels:
        app: devops-app
    spec:
      containers:
      - name: app
        image: gcr.io/my-project/devops-app:latest
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
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

**Cloud Run:**
```yaml
# Cloud Run Service
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: devops-service
spec:
  template:
    spec:
      containers:
      - image: gcr.io/my-project/devops-app:latest
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: 1000m
            memory: 512Mi
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: url
  traffic:
  - percent: 100
    latestRevision: true
```

### 5. Serverless Computing

**Cloud Functions:**
```python
# main.py
import functions_framework
from google.cloud import firestore

@functions_framework.http
def hello_http(request):
    """HTTP Cloud Function."""
    request_json = request.get_json(silent=True)
    request_args = request.args

    if request_json and 'name' in request_json:
        name = request_json['name']
    elif request_args and 'name' in request_args:
        name = request_args['name']
    else:
        name = 'World'

    return f'Hello {name}!'

@functions_framework.cloud_event
def hello_cloud_event(cloud_event):
    """Cloud Event Function."""
    data = cloud_event.data
    print(f"Event: {data}")
```

**Cloud Run Functions:**
```yaml
# Function deployment
gcloud functions deploy devops-function \
  --runtime python39 \
  --trigger-http \
  --allow-unauthenticated \
  --source . \
  --entry-point hello_http
```

### 6. Security in DevOps

**Security Command Center:**
```yaml
# Security Health Analytics
# Enable security health analytics for the project
gcloud alpha security security-center settings describe

# Create custom security health analytics module
gcloud alpha security security-center custom-modules create \
  --organization=123456789 \
  --display-name="DevOps Security Checks" \
  --custom-config-file=config.yaml
```

**Binary Authorization:**
```yaml
# Binary Authorization Policy
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: devops-image-policy
spec:
  images:
  - glob: gcr.io/my-project/devops-app/**
  authorities:
  - key:
      data: |
        -----BEGIN PUBLIC KEY-----
        MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE...
        -----END PUBLIC KEY-----
```

GCP cung cấp infrastructure hiện đại với performance cao và tích hợp sâu với AI/ML services!
