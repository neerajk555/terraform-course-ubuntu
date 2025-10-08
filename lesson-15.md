# Lesson 15: Final Project — Deploy a Scalable Web App Infrastructure on AWS

## Learning Objectives
- Build a production-style environment with VPC, subnets, security groups, ALB, and Auto Scaling.
- Use Terraform Registry modules to move fast.
- Validate the deployment and understand the architecture.

## Concept Overview

### 🎯 What We're Building: Production-Ready Web Application Infrastructure

Congratulations! You've reached the capstone project. Today, you'll deploy a **real production-style architecture** that could handle thousands of users. This isn't just a single server - it's a complete, scalable, highly available system.

### 🏗️ Architecture Overview

```
Internet
    │
    ├── Application Load Balancer (ALB)
    │   ├── Public Subnet AZ-1a
    │   └── Public Subnet AZ-1b
    │
    └── Private Network
        ├── Web Server 1 (AZ-1a)
        ├── Web Server 2 (AZ-1b)
        └── Auto Scaling (adds/removes servers as needed)
```

### 🧩 Components We'll Deploy

#### **🌐 Networking Layer**
- **Custom VPC:** Your own private network (instead of default)
- **Public Subnets:** Where the load balancer lives (internet-accessible)
- **Private Subnets:** Where web servers live (protected from direct internet access)
- **Multi-AZ:** Spans 2 availability zones for high availability
- **NAT Gateway:** Allows private servers to reach the internet for updates

#### **⚖️ Load Balancing Layer**
- **Application Load Balancer (ALB):** Distributes traffic across servers
- **Health Checks:** Automatically removes unhealthy servers
- **Public-facing:** Gets a DNS name you can visit in your browser

#### **💻 Compute Layer**
- **Auto Scaling Group:** Automatically adds/removes servers based on load
- **Launch Template:** Blueprint for creating identical servers
- **User Data Script:** Automatically installs web server on each instance
- **Multi-AZ deployment:** Servers spread across availability zones

#### **🔒 Security Layer**
- **ALB Security Group:** Only allows HTTP traffic from internet
- **App Security Group:** Only allows traffic from the load balancer
- **Principle of least privilege:** Each component can only access what it needs

### 💡 Why This Architecture?

#### **High Availability** 🔄
- If one availability zone fails, the other keeps serving traffic
- If one server fails, others continue serving requests

#### **Scalability** 📈
- Auto Scaling automatically adds servers during traffic spikes
- Load balancer distributes requests evenly

#### **Security** 🛡️
- Web servers aren't directly accessible from internet
- Each layer has appropriate security controls

#### **Cost Efficiency** 💰
- Auto Scaling removes servers when traffic is low
- Only pay for what you actually need

### 🚀 Real-World Applications

This exact architecture pattern is used by:
- **E-commerce sites** handling holiday shopping spikes
- **News websites** during breaking news events  
- **Startups** that need to scale quickly
- **Enterprise applications** requiring high availability

By the end of this lesson, you'll have built the same infrastructure that powers major web applications!

## Step-by-Step Tutorial

### Step 1: Project Setup and Planning

#### Create Your Final Project Workspace
```bash
# Navigate to your course directory
cd ~/terraform-course

# Create the final project directory
mkdir final-project
cd final-project

# Create a project structure for organization
mkdir -p {modules,environments}
ls -la
```

#### Create a Project README
```bash
cat > README.md << 'EOF'
# Scalable Web Application Infrastructure

This Terraform project deploys a production-ready, scalable web application infrastructure on AWS.

## Architecture
- Application Load Balancer for traffic distribution
- Auto Scaling Group across multiple Availability Zones
- Custom VPC with public/private subnets
- Security groups following least privilege principle

## Components
- 1 Application Load Balancer
- 2-3 EC2 instances (auto-scaled)
- 1 Custom VPC with 4 subnets
- Multiple security groups
- 1 NAT Gateway for outbound internet access

## Estimated Cost
- ~$20-30/month if left running continuously
- Most components are free-tier eligible for new AWS accounts
EOF
```

### Step 2: Create Variables Configuration

First, let's define all the configurable values:

```bash
cat > variables.tf << 'EOF'
# Project Configuration
variable "project_name" {
  type        = string
  description = "Project name prefix for all resources"
  default     = "final-webapp"
  
  validation {
    condition     = length(var.project_name) <= 10
    error_message = "Project name must be 10 characters or less for resource naming limits."
  }
}

# AWS Configuration
variable "region" {
  type        = string
  description = "AWS region to deploy resources"
  default     = "us-east-1"
}

# Instance Configuration  
variable "instance_type" {
  type        = string
  description = "EC2 instance type for app servers"
  default     = "t3.micro"
}

variable "min_size" {
  type        = number
  description = "Minimum number of instances in Auto Scaling Group"
  default     = 2
}

variable "max_size" {
  type        = number
  description = "Maximum number of instances in Auto Scaling Group"
  default     = 5
}

variable "desired_capacity" {
  type        = number
  description = "Desired number of instances in Auto Scaling Group"
  default     = 2
}

# Network Configuration
variable "azs" {
  type        = list(string)
  description = "Availability Zones"
  default     = ["us-east-1a", "us-east-1b"]
}

variable "public_subnets" {
  type        = list(string)
  description = "CIDR blocks for public subnets"
  default     = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "private_subnets" {
  type        = list(string)
  description = "CIDR blocks for private subnets"
  default     = ["10.0.101.0/24", "10.0.102.0/24"]
}

# Application Configuration
variable "app_port" {
  type        = number
  description = "Port the application runs on"
  default     = 80
}

variable "health_check_path" {
  type        = string
  description = "Health check path for load balancer"
  default     = "/"
}
EOF
```

### Step 3: Create Main Infrastructure Configuration

Now for the main infrastructure. We'll use well-tested community modules:

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
  region = var.region
  
  default_tags {
    tags = {
      Project     = var.project_name
      Environment = "production"
      ManagedBy   = "terraform"
      CreatedBy   = "terraform-course"
    }
  }
}

# Get current AWS region and account info
data "aws_region" "current" {}
data "aws_caller_identity" "current" {}

# 1) VPC Module - Creates our network foundation
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "${var.project_name}-vpc"
  cidr = "10.0.0.0/16"

  azs             = var.azs
  public_subnets  = var.public_subnets
  private_subnets = var.private_subnets

  # Enable NAT Gateway for private subnet internet access
  enable_nat_gateway = true
  single_nat_gateway = true  # Cost optimization: 1 NAT instead of 1 per AZ

  # Enable DNS
  enable_dns_hostnames = true
  enable_dns_support   = true

  # Tagging
  tags = {
    Name = "${var.project_name}-vpc"
  }
  
  public_subnet_tags = {
    Type = "public"
    Tier = "web"
  }
  
  private_subnet_tags = {
    Type = "private"
    Tier = "app"
  }
}

# 2) Security Group for Application Load Balancer
module "sg_alb" {
  source  = "terraform-aws-modules/security-group/aws"
  version = "~> 5.1"

  name        = "${var.project_name}-alb-sg"
  description = "Security group for Application Load Balancer"
  vpc_id      = module.vpc.vpc_id

  # Ingress rules
  ingress_rules       = ["http-80-tcp", "https-443-tcp"]
  ingress_cidr_blocks = ["0.0.0.0/0"]

  # Egress rules
  egress_rules = ["all-all"]
}

# 3) Security Group for Application Servers
module "sg_app" {
  source  = "terraform-aws-modules/security-group/aws"
  version = "~> 5.1"

  name        = "${var.project_name}-app-sg"
  description = "Security group for application servers"
  vpc_id      = module.vpc.vpc_id

  # Only allow HTTP traffic from ALB
  ingress_with_source_security_group_id = [
    {
      rule                     = "http-80-tcp"
      source_security_group_id = module.sg_alb.security_group_id
      description              = "HTTP from ALB"
    }
  ]

  # Allow all outbound (for package updates, etc.)
  egress_rules = ["all-all"]
}

# 4) Application Load Balancer
module "alb" {
  source  = "terraform-aws-modules/alb/aws"
  version = "~> 9.8"

  name               = "${var.project_name}-alb"
  load_balancer_type = "application"
  vpc_id             = module.vpc.vpc_id
  subnets            = module.vpc.public_subnets
  security_groups    = [module.sg_alb.security_group_id]

  # Target group configuration
  target_groups = [
    {
      name_prefix      = "app-"
      backend_protocol = "HTTP"
      backend_port     = var.app_port
      target_type      = "instance"
      
      health_check = {
        enabled             = true
        healthy_threshold   = 2
        interval            = 30
        matcher             = "200"
        path                = var.health_check_path
        port                = "traffic-port"
        protocol            = "HTTP"
        timeout             = 5
        unhealthy_threshold = 2
      }
      
      stickiness = {
        enabled = false
        type    = "lb_cookie"
      }
    }
  ]

  # Listener configuration
  listeners = [
    {
      port               = 80
      protocol           = "HTTP"
      target_group_index = 0
      
      action_type = "forward"
    }
  ]
}

# 5) Get the latest Amazon Linux 2 AMI
data "aws_ami" "amazon_linux_2" {
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

# 6) User data script to set up web server
locals {
  user_data = base64encode(<<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              
              # Get instance metadata
              INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
              REGION=$(curl -s http://169.254.169.254/latest/meta-data/placement/region)
              AZ=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)
              
              # Create a simple web page
              cat > /var/www/html/index.html <<HTML
<!DOCTYPE html>
<html>
<head>
    <title>${var.project_name} - Scalable Web App</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; background: #f0f8ff; }
        .container { max-width: 800px; margin: 0 auto; background: white; padding: 30px; border-radius: 10px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); }
        .header { color: #333; border-bottom: 2px solid #007acc; padding-bottom: 10px; }
        .info { background: #e7f3ff; padding: 15px; border-radius: 5px; margin: 20px 0; }
        .success { color: #28a745; font-weight: bold; }
    </style>
</head>
<body>
    <div class="container">
        <h1 class="header">🚀 Terraform Course - Final Project</h1>
        <p class="success">✅ Congratulations! Your scalable web application is running successfully!</p>
        
        <div class="info">
            <h3>Server Information:</h3>
            <ul>
                <li><strong>Instance ID:</strong> $INSTANCE_ID</li>
                <li><strong>Region:</strong> $REGION</li>
                <li><strong>Availability Zone:</strong> $AZ</li>
                <li><strong>Project:</strong> ${var.project_name}</li>
                <li><strong>Deployed via:</strong> Terraform</li>
            </ul>
        </div>
        
        <h3>Architecture Features:</h3>
        <ul>
            <li>🌐 Custom VPC with public/private subnets</li>
            <li>⚖️ Application Load Balancer for high availability</li>
            <li>📈 Auto Scaling Group for automatic scaling</li>
            <li>🔒 Security groups with least privilege access</li>
            <li>🔄 Multi-AZ deployment for fault tolerance</li>
        </ul>
        
        <p><em>Refresh this page multiple times to see different server instances!</em></p>
    </div>
</body>
</html>
HTML
              
              # Create a health check endpoint
              cat > /var/www/html/health.html <<HTML
<!DOCTYPE html>
<html><body><h1>OK</h1><p>Server is healthy</p></body></html>
HTML
              
              EOF
  )
}

# 7) Launch Template for Auto Scaling
resource "aws_launch_template" "app" {
  name_prefix   = "${var.project_name}-lt-"
  image_id      = data.aws_ami.amazon_linux_2.id
  instance_type = var.instance_type
  
  vpc_security_group_ids = [module.sg_app.security_group_id]
  user_data              = local.user_data
  
  tag_specifications {
    resource_type = "instance"
    tags = {
      Name = "${var.project_name}-app-server"
      Type = "application-server"
    }
  }
  
  lifecycle {
    create_before_destroy = true
  }
}

# 8) Auto Scaling Group
resource "aws_autoscaling_group" "app" {
  name                = "${var.project_name}-asg"
  vpc_zone_identifier = module.vpc.private_subnets
  target_group_arns   = [module.alb.target_group_arns[0]]
  health_check_type   = "ELB"
  health_check_grace_period = 300
  
  min_size         = var.min_size
  max_size         = var.max_size
  desired_capacity = var.desired_capacity
  
  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }
  
  # Instance refresh for rolling updates
  instance_refresh {
    strategy = "Rolling"
    preferences {
      min_healthy_percentage = 50
    }
  }
  
  tag {
    key                 = "Name"
    value               = "${var.project_name}-asg"
    propagate_at_launch = false
  }
}

# 9) Auto Scaling Policies
resource "aws_autoscaling_policy" "scale_up" {
  name                   = "${var.project_name}-scale-up"
  scaling_adjustment     = 1
  adjustment_type        = "ChangeInCapacity"
  cooldown              = 300
  autoscaling_group_name = aws_autoscaling_group.app.name
}

resource "aws_autoscaling_policy" "scale_down" {
  name                   = "${var.project_name}-scale-down"
  scaling_adjustment     = -1
  adjustment_type        = "ChangeInCapacity"
  cooldown              = 300
  autoscaling_group_name = aws_autoscaling_group.app.name
}

# 10) CloudWatch Alarms for Auto Scaling
resource "aws_cloudwatch_metric_alarm" "cpu_high" {
  alarm_name          = "${var.project_name}-cpu-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = "120"
  statistic           = "Average"
  threshold           = "80"
  alarm_description   = "This metric monitors ec2 cpu utilization"
  alarm_actions       = [aws_autoscaling_policy.scale_up.arn]

  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.app.name
  }
}

resource "aws_cloudwatch_metric_alarm" "cpu_low" {
  alarm_name          = "${var.project_name}-cpu-low"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = "120"
  statistic           = "Average"
  threshold           = "10"
  alarm_description   = "This metric monitors ec2 cpu utilization"
  alarm_actions       = [aws_autoscaling_policy.scale_down.arn]

  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.app.name
  }
}
EOF
```

### Step 4: Create Outputs Configuration

Create comprehensive outputs to track your infrastructure:

```bash
cat > outputs.tf << 'EOF'
# Load Balancer Information
output "load_balancer_url" {
  value       = "http://${module.alb.lb_dns_name}"
  description = "URL of the Application Load Balancer - visit this in your browser!"
}

output "load_balancer_dns" {
  value       = module.alb.lb_dns_name
  description = "DNS name of the Application Load Balancer"
}

output "load_balancer_zone_id" {
  value       = module.alb.lb_zone_id
  description = "Zone ID of the Application Load Balancer (for Route53 alias records)"
}

# Network Information
output "vpc_id" {
  value       = module.vpc.vpc_id
  description = "ID of the VPC"
}

output "vpc_cidr_block" {
  value       = module.vpc.vpc_cidr_block
  description = "CIDR block of the VPC"
}

output "public_subnets" {
  value       = module.vpc.public_subnets
  description = "List of public subnet IDs"
}

output "private_subnets" {
  value       = module.vpc.private_subnets
  description = "List of private subnet IDs"
}

output "nat_gateway_ips" {
  value       = module.vpc.nat_public_ips
  description = "Public IPs of the NAT Gateways"
}

# Auto Scaling Information
output "autoscaling_group_arn" {
  value       = aws_autoscaling_group.app.arn
  description = "ARN of the Auto Scaling Group"
}

output "autoscaling_group_name" {
  value       = aws_autoscaling_group.app.name
  description = "Name of the Auto Scaling Group"
}

output "launch_template_id" {
  value       = aws_launch_template.app.id
  description = "ID of the Launch Template"
}

# Security Information
output "alb_security_group_id" {
  value       = module.sg_alb.security_group_id
  description = "ID of the ALB security group"
}

output "app_security_group_id" {
  value       = module.sg_app.security_group_id
  description = "ID of the application security group"
}

# Project Information
output "deployment_info" {
  value = {
    project_name    = var.project_name
    region          = data.aws_region.current.name
    account_id      = data.aws_caller_identity.current.account_id
    availability_zones = var.azs
    instance_type   = var.instance_type
    min_capacity    = var.min_size
    max_capacity    = var.max_size
    desired_capacity = var.desired_capacity
  }
  description = "Summary of deployment configuration"
}

# Cost Estimation (approximate)
output "estimated_monthly_cost" {
  value = {
    message = "Estimated monthly cost breakdown (USD, approximate):"
    alb = "$18-22 (Application Load Balancer)"
    nat_gateway = "$32-45 (NAT Gateway)"
    instances = "$${var.desired_capacity * 8}-${var.max_size * 8} (EC2 instances - t3.micro)"
    data_transfer = "$5-15 (Data transfer, varies by usage)"
    total_range = "$63-90 per month if running continuously"
    note = "Most costs can be reduced with Reserved Instances or by stopping instances when not needed"
  }
  description = "Estimated AWS costs for this infrastructure"
}
EOF
```

### Step 5: Create Environment-Specific Configuration

Create a production configuration file:

```bash
cat > prod.tfvars << 'EOF'
# Production Configuration
project_name = "webapp-prod"
region       = "us-east-1"

# Instance Configuration
instance_type    = "t3.small"    # Slightly larger for production
min_size         = 2
max_size         = 6
desired_capacity = 3

# Network Configuration (using different AZs for better distribution)
azs = ["us-east-1a", "us-east-1b", "us-east-1c"]
public_subnets  = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
private_subnets = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
EOF
```

### Step 6: Deploy Your Infrastructure

Now let's deploy this production-grade infrastructure:

#### Initialize the Project
```bash
terraform init
```

**Expected output:** Downloads multiple modules (VPC, ALB, Security Group)

#### Validate Configuration
```bash
terraform validate
terraform fmt
```

#### Create Deployment Plan
```bash
# Generate a detailed plan
terraform plan -out=deployment.tfplan

# Or use production configuration
terraform plan -var-file="prod.tfvars" -out=prod.tfplan
```

**Study the plan output:** You should see approximately:
- 20-25 resources to be created
- VPC with subnets across multiple AZs
- Load balancer and target groups
- Auto scaling group and launch template
- Security groups and policies
- CloudWatch alarms

#### Deploy the Infrastructure
```bash
# Deploy with default configuration
terraform apply deployment.tfplan

# OR deploy with production configuration
terraform apply prod.tfplan

# This will take 5-10 minutes to complete
```

**During deployment, you'll see:**
- VPC and networking components created first
- Security groups and load balancer next
- Launch template and auto scaling group last
- CloudWatch alarms configured

#### Get Your Results
```bash
# View all outputs
terraform output

# Get just the URL to visit
terraform output load_balancer_url

# Get deployment summary
terraform output deployment_info

# Check cost estimation
terraform output estimated_monthly_cost
```

### Step 7: Test Your Scalable Web Application

#### Visit Your Application
```bash
# Get the load balancer URL
ALB_URL=$(terraform output -raw load_balancer_url)
echo "Your application is available at: $ALB_URL"

# Test with curl
curl -I $ALB_URL

# Or open in browser (if you have a GUI)
# firefox $ALB_URL
```

#### Test Load Balancing
```bash
# Make multiple requests to see different servers
for i in {1..5}; do
  curl -s $ALB_URL | grep "Instance ID" | head -1
done
```

**You should see different instance IDs, proving load balancing works!**

#### Test Auto Scaling (Optional Advanced Test)
```bash
# Check current instances
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=final-webapp-app-server" \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name]' \
  --output table

# Trigger scaling by creating CPU load (be careful!)
# This is advanced - only do if you want to see auto scaling in action
```

#### Monitor Your Infrastructure
```bash
# Check Auto Scaling Group status
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names $(terraform output -raw autoscaling_group_name) \
  --query 'AutoScalingGroups[0].{DesiredCapacity:DesiredCapacity,Instances:length(Instances)}'

# Check load balancer target health
aws elbv2 describe-target-health \
  --target-group-arn $(terraform output -json | jq -r '.load_balancer_url.value' | xargs -I {} aws elbv2 describe-load-balancers --query 'LoadBalancers[?DNSName==`{}`].LoadBalancerArn' --output text)
```

### Step 8: Explore Your Infrastructure

#### In AWS Console
1. **EC2 Dashboard**
   - View your instances across multiple AZs
   - Check Auto Scaling Group configuration
   - Review Launch Template details

2. **Load Balancer Dashboard**
   - View ALB configuration
   - Check target group health
   - Monitor request metrics

3. **VPC Dashboard**
   - Explore your custom VPC
   - See public/private subnet layout
   - Check route tables and NAT Gateway

4. **CloudWatch Dashboard**
   - View auto scaling alarms
   - Monitor CPU metrics
   - Check load balancer metrics

### Step 9: Understanding the Cost

Your infrastructure includes these billable components:

#### Always-On Costs (24/7):
- **ALB:** ~$18/month (load balancer)
- **NAT Gateway:** ~$32/month (internet access for private subnets)
- **EBS Storage:** ~$2/month (for instance storage)

#### Variable Costs:
- **EC2 Instances:** $8.50/month per t3.micro instance
- **Data Transfer:** Depends on usage
- **CloudWatch:** Minimal for basic metrics

#### Cost Optimization Tips:
```bash
# Stop instances when not needed (development only!)
terraform destroy -target="aws_autoscaling_group.app"

# Or reduce capacity
terraform apply -var="desired_capacity=1" -var="min_size=1"
```

### Step 10: Clean Up Resources ⚠️ CRITICAL

This final project creates the most expensive infrastructure in the entire course. **Cleanup is essential** to avoid significant charges!

#### 💰 Cost Impact Before Cleanup

**Always-On Charges (24/7):**
- Application Load Balancer: ~$18/month
- NAT Gateway: ~$32/month  
- EBS volumes: ~$2-4/month
- **Base cost: ~$52+/month**

**Variable Charges:**
- EC2 instances: $8.50/month per t3.micro (2-5 instances)
- Data transfer: $5-15/month
- **Total potential: $75-90/month if left running**

#### Step-by-Step Cleanup Process

**1. Preview Complete Destruction**
```bash
# See exactly what will be destroyed (20+ resources)
terraform plan -destroy

# OR if you used production config
terraform plan -destroy -var-file="prod.tfvars"
```

**2. Execute Full Cleanup**
```bash
# Option 1: Destroy everything (default config)
terraform destroy

# Option 2: Destroy with production config
terraform destroy -var-file="prod.tfvars"

# Confirm by typing 'yes' when prompted
```

**3. Monitor Destruction Progress**
```bash
# Destruction takes 5-15 minutes - you'll see:
# - Auto Scaling Group deletion
# - EC2 instances termination  
# - Load balancer deletion
# - VPC component cleanup
# - Security group removal
```

#### Verify Complete Cleanup

```bash
# Check Terraform state is empty
terraform state list
# Should output: (nothing)

# Verify no EC2 instances remain
aws ec2 describe-instances \
  --filters "Name=tag:Project,Values=final-webapp,webapp-prod" \
  --query 'Reservations[*].Instances[?State.Name!=`terminated`].[InstanceId,State.Name,Tags[?Key==`Name`].Value|[0]]' \
  --output table

# Check no load balancers remain
aws elbv2 describe-load-balancers \
  --query 'LoadBalancers[?starts_with(LoadBalancerName, `final-webapp`) || starts_with(LoadBalancerName, `webapp-prod`)].LoadBalancerName' \
  --output table

# Verify no custom VPCs remain (should only see default VPC)
aws ec2 describe-vpcs \
  --filters "Name=tag:Project,Values=final-webapp,webapp-prod" \
  --query 'Vpcs[*].[VpcId,Tags[?Key==`Name`].Value|[0]]' \
  --output table
```

#### What Gets Destroyed

**✅ Complete Infrastructure Removal:**
- 🏗️ **Networking:** Custom VPC, subnets, route tables, NAT gateway, internet gateway
- ⚖️ **Load Balancing:** Application Load Balancer, target groups, health checks
- 💻 **Compute:** All EC2 instances, launch templates, auto scaling groups
- 🔒 **Security:** All custom security groups and rules
- 📊 **Monitoring:** CloudWatch alarms, scaling policies
- 🏷️ **Metadata:** All tags, resource names, configurations

**✅ What Remains (Safe & Free):**
- 📄 Your Terraform configuration files (keep these!)
- 🌐 Default VPC and subnets (AWS-managed, always free)
- 📋 Your learning notes and screenshots
- 🎓 The knowledge you gained building this!

#### Emergency Cleanup (If Terraform Destroy Fails)

Sometimes resources get stuck. Here's manual cleanup:

```bash
# 1. Force delete Auto Scaling Groups
aws autoscaling describe-auto-scaling-groups \
  --query 'AutoScalingGroups[?contains(AutoScalingGroupName, `final-webapp`) || contains(AutoScalingGroupName, `webapp-prod`)].AutoScalingGroupName' \
  --output text | xargs -I {} aws autoscaling delete-auto-scaling-group --auto-scaling-group-name {} --force-delete

# 2. Terminate any remaining instances
aws ec2 describe-instances \
  --filters "Name=tag:Project,Values=final-webapp,webapp-prod" "Name=instance-state-name,Values=running,pending" \
  --query 'Reservations[*].Instances[*].InstanceId' \
  --output text | xargs -I {} aws ec2 terminate-instances --instance-ids {}

# 3. Delete load balancers
aws elbv2 describe-load-balancers \
  --query 'LoadBalancers[?starts_with(LoadBalancerName, `final-webapp`) || starts_with(LoadBalancerName, `webapp-prod`)].LoadBalancerArn' \
  --output text | xargs -I {} aws elbv2 delete-load-balancer --load-balancer-arn {}

# 4. Wait 10 minutes, then delete VPC (last step)
```

#### Post-Cleanup Verification

**Check AWS Billing Dashboard:**
1. Go to AWS Console → Billing & Cost Management
2. Check "Bills" section for current month
3. Verify no new charges after cleanup timestamp
4. Set up billing alerts for future projects

**Set Billing Alerts (Recommended):**
```bash
# Create a simple billing alert for $10
aws budgets create-budget \
  --account-id $(aws sts get-caller-identity --query Account --output text) \
  --budget file://billing-alert.json
```

#### Cleanup Best Practices for Future

**🔄 Regular Cleanup Schedule:**
- Always `terraform destroy` after learning sessions
- Set calendar reminders for weekend resource reviews
- Use tags consistently to track learning resources

**💡 Cost Management Tips:**
- Never leave learning infrastructure running overnight
- Use `terraform plan -destroy` before every break
- Consider using `terraform apply -auto-approve` only for destruction
- Take screenshots of working infrastructure before destroying

**🎯 Final Project Achievement Unlocked:**
You've successfully built and cleaned up production-grade infrastructure! This same pattern scales to support millions of users. The configuration you wrote could handle a real business workload with minimal changes.

**Destruction takes 5-15 minutes to complete all resource cleanup**
  security_groups    = [module.sg_alb.security_group_id]

  target_groups = [
    {
      name_prefix      = "tg-"
      backend_protocol = "HTTP"
      backend_port     = 80
      target_type      = "instance"
      health_check = {
        enabled = true
        path    = "/"
        matcher = "200"
      }
    }
  ]

  listeners = [
    {
      port               = 80
      protocol           = "HTTP"
      target_group_index = 0
    }
  ]

  tags = { Project = var.project_name }
}

# 5) Launch template for app instances (user_data installs a simple web page)
data "aws_ami" "al2" {
  most_recent = true
  owners      = ["amazon"]
  filter { name = "name", values = ["amzn2-ami-hvm-*-x86_64-gp2"] }
}

locals {
  user_data = <<-EOT
              #!/bin/bash
              yum -y update
              yum -y install nginx
              cat > /usr/share/nginx/html/index.html <<EOF
              <h1>${var.project_name} - Hello from $(hostname)</h1>
              EOF
              systemctl enable nginx
              systemctl start nginx
              EOT
}

resource "aws_launch_template" "app" {
  name_prefix   = "${var.project_name}-lt-"
  image_id      = data.aws_ami.al2.id
  instance_type = var.instance_type

  user_data = base64encode(local.user_data)

  vpc_security_group_ids = [module.sg_app.security_group_id]

  tag_specifications {
    resource_type = "instance"
    tags = { Project = var.project_name, Role = "app" }
  }
}

# 6) Auto Scaling Group across private subnets
resource "aws_autoscaling_group" "app" {
  name                      = "${var.project_name}-asg"
  desired_capacity          = 2
  max_size                  = 3
  min_size                  = 2
  vpc_zone_identifier       = module.vpc.private_subnets
  health_check_type         = "EC2"
  force_delete              = true
  target_group_arns         = [module.alb.target_group_arns[0]]

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  tag {
    key                 = "Project"
    value               = var.project_name
    propagate_at_launch = true
  }

  lifecycle {
    create_before_destroy = true
  }
}

# Ensure ASG capacity is created (helpful for first deployment)
resource "aws_autoscaling_policy" "scale_up" {
  name                   = "${var.project_name}-scale-up"
  autoscaling_group_name = aws_autoscaling_group.app.name
  policy_type            = "TargetTrackingScaling"
  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value = 60
  }
}

output "alb_dns" {
  value       = module.alb.lb_dns_name
  description = "ALB DNS name - open in your browser"
}
EOF
```

- `variables.tf`:
```bash
cat > variables.tf << 'EOF'
variable "project_name" {
  type        = string
  description = "Project name prefix"
  default     = "final"
}

variable "region" {
  type        = string
  description = "AWS region"
  default     = "us-east-1"
}

variable "instance_type" {
  type        = string
  description = "EC2 instance type for app servers"
  default     = "t3.micro"
}

variable "azs" {
  type        = list(string)
  description = "Availability Zones"
  default     = ["us-east-1a", "us-east-1b"]
}

variable "public_subnets" {
  type        = list(string)
  description = "Public subnets"
  default     = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "private_subnets" {
  type        = list(string)
  description = "Private subnets"
  default     = ["10.0.101.0/24", "10.0.102.0/24"]
}
EOF
```

- Initialize and apply:
```bash
terraform init
terraform apply -auto-approve
terraform output alb_dns
```

- Test:
```bash
curl -I http://$(terraform output -raw alb_dns)
# Or open the DNS in a browser. You should see the NGINX default page or the custom message.
```

- Clean up:
```bash
terraform destroy -auto-approve
```

## Code Examples

### Architecture Patterns Demonstrated

This project showcases several production-ready patterns:

#### **Multi-Tier Architecture**
```
Internet → Load Balancer → Application Servers → (Future: Database)
```

#### **Infrastructure as Code Best Practices**
- Modular design using community modules
- Comprehensive variable configuration
- Detailed outputs for integration
- Environment-specific configurations
- Proper resource tagging

#### **Security Best Practices**
- Principle of least privilege (security groups)
- Private subnets for application servers
- No direct internet access to app servers
- Encrypted data in transit

#### **High Availability Design**
- Multi-AZ deployment
- Auto scaling for fault tolerance
- Load balancer health checks
- Rolling instance updates

## Hands-On Exercise

### Exercise 1: Scale Testing
Test the auto scaling functionality:

```bash
# 1. Increase desired capacity
terraform apply -var="desired_capacity=4"

# 2. Watch instances launch
watch "aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names \$(terraform output -raw autoscaling_group_name) --query 'AutoScalingGroups[0].Instances[*].[InstanceId,LifecycleState]' --output table"

# 3. Test load balancing across all instances
for i in {1..10}; do
  curl -s $(terraform output -raw load_balancer_url) | grep "Instance ID" | sed 's/.*<strong>//' | sed 's/<\/strong>.*//'
done
```

### Exercise 2: Failure Simulation
Test high availability by simulating instance failure:

```bash
# 1. Get current instances
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names $(terraform output -raw autoscaling_group_name) \
  --query 'AutoScalingGroups[0].Instances[*].InstanceId' \
  --output text

# 2. Terminate one instance (simulate failure)
# Replace i-xxxxxx with an actual instance ID from step 1
aws ec2 terminate-instances --instance-ids i-xxxxxx

# 3. Watch auto scaling launch a replacement
watch "aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names \$(terraform output -raw autoscaling_group_name) --query 'AutoScalingGroups[0].Instances[*].[InstanceId,LifecycleState]' --output table"
```

### Exercise 3: Configuration Changes
Practice infrastructure updates:

```bash
# 1. Change instance type
terraform apply -var="instance_type=t3.small"

# 2. Update the application with a new user data script
# Edit main.tf and change the HTML title to "Updated Web App"

# 3. Apply with instance refresh
terraform apply

# 4. Watch rolling update
watch "aws autoscaling describe-instance-refreshes --auto-scaling-group-name \$(terraform output -raw autoscaling_group_name)"
```

## Troubleshooting Tips

### Common Deployment Issues

#### **Module Download Failures**
```
Error: Failed to download module
```
**Solutions:**
- Check internet connectivity
- Verify Terraform registry access
- Try: `terraform init -upgrade`

#### **ALB Health Check Failures**
```
Targets are unhealthy
```
**Debugging steps:**
1. Wait 5-10 minutes for instances to fully boot
2. Check security groups allow ALB → instances communication
3. Verify user data script runs successfully:
   ```bash
   # SSH to an instance and check
   sudo tail -f /var/log/cloud-init-output.log
   ```

#### **Auto Scaling Issues**
```
Instances launching but not joining target group
```
**Solutions:**
- Check subnet configuration (instances should be in private subnets)
- Verify security groups allow health check traffic
- Check launch template configuration

#### **Cost Concerns**
```
Unexpected AWS charges
```
**Prevention:**
- Set up AWS billing alerts
- Use `terraform plan` to review costs before applying
- Remember to destroy resources when done learning
- Monitor your AWS billing dashboard regularly

### Advanced Troubleshooting Commands

```bash
# Check instance logs
aws logs describe-log-streams --log-group-name /aws/ec2/user-data

# Verify security group rules
aws ec2 describe-security-groups --group-ids $(terraform output -raw app_security_group_id)

# Check load balancer target health
aws elbv2 describe-target-health --target-group-arn $(terraform state show 'module.alb.aws_lb_target_group.main[0]' | grep arn | head -1 | awk '{print $3}')

# Review Auto Scaling activities  
aws autoscaling describe-scaling-activities --auto-scaling-group-name $(terraform output -raw autoscaling_group_name)
```

## Summary / Key Takeaways
- You deployed a scalable, load-balanced app infrastructure using Terraform and modules.
- You validated it via the ALB DNS.

## Next Steps
- Extend with RDS, CloudWatch alarms, and CI/CD pipelines.
- Explore Terraform Cloud for team workflows, runs, and policies.

## Course Completion 🎉

Congratulations! You've completed the "Learn Terraform from Scratch on Ubuntu" course. You now have the skills to:

- Set up Terraform environments
- Write infrastructure as code
- Manage state and variables
- Use modules for reusability
- Implement best practices
- Deploy production-ready infrastructure

Continue your learning journey by exploring advanced Terraform features, integrating with CI/CD pipelines, and building more complex architectures!