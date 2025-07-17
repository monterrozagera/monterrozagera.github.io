---
title: Automating with Ansible and AWS AMI Baking
permalink: /automating-ansible-aws-ami.html
---

#  Automating Immutable Infrastructure with Ansible and AWS AMI Baking

In modern DevOps and cloud security practices, building **immutable infrastructure** is a core strategy for ensuring predictable, secure, and consistent deployments. One advanced use case is **automating the creation of preconfigured AMIs (Amazon Machine Images)** using **Ansible**. These AMIs can be deployed at scale without needing to reconfigure systems at boot time.

In this article, we’ll walk through how to:

* Use **Ansible** to configure EC2 instances.
* Automate **AMI baking** using **Packer** and Ansible.
* Ensure repeatable and secure server provisioning.

---

## Tools Required

* **Ansible**
* **HashiCorp Packer**
* **AWS CLI** or credentials
* An existing **IAM role/user** with `ec2:CreateImage`, `ec2:RunInstances`, etc.

---

## Why Use Immutable Infrastructure?

With immutable infrastructure, you don’t patch or modify live servers — instead, you bake changes into a new image and deploy it. This:

* Reduces configuration drift
* Ensures faster recovery and deployment
* Enables better security auditing

---

## Step-by-Step: Automating AMI Baking with Ansible

### 1. Project Structure

```bash
aws-ami-baking/
├── packer-template.json
├── ansible/
│   ├── playbook.yml
│   └── roles/
│       └── harden/
│           └── tasks/main.yml
```

---

### 2. Sample Packer Template (`packer-template.json`)

```json
{
  "variables": {
    "aws_region": "us-west-2"
  },
  "builders": [{
    "type": "amazon-ebs",
    "region": "{{user `aws_region`}}",
    "source_ami_filter": {
      "filters": {
        "virtualization-type": "hvm",
        "name": "ubuntu/images/*ubuntu-20.04-amd64-server-*",
        "root-device-type": "ebs"
      },
      "owners": ["099720109477"],
      "most_recent": true
    },
    "instance_type": "t2.micro",
    "ssh_username": "ubuntu",
    "ami_name": "secure-ami-{{timestamp}}"
  }],
  "provisioners": [{
    "type": "ansible",
    "playbook_file": "ansible/playbook.yml"
  }]
}
```

---

### 3. Ansible Playbook (`ansible/playbook.yml`)

```yaml
- name: Harden and prepare AMI
  hosts: all
  become: true
  roles:
    - harden
```

---

### 4. Security Hardening Role (`roles/harden/tasks/main.yml`)

```yaml
---
- name: Update all packages
  apt:
    upgrade: dist
    update_cache: yes

- name: Install essential packages
  apt:
    name: ['fail2ban', 'ufw', 'unattended-upgrades']
    state: present

- name: Enable UFW firewall
  ufw:
    state: enabled
    policy: deny

- name: Allow SSH through firewall
  ufw:
    rule: allow
    port: '22'
    proto: tcp

- name: Disable root SSH login
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^PermitRootLogin'
    line: 'PermitRootLogin no'

- name: Ensure SSH uses protocol 2
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^Protocol'
    line: 'Protocol 2'

- name: Reboot system (optional)
  reboot:
    reboot_timeout: 600
```

---

## Running the Bake Process

Make sure you have your AWS credentials set via environment variables or config file, then:

```bash
cd aws-ami-baking/
packer build -var 'aws_region=us-west-2' packer-template.json
```

This will:

1. Launch a temporary EC2 instance.
2. Run the Ansible playbook to configure it.
3. Create a new AMI from the instance.
4. Terminate the instance.

---

## Use Cases

* **Security-hardened base images** for production
* **Golden images** for CI/CD pipelines
* **Incident recovery** via consistent AMI deployment
* **Cloud forensics**: pre-bake investigation tools and logging agents

---

## Security Tips

* Bake in **CIS benchmark settings** using community roles
* Preinstall **CloudWatch Agents** for logging
* Store AMI creation logs and hashes for **forensic reproducibility**

---

## Final Thoughts

Automating secure, immutable infrastructure with Ansible and Packer is a powerful way to bring DevSecOps practices to AWS environments. It ensures repeatability, improves security posture, and reduces configuration drift in live systems.

You can extend this by:

* Using **Terraform** to deploy AMIs
* Adding **Molecule** tests to your Ansible roles
* Integrating with **GitHub Actions** or other CI tools
