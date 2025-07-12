---
title: Misconfigured S3 Bucket Exposes Sensitive Data
permalink: /cloud-sec1.html
---

# Cloud Security Incident: Misconfigured S3 Bucket Exposes Sensitive Data

## Scenario Overview

**Company**: [ redacted ]

**Cloud Provider**: AWS

**Services in Use**: Amazon S3, IAM, Lambda

**Security Tooling**: AWS Config, GuardDuty, Custom Lambda Audit Script

---

## The Problem

[ redacted ], a financial tech company, uses Amazon S3 to store customer statements and transaction logs. These objects are encrypted and stored in private buckets with access restricted to internal systems via IAM roles.

After deploying a new feature to share customer transaction exports via temporary download links, a DevOps engineer accidentally modified the bucket's policy to allow **public read access** for testing purposes — and forgot to revert it.

A week later, **GuardDuty** detects multiple anomaly events indicating that IP addresses from multiple countries are accessing the bucket’s contents. Further investigation shows that sensitive data, including partial account numbers and names, was publicly exposed.

---

## Root Cause

The bucket policy had the following misconfiguration:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::fintrust-exports/*"
    }
  ]
}
```

This `Principal: "*"` combined with `s3:GetObject` on the entire bucket granted **anonymous users** read access to **all** files.

---

## The Solution

### 1. **Immediate Fix: Revoke Public Access**

The DevOps team immediately removed the faulty policy and enabled S3 Block Public Access:

```bash
aws s3api put-bucket-policy --bucket fintrust-exports --policy file://corrected-policy.json
aws s3api put-public-access-block --bucket fintrust-exports --public-access-block-configuration \
    BlockPublicAcls=true \
    IgnorePublicAcls=true \
    BlockPublicPolicy=true \
    RestrictPublicBuckets=true
```

### 2. **Audit & Notification Lambda**

To ensure this never happens again, the security team built a Lambda function triggered by **S3 bucket policy changes** using **CloudTrail** and **EventBridge**.

Here’s a simplified version of the Lambda in Python (using `boto3`) that audits new policies and alerts if any public access is granted:

```python
import json
import boto3

def lambda_handler(event, context):
    s3 = boto3.client('s3')
    sns = boto3.client('sns')
    
    bucket_name = event['detail']['requestParameters']['bucketName']
    response = s3.get_bucket_policy(Bucket=bucket_name)
    policy = json.loads(response['Policy'])
    
    for statement in policy.get('Statement', []):
        if statement.get('Principal') == "*" and statement.get('Effect') == "Allow":
            if "s3:GetObject" in statement.get('Action', []):
                # Send alert to Security SNS topic
                sns.publish(
                    TopicArn='arn:aws:sns:us-west-2:123456789012:SecurityAlerts',
                    Subject='⚠️ Public Access Alert',
                    Message=f'Bucket "{bucket_name}" allows public read access!'
                )
                break

    return {"status": "checked"}
```

This function ensures that **every bucket policy update** is immediately scanned and alerts are sent if public access is detected.

### 3. **Implement Preventive Controls**

* **Service Control Policies (SCPs)** to deny public S3 bucket policies at the org level.
* **AWS Config Rules** to continuously monitor S3 bucket ACLs and policies.
* **Security Awareness Training** for DevOps teams on secure-by-default configurations.

---

## Final Outcome

Thanks to fast detection and remediation, [ redacted ] minimized the breach impact. The team published an internal postmortem and added automated guardrails across environments to prevent future misconfigurations.

---

## Takeaways

* **Least privilege** should be enforced at all times — especially for storage.
* **Automation** can significantly reduce human error.
* **Monitoring and alerting** are key in catching misconfigurations early.

## **Corrected S3 Bucket Policy (JSON)**

This policy **only allows access to the bucket for a specific IAM role** (e.g., a Lambda function or EC2 instance), avoiding any public access:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOnlyTrustedRoleAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/FintrustExportReader"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::fintrust-exports/*"
    }
  ]
}
```

> Make sure Block Public Access settings are still enabled at the bucket level to enforce security, even with this restrictive policy.

---

## **Set Up EventBridge Rule to Trigger Lambda on S3 Bucket Policy Changes**

You can use the AWS CLI to create the rule that listens for S3 bucket policy changes via CloudTrail and routes them to the auditing Lambda function.

### **Step A: Create EventBridge Rule**

```bash
aws events put-rule \
  --name "DetectS3PublicPolicyChange" \
  --event-pattern '{
    "source": ["aws.s3"],
    "detail-type": ["AWS API Call via CloudTrail"],
    "detail": {
      "eventSource": ["s3.amazonaws.com"],
      "eventName": ["PutBucketPolicy"]
    }
  }' \
  --state ENABLED
```

### **Step B: Attach Lambda Function as Target**

```bash
aws events put-targets \
  --rule "DetectS3PublicPolicyChange" \
  --targets "Id"="1","Arn"="arn:aws:lambda:us-west-2:123456789012:function:S3PolicyAuditLambda"
```

### **Step C: Give EventBridge Permission to Invoke the Lambda**

```bash
aws lambda add-permission \
  --function-name S3PolicyAuditLambda \
  --statement-id "AllowEventBridgeInvoke" \
  --action "lambda:InvokeFunction" \
  --principal events.amazonaws.com \
  --source-arn "arn:aws:events:us-west-2:123456789012:rule/DetectS3PublicPolicyChange"
```

---

## Final Architecture Flow

1. **DevOps makes a policy change** on S3 bucket.
2. **CloudTrail logs the event**, EventBridge detects `PutBucketPolicy`.
3. EventBridge **triggers the Lambda** (`S3PolicyAuditLambda`).
4. Lambda **analyzes the new policy**.
5. If the policy allows public access, it **sends an SNS alert** to the security team.

---