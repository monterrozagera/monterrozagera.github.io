---
title: AWS Intrusion via Compromised CI/CD Pipeline
permalink: /cloud-sec3.html
---

## Cloud Security Incident: AWS Cloud Intrusion via Compromised CI/CD Pipeline

## Scenario Overview

**Company**: [ redacted ]

**Cloud Provider**: AWS

**Services in Use**: ECS Fargate, S3, Automation roles in IAM, AWS Secrets Manager, CI/CD via GitHub Actions

---

## Step-by-Step Attack Flow

### 1. **Initial Access via Stolen GitHub Token**

An attacker:

* Phishes or leaks a GitHub **Personal Access Token (PAT)**.
* Gets access to the company's private CI/CD repo.
* Modifies a workflow YAML file to include:

  ```yaml
  - run: aws sts get-caller-identity
  - run: aws s3 sync s3://sensitive-bucket ./exfil
  ```
* Uses `aws configure` with **long-lived IAM user credentials** embedded in the repo (bad practice).

**Impact:** They now exfiltrate S3 data via a poisoned CI pipeline.

---

### 2. **Lateral Movement via Overprivileged IAM Roles**

* Attacker extracts credentials for the **deployer IAM role** from CI logs or environment variables.
* Uses `sts:AssumeRole` to escalate to a **high-privilege role** (e.g., `AdminAssumeRole`).
* Deploys malicious containers to ECS via CLI.

---

### 3. **Persistence & Stealth**

* Sets up a Lambda function that triggers every hour to:

  * Check if their backdoor is removed
  * Re-create malicious ECS tasks
* Attaches `CloudTrail:StopLogging` or modifies logging retention to hide traces.

---

## Detection & Mitigation

### 1. **IAM Best Practices**

* Enable **least privilege**: limit `sts:AssumeRole`, especially in CI/CD roles.
* Rotate credentials regularly.
* Use **short-lived tokens**, like GitHub OIDC + IAM federation.

### 2. **Guard CI/CD**

* Use **branch protections** and signed commits.
* Store secrets in AWS Secrets Manager or GitHub Encrypted Secrets — not in code.
* Monitor CI workflows for unusual changes.

### 3. **S3 Hardening**

* Disable public access on all buckets.
* Enable bucket policies with `aws:PrincipalOrgID` conditions.
* Enable **S3 server-side encryption** and **access logs**.

### 4. **CloudTrail & GuardDuty**

* Ensure **CloudTrail** is **enabled in all regions** and logging to a protected S3 bucket.
* Use **Amazon GuardDuty** to detect:

  * Unusual data exfiltration
  * STS role abuse
  * EC2 credential theft

### 5. **Automated Response**

* Use AWS Config or Security Hub to flag violations.
* Use Lambda or Step Functions to automatically:

  * Revoke compromised keys
  * Delete unauthorized ECS services
  * Notify the security team via SNS or Slack

---

## Example Mitigation Policy Snippet

Deny overbroad `AssumeRole`:

```json
{
  "Effect": "Deny",
  "Action": "sts:AssumeRole",
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "aws:RequestedRegion": "us-west-2"
    }
  }
}
```

---

## Key Takeaways

| Threat Vector            | Mitigation                                         |
| ------------------------ | -------------------------------------------------- |
| Stolen CI tokens         | Use GitHub OIDC + no static credentials            |
| Overprivileged IAM roles | Apply least privilege + resource-level permissions |
| S3 data exfil            | Use SCPs, object-level logging                     |
| Malicious persistence    | Monitor CloudTrail, limit `lambda:*` actions       |
| Logging tampering        | Lock CloudTrail S3 bucket with bucket policy       |