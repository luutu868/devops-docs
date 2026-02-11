# AWS Basics - Amazon Web Services

## Giới Thiệu AWS

Amazon Web Services (AWS) là nền tảng điện toán đám mây toàn diện và được sử dụng rộng rãi nhất thế giới, cung cấp hơn 200 dịch vụ từ các data centers trên toàn cầu.

**Ưu điểm AWS:**
```yaml
- Phổ biến nhất: Market leader với 32% thị phần
- Dịch vụ đa dạng: 200+ dịch vụ covering mọi use case
- Global infrastructure: 30+ regions, 96+ availability zones
- Mature ecosystem: Tools, documentation, community lớn
- Enterprise ready: Compliance, security, support tốt
```

## AWS Global Infrastructure

### Regions và Availability Zones

```
Region (us-east-1)
├── Availability Zone A (us-east-1a)
│   └── Data Centers (1 or more)
├── Availability Zone B (us-east-1b)
│   └── Data Centers (1 or more)
└── Availability Zone C (us-east-1c)
    └── Data Centers (1 or more)
```

**Regions:**
- Geographic location với multiple AZs
- Complete copy của AWS services
- Isolated từ các regions khác
- Chọn region based on: latency, cost, compliance

**Availability Zones (AZ):**
- 1+ data centers trong region
- Isolated infrastructure (power, cooling, network)
- Connected via high-bandwidth, low-latency network
- Multi-AZ deployment for high availability

**Edge Locations:**
- Content delivery network (CloudFront)
- 400+ edge locations worldwide
- Caching content gần users

### Region Selection Criteria

```yaml
Factors:
  Latency:
    - Geographic proximity to users
    - Network latency testing
  
  Cost:
    - Pricing varies by region
    - us-east-1 typically cheapest
    - EU regions more expensive
  
  Compliance:
    - Data residency requirements
    - GDPR, HIPAA, PCI-DSS
    - Government regulations
  
  Service Availability:
    - Not all services in all regions
    - New services launch in us-east-1 first
    - Check service availability
```

## IAM - Identity and Access Management

### IAM Core Concepts

**Users:**
```json
{
  "User": "john.doe",
  "Type": "Individual person",
  "Credentials": [
    "Console password",
    "Access keys (programmatic)"
  ],
  "MFA": "Enabled"
}
```

**Groups:**
```
Developers Group
├── User: alice
├── User: bob
└── User: charlie
└── Policy: DevelopersPolicy (Read/Write S3, EC2)

Admins Group
├── User: admin1
└── User: admin2
└── Policy: AdministratorAccess
```

**Roles:**
```json
{
  "Role": "EC2-S3-Access-Role",
  "Type": "AWS Service Role",
  "TrustPolicy": "EC2 service can assume this role",
  "Permissions": "Read/Write S3 buckets",
  "Use Case": "EC2 instance needs S3 access without hardcoded keys"
}
```

**Policies:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/*"
    },
    {
      "Effect": "Deny",
      "Action": "s3:DeleteObject",
      "Resource": "*"
    }
  ]
}
```

### IAM Best Practices

**1. Root Account:**
```bash
# Never use root account for daily tasks
# Enable MFA on root account
# Lock away root credentials
# Use root only for:
#   - Account settings
#   - Billing
#   - Close account
```

**2. Least Privilege:**
```json
// Bad - Too permissive
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}

// Good - Specific permissions
{
  "Effect": "Allow",
  "Action": [
    "s3:GetObject",
    "s3:ListBucket"
  ],
  "Resource": [
    "arn:aws:s3:::my-app-bucket",
    "arn:aws:s3:::my-app-bucket/*"
  ]
}
```

**3. Use Roles Instead of Keys:**
```bash
# Bad - Hardcoded credentials
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

# Good - Use IAM role
# Attach role to EC2, Lambda, ECS
# No credentials in code
# Automatic rotation
```

**4. Enable MFA:**
```bash
# Virtual MFA (Google Authenticator, Authy)
# Hardware MFA (YubiKey, Gemalto)
# SMS MFA (not recommended)

# Enable MFA for:
# - Root account (required)
# - Admin users (required)
# - All users (recommended)
```

**5. Password Policy:**
```yaml
Password Requirements:
  MinimumLength: 14
  RequireUppercase: true
  RequireLowercase: true
  RequireNumbers: true
  RequireSymbols: true
  ExpirePasswords: 90 days
  PreventReuse: 24 passwords
  AllowUsersToChange: true
```

### IAM CLI Commands

```bash
# List users
aws iam list-users

# Create user
aws iam create-user --user-name john.doe

# Create access key
aws iam create-access-key --user-name john.doe

# List policies attached to user
aws iam list-attached-user-policies --user-name john.doe

# Attach policy to user
aws iam attach-user-policy \
  --user-name john.doe \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess

# Create group
aws iam create-group --group-name Developers

# Add user to group
aws iam add-user-to-group \
  --user-name john.doe \
  --group-name Developers

# Create role
aws iam create-role \
  --role-name EC2-S3-Role \
  --assume-role-policy-document file://trust-policy.json

# Attach policy to role
aws iam attach-role-policy \
  --role-name EC2-S3-Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

## EC2 - Elastic Compute Cloud

### EC2 Instance Types

```
General Purpose (T3, M5, M6i)
├── Use: Web servers, app servers, dev environments
├── Balance: CPU, memory, network
└── Example: t3.medium, m5.large

Compute Optimized (C5, C6i)
├── Use: High-performance computing, batch processing
├── Focus: High CPU performance
└── Example: c5.xlarge, c6i.2xlarge

Memory Optimized (R5, R6i, X2)
├── Use: Databases, caching, real-time analytics
├── Focus: High memory-to-CPU ratio
└── Example: r5.large, r6i.xlarge

Storage Optimized (I3, D2, H1)
├── Use: Data warehousing, Hadoop, log processing
├── Focus: High disk I/O
└── Example: i3.large, d2.xlarge

GPU Instances (P3, P4, G4)
├── Use: Machine learning, graphics rendering
├── Focus: GPU processing
└── Example: p3.2xlarge, g4dn.xlarge
```

### EC2 Pricing Models

**1. On-Demand:**
```yaml
Pricing:
  - Pay per hour/second
  - No commitment
  - Most expensive
  
Use Case:
  - Short-term workloads
  - Unpredictable workloads
  - Development/testing
  
Example:
  t3.medium: $0.0416/hour
  m5.large: $0.096/hour
```

**2. Reserved Instances:**
```yaml
Pricing:
  - 1 or 3 year commitment
  - Up to 75% discount
  - Payment: All Upfront, Partial Upfront, No Upfront
  
Types:
  Standard RI:
    - Fixed instance type, region
    - Maximum discount (up to 75%)
  
  Convertible RI:
    - Can change instance type
    - Lower discount (up to 54%)
  
Use Case:
  - Steady-state workloads
  - Production databases
  - Long-running applications
```

**3. Spot Instances:**
```yaml
Pricing:
  - Up to 90% discount
  - Can be terminated by AWS
  - Bid on unused capacity
  
Use Case:
  - Fault-tolerant workloads
  - Batch processing
  - Big data analysis
  - CI/CD workers
  
Example:
  t3.medium On-Demand: $0.0416/hour
  t3.medium Spot: $0.0125/hour (70% savings)
```

**4. Savings Plans:**
```yaml
Pricing:
  - 1 or 3 year commitment
  - Commit to $/hour spend
  - Up to 72% discount
  - More flexible than Reserved
  
Types:
  Compute Savings Plan:
    - Any instance family, size, region, OS
    - Maximum flexibility
  
  EC2 Instance Savings Plan:
    - Specific instance family + region
    - Higher discount
```

### Launch EC2 Instance

```bash
# Create key pair
aws ec2 create-key-pair \
  --key-name MyKeyPair \
  --query 'KeyMaterial' \
  --output text > MyKeyPair.pem
chmod 400 MyKeyPair.pem

# Create security group
aws ec2 create-security-group \
  --group-name MySecurityGroup \
  --description "My security group" \
  --vpc-id vpc-1234567890abcdef0

# Add SSH rule
aws ec2 authorize-security-group-ingress \
  --group-id sg-0123456789abcdef0 \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0

# Add HTTP rule
aws ec2 authorize-security-group-ingress \
  --group-id sg-0123456789abcdef0 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# Launch instance
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t3.micro \
  --key-name MyKeyPair \
  --security-group-ids sg-0123456789abcdef0 \
  --subnet-id subnet-0123456789abcdef0 \
  --user-data file://userdata.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=WebServer}]'

# List instances
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=WebServer" \
  --query 'Reservations[].Instances[].[InstanceId,State.Name,PublicIpAddress]' \
  --output table

# Stop instance
aws ec2 stop-instances --instance-ids i-1234567890abcdef0

# Start instance
aws ec2 start-instances --instance-ids i-1234567890abcdef0

# Terminate instance
aws ec2 terminate-instances --instance-ids i-1234567890abcdef0
```

**User Data Script:**
```bash
#!/bin/bash
# userdata.sh

# Update system
yum update -y

# Install Apache
yum install -y httpd

# Start Apache
systemctl start httpd
systemctl enable httpd

# Create simple webpage
echo "<h1>Hello from $(hostname)</h1>" > /var/www/html/index.html
```

## S3 - Simple Storage Service

### S3 Concepts

```
S3 Bucket
├── Globally unique name
├── Region-specific
├── Objects (files)
│   ├── Key (filename/path)
│   ├── Value (content)
│   ├── Metadata
│   └── Version ID
└── Properties
    ├── Versioning
    ├── Encryption
    ├── Lifecycle policies
    └── Access control
```

**S3 Storage Classes:**
```yaml
S3 Standard:
  Durability: 99.999999999% (11 9's)
  Availability: 99.99%
  Use: Frequently accessed data
  Cost: Highest storage cost

S3 Intelligent-Tiering:
  Automatic: Moves between tiers based on access
  Use: Unknown or changing access patterns
  Cost: Small monitoring fee

S3 Standard-IA (Infrequent Access):
  Availability: 99.9%
  Use: Infrequently accessed, rapid access needed
  Cost: Lower storage, retrieval fee

S3 One Zone-IA:
  Availability: 99.5% (single AZ)
  Use: Reproductible data, infrequent access
  Cost: 20% cheaper than Standard-IA

S3 Glacier Instant Retrieval:
  Retrieval: Milliseconds
  Use: Archive with instant access
  Cost: Lower storage cost

S3 Glacier Flexible Retrieval:
  Retrieval: Minutes to hours
  Use: Archive, backup
  Cost: Very low storage cost

S3 Glacier Deep Archive:
  Retrieval: 12-48 hours
  Use: Long-term archive, compliance
  Cost: Lowest storage cost
```

### S3 CLI Commands

```bash
# Create bucket
aws s3 mb s3://my-unique-bucket-name --region us-east-1

# List buckets
aws s3 ls

# Upload file
aws s3 cp myfile.txt s3://my-bucket/

# Upload directory
aws s3 cp mydir/ s3://my-bucket/mydir/ --recursive

# Download file
aws s3 cp s3://my-bucket/myfile.txt ./

# Sync directory (upload only changed files)
aws s3 sync mydir/ s3://my-bucket/mydir/

# List objects in bucket
aws s3 ls s3://my-bucket/
aws s3 ls s3://my-bucket/mydir/ --recursive

# Delete object
aws s3 rm s3://my-bucket/myfile.txt

# Delete all objects
aws s3 rm s3://my-bucket/ --recursive

# Delete bucket
aws s3 rb s3://my-bucket --force

# Set bucket public access (NOT recommended)
aws s3api put-public-access-block \
  --bucket my-bucket \
  --public-access-block-configuration \
    "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket my-bucket \
  --versioning-configuration Status=Enabled

# Enable encryption
aws s3api put-bucket-encryption \
  --bucket my-bucket \
  --server-side-encryption-configuration \
    '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
```

### S3 Bucket Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/*"
    },
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    },
    {
      "Sid": "AllowSpecificAccount",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:root"
      },
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/shared/*"
    }
  ]
}
```

## RDS - Relational Database Service

### RDS Engines

```yaml
Amazon Aurora:
  - AWS proprietary database
  - MySQL and PostgreSQL compatible
  - 5x faster than MySQL, 3x faster than PostgreSQL
  - Auto-scaling storage
  - Up to 15 read replicas
  - Use: High-performance applications

MySQL:
  - Popular open-source database
  - Community and Commercial editions
  - Versions: 5.7, 8.0
  - Use: Web applications, general purpose

PostgreSQL:
  - Advanced open-source database
  - ACID compliant
  - Extensions support
  - Use: Complex queries, data integrity

MariaDB:
  - MySQL fork
  - Improved performance
  - Open-source
  - Use: MySQL replacement

Oracle:
  - Enterprise database
  - BYOL (Bring Your Own License)
  - Use: Enterprise applications

SQL Server:
  - Microsoft database
  - Express, Web, Standard, Enterprise
  - Use: .NET applications
```

### RDS Multi-AZ vs Read Replicas

**Multi-AZ Deployment:**
```
Primary Instance (us-east-1a)
├── Synchronous replication
└── Standby Instance (us-east-1b)
    └── Automatic failover (60-120 seconds)

Purpose: High Availability
Replication: Synchronous
Failover: Automatic
Read Traffic: No (standby not accessible)
Cost: 2x instance cost
```

**Read Replicas:**
```
Primary Instance (us-east-1a)
├── Asynchronous replication
├── Read Replica 1 (us-east-1b)
├── Read Replica 2 (us-east-1c)
└── Read Replica 3 (us-west-2)

Purpose: Scalability, Read Performance
Replication: Asynchronous
Failover: Manual promotion
Read Traffic: Yes
Cost: Per replica
```

### Create RDS Instance

```bash
# Create DB subnet group
aws rds create-db-subnet-group \
  --db-subnet-group-name mydbsubnetgroup \
  --db-subnet-group-description "My DB subnet group" \
  --subnet-ids subnet-12345678 subnet-87654321

# Create RDS instance
aws rds create-db-instance \
  --db-instance-identifier mydbinstance \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --engine-version 8.0.32 \
  --master-username admin \
  --master-user-password MyPassword123! \
  --allocated-storage 20 \
  --storage-type gp3 \
  --storage-encrypted \
  --vpc-security-group-ids sg-0123456789abcdef0 \
  --db-subnet-group-name mydbsubnetgroup \
  --multi-az \
  --backup-retention-period 7 \
  --preferred-backup-window "03:00-04:00" \
  --preferred-maintenance-window "sun:04:00-sun:05:00" \
  --enable-cloudwatch-logs-exports '["error","general","slowquery"]' \
  --deletion-protection

# List RDS instances
aws rds describe-db-instances \
  --query 'DBInstances[].[DBInstanceIdentifier,DBInstanceStatus,Endpoint.Address]' \
  --output table

# Create read replica
aws rds create-db-instance-read-replica \
  --db-instance-identifier mydbinstance-replica \
  --source-db-instance-identifier mydbinstance \
  --db-instance-class db.t3.micro

# Create snapshot
aws rds create-db-snapshot \
  --db-instance-identifier mydbinstance \
  --db-snapshot-identifier mydbinstance-snapshot-$(date +%Y%m%d)

# Restore from snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier mydbinstance-restored \
  --db-snapshot-identifier mydbinstance-snapshot-20240211

# Delete RDS instance
aws rds delete-db-instance \
  --db-instance-identifier mydbinstance \
  --skip-final-snapshot
  # or with final snapshot:
  # --final-db-snapshot-identifier mydbinstance-final-snapshot
```

## VPC - Virtual Private Cloud

### VPC Components

```
VPC (10.0.0.0/16)
├── Internet Gateway (IGW)
├── NAT Gateway (in public subnet)
├── Public Subnets
│   ├── Public Subnet A (10.0.1.0/24) - us-east-1a
│   └── Public Subnet B (10.0.2.0/24) - us-east-1b
│       └── Route: 0.0.0.0/0 → IGW
├── Private Subnets
│   ├── Private Subnet A (10.0.11.0/24) - us-east-1a
│   └── Private Subnet B (10.0.12.0/24) - us-east-1b
│       └── Route: 0.0.0.0/0 → NAT Gateway
├── Security Groups (Stateful firewall)
│   ├── Web SG (Allow 80, 443)
│   └── DB SG (Allow 3306 from Web SG)
└── Network ACLs (Stateless firewall)
```

**Security Group vs NACL:**
```yaml
Security Group:
  Level: Instance level
  State: Stateful (return traffic automatic)
  Rules: Allow rules only
  Evaluation: All rules evaluated
  Default: Deny all inbound, allow all outbound
  
Network ACL:
  Level: Subnet level
  State: Stateless (return traffic must be explicitly allowed)
  Rules: Allow and Deny rules
  Evaluation: Rules in order (lowest number first)
  Default: Allow all inbound and outbound
```

### Create VPC

```bash
# Create VPC
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=MyVPC}]'

# Enable DNS hostnames
aws ec2 modify-vpc-attribute \
  --vpc-id vpc-0123456789abcdef0 \
  --enable-dns-hostnames

# Create Internet Gateway
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=MyIGW}]'

# Attach IGW to VPC
aws ec2 attach-internet-gateway \
  --internet-gateway-id igw-0123456789abcdef0 \
  --vpc-id vpc-0123456789abcdef0

# Create public subnet
aws ec2 create-subnet \
  --vpc-id vpc-0123456789abcdef0 \
  --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=PublicSubnetA}]'

# Create private subnet
aws ec2 create-subnet \
  --vpc-id vpc-0123456789abcdef0 \
  --cidr-block 10.0.11.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=PrivateSubnetA}]'

# Create route table for public subnet
aws ec2 create-route-table \
  --vpc-id vpc-0123456789abcdef0 \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=PublicRT}]'

# Add route to IGW
aws ec2 create-route \
  --route-table-id rtb-0123456789abcdef0 \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id igw-0123456789abcdef0

# Associate route table with subnet
aws ec2 associate-route-table \
  --route-table-id rtb-0123456789abcdef0 \
  --subnet-id subnet-0123456789abcdef0

# Create NAT Gateway (requires Elastic IP)
aws ec2 allocate-address --domain vpc
aws ec2 create-nat-gateway \
  --subnet-id subnet-0123456789abcdef0 \
  --allocation-id eipalloc-0123456789abcdef0

# Create route table for private subnet
aws ec2 create-route-table \
  --vpc-id vpc-0123456789abcdef0 \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=PrivateRT}]'

# Add route to NAT Gateway
aws ec2 create-route \
  --route-table-id rtb-9876543210abcdef0 \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id nat-0123456789abcdef0
```

## AWS Best Practices

### Cost Optimization

```yaml
1. Right-Sizing:
   - Start small, scale up as needed
   - Use CloudWatch metrics to monitor usage
   - Resize instances based on actual usage
   
2. Reserved Instances/Savings Plans:
   - Purchase for steady-state workloads
   - 1-3 year commitment
   - Up to 75% savings

3. Spot Instances:
   - Use for fault-tolerant workloads
   - Up to 90% savings
   - Batch processing, CI/CD, dev/test

4. Auto Scaling:
   - Scale resources based on demand
   - Scale down during off-hours
   - Save costs on idle resources

5. S3 Lifecycle Policies:
   - Move old data to cheaper storage
   - IA → Glacier → Deep Archive
   - Delete old backups automatically

6. Delete Unused Resources:
   - Unattached EBS volumes
   - Old snapshots
   - Idle load balancers
   - Unused Elastic IPs

7. Cost Monitoring:
   - Enable AWS Cost Explorer
   - Set up billing alarms
   - Use AWS Budgets
   - Tag resources for cost allocation
```

### Security Best Practices

```yaml
1. Identity:
   - Use IAM roles instead of access keys
   - Enable MFA for all users
   - Rotate credentials regularly
   - Follow least privilege principle

2. Network:
   - Use VPC for network isolation
   - Use security groups (deny by default)
   - Enable VPC Flow Logs
   - Use AWS WAF for web applications

3. Data:
   - Encrypt data at rest (S3, EBS, RDS)
   - Encrypt data in transit (HTTPS, TLS)
   - Use AWS KMS for key management
   - Enable S3 bucket versioning

4. Monitoring:
   - Enable CloudTrail (API logging)
   - Use CloudWatch for monitoring
   - Set up security alarms
   - Use AWS Config for compliance

5. Compliance:
   - Enable AWS Config Rules
   - Use AWS Security Hub
   - Run AWS Inspector scans
   - Regular security audits
```

### High Availability Patterns

```yaml
Multi-AZ Deployment:
  Application:
    - EC2 instances in multiple AZs
    - Auto Scaling Group across AZs
    - Elastic Load Balancer
  
  Database:
    - RDS Multi-AZ
    - Aurora with multiple AZs
    - DynamoDB (multi-AZ by default)
  
  Storage:
    - S3 (99.999999999% durability)
    - EFS (multi-AZ by default)

Multi-Region Deployment:
  Active-Passive:
    - Primary region serves traffic
    - Secondary region for disaster recovery
    - Route53 healthchecks for failover
  
  Active-Active:
    - Both regions serve traffic
    - Route53 for geo-routing
    - Data replication between regions
```

## DevOps on AWS

### 1. CI/CD with AWS Developer Tools

**AWS CodePipeline:**
```yaml
# AWS CodePipeline for CI/CD
Resources:
  MyPipeline:
    Type: AWS::CodePipeline::Pipeline
    Properties:
      Name: my-app-pipeline
      RoleArn: !GetAtt PipelineRole.Arn
      Stages:
        - Name: Source
          Actions:
            - Name: SourceAction
              ActionTypeId:
                Category: Source
                Owner: AWS
                Provider: S3
                Version: '1'
              Configuration:
                S3Bucket: my-source-bucket
                S3ObjectKey: source.zip
              OutputArtifacts:
                - Name: SourceArtifact
        - Name: Build
          Actions:
            - Name: BuildAction
              ActionTypeId:
                Category: Build
                Owner: AWS
                Provider: CodeBuild
                Version: '1'
              Configuration:
                ProjectName: !Ref MyCodeBuildProject
              InputArtifacts:
                - Name: SourceArtifact
              OutputArtifacts:
                - Name: BuildArtifact
        - Name: Deploy
          Actions:
            - Name: DeployAction
              ActionTypeId:
                Category: Deploy
                Owner: AWS
                Provider: CloudFormation
                Version: '1'
              Configuration:
                ActionMode: CREATE_UPDATE
                StackName: my-app-stack
                TemplatePath: BuildArtifact::template.yaml
              InputArtifacts:
                - Name: BuildArtifact
```

**AWS CodeBuild:**
```yaml
# buildspec.yml
version: 0.2

phases:
  install:
    runtime-versions:
      python: 3.8
    commands:
      - pip install -r requirements.txt
  build:
    commands:
      - python -m pytest tests/
      - sam build
  post_build:
    commands:
      - sam package --output-template-file packaged.yaml --s3-bucket my-artifacts-bucket

artifacts:
  type: zip
  files:
    - packaged.yaml
    - template.yaml
```

### 2. Infrastructure as Code

**AWS CloudFormation:**
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'DevOps Infrastructure Stack'

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsHostnames: true
      EnableDnsSupport: true

  InternetGateway:
    Type: AWS::EC2::InternetGateway

  VPCGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
      MapPublicIpOnLaunch: true

  ECSCluster:
    Type: AWS::ECS::Cluster
    Properties:
      ClusterName: devops-cluster

  ECSTaskDefinition:
    Type: AWS::ECS::TaskDefinition
    Properties:
      Family: devops-app
      Cpu: 256
      Memory: 512
      NetworkMode: awsvpc
      RequiresCompatibilities:
        - FARGATE
      ExecutionRoleArn: !Ref ECSExecutionRole
      ContainerDefinitions:
        - Name: app
          Image: nginx:latest
          Essential: true
          PortMappings:
            - ContainerPort: 80
              Protocol: tcp
```

**AWS CDK (Cloud Development Kit):**
```typescript
import * as cdk from '@aws-cdk/core';
import * as ec2 from '@aws-cdk/aws-ec2';
import * as ecs from '@aws-cdk/aws-ecs';
import * as ecs_patterns from '@aws-cdk/aws-ecs-patterns';

export class DevOpsStack extends cdk.Stack {
  constructor(scope: cdk.Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // VPC
    const vpc = new ec2.Vpc(this, 'DevOpsVPC', {
      maxAzs: 2
    });

    // ECS Cluster
    const cluster = new ecs.Cluster(this, 'DevOpsCluster', {
      vpc
    });

    // Fargate Service
    new ecs_patterns.ApplicationLoadBalancedFargateService(this, 'DevOpsService', {
      cluster,
      taskImageOptions: {
        image: ecs.ContainerImage.fromRegistry('nginx:latest'),
      },
      publicLoadBalancer: true
    });
  }
}
```

### 3. Monitoring & Observability

**AWS CloudWatch:**
```yaml
# CloudWatch Dashboard
Resources:
  DevOpsDashboard:
    Type: AWS::CloudWatch::Dashboard
    Properties:
      DashboardName: devops-dashboard
      DashboardBody: !Sub |
        {
          "widgets": [
            {
              "type": "metric",
              "x": 0,
              "y": 0,
              "width": 12,
              "height": 6,
              "properties": {
                "metrics": [
                  ["AWS/EC2", "CPUUtilization", "InstanceId", "${EC2Instance}"]
                ],
                "view": "timeSeries",
                "stacked": false,
                "region": "${AWS::Region}",
                "title": "EC2 CPU Utilization",
                "period": 300
              }
            }
          ]
        }
```

**AWS X-Ray:**
```yaml
# X-Ray configuration
Resources:
  XRaySamplingRule:
    Type: AWS::XRay::SamplingRule
    Properties:
      SamplingRule:
        RuleName: devops-sampling
        ResourceARN: "*"
        Priority: 1
        FixedRate: 0.1
        ReservoirSize: 1
        ServiceName: "*"
        ServiceType: "*"
        Host: "*"
        HTTPMethod: "*"
        URLPath: "*"
        Version: 1
```

### 4. Container Services

**Amazon ECS (Elastic Container Service):**
```yaml
# ECS Cluster with Fargate
Resources:
  ECSCluster:
    Type: AWS::ECS::Cluster
    Properties:
      ClusterName: devops-cluster
      CapacityProviders:
        - FARGATE
        - FARGATE_SPOT

  ECSTaskDefinition:
    Type: AWS::ECS::TaskDefinition
    Properties:
      Family: devops-app
      Cpu: 256
      Memory: 512
      NetworkMode: awsvpc
      RequiresCompatibilities:
        - FARGATE
      ExecutionRoleArn: !Ref ECSExecutionRole
      ContainerDefinitions:
        - Name: app
          Image: nginx:latest
          Essential: true
          PortMappings:
            - ContainerPort: 80
            - HostPort: 80
            - Protocol: tcp
          LogConfiguration:
            LogDriver: awslogs
            Options:
              awslogs-group: !Ref LogGroup
              awslogs-region: !Ref AWS::Region
              awslogs-stream-prefix: ecs

  ECSService:
    Type: AWS::ECS::Service
    Properties:
      ServiceName: devops-service
      Cluster: !Ref ECSCluster
      TaskDefinition: !Ref ECSTaskDefinition
      DesiredCount: 2
      LaunchType: FARGATE
      NetworkConfiguration:
        AwsvpcConfiguration:
          Subnets:
            - !Ref PublicSubnet
          SecurityGroups:
            - !Ref ECSServiceSecurityGroup
          AssignPublicIp: ENABLED
```

**Amazon EKS (Elastic Kubernetes Service):**
```yaml
# EKS Cluster
Resources:
  EKSCluster:
    Type: AWS::EKS::Cluster
    Properties:
      Name: devops-cluster
      Version: '1.21'
      RoleArn: !GetAtt EKSServiceRole.Arn
      ResourcesVpcConfig:
        SubnetIds:
          - !Ref PublicSubnet1
          - !Ref PublicSubnet2
        SecurityGroupIds:
          - !Ref EKSClusterSecurityGroup

  EKSNodeGroup:
    Type: AWS::EKS::Nodegroup
    Properties:
      ClusterName: !Ref EKSCluster
      NodegroupName: devops-nodes
      NodeRole: !GetAtt EKSNodeRole.Arn
      Subnets:
        - !Ref PublicSubnet1
        - !Ref PublicSubnet2
      InstanceTypes:
        - t3.medium
      ScalingConfig:
        MinSize: 1
        MaxSize: 10
        DesiredSize: 2
```

### 5. Serverless Computing

**AWS Lambda:**
```python
# lambda_function.py
import json
import boto3
import os

def lambda_handler(event, context):
    # Get environment variables
    table_name = os.environ['DYNAMODB_TABLE']
    
    # Initialize DynamoDB client
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table(table_name)
    
    # Process the event
    if event['httpMethod'] == 'POST':
        # Create item
        item = json.loads(event['body'])
        table.put_item(Item=item)
        return {
            'statusCode': 201,
            'body': json.dumps({'message': 'Item created'})
        }
    elif event['httpMethod'] == 'GET':
        # Get item
        key = event['pathParameters']['id']
        response = table.get_item(Key={'id': key})
        if 'Item' in response:
            return {
                'statusCode': 200,
                'body': json.dumps(response['Item'])
            }
        else:
            return {
                'statusCode': 404,
                'body': json.dumps({'error': 'Item not found'})
            }
```

**API Gateway + Lambda:**
```yaml
Resources:
  ApiGateway:
    Type: AWS::ApiGateway::RestApi
    Properties:
      Name: devops-api
      Description: DevOps API Gateway

  ApiResource:
    Type: AWS::ApiGateway::Resource
    Properties:
      RestApiId: !Ref ApiGateway
      ParentId: !GetAtt ApiGateway.RootResourceId
      PathPart: 'items'

  ApiMethod:
    Type: AWS::ApiGateway::Method
    Properties:
      RestApiId: !Ref ApiGateway
      ResourceId: !Ref ApiResource
      HttpMethod: POST
      AuthorizationType: NONE
      Integration:
        Type: AWS_PROXY
        IntegrationHttpMethod: POST
        Uri: !Sub arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${LambdaFunction.Arn}/invocations

  ApiDeployment:
    Type: AWS::ApiGateway::Deployment
    DependsOn: ApiMethod
    Properties:
      RestApiId: !Ref ApiGateway
      StageName: prod
```

### 6. Security in DevOps

**AWS Security Services:**
```yaml
# AWS Config Rules
Resources:
  S3BucketVersioningEnabled:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: s3-bucket-versioning-enabled
      Description: Checks that S3 buckets have versioning enabled
      Source:
        Owner: AWS
        SourceIdentifier: S3_BUCKET_VERSIONING_ENABLED
      Scope:
        ComplianceResourceTypes:
          - AWS::S3::Bucket

  SecurityGroupNoUnrestrictedSSH:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: security-group-no-unrestricted-ssh
      Description: Checks that security groups do not allow unrestricted SSH access
      Source:
        Owner: AWS
        SourceIdentifier: SECURITY_GROUP_NO_UNRESTRICTED_SSH
```

**AWS Systems Manager:**
```yaml
# SSM Parameter Store
Resources:
  DatabasePassword:
    Type: AWS::SSM::Parameter
    Properties:
      Name: /devops/database/password
      Type: SecureString
      Value: !Ref DatabasePassword
      Description: Database password for devops environment

  AppConfig:
    Type: AWS::SSM::Parameter
    Properties:
      Name: /devops/app/config
      Type: String
      Value: !Sub |
        {
          "database": {
            "host": "${RDSInstance.Endpoint.Address}",
            "port": "${RDSInstance.Endpoint.Port}",
            "name": "devopsdb"
          },
          "cache": {
            "host": "${ElastiCacheCluster.RedisEndpoint.Address}",
            "port": "${ElastiCacheCluster.RedisEndpoint.Port}"
          }
        }
```

AWS cung cấp nền tảng cloud toàn diện với infrastructure mature và ecosystem phong phú cho mọi nhu cầu DevOps!
