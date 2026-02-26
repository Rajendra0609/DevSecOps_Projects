# INFRASTRUCTURE AS CODE WITH TERRAFORM

## Project Overview

This project demonstrates using Terraform for Infrastructure as Code (IaC) to provision and manage AWS infrastructure. Terraform allows you to define, provision, and manage cloud infrastructure using a declarative configuration language.

### Objectives
1. Install and configure Terraform
2. Create Terraform configurations for AWS resources
3. Implement security best practices in Terraform
4. Use Terraform modules for reusable infrastructure
5. Implement state management and backends
6. Use Terraform with CI/CD pipelines

## Prerequisites
- AWS Account with appropriate IAM permissions
- AWS CLI installed and configured
- SSH key pair for EC2 instances

## Step 1: Install Terraform

### Install Terraform on Ubuntu/Debian

```
bash
# Add HashiCorp GPG key
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

# Add HashiCorp repository
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

# Update and install Terraform
sudo apt update
sudo apt install terraform -y

# Verify installation
terraform version
```

### Install Terraform on macOS

```
bash
# Using Homebrew
brew install terraform

# Verify installation
terraform version
```

## Step 2: Configure AWS Credentials

```
bash
# Configure AWS CLI
aws configure

# Or set environment variables
export AWS_ACCESS_KEY_ID="your_access_key"
export AWS_SECRET_ACCESS_KEY="your_secret_key"
export AWS_DEFAULT_REGION="us-east-1"

# Verify credentials
aws sts get-caller-identity
```

## Step 3: Create Basic Terraform Configuration

```
bash
# Create project directory
mkdir -p terraform-project && cd terraform-project

# Create main configuration file
cat > main.tf <<EOF
provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Environment = var.environment
      Project     = "DevOps-Learning"
      ManagedBy   = "Terraform"
    }
  }
}

# Get latest Amazon Linux 2 AMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# Create VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name = "\${var.environment}-vpc"
  }
}

# Create Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name = "\${var.environment}-igw"
  }
}

# Create Public Subnets
resource "aws_subnet" "public_1" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_1
  availability_zone       = "\${var.aws_region}a"
  map_public_ip_on_launch = true
  
  tags = {
    Name = "\${var.environment}-public-subnet-1"
    Type = "Public"
  }
}

resource "aws_subnet" "public_2" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_2
  availability_zone       = "\${var.aws_region}b"
  map_public_ip_on_launch = true
  
  tags = {
    Name = "\${var.environment}-public-subnet-2"
    Type = "Public"
  }
}

# Create Private Subnets
resource "aws_subnet" "private_1" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_1
  availability_zone = "\${var.aws_region}a"
  
  tags = {
    Name = "\${var.environment}-private-subnet-1"
    Type = "Private"
  }
}

resource "aws_subnet" "private_2" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_2
  availability_zone = "\${var.aws_region}b"
  
  tags = {
    Name = "\${var.environment}-private-subnet-2"
    Type = "Private"
  }
}

# Create Elastic IP for NAT Gateway
resource "aws_eip" "nat" {
  domain = "vpc"
  
  tags = {
    Name = "\${var.environment}-nat-eip"
  }
}

# Create NAT Gateway
resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public_1.id
  
  tags = {
    Name = "\${var.environment}-nat-gw"
  }
  
  depends_on = [aws_internet_gateway.main]
}

# Create Public Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  tags = {
    Name = "\${var.environment}-public-rt"
  }
}

# Associate Public Subnets with Public Route Table
resource "aws_route_table_association" "public_1" {
  subnet_id       = aws_subnet.public_1.id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "public_2" {
  subnet_id       = aws_subnet.public_2.id
  route_table_id = aws_route_table.public.id
}

# Create Private Route Table
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main.id
  }
  
  tags = {
    Name = "\${var.environment}-private-rt"
  }
}

# Associate Private Subnets with Private Route Table
resource "aws_route_table_association" "private_1" {
  subnet_id       = aws_subnet.private_1.id
  route_table_id = aws_route_table.private.id
}

resource "aws_route_table_association" "private_2" {
  subnet_id       = aws_subnet.private_2.id
  route_table_id = aws_route_table.private.id
}

# Create Security Group
resource "aws_security_group" "web" {
  name        = "\${var.environment}-web-sg"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id
  
  # Inbound rules
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
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  # Outbound rules
  egress {
    description = "All outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name = "\${var.environment}-web-sg"
  }
}

# Create EC2 Key Pair
resource "aws_key_pair" "deployer" {
  key_name   = "\${var.environment}-key"
  public_key = var.public_key
  
  tags = {
    Name = "\${var.environment}-key"
  }
}

# Create Web Server 1
resource "aws_instance" "web_1" {
  ami               = data.aws_ami.amazon_linux.id
  instance_type     = var.instance_type
  subnet_id         = aws_subnet.public_1.id
  key_name          = aws_key_pair.deployer.key_name
  security_groups   = [aws_security_group.web.id]
  user_data         = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              echo "<h1>Web Server 1</h1>" > /var/www/html/index.html
              EOF
  
  tags = {
    Name = "\${var.environment}-web-1"
  }
}

# Create Web Server 2
resource "aws_instance" "web_2" {
  ami               = data.aws_ami.amazon_linux.id
  instance_type     = var.instance_type
  subnet_id         = aws_subnet.public_2.id
  key_name          = aws_key_pair.deployer.key_name
  security_groups   = [aws_security_group.web.id]
  user_data         = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              echo "<h1>Web Server 2</h1>" > /var/www/html/index.html
              EOF
  
  tags = {
    Name = "\${var.environment}-web-2"
  }
}

# Create Application Load Balancer
resource "aws_lb" "main" {
  name               = "\${var.environment}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.web.id]
  subnets            = [aws_subnet.public_1.id, aws_subnet.public_2.id]
  
  enable_deletion_protection = false
  
  tags = {
    Name = "\${var.environment}-alb"
  }
}

# Create Target Group
resource "aws_lb_target_group" "main" {
  name     = "\${var.environment}-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id
  
  health_check {
    enabled             = true
    healthy_threshold   = 2
    unhealthy_threshold = 2
    timeout             = 5
    interval            = 30
    path                = "/"
    matcher             = "200"
  }
}

# Register targets
resource "aws_lb_target_group_attachment" "web_1" {
  target_group_arn = aws_lb_target_group.main.arn
  target_id        = aws_instance.web_1.id
  port             = 80
}

resource "aws_lb_target_group_attachment" "web_2" {
  target_group_arn = aws_lb_target_group.main.arn
  target_id        = aws_instance.web_2.id
  port             = 80
}

# Create ALB Listener
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.main.arn
  port              = "80"
  protocol          = "HTTP"
  
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.main.arn
  }
}

# Outputs
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "web_server_1_ip" {
  description = "Public IP of Web Server 1"
  value       = aws_instance.web_1.public_ip
}

output "web_server_2_ip" {
  description = "Public IP of Web Server 2"
  value       = aws_instance.web_2.public_ip
}

output "alb_dns_name" {
  description = "DNS name of the Application Load Balancer"
  value       = aws_lb.main.dns_name
}
EOF
```

### Create Variables File

```
bash
# Create variables file
cat > variables.tf <<EOF
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnet_1" {
  description = "CIDR block for public subnet 1"
  type        = string
  default     = "10.0.1.0/24"
}

variable "public_subnet_2" {
  description = "CIDR block for public subnet 2"
  type        = string
  default     = "10.0.2.0/24"
}

variable "private_subnet_1" {
  description = "CIDR block for private subnet 1"
  type        = string
  default     = "10.0.10.0/24"
}

variable "private_subnet_2" {
  description = "CIDR block for private subnet 2"
  type        = string
  default     = "10.0.11.0/24"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}

variable "public_key" {
  description = "SSH public key"
  type        = string
}
EOF
```

### Create tfvars File

```
bash
# Create environment-specific tfvars
cat > dev.tfvars <<EOF
aws_region    = "us-east-1"
environment   = "dev"
instance_type = "t3.micro"
public_key    = "ssh-rsa AAAAB3NzaC1..."
EOF
```

## Step 4: Terraform Commands

```
bash
# Initialize Terraform
terraform init

# Format code
terraform fmt

# Validate configuration
terraform validate

# Plan changes
terraform plan -var-file="dev.tfvars"

# Apply changes
terraform apply -var-file="dev.tfvars"

# Apply with auto-approve
terraform apply -var-file="dev.tfvars" -auto-approve

# Show current state
terraform show

# List resources
terraform state list

# Destroy resources
terraform destroy -var-file="dev.tfvars"
```

## Step 5: Terraform Modules

```
bash
# Create modules directory
mkdir -p modules/vpc modules/ec2 modules/alb

# VPC Module
cat > modules/vpc/main.tf <<EOF
variable "environment" {
  description = "Environment name"
  type        = string
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
}

variable "aws_region" {
  description = "AWS region"
  type        = string
}

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name = "\${var.environment}-vpc"
  }
}

resource "aws_subnet" "public" {
  count                = 2
  vpc_id               = aws_vpc.main.id
  cidr_block           = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone    = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  
  tags = {
    Name = "\${var.environment}-public-subnet-\${count.index + 1}"
  }
}

data "aws_availability_zones" "available" {
  state = "available"
}

output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}
EOF

# Update main.tf to use module
cat > main.tf <<EOF
provider "aws" {
  region = "us-east-1"
}

module "vpc" {
  source   = "./modules/vpc"
  environment = "dev"
  vpc_cidr = "10.0.0.0/16"
  aws_region = "us-east-1"
}
EOF
```

## Step 6: Remote State Backend

```
bash
# Create backend configuration
cat > backend.tf <<EOF
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "dev/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
EOF
```

### Create S3 Bucket and DynamoDB Table

```
bash
# Create S3 bucket for state storage
aws s3 mb s3://my-terraform-state-bucket --region us-east-1

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket my-terraform-state-bucket \
  --versioning-configuration Status=Enabled

# Enable encryption
aws s3api put-bucket-encryption \
  --bucket my-terraform-state-bucket \
  --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'

# Create DynamoDB table for state locking
aws dynamodb create-table \
  --table-name terraform-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

## Step 7: Security Best Practices

### Enable MFA for AWS Credentials

```
bash
# Add MFA requirement to Terraform
cat > main.tf <<EOF
provider "aws" {
  region = "us-east-1"
  
  # Require MFA for console login
  assume_role {
    role_arn = "arn:aws:iam::123456789012:role/TerraformRole"
    session_name = "terraform-session"
  }
}
EOF
```

### Secrets Management

```
bash
# Use AWS Secrets Manager with Terraform
resource "aws_secretsmanager_secret" "db_password" {
  name = "dev/db_password"
  
  recovery_window_in_days = 0
}

resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id = aws_secretsmanager_secret.db_password.id
  secret_string = jsonencode({
    username = "admin"
    password = "secure-password"
  })
}
```

## Step 8: CI/CD Integration

```
bash
# Create GitHub Actions workflow
mkdir -p .github/workflows

cat > .github/workflows/terraform.yml <<EOF
name: Terraform CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  terraform:
    name: Terraform
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.0
      
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: \${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: \${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Terraform Format
        run: terraform fmt -check
      
      - name: Terraform Init
        run: terraform init
      
      - name: Terraform Validate
        run: terraform validate
      
      - name: Terraform Plan
        run: terraform plan -var-file="dev.tfvars"
        continue-on-error: true
      
      - name: Terraform Apply
        if: github.ref == 'refs/heads/main'
        run: terraform apply -var-file="dev.tfvars" -auto-approve
EOF
```

## Step 9: Workspaces

```
bash
# Create new workspace
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

# List workspaces
terraform workspace list

# Select workspace
terraform workspace select dev

# Use workspace in configuration
resource "aws_instance" "example" {
  ami           = "ami-12345678"
  instance_type = terraform.workspace == "prod" ? "t3.large" : "t3.micro"
  
  tags = {
    Name = "example-\${terraform.workspace}"
  }
}
```

## Conclusion

This project demonstrates:
- Installing and configuring Terraform
- Creating AWS infrastructure with Terraform
- Using variables and outputs
- Implementing Terraform modules
- Configuring remote state backend with S3
- Security best practices
- CI/CD integration with GitHub Actions
- Using Terraform workspaces

## Additional Resources
- [Terraform Documentation](https://www.terraform.io/docs)
- [AWS Provider Documentation](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Terraform Best Practices](https://www.terraform.io/docs/cloud/guides/recommended-practices/index.html)
