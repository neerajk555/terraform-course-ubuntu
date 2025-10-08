# Lesson 12: Storing State Remotely with AWS S3 or Terraform Cloud

## Learning Objectives
- Configure remote state in S3 with DynamoDB state locking.
- Migrate local state to remote.
- Understand Terraform Cloud as an alternative.

## Concept Overview
Remote state:
- Centralizes state for teams.
- Adds locking to prevent conflicts.

S3 + DynamoDB is a common, reliable setup on AWS.

## Step-by-Step Tutorial
- Create S3 bucket and DynamoDB table (use a bootstrap step; don't use Terraform for the backend itself initially):
```bash
AWS_REGION=us-east-1
BUCKET_NAME="tf-state-$(date +%s)"
LOCK_TABLE="tf-state-locks"

aws s3api create-bucket --bucket "$BUCKET_NAME" --region "$AWS_REGION" --create-bucket-configuration LocationConstraint="$AWS_REGION"
aws s3api put-bucket-versioning --bucket "$BUCKET_NAME" --versioning-configuration Status=Enabled
aws dynamodb create-table --table-name "$LOCK_TABLE" --attribute-definitions AttributeName=LockID,AttributeType=S --key-schema AttributeName=LockID,KeyType=HASH --billing-mode PAYPERREQUEST --region "$AWS_REGION"
```

Note: For us-east-1 region, omit the `--create-bucket-configuration` parameter:
```bash
aws s3api create-bucket --bucket "$BUCKET_NAME" --region us-east-1
```

- Update your Terraform (e.g., in `lesson-11`):
```bash
cat > backend.tf << 'EOF'
terraform {
  backend "s3" {
    bucket         = "REPLACE_WITH_YOUR_BUCKET"
    key            = "lesson-11/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "REPLACE_WITH_YOUR_LOCK_TABLE"
    encrypt        = true
  }
}
EOF
```
Replace placeholders with your names.

- Migrate state:
```bash
terraform init -migrate-state
```
Why:
- Moves your existing local state to S3 with locking enabled.

Terraform Cloud alternative:
- Create an organization and workspace in Terraform Cloud UI.
- Update your `terraform` block:
```hcl
terraform {
  cloud {
    organization = "your-org-name"
    workspaces {
      name = "lesson-11"
    }
  }
}
```
- Run `terraform init` and follow prompts to connect.

## Code Examples
Included above.

## Hands-On Exercise
- Configure S3 backend for your `lesson-9` or `lesson-11` project and verify state is stored in the bucket.

## Troubleshooting Tips
- S3 create-bucket errors in us-east-1: Omit the `--create-bucket-configuration` for that region.
- AccessDenied: Ensure your IAM user can access S3 and DynamoDB.
- Lock stuck: Check DynamoDB for stale locks; clear carefully if no operations are running.

## Summary / Key Takeaways
- Remote state enables collaboration and safety with locking.
- S3 + DynamoDB is a common choice; Terraform Cloud is a managed alternative.

## Next Steps
Integrate Terraform with Ansible for configuration management.