# AWS Infrastructure Migration with Terraform - Complete Guide

This repository showcases a comprehensive, incremental AWS infrastructure migration project using Terraform. The Nautilus DevOps team approached their cloud migration strategically by breaking down large tasks into smaller, manageable units, ensuring smooth implementation and minimizing disruption.

## Table of Contents
- [Project Overview](#project-overview)
- [Prerequisites](#prerequisites)
- [Tasks Completed](#tasks-completed)
  - [Task 1: VPC Creation](#task-1-vpc-creation)
  - [Task 2: Security Group Setup](#task-2-security-group-setup)
  - [Task 3: EC2 Instance Deployment](#task-3-ec2-instance-deployment)
  - [Task 4: IAM Policy Configuration](#task-4-iam-policy-configuration)
  - [Task 5: Private VPC Infrastructure](#task-5-private-vpc-infrastructure)
  - [Task 6: DynamoDB with IAM Controls](#task-6-dynamodb-with-iam-controls)
  - [Task 7: EC2 Monitoring with CloudWatch](#task-7-ec2-monitoring-with-cloudwatch)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)

---

## Project Overview

The Nautilus DevOps team migrated their infrastructure to AWS cloud using an incremental approach. This strategy allowed for:
- Better control over the migration process
- Reduced risk through gradual implementation
- Optimization of resources at each stage
- Minimal disruption to ongoing operations

**Working Directory:** `/home/bob/terraform`

**Region:** `us-east-1`

---

## Prerequisites

- Terraform installed (v1.0+)
- AWS CLI configured
- VS Code (optional, for integrated terminal)
- Basic understanding of AWS services

---

## Tasks Completed

### Task 1: VPC Creation

**Objective:** Create a VPC named `devops-vpc` with IPv4 CIDR block.

#### Files Created:
- `main.tf`

#### Code:

**main.tf:**
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_vpc" "devops_vpc" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "devops-vpc"
  }
}
```

#### Deployment:
```bash
cd /home/bob/terraform
terraform init
terraform plan
terraform apply
```

#### Results:
- ✅ VPC Created: `vpc-a473d39b727d680ac`
- ✅ CIDR Block: `10.0.0.0/16`
- ✅ DNS support enabled by default

---

### Task 2: Security Group Setup

**Objective:** Create a security group under the default VPC for Nautilus App Servers.

#### Requirements:
- Name: `nautilus-sg`
- Description: `Security group for Nautilus App Servers`
- Inbound Rules:
  - HTTP (port 80) from `0.0.0.0/0`
  - SSH (port 22) from `0.0.0.0/0`

#### Code:

**main.tf:**
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

data "aws_vpc" "default" {
  default = true
}

resource "aws_security_group" "nautilus_sg" {
  name        = "nautilus-sg"
  description = "Security group for Nautilus App Servers"
  vpc_id      = data.aws_vpc.default.id

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "nautilus-sg"
  }
}
```

#### Results:
- ✅ Security Group ID: `sg-9f6b3c251906e5cb0`
- ✅ HTTP and SSH access configured
- ✅ Attached to default VPC

---

### Task 3: EC2 Instance Deployment

**Objective:** Launch an EC2 instance with RSA key pair.

#### Requirements:
- Instance Name: `devops-ec2`
- AMI: `ami-0c101f26f147fa7fd` (Amazon Linux)
- Instance Type: `t2.micro`
- Key Pair: `devops-kp` (RSA)
- Security Group: Default

#### Code:

**main.tf:**
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "tls_private_key" "devops_key" {
  algorithm = "RSA"
  rsa_bits  = 2048
}

resource "aws_key_pair" "devops_kp" {
  key_name   = "devops-kp"
  public_key = tls_private_key.devops_key.public_key_openssh
}

resource "local_file" "private_key" {
  content         = tls_private_key.devops_key.private_key_pem
  filename        = "${path.module}/devops-kp.pem"
  file_permission = "0400"
}

data "aws_security_group" "default" {
  name   = "default"
  vpc_id = data.aws_vpc.default.id
}

data "aws_vpc" "default" {
  default = true
}

resource "aws_instance" "devops_ec2" {
  ami                    = "ami-0c101f26f147fa7fd"
  instance_type          = "t2.micro"
  key_name               = aws_key_pair.devops_kp.key_name
  vpc_security_group_ids = [data.aws_security_group.default.id]

  tags = {
    Name = "devops-ec2"
  }
}
```

#### Results:
- ✅ Instance ID: `i-37dad303fc32e55c0`
- ✅ Key pair created and private key saved to `devops-kp.pem`
- ✅ SSH access: `ssh -i devops-kp.pem ec2-user@<public-ip>`

---

### Task 4: IAM Policy Configuration

**Objective:** Create an IAM policy for read-only EC2 console access.

#### Requirements:
- Policy Name: `iampolicy_mark`
- Access: Read-only to EC2 (instances, AMIs, snapshots)

#### Code:

**main.tf:**
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_iam_policy" "iampolicy_mark" {
  name        = "iampolicy_mark"
  description = "Read-only access to EC2 console"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ec2:Describe*",
          "ec2:Get*",
          "ec2:List*"
        ]
        Resource = "*"
      }
    ]
  })
}
```

#### Results:
- ✅ Policy created with read-only EC2 permissions
- ✅ Can be attached to users, groups, or roles

---

### Task 5: Private VPC Infrastructure

**Objective:** Create a private VPC with subnet and EC2 instance accessible only from within VPC.

#### Requirements:
- VPC: `devops-priv-vpc` (CIDR: `10.0.0.0/16`)
- Subnet: `devops-priv-subnet` (CIDR: `10.0.1.0/24`, no auto-assign IP)
- EC2: `devops-priv-ec2` (t2.micro)
- Security: Access only from VPC CIDR

#### Files Created:
- `main.tf`
- `variables.tf`
- `outputs.tf`

#### Code:

**main.tf:**
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

resource "aws_vpc" "devops_priv_vpc" {
  cidr_block = var.KKE_VPC_CIDR

  tags = {
    Name = "devops-priv-vpc"
  }
}

resource "aws_subnet" "devops_priv_subnet" {
  vpc_id                  = aws_vpc.devops_priv_vpc.id
  cidr_block              = var.KKE_SUBNET_CIDR
  map_public_ip_on_launch = false

  tags = {
    Name = "devops-priv-subnet"
  }
}

resource "aws_security_group" "devops_priv_sg" {
  name        = "devops-priv-sg"
  description = "Security group allowing access only from within VPC"
  vpc_id      = aws_vpc.devops_priv_vpc.id

  ingress {
    description = "Allow all traffic from within VPC"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = [var.KKE_VPC_CIDR]
  }

  egress {
    description = "Allow all outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "devops-priv-sg"
  }
}

resource "aws_instance" "devops_priv_ec2" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = "t2.micro"
  subnet_id              = aws_subnet.devops_priv_subnet.id
  vpc_security_group_ids = [aws_security_group.devops_priv_sg.id]

  tags = {
    Name = "devops-priv-ec2"
  }
}
```

**variables.tf:**
```hcl
variable "KKE_VPC_CIDR" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "KKE_SUBNET_CIDR" {
  description = "CIDR block for subnet"
  type        = string
  default     = "10.0.1.0/24"
}
```

**outputs.tf:**
```hcl
output "KKE_vpc_name" {
  description = "Name of the VPC"
  value       = aws_vpc.devops_priv_vpc.tags["Name"]
}

output "KKE_subnet_name" {
  description = "Name of the subnet"
  value       = aws_subnet.devops_priv_subnet.tags["Name"]
}

output "KKE_ec2_private" {
  description = "Name of the EC2 instance"
  value       = aws_instance.devops_priv_ec2.tags["Name"]
}
```

#### Results:
- ✅ Private VPC with isolated network
- ✅ No public IP assignment
- ✅ Security restricted to VPC CIDR only

---

### Task 6: DynamoDB with IAM Controls

**Objective:** Create a secure DynamoDB table with fine-grained IAM access control.

#### Requirements:
- Table: `datacenter-table`
- Role: `datacenter-role`
- Policy: `datacenter-readonly-policy` (GetItem, Scan, Query only)

#### Files Created:
- `main.tf`
- `variables.tf`
- `outputs.tf`
- `terraform.tfvars`

#### Code:

**main.tf:**
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_dynamodb_table" "datacenter_table" {
  name         = var.KKE_TABLE_NAME
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "id"

  attribute {
    name = "id"
    type = "S"
  }

  tags = {
    Name = var.KKE_TABLE_NAME
  }
}

resource "aws_iam_role" "datacenter_role" {
  name = var.KKE_ROLE_NAME

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
        Action = "sts:AssumeRole"
      }
    ]
  })

  tags = {
    Name = var.KKE_ROLE_NAME
  }
}

resource "aws_iam_policy" "datacenter_readonly_policy" {
  name        = var.KKE_POLICY_NAME
  description = "Read-only access to specific DynamoDB table"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "dynamodb:GetItem",
          "dynamodb:Scan",
          "dynamodb:Query"
        ]
        Resource = aws_dynamodb_table.datacenter_table.arn
      }
    ]
  })

  tags = {
    Name = var.KKE_POLICY_NAME
  }
}

resource "aws_iam_role_policy_attachment" "datacenter_policy_attachment" {
  role       = aws_iam_role.datacenter_role.name
  policy_arn = aws_iam_policy.datacenter_readonly_policy.arn
}
```

**variables.tf:**
```hcl
variable "KKE_TABLE_NAME" {
  description = "Name of the DynamoDB table"
  type        = string
}

variable "KKE_ROLE_NAME" {
  description = "Name of the IAM role"
  type        = string
}

variable "KKE_POLICY_NAME" {
  description = "Name of the IAM policy"
  type        = string
}
```

**outputs.tf:**
```hcl
output "kke_dynamodb_table" {
  description = "Name of the DynamoDB table"
  value       = aws_dynamodb_table.datacenter_table.name
}

output "kke_iam_role_name" {
  description = "Name of the IAM role"
  value       = aws_iam_role.datacenter_role.name
}

output "kke_iam_policy_name" {
  description = "Name of the IAM policy"
  value       = aws_iam_policy.datacenter_readonly_policy.name
}
```

**terraform.tfvars:**
```hcl
KKE_TABLE_NAME  = "datacenter-table"
KKE_ROLE_NAME   = "datacenter-role"
KKE_POLICY_NAME = "datacenter-readonly-policy"
```

#### Results:
- ✅ DynamoDB table with minimal configuration
- ✅ IAM role for trusted service access
- ✅ Read-only policy (no write permissions)
- ✅ Policy attached to role

---

### Task 7: EC2 Monitoring with CloudWatch

**Objective:** Set up EC2 instance with CloudWatch CPU monitoring alarm.

#### Requirements:
- Instance: `devops-ec2` (Ubuntu, t2.micro)
- Alarm: `devops-alarm`
- Threshold: CPU >= 90% for 1 consecutive 5-minute period
- Notification: SNS topic `devops-sns-topic`

#### Files Created:
- `main.tf`
- `outputs.tf`

#### Code:

**main.tf:**
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_sns_topic" "devops_sns_topic" {
  name = "devops-sns-topic"
}

resource "aws_instance" "devops_ec2" {
  ami           = "ami-0c02fb55956c7d316"
  instance_type = "t2.micro"

  tags = {
    Name = "devops-ec2"
  }
}

resource "aws_cloudwatch_metric_alarm" "devops_alarm" {
  alarm_name          = "devops-alarm"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 1
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 300
  statistic           = "Average"
  threshold           = 90
  alarm_description   = "This metric monitors EC2 CPU utilization"
  alarm_actions       = [aws_sns_topic.devops_sns_topic.arn]

  dimensions = {
    InstanceId = aws_instance.devops_ec2.id
  }

  tags = {
    Name = "devops-alarm"
  }
}
```

**outputs.tf:**
```hcl
output "KKE_instance_name" {
  description = "Name of the EC2 instance"
  value       = aws_instance.devops_ec2.tags["Name"]
}

output "KKE_alarm_name" {
  description = "Name of the CloudWatch alarm"
  value       = aws_cloudwatch_metric_alarm.devops_alarm.alarm_name
}
```

#### Deployment Output:
```
Apply complete! Resources: 2 added, 1 changed, 0 destroyed.

Outputs:

KKE_alarm_name = "devops-alarm"
KKE_instance_name = "devops-ec2"
```

#### Verification:
```bash
terraform plan
# Output: No changes. Your infrastructure matches the configuration.
```

#### Results:
- ✅ EC2 instance: `i-29a7f158a32eebce5`
- ✅ SNS topic created for notifications
- ✅ CloudWatch alarm monitoring CPU utilization
- ✅ Automatic alerts when CPU >= 90%

---

## Best Practices

### 1. **Incremental Migration**
- Break large tasks into smaller, manageable units
- Test each component before moving to the next
- Minimize disruption to ongoing operations

### 2. **Security First**
- Use security groups to restrict access
- Implement least privilege IAM policies
- Create private subnets for sensitive workloads
- Store private keys securely with proper permissions (0400)

### 3. **Infrastructure as Code**
- Use variables for reusability
- Implement outputs for important values
- Version control all Terraform files
- Use consistent naming conventions

### 4. **Monitoring & Alerts**
- Set up CloudWatch alarms for critical metrics
- Configure SNS topics for notifications
- Monitor CPU, memory, and disk utilization
- Establish baseline thresholds

### 5. **Resource Organization**
- Use meaningful tags for all resources
- Group related resources logically
- Document CIDR ranges and IP allocations
- Maintain clear outputs for reference

---

## Lessons Learned

### 1. **Data Sources vs Resources**
- Use `data` sources for existing resources
- Use `resource` blocks for resources managed by Terraform
- Ensure consistency between state and configuration

### 2. **Dependencies**
- Terraform automatically handles most dependencies
- Use `depends_on` when explicit dependency is needed
- Reference resources using interpolation for implicit dependencies

### 3. **State Management**
- Always verify with `terraform plan` before applying
- Check that "No changes" appears after successful deployment
- Use remote state for team collaboration

### 4. **Security Group Rules**
- Opening ports to `0.0.0.0/0` is convenient but not secure for production
- Consider restricting SSH access to specific IP ranges
- Use separate security groups for different tiers (web, app, database)

### 5. **Key Pair Management**
- Store private keys securely
- Use proper file permissions (0400)
- Consider using AWS Systems Manager Session Manager instead of SSH

---

## Common Commands

```bash
# Initialize Terraform
terraform init

# Validate configuration
terraform validate

# Plan changes
terraform plan

# Apply changes
terraform apply

# Verify state
terraform show

# List resources
terraform state list

# Destroy infrastructure
terraform destroy
```

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                         AWS Cloud                            │
│  ┌───────────────────────────────────────────────────────┐  │
│  │               VPC (devops-vpc)                        │  │
│  │           CIDR: 10.0.0.0/16                           │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │  Public Subnet                                  │  │  │
│  │  │  ┌─────────────────┐  ┌──────────────────────┐ │  │  │
│  │  │  │   EC2 Instance  │  │  Security Group      │ │  │  │
│  │  │  │   devops-ec2    │──│  nautilus-sg         │ │  │  │
│  │  │  │   t2.micro      │  │  - HTTP (80)         │ │  │  │
│  │  │  └────────┬────────┘  │  - SSH (22)          │ │  │  │
│  │  │           │            └──────────────────────┘ │  │  │
│  │  │           │                                      │  │  │
│  │  │  ┌────────▼────────┐                            │  │  │
│  │  │  │  CloudWatch     │                            │  │  │
│  │  │  │  Alarm          │                            │  │  │
│  │  │  │  CPU >= 90%     │                            │  │  │
│  │  │  └────────┬────────┘                            │  │  │
│  │  └───────────┼─────────────────────────────────────┘  │  │
│  └──────────────┼─────────────────────────────────────────┘  │
│                 │                                             │
│        ┌────────▼────────┐                                    │
│        │  SNS Topic      │                                    │
│        │  Notifications  │                                    │
│        └─────────────────┘                                    │
│                                                                │
│  ┌───────────────────────────────────────────────────────┐   │
│  │      Private VPC (devops-priv-vpc)                    │   │
│  │      CIDR: 10.0.0.0/16                                │   │
│  │  ┌─────────────────────────────────────────────────┐  │   │
│  │  │  Private Subnet (10.0.1.0/24)                   │  │   │
│  │  │  ┌──────────────────┐                            │  │   │
│  │  │  │  EC2 Instance    │                            │  │   │
│  │  │  │  devops-priv-ec2 │ (No Public IP)             │  │   │
│  │  │  │  t2.micro        │                            │  │   │
│  │  │  └──────────────────┘                            │  │   │
│  │  └─────────────────────────────────────────────────┘  │   │
│  └───────────────────────────────────────────────────────┘   │
│                                                                │
│  ┌───────────────────────────────────────────────────────┐   │
│  │                  IAM & DynamoDB                       │   │
│  │  ┌──────────────┐   ┌────────────────┐               │   │
│  │  │  IAM Role    │───│  IAM Policy    │               │   │
│  │  │  datacenter- │   │  Read-Only     │               │   │
│  │  │  role        │   │  DynamoDB      │               │   │
│  │  └──────┬───────┘   └────────────────┘               │   │
│  │         │                                              │   │
│  │  ┌──────▼────────────────────────┐                    │   │
│  │  │  DynamoDB Table               │                    │   │
│  │  │  datacenter-table             │                    │   │
│  │  │  (GetItem, Scan, Query only)  │                    │   │
│  │  └───────────────────────────────┘                    │   │
│  └───────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## Conclusion

This project demonstrates a successful incremental migration to AWS cloud infrastructure using Terraform. By breaking down the migration into manageable tasks, the Nautilus DevOps team achieved:

- **Controlled Deployment:** Each component was tested and verified before proceeding
- **Security:** Implemented multiple layers of security (VPCs, security groups, IAM policies)
- **Monitoring:** Set up proactive monitoring with CloudWatch and SNS
- **Scalability:** Infrastructure is ready to scale as needed
- **Reproducibility:** All infrastructure is defined as code and version-controlled

The incremental approach proved effective in minimizing risks while maintaining operational continuity throughout the migration process.

---

## Repository Structure

```
terraform-aws-migration/
├── README.md
├── task-1-vpc/
│   └── main.tf
├── task-2-security-group/
│   └── main.tf
├── task-3-ec2-instance/
│   └── main.tf
├── task-4-iam-policy/
│   └── main.tf
├── task-5-private-vpc/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── task-6-dynamodb-iam/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── terraform.tfvars
└── task-7-cloudwatch-monitoring/
    ├── main.tf
    └── outputs.tf
```

---
## Contact

For questions or feedback, please open an issue in this repository.
