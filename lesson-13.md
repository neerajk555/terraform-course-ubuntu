# Lesson 13: Integrating Terraform with Ansible

## Learning Objectives
- Use Terraform outputs to generate an Ansible inventory.
- Run Ansible to configure instances after provisioning.
- Understand integration workflows.

## Concept Overview
Terraform builds infrastructure. Ansible configures software. They pair well:
- Terraform outputs IPs/hosts.
- Ansible uses inventory and SSH to configure.

## Step-by-Step Tutorial
- Install Ansible on Ubuntu:
```bash
sudo apt-get update
sudo apt-get install -y ansible
ansible --version
```

- New folder with Terraform + Ansible:
```bash
mkdir -p ~/terraform-course/lesson-13
cd ~/terraform-course/lesson-13
```

- `main.tf`: create an instance and write an inventory file with `local_file`, then run `ansible-playbook` via `null_resource`:
```bash
cat > main.tf << 'EOF'
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
    local = { source = "hashicorp/local", version = "~> 2.4" }
  }
}

provider "aws" { region = "us-east-1" }

data "aws_vpc" "default" { default = true }
data "aws_subnet_ids" "default" { vpc_id = data.aws_vpc.default.id }

resource "aws_security_group" "web" {
  name        = "lesson-13-web"
  description = "Allow SSH and HTTP"
  vpc_id      = data.aws_vpc.default.id
  ingress { from_port = 22, to_port = 22, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] }
  ingress { from_port = 80, to_port = 80, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] }
  egress  { from_port = 0, to_port = 0, protocol = "-1", cidr_blocks = ["0.0.0.0/0"] }
}

resource "aws_key_pair" "this" {
  key_name   = "lesson-13-key"
  public_key = file("~/.ssh/terraform_course.pub")
}

data "aws_ami" "al2" {
  most_recent = true
  owners      = ["amazon"]
  filter { name = "name", values = ["amzn2-ami-hvm-*-x86_64-gp2"] }
}

resource "aws_instance" "web" {
  ami                    = data.aws_ami.al2.id
  instance_type          = "t2.micro"
  subnet_id              = element(data.aws_subnet_ids.default.ids, 0)
  vpc_security_group_ids = [aws_security_group.web.id]
  key_name               = aws_key_pair.this.key_name
  associate_public_ip_address = true
  tags = { Name = "lesson-13-web" }
}

resource "local_file" "inventory" {
  filename = "${path.module}/inventory.ini"
  content  = <<EOT
[web]
${aws_instance.web.public_ip} ansible_user=ec2-user ansible_ssh_private_key_file=~/.ssh/terraform_course
EOT
}

resource "null_resource" "configure" {
  triggers = {
    ip = aws_instance.web.public_ip
  }

  provisioner "local-exec" {
    command = "ansible-playbook -i inventory.ini playbook.yml"
  }
}

output "web_ip" { value = aws_instance.web.public_ip }
EOF
```

- Create `playbook.yml`:
```bash
cat > playbook.yml << 'EOF'
- hosts: web
  become: yes
  tasks:
    - name: Install NGINX
      yum:
        name: nginx
        state: present

    - name: Enable and start NGINX
      systemd:
        name: nginx
        enabled: yes
        state: started

    - name: Create index page
      copy:
        dest: /usr/share/nginx/html/index.html
        content: "Hello from Ansible!\n"
EOF
```

- Run:
```bash
terraform init
terraform apply -auto-approve
curl -I http://$(terraform output -raw web_ip)
```

## Code Examples
Included above.

## Hands-On Exercise
- Modify the playbook to install `git` and verify with `ansible -i inventory.ini -m command -a "git --version" web`.

## Clean Up Resources ⚠️ IMPORTANT

This lesson creates EC2 instances, security groups, and key pairs, plus generates local files that should be cleaned up.

### Standard Infrastructure Cleanup

```bash
# Preview what will be destroyed
terraform plan -destroy

# Destroy all AWS resources
terraform destroy -auto-approve

# Verify AWS resources are gone
terraform state list
# Should be empty

aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=lesson-13-web" \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name]' \
  --output table
```

### Clean Up Generated Files

This lesson creates local files that you might want to remove:

```bash
# Remove Ansible inventory file
rm -f inventory.ini

# Remove Ansible playbook (or keep for reference)
# rm -f playbook.yml

# Check what files remain
ls -la
# Should show: main.tf and optionally playbook.yml
```

### Understanding Terraform + Ansible Cleanup

**🔗 Integration Impact:**
- Terraform manages AWS infrastructure lifecycle
- Ansible only configures - no cleanup needed for software
- Local files created by `local_file` resource are tracked by Terraform

**🗂️ What Gets Cleaned Up:**

**✅ AWS Resources (via Terraform):**
- 🖥️ EC2 instance (with all Ansible-installed software)
- 🛡️ Security group
- 🔑 SSH key pair in AWS
- 🏷️ All tags and metadata

**✅ Local Files (via Terraform):**
- 📄 `inventory.ini` (tracked by `local_file` resource)
- 🔄 Terraform removes this during `destroy`

**✅ Manual Cleanup (Optional):**
- 📋 `playbook.yml` (not managed by Terraform)
- 📁 Any other files you created

### Verify Complete Integration Cleanup

```bash
# Check no Terraform-managed resources remain
terraform state list

# Check no AWS resources with lesson tags
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=lesson-13-web" \
  --query 'Reservations[*].Instances[?State.Name!=`terminated`].[InstanceId,State.Name]' \
  --output table

# Verify local inventory file was removed
ls inventory.ini 2>/dev/null || echo "inventory.ini successfully removed by Terraform"

# Check if playbook still exists (this is normal)
ls -la playbook.yml
```

### Cost Considerations

**💰 Resources That Were Costing Money:**
- EC2 t2.micro instance (~$8.50/month)
- EBS storage (~$0.10/GB/month)
- Data transfer costs

**🆓 Free Components:**
- Ansible software (open source)
- Local file operations
- SSH connections
- Terraform state operations

### Integration Best Practices for Future

**🔄 Cleanup Workflow:**
1. Always `terraform destroy` first (removes infrastructure)
2. Manually remove any generated files not tracked by Terraform
3. Keep playbooks for reuse but remove generated inventories

**💡 Pro Tip:** In production, consider using dynamic inventories or external inventory scripts instead of `local_file` resources to avoid mixing infrastructure and configuration management concerns.

## Troubleshooting Tips
- SSH issues: Ensure key path and username `ec2-user` are correct.
- Ansible `UNREACHABLE`: Instance might not be ready—rerun the `null_resource` by tainting or `terraform apply` again.

## Summary / Key Takeaways
- Terraform and Ansible integrate cleanly using outputs and local-exec.
- You automated server config after provisioning.

## Next Steps
Learn best practices and how to organize your Terraform projects.