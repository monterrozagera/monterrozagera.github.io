---
title: Learning Terraform
permalink: /learning-terraform.html
---

## Learning Terraform

Terraform is an **Infrastructure as Code (IaC)** tool designed to help **provision, manage, and version cloud infrastructure** using declarative configuration files.

It supports a wide range of providers (AWS, Azure, GCP, etc.) and enables managing **resources in a consistent, repeatable, and automated** way.

---

### Basic Concepts

#### Providers

Define the platform you're interacting with (AWS, Azure, etc.).

```hcl
provider "aws" {
  region = "us-west-2"
}
```

#### Resources

Define the actual infrastructure you want to create.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```

#### Variables

Allow dynamic input into your configuration.

```hcl
variable "region" {
  default = "us-west-2"
}
```

#### Outputs

Return values from your Terraform configuration.

```hcl
output "instance_ip" {
  value = aws_instance.web.public_ip
}
```

---

### Example: Creating an EC2 Instance

1. **main.tf**

```hcl
provider "aws" {
  region = var.region
}

resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  tags = {
    Name = "ExampleInstance"
  }
}
```

2. **variables.tf**

```hcl
variable "region" {
  default = "us-west-2"
}
```

3. **outputs.tf**

```hcl
output "public_ip" {
  value = aws_instance.example.public_ip
}
```

---

### Workflow

```bash
terraform init      # Initialize the working directory
terraform plan      # Show what Terraform will do
terraform apply     # Apply the configuration and create resources
terraform destroy   # Tear down the resources
```

---

### Modules

Terraform modules are reusable pieces of code that help you organize and scale your infrastructure.

Example of calling a module:

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "4.0.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-west-2a", "us-west-2b"]
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
}
```

---

### Best Practices

* Use remote backends (e.g., S3 + DynamoDB for AWS) for shared state
* Use `terraform fmt` and `terraform validate` before committing
* Break infrastructure into **environments** (dev/staging/prod)
* Keep secrets in secure tools like AWS Secrets Manager or Vault
* Use modules to avoid repetition

---

### Real-World Use Cases

* Provision AWS infrastructure: EC2, VPC, RDS, S3, IAM
* Automate GCP or Azure environments
* Integrate with CI/CD pipelines (GitHub Actions, Jenkins)
* Spin up test environments automatically
* Reuse and version environments across multiple teams