# Lesson 10: Provisioners and Remote Execution

## Learning Objectives
- Use `remote-exec` to run commands on new instances.
- Understand when to use provisioners (and when not to).
- Set up SSH connections in Terraform.

## Concept Overview
Provisioners can run scripts/commands during resource creation (e.g., install packages). They're a last resort—prefer user-data or configuration management tools. But they're useful to learn.

## Step-by-Step Tutorial
- Use your key pair from Lesson 4, or generate a new one.
- New folder:
```bash
mkdir -p ~/terraform-course/lesson-10
cd ~/terraform-course/lesson-10
```

- `main.tf` with `remote-exec` to install NGINX:
```bash
cat > main.tf << 'EOF'
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}

provider "aws" { region = "us-east-1" }

data "aws_vpc" "default" { default = true }
data "aws_subnet_ids" "default" { vpc_id = data.aws_vpc.default.id }

resource "aws_security_group" "web" {
  name        = "lesson-10-web"
  description = "Allow SSH and HTTP"
  vpc_id      = data.aws_vpc.default.id

  ingress { from_port = 22, to_port = 22, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] }
  ingress { from_port = 80, to_port = 80, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] }

  egress  { from_port = 0, to_port = 0, protocol = "-1", cidr_blocks = ["0.0.0.0/0"] }
}

resource "aws_key_pair" "this" {
  key_name   = "lesson-10-key"
  public_key = file("~/.ssh/terraform_course.pub")
}

data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]
  filter { name = "name", values = ["amzn2-ami-hvm-*-x86_64-gp2"] }
}

resource "aws_instance" "web" {
  ami                    = data.aws_ami.amazon_linux_2.id
  instance_type          = "t2.micro"
  subnet_id              = element(data.aws_subnet_ids.default.ids, 0)
  vpc_security_group_ids = [aws_security_group.web.id]
  key_name               = aws_key_pair.this.key_name
  associate_public_ip_address = true

  provisioner "remote-exec" {
    inline = [
      "sudo yum -y update",
      "sudo yum -y install nginx",
      "sudo systemctl enable nginx",
      "sudo systemctl start nginx"
    ]

    connection {
      type        = "ssh"
      user        = "ec2-user"
      private_key = file("~/.ssh/terraform_course")
      host        = self.public_ip
    }
  }

  tags = { Name = "lesson-10-web" }
}

output "web_ip" { value = aws_instance.web.public_ip }
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
- Modify the inline commands to write a custom index.html to `/usr/share/nginx/html/index.html` and verify via curl.

## Troubleshooting Tips
- If remote-exec fails: Ensure the security group allows SSH (22) and you're using the correct username (`ec2-user` for Amazon Linux 2).
- If yum install fails: Wait a minute after instance creation—then re-apply with `-replace` if needed.

## Summary / Key Takeaways
- Provisioners can bootstrap instances, but use them sparingly.
- You installed and started NGINX automatically.

## Next Steps
Manage multiple environments (dev/staging/prod).