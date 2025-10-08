# Lesson 1: Introduction to Terraform and Infrastructure as Code (IaC)

## Learning Objectives
- Understand what Infrastructure as Code (IaC) is and why it matters.
- Learn what Terraform is and where it fits in cloud automation.
- Get familiar with key Terraform concepts: providers, resources, state.

## Concept Overview

### What is Infrastructure as Code (IaC)?
Imagine you're setting up a restaurant. The traditional way would be to:
1. Call different suppliers manually
2. Set up tables one by one
3. Configure the kitchen equipment by hand
4. Remember all these steps for the next restaurant

**Infrastructure as Code (IaC)** is like having a detailed recipe book that automatically sets up your entire restaurant. Instead of clicking buttons in cloud consoles (like AWS web interface), you write a "recipe" (code) that describes exactly what you want.

### Real-World Example
**Traditional way (Manual):**
- Log into AWS console
- Click "Launch Instance"
- Select instance type, security groups, etc.
- Click through 10+ screens
- Repeat for every server you need

**IaC way (Automated):**
- Write one text file describing your servers
- Run one command: `terraform apply`
- Get identical infrastructure every time

### What Makes Terraform Special?

**🌍 Cloud-Agnostic:** Works with 100+ cloud providers
- AWS (Amazon), Azure (Microsoft), GCP (Google)
- Even works with Docker, GitHub, and databases
- Write once, deploy anywhere (with minor changes)

**📝 Declarative:** You describe the "what," not the "how"
- You say: "I want 3 web servers"
- Terraform figures out: "Create server 1, create server 2, create server 3"
- If you later say "I want 5 web servers," Terraform adds 2 more

**🔄 Reproducible:** Same code = identical infrastructure
- Deploy to development, staging, and production
- Share with team members
- Disaster recovery becomes simple

### Essential Terraform Vocabulary

**📦 Provider:** Think of it as a translator
- Terraform speaks "Terraform language"
- AWS speaks "AWS language"
- Provider translates between them
- Examples: `aws`, `azurerm`, `google`, `docker`

**🏗️ Resource:** Any "thing" you want to create
- Virtual machines (EC2 instances)
- Networks (VPCs)
- Databases (RDS)
- Load balancers
- DNS records

**📋 State:** Terraform's memory
- A file that remembers what it created
- Maps your code to real-world resources
- Helps Terraform know what to update/delete

### Why Should You Care About IaC?

**⚡ Speed:** Create complex infrastructure in minutes, not hours
**🔒 Consistency:** No more "it works on my machine" problems  
**📚 Documentation:** Your code IS your documentation
**👥 Collaboration:** Version control for infrastructure
**💰 Cost Control:** Easily tear down expensive resources when not needed

## Step-by-Step Tutorial
No commands yet. You'll install Terraform in Lesson 2.

## Code Examples
No code yet. You'll write your first configuration in Lesson 4.

## Hands-On Exercise

Let's start by setting up your workspace:

### 1. Open Terminal
- Press `Ctrl + Alt + T` to open Terminal
- This is where you'll type all commands throughout the course

### 2. Create Your Course Directory
```bash
# Navigate to your home directory
cd ~

# Create a dedicated folder for this course
mkdir terraform-course

# Navigate into the folder
cd terraform-course

# Verify you're in the right place
pwd
# Should output: /home/your-username/terraform-course
```

**What each command does:**
- `cd ~`: Changes to your home directory (`/home/your-username`)
- `mkdir terraform-course`: Creates a new folder called "terraform-course"
- `cd terraform-course`: Enters that folder
- `pwd`: Shows your current directory path

### 3. Create a Welcome File (Optional)
```bash
# Create a simple text file to test your setup
echo "Welcome to my Terraform learning journey!" > welcome.txt

# View the file contents
cat welcome.txt

# List files in current directory
ls -la
```

**Important Note:** We'll use `~/terraform-course` as our base directory throughout all 15 lessons. Keep all your practice files organized here!

## Troubleshooting Tips
- None for now—setup begins next lesson.

## Summary / Key Takeaways
- IaC lets you manage cloud resources with code.
- Terraform is a popular, cloud-agnostic IaC tool.
- You'll write code to define infrastructure and Terraform will apply it.

## Next Steps
Install Terraform on Ubuntu.