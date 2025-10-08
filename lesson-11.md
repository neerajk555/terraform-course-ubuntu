# Lesson 11: Managing Multiple Environments (Dev, Staging, Prod)

## Learning Objectives
- Use workspaces and/or directory patterns to separate environments.
- Use tfvars for environment-specific settings.
- Avoid accidental cross-environment changes.

## Concept Overview
Two common patterns:
- Workspaces: One codebase, multiple states via `terraform workspace`.
- Directory layout: Separate folders (dev/, staging/, prod/) each with its own state.

## Step-by-Step Tutorial
Workspace approach:
```bash
mkdir -p ~/terraform-course/lesson-11
cd ~/terraform-course/lesson-11

cat > main.tf << 'EOF'
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}

provider "aws" { region = var.region }

variable "region" { type = string, default = "us-east-1" }
variable "instance_name" { type = string }

resource "aws_instance" "app" {
  ami           = "ami-0c02fb55956c7d316"
  instance_type = "t2.micro"
  tags = { Name = var.instance_name }
}
EOF

cat > dev.tfvars << 'EOF'
region = "us-east-1"
instance_name = "app-dev"
EOF

cat > prod.tfvars << 'EOF'
region = "us-east-1"
instance_name = "app-prod"
EOF

terraform init
terraform workspace new dev
terraform workspace select dev
terraform apply -var-file=dev.tfvars -auto-approve

terraform workspace new prod
terraform workspace select prod
terraform apply -var-file=prod.tfvars -auto-approve
```

Why:
- Each workspace has a separate state; the same code can manage multiple environments.

Alternative directory layout:
```
environments/
  dev/
    main.tf
    dev.tfvars
  prod/
    main.tf
    prod.tfvars
```
Each folder runs its own `terraform init` and maintains separate state by default.

## Code Examples
Included above.

## Hands-On Exercise
- List workspaces: `terraform workspace list`.
- Destroy only dev: `terraform workspace select dev && terraform destroy -var-file=dev.tfvars -auto-approve`.

## Troubleshooting Tips
- If you forget to switch workspace: You might deploy to the wrong environment—always check `terraform workspace show`.
- If you prefer stricter separation: Use directory layout with separate backends (see Lesson 12).

## Summary / Key Takeaways
- Workspaces or directories help isolate environments.
- tfvars files drive per-environment customization.

## Next Steps
Store state remotely in S3 (with DynamoDB locking) or use Terraform Cloud.