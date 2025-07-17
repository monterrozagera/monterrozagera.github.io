---
title: Immutable Infrastructure with Ansible, Packer & Terraform on AWS
permalink: /automating-ansible-aws-ami.html
---

# Immutable Infrastructure with Ansible, Packer & Terraform on AWS

Building secure and consistent infrastructure doesn't end with baking an AMI — you need a scalable way to **deploy** it across environments. In this advanced follow-up, we’ll extend our previous AWS + Ansible AMI baking setup by using **Terraform** to:

* Deploy infrastructure based on our **prebaked AMIs**
* Automate **networking, EC2 instances**, and **tags**
* Manage deployments as **code with version control**

---

## Workflow Overview

1. Bake hardened AMIs with Ansible + Packer
2. Upload to AWS with meaningful naming
3. Use Terraform to **deploy** EC2 instances from baked AMIs

---

## Directory Structure

```bash
aws-immutable-infra/
├── packer/
│   └── packer-template.json
├── ansible/
│   └── playbook.yml
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
```

---

## Terraform: main.tf

This file deploys an EC2 instance using the **latest baked AMI** filtered by name and tag.

```hcl
provider "aws" {
  region = var.aws_region
}

data "aws_ami" "secure_ami" {
  most_recent = true
  owners      = ["self"]

  filter {
    name   = "name"
    values = ["secure-ami-*"]
  }
}

resource "aws_instance" "hardened_ec2" {
  ami                         = data.aws_ami.secure_ami.id
  instance_type               = var.instance_type
  subnet_id                   = var.subnet_id
  associate_public_ip_address = true

  vpc_security_group_ids = [var.security_group_id]
  key_name               = var.key_name

  tags = {
    Name        = "SecureInstance"
    Environment = "Production"
  }
}
```

---

## variables.tf

```hcl
variable "aws_region" {
  default = "us-west-2"
}

variable "instance_type" {
  default = "t2.micro"
}

variable "subnet_id" {}
variable "security_group_id" {}
variable "key_name" {}
```

---

## outputs.tf

```hcl
output "instance_public_ip" {
  value = aws_instance.hardened_ec2.public_ip
}
```

---

## Security Group and Subnet Example (Optional)

If you don’t already have infrastructure to plug into, you can add:

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "main" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}

resource "aws_security_group" "ssh_only" {
  name        = "allow_ssh"
  description = "Allow SSH"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

## Full Flow: Bake and Deploy

```bash
# Step 1: Bake AMI
cd packer
packer build packer-template.json

# Step 2: Deploy infrastructure
cd ../terraform
terraform init
terraform apply -auto-approve \
  -var 'subnet_id=subnet-xxxx' \
  -var 'security_group_id=sg-xxxx' \
  -var 'key_name=my-ssh-key'
```

---

## Why This Is Powerful

* **Consistency**: AMI is configured once, reused many times.
* **Security**: Hardened at the image level before deployment.
* **Reproducibility**: Deploy the same infra with the same config.
* **Auditing**: Infrastructure changes are tracked in Git.

---

## Bonus Ideas

* Add **Ansible roles** for vulnerability scanning (e.g. OpenSCAP).
* Use **Terraform Cloud** or **GitHub Actions** for CI/CD.
* Register your AMI in **multiple regions** using `aws_ami_copy`.

---

## Summary

By combining **Ansible + Packer + Terraform**, you get a fully automated and secure path from configuration to deployment — the foundation of immutable, production-grade infrastructure on AWS.