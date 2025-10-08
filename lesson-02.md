# Lesson 2: Installing Terraform on Ubuntu

## Learning Objectives
- Install Terraform on Ubuntu using the official apt repository.
- Install AWS CLI v2 and configure credentials.
- Verify your environment is ready.

## Concept Overview

Before we can start building infrastructure, we need to install the right tools. Think of this like setting up a workshop before building furniture.

### What We're Installing

**🔧 Terraform CLI (Command Line Interface):**
- The main program that reads your infrastructure code
- Converts your code into API calls to cloud providers
- Manages the lifecycle of your resources (create, update, delete)

**☁️ AWS CLI v2:**
- Amazon's official command-line tool
- Handles authentication with your AWS account
- Allows Terraform to securely access AWS services
- Useful for quick checks and troubleshooting

### Prerequisites Check

Before starting, make sure you have:
- ✅ Ubuntu 18.04 or newer
- ✅ Internet connection
- ✅ Administrator access (sudo privileges)
- ✅ An AWS account (free tier is fine)

**Don't have an AWS account?** 
1. Go to [aws.amazon.com](https://aws.amazon.com)
2. Click "Create an AWS Account"
3. Follow the signup process (requires credit card, but we'll use free services)

## Step-by-Step Tutorial

### Part 1: Install Terraform

Open your terminal (`Ctrl + Alt + T`) and navigate to your course directory:
```bash
cd ~/terraform-course
```

#### Step 1: Update Your System
```bash
sudo apt-get update
```
**What this does:** Updates the list of available packages to ensure you get the latest versions.

#### Step 2: Install Required Dependencies
```bash
sudo apt-get install -y gnupg software-properties-common curl unzip
```
**What each tool does:**
- `gnupg`: Handles encryption keys for security
- `software-properties-common`: Allows adding external software repositories
- `curl`: Downloads files from the internet
- `unzip`: Extracts compressed files

#### Step 3: Add HashiCorp's Official Repository
```bash
# Download HashiCorp's security key
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
```
**Why we need this:** Ensures the software we download is authentic and hasn't been tampered with.

```bash
# Add HashiCorp's repository to your system
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
```
**What this does:** Tells Ubuntu where to find HashiCorp's software packages.

#### Step 4: Install Terraform
```bash
# Refresh package list with new repository
sudo apt-get update

# Install Terraform
sudo apt-get install -y terraform
```

#### Step 5: Verify Terraform Installation
```bash
terraform -version
```
**Expected output:** Something like `Terraform v1.6.0` (version may vary)

**🎉 Success!** If you see a version number, Terraform is installed correctly.

### Part 2: Install AWS CLI v2

#### Step 1: Download AWS CLI
```bash
curl -fsSL "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
```
**What this does:** Downloads the AWS CLI installer to a file called `awscliv2.zip`

#### Step 2: Extract and Install
```bash
# Extract the downloaded file
unzip awscliv2.zip

# Install AWS CLI (requires sudo for system-wide installation)
sudo ./aws/install

# Clean up installation files (optional)
rm -rf aws awscliv2.zip
```

#### Step 3: Verify AWS CLI Installation
```bash
aws --version
```
**Expected output:** Something like `aws-cli/2.x.x Python/3.x.x`

**Why AWS CLI v2?** 
- Faster and more reliable than v1
- Better security features
- Required for modern AWS services
- Official recommendation from Amazon

### Part 3: Set Up AWS Credentials

Before configuring, you need AWS access keys. Here's how to get them:

#### Step 1: Create AWS Access Keys (First-time setup)

**If you already have access keys, skip to Step 2.**

1. **Log into AWS Console:**
   - Go to [aws.amazon.com](https://aws.amazon.com)
   - Click "Sign In to the Console"
   - Enter your account credentials

2. **Navigate to IAM (Identity and Access Management):**
   - In the search bar, type "IAM"
   - Click on "IAM" service

3. **Create a New User:**
   - Click "Users" in the left sidebar
   - Click "Add users"
   - User name: `terraform-user`
   - Select "Programmatic access"
   - Click "Next: Permissions"

4. **Set Permissions:**
   - Click "Attach existing policies directly"
   - Search for and select: `PowerUserAccess` (gives broad access for learning)
   - ⚠️ **Important:** For production, use more restrictive policies
   - Click "Next: Tags" → "Next: Review" → "Create user"

5. **Save Your Credentials:**
   - **Access Key ID:** Copy this value
   - **Secret Access Key:** Copy this value (you won't see it again!)
   - Download the CSV file or write these down securely

#### Step 2: Configure AWS CLI
```bash
aws configure
```

When prompted, enter your information:
```
AWS Access Key ID [None]: AKIA...your-access-key...
AWS Secret Access Key [None]: wJalr...your-secret-key...
Default region name [None]: us-east-1
Default output format [None]: json
```

**What each setting means:**
- **Access Key ID:** Your unique identifier
- **Secret Access Key:** Your password (keep this secret!)
- **Default region:** Where AWS will create resources (us-east-1 = N. Virginia)
- **Output format:** How AWS CLI displays results (json is most readable)

#### Step 3: Test Your Configuration
```bash
# This command asks AWS "who am I?"
aws sts get-caller-identity
```

**Expected output:**
```json
{
    "UserId": "AIDA...",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/terraform-user"
}
```

**🎉 Success!** If you see your account information, AWS CLI is configured correctly.

(Optional) Set environment variables for Terraform variables later:
```bash
echo 'export TF_VAR_region="us-east-1"' >> ~/.bashrc
echo 'export TF_LOG=""' >> ~/.bashrc
source ~/.bashrc
```

## Code Examples
None yet.

## Hands-On Exercise
- Run `terraform -version` and `aws sts get-caller-identity` to ensure both tools work.

## Troubleshooting Tips
- If `terraform` not found: Ensure the HashiCorp repo steps ran without errors and re-run `sudo apt-get update && sudo apt-get install -y terraform`.
- If `aws` not found: Ensure unzip worked and that you ran `sudo ./aws/install`.
- If `aws sts get-caller-identity` fails: Check your credentials with `aws configure list`.

## Summary / Key Takeaways
- You now have Terraform and AWS CLI v2 set up on Ubuntu.
- Credentials are configured for AWS operations.

## Next Steps
Understand providers and the Terraform workflow.