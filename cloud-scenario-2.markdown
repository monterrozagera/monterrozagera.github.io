---
title: IAM Privilege Escalation Across AWS Accounts
permalink: /cloud-sec2.html
---

# Cloud Security Incident: IAM Privilege Escalation Across AWS Accounts

## Scenario Overview

**Company**: [ redacted ]

**Cloud Provider**: AWS

**Services in Use**: IAM, AWS Organizations, STS, Lambda, CloudTrail, CloudWatch, AWS Config

**Security Tooling**: Detective, GuardDuty, Custom STS Log Analyzer Lambda

---

## The Problem

[ redacted ] uses a multi-account AWS structure governed by **AWS Organizations**:

* **Account A (Dev)**: Developer sandbox
* **Account B (Prod)**: Production services
* **Account C (Security)**: GuardDuty, centralized logging

A developer with access to **Account A** discovers they can assume a role into **Account B** due to overly broad **IAM trust policies** and missing condition checks.

They accidentally run infrastructure scripts in Prod using an elevated role and overwrite production S3 bucket permissions, taking down customer-facing dashboards for two hours.

---

## Root Cause

### Misconfigured IAM Role Trust Policy in Account B:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111111111111:user/dev-john"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

This policy allows `dev-john` (Account A) to assume a `ProdAdminRole` in Account B **without proper conditions** like MFA, source IP, or tag checks.

---

## The Solution

### 1. **Immediate Fix: Harden Trust Policies**

The security team rewrote the role trust policy with **tight conditions** and allowed only a federated role from a controlled identity provider:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111111111111:role/OrgCrossAccountRole"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        },
        "StringEquals": {
          "aws:PrincipalOrgID": "o-abc123xyz"
        }
      }
    }
  ]
}
```

---

### 2. **Set Up an STS AssumeRole Monitor with Lambda**

To monitor suspicious cross-account role assumptions, a **CloudTrail trail** was already in place. They added a **Lambda function** triggered by EventBridge to scan all `AssumeRole` events and flag unexpected patterns.

#### Lambda Code Snippet:

```python
import json
import boto3

SECURITY_ACCOUNTS = ['222222222222']  # security account
ORG_ID = 'o-abc123xyz'

def lambda_handler(event, context):
    detail = event['detail']
    assumed_role = detail['requestParameters']['roleArn']
    source_user = detail['userIdentity']['arn']
    org_id = detail['userIdentity'].get('sessionContext', {}).get('sessionIssuer', {}).get('orgId', '')

    if org_id != ORG_ID:
        sns = boto3.client('sns')
        sns.publish(
            TopicArn='arn:aws:sns:us-west-2:222222222222:SecurityAlerts',
            Subject='🚨 Unexpected Cross-Account AssumeRole',
            Message=f'User {source_user} assumed {assumed_role} from outside expected org {ORG_ID}'
        )

    return {"status": "checked"}
```

> This script only sends alerts if the assume role event occurs **outside the organization boundary** — helping detect insider threats or misconfigurations.

---

### 3. **Enforce SCP Guardrails**

They implemented a **Service Control Policy (SCP)** at the Org level to **explicitly deny cross-account role assumptions unless from known roles**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyAssumeRoleOutsideTrustedRoles",
      "Effect": "Deny",
      "Action": "sts:AssumeRole",
      "Resource": "*",
      "Condition": {
        "StringNotEqualsIfExists": {
          "aws:PrincipalArn": [
            "arn:aws:iam::*:role/OrgCrossAccountRole"
          ]
        }
      }
    }
  ]
}
```

---

## Final Outcome

* Incident impact was minimized to 2 hours of service disruption.
* Dev access to prod is now only possible through **federated access via SSO**.
* The organization added **real-time STS audit logging**.
* All cross-account trust relationships were validated with **Config Rules**.

---

## Takeaways

* Trust policies **must include conditions** like `aws:PrincipalOrgID` and `aws:MultiFactorAuthPresent`.
* **SCPs and federated access** should enforce strict boundaries across accounts.
* **Real-time detection and notification** can reduce damage during IAM misuse.
* Review all **AssumeRole permissions and trust relationships** in any multi-account AWS environment.

---