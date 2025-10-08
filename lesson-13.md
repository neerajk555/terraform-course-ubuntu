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

## Troubleshooting Tips
- SSH issues: Ensure key path and username `ec2-user` are correct.
- Ansible `UNREACHABLE`: Instance might not be ready—rerun the `null_resource` by tainting or `terraform apply` again.

## Summary / Key Takeaways
- Terraform and Ansible integrate cleanly using outputs and local-exec.
- You automated server config after provisioning.

## Next Steps
Learn best practices and how to organize your Terraform projects.