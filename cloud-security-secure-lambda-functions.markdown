---
title: Secure Lambda Functions
permalink: /cloud-sec-lambda.html
---

## Key Lambda Security Principles

| Area                      | Best Practice                                                   |
| ------------------------- | --------------------------------------------------------------- |
| **IAM Permissions**       | Use the principle of least privilege.                           |
| **Environment Variables** | Encrypt sensitive data with KMS.                                |
| **VPC Settings**          | Attach to private subnets, avoid public internet if not needed. |
| **Logging**               | Enable CloudWatch Logs, scrub sensitive info.                   |
| **Code Safety**           | Validate input, avoid dependency vulnerabilities.               |
| **Triggers**              | Validate sources (e.g., API Gateway or S3 events).              |
| **Timeout & Memory**      | Use minimal values to reduce exposure window.                   |

---

## Secure Lambda Function Example

Let’s say you have a Lambda that handles a webhook POST request from a third-party API.

---

### Scenario

**Goal**: Validate and log incoming JSON requests from a webhook, then store sanitized data in DynamoDB.

---

### 1. IAM Role for Lambda (least privilege)

Use a role like this (attach via Lambda console or IAM):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["dynamodb:PutItem"],
      "Resource": "arn:aws:dynamodb:us-west-2:123456789012:table/SafeWebhookData"
    },
    {
      "Effect": "Allow",
      "Action": ["logs:CreateLogStream", "logs:PutLogEvents"],
      "Resource": "*"
    }
  ]
}
```

---

### 2. Secure Lambda Function Code (Node.js Example)

```js
const AWS = require("aws-sdk");
const dynamo = new AWS.DynamoDB.DocumentClient();

exports.handler = async (event) => {
  try {
    if (event.httpMethod !== "POST") {
      return { statusCode: 405, body: "Method Not Allowed" };
    }

    const data = JSON.parse(event.body || "{}");

    if (!data.email || typeof data.email !== "string") {
      return { statusCode: 400, body: "Invalid input" };
    }

    const safeData = {
      email: data.email.toLowerCase().trim(),
      timestamp: new Date().toISOString()
    };

    await dynamo.put({
      TableName: "SafeWebhookData",
      Item: safeData
    }).promise();

    return {
      statusCode: 200,
      body: JSON.stringify({ message: "Success" })
    };

  } catch (err) {
    console.error("Lambda error:", err);

    return {
      statusCode: 500,
      body: JSON.stringify({ message: "Internal error" })
    };
  }
};
```

---

### 3. Additional Security Tips

#### Environment Variables

If you're using API keys or tokens:

* Use **encrypted environment variables**.
* Turn on **“Enable helpers for encryption in transit”**.
* Set `KMS` key and use `process.env.SECRET_TOKEN` in your code.

#### Input Validation

Never trust the incoming event blindly:

* Always `JSON.parse` with error handling.
* Whitelist expected fields.
* Use libraries like `joi` or `zod` for structured validation (in Node).

#### API Gateway Source Validation (Optional)

Add a **custom header or HMAC token** to verify the sender:

```js
if (event.headers['x-api-key'] !== process.env.EXPECTED_KEY) {
  return { statusCode: 403, body: "Forbidden" };
}
```

#### Logging

* Use `console.log()` for visibility (but avoid logging sensitive data).
* Configure CloudWatch log retention.

#### Use Lambda Layers

* Separate dependencies into layers to reduce attack surface in your code bundle.

---

## Test Your Lambda Security

* **Use AWS IAM Access Analyzer** to review permissions.
* Simulate IAM access with `sts:SimulatePrincipalPolicy`.
* Scan your deployment package with tools like:

  * [AWS Lambda Powertools](https://awslabs.github.io/aws-lambda-powertools/)
  * [npm audit](https://docs.npmjs.com/cli/v9/commands/npm-audit)

---
