# Terraform Basics

## Terraform là gì?

Terraform là Infrastructure as Code (IaC) tool để provision và manage cloud infrastructure. Thay vì click trên UI, bạn viết code để define infrastructure.

**Without Terraform:**
- Login vào AWS Console
- Click Create VPC
- Click Create Subnet
- Click Create EC2...
- Không reproduce được
- Khó track changes

**With Terraform:**
```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}
```

- Code có thể version control
- Reproduce infrastructure dễ dàng
- Automate deployment
- Multi-cloud support

## Installation

### Ubuntu/Debian

```bash
# Add HashiCorp GPG key
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

# Add repository
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

# Install
sudo apt update
sudo apt install terraform

# Verify
terraform version
```

### CentOS/RHEL

```bash
# Add repository
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo

# Install
sudo yum install terraform

# Verify
terraform version
```

### Using tfenv (Version Manager)

```bash
# Install tfenv
git clone https://github.com/tfutils/tfenv.git ~/.tfenv
echo 'export PATH="$HOME/.tfenv/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# List available versions
tfenv list-remote

# Install specific version
tfenv install 1.6.0
tfenv use 1.6.0

# Install latest
tfenv install latest
tfenv use latest
```

## Terraform Workflow

```
1. Write     → 2. Init     → 3. Plan      → 4. Apply     → 5. Destroy
   (.tf)        (download)    (preview)      (execute)      (cleanup)
```

1. **Write**: Viết Terraform configuration (`.tf` files)
2. **Init**: `terraform init` - Download providers
3. **Plan**: `terraform plan` - Preview changes
4. **Apply**: `terraform apply` - Execute changes
5. **Destroy**: `terraform destroy` - Delete infrastructure

## HCL Syntax (HashiCorp Configuration Language)

### Basic Blocks

```hcl
# Terraform block - configure Terraform itself
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Provider block - configure cloud provider
provider "aws" {
  region = "us-east-1"
}

# Resource block - define infrastructure
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

# Data block - query existing resources
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
}

# Variable block - input variables
variable "instance_type" {
  type    = string
  default = "t2.micro"
}

# Output block - output values
output "instance_ip" {
  value = aws_instance.web.public_ip
}

# Local block - local variables
locals {
  common_tags = {
    Environment = "dev"
    Project     = "myapp"
  }
}

# Module block - reusable components
module "vpc" {
  source = "./modules/vpc"
  cidr   = "10.0.0.0/16"
}
```

## Variables

### Defining Variables

```hcl
# variables.tf

# String variable
variable "aws_region" {
  type        = string
  description = "AWS region"
  default     = "us-east-1"
}

# Number variable
variable "instance_count" {
  type    = number
  default = 2
}

# Boolean variable
variable "enable_monitoring" {
  type    = bool
  default = true
}

# List variable
variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b"]
}

# Map variable
variable "instance_types" {
  type = map(string)
  default = {
    dev  = "t2.micro"
    prod = "t2.medium"
  }
}

# Object variable
variable "server_config" {
  type = object({
    name          = string
    instance_type = string
    disk_size     = number
  })
  default = {
    name          = "web-server"
    instance_type = "t2.micro"
    disk_size     = 20
  }
}
```

### Providing Variable Values

```hcl
# Method 1: terraform.tfvars
aws_region     = "us-west-2"
instance_count = 3

# Method 2: CLI
terraform apply -var="aws_region=us-west-2" -var="instance_count=3"

# Method 3: Environment variables
export TF_VAR_aws_region="us-west-2"
export TF_VAR_instance_count=3

# Method 4: .auto.tfvars (automatically loaded)
# prod.auto.tfvars
aws_region = "us-west-2"
```

## Outputs

```hcl
# outputs.tf

output "instance_id" {
  value       = aws_instance.web.id
  description = "ID of EC2 instance"
}

output "instance_public_ip" {
  value     = aws_instance.web.public_ip
  sensitive = true  # Hide in CLI output
}

output "instance_private_ips" {
  value = aws_instance.web[*].private_ip  # Splat expression
}

# View outputs
# terraform output
# terraform output instance_id
# terraform output -json
```

## Locals

```hcl
locals {
  # Common tags for all resources
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "Terraform"
  }
  
  # Computed values
  instance_name = "${var.project_name}-${var.environment}-instance"
  
  # Conditionals
  instance_type = var.environment == "prod" ? "t2.medium" : "t2.micro"
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = local.instance_type
  tags          = merge(local.common_tags, { Name = local.instance_name })
}
```

## Providers

### AWS Provider

```hcl
provider "aws" {
  region  = var.aws_region
  profile = "default"  # AWS CLI profile
  
  # Default tags for all resources
  default_tags {
    tags = {
      Environment = "dev"
      ManagedBy   = "Terraform"
    }
  }
}

# Multiple AWS accounts/regions
provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

provider "aws" {
  alias   = "production"
  region  = "us-east-1"
  profile = "prod-account"
}

resource "aws_instance" "west_instance" {
  provider = aws.west
  ami      = "ami-xxx"
}
```

### Azure Provider

```hcl
provider "azurerm" {
  features {}
  subscription_id = var.subscription_id
  tenant_id       = var.tenant_id
}
```

### GCP Provider

```hcl
provider "google" {
  project = var.project_id
  region  = "us-central1"
}
```

## Resources

### Basic Resource

```hcl
resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  tags = {
    Name = "example-instance"
  }
}
```

### Resource Dependencies

```hcl
# Implicit dependency (via reference)
resource "aws_eip" "ip" {
  instance = aws_instance.web.id  # Depends on aws_instance.web
}

# Explicit dependency
resource "aws_instance" "web" {
  ami           = "ami-xxx"
  instance_type = "t2.micro"
  
  depends_on = [aws_security_group.sg]
}
```

### Resource Meta-Arguments

#### count

```hcl
resource "aws_instance" "server" {
  count         = 3
  ami           = "ami-xxx"
  instance_type = "t2.micro"
  
  tags = {
    Name = "server-${count.index}"  # server-0, server-1, server-2
  }
}

# Access instances
output "instance_ids" {
  value = aws_instance.server[*].id  # All IDs
}
```

#### for_each

```hcl
variable "instances" {
  type = map(object({
    instance_type = string
    ami           = string
  }))
  default = {
    web = {
      instance_type = "t2.micro"
      ami           = "ami-web"
    }
    db = {
      instance_type = "t2.small"
      ami           = "ami-db"
    }
  }
}

resource "aws_instance" "server" {
  for_each      = var.instances
  ami           = each.value.ami
  instance_type = each.value.instance_type
  
  tags = {
    Name = each.key  # "web" or "db"
  }
}

# Access specific instance
output "web_ip" {
  value = aws_instance.server["web"].public_ip
}
```

#### lifecycle

```hcl
resource "aws_instance" "web" {
  ami           = "ami-xxx"
  instance_type = "t2.micro"
  
  lifecycle {
    create_before_destroy = true   # Create new before destroying old
    prevent_destroy       = true   # Prevent terraform destroy
    ignore_changes        = [tags] # Ignore changes to tags
  }
}
```

#### provisioner

```hcl
resource "aws_instance" "web" {
  ami           = "ami-xxx"
  instance_type = "t2.micro"
  key_name      = "my-key"
  
  # Remote exec provisioner
  provisioner "remote-exec" {
    inline = [
      "sudo apt update",
      "sudo apt install -y nginx"
    ]
    
    connection {
      type        = "ssh"
      user        = "ubuntu"
      private_key = file("~/.ssh/id_rsa")
      host        = self.public_ip
    }
  }
  
  # Local exec provisioner
  provisioner "local-exec" {
    command = "echo ${self.public_ip} >> ips.txt"
  }
}
```

## Data Sources

Query existing resources.

```hcl
# Get Ubuntu AMI
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
}

# Get availability zones
data "aws_availability_zones" "available" {
  state = "available"
}

# Get existing VPC
data "aws_vpc" "default" {
  default = true
}

# Get current region
data "aws_region" "current" {}

# Get AWS account ID
data "aws_caller_identity" "current" {}

# Use data sources
resource "aws_instance" "web" {
  ami               = data.aws_ami.ubuntu.id
  availability_zone = data.aws_availability_zones.available.names[0]
  subnet_id         = data.aws_vpc.default.id
}

output "account_id" {
  value = data.aws_caller_identity.current.account_id
}
```

## State Management

Terraform state (`terraform.tfstate`) tracks infrastructure.

### Local State

```hcl
# Default: terraform.tfstate in current directory
# Contains all resource information
```

### Remote State (S3 Backend)

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"  # State locking
  }
}

# Create S3 bucket first
resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-terraform-state"
  
  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

# DynamoDB for locking
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

### State Commands

```bash
# List resources in state
terraform state list

# Show resource details
terraform state show aws_instance.web

# Move resource in state
terraform state mv aws_instance.old aws_instance.new

# Remove resource from state (doesn't destroy)
terraform state rm aws_instance.web

# Import existing resource
terraform import aws_instance.web i-1234567890abcdef0

# Refresh state
terraform refresh
```

## Terraform Commands

### Basic Commands

```bash
# Initialize (download providers)
terraform init

# Validate syntax
terraform validate

# Format code
terraform fmt
terraform fmt -recursive

# Plan changes
terraform plan
terraform plan -out=tfplan

# Apply changes
terraform apply
terraform apply tfplan
terraform apply -auto-approve

# Destroy infrastructure
terraform destroy
terraform destroy -target=aws_instance.web

# Show current state
terraform show

# Output values
terraform output
terraform output instance_ip
```

### Advanced Commands

```bash
# Visualize dependency graph
terraform graph | dot -Tpng > graph.png

# Interactive console
terraform console

# Workspace management
terraform workspace list
terraform workspace new dev
terraform workspace select prod

# Target specific resource
terraform apply -target=aws_instance.web

# Parallelism
terraform apply -parallelism=10

# Debug
TF_LOG=DEBUG terraform apply
TF_LOG=TRACE terraform apply
```

## Complete Example

**Directory structure:**
```
project/
├── main.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars
└── .gitignore
```

**main.tf:**
```hcl
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

# Data sources
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
}

data "aws_availability_zones" "available" {
  state = "available"
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = merge(var.common_tags, {
    Name = "${var.project_name}-vpc"
  })
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = merge(var.common_tags, {
    Name = "${var.project_name}-igw"
  })
}

# Public Subnets
resource "aws_subnet" "public" {
  count                   = var.public_subnet_count
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  
  tags = merge(var.common_tags, {
    Name = "${var.project_name}-public-${count.index + 1}"
  })
}

# Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  tags = merge(var.common_tags, {
    Name = "${var.project_name}-public-rt"
  })
}

# Route Table Association
resource "aws_route_table_association" "public" {
  count          = var.public_subnet_count
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Security Group
resource "aws_security_group" "web" {
  name        = "${var.project_name}-web-sg"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    description = "HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.ssh_cidr]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = merge(var.common_tags, {
    Name = "${var.project_name}-web-sg"
  })
}

# EC2 Instances
resource "aws_instance" "web" {
  count                  = var.instance_count
  ami                    = data.aws_ami.ubuntu.id
  instance_type          = var.instance_type
  subnet_id              = aws_subnet.public[count.index % var.public_subnet_count].id
  vpc_security_group_ids = [aws_security_group.web.id]
  user_data              = file("${path.module}/user_data.sh")
  
  tags = merge(var.common_tags, {
    Name = "${var.project_name}-web-${count.index + 1}"
  })
}
```

**variables.tf:**
```hcl
variable "aws_region" {
  type        = string
  description = "AWS region"
  default     = "us-east-1"
}

variable "project_name" {
  type        = string
  description = "Project name"
  default     = "myproject"
}

variable "vpc_cidr" {
  type        = string
  description = "VPC CIDR block"
  default     = "10.0.0.0/16"
}

variable "public_subnet_count" {
  type        = number
  description = "Number of public subnets"
  default     = 2
}

variable "instance_count" {
  type        = number
  description = "Number of EC2 instances"
  default     = 2
}

variable "instance_type" {
  type        = string
  description = "EC2 instance type"
  default     = "t2.micro"
}

variable "ssh_cidr" {
  type        = string
  description = "CIDR block for SSH access"
  default     = "0.0.0.0/0"
}

variable "common_tags" {
  type        = map(string)
  description = "Common tags for all resources"
  default = {
    Environment = "dev"
    ManagedBy   = "Terraform"
  }
}
```

**outputs.tf:**
```hcl
output "vpc_id" {
  value       = aws_vpc.main.id
  description = "VPC ID"
}

output "public_subnet_ids" {
  value       = aws_subnet.public[*].id
  description = "Public subnet IDs"
}

output "instance_ids" {
  value       = aws_instance.web[*].id
  description = "EC2 instance IDs"
}

output "instance_public_ips" {
  value       = aws_instance.web[*].public_ip
  description = "EC2 instance public IPs"
}

output "security_group_id" {
  value       = aws_security_group.web.id
  description = "Security group ID"
}
```

**terraform.tfvars:**
```hcl
aws_region          = "us-west-2"
project_name        = "webapp"
vpc_cidr            = "10.0.0.0/16"
public_subnet_count = 2
instance_count      = 2
instance_type       = "t2.micro"
ssh_cidr            = "1.2.3.4/32"

common_tags = {
  Environment = "production"
  Project     = "webapp"
  ManagedBy   = "Terraform"
}
```

**user_data.sh:**
```bash
#!/bin/bash
apt-get update
apt-get install -y nginx
systemctl start nginx
systemctl enable nginx
echo "<h1>Hello from $(hostname)</h1>" > /var/www/html/index.html
```

**Deploy:**
```bash
# Initialize
terraform init

# Validate
terraform validate
terraform fmt

# Plan
terraform plan

# Apply
terraform apply

# Outputs
terraform output
terraform output instance_public_ips

# Destroy
terraform destroy
```

## Best Practices

1. **State Management:**
   - Use remote state (S3) with versioning
   - Enable state locking (DynamoDB)
   - Never commit state files to Git
   - Separate state per environment

2. **Code Organization:**
   - Separate variables, outputs, resources
   - Use modules for reusable code
   - One environment per directory
   - Use terraform workspaces

3. **Security:**
   - Never commit credentials
   - Use IAM roles when possible
   - Encrypt state file
   - Use Terraform Cloud for secrets

4. **Version Control:**
   - Commit `.tf` files
   - Commit `.tfvars.example`
   - Ignore `.terraform/`, `*.tfstate`, `*.tfvars`
   - Use semantic versioning for modules

5. **Planning:**
   - Always run `terraform plan`
   - Use `-out` flag to save plan
   - Review plan before apply
   - Test in dev before prod

**.gitignore:**
```
# Terraform
.terraform/
*.tfstate
*.tfstate.backup
*.tfvars
!example.tfvars
!terraform.tfvars.example
*.tfplan
.terraform.lock.hcl
crash.log
override.tf
override.tf.json
```

Master Terraform for infrastructure automation!
