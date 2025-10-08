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

## Clean Up Resources ⚠️ CRITICAL - Multiple Environments

This lesson creates resources in MULTIPLE workspaces, which means you could have double the charges if not properly cleaned up!

### Understanding Multi-Environment Cleanup

**⚠️ Why This Matters More:**
- You created instances in both `dev` AND `prod` workspaces
- Each workspace = separate infrastructure = separate costs
- Must clean up EACH workspace individually
- Forgetting one workspace = paying for unused resources

### Step-by-Step Multi-Workspace Cleanup

#### 1. List All Workspaces
```bash
# See all environments you created
terraform workspace list
# * default
#   dev
#   prod
```

#### 2. Clean Up Each Environment

**Clean up DEV environment:**
```bash
# Switch to dev workspace
terraform workspace select dev

# Preview what will be destroyed in dev
terraform plan -destroy -var-file=dev.tfvars

# Destroy dev resources
terraform destroy -var-file=dev.tfvars -auto-approve

# Verify dev is clean
terraform state list
# Should be empty
```

**Clean up PROD environment:**
```bash
# Switch to prod workspace
terraform workspace select prod

# Preview what will be destroyed in prod
terraform plan -destroy -var-file=prod.tfvars

# Destroy prod resources
terraform destroy -var-file=prod.tfvars -auto-approve

# Verify prod is clean
terraform state list
# Should be empty
```

**Clean up DEFAULT workspace (if used):**
```bash
# Switch to default workspace
terraform workspace select default

# Check if anything exists
terraform state list

# If resources exist, destroy them
terraform destroy -auto-approve
```

#### 3. Delete Empty Workspaces (Optional)

```bash
# Switch to default first
terraform workspace select default

# Delete empty workspaces
terraform workspace delete dev
terraform workspace delete prod

# Verify only default remains
terraform workspace list
# * default
```

### Verification: Ensure Complete Cleanup

```bash
# Check NO EC2 instances remain with lesson-11 tags
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=app-dev,app-prod" \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name,Tags[?Key==`Name`].Value|[0]]' \
  --output table
# Should show no running/pending instances

# List all currently running instances (to be sure)
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running,pending" \
  --query 'Reservations[*].Instances[*].[InstanceId,Tags[?Key==`Name`].Value|[0],InstanceType]' \
  --output table
```

### Common Multi-Environment Mistakes

**❌ Mistake 1: Only cleaning up one workspace**
```bash
# WRONG - only cleans current workspace
terraform destroy -auto-approve
```

**✅ Correct: Clean each workspace**
```bash
# RIGHT - clean each environment individually
terraform workspace select dev && terraform destroy -var-file=dev.tfvars -auto-approve
terraform workspace select prod && terraform destroy -var-file=prod.tfvars -auto-approve
```

**❌ Mistake 2: Forgetting which workspace you're in**
```bash
terraform workspace show  # Always check first!
```

**❌ Mistake 3: Using wrong tfvars file**
```bash
# WRONG - using prod vars in dev workspace
terraform workspace select dev
terraform destroy -var-file=prod.tfvars  # This might not work or cause confusion
```

### Cost Impact Analysis

**💰 Before Cleanup (Potential Monthly Cost):**
- Dev instance: ~$8.50/month (t2.micro)
- Prod instance: ~$8.50/month (t2.micro)
- **Total: ~$17/month** for just these basic instances

**✅ After Cleanup:**
- All instances terminated
- All charges stopped
- Only pay for actual usage time

### Emergency: "I Lost Track of My Workspaces!"

```bash
# 1. List all workspaces
terraform workspace list

# 2. Check each workspace for resources
for workspace in $(terraform workspace list | grep -v "^*" | sed 's/^  //'); do
    echo "=== Checking workspace: $workspace ==="
    terraform workspace select $workspace
    terraform state list
done

# 3. Clean up any workspace with resources
# terraform workspace select WORKSPACE_NAME
# terraform destroy -auto-approve

# 4. Return to default
terraform workspace select default
```

**💡 Pro Tip for Future Lessons:** Always use descriptive workspace names and clean up immediately after learning. Consider setting calendar reminders to review and clean up AWS resources weekly.

## Troubleshooting Tips
- If you forget to switch workspace: You might deploy to the wrong environment—always check `terraform workspace show`.
- If you prefer stricter separation: Use directory layout with separate backends (see Lesson 12).

## Summary / Key Takeaways
- Workspaces or directories help isolate environments.
- tfvars files drive per-environment customization.

## Next Steps
Store state remotely in S3 (with DynamoDB locking) or use Terraform Cloud.