# Terraform Modules

## Module là gì?

Module là container để group nhiều resources lại thành một reusable component. Module giúp organize code, reuse, và share infrastructure.

**Without modules:**
```hcl
# Duplicate code for each VPC
resource "aws_vpc" "dev_vpc" {
  cidr_block = "10.0.0.0/16"
}
resource "aws_subnet" "dev_subnet" {
  vpc_id     = aws_vpc.dev_vpc.id
  cidr_block = "10.0.1.0/24"
}

# Same code repeated for prod
resource "aws_vpc" "prod_vpc" {
  cidr_block = "10.1.0.0/16"
}
resource "aws_subnet" "prod_subnet" {
  vpc_id     = aws_vpc.prod_vpc.id
  cidr_block = "10.1.1.0/24"
}
```

**With modules:**
```hcl
module "dev_vpc" {
  source     = "./modules/vpc"
  cidr_block = "10.0.0.0/16"
  env        = "dev"
}

module "prod_vpc" {
  source     = "./modules/vpc"
  cidr_block = "10.1.0.0/16"
  env        = "prod"
}
```

## Benefits

- **Reusability**: Tái sử dụng code
- **Organization**: Cấu trúc code rõ ràng
- **Encapsulation**: Hide complexity
- **Consistency**: Đảm bảo standards
- **Testability**: Test từng module

## Module Structure

### Basic Structure

```
modules/
  └── vpc/
      ├── main.tf        # Resources
      ├── variables.tf   # Input variables
      └── outputs.tf     # Output values
```

### Advanced Structure

```
modules/
  └── vpc/
      ├── main.tf
      ├── variables.tf
      ├── outputs.tf
      ├── README.md      # Documentation
      ├── examples/      # Usage examples
      │   └── basic/
      │       └── main.tf
      ├── tests/         # Tests
      │   └── vpc_test.go
      └── versions.tf    # Provider versions
```

## Creating Modules

### Simple VPC Module

**modules/vpc/variables.tf:**
```hcl
variable "cidr_block" {
  type        = string
  description = "CIDR block for VPC"
}

variable "name" {
  type        = string
  description = "Name prefix for resources"
}

variable "enable_dns_hostnames" {
  type        = bool
  description = "Enable DNS hostnames"
  default     = true
}

variable "tags" {
  type        = map(string)
  description = "Tags for resources"
  default     = {}
}
```

**modules/vpc/main.tf:**
```hcl
resource "aws_vpc" "this" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = var.enable_dns_hostnames
  enable_dns_support   = true
  
  tags = merge(var.tags, {
    Name = "${var.name}-vpc"
  })
}

resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id
  
  tags = merge(var.tags, {
    Name = "${var.name}-igw"
  })
}

data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_subnet" "public" {
  count                   = 2
  vpc_id                  = aws_vpc.this.id
  cidr_block              = cidrsubnet(var.cidr_block, 8, count.index)
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  
  tags = merge(var.tags, {
    Name = "${var.name}-public-${count.index + 1}"
  })
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.this.id
  }
  
  tags = merge(var.tags, {
    Name = "${var.name}-public-rt"
  })
}

resource "aws_route_table_association" "public" {
  count          = 2
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}
```

**modules/vpc/outputs.tf:**
```hcl
output "vpc_id" {
  value       = aws_vpc.this.id
  description = "VPC ID"
}

output "vpc_cidr_block" {
  value       = aws_vpc.this.cidr_block
  description = "VPC CIDR block"
}

output "public_subnet_ids" {
  value       = aws_subnet.public[*].id
  description = "Public subnet IDs"
}

output "internet_gateway_id" {
  value       = aws_internet_gateway.this.id
  description = "Internet Gateway ID"
}
```

**modules/vpc/README.md:**
```markdown
# VPC Module

Creates a VPC with public subnets and Internet Gateway.

## Usage

```hcl
module "vpc" {
  source     = "./modules/vpc"
  cidr_block = "10.0.0.0/16"
  name       = "myapp"
  tags = {
    Environment = "dev"
  }
}
```

## Inputs

| Name | Type | Default | Description |
|------|------|---------|-------------|
| cidr_block | string | - | VPC CIDR block |
| name | string | - | Name prefix |
| enable_dns_hostnames | bool | true | Enable DNS hostnames |
| tags | map(string) | {} | Tags |

## Outputs

| Name | Description |
|------|-------------|
| vpc_id | VPC ID |
| public_subnet_ids | Public subnet IDs |
```

## Using Modules

### Local Module

```hcl
# main.tf
module "dev_vpc" {
  source     = "./modules/vpc"
  cidr_block = "10.0.0.0/16"
  name       = "dev"
  
  tags = {
    Environment = "development"
  }
}

# Access module outputs
output "dev_vpc_id" {
  value = module.dev_vpc.vpc_id
}

resource "aws_instance" "web" {
  subnet_id = module.dev_vpc.public_subnet_ids[0]
  # ...
}
```

### Module Sources

```hcl
# Local path
module "vpc" {
  source = "./modules/vpc"
}

# Git (HTTPS)
module "vpc" {
  source = "git::https://github.com/user/terraform-aws-vpc.git"
}

# Git (SSH)
module "vpc" {
  source = "git::ssh://git@github.com/user/terraform-aws-vpc.git"
}

# Git with branch
module "vpc" {
  source = "git::https://github.com/user/terraform-aws-vpc.git?ref=develop"
}

# Git with tag
module "vpc" {
  source = "git::https://github.com/user/terraform-aws-vpc.git?ref=v1.2.0"
}

# Git with commit
module "vpc" {
  source = "git::https://github.com/user/terraform-aws-vpc.git?ref=abc123"
}

# Terraform Registry
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
}

# HTTP URL
module "vpc" {
  source = "https://example.com/vpc.zip"
}

# S3 bucket
module "vpc" {
  source = "s3::https://s3-us-west-2.amazonaws.com/bucket/vpc.zip"
}
```

## Module Versioning

```hcl
# Exact version
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
}

# >= version
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = ">= 5.0.0"
}

# ~> pessimistic constraint (allows rightmost version to increment)
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"  # Allows >= 5.0 and < 6.0
}

# >= and < constraints
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = ">= 5.0.0, < 6.0.0"
}
```

### Git Tags for Versioning

```bash
# Tag a release
git tag -a v1.0.0 -m "Release v1.0.0"
git push origin v1.0.0

# Use tagged version
module "vpc" {
  source = "git::https://github.com/user/terraform-modules.git//vpc?ref=v1.0.0"
}
```

## Advanced Module Examples

### Compute Module

**modules/compute/variables.tf:**
```hcl
variable "vpc_id" {
  type = string
}

variable "subnet_ids" {
  type = list(string)
}

variable "instance_count" {
  type    = number
  default = 2
}

variable "instance_type" {
  type    = string
  default = "t2.micro"
}

variable "name" {
  type = string
}

variable "ssh_cidr" {
  type    = string
  default = "0.0.0.0/0"
}
```

**modules/compute/main.tf:**
```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
}

resource "aws_security_group" "instance" {
  name        = "${var.name}-instance-sg"
  description = "Security group for instances"
  vpc_id      = var.vpc_id
  
  dynamic "ingress" {
    for_each = [22, 80, 443]
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = [var.ssh_cidr]
    }
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "this" {
  count                  = var.instance_count
  ami                    = data.aws_ami.ubuntu.id
  instance_type          = var.instance_type
  subnet_id              = var.subnet_ids[count.index % length(var.subnet_ids)]
  vpc_security_group_ids = [aws_security_group.instance.id]
  
  user_data = <<-EOF
              #!/bin/bash
              apt-get update
              apt-get install -y nginx
              systemctl start nginx
              echo "<h1>Instance ${count.index + 1}</h1>" > /var/www/html/index.html
              EOF
  
  tags = {
    Name = "${var.name}-instance-${count.index + 1}"
  }
}
```

**modules/compute/outputs.tf:**
```hcl
output "instance_ids" {
  value = aws_instance.this[*].id
}

output "instance_public_ips" {
  value = aws_instance.this[*].public_ip
}

output "security_group_id" {
  value = aws_security_group.instance.id
}
```

### Database Module

**modules/rds/variables.tf:**
```hcl
variable "vpc_id" {
  type = string
}

variable "subnet_ids" {
  type = list(string)
}

variable "db_name" {
  type = string
}

variable "db_username" {
  type = string
}

variable "db_password" {
  type      = string
  sensitive = true
}

variable "instance_class" {
  type    = string
  default = "db.t3.micro"
}

variable "allocated_storage" {
  type    = number
  default = 20
}

variable "name_prefix" {
  type = string
}
```

**modules/rds/main.tf:**
```hcl
resource "aws_db_subnet_group" "this" {
  name       = "${var.name_prefix}-db-subnet-group"
  subnet_ids = var.subnet_ids
  
  tags = {
    Name = "${var.name_prefix}-db-subnet-group"
  }
}

resource "aws_security_group" "db" {
  name        = "${var.name_prefix}-db-sg"
  description = "Security group for RDS"
  vpc_id      = var.vpc_id
  
  ingress {
    from_port   = 5432
    to_port     = 5432
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_db_instance" "this" {
  identifier           = "${var.name_prefix}-db"
  engine               = "postgres"
  engine_version       = "14.7"
  instance_class       = var.instance_class
  allocated_storage    = var.allocated_storage
  storage_type         = "gp3"
  db_name              = var.db_name
  username             = var.db_username
  password             = var.db_password
  db_subnet_group_name = aws_db_subnet_group.this.name
  vpc_security_group_ids = [aws_security_group.db.id]
  skip_final_snapshot  = true
  
  backup_retention_period = 7
  backup_window           = "03:00-04:00"
  maintenance_window      = "mon:04:00-mon:05:00"
  
  multi_az = var.instance_class != "db.t3.micro"
  
  tags = {
    Name = "${var.name_prefix}-db"
  }
}
```

**modules/rds/outputs.tf:**
```hcl
output "db_endpoint" {
  value = aws_db_instance.this.endpoint
}

output "db_name" {
  value = aws_db_instance.this.db_name
}

output "db_address" {
  value = aws_db_instance.this.address
}
```

## Using Multiple Modules

**main.tf:**
```hcl
# VPC Module
module "vpc" {
  source     = "./modules/vpc"
  cidr_block = "10.0.0.0/16"
  name       = "myapp"
  tags = {
    Environment = "production"
  }
}

# Compute Module
module "compute" {
  source = "./modules/compute"
  
  vpc_id         = module.vpc.vpc_id
  subnet_ids     = module.vpc.public_subnet_ids
  instance_count = 2
  instance_type  = "t2.micro"
  name           = "myapp"
  ssh_cidr       = "1.2.3.4/32"
}

# Database Module
module "database" {
  source = "./modules/rds"
  
  vpc_id            = module.vpc.vpc_id
  subnet_ids        = module.vpc.public_subnet_ids
  db_name           = "myappdb"
  db_username       = "admin"
  db_password       = var.db_password
  instance_class    = "db.t3.small"
  allocated_storage = 50
  name_prefix       = "myapp"
}

# Outputs
output "instance_ips" {
  value = module.compute.instance_public_ips
}

output "database_endpoint" {
  value = module.database.db_endpoint
}
```

## Module Composition

Modules có thể sử dụng modules khác.

**Parent Module:**
```hcl
# modules/app-stack/main.tf
module "vpc" {
  source     = "../vpc"
  cidr_block = var.vpc_cidr
  name       = var.name
}

module "compute" {
  source         = "../compute"
  vpc_id         = module.vpc.vpc_id
  subnet_ids     = module.vpc.public_subnet_ids
  instance_count = var.instance_count
  name           = var.name
}

module "database" {
  source     = "../rds"
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.public_subnet_ids
  db_name    = var.db_name
  # ...
}
```

**Using Parent Module:**
```hcl
module "my_app" {
  source         = "./modules/app-stack"
  name           = "myapp"
  vpc_cidr       = "10.0.0.0/16"
  instance_count = 2
  db_name        = "mydb"
  # ...
}
```

## Best Practices

### 1. Input Validation

```hcl
variable "instance_type" {
  type = string
  
  validation {
    condition     = contains(["t2.micro", "t2.small", "t2.medium"], var.instance_type)
    error_message = "Instance type must be t2.micro, t2.small, or t2.medium."
  }
}

variable "cidr_block" {
  type = string
  
  validation {
    condition     = can(cidrhost(var.cidr_block, 0))
    error_message = "Must be a valid CIDR block."
  }
}
```

### 2. Sensible Defaults

```hcl
variable "enable_monitoring" {
  type        = bool
  description = "Enable detailed monitoring"
  default     = false
}

variable "instance_type" {
  type        = string
  description = "Instance type"
  default     = "t2.micro"
}
```

### 3. Comprehensive Outputs

```hcl
output "vpc_id" {
  value       = aws_vpc.this.id
  description = "VPC ID"
}

output "vpc_cidr" {
  value       = aws_vpc.this.cidr_block
  description = "VPC CIDR block"
}

output "all_subnet_ids" {
  value       = concat(aws_subnet.public[*].id, aws_subnet.private[*].id)
  description = "All subnet IDs"
}
```

### 4. Use Data Sources

```hcl
# Get latest AMI dynamically
data "aws_ami" "latest" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}
```

### 5. Use Locals for Computed Values

```hcl
locals {
  common_tags = merge(var.tags, {
    ManagedBy = "Terraform"
    Module    = "vpc"
  })
  
  az_count = length(data.aws_availability_zones.available.names)
  
  public_subnet_cidrs = [
    for i in range(local.az_count) :
    cidrsubnet(var.cidr_block, 8, i)
  ]
}
```

### 6. Document Your Modules

```markdown
# VPC Module

## Description
Creates VPC with public/private subnets, IGW, NAT Gateway.

## Usage
```hcl
module "vpc" {
  source = "./modules/vpc"
  # ...
}
```

## Requirements
- Terraform >= 1.0
- AWS Provider >= 5.0

## Inputs
| Name | Type | Default | Required | Description |
|------|------|---------|----------|-------------|
| cidr_block | string | - | yes | VPC CIDR |

## Outputs
| Name | Description |
|------|-------------|
| vpc_id | VPC ID |
```

## Testing Modules

### Manual Testing

```
tests/
  └── vpc/
      ├── main.tf
      ├── variables.tf
      └── terraform.tfvars
```

**tests/vpc/main.tf:**
```hcl
module "test_vpc" {
  source = "../../modules/vpc"
  
  cidr_block = var.cidr_block
  name       = "test"
}

output "test_vpc_id" {
  value = module.test_vpc.vpc_id
}
```

### Automated Testing (Terratest)

```go
// test/vpc_test.go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestVPCModule(t *testing.T) {
    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../examples/basic",
    })

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    vpcID := terraform.Output(t, terraformOptions, "vpc_id")
    assert.NotEmpty(t, vpcID)
}
```

## Publishing Modules

### Terraform Registry

Requirements:
- GitHub repository
- Named `terraform-<PROVIDER>-<NAME>`
- Semantic versioning (Git tags: v1.0.0)
- Standard module structure

**Example:**
```
terraform-aws-vpc/
├── main.tf
├── variables.tf
├── outputs.tf
├── README.md
├── LICENSE
└── examples/
    └── basic/
```

```bash
# Tag and publish
git tag v1.0.0
git push origin v1.0.0
```

### Private Registry

Use Terraform Cloud:

```hcl
module "vpc" {
  source  = "app.terraform.io/myorg/vpc/aws"
  version = "1.0.0"
}
```

## Module Checklist

- [ ] Clear variable names and descriptions
- [ ] Sensible defaults
- [ ] Input validation
- [ ] Comprehensive outputs
- [ ] README with usage examples
- [ ] Examples directory
- [ ] Tests
- [ ] Version constraints (versions.tf)
- [ ] License file
- [ ] Changelog

Master Terraform modules for scalable infrastructure!
