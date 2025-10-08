# Lesson 7: State Files Explained (terraform.tfstate)

## Learning Objectives
- Understand what Terraform state is and why it matters.
- Learn basic state commands.
- Avoid common state pitfalls.

## Concept Overview
State is how Terraform knows what exists in the real world. The file `terraform.tfstate` tracks resource IDs, attributes, and dependencies.

Key rules:
- Don't edit state manually.
- Don't commit state to Git (contains sensitive info).
- Use remote state for teams (see Lesson 12).

## Step-by-Step Tutorial
- Create a small resource (or reuse Lesson 6), then inspect state:
```bash
terraform state list
terraform show
terraform show -json | jq '.values.root_module.resources[] | .address'
```
Why:
- `state list` shows tracked resources.
- `show` displays details.
- JSON helps integrate with scripts.

- Move/rename resource references (advanced):
```bash
# Example (do not run unless you know why):
# terraform state mv aws_instance.example aws_instance.example_renamed
```

## Code Examples
No new code. Explore your existing state.

## Hands-On Exercise
- Run `terraform show` and scan for resource attributes.
- Add a new tag to your instance and observe how state changes after apply.

## Troubleshooting Tips
- If state is locked (S3 backends): You may see a DynamoDB lock—wait or release carefully.
- If state gets corrupted: Restore from backup (Terraform makes backups like terraform.tfstate.backup).

## Summary / Key Takeaways
- State is critical metadata—treat it carefully.
- Don't commit state; consider remote storage for collaboration.

## Next Steps
Use data sources to discover information dynamically.