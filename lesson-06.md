# Lesson 6: Variables and Outputs

## Learning Objectives
- Define variables with types, defaults, and descriptions.
- Pass variables via CLI, environment variables, and tfvars files.
- Create and consume outputs.

## Concept Overview
Variables make your config reusable across environments. Outputs share results (like IPs) with you or other tools.

## Step-by-Step Tutorial
- In a new folder:
```bash
mkdir -p ~/terraform-course/lesson-6
cd ~/terraform-course/lesson-6
```

- Create `variables.tf`:
```bash
cat > variables.tf << 'EOF'
variable "region" {
  type        = string
  description = "AWS region"
  default     = "us-east-1"
}

variable "instance_type" {
  type        = string
  description = "EC2 instance type"
  default     = "t2.micro"
}

variable "tags" {
  type        = map(string)
  description = "Common tags"
  default     = { Project = "Lesson-6", Owner = "student" }
}
EOF
```

- Create `main.tf`:
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
}

resource "aws_instance" "example" {
  ami           = "ami-0c02fb55956c7d316"  # Replace if region differs
  instance_type = var.instance_type
  tags          = merge(var.tags, { Name = "lesson-6" })
}
EOF
```

- Create `outputs.tf`:
```bash
cat > outputs.tf << 'EOF'
output "instance_type_used" {
  value       = aws_instance.example.instance_type
  description = "What instance type was used"
}

output "tags_merged" {
  value       = aws_instance.example.tags
  description = "Tags applied to the instance"
}
EOF
```

- Use variables via different methods:
```bash
terraform init
terraform plan -var="instance_type=t3.micro"
echo 'instance_type="t3.small"' > dev.tfvars
terraform plan -var-file="dev.tfvars"

# Using environment variable (takes precedence over default):
export TF_VAR_instance_type="t3.nano"
terraform apply -auto-approve

# Show outputs:
terraform output
```

## Code Examples
Included above.

## Hands-On Exercise
- Create `prod.tfvars` with `instance_type="t3.micro"` and different tags. Run `terraform plan -var-file="prod.tfvars"` and observe changes.

## Troubleshooting Tips
- If `TF_VAR_*` not picked up: Ensure you're exporting in the same shell session or add to `~/.bashrc`.
- If AMI not valid: Switch regions or use Lesson 8 technique.

## Summary / Key Takeaways
- Variables and outputs make Terraform flexible and integrate with other tools.
- tfvars files help separate environment-specific settings.

## Next Steps
Understand Terraform state—what it is and how to manage it.