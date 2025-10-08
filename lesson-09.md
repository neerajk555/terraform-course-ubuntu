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

## Troubleshooting Tips
- If the module path is wrong: Ensure `source = "./modules/ec2-basic"` matches your folder structure.
- If AMI not found: Region/filter mismatch—see Lesson 8.

## Summary / Key Takeaways
- Modules make Terraform code DRY and reusable.
- You created and consumed your own module.

## Next Steps
Use provisioners to bootstrap instances remotely.