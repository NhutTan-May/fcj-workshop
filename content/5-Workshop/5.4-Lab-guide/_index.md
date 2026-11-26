---
title: "Step-by-Step Lab Guide"
date: 2026-11-20
weight: 4
chapter: false
pre: " <b> 5.4. </b> "
---

Complete source code (all 3 Lambdas, static frontend, and automation scripts): [static/code/imageguard](https://github.com/NhutTan-May/fcj-workshop/tree/main/static/code/imageguard). Throughout this lab, replace `<ACCOUNT_ID>` with your 12-digit AWS Account ID.

## Step 1 – Provision Foundation Resources

### 1.1. Create the Image S3 Bucket
Open **S3 → Create bucket** and configure:
- Bucket name: `imageguard-images-<ACCOUNT_ID>`
- **Object Ownership:** ACLs disabled
- **Block Public Access:** enable **Block all public access**
- **Versioning:** Disabled · **Default encryption:** SSE-S3

After creation, apply two additional settings:
- **Management → Lifecycle rule**: create `delete-uploads-30d` scoped to prefix `uploads/`, expiring objects after **30 days**.
- **Permissions → Bucket policy**: deny any request where `aws:SecureTransport` is `false` (enforce HTTPS).

![S3 Block public access and bucket policy](/fcj-workshop/images/5-workshop/1-4-s3-permissions.png)

### 1.2. Create the DynamoDB Table
Open **DynamoDB → Create table**:
- Table name: `imageguard-images`
- Partition key: `userId` (`String`)
- Sort key: `imageId` (`String`)
- Capacity mode: **On-demand**

### 1.3. Create the SNS Topic and SQS Dead-Letter Queue
- **SNS:** create a **Standard** topic named `imageguard-alerts` with **encryption disabled** (see [Troubleshooting](../5.5-testing/)). Add an Email subscription for the admin inbox and click **Confirm subscription** in the confirmation email.
- **SQS:** create a **Standard** queue named `imageguard-process-dlq` with **14-day message retention** and default SSE-SQS encryption.

### 1.4. Create the IAM Role for `process-image`
Open **IAM → Roles → Create role**:
- Trusted entity: **AWS service → Lambda**
- Do **not** attach any AWS managed policy.
- Role name: `imageguard-process-image-role`.
- Open **Add permissions → Create inline policy** (JSON editor) and paste:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadUploadedImages",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::imageguard-images-<ACCOUNT_ID>/uploads/*"
    },
    {
      "Sid": "AnalyzeImagesWithRekognition",
      "Effect": "Allow",
      "Action": [
        "rekognition:DetectModerationLabels",
        "rekognition:DetectLabels"
      ],
      "Resource": "*"
    },
    {
      "Sid": "UpdateImageResults",
      "Effect": "Allow",
      "Action": "dynamodb:UpdateItem",
      "Resource": "arn:aws:dynamodb:us-east-1:<ACCOUNT_ID>:table/imageguard-images"
    },
    {
      "Sid": "SendModerationAlerts",
      "Effect": "Allow",
      "Action": "sns:Publish",
      "Resource": "arn:aws:sns:us-east-1:<ACCOUNT_ID>:imageguard-alerts"
    },
    {
      "Sid": "SendFailedEventsToDLQ",
      "Effect": "Allow",
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:us-east-1:<ACCOUNT_ID>:imageguard-process-dlq"
    },
    {
      "Sid": "WriteOwnLogsOnly",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": [
        "arn:aws:logs:us-east-1:<ACCOUNT_ID>:log-group:/aws/lambda/imageguard-process-image",
        "arn:aws:logs:us-east-1:<ACCOUNT_ID>:log-group:/aws/lambda/imageguard-process-image:*"
      ]
    }
  ]
}
```

Verify that the Review screen lists **6 services**. If IAM reports *"The policy failed legacy parsing"*, inspect `<ACCOUNT_ID>` formatting (Section 5.5).

![The role has a single inline policy](/fcj-workshop/images/5-workshop/5-4-iam-role-summary.png)

## Step 2 – Deploy `process-image` Lambda and S3 Trigger

### 2.1. Create and Configure the Function
Open **Lambda → Create function → Author from scratch**:
- Function name: `imageguard-process-image`
- Runtime: **Python 3.12** · Architecture: **arm64**
- **Additional settings → Custom execution role:** select `imageguard-process-image-role` · keep Function URL disabled.

![Creating the process-image Lambda](/fcj-workshop/images/5-workshop/6-1-lambda-create.png)

Paste [`process_image/lambda_function.py`](https://github.com/NhutTan-May/fcj-workshop/blob/main/static/code/imageguard/process_image/lambda_function.py) and click **Deploy**. Under **Configuration → General configuration**, set **Memory = 256 MB** and **Timeout = 30 seconds**, then define the environment variables:

| Environment Variable | Value |
|---|---|
| `TABLE_NAME` | `imageguard-images` |
| `BUCKET_NAME` | `imageguard-images-<ACCOUNT_ID>` |
| `TOPIC_ARN` | `arn:aws:sns:us-east-1:<ACCOUNT_ID>:imageguard-alerts` |
| `BLOCKED_CATEGORIES` | `Explicit,Violence,Visually Disturbing,Drugs & Tobacco,Hate Symbols` |
| `MIN_MODERATION_CONFIDENCE` | `60` |
| `MAX_LABELS` | `10` |
| `MIN_LABEL_CONFIDENCE` | `80` |

### 2.2. Key Implementation Details

```python
# Exponential-backoff retries for throttling / 5xx errors
BOTO_CONFIG = Config(retries={"max_attempts": 5, "mode": "standard"})

# Enforce strict uploads/{userId}/{imageId}.{ext} pattern against path traversal
KEY_PATTERN = re.compile(r"^uploads/([A-Za-z0-9-]{1,64})/([A-Za-z0-9-]{1,64})\.(jpg|jpeg|png)$")

# Conditional UpdateItem: ensures idempotency and ignores untracked uploads
ConditionExpression="attribute_exists(#uid) AND #st = :pending"

# Send SNS alert only after DynamoDB confirms REJECTED update
if blocked and updated:
    publish_alert(user_id, image_id, key, blocked)
```

Custom metrics are emitted using **CloudWatch Embedded Metric Format (EMF)**: printing a structured JSON payload containing `_aws` to stdout automatically publishes `ImagesProcessed`, `ImagesApproved`, `ImagesRejected`, `ProcessingLatency`, and `AlertPublishFailed` under the `ImageGuard` namespace—requiring zero `cloudwatch:PutMetricData` permissions.

### 2.3. Configure Async Retries, DLQ, and S3 Event Trigger
- Under **Configuration → Asynchronous invocation**: set Maximum age of event = **1 hour**, Retry attempts = **2**.
- Under **Destinations → Add destination**: choose **On failure** → **SQS queue** `imageguard-process-dlq`.
- Click **Add trigger → S3**: select `imageguard-images-<ACCOUNT_ID>`, Event type **PUT**, Prefix `uploads/`, Suffix `.jpg`.

![Function overview: S3 → Lambda → SQS](/fcj-workshop/images/5-workshop/11-2-lambda-overview.png)

## Step 3 – Set Up Cognito, API Gateway, and Static Web App

You can configure Step 3 manually in the Console as outlined below, or run [`setup.sh`](https://github.com/NhutTan-May/fcj-workshop/blob/main/static/code/imageguard/scripts/setup.sh) inside **AWS CloudShell** to provision the entire stack via AWS CLI in ~3 minutes.

![Output of setup.sh in CloudShell](/fcj-workshop/images/5-workshop/23-0-cloudshell-setup-output.jpg)

### 3.1. Create the Two API Lambdas
Create two dedicated least-privilege IAM roles, then deploy two **Python 3.12 (arm64)** Lambda functions configured with **128 MB RAM** and **10-second timeout**:

| Lambda Function | Inline Policy Permissions | Environment Variables |
|---|---|---|
| `imageguard-presign-upload` | `s3:PutObject` (`uploads/*`), `dynamodb:PutItem`, own logs | `BUCKET_NAME`, `TABLE_NAME`, `URL_EXPIRES_SECONDS=300` |
| `imageguard-get-images` | `dynamodb:Query`, `s3:GetObject` (`uploads/*`), own logs | `BUCKET_NAME`, `TABLE_NAME`, `URL_EXPIRES_SECONDS=300`, `MAX_ITEMS=50` |

`presign-upload` reads `userId` from `requestContext.authorizer.jwt.claims.sub`, generates a UUID `imageId`, writes a `PENDING` record to DynamoDB, and signs a Presigned PUT URL. Because presigned URLs inherit the signing Lambda's permissions, its role must hold `s3:PutObject`.

### 3.2. Host the Static Frontend on S3 + CloudFront (OAC)
1. Create private bucket `imageguard-web-<ACCOUNT_ID>` (Block Public Access enabled, SSE-S3) and upload `index.html`, `app.js`, and `config.js` from [`static/code/imageguard/web`](https://github.com/NhutTan-May/fcj-workshop/tree/main/static/code/imageguard/web).
2. Under **CloudFront → Create distribution**, select `imageguard-web-<ACCOUNT_ID>` with **OAC**, **Redirect HTTP to HTTPS**, `CachingOptimized`, `SecurityHeadersPolicy`, default root object `index.html`, and WAF disabled.
3. Update the web bucket policy to grant read access exclusively to `cloudfront.amazonaws.com` scoped by `AWS:SourceArn`.

![ImageGuard served through CloudFront](/fcj-workshop/images/5-workshop/21-4-web-first-load.jpg)

### 3.3. Configure Amazon Cognito
Create User Pool `imageguard-users` with **email** sign-in and self-registration enabled. Add a **Public SPA App Client (no client secret)** configured with:
- Grant type: **Authorization code grant**
- Scopes: `openid`, `email`
- Callback and Sign-out URL: `https://<CLOUDFRONT_DOMAIN>/`
- Provision a Cognito Domain to enable **Managed Login**.

![Cognito Managed Login page](/fcj-workshop/images/5-workshop/26-0-cognito-managed-login.jpg)

### 3.4. Configure API Gateway HTTP API
- Create HTTP API `imageguard-api` with an auto-deploying `$default` stage and routes `POST /upload-url` → `presign-upload` and `GET /images` → `get-images`.
- Attach a **JWT Authorizer** (`cognito-jwt`) with Issuer `https://cognito-idp.us-east-1.amazonaws.com/<USER_POOL_ID>` and Audience `<APP_CLIENT_ID>` to both routes.
- Restrict **CORS** to origin `https://<CLOUDFRONT_DOMAIN>`, headers `authorization, content-type`, and methods `GET, POST, OPTIONS`.
- Set default-route **throttling** to `rate = 10` and `burst = 20`.

### 3.5. Connect All Components
- Add a **CORS** rule on `imageguard-images-<ACCOUNT_ID>` permitting `PUT, GET` from your CloudFront domain.
- Populate `config.js` with your Cognito domain, App Client ID, redirect URI, and API Gateway URL, re-upload it to S3, and run a CloudFront invalidation for `/*`.

## Step 4 – Configure CloudWatch Observability

Run [`step4.sh`](https://github.com/NhutTan-May/fcj-workshop/blob/main/static/code/imageguard/scripts/step4.sh) in CloudShell, or configure these 7 alarms manually in the CloudWatch Console:

| Alarm Name | Metric | Threshold (5-minute period) |
|---|---|---|
| `imageguard-{process-image,presign-upload,get-images}-errors` | `AWS/Lambda` → `Errors` | ≥ 1 |
| `imageguard-process-image-throttles` | `AWS/Lambda` → `Throttles` | ≥ 1 |
| `imageguard-dlq-not-empty` | `AWS/SQS` → `ApproximateNumberOfMessagesVisible` | ≥ 1 |
| `imageguard-alert-publish-failed` | `ImageGuard` → `AlertPublishFailed` | ≥ 1 |
| `imageguard-api-5xx` | `AWS/ApiGateway` → `5xx` | ≥ 5 |

All alarms notify `imageguard-alerts` on both **ALARM** and **OK** state transitions. The `imageguard` CloudWatch Dashboard surfaces `APPROVED`/`REJECTED` counts, processing latency, Lambda errors, API 5xx counts, DLQ depth, and alarm states. Finally, enforce **14-day log retention** on all three Lambda log groups.

![imageguard CloudWatch dashboard](/fcj-workshop/images/5-workshop/28-4-dashboard.png)
