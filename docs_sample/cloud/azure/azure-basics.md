# Azure Basics - Microsoft Azure

## Giới Thiệu Azure

Microsoft Azure là nền tảng điện toán đám mây của Microsoft, cung cấp 200+ services cho computing, analytics, storage và networking.

**Ưu điểm Azure:**
- **Enterprise ready**: Strong integration với Microsoft ecosystem
- **Hybrid cloud**: Best hybrid capabilities (Azure Arc, Azure Stack)
- **Windows workloads**: Native support for .NET, SQL Server, Active Directory
- **Global reach**: 60+ regions (most of any cloud provider)
- **Compliance**: 90+ compliance certifications
- **Innovation**: Leading in enterprise AI, IoT, mixed reality

## Azure Global Infrastructure

### Regions và Availability Zones

```
Geography (United States)
├── Region Pair
│   ├── East US (Virginia)
│   │   ├── Availability Zone 1
│   │   ├── Availability Zone 2
│   │   └── Availability Zone 3
│   └── West US (California)
│       ├── Availability Zone 1
│       ├── Availability Zone 2
│       └── Availability Zone 3
└── Regional Services
```

**Geography:**
- Data residency boundary
- Example: United States, Europe, Asia Pacific

**Regions:**
- 60+ regions globally
- Most regions worldwide
- Paired for disaster recovery

**Region Pairs:**
- Physical isolation (300+ miles)
- Platform updates sequentially
- Priority recovery in outages
- Data residency within geography

**Availability Zones:**
- Physically separate locations
- Independent power, cooling, networking
- High-speed private fiber
- Not all regions have AZs

### Region Selection

```yaml
Factors:
  Proximity:
    - User/customer location
    - Latency requirements
    - Data sovereignty
  
  Features:
    - Not all services in all regions
    - AZ availability varies
    - Preview features limited
  
  Pricing:
    - Varies by region
    - US regions typically cheaper
    - Check regional pricing
  
  Compliance:
    - Government clouds (Azure Government, China)
    - GDPR (European regions)
    - Industry-specific requirements
```

## Azure Active Directory (Azure AD)

### Azure AD Concepts

**Identity Types:**
```
Azure AD Tenant (company.onmicrosoft.com)
├── Users
│   ├── Cloud identities (alice@company.onmicrosoft.com)
│   ├── Synchronized identities (from on-premises AD)
│   └── Guest users (B2B collaboration)
├── Groups
│   ├── Security groups
│   └── Microsoft 365 groups
├── Service Principals
│   ├── Application identity
│   └── Managed identity
└── Devices
    ├── Azure AD joined
    ├── Hybrid Azure AD joined
    └── Registered
```

**Resource Hierarchy:**
```
Azure AD Tenant
└── Management Groups
    ├── Production MG
    │   ├── Subscription: Prod-App
    │   │   ├── Resource Group: RG-Web
    │   │   │   ├── VM, Storage, Database
    │   │   │   └── RBAC assignments
    │   │   └── Resource Group: RG-API
    │   └── Subscription: Prod-Data
    └── Development MG
        └── Subscription: Dev-All
            └── Resource Groups

Policies and permissions inherit down
```

### RBAC - Role-Based Access Control

**Built-in Roles:**
```yaml
Owner:
  - Full access to resources
  - Grant access to others
  - Highest privilege

Contributor:
  - Create and manage resources
  - Cannot grant access to others
  - No RBAC permissions

Reader:
  - View resources only
  - No modifications
  - Read-only access

Specific Roles:
  Virtual Machine Contributor:
    - Manage VMs but not network/storage
  
  Storage Blob Data Contributor:
    - Read, write, delete blob data
  
  SQL DB Contributor:
    - Manage SQL databases
  
  Network Contributor:
    - Manage network resources
```

**Custom Roles:**
```json
{
  "Name": "VM Operator",
  "Description": "Can start and stop VMs",
  "Actions": [
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/restart/action",
    "Microsoft.Compute/virtualMachines/deallocate/action",
    "Microsoft.Compute/virtualMachines/read"
  ],
  "NotActions": [],
  "AssignableScopes": [
    "/subscriptions/{subscription-id}"
  ]
}
```

### Azure CLI Commands

```bash
# Login
az login

# List accounts
az account list --output table

# Set subscription
az account set --subscription "My Subscription"

# Show current subscription
az account show

# List resource groups
az group list --output table

# Create resource group
az group create \
  --name MyResourceGroup \
  --location eastus

# List role assignments
az role assignment list \
  --resource-group MyResourceGroup \
  --output table

# Assign role to user
az role assignment create \
  --assignee alice@company.com \
  --role "Virtual Machine Contributor" \
  --resource-group MyResourceGroup

# Remove role assignment
az role assignment delete \
  --assignee alice@company.com \
  --role "Contributor" \
  --resource-group MyResourceGroup

# List service principals
az ad sp list --all

# Create service principal
az ad sp create-for-rbac \
  --name MyAppServicePrincipal \
  --role Contributor \
  --scopes /subscriptions/{subscription-id}

# Reset service principal credentials
az ad sp credential reset \
  --name MyAppServicePrincipal
```

### Managed Identities

```yaml
System-Assigned:
  - Lifecycle tied to resource
  - Automatically created/deleted
  - 1:1 relationship with resource
  - Use: Single resource access
  
  Example:
    VM → System-assigned identity → Access Key Vault

User-Assigned:
  - Independent lifecycle
  - Can be shared across resources
  - Managed separately
  - Use: Multiple resources, long-lived identity
  
  Example:
    VM1, VM2, VM3 → User-assigned identity → Access Storage
```

## Virtual Machines

### VM Sizes

```yaml
General Purpose (B, D, DS, A):
  B-series: Burstable, dev/test
    - B1s, B1ms, B2s, B2ms
  
  D-series: Balanced CPU/memory
    - D2s_v5, D4s_v5, D8s_v5

Compute Optimized (F, FS):
  F-series: High CPU-to-memory
    - F2s_v2, F4s_v2, F8s_v2
  - Batch processing, web servers

Memory Optimized (E, ES, M):
  E-series: High memory-to-CPU
    - E2s_v5, E4s_v5, E8s_v5
  - Databases, in-memory analytics
  
  M-series: Largest memory (4TB - 12TB)
    - SAP HANA workloads

Storage Optimized (L, LS):
  L-series: High disk throughput
    - L8s_v3, L16s_v3
  - NoSQL databases, data warehousing

GPU (NC, ND, NV):
  NC-series: GPU compute
  ND-series: Deep learning
  NV-series: Visualization
```

### Pricing Models

**1. Pay-as-you-go:**
```yaml
No commitment
Most expensive
Per-minute billing

Use:
  - Short-term workloads
  - Development/testing
  - Unpredictable workloads
```

**2. Reserved Instances:**
```yaml
1 or 3 year commitment
Up to 72% savings
Payment: All upfront, Partial, Monthly

Use:
  - Steady-state workloads
  - Production systems
  - Long-running applications
```

**3. Spot VMs:**
```yaml
Up to 90% savings
Eviction with 30-second notice
Bid on unused capacity

Use:
  - Batch processing
  - Non-critical workloads
  - Stateless applications
```

**4. Azure Hybrid Benefit:**
```yaml
Use existing Windows Server licenses
Up to 40% savings on VMs
Combined with Reserved Instances

Use:
  - Windows Server workloads
  - SQL Server on VMs
```

### Create Virtual Machine

```bash
# Create VM with defaults
az vm create \
  --resource-group MyResourceGroup \
  --name MyVM \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --admin-username azureuser \
  --generate-ssh-keys

# Create Windows VM
az vm create \
  --resource-group MyResourceGroup \
  --name MyWindowsVM \
  --image Win2022Datacenter \
  --size Standard_D2s_v5 \
  --admin-username azureadmin \
  --admin-password 'SecurePassword123!'

# Create VM with custom script
az vm create \
  --resource-group MyResourceGroup \
  --name WebServer \
  --image Ubuntu2204 \
  --size Standard_B1ms \
  --admin-username azureuser \
  --generate-ssh-keys \
  --custom-data cloud-init.txt

# cloud-init.txt
#cloud-config
package_upgrade: true
packages:
  - nginx
runcmd:
  - systemctl start nginx
  - systemctl enable nginx
  - echo "Hello from $(hostname)" > /var/www/html/index.html

# Create VM with managed identity
az vm create \
  --resource-group MyResourceGroup \
  --name MyVM \
  --image Ubuntu2204 \
  --assign-identity \
  --role Contributor \
  --scope /subscriptions/{subscription-id}

# Create spot VM
az vm create \
  --resource-group MyResourceGroup \
  --name SpotVM \
  --image Ubuntu2204 \
  --priority Spot \
  --max-price -1 \
  --eviction-policy Deallocate

# List VMs
az vm list --output table

# Start VM
az vm start --resource-group MyResourceGroup --name MyVM

# Stop VM (still charged for storage)
az vm stop --resource-group MyResourceGroup --name MyVM

# Deallocate VM (not charged)
az vm deallocate --resource-group MyResourceGroup --name MyVM

# Delete VM
az vm delete --resource-group MyResourceGroup --name MyVM --yes

# Open port
az vm open-port \
  --resource-group MyResourceGroup \
  --name MyVM \
  --port 80

# Get VM IP
az vm show \
  --resource-group MyResourceGroup \
  --name MyVM \
  --show-details \
  --query [publicIps] \
  --output tsv
```

### VM Scale Sets

```bash
# Create VM scale set
az vmss create \
  --resource-group MyResourceGroup \
  --name MyScaleSet \
  --image Ubuntu2204 \
  --vm-sku Standard_B2s \
  --instance-count 2 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --load-balancer MyLoadBalancer \
  --upgrade-policy-mode Automatic

# Scale manually
az vmss scale \
  --resource-group MyResourceGroup \
  --name MyScaleSet \
  --new-capacity 5

# Configure autoscale
az monitor autoscale create \
  --resource-group MyResourceGroup \
  --resource MyScaleSet \
  --resource-type Microsoft.Compute/virtualMachineScaleSets \
  --name MyAutoscaleProfile \
  --min-count 2 \
  --max-count 10 \
  --count 3

# Add scale-out rule
az monitor autoscale rule create \
  --resource-group MyResourceGroup \
  --autoscale-name MyAutoscaleProfile \
  --condition "Percentage CPU > 70 avg 5m" \
  --scale out 2

# Add scale-in rule
az monitor autoscale rule create \
  --resource-group MyResourceGroup \
  --autoscale-name MyAutoscaleProfile \
  --condition "Percentage CPU < 30 avg 5m" \
  --scale in 1
```

## Azure Storage

### Storage Account Types

```yaml
General Purpose v2:
  - Blob, File, Queue, Table
  - Hot, Cool, Archive tiers
  - Most cost-effective
  - Recommended for most scenarios

General Purpose v1:
  - Legacy
  - No tiering
  - Not recommended

Blob Storage:
  - Block blobs and append blobs only
  - Legacy
  - Use GPv2 instead

Premium:
  - SSD storage
  - Low latency
  - Higher cost
  
  Types:
    - Premium Block Blob
    - Premium File Shares
    - Premium Page Blob (VM disks)
```

### Blob Access Tiers

```yaml
Hot:
  - Frequently accessed data
  - Highest storage cost
  - Lowest access cost
  - Use: Active data

Cool:
  - Infrequently accessed (30+ days)
  - Lower storage cost
  - Higher access cost
  - Use: Short-term backup

Archive:
  - Rarely accessed (180+ days)
  - Lowest storage cost
  - Highest access cost
  - Rehydration required (hours)
  - Use: Long-term archive
```

### Azure Storage Commands

```bash
# Create storage account
az storage account create \
  --name mystorageaccount123 \
  --resource-group MyResourceGroup \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2

# Get connection string
az storage account show-connection-string \
  --name mystorageaccount123 \
  --resource-group MyResourceGroup

# Create container
az storage container create \
  --name mycontainer \
  --account-name mystorageaccount123

# Upload blob
az storage blob upload \
  --container-name mycontainer \
  --name myblob.txt \
  --file ./myfile.txt \
  --account-name mystorageaccount123

# Upload directory
az storage blob upload-batch \
  --destination mycontainer \
  --source ./mydir \
  --account-name mystorageaccount123

# List blobs
az storage blob list \
  --container-name mycontainer \
  --account-name mystorageaccount123 \
  --output table

# Download blob
az storage blob download \
  --container-name mycontainer \
  --name myblob.txt \
  --file ./downloaded.txt \
  --account-name mystorageaccount123

# Set blob tier
az storage blob set-tier \
  --name myblob.txt \
  --container-name mycontainer \
  --tier Cool \
  --account-name mystorageaccount123

# Generate SAS token
az storage blob generate-sas \
  --container-name mycontainer \
  --name myblob.txt \
  --permissions r \
  --expiry 2024-12-31 \
  --account-name mystorageaccount123

# Delete blob
az storage blob delete \
  --container-name mycontainer \
  --name myblob.txt \
  --account-name mystorageaccount123
```

### Storage Redundancy

```yaml
Locally Redundant (LRS):
  - 3 copies within datacenter
  - 99.999999999% durability
  - Lowest cost
  - Use: Non-critical data

Zone Redundant (ZRS):
  - 3 copies across availability zones
  - Higher availability
  - Regional disasters protected
  - Use: High availability needs

Geo-Redundant (GRS):
  - LRS + async replication to paired region
  - 6 copies total (3 primary + 3 secondary)
  - Read access only on failover
  - Use: Disaster recovery

Geo-Zone Redundant (GZRS):
  - ZRS + async replication to paired region
  - Highest durability
  - Highest cost
  - Use: Critical data, maximum availability
```

## Azure SQL Database

### Service Tiers

```yaml
DTU-based:
  Basic:
    - Dev/test, small databases
    - Up to 2GB
    - Low performance
  
  Standard:
    - Most workloads
    - Up to 1TB
    - Moderate performance
  
  Premium:
    - High performance
    - Up to 4TB
    - In-memory OLTP

vCore-based (Recommended):
  General Purpose:
    - Balanced compute/memory
    - Remote storage
    - Most workloads
  
  Business Critical:
    - High performance
    - Local SSD storage
    - Built-in HA
    - Read replicas included
  
  Hyperscale:
    - Up to 100TB
    - Rapid scale
    - Fast backups
```

### Deployment Options

```yaml
Single Database:
  - Independent database
  - Own resources
  - Predictable performance
  - Use: Single application

Elastic Pool:
  - Shared resources across databases
  - Cost-effective for multiple DBs
  - Variable usage patterns
  - Use: SaaS applications

Managed Instance:
  - Near 100% SQL Server compatibility
  - VNet integration
  - SQL Agent, CLR, cross-DB queries
  - Use: Lift-and-shift migrations
```

### Create Azure SQL Database

```bash
# Create SQL server
az sql server create \
  --name myserver123 \
  --resource-group MyResourceGroup \
  --location eastus \
  --admin-user sqladmin \
  --admin-password 'SecurePassword123!'

# Configure firewall (allow Azure services)
az sql server firewall-rule create \
  --resource-group MyResourceGroup \
  --server myserver123 \
  --name AllowAzureServices \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0

# Add your IP
az sql server firewall-rule create \
  --resource-group MyResourceGroup \
  --server myserver123 \
  --name AllowMyIP \
  --start-ip-address 203.0.113.10 \
  --end-ip-address 203.0.113.10

# Create database
az sql db create \
  --resource-group MyResourceGroup \
  --server myserver123 \
  --name mydatabase \
  --service-objective S0 \
  --backup-storage-redundancy Geo

# Create vCore database
az sql db create \
  --resource-group MyResourceGroup \
  --server myserver123 \
  --name myvcoredatabase \
  --edition GeneralPurpose \
  --family Gen5 \
  --capacity 2 \
  --compute-model Serverless \
  --auto-pause-delay 60

# List databases
az sql db list \
  --resource-group MyResourceGroup \
  --server myserver123 \
  --output table

# Scale database
az sql db update \
  --resource-group MyResourceGroup \
  --server myserver123 \
  --name mydatabase \
  --service-objective S1

# Create geo-replica
az sql db replica create \
  --resource-group MyResourceGroup \
  --server myserver123 \
  --name mydatabase \
  --partner-server myserver-secondary \
  --partner-resource-group MyResourceGroup-Secondary

# Get connection string
az sql db show-connection-string \
  --client sqlcmd \
  --name mydatabase \
  --server myserver123
```

## AKS - Azure Kubernetes Service

### Create AKS Cluster

```bash
# Create AKS cluster
az aks create \
  --resource-group MyResourceGroup \
  --name MyAKSCluster \
  --node-count 3 \
  --node-vm-size Standard_D2s_v3 \
  --enable-managed-identity \
  --enable-addons monitoring \
  --generate-ssh-keys

# Create with availability zones
az aks create \
  --resource-group MyResourceGroup \
  --name MyAKSCluster \
  --node-count 3 \
  --zones 1 2 3 \
  --enable-managed-identity

# Create with autoscaling
az aks create \
  --resource-group MyResourceGroup \
  --name MyAKSCluster \
  --node-count 2 \
  --min-count 1 \
  --max-count 5 \
  --enable-cluster-autoscaler

# Get credentials
az aks get-credentials \
  --resource-group MyResourceGroup \
  --name MyAKSCluster

# List clusters
az aks list --output table

# Scale node pool
az aks scale \
  --resource-group MyResourceGroup \
  --name MyAKSCluster \
  --node-count 5

# Upgrade cluster
az aks upgrade \
  --resource-group MyResourceGroup \
  --name MyAKSCluster \
  --kubernetes-version 1.27.3

# Add node pool
az aks nodepool add \
  --resource-group MyResourceGroup \
  --cluster-name MyAKSCluster \
  --name spotpool \
  --node-count 3 \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1

# Delete cluster
az aks delete \
  --resource-group MyResourceGroup \
  --name MyAKSCluster \
  --yes
```

### AAD Pod Identity

```bash
# Enable AAD Pod Identity
az aks update \
  --resource-group MyResourceGroup \
  --name MyAKSCluster \
  --enable-managed-identity \
  --enable-pod-identity

# Create managed identity
az identity create \
  --resource-group MyResourceGroup \
  --name MyPodIdentity

# Get identity info
IDENTITY_CLIENT_ID=$(az identity show \
  --resource-group MyResourceGroup \
  --name MyPodIdentity \
  --query clientId \
  --output tsv)

IDENTITY_RESOURCE_ID=$(az identity show \
  --resource-group MyResourceGroup \
  --name MyPodIdentity \
  --query id \
  --output tsv)

# Grant permissions to identity
az role assignment create \
  --role "Storage Blob Data Contributor" \
  --assignee $IDENTITY_CLIENT_ID \
  --scope /subscriptions/{subscription-id}/resourceGroups/MyResourceGroup

# Create pod identity in cluster
az aks pod-identity add \
  --resource-group MyResourceGroup \
  --cluster-name MyAKSCluster \
  --namespace default \
  --name my-pod-identity \
  --identity-resource-id $IDENTITY_RESOURCE_ID
```

## Azure Best Practices

### Cost Optimization

```yaml
1. Reserved Instances:
   - 1 or 3 year commitment
   - Up to 72% savings
   - VM, SQL Database, Cosmos DB
   
2. Azure Hybrid Benefit:
   - Use existing licenses
   - Windows Server, SQL Server
   - Up to 40% savings

3. Spot VMs:
   - Up to 90% savings
   - Batch processing, dev/test
   - Stateless workloads

4. Auto-shutdown:
   - Schedule VM shutdown
   - Development/test environments
   - Significant savings

5. Right-sizing:
   - Azure Advisor recommendations
   - Start small, scale up
   - Monitor with Azure Monitor

6. Storage Tiers:
   - Hot for active data
   - Cool for 30+ days
   - Archive for long-term

7. Cost Management:
   - Set budgets and alerts
   - Cost analysis
   - Resource tagging
   - Departmental chargebacks
```

### Security Best Practices

```yaml
1. Identity:
   - Azure AD authentication
   - Managed identities
   - Conditional access
   - MFA for all users

2. Network:
   - Network Security Groups
   - Application Security Groups
   - Azure Firewall
   - DDoS Protection

3. Data:
   - Encryption at rest (Azure Storage Service Encryption)
   - Encryption in transit (TLS/SSL)
   - Azure Key Vault for secrets
   - Customer-managed keys

4. Monitoring:
   - Azure Security Center
   - Azure Sentinel (SIEM)
   - Azure Monitor
   - Activity logs

5. Governance:
   - Azure Policy
   - Azure Blueprints
   - Management groups
   - Role-based access control
```

### High Availability

```yaml
Availability Sets:
  - 99.95% SLA
  - Fault domains (rack level)
  - Update domains (planned maintenance)
  - Max 3 fault domains, 20 update domains

Availability Zones:
  - 99.99% SLA
  - Datacenter level isolation
  - 3 zones per region
  - Zone-redundant services

Load Balancer:
  - Distribute traffic
  - Health probes
  - Outbound connections
  - Standard SKU supports zones

Application Gateway:
  - Layer 7 load balancer
  - WAF capabilities
  - SSL termination
  - URL-based routing

Traffic Manager:
  - DNS-based load balancing
  - Global distribution
  - Multiple routing methods
  - Cross-region failover
```

## DevOps on Azure

### 1. CI/CD with Azure DevOps

**Azure Pipelines:**
```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
    - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  azureSubscription: 'my-subscription'
  appName: 'devops-app'
  resourceGroup: 'devops-rg'

stages:
- stage: Build
  jobs:
  - job: Build
    steps:
    - task: UsePythonVersion@0
      inputs:
        versionSpec: '3.8'
    
    - script: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
      displayName: 'Install dependencies'
    
    - script: |
        python -m pytest tests/ --junitxml=test-results.xml
      displayName: 'Run tests'
    
    - task: PublishTestResults@2
      inputs:
        testResultsFiles: 'test-results.xml'
        testRunTitle: 'Python Tests'
    
    - task: Docker@2
      inputs:
        containerRegistry: 'my-acr'
        repository: '$(appName)'
        command: 'buildAndPush'
        Dockerfile: '**/Dockerfile'
        tags: |
          $(Build.BuildId)
          latest

- stage: Deploy
  jobs:
  - deployment: Deploy
    environment: 'production'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureWebAppContainer@1
            inputs:
              azureSubscription: '$(azureSubscription)'
              appName: '$(appName)'
              containers: 'myacr.azurecr.io/$(appName):$(Build.BuildId)'
```

**GitHub Actions with Azure:**
```yaml
name: Deploy to Azure
on:
  push:
    branches: [ main ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Login to Azure
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    
    - name: Build and push Docker image
      uses: azure/docker-login@v1
      with:
        login-server: myacr.azurecr.io
        username: ${{ secrets.ACR_USERNAME }}
        password: ${{ secrets.ACR_PASSWORD }}
    
    - run: |
        docker build . -t myacr.azurecr.io/devops-app:${{ github.sha }}
        docker push myacr.azurecr.io/devops-app:${{ github.sha }}
    
    - name: Deploy to Azure Web App
      uses: azure/webapps-deploy@v2
      with:
        app-name: 'devops-app'
        images: 'myacr.azurecr.io/devops-app:${{ github.sha }}'
```

### 2. Infrastructure as Code

**Azure Resource Manager (ARM) Templates:**
```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "appName": {
      "type": "string",
      "metadata": {
        "description": "Name of the application"
      }
    },
    "location": {
      "type": "string",
      "defaultValue": "[resourceGroup().location]",
      "metadata": {
        "description": "Location for all resources"
      }
    }
  },
  "variables": {
    "storageAccountName": "[concat(parameters('appName'), 'storage')]",
    "appServicePlanName": "[concat(parameters('appName'), '-plan')]"
  },
  "resources": [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2021-04-01",
      "name": "[variables('storageAccountName')]",
      "location": "[parameters('location')]",
      "sku": {
        "name": "Standard_LRS"
      },
      "kind": "StorageV2"
    },
    {
      "type": "Microsoft.Web/serverfarms",
      "apiVersion": "2021-02-01",
      "name": "[variables('appServicePlanName')]",
      "location": "[parameters('location')]",
      "sku": {
        "name": "B1",
        "tier": "Basic",
        "size": "B1",
        "family": "B",
        "capacity": 1
      }
    }
  ],
  "outputs": {
    "storageAccountName": {
      "type": "string",
      "value": "[variables('storageAccountName')]"
    }
  }
}
```

**Azure Bicep:**
```bicep
param appName string
param location string = resourceGroup().location

var storageAccountName = '${appName}storage'
var appServicePlanName = '${appName}-plan'

resource storageAccount 'Microsoft.Storage/storageAccounts@2021-04-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
}

resource appServicePlan 'Microsoft.Web/serverfarms@2021-02-01' = {
  name: appServicePlanName
  location: location
  sku: {
    name: 'B1'
    tier: 'Basic'
  }
}

output storageAccountName string = storageAccountName
```

### 3. Monitoring & Observability

**Azure Monitor:**
```json
{
  "type": "Microsoft.Insights/components",
  "apiVersion": "2020-02-02",
  "name": "devops-app-insights",
  "location": "[resourceGroup().location]",
  "properties": {
    "Application_Type": "web",
    "Flow_Type": "Bluefield",
    "Request_Source": "rest"
  }
}
```

**Application Insights:**
```csharp
// In your .NET application
using Microsoft.ApplicationInsights;

var telemetry = new TelemetryClient();
telemetry.TrackEvent("DevOps Operation");
telemetry.TrackMetric("ProcessingTime", processingTime);
telemetry.TrackException(ex);
```

### 4. Container Services

**Azure Kubernetes Service (AKS):**
```yaml
# AKS Cluster
apiVersion: containerservice.azure.com/v1alpha1
kind: ManagedCluster
metadata:
  name: devops-cluster
spec:
  location: eastus
  resourceGroup: devops-rg
  kubernetesVersion: 1.21.9
  dnsPrefix: devops-cluster
  agentPoolProfiles:
  - name: agentpool
    count: 3
    vmSize: Standard_DS2_v2
    osType: Linux
    mode: System
  networkProfile:
    networkPlugin: azure
    loadBalancerSku: standard
  identity:
    type: SystemAssigned

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
        image: myacr.azurecr.io/devops-app:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

**Azure Container Instances:**
```yaml
apiVersion: 2019-12-01
location: eastus
name: devops-container
properties:
  containers:
  - name: app
    properties:
      image: myacr.azurecr.io/devops-app:latest
      ports:
      - port: 80
        protocol: TCP
      resources:
        requests:
          cpu: 1.0
          memoryInGB: 1.5
  osType: Linux
  restartPolicy: Always
  ipAddress:
    type: Public
    ports:
    - port: 80
      protocol: TCP
tags: {}
type: Microsoft.ContainerInstance/containerGroups
```

### 5. Serverless Computing

**Azure Functions:**
```csharp
// C# Function
using System;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;

public static class DevOpsFunction
{
    [FunctionName("DevOpsFunction")]
    public static async Task<IActionResult> Run(
        [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post", Route = null)] HttpRequest req,
        ILogger log)
    {
        log.LogInformation("C# HTTP trigger function processed a request.");

        string name = req.Query["name"];

        string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
        dynamic data = JsonConvert.DeserializeObject(requestBody);
        name = name ?? data?.name;

        return name != null
            ? (ActionResult)new OkObjectResult($"Hello, {name}")
            : new BadRequestObjectResult("Please pass a name on the query string or in the request body");
    }
}
```

**Azure Logic Apps:**
```json
{
  "definition": {
    "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
    "actions": {
      "HTTP": {
        "type": "Http",
        "inputs": {
          "method": "GET",
          "uri": "https://api.github.com/repos/myorg/myrepo/issues"
        },
        "runAfter": {}
      },
      "Compose": {
        "type": "Compose",
        "inputs": "@length(body('HTTP'))",
        "runAfter": {
          "HTTP": ["Succeeded"]
        }
      }
    },
    "contentVersion": "1.0.0.0",
    "outputs": {},
    "parameters": {},
    "triggers": {
      "Recurrence": {
        "type": "Recurrence",
        "recurrence": {
          "frequency": "Hour",
          "interval": 1
        }
      }
    }
  }
}
```

### 6. Security in DevOps

**Azure Security Center:**
```json
{
  "type": "Microsoft.Security/securityContacts",
  "apiVersion": "2017-08-01-preview",
  "name": "default",
  "properties": {
    "email": "security@mycompany.com",
    "phone": "+1-555-555-5555",
    "alertNotifications": "On",
    "alertsToAdmins": "On"
  }
}
```

**Azure Key Vault:**
```json
{
  "type": "Microsoft.KeyVault/vaults",
  "apiVersion": "2021-04-01-preview",
  "name": "devops-keyvault",
  "location": "[resourceGroup().location]",
  "properties": {
    "sku": {
      "family": "A",
      "name": "standard"
    },
    "tenantId": "[subscription().tenantId]",
    "accessPolicies": [
      {
        "tenantId": "[subscription().tenantId]",
        "objectId": "[reference(resourceId('Microsoft.ManagedIdentity/userAssignedIdentities', 'devops-identity'), '2018-11-30').principalId]",
        "permissions": {
          "keys": ["get", "list"],
          "secrets": ["get", "list"],
          "certificates": ["get", "list"]
        }
      }
    ],
    "enabledForDeployment": true,
    "enabledForTemplateDeployment": true,
    "enabledForDiskEncryption": true
  }
}
```

Azure cung cấp giải pháp cloud toàn diện với khả năng hybrid cloud mạnh mẽ và tích hợp sâu với Microsoft ecosystem!
