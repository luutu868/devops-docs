# Multi-Cloud Strategies

## Giới Thiệu Multi-Cloud

Multi-cloud là chiến lược sử dụng nhiều cloud providers đồng thời để tối ưu hóa performance, cost, và risk management. Thay vì lock-in vào một provider, organizations có thể leverage strengths của từng cloud.

**Lợi ích Multi-Cloud:**
- **Avoid vendor lock-in**: Không phụ thuộc vào một provider
- **Best of breed**: Chọn service tốt nhất cho từng use case
- **Disaster recovery**: Backup across providers
- **Cost optimization**: Leverage pricing differences
- **Compliance**: Meet data residency requirements
- **Geographic coverage**: Global presence

## Multi-Cloud Architecture Patterns

### 1. Active-Active Multi-Cloud

```
Production Workloads
├── AWS (Primary)
│   ├── US East: Web Apps
│   └── EU West: Database
└── Azure (Secondary)
    ├── Asia Pacific: Analytics
    └── South America: Backup
```

**Characteristics:**
- Workloads chạy trên multiple clouds simultaneously
- Load balancing across providers
- High availability và performance
- Complex management

### 2. Active-Passive Multi-Cloud

```
Primary Cloud (AWS)
├── Production Workloads
├── Database
└── Storage

Secondary Cloud (GCP)
├── Disaster Recovery
├── Backup
└── Read Replicas
```

**Characteristics:**
- Primary cloud handles production
- Secondary cloud for DR/failover
- Cost-effective cho DR
- Simpler management

### 3. Cloud Bursting

```
Private Cloud / On-Premises
├── Normal Operations
└── Scale Out → Public Cloud

Public Cloud (AWS/Azure/GCP)
├── Additional Capacity
├── Temporary Resources
└── Auto-scaling
```

**Characteristics:**
- Hybrid approach
- Scale to cloud khi cần
- Cost optimization
- Workload portability

## Multi-Cloud Management Tools

### 1. Infrastructure as Code

**Terraform Multi-Cloud:**
```hcl
# AWS Provider
provider "aws" {
  region = "us-east-1"
}

# Azure Provider
provider "azurerm" {
  features {}
}

# GCP Provider
provider "google" {
  project = "my-project"
  region  = "us-central1"
}
```

**Cross-Cloud Resources:**
```hcl
# AWS VPC
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

# Azure VNet
resource "azurerm_virtual_network" "main" {
  address_space = ["10.1.0.0/16"]
}

# GCP VPC
resource "google_compute_network" "main" {
  name = "main-network"
}
```

### 2. Container Orchestration

**Kubernetes Multi-Cloud:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: multi-cloud-config
data:
  cloud-provider: "aws"  # or azure, gcp
  region: "us-east-1"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multi-cloud-app
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        env:
        - name: CLOUD_PROVIDER
          valueFrom:
            configMapKeyRef:
              name: multi-cloud-config
              key: cloud-provider
```

### 3. Multi-Cloud Management Platforms

**Popular Tools:**
- **HashiCorp Consul**: Service mesh across clouds
- **Istio**: Traffic management multi-cloud
- **CloudHealth**: Cost management
- **Cisco Intersight**: Multi-cloud operations
- **IBM Multicloud Manager**: Hybrid cloud management

## Data Management in Multi-Cloud

### 1. Data Replication Strategies

**Active-Active Replication:**
```yaml
AWS S3 Bucket
├── Primary Region: us-east-1
└── Cross-Region Replication
    ├── Azure Blob: eastus
    └── GCP Cloud Storage: us-central1
```

**Database Replication:**
```sql
-- AWS RDS Primary
CREATE PUBLICATION my_publication FOR ALL TABLES;

-- Azure Database Secondary
CREATE SUBSCRIPTION my_subscription
    CONNECTION 'host=aws-rds-endpoint port=5432 user=replicator dbname=mydb'
    PUBLICATION my_publication;
```

### 2. Data Consistency

**Eventual Consistency:**
- S3 Cross-Region Replication
- DynamoDB Global Tables
- Cosmos DB multi-region

**Strong Consistency:**
- Database transactions
- File system replication
- Application-level consistency

### 3. Data Transfer Solutions

**AWS DataSync:**
```bash
# Transfer data between clouds
aws datasync create-location-s3 \
    --s3-bucket-arn arn:aws:s3:::my-bucket \
    --s3-config '{"BucketAccessRoleArn":"arn:aws:iam::123456789012:role/DataSyncRole"}'
```

**Azure Data Factory:**
```json
{
  "name": "CrossCloudCopy",
  "type": "Microsoft.DataFactory/factories/datasets",
  "properties": {
    "linkedServiceName": {
      "referenceName": "AWSS3LinkedService",
      "type": "LinkedServiceReference"
    }
  }
}
```

## Networking in Multi-Cloud

### 1. Inter-Cloud Connectivity

**VPN Connections:**
```bash
# AWS to Azure VPN
aws ec2 create-vpn-connection \
    --type ipsec.1 \
    --customer-gateway-id cgw-12345678 \
    --vpn-gateway-id vgw-12345678
```

**Direct Connect / ExpressRoute:**
- AWS Direct Connect
- Azure ExpressRoute
- GCP Cloud Interconnect

### 2. DNS Management

**Multi-Cloud DNS:**
```yaml
# Route53 Primary
aws route53 change-resource-record-sets \
    --hosted-zone-id Z123456789 \
    --change-batch '{
      "Changes": [{
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "app.example.com",
          "Type": "CNAME",
          "SetIdentifier": "aws-primary",
          "Weight": 100,
          "TTL": 300,
          "ResourceRecords": [{"Value": "aws-lb-123.elb.amazonaws.com"}]
        }
      }]
    }'
```

### 3. Load Balancing

**Global Load Balancing:**
- AWS Global Accelerator
- Azure Front Door
- GCP Cloud Load Balancing
- Cloudflare Load Balancing

## Security in Multi-Cloud

### 1. Identity Management

**Federated Identity:**
```yaml
# AWS IAM Identity Provider
resource "aws_iam_openid_connect_provider" "azure_ad" {
  url = "https://login.microsoftonline.com/12345678-1234-1234-1234-123456789012"
  client_id_list = ["sts.amazonaws.com"]
  thumbprint_list = ["1234567890123456789012345678901234567890"]
}
```

### 2. Access Control

**Attribute-Based Access Control (ABAC):**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalTag/CloudProvider": "AWS"
        }
      }
    }
  ]
}
```

### 3. Security Monitoring

**Multi-Cloud SIEM:**
- AWS CloudWatch + Azure Monitor + GCP Cloud Monitoring
- Third-party tools: Splunk, Datadog, New Relic
- Custom dashboards và alerts

## Cost Optimization

### 1. Pricing Comparison

**Compute Pricing (per hour):**
| Instance Type | AWS | Azure | GCP |
|---------------|-----|-------|-----|
| 2 vCPU, 4GB RAM | $0.096 | $0.088 | $0.076 |
| 4 vCPU, 16GB RAM | $0.192 | $0.176 | $0.152 |
| 8 vCPU, 32GB RAM | $0.384 | $0.352 | $0.304 |

### 2. Reserved Instances

**AWS Reserved Instances:**
```bash
aws ec2 purchase-reserved-instances-offering \
    --reserved-instances-offering-id offering-id \
    --instance-count 1
```

**Azure Reservations:**
```bash
az reservations purchase \
    --reservation-order-id order-id \
    --applied-scope-type Shared
```

### 3. Spot Instances / Preemptible VMs

**AWS Spot Instances:**
```bash
aws ec2 request-spot-instances \
    --spot-price "0.10" \
    --instance-count 1 \
    --type "persistent" \
    --launch-specification file://launch-spec.json
```

**GCP Preemptible VMs:**
```yaml
apiVersion: compute/v1
kind: Instance
metadata:
  name: preemptible-vm
spec:
  scheduling:
    preemptible: true
    automaticRestart: false
    onHostMaintenance: TERMINATE
```

## DevOps in Multi-Cloud

### 1. CI/CD Pipelines

**GitHub Actions Multi-Cloud:**
```yaml
name: Multi-Cloud Deploy
on: [push]
jobs:
  deploy-aws:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Deploy to AWS
      run: |
        aws configure set aws_access_key_id ${{ secrets.AWS_ACCESS_KEY }}
        aws configure set aws_secret_access_key ${{ secrets.AWS_SECRET_KEY }}
        aws s3 sync . s3://my-bucket --delete

  deploy-azure:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Deploy to Azure
      uses: azure/webapps-deploy@v2
      with:
        app-name: my-app
        publish-profile: ${{ secrets.AZURE_PUBLISH_PROFILE }}
```

### 2. Configuration Management

**Ansible Multi-Cloud:**
```yaml
---
- name: Configure Multi-Cloud Infrastructure
  hosts: all
  vars:
    cloud_provider: "{{ lookup('env', 'CLOUD_PROVIDER') }}"
  tasks:
    - name: Install cloud-specific packages
      package:
        name: "{{ item }}"
        state: present
      loop: "{{ cloud_packages[cloud_provider] }}"
      vars:
        cloud_packages:
          aws: [awscli]
          azure: [azure-cli]
          gcp: [google-cloud-sdk]
```

### 3. Monitoring & Observability

**Prometheus Multi-Cloud:**
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'aws-ec2'
    ec2_sd_configs:
      - region: us-east-1
        access_key: ${{AWS_ACCESS_KEY}}
        secret_key: ${{AWS_SECRET_KEY}}
    relabel_configs:
      - source_labels: [__meta_ec2_tag_Name]
        target_label: instance_name

  - job_name: 'azure-vm'
    azure_sd_configs:
      - environment: AzurePublicCloud
        subscription_id: ${{AZURE_SUBSCRIPTION_ID}}
        client_id: ${{AZURE_CLIENT_ID}}
        client_secret: ${{AZURE_CLIENT_SECRET}}
        tenant_id: ${{AZURE_TENANT_ID}}
```

## Migration Strategies

### 1. Assessment Phase

**Cloud Readiness Assessment:**
```bash
# AWS Application Discovery Service
aws discovery start-data-collection-by-agent-ids \
    --agent-ids agent-id-1 agent-id-2

# Azure Migrate
az migrate assessment create \
    --name my-assessment \
    --resource-group myRG \
    --target-region eastus
```

### 2. Migration Approaches

**6R Migration Strategy:**
- **Rehost**: Lift and shift
- **Refactor**: Optimize for cloud
- **Rearchitect**: Major changes
- **Rebuild**: Complete rewrite
- **Replace**: Switch to SaaS
- **Retire**: Decommission

### 3. Migration Tools

**AWS Migration Tools:**
- AWS Server Migration Service (SMS)
- AWS Database Migration Service (DMS)
- AWS Migration Hub

**Azure Migration Tools:**
- Azure Migrate
- Azure Database Migration Service
- Azure Site Recovery

**GCP Migration Tools:**
- Migrate for Compute Engine
- Database Migration Service
- Transfer Service

## Best Practices

### 1. Governance

**Multi-Cloud Governance Framework:**
```yaml
policies:
  - name: resource-tagging
    description: "All resources must be tagged"
    rules:
      - cloud: aws
        service: ec2
        tags: [Environment, Owner, CostCenter]
      - cloud: azure
        service: vm
        tags: [Environment, Owner, CostCenter]
      - cloud: gcp
        service: compute
        labels: [environment, owner, costcenter]
```

### 2. Compliance

**Multi-Cloud Compliance:**
- **Data Sovereignty**: Regional data storage
- **GDPR**: Data processing agreements
- **HIPAA**: Healthcare data protection
- **PCI DSS**: Payment card industry

### 3. Team Structure

**Multi-Cloud Team Organization:**
```
Cloud Center of Excellence (CCoE)
├── Cloud Architects
│   ├── AWS Specialists
│   ├── Azure Specialists
│   └── GCP Specialists
├── DevOps Engineers
├── Security Team
└── FinOps Team
```

## Challenges & Solutions

### 1. Complexity Management

**Challenge:** Managing multiple cloud platforms
**Solutions:**
- Unified management tools
- Infrastructure as Code
- Automation scripts
- Centralized monitoring

### 2. Cost Management

**Challenge:** Cost visibility across clouds
**Solutions:**
- Multi-cloud cost monitoring tools
- Budget alerts
- Resource optimization
- Reserved instance management

### 3. Skill Gaps

**Challenge:** Diverse skill requirements
**Solutions:**
- Cross-training programs
- Specialized roles
- Vendor certifications
- Knowledge sharing

## Future Trends

### 1. Edge Computing

**Multi-Cloud Edge:**
- AWS Outposts
- Azure Arc
- GCP Anthos
- Distributed cloud computing

### 2. Serverless Multi-Cloud

**Serverless Platforms:**
- AWS Lambda + Azure Functions + GCP Cloud Functions
- Cross-cloud function orchestration
- Event-driven architectures

### 3. AI/ML Multi-Cloud

**AI Services:**
- AWS SageMaker + Azure ML + GCP AI Platform
- Model training across clouds
- Inference optimization

Multi-cloud strategies provide flexibility, resilience, và optimization opportunities, nhưng đòi hỏi careful planning và robust management practices để thành công!