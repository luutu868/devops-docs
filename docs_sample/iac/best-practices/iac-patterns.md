# Infrastructure as Code Best Practices

## IaC Principles

### Core Principles

**1. Declarative over Imperative**
```yaml
# Declarative (good)
resource "aws_instance" "web" {
  ami           = "ami-123456"
  instance_type = "t2.micro"
}

# vs Imperative (avoid)
# Run command to create instance
# Run command to modify instance
# Run command to tag instance
```

**2. Immutable Infrastructure**
```
Replace servers instead of modifying them

Don't:
- SSH into server
- Manually update config
- Apply patches directly

Do:
- Build new server image
- Deploy new version
- Terminate old servers
```

**3. Version Everything**
```bash
# Infrastructure code
git add terraform/
git commit -m "Add production VPC"
git push

# Configuration
git add ansible/
git commit -m "Update nginx config"

# Documentation
git add docs/
git commit -m "Document network architecture"
```

**4. Single Source of Truth**
```
One repository → One infrastructure state

├── terraform/          # All infrastructure
├── ansible/           # All configuration
├── scripts/           # All automation
└── docs/              # All documentation
```

## Project Structure

### Terraform Structure

**Small project:**
```
terraform/
├── main.tf            # Main resources
├── variables.tf       # Input variables
├── outputs.tf         # Output values
├── versions.tf        # Provider versions
├── terraform.tfvars   # Variable values
└── README.md
```

**Medium project:**
```
terraform/
├── modules/
│   ├── vpc/
│   ├── compute/
│   └── database/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   └── production/
└── README.md
```

**Large project:**
```
infrastructure/
├── terraform/
│   ├── modules/
│   │   ├── networking/
│   │   │   ├── vpc/
│   │   │   ├── subnets/
│   │   │   └── security-groups/
│   │   ├── compute/
│   │   │   ├── ec2/
│   │   │   ├── asg/
│   │   │   └── lambda/
│   │   ├── storage/
│   │   │   ├── s3/
│   │   │   ├── efs/
│   │   │   └── ebs/
│   │   └── database/
│   │       ├── rds/
│   │       ├── dynamodb/
│   │       └── elasticache/
│   ├── environments/
│   │   ├── dev/
│   │   ├── staging/
│   │   └── production/
│   ├── global/
│   │   ├── iam/
│   │   ├── route53/
│   │   └── s3-backend/
│   └── live/
│       ├── us-east-1/
│       └── eu-west-1/
├── ansible/
│   ├── inventories/
│   ├── playbooks/
│   ├── roles/
│   └── group_vars/
├── scripts/
│   ├── deploy.sh
│   └── backup.sh
├── docs/
│   ├── architecture.md
│   └── runbooks.md
└── .github/
    └── workflows/
        ├── terraform.yml
        └── ansible.yml
```

### Ansible Structure

```
ansible/
├── inventories/
│   ├── production/
│   │   ├── hosts
│   │   └── group_vars/
│   │       ├── all.yml
│   │       ├── webservers.yml
│   │       └── databases.yml
│   └── staging/
│       ├── hosts
│       └── group_vars/
├── roles/
│   ├── common/
│   │   ├── tasks/
│   │   ├── handlers/
│   │   ├── templates/
│   │   ├── files/
│   │   ├── vars/
│   │   └── defaults/
│   ├── nginx/
│   ├── postgresql/
│   └── monitoring/
├── playbooks/
│   ├── site.yml
│   ├── webservers.yml
│   └── databases.yml
├── group_vars/
│   ├── all.yml
│   └── production.yml
├── host_vars/
│   └── web01.yml
├── ansible.cfg
└── requirements.yml
```

## Naming Conventions

### Resource Naming

**Pattern:** `<project>-<environment>-<resource>-<description>`

```hcl
# Good
resource "aws_instance" "myapp_prod_web_server_01" {}
resource "aws_s3_bucket" "myapp_staging_assets" {}
resource "aws_rds_instance" "myapp_prod_db_primary" {}

# Bad
resource "aws_instance" "server1" {}
resource "aws_s3_bucket" "bucket" {}
resource "aws_rds_instance" "database" {}
```

**Tags:**
```hcl
tags = {
  Name        = "myapp-prod-web-01"
  Environment = "production"
  Project     = "myapp"
  ManagedBy   = "terraform"
  CostCenter  = "engineering"
  Owner       = "john@company.com"
  Backup      = "daily"
}
```

### Variable Naming

```hcl
# Good
variable "instance_type" {}
variable "vpc_cidr_block" {}
variable "enable_monitoring" {}
variable "db_allocated_storage" {}

# Bad
variable "type" {}
variable "cidr" {}
variable "mon" {}
variable "storage" {}
```

### Module Naming

```
modules/
├── vpc-standard/         # Descriptive
├── rds-mysql/           # Technology specific
├── eks-cluster/         # Service specific
└── s3-static-website/   # Purpose specific

# Avoid
modules/
├── infra/
├── compute/
└── data/
```

## Environment Management

### Strategy 1: Separate Directories

```
environments/
├── dev/
│   ├── main.tf
│   ├── variables.tf
│   └── terraform.tfvars
├── staging/
│   ├── main.tf
│   ├── variables.tf
│   └── terraform.tfvars
└── production/
    ├── main.tf
    ├── variables.tf
    └── terraform.tfvars
```

**Pros:**
- Complete isolation
- Different backends
- Independent state files

**Cons:**
- Code duplication
- Harder to maintain consistency

### Strategy 2: Workspaces

```bash
# Create workspaces
terraform workspace new dev
terraform workspace new staging
terraform workspace new production

# Switch workspace
terraform workspace select production

# List workspaces
terraform workspace list

# Use in code
resource "aws_instance" "web" {
  instance_type = terraform.workspace == "production" ? "t3.large" : "t3.micro"
  
  tags = {
    Environment = terraform.workspace
  }
}
```

**Pros:**
- Single codebase
- Easy switching
- Less duplication

**Cons:**
- Shared state file location
- Less isolation

### Strategy 3: Terraform Cloud/Enterprise

```hcl
terraform {
  cloud {
    organization = "mycompany"
    
    workspaces {
      tags = ["app:myapp", "env:production"]
    }
  }
}
```

**Pros:**
- Centralized management
- Team collaboration
- Policy enforcement
- Cost estimation

### Best Approach: Hybrid

```
infrastructure/
├── modules/              # Reusable modules
│   ├── vpc/
│   ├── compute/
│   └── database/
├── environments/         # Environment-specific
│   ├── dev/
│   │   └── main.tf      # Use modules
│   ├── staging/
│   │   └── main.tf      # Use modules
│   └── production/
│       └── main.tf      # Use modules
└── global/              # Shared resources
    ├── iam/
    └── route53/
```

## State Management

### Remote State

**S3 Backend:**
```hcl
terraform {
  backend "s3" {
    bucket         = "myapp-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

**Create S3 backend:**
```bash
# Create S3 bucket
aws s3api create-bucket \
  --bucket myapp-terraform-state \
  --region us-east-1

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket myapp-terraform-state \
  --versioning-configuration Status=Enabled

# Enable encryption
aws s3api put-bucket-encryption \
  --bucket myapp-terraform-state \
  --server-side-encryption-configuration \
    '{"Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"}}]}'

# Create DynamoDB table for locking
aws dynamodb create-table \
  --table-name terraform-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5
```

### State Best Practices

**1. Never commit state files**
```bash
# .gitignore
*.tfstate
*.tfstate.*
*.tfstate.backup
.terraform/
.terraform.lock.hcl
```

**2. Use state locking**
```hcl
# Enabled automatically with DynamoDB
terraform {
  backend "s3" {
    dynamodb_table = "terraform-state-lock"
  }
}
```

**3. Separate state per environment**
```
s3://myapp-terraform-state/
├── dev/terraform.tfstate
├── staging/terraform.tfstate
└── production/terraform.tfstate
```

**4. Backup state regularly**
```bash
# Manual backup
terraform state pull > terraform.tfstate.backup

# S3 versioning handles this automatically
```

**5. State operations**
```bash
# List resources
terraform state list

# Show resource
terraform state show aws_instance.web

# Move resource
terraform state mv aws_instance.old aws_instance.new

# Remove resource from state
terraform state rm aws_instance.legacy

# Import existing resource
terraform import aws_instance.web i-1234567890abcdef0
```

## Secrets Management

### Don't Store Secrets in Code

**Bad:**
```hcl
resource "aws_db_instance" "db" {
  password = "MyPassword123!"  # Never do this!
}
```

**Good - Use variables:**
```hcl
variable "db_password" {
  type      = string
  sensitive = true
}

resource "aws_db_instance" "db" {
  password = var.db_password
}
```

### Secret Storage Options

**1. Environment Variables**
```bash
export TF_VAR_db_password="SecurePassword123!"
terraform apply
```

**2. Terraform Cloud Variables**
```
# Mark as sensitive in Terraform Cloud UI
# Never shown in logs or output
```

**3. AWS Secrets Manager**
```hcl
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "production/db/password"
}

resource "aws_db_instance" "db" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}
```

**4. HashiCorp Vault**
```hcl
provider "vault" {
  address = "https://vault.example.com"
}

data "vault_generic_secret" "db" {
  path = "secret/database/production"
}

resource "aws_db_instance" "db" {
  password = data.vault_generic_secret.db.data["password"]
}
```

**5. Ansible Vault**
```bash
# Create encrypted file
ansible-vault create secrets.yml

# Edit encrypted file
ansible-vault edit secrets.yml

# Run with vault
ansible-playbook playbook.yml --ask-vault-pass
```

### Sensitive Output

```hcl
output "db_endpoint" {
  value = aws_db_instance.db.endpoint
}

output "db_password" {
  value     = aws_db_instance.db.password
  sensitive = true  # Hidden from console output
}
```

## Testing

### Terraform Testing

**1. Validation**
```bash
# Syntax check
terraform validate

# Format check
terraform fmt -check

# Plan check
terraform plan
```

**2. Pre-commit Hooks**
```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.77.0
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_docs
      - id: terraform_tflint
      - id: terraform_tfsec
```

**3. Automated Testing - Terratest**
```go
// test/vpc_test.go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestVPCCreation(t *testing.T) {
    terraformOptions := &terraform.Options{
        TerraformDir: "../modules/vpc",
        Vars: map[string]interface{}{
            "cidr_block": "10.0.0.0/16",
        },
    }
    
    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)
    
    vpcId := terraform.Output(t, terraformOptions, "vpc_id")
    assert.NotEmpty(t, vpcId)
}
```

**4. Static Analysis**
```bash
# tflint
tflint --init
tflint

# tfsec
tfsec .

# checkov
checkov -d .
```

### Ansible Testing

**1. Syntax Check**
```bash
ansible-playbook playbook.yml --syntax-check
```

**2. Dry Run**
```bash
ansible-playbook playbook.yml --check
ansible-playbook playbook.yml --check --diff
```

**3. Molecule Testing**
```bash
# Initialize molecule
molecule init role myrole

# Test role
molecule test

# Directory structure
myrole/
├── molecule/
│   └── default/
│       ├── converge.yml
│       ├── molecule.yml
│       └── verify.yml
├── tasks/
└── handlers/
```

**4. Linting**
```bash
# ansible-lint
ansible-lint playbook.yml

# yamllint
yamllint .
```

## CI/CD Integration

### Terraform CI/CD

**GitHub Actions:**
```yaml
name: Terraform

on:
  push:
    branches: [ main ]
  pull_request:

jobs:
  terraform:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v2
      with:
        terraform_version: 1.5.0
    
    - name: Terraform Format
      run: terraform fmt -check
    
    - name: Terraform Init
      run: terraform init
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    
    - name: Terraform Validate
      run: terraform validate
    
    - name: Terraform Plan
      run: terraform plan -out=tfplan
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    
    - name: Terraform Apply
      if: github.ref == 'refs/heads/main' && github.event_name == 'push'
      run: terraform apply -auto-approve tfplan
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

**GitLab CI:**
```yaml
stages:
  - validate
  - plan
  - apply

variables:
  TF_VERSION: "1.5.0"

.terraform:
  image:
    name: hashicorp/terraform:$TF_VERSION
    entrypoint: [""]
  before_script:
    - terraform init

validate:
  extends: .terraform
  stage: validate
  script:
    - terraform fmt -check
    - terraform validate

plan:
  extends: .terraform
  stage: plan
  script:
    - terraform plan -out=tfplan
  artifacts:
    paths:
      - tfplan

apply:
  extends: .terraform
  stage: apply
  script:
    - terraform apply -auto-approve tfplan
  when: manual
  only:
    - main
```

### Ansible CI/CD

**GitHub Actions:**
```yaml
name: Ansible

on:
  push:
    branches: [ main ]

jobs:
  ansible:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.10'
    
    - name: Install Ansible
      run: pip install ansible ansible-lint
    
    - name: Lint playbooks
      run: ansible-lint playbooks/
    
    - name: Syntax check
      run: ansible-playbook playbooks/site.yml --syntax-check
    
    - name: Run playbook
      run: |
        ansible-playbook playbooks/site.yml \
          -i inventories/production/hosts \
          --vault-password-file <(echo "${{ secrets.VAULT_PASSWORD }}")
```

## Documentation

### Infrastructure Documentation

**README.md template:**
```markdown
# Project Infrastructure

## Overview
Description of infrastructure

## Architecture
[Architecture diagram]

## Prerequisites
- Terraform >= 1.5.0
- AWS CLI configured
- Access to S3 state bucket

## Usage

### Initialize
```bash
terraform init
```

### Plan
```bash
terraform plan -var-file=production.tfvars
```

### Apply
```bash
terraform apply -var-file=production.tfvars
```

## Variables

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|----------|
| instance_type | EC2 instance type | string | t2.micro | no |
| vpc_cidr | VPC CIDR block | string | - | yes |

## Outputs

| Name | Description |
|------|-------------|
| vpc_id | VPC identifier |
| instance_ip | Instance public IP |

## Resources Created

- VPC
- 2 Public Subnets
- 2 Private Subnets
- Internet Gateway
- NAT Gateway
- EC2 Instances

## Costs
Estimated monthly cost: $X

## Maintenance
- Review quarterly
- Update Terraform version monthly
- Review security groups monthly
```

### Terraform Docs

```bash
# Install terraform-docs
brew install terraform-docs

# Generate documentation
terraform-docs markdown table . > README.md

# Auto-generate with pre-commit
# .pre-commit-config.yaml
- id: terraform_docs
  args:
    - --hook-config=--path-to-file=README.md
```

## Monitoring & Observability

### Terraform Cloud

```hcl
terraform {
  cloud {
    organization = "mycompany"
    
    workspaces {
      name = "production"
    }
  }
}
```

**Features:**
- Run history
- Cost estimation
- Policy enforcement (Sentinel)
- Team collaboration
- VCS integration

### State Drift Detection

```bash
# Detect drift
terraform plan -detailed-exitcode

# Exit codes:
# 0 = No changes
# 1 = Error
# 2 = Changes detected

# Automated drift detection
#!/bin/bash
terraform plan -detailed-exitcode
if [ $? -eq 2 ]; then
    echo "Drift detected!"
    # Send alert
fi
```

### Resource Tagging

```hcl
locals {
  common_tags = {
    ManagedBy   = "Terraform"
    Environment = var.environment
    Project     = var.project_name
    CostCenter  = var.cost_center
    Backup      = var.backup_enabled
    Compliance  = var.compliance_level
  }
}

resource "aws_instance" "web" {
  # ...
  tags = merge(
    local.common_tags,
    {
      Name = "web-server-01"
      Role = "webserver"
    }
  )
}
```

## Security Best Practices

### 1. Least Privilege

```hcl
# IAM policy for Terraform
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:*",
        "s3:*",
        "rds:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "us-east-1"
        }
      }
    }
  ]
}
```

### 2. Encryption

```hcl
# Encrypt at rest
resource "aws_ebs_volume" "data" {
  encrypted = true
  kms_key_id = aws_kms_key.main.arn
}

# Encrypt in transit
resource "aws_lb_listener" "https" {
  protocol = "HTTPS"
  ssl_policy = "ELBSecurityPolicy-TLS-1-2-2017-01"
}
```

### 3. Network Security

```hcl
# Security groups with minimal access
resource "aws_security_group" "web" {
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### 4. Compliance Checks

```bash
# Run compliance checks
checkov -d . --framework terraform

# tfsec
tfsec . --format json

# Integrate with CI/CD
- name: Run tfsec
  run: |
    tfsec . --soft-fail
```

## Disaster Recovery

### Backup Strategy

```hcl
# Automated backups
resource "aws_db_instance" "db" {
  backup_retention_period = 7
  backup_window          = "03:00-04:00"
  
  final_snapshot_identifier = "db-final-snapshot-${timestamp()}"
  skip_final_snapshot      = false
}

# Snapshot schedule
resource "aws_backup_plan" "main" {
  name = "daily-backups"
  
  rule {
    rule_name         = "daily_backup"
    target_vault_name = aws_backup_vault.main.name
    schedule          = "cron(0 2 * * ? *)"
    
    lifecycle {
      delete_after = 30
    }
  }
}
```

### Recovery Procedures

```bash
# 1. Restore from state backup
aws s3 cp s3://state-bucket/backup/terraform.tfstate .
terraform init -backend-config="key=restored-state"

# 2. Import existing resources
terraform import aws_instance.web i-1234567890

# 3. Recreate from scratch
terraform destroy
terraform apply
```

## Cost Optimization

### 1. Resource Rightsizing

```hcl
# Use appropriate instance types
locals {
  instance_type = {
    dev        = "t3.micro"
    staging    = "t3.small"
    production = "t3.large"
  }
}

resource "aws_instance" "web" {
  instance_type = local.instance_type[var.environment]
}
```

### 2. Auto Scaling

```hcl
resource "aws_autoscaling_group" "web" {
  min_size         = var.environment == "production" ? 2 : 1
  max_size         = var.environment == "production" ? 10 : 2
  desired_capacity = var.environment == "production" ? 3 : 1
}
```

### 3. Cost Tagging

```hcl
tags = {
  CostCenter  = "engineering"
  Project     = "myapp"
  Environment = var.environment
}
```

### 4. Terraform Cost Estimation

```bash
# Infracost
infracost breakdown --path .
infracost diff --path .

# GitHub Action
- uses: infracost/actions/setup@v2
- run: infracost breakdown --path=.
```

## Common Patterns

### Multi-Region Deployment

```hcl
# modules/app/main.tf
resource "aws_instance" "web" {
  count = var.instance_count
  # ...
}

# environments/production/main.tf
module "app_us_east" {
  source = "../../modules/app"
  
  providers = {
    aws = aws.us_east_1
  }
  
  instance_count = 3
  region         = "us-east-1"
}

module "app_eu_west" {
  source = "../../modules/app"
  
  providers = {
    aws = aws.eu_west_1
  }
  
  instance_count = 2
  region         = "eu-west-1"
}
```

### Blue-Green Deployment

```hcl
resource "aws_lb_target_group" "blue" {
  name     = "myapp-blue"
  port     = 80
  protocol = "HTTP"
  vpc_id   = var.vpc_id
}

resource "aws_lb_target_group" "green" {
  name     = "myapp-green"
  port     = 80
  protocol = "HTTP"
  vpc_id   = var.vpc_id
}

resource "aws_lb_listener" "main" {
  load_balancer_arn = aws_lb.main.arn
  
  default_action {
    type             = "forward"
    target_group_arn = var.active_environment == "blue" ? 
                       aws_lb_target_group.blue.arn : 
                       aws_lb_target_group.green.arn
  }
}
```

Master IaC best practices for reliable infrastructure automation!
