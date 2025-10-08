# Lesson 14: Best Practices, Directory Structure, and Version Control

## Learning Objectives
- Organize Terraform code effectively.
- Adopt version control and security best practices.
- Use linters and policy tools.

## Concept Overview
Good habits scale your IaC:
- Structure: separate modules and environments.
- Version control: Git for history and collaboration.
- Security: Never commit secrets or state.

Recommended structure:
```
project/
  modules/
    <module-a>/
    <module-b>/
  envs/
    dev/
      main.tf
      variables.tf
      dev.tfvars
    prod/
      main.tf
      variables.tf
      prod.tfvars
  README.md
```

## Step-by-Step Tutorial
- Initialize Git and .gitignore:
```bash
cd ~/terraform-course
git init
cat > .gitignore << 'EOF'
.terraform/
.terraform.lock.hcl
terraform.tfstate
terraform.tfstate.backup
*.tfvars
*.tfvars.json
crash.log
override.tf
override.tf.json
*.override.tf
*.override.tf.json
EOF
git add .
git commit -m "Initial Terraform course"
```
Why:
- Prevents committing state and secrets.

- Install helpful tools:
```bash
# tflint
curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash
tflint --version

# tfsec (security checks)
curl -s https://raw.githubusercontent.com/aquasecurity/tfsec/master/scripts/install_linux.sh | bash
tfsec --version
```

- Run checks:
```bash
tflint
tfsec
terraform fmt -recursive
```

## Code Examples
Not code-specific; structural guidance.

## Hands-On Exercise
- Apply `.gitignore` to one of your lesson folders and run `tflint` and `tfsec`. Fix any warnings you understand.

## Troubleshooting Tips
- If `tflint`/`tfsec` not found: Ensure install scripts ran and the binaries are on PATH (reopen your shell if needed).

## Summary / Key Takeaways
- Proper structure, version control, and linting improve quality and security.
- Avoid committing sensitive files.

## Next Steps
Capstone: deploy a scalable web app infrastructure on AWS.