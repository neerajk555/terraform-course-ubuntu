# Lesson 5: Terraform Commands: init, plan, apply, destroy

## Learning Objectives
- Learn what each core command does.
- Understand when and why to use each flag.
- Practice reading Terraform plans.

## Concept Overview

Mastering Terraform commands is like learning to drive - you need to understand what each control does before hitting the road. Let's explore the core commands that make up Terraform's workflow.

### The Core Four Commands 🚦

#### 1. **`terraform init` - Initialize Your Project**
**What it does:** Prepares your working directory for Terraform operations
**Like:** Setting up your workspace before starting a project

**When init runs, it:**
- Downloads required providers (AWS, Azure, etc.)
- Sets up the backend (where state is stored)
- Creates `.terraform/` directory with cached files
- Generates `.terraform.lock.hcl` to lock provider versions

**When to run:**
- ✅ First time in a new project
- ✅ After adding new providers
- ✅ After changing backend configuration
- ✅ When someone else modified provider requirements

#### 2. **`terraform plan` - Preview Changes**
**What it does:** Shows you exactly what Terraform will do, without doing it
**Like:** Getting a quote from a contractor before they start work

**Plan output symbols:**
- `+` = Create new resource
- `-` = Destroy resource  
- `~` = Modify existing resource
- `-/+` = Destroy and recreate

**When to run:**
- ✅ ALWAYS before `terraform apply`
- ✅ To understand what changed
- ✅ Before code reviews
- ✅ To troubleshoot issues

#### 3. **`terraform apply` - Execute Changes**
**What it does:** Creates, modifies, or destroys infrastructure according to your configuration
**Like:** Giving the contractor approval to start construction

**Safety features:**
- Shows plan first (unless you use `-auto-approve`)
- Asks for confirmation before proceeding
- Tracks changes in state file
- Can be interrupted and resumed safely

#### 4. **`terraform destroy` - Clean Up Everything**
**What it does:** Removes ALL infrastructure managed by this Terraform project
**Like:** Demolishing a building - it's gone forever!

**⚠️ DANGER:** This destroys REAL infrastructure and data!

### Essential Flags and Options

#### **`-auto-approve`** - Skip Confirmation Prompts
```bash
terraform apply -auto-approve    # Apply without asking "yes/no"
terraform destroy -auto-approve  # Destroy without asking "yes/no"
```
**Use cases:**
- ✅ Automation scripts and CI/CD
- ✅ When you're 100% confident
- ❌ Never in production without careful review

#### **`-var` and `-var-file`** - Pass Variables
```bash
terraform apply -var="instance_type=t3.micro"           # Single variable
terraform apply -var-file="production.tfvars"          # File of variables
```

#### **`-target`** - Focus on Specific Resources
```bash
terraform apply -target=aws_instance.web               # Only touch this resource
```
**⚠️ Use sparingly:** Breaks Terraform's dependency management

#### **`-refresh=false`** - Skip State Refresh
```bash
terraform plan -refresh=false                          # Use cached state
```
**Use case:** Faster plans when state is large

## Step-by-Step Tutorial

### Setup: Navigate to Your Previous Lesson

```bash
cd ~/terraform-course/lesson-4
```

**Verify you have the right files:**
```bash
ls -la
# Should show: main.tf, variables.tf, outputs.tf, .terraform/, terraform.tfstate
```

### Practice Session 1: The Safe Commands

#### Experiment with `terraform init`
```bash
# Run init again (safe - it's idempotent)
terraform init

# Notice: "Terraform has been successfully initialized!"
# This is safe to run multiple times
```

#### Master `terraform plan`
```bash
# Generate a plan (read-only, always safe)
terraform plan

# Save a plan to a file for later use
terraform plan -out=myplan

# Apply a saved plan (ensures exactly what you reviewed gets applied)
terraform apply myplan

# Remove the plan file
rm myplan
```

**Understanding plan output:**
```
Terraform will perform the following actions:

  # aws_instance.web will be created
  + resource "aws_instance" "web" {
      + ami                    = "ami-0c02fb55956c7d316"
      + instance_type          = "t2.micro"
      + public_ip              = (known after apply)
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

### Practice Session 2: Making Changes

#### Test Variable Override
```bash
# Change instance type via command line
terraform plan -var="instance_type=t3.nano"

# Notice how Terraform shows it will modify the instance
# Look for the ~ symbol indicating modification
```

#### Apply a Change
```bash
# Apply the change with confirmation
terraform apply -var="instance_type=t3.nano"
# Type 'yes' when prompted

# Check the result
terraform show | grep instance_type
```

### Practice Session 3: Advanced Commands

#### Target Specific Resources
```bash
# Plan changes for only the security group
terraform plan -target=aws_security_group.ssh

# This shows how to focus on specific resources
```

#### Refresh State
```bash
# Refresh state to detect manual changes
terraform refresh

# Show current state in human-readable format  
terraform show
```

#### Format Your Code
```bash
# Format all .tf files (good practice)
terraform fmt

# Check if formatting is needed (returns exit code)
terraform fmt -check
```

### Practice Session 4: State Management

#### Inspect State
```bash
# List all resources in state
terraform state list

# Show details of a specific resource
terraform state show aws_instance.web

# Move a resource in state (advanced)
# terraform state mv aws_instance.web aws_instance.renamed
```

### Practice Session 5: The Nuclear Option

#### Destroy Everything (Carefully!)
```bash
# Preview what would be destroyed
terraform plan -destroy

# Destroy with confirmation
terraform destroy
# Type 'yes' when prompted

# OR destroy without confirmation (use very carefully!)
# terraform destroy -auto-approve
```

**After destroy, verify cleanup:**
```bash
# State should be empty
terraform state list

# Plan should show everything needs to be created
terraform plan
```

## Code Examples
No changes to code here—practice commands on your `lesson-4` code.

## Hands-On Exercise

### Exercise 1: Command Workflow Practice

Complete this entire workflow and observe the outputs:

```bash
# 1. Start fresh (if you destroyed earlier)
terraform apply -auto-approve

# 2. Make a small change to test the workflow
echo 'variable "environment" { default = "development" }' >> variables.tf

# 3. Update main.tf to use the new variable
# Add this tag to the aws_instance resource:
# Environment = var.environment

# 4. Follow the complete workflow
terraform init     # (checks for new providers - none needed)
terraform plan     # (shows the tag will be added)
terraform apply    # (adds the tag)

# 5. Verify the change
terraform show | grep -A5 tags

# 6. Clean up
terraform destroy -auto-approve
```

### Exercise 2: Error Recovery Practice

Intentionally create an error and practice recovery:

```bash
# 1. Create infrastructure
terraform apply -auto-approve

# 2. Manually break something (simulate real-world scenario)
# Edit main.tf and change the AMI ID to something invalid like "ami-invalid"

# 3. Try to apply
terraform plan
# Observe: Terraform catches the error during planning!

# 4. Fix the AMI ID back to the original

# 5. Apply successfully
terraform apply -auto-approve
```

### Exercise 3: Plan Analysis Challenge

Study this plan output and answer the questions:

```bash
terraform plan -var="instance_type=t3.small"
```

**Questions:**
1. What symbol indicates the resource will be modified?
2. Which attributes will change?
3. Which attributes are "known after apply"?
4. What would happen if you applied this plan?

### Exercise 4: State Inspection

Practice examining your infrastructure state:

```bash
# If no infrastructure exists, create it first
terraform apply -auto-approve

# Now practice state commands
terraform state list
terraform state show aws_instance.web
terraform show

# Challenge: Find the private IP address of your instance using state commands
```

## Troubleshooting Tips

### Common Issues and Solutions

#### **Init Problems**

**Problem:** `terraform init` keeps re-downloading providers
```
Initializing provider plugins...
- Downloading plugin...
```
**Solutions:**
- This is normal after changing provider versions
- Delete `.terraform/` folder if corrupted: `rm -rf .terraform/`
- Check internet connection for download issues

**Problem:** Provider version conflicts
```
Error: Failed to query available provider packages
```
**Solutions:**
- Update version constraints in your terraform block
- Delete `.terraform.lock.hcl` to allow version updates
- Run `terraform init -upgrade` to update to latest compatible versions

#### **Plan/Apply Problems**

**Problem:** Apply fails partway through
```
Error: Error creating EC2 Instance: InvalidAMIID.NotFound
```
**Solution:**
- **Don't panic!** Terraform tracks what succeeded
- Fix the error (e.g., correct AMI ID)
- Run `terraform apply` again - it will continue where it left off

**Problem:** "Resource already exists" error
```
Error: resource already exists
```
**Solutions:**
- Someone created resources manually outside Terraform
- Import existing resource: `terraform import aws_instance.web i-1234567890abcdef0`
- Or delete the conflicting resource manually

#### **State Problems**

**Problem:** State file locked
```
Error: Error locking state: Error acquiring the state lock
```
**Solutions:**
- Wait - another Terraform process is running
- If stuck, force unlock: `terraform force-unlock LOCK_ID` (use carefully!)
- Check for crashed Terraform processes: `ps aux | grep terraform`

**Problem:** State drift (manual changes)
```
Note: Objects have changed outside of Terraform
```
**Solutions:**
- Run `terraform refresh` to update state
- Or run `terraform apply` to bring resources back to desired state

#### **Destroy Problems**

**Problem:** Resources won't destroy
```
Error: Error deleting VPC: DependencyViolation
```
**Solutions:**
- Dependencies exist that Terraform doesn't know about
- Manually clean up dependencies in AWS console
- Use `terraform destroy -target=resource_name` to destroy specific resources first

### Best Practices for Command Usage

#### **DO:**
- ✅ Always run `terraform plan` before `apply`
- ✅ Use `-auto-approve` only in automation
- ✅ Run `terraform fmt` to keep code clean
- ✅ Save important plans: `terraform plan -out=plan.tfplan`

#### **DON'T:**
- ❌ Run `destroy` in production without careful review
- ❌ Ignore warnings in plan output
- ❌ Use `-target` unless absolutely necessary
- ❌ Edit state files manually

### Emergency Commands

If you're in trouble, these commands can help:

```bash
# See what Terraform thinks exists
terraform state list

# Get detailed info about the current state
terraform show

# Check if configuration is valid
terraform validate

# See provider and Terraform versions
terraform version

# Get help for any command
terraform plan -help
terraform apply -help
```

## Summary / Key Takeaways
- You practiced the core commands safely.
- Plans are your best friend to avoid surprises.

## Next Steps
Use variables and outputs to make your code flexible and reusable.