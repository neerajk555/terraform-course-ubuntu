# Lesson 8: Using Data Sources

## Learning Objectives
- Learn how data sources query existing information in AWS.
- Use data sources to dynamically select AMIs and subnets.
- Reduce hardcoding in your configuration.

## Concept Overview

### What Are Data Sources? 🔍

Imagine you're planning a road trip, but instead of memorizing every street address, you use GPS to find locations dynamically. **Data sources** are Terraform's GPS - they look up existing information in your cloud environment.

### Data Sources vs Resources: The Key Difference

```
┌─────────────────┬─────────────────────────────────────────┐
│ RESOURCES       │ DATA SOURCES                            │
├─────────────────┼─────────────────────────────────────────┤
│ CREATE things   │ FIND existing things                    │
│ Terraform owns  │ Someone else owns                       │
│ Can be modified │ Read-only information                   │
│ Cost money      │ Free queries                           │
│ "I want..."     │ "What exists?"                         │
└─────────────────┴─────────────────────────────────────────┘
```

### Why Use Data Sources?

#### 🎯 **Problem:** Hardcoded Values Break
```hcl
# BAD: Hardcoded AMI (breaks in different regions)
resource "aws_instance" "web" {
  ami = "ami-0c02fb55956c7d316"  # Only works in us-east-1!
}
```

#### ✅ **Solution:** Dynamic Lookup
```hcl
# GOOD: Find the latest AMI automatically
data "aws_ami" "latest" {
  most_recent = true
  owners      = ["amazon"]
  # ... filters ...
}

resource "aws_instance" "web" {
  ami = data.aws_ami.latest.id  # Works in any region!
}
```

### Real-World Data Source Examples

#### 📡 **Infrastructure Discovery**
- "What's my default VPC ID?"
- "Which subnets are available?"
- "What availability zones exist here?"

#### 🖥️ **Image Management**  
- "What's the latest Ubuntu AMI?"
- "Find Windows Server 2022 image"
- "Get the most recent Amazon Linux"

#### 🔐 **Security & Access**
- "What's my current AWS account ID?"
- "Find existing IAM role ARN"
- "Get SSL certificate details"

#### 🌍 **Environment Information**
- "List all regions"
- "What instance types are available?"
- "Get current IP address ranges"

### How Data Sources Work

```
1. Terraform reads your data source configuration
2. Makes API call to AWS (during plan/apply)
3. Filters results based on your criteria  
4. Returns the information you requested
5. You use that info in resources or outputs
```

## Step-by-Step Tutorial

### Step 1: Set Up Your Project

```bash
# Navigate to course directory
cd ~/terraform-course

# Create lesson folder
mkdir lesson-8
cd lesson-8
```

### Step 2: Create a Data Source Configuration

Let's build a configuration that uses multiple data sources to create portable, region-agnostic infrastructure:

```bash
cat > main.tf << 'EOF'
terraform {
  required_version = ">= 1.5.0"
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

# Data Source 1: Find the default VPC
data "aws_vpc" "default" {
  default = true
}

# Data Source 2: Find subnets in the default VPC
data "aws_subnet_ids" "default" {
  vpc_id = data.aws_vpc.default.id
}

# Data Source 3: Get current AWS region info
data "aws_region" "current" {}

# Data Source 4: Get current AWS account info
data "aws_caller_identity" "current" {}

# Data Source 5: Find the latest Amazon Linux 2 AMI
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
  
  filter {
    name   = "architecture"
    values = ["x86_64"]
  }
  
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Data Source 6: Get available AZs in current region
data "aws_availability_zones" "available" {
  state = "available"
}

# Use data sources to create an instance
resource "aws_instance" "web" {
  ami                         = data.aws_ami.amazon_linux_2.id
  instance_type               = "t2.micro"
  subnet_id                   = element(data.aws_subnet_ids.default.ids, 0)
  associate_public_ip_address = true

  tags = {
    Name        = "lesson-8-server"
    Region      = data.aws_region.current.name
    AccountId   = data.aws_caller_identity.current.account_id
    AMI         = data.aws_ami.amazon_linux_2.name
  }
}

# Output all the discovered information
output "discovered_info" {
  value = {
    # Region information
    region_name        = data.aws_region.current.name
    region_description = data.aws_region.current.description
    
    # Account information
    account_id = data.aws_caller_identity.current.account_id
    user_id    = data.aws_caller_identity.current.user_id
    
    # Network information
    vpc_id            = data.aws_vpc.default.id
    vpc_cidr          = data.aws_vpc.default.cidr_block
    available_azs     = data.aws_availability_zones.available.names
    subnet_count      = length(data.aws_subnet_ids.default.ids)
    
    # AMI information
    ami_id           = data.aws_ami.amazon_linux_2.id
    ami_name         = data.aws_ami.amazon_linux_2.name
    ami_description  = data.aws_ami.amazon_linux_2.description
    ami_creation_date = data.aws_ami.amazon_linux_2.creation_date
    
    # Instance information
    instance_id  = aws_instance.web.id
    instance_ip  = aws_instance.web.public_ip
  }
}
EOF
```

### Step 3: Understand Each Data Source

Let's break down what each data source does:

#### **VPC Discovery**
```hcl
data "aws_vpc" "default" {
  default = true                    # Find the default VPC
}
```
**What it finds:** The VPC that AWS automatically creates in every region
**Why useful:** You don't need to hardcode VPC IDs

#### **Subnet Discovery**  
```hcl
data "aws_subnet_ids" "default" {
  vpc_id = data.aws_vpc.default.id  # Find subnets in the default VPC
}
```
**What it finds:** All subnet IDs in the specified VPC
**Why useful:** Automatically adapts to different AZ configurations

#### **AMI Discovery with Filters**
```hcl
data "aws_ami" "amazon_linux_2" {
  most_recent = true               # Get the newest matching AMI
  owners      = ["amazon"]         # Only AMIs owned by Amazon
  
  filter {
    name   = "name"               
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]  # Amazon Linux 2 pattern
  }
}
```
**What it finds:** The latest Amazon Linux 2 AMI in your region
**Why useful:** Always gets security updates and patches

#### **Region & Account Info**
```hcl
data "aws_region" "current" {}          # Current region details
data "aws_caller_identity" "current" {} # Your AWS account info
```
**What they find:** Metadata about where and who you are
**Why useful:** Dynamic tagging and environment-aware configurations

### Step 4: Deploy and Explore

```bash
# Initialize and apply
terraform init
terraform apply -auto-approve

# View all discovered information
terraform output discovered_info
```

**Expected output example:**
```json
{
  "account_id" = "123456789012"
  "ami_creation_date" = "2023-10-01T00:00:00.000Z"
  "ami_id" = "ami-0abcdef1234567890"
  "region_name" = "us-east-1"
  "vpc_cidr" = "172.31.0.0/16"
  ...
}
```

### Step 5: Test Portability

Let's test how portable this configuration is by changing regions:

```bash
# Modify the provider region
sed -i 's/us-east-1/us-west-2/g' main.tf

# Plan the changes
terraform plan

# Notice: Different AMI ID, VPC ID, subnets - but same logic!
```

**Key observation:** The configuration works in any region without hardcoded values!

## Code Examples

### Advanced Data Source Patterns

#### **Conditional Data Sources**
```hcl
# Find Ubuntu AMI if we're not using Amazon Linux
data "aws_ami" "ubuntu" {
  count       = var.use_ubuntu ? 1 : 0
  most_recent = true
  owners      = ["099720109477"] # Canonical
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}
```

#### **Multiple Filter Combinations**
```hcl
# Find specific instance type availability
data "aws_ec2_instance_type" "example" {
  instance_type = "t3.micro"
}

# Find AMI by tag
data "aws_ami" "custom" {
  most_recent = true
  owners      = ["self"]  # Your account only
  
  filter {
    name   = "tag:Environment"
    values = ["production"]
  }
  
  filter {
    name   = "tag:Application"
    values = ["web-server"]
  }
}
```

#### **Cross-Reference Data Sources**
```hcl
# Find security group by name
data "aws_security_group" "default" {
  name   = "default"
  vpc_id = data.aws_vpc.default.id
}

# Find subnet in specific AZ
data "aws_subnet" "specific_az" {
  vpc_id            = data.aws_vpc.default.id
  availability_zone = "us-east-1a"
}
```

## Hands-On Exercise

### Exercise 1: AMI Comparison

Compare different AMI types and their properties:

```bash
# Add these data sources to your main.tf
cat >> main.tf << 'EOF'

# Find different AMI types
data "aws_ami" "ubuntu_22" {
  most_recent = true
  owners      = ["099720109477"] # Canonical
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

data "aws_ami" "windows" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["Windows_Server-2022-English-Full-Base-*"]
  }
}

output "ami_comparison" {
  value = {
    amazon_linux = {
      id   = data.aws_ami.amazon_linux_2.id
      name = data.aws_ami.amazon_linux_2.name
    }
    ubuntu = {
      id   = data.aws_ami.ubuntu_22.id
      name = data.aws_ami.ubuntu_22.name
    }
    windows = {
      id   = data.aws_ami.windows.id
      name = data.aws_ami.windows.name
    }
  }
}
EOF

# Apply and compare
terraform apply -auto-approve
terraform output ami_comparison
```

### Exercise 2: Network Discovery

Explore your network infrastructure:

```bash
# Create a new file for network exploration
cat > network_discovery.tf << 'EOF'
# Get detailed subnet information
data "aws_subnet" "all_subnets" {
  for_each = data.aws_subnet_ids.default.ids
  id       = each.value
}

output "network_details" {
  value = {
    for subnet_id, subnet in data.aws_subnet.all_subnets : subnet_id => {
      availability_zone = subnet.availability_zone
      cidr_block       = subnet.cidr_block
      map_public_ip    = subnet.map_public_ip_on_launch
    }
  }
}
EOF

terraform apply -auto-approve
terraform output network_details
```

### Exercise 3: Region Portability Test

Test your configuration in different regions:

```bash
# Test in us-west-2
terraform plan -var="region=us-west-2"

# Apply in us-west-2 (creates separate infrastructure)
terraform workspace new us-west-2
terraform apply -var="region=us-west-2" -auto-approve

# Compare outputs between regions
terraform output discovered_info

# Switch back to default workspace
terraform workspace select default
```

## Troubleshooting Tips
- If no AMI found: Verify filters and region.
- Permissions errors: Ensure your IAM user has describe permissions (typical for most AWS policies).

## Summary / Key Takeaways
- Data sources make your configs dynamic and portable.
- You reduced hard-coded values and learned to query AWS metadata.

## Next Steps
Package Terraform code into reusable modules.