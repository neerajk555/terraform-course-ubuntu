# Lesson 4: Writing Your First Terraform Configuration (Provisioning an EC2 Instance)

## Learning Objectives
- Create an EC2 instance with Terraform.
- Understand resources, arguments, and outputs.
- Generate and use an SSH key pair.

## Concept Overview

### What We're Building Today 🏗️

Think of this lesson like ordering a custom computer online. You'll specify:
- **What type** of computer (EC2 instance type)
- **Where to ship it** (AWS region)  
- **Security settings** (who can connect)
- **Access keys** (how YOU can connect)

### Key Concepts You'll Learn

**🖥️ EC2 Instance:** Amazon's virtual computer
- Like renting a computer in Amazon's data center
- You choose the size (CPU, RAM, storage)
- Charged by the hour while it's running

**🔑 SSH Key Pair:** Your digital key to access the instance
- **Public Key:** Goes to AWS (like giving someone your address)
- **Private Key:** Stays with you (like keeping your house key)
- Enables secure remote login

**🛡️ Security Group:** Virtual firewall rules
- Controls what traffic can reach your instance
- Like a bouncer at a club - decides who gets in

**📊 Terraform Resources:** The "things" you want to create
- Each resource has a **type** (aws_instance, aws_security_group)
- Each resource has **arguments** (configuration settings)
- Each resource gets a **name** (for Terraform to track it)

### The Security Setup We'll Create

```
Internet → Security Group → EC2 Instance
    ↓            ↓              ↓
  Port 22    Allow SSH     Your Ubuntu Server
  (SSH)      from anywhere   (with your SSH key)
```

**⚠️ Important Security Note:** 
We'll open SSH to the entire internet (0.0.0.0/0) for simplicity. In production, you'd restrict this to your IP address only.

## Step-by-Step Tutorial

### Step 1: Set Up Your Project Directory

```bash
# Navigate to your course folder
cd ~/terraform-course

# Create a new directory for this lesson
mkdir lesson-4
cd lesson-4

# Verify you're in the right place
pwd
# Should show: /home/your-username/terraform-course/lesson-4
```

### Step 2: Generate SSH Keys for Secure Access

SSH (Secure Shell) keys are your way to securely connect to Linux servers. We need to create a key pair before creating the instance.

```bash
# Generate a new SSH key pair specifically for this course
ssh-keygen -t ed25519 -C "terraform-course" -f ~/.ssh/terraform_course -N ""
```

**Let's break down this command:**
- `ssh-keygen`: The tool that creates SSH keys
- `-t ed25519`: Type of encryption (modern and secure)
- `-C "terraform-course"`: A comment to help identify this key
- `-f ~/.ssh/terraform_course`: Where to save the key files
- `-N ""`: No passphrase (empty string for simplicity)

**This creates two files:**
- `~/.ssh/terraform_course` → **Private key** (keep secret!)
- `~/.ssh/terraform_course.pub` → **Public key** (safe to share)

**Verify your keys were created:**
```bash
# List your SSH keys
ls -la ~/.ssh/terraform_course*

# View your public key (we'll upload this to AWS)
cat ~/.ssh/terraform_course.pub
```

**Expected output:** A long string starting with `ssh-ed25519 AAAA...`

### Step 3: Create Variables File

Variables make your Terraform code flexible and reusable. Instead of hardcoding values, you define them once and reference them throughout your configuration.

```bash
cat > variables.tf << 'EOF'
variable "region" {
  type        = string
  default     = "us-east-1"          # Change if you prefer another region
  description = "AWS region to deploy resources"
}

variable "instance_name" {
  type        = string
  default     = "tf-lesson-4"
  description = "Name tag for the EC2 instance"
}

variable "instance_type" {
  type        = string
  default     = "t2.micro"           # Free-tier eligible
  description = "Instance type"
}
EOF
```

**Understanding Variables:**
```hcl
variable "region" {                    # Variable name (used as var.region)
  type        = string                 # Data type (string, number, bool, list, map)
  default     = "us-east-1"           # Default value if not specified
  description = "AWS region to..."     # Human-readable explanation
}
```

**Why use variables?**
- **Flexibility:** Change values without editing main code
- **Reusability:** Use same code for dev/staging/prod
- **Documentation:** Descriptions explain what each value does
- **Validation:** Type checking prevents mistakes

### Step 4: Create Outputs File

Outputs are how Terraform shares information with you (or other tools) after creating resources.

```bash
cat > outputs.tf << 'EOF'
output "instance_id" {
  value       = aws_instance.web.id
  description = "The ID of the EC2 instance"
}

output "public_ip" {
  value       = aws_instance.web.public_ip
  description = "Public IP of the EC2 instance"
}
EOF
```

**Understanding Outputs:**
```hcl
output "public_ip" {                   # Output name
  value       = aws_instance.web.public_ip  # What to display
  description = "Public IP of..."      # Explanation for users
}
```

**Real-world output examples:**
- **Instance IPs:** For SSH connection or DNS setup
- **Database endpoints:** For application configuration
- **Load balancer URLs:** For testing your application
- **Resource IDs:** For referencing in other Terraform projects

### Step 5: Create the Main Configuration

Now for the main event! This file defines all the infrastructure we want to create.

```bash
cat > main.tf << 'EOF'
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"   # AWS provider
      version = "~> 5.0"          # Provider version
    }
  }
}

provider "aws" {
  region = var.region              # Use variable for region
}

# 1) Register the local SSH public key as an AWS Key Pair
resource "aws_key_pair" "this" {
  key_name   = "terraform-course-key"                               # Name of the key pair in AWS
  public_key = file("~/.ssh/terraform_course.pub")                  # Use the public key we generated
}

# 2) A simple security group to allow SSH
resource "aws_security_group" "ssh" {
  name        = "allow-ssh-lesson-4"                                # SG name
  description = "Allow SSH from anywhere (demo only)"
  vpc_id      = data.aws_vpc.default.id                             # Attach to default VPC

  ingress {
    description = "SSH"
    from_port   = 22                                                # SSH port
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]                                     # Open to world (for demo)
  }

  egress {
    from_port   = 0                                                 # All outbound
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "allow-ssh-lesson-4" }
}

# 3) Data source to fetch the default VPC
data "aws_vpc" "default" {
  default = true                                                    # The default VPC in the region
}

# 4) Data source to get a default subnet (first one)
data "aws_subnet_ids" "default" {
  vpc_id = data.aws_vpc.default.id                                  # Use default VPC
}

# 5) EC2 instance
resource "aws_instance" "web" {
  ami                         = "ami-0c02fb55956c7d316"             # Example Amazon Linux 2 AMI in us-east-1
  instance_type               = var.instance_type                   # t2.micro free-tier eligible
  subnet_id                   = element(data.aws_subnet_ids.default.ids, 0)
  vpc_security_group_ids      = [aws_security_group.ssh.id]
  key_name                    = aws_key_pair.this.key_name

  tags = {
    Name = var.instance_name                                        # Name tag
  }
}
EOF
```

**Let's understand each resource in detail:**

#### 🔑 AWS Key Pair Resource
```hcl
resource "aws_key_pair" "this" {
  key_name   = "terraform-course-key"              # What AWS calls this key
  public_key = file("~/.ssh/terraform_course.pub") # Reads your public key file
}
```
- **Purpose:** Uploads your SSH public key to AWS
- **file() function:** Reads the content of a file
- **Result:** AWS stores your public key; you can use it for EC2 instances

#### 🛡️ Security Group Resource
```hcl
resource "aws_security_group" "ssh" {
  name        = "allow-ssh-lesson-4"               # Security group name
  description = "Allow SSH from anywhere (demo only)"
  vpc_id      = data.aws_vpc.default.id           # Which network to attach to
  
  ingress {                                       # Incoming traffic rules
    description = "SSH"
    from_port   = 22                              # SSH uses port 22
    to_port     = 22                              # Same port (range of 1)
    protocol    = "tcp"                           # TCP protocol
    cidr_blocks = ["0.0.0.0/0"]                  # Allow from anywhere
  }
  
  egress {                                        # Outgoing traffic rules
    from_port   = 0                               # All ports
    to_port     = 0                               # All ports  
    protocol    = "-1"                            # All protocols
    cidr_blocks = ["0.0.0.0/0"]                  # To anywhere
  }
}
```

**Security Groups Explained:**
- **Ingress:** Traffic coming IN to your server
- **Egress:** Traffic going OUT from your server
- **CIDR 0.0.0.0/0:** Means "the entire internet" (use carefully!)

#### 📡 Data Sources (Read-Only Lookups)
```hcl
data "aws_vpc" "default" {
  default = true                                  # Find the default VPC
}

data "aws_subnet_ids" "default" {
  vpc_id = data.aws_vpc.default.id               # Find subnets in that VPC
}
```
- **Data sources:** Query existing AWS resources (don't create new ones)
- **Default VPC:** Every AWS account gets a free default network
- **Subnets:** Subdivisions of the VPC where instances live

#### 💻 EC2 Instance Resource
```hcl
resource "aws_instance" "web" {
  ami                         = "ami-0c02fb55956c7d316"       # OS image
  instance_type               = var.instance_type             # Size (t2.micro)
  subnet_id                   = element(data.aws_subnet_ids.default.ids, 0)  # Which subnet
  vpc_security_group_ids      = [aws_security_group.ssh.id]  # Security rules
  key_name                    = aws_key_pair.this.key_name   # SSH key to use
  
  tags = { Name = var.instance_name }                         # Name tag
}
```

**Key points:**
- **AMI:** Amazon Machine Image (pre-built OS template)
- **element(list, index):** Gets the first subnet from the list
- **Resource references:** `aws_key_pair.this.key_name` refers to the key pair above
EOF
```

### Step 6: Deploy Your Infrastructure

Now let's bring your infrastructure to life!

#### Initialize the Project
```bash
terraform init
```
**What happens:** Downloads the AWS provider and sets up the project.

#### Preview the Changes
```bash
terraform plan
```
**What to expect:**
- Shows all resources that will be created
- Displays a summary like "Plan: 4 to add, 0 to change, 0 to destroy"
- **Important:** This is read-only - nothing gets created yet!

#### Apply the Configuration
```bash
# Apply changes (will prompt for confirmation)
terraform apply

# OR apply automatically without prompts (use carefully!)
terraform apply -auto-approve
```

**During apply, you'll see:**
```
aws_key_pair.this: Creating...
aws_vpc.default: Reading...
aws_security_group.ssh: Creating...
aws_instance.web: Creating...
...
Apply complete! Resources: 4 added, 0 changed, 0 destroyed.
```

#### View Your Outputs
```bash
terraform output
```
**Expected output:**
```
instance_id = "i-0123456789abcdef0"
public_ip = "54.123.45.67"
```

### Step 7: Connect to Your Instance

Wait 1-2 minutes for the instance to boot up, then connect via SSH:

```bash
# Connect using your private key
ssh -i ~/.ssh/terraform_course ec2-user@$(terraform output -raw public_ip)
```

**If successful, you'll see:**
```
The authenticity of host '54.123.45.67 (54.123.45.67)' can't be established.
...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
```
Type `yes` and press Enter.

**You should now be connected to your EC2 instance!**
```
[ec2-user@ip-172-31-x-x ~]$ 
```

**Try some commands on your new server:**
```bash
# Check the OS version
cat /etc/os-release

# Check available disk space
df -h

# Check system info
uname -a

# Exit back to your local machine
exit
```

### Step 8: Clean Up Resources (Optional)

⚠️ **Important:** AWS charges for running instances. Clean up when done practicing!

```bash
# Destroy all resources created by Terraform
terraform destroy
```
Type `yes` when prompted.

**What gets destroyed:**
- Your EC2 instance (and all data on it)
- The security group
- The SSH key pair (from AWS, not your local files)

**What remains:**
- Your local SSH keys
- Your Terraform configuration files
- The default VPC and subnets (these are free)

## Code Examples
Included above with inline comments.

Note: The AMI is region-specific. If you change regions, update the AMI or see Lesson 8 to dynamically find AMIs.

## Hands-On Exercise

### Exercise 1: Modify and Re-apply
Practice the Terraform workflow by making a simple change:

```bash
# Edit variables.tf to change the instance name
sed -i 's/tf-lesson-4/my-first-server/g' variables.tf

# Apply the change
terraform apply
```

**Question:** What does Terraform show in the plan? Why doesn't it destroy and recreate the instance?

### Exercise 2: Explore Your Instance
SSH into your instance and explore:

```bash
# Connect to your instance
ssh -i ~/.ssh/terraform_course ec2-user@$(terraform output -raw public_ip)

# Once connected, try these commands:
whoami                  # Current user
pwd                     # Current directory
ls -la                  # List files
free -h                 # Memory usage
lscpu                   # CPU information
curl ifconfig.me        # Check public IP from inside the instance
```

### Exercise 3: Check AWS Console
1. Log into your AWS Management Console
2. Navigate to EC2 → Instances
3. Find your instance and verify:
   - Name matches your variable
   - Instance type is t2.micro
   - Security group has SSH rule

## Troubleshooting Tips

### Common Issues and Solutions

#### 1. **AMI ID Invalid Error**
```
Error: InvalidAMIID.NotFound: The image id '[ami-xxx]' does not exist
```
**Cause:** AMI IDs are region-specific. The hardcoded AMI might not exist in your region.

**Solutions:**
- Change your region to `us-east-1` in `variables.tf`
- OR find the correct AMI ID for your region:
  ```bash
  aws ec2 describe-images \
    --owners amazon \
    --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
    --query 'Images[*].[ImageId,Name]' \
    --output table
  ```

#### 2. **SSH Connection Fails**
```
ssh: connect to host 1.2.3.4 port 22: Connection timed out
```

**Troubleshooting checklist:**
1. **Wait longer:** Instance needs 1-2 minutes to boot
2. **Check instance state:** Should be "running" in AWS console
3. **Verify security group:** Should allow port 22 from 0.0.0.0/0
4. **Test with verbose SSH:**
   ```bash
   ssh -v -i ~/.ssh/terraform_course ec2-user@$(terraform output -raw public_ip)
   ```

#### 3. **Permission Denied (SSH Key)**
```
Permission denied (publickey)
```
**Solutions:**
- Check key permissions: `chmod 600 ~/.ssh/terraform_course`
- Verify key path in SSH command
- Ensure you're using `ec2-user` (not `root` or `ubuntu`)

#### 4. **Terraform Apply Fails**
```
Error: error creating EC2 Instance: UnauthorizedOperation
```
**Cause:** AWS credentials don't have sufficient permissions.

**Solution:** Ensure your IAM user has EC2 permissions:
```bash
# Test your permissions
aws sts get-caller-identity
aws ec2 describe-regions
```

#### 5. **File Not Found Error**
```
Error: contents of "~/.ssh/terraform_course.pub": no such file or directory
```
**Solution:** Regenerate your SSH key:
```bash
ssh-keygen -t ed25519 -C "terraform-course" -f ~/.ssh/terraform_course -N ""
```

## Summary / Key Takeaways
- You created your first EC2 instance with Terraform.
- You learned about resources, data sources, and outputs.

## Next Steps
Master the core Terraform commands and what they do.