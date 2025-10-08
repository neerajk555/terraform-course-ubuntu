# Lesson 9: Terraform Modules (Reusable Code)

## Learning Objectives
- Understand what modules are and why to use them.
- Create a simple custom module.
- Use a module from the Terraform Registry.

## Concept Overview
Modules are folders of Terraform code that you can reuse. You pass inputs and get outputs back—like functions.

## Step-by-Step Tutorial
- Create a root folder and a module:
```bash
mkdir -p ~/terraform-course/lesson-9/modules/ec2-basic
cd ~/terraform-course/lesson-9
```

- Module code `modules/ec2-basic/main.tf`:
```bash
cat > modules/ec2-basic/main.tf << 'EOF'
variable "name" { type = string }
variable "instance_type" { type = string }
variable "subnet_id" { type = string }
variable "tags" { type = map(string) default = {} }

data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_instance" "this" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = var.instance_type
  subnet_id     = var.subnet_id
  tags          = merge(var.tags, { Name = var.name })
}

output "instance_id" {
  value = aws_instance.this.id
}

output "public_ip" {
  value = aws_instance.this.public_ip
}
EOF
```

- Root `main.tf` to call the module:
```bash
cat > main.tf << 'EOF'
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}

provider "aws" {
  region = "us-east-1"
}

data "aws_vpc" "default" { default = true }
data "aws_subnet_ids" "default" { vpc_id = data.aws_vpc.default.id }

module "web" {
  source        = "./modules/ec2-basic"
  name          = "lesson-9-web"
  instance_type = "t2.micro"
  subnet_id     = element(data.aws_subnet_ids.default.ids, 0)
  tags          = { Project = "Lesson-9" }
}

output "web_ip" { value = module.web.public_ip }
EOF
```

- Run:
```bash
terraform init
terraform apply -auto-approve
terraform output web_ip
```

- Example from Registry (reference only, don't apply yet):
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "demo-vpc"
  cidr = "10.0.0.0/16"
  azs  = ["us-east-1a", "us-east-1b"]
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets = ["10.0.101.0/24","10.0.102.0/24"]
}
```

## Code Examples
Included above.

## Hands-On Exercise
- Add another module call to create a second instance in a different subnet (element index 1).

## Clean Up Resources ⚠️ IMPORTANT

This lesson creates EC2 instances using modules, so cleanup is essential to avoid charges.

### Standard Cleanup Process

```bash
# Preview all resources that will be destroyed
terraform plan -destroy

# Destroy all resources (both root and module resources)
terraform destroy -auto-approve

# Verify complete cleanup
terraform state list
# Should be empty

# Verify no EC2 instances remain
aws ec2 describe-instances \
  --filters "Name=tag:Project,Values=Lesson-9" \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name,Tags[?Key==`Name`].Value|[0]]' \
  --output table
```

### Understanding Module Resource Cleanup

**📦 What Modules Don't Change About Cleanup:**
- `terraform destroy` works the same way
- All resources created by modules get destroyed
- State tracking includes both root and module resources

**🔍 Module Resources in State:**
```bash
# Before destroy, see how modules appear in state
terraform state list
# You'll see resources like:
# module.web.aws_instance.this
# module.web.data.aws_ami.amazon_linux_2
```

### If You Did the Hands-On Exercise (Multiple Instances)

```bash
# If you added a second module call, both instances get destroyed
terraform destroy -auto-approve

# Verify both instances are gone
aws ec2 describe-instances \
  --filters "Name=tag:Project,Values=Lesson-9" \
  --query 'Reservations[*].Instances[?State.Name!=`terminated`].[InstanceId,State.Name]' \
  --output table
# Should show no non-terminated instances
```

### Cost Impact

**💰 Resources That Cost Money:**
- EC2 instances (~$8.50/month each if left running)
- EBS storage attached to instances
- Any additional instances from the exercise

**✅ Free Resources:**
- Module definitions (they're just code)
- Data source queries
- Default VPC and subnets

**💡 Module Cleanup Tip:** Modules don't change how resources are destroyed - Terraform treats module resources like any other resources in the dependency graph.

## Troubleshooting Tips
- If the module path is wrong: Ensure `source = "./modules/ec2-basic"` matches your folder structure.
- If AMI not found: Region/filter mismatch—see Lesson 8.

## Summary / Key Takeaways
- Modules make Terraform code DRY and reusable.
- You created and consumed your own module.

## Next Steps
Use provisioners to bootstrap instances remotely.