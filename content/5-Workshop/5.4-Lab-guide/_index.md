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

![Figure 1.1 – Bucket name and ACLs disabled](/fcj-workshop/images/5-workshop/1-1-s3-create-general.png)
*Figure 1.1 – Bucket name and ACLs disabled*

![Figure 1.2 – Block all public access, versioning disabled, SSE-S3 encryption](/fcj-workshop/images/5-workshop/1-2-s3-create-security.png)
*Figure 1.2 – Block all public access, versioning disabled, SSE-S3 encryption*

![Figure 1.3 – Lifecycle rule expiring uploads/ objects after 30 days](/fcj-workshop/images/5-workshop/1-3-s3-lifecycle.png)
*Figure 1.3 – Lifecycle rule expiring uploads/ objects after 30 days*

![Figure 1.4 – Permissions tab: Block public access On and HTTPS-only bucket policy](/fcj-workshop/images/5-workshop/1-4-s3-permissions.png)
*Figure 1.4 – Permissions tab: Block public access On and HTTPS-only bucket policy*

### 1.2. Create the DynamoDB Table
Open **DynamoDB → Create table**:
- Table name: `imageguard-images`
- Partition key: `userId` (`String`)
- Sort key: `imageId` (`String`)
- Capacity mode: **On-demand**

![Figure 1.5 – imageguard-images table: userId / imageId, On-demand mode](/fcj-workshop/images/5-workshop/2-1-dynamodb.png)
*Figure 1.5 – imageguard-images table: userId / imageId, On-demand mode*

### 1.3. Create the SNS Topic and SQS Dead-Letter Queue
- **SNS:** create a **Standard** topic named `imageguard-alerts` with **encryption disabled** (see [Troubleshooting](../5.5-testing/)). Add an Email subscription for the admin inbox and click **Confirm subscription** in the confirmation email.
- **SQS:** create a **Standard** queue named `imageguard-process-dlq` with **14-day message retention** and default SSE-SQS encryption.

![Figure 1.6 – Create Standard SNS topic imageguard-alerts](/fcj-workshop/images/5-workshop/3-1-sns-create-topic.png)
*Figure 1.6 – Create Standard SNS topic imageguard-alerts*

![Figure 1.7 – Subscription confirmation email](/fcj-workshop/images/5-workshop/3-2-sns-confirm-email.png)
*Figure 1.7 – Subscription confirmation email*

![Figure 1.8 – Subscription status Confirmed](/fcj-workshop/images/5-workshop/3-3-sns-confirmed.png)
*Figure 1.8 – Subscription status Confirmed*

![Figure 1.9 – Create Standard SQS queue imageguard-process-dlq with 14-day retention](/fcj-workshop/images/5-workshop/4-1-sqs-create.png)
*Figure 1.9 – Create Standard SQS queue imageguard-process-dlq with 14-day retention*

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

![Figure 1.10 – Trusted entity: AWS service Lambda](/fcj-workshop/images/5-workshop/5-1-iam-role-trusted.png)
*Figure 1.10 – Trusted entity: AWS service Lambda*

![Figure 1.11 – No managed policies attached](/fcj-workshop/images/5-workshop/5-2-iam-role-no-managed.png)
*Figure 1.11 – No managed policies attached*

![Figure 1.12 – JSON inline policy](/fcj-workshop/images/5-workshop/5-3-iam-inline-policy.png)
*Figure 1.12 – JSON inline policy*

![Figure 1.13 – Role summary showing a single inline policy](/fcj-workshop/images/5-workshop/5-4-iam-role-summary.png)
*Figure 1.13 – Role summary showing a single inline policy*

## Step 2 – Deploy `process-image` Lambda and S3 Trigger

### 2.1. Create and Configure the Function
Open **Lambda → Create function → Author from scratch**:
- Function name: `imageguard-process-image`
- Runtime: **Python 3.12** · Architecture: **arm64**
- **Additional settings → Custom execution role:** select `imageguard-process-image-role` · keep Function URL disabled.

![Figure 2.1 – Create Lambda: Python 3.12, ARM64, existing role, Function URL disabled](/fcj-workshop/images/5-workshop/6-1-lambda-create.png)
*Figure 2.1 – Create Lambda: Python 3.12, ARM64, existing role, Function URL disabled*

Paste [`process_image/lambda_function.py`](https://github.com/NhutTan-May/fcj-workshop/blob/main/static/code/imageguard/process_image/lambda_function.py) and click **Deploy**. Under **Configuration → General configuration**, set **Memory = 256 MB** and **Timeout = 30 seconds**, then define the environment variables:

![Figure 2.2 – Code pasted and deployed successfully](/fcj-workshop/images/5-workshop/7-1-lambda-code-deployed.png)
*Figure 2.2 – Code pasted and deployed successfully*

![Figure 2.3 – 256 MB memory and 30-second timeout](/fcj-workshop/images/5-workshop/8-1-lambda-general-config.png)
*Figure 2.3 – 256 MB memory and 30-second timeout*

| Environment Variable | Value |
|---|---|
| `TABLE_NAME` | `imageguard-images` |
| `BUCKET_NAME` | `imageguard-images-<ACCOUNT_ID>` |
| `TOPIC_ARN` | `arn:aws:sns:us-east-1:<ACCOUNT_ID>:imageguard-alerts` |
| `BLOCKED_CATEGORIES` | `Explicit,Violence,Visually Disturbing,Drugs & Tobacco,Hate Symbols` |
| `MIN_MODERATION_CONFIDENCE` | `60` |
| `MAX_LABELS` | `10` |
| `MIN_LABEL_CONFIDENCE` | `80` |

![Figure 2.4 – Seven environment variables on process-image](/fcj-workshop/images/5-workshop/9-1-lambda-env-vars.png)
*Figure 2.4 – Seven environment variables on process-image*

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

![Figure 2.5 – Asynchronous invocation: 1-hour max age, 2 retry attempts](/fcj-workshop/images/5-workshop/10-1-lambda-async-config.png)
*Figure 2.5 – Asynchronous invocation: 1-hour max age, 2 retry attempts*

![Figure 2.6 – On-failure destination routed to SQS DLQ](/fcj-workshop/images/5-workshop/10-2-lambda-destination.png)
*Figure 2.6 – On-failure destination routed to SQS DLQ*

![Figure 2.7 – S3 trigger: PUT events, prefix uploads/, suffix .jpg](/fcj-workshop/images/5-workshop/11-1-s3-trigger-config.png)
*Figure 2.7 – S3 trigger: PUT events, prefix uploads/, suffix .jpg*

![Figure 2.8 – Function overview: S3 → Lambda → SQS](/fcj-workshop/images/5-workshop/11-2-lambda-overview.png)
*Figure 2.8 – Function overview: S3 → Lambda → SQS*

![Figure 2.9 – Resource-based policy with aws:SourceArn and aws:SourceAccount (confused-deputy protection)](/fcj-workshop/images/5-workshop/11-3-lambda-resource-policy.png)
*Figure 2.9 – Resource-based policy with aws:SourceArn and aws:SourceAccount (confused-deputy protection)*

### 2.4. Quick Pipeline Verification (Before Building the Frontend)
1. Under **DynamoDB → Explore items → Create item**, create two `PENDING` items (`userId = test-user-001`, `imageId = img-001` and `img-002`, with String attribute `status = PENDING`).
2. In **S3**, create folder `uploads/test-user-001/` and upload the beer image as `img-001.jpg` and the toy gun image as `img-002.jpg`.
3. Wait ~10 seconds and click **Explore items → Run** to verify updated statuses and labels.

![Figure 2.10 – Test items initialized in PENDING state](/fcj-workshop/images/5-workshop/12-1-dynamodb-pending.png)
*Figure 2.10 – Test items initialized in PENDING state*

![Figure 2.11 – Sample images uploaded under uploads/test-user-001/](/fcj-workshop/images/5-workshop/13-1-s3-uploaded.png)
*Figure 2.11 – Sample images uploaded under uploads/test-user-001/*

![Figure 2.12 – Results: img-001 APPROVED, img-002 REJECTED](/fcj-workshop/images/5-workshop/14-1-dynamodb-result.png)
*Figure 2.12 – Results: img-001 APPROVED, img-002 REJECTED*

![Figure 2.13 – Approved image labels in DynamoDB](/fcj-workshop/images/5-workshop/14-2-item-approved-labels.png)
*Figure 2.13 – Approved image labels in DynamoDB*

![Figure 2.14 – Rejected image with blockedCategories = Violence](/fcj-workshop/images/5-workshop/14-3-item-rejected.png)
*Figure 2.14 – Rejected image with blockedCategories = Violence*

![Figure 2.15 – Alert email delivered to the administrator](/fcj-workshop/images/5-workshop/14-4-sns-alert-email.png)
*Figure 2.15 – Alert email delivered to the administrator*

![Figure 2.16 – image_processed entry in CloudWatch Logs](/fcj-workshop/images/5-workshop/14-5-cloudwatch-log.png)
*Figure 2.16 – image_processed entry in CloudWatch Logs*

### 2.5. Verify Dead-Letter Queue (Intentional Failure Test)
1. Temporarily point `TABLE_NAME` to a non-existent table name (e.g., `khong-ton-tai`).
2. Open the **Test** tab, set Invocation type to **Asynchronous**, and trigger with a sample S3 Put event pointing to an existing image key.
3. Wait ~3 minutes (1 initial run + 2 retries) and inspect **SQS → `imageguard-process-dlq` → Poll for messages**.
4. Restore `TABLE_NAME` back to `imageguard-images`.

![Figure 2.17 – Temporarily override TABLE_NAME to simulate a failure](/fcj-workshop/images/5-workshop/15-0-env-table-name-wrong.jpg)
*Figure 2.17 – Temporarily override TABLE_NAME to simulate a failure*

![Figure 2.18 – Asynchronous invocation from the Lambda Test tab](/fcj-workshop/images/5-workshop/15-1-lambda-async-invoke.jpg)
*Figure 2.18 – Asynchronous invocation from the Lambda Test tab*

![Figure 2.19 – One message delivered to the DLQ after retries are exhausted](/fcj-workshop/images/5-workshop/15-2a-dlq-messages-list.jpg)
*Figure 2.19 – One message delivered to the DLQ after retries are exhausted*

![Figure 2.20 – Restore TABLE_NAME to its valid value](/fcj-workshop/images/5-workshop/15-3-env-restored.jpg)
*Figure 2.20 – Restore TABLE_NAME to its valid value*

See [Section 5.5](../5.5-testing/) for the captured DLQ message payload.

## Step 3 – Set Up Cognito, API Gateway, and Static Web App

You can configure Step 3 manually in the Console as outlined below, or run [`setup.sh`](https://github.com/NhutTan-May/fcj-workshop/blob/main/static/code/imageguard/scripts/setup.sh) inside **AWS CloudShell** to provision the entire stack via AWS CLI in ~3 minutes.

![Figure 3.1 – Output of setup.sh in CloudShell](/fcj-workshop/images/5-workshop/23-0-cloudshell-setup-output.jpg)
*Figure 3.1 – Output of setup.sh in CloudShell*

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

> **Why OAC and `SecurityHeadersPolicy`?** Keeping *Block all public access* enabled guarantees only this CloudFront distribution can read static files, denying direct S3 URL access (`403`, verified in Section 5.5). `SecurityHeadersPolicy` attaches HSTS, X-Frame-Options, and X-Content-Type-Options against clickjacking and MIME sniffing. AWS WAF is omitted in the lab environment and noted as a production enhancement.

![Figure 3.2 – ImageGuard web app served via CloudFront](/fcj-workshop/images/5-workshop/21-6-web-home-dark.jpg)
*Figure 3.2 – ImageGuard web app served via CloudFront*

### 3.3. Configure Amazon Cognito
Create User Pool `imageguard-users` with **email** sign-in and self-registration enabled. Add a **Public SPA App Client (no client secret)** configured with:
- Grant type: **Authorization code grant**
- Scopes: `openid`, `email`
- Callback and Sign-out URL: `https://<CLOUDFRONT_DOMAIN>/`
- Provision a Cognito Domain to enable **Managed Login**.

> **Why use PKCE without a Client Secret?** Browser-based SPAs cannot safely conceal a secret inside client-side JavaScript. **PKCE** replaces the secret with a per-login random `code_verifier` and its SHA-256 hash, rendering an intercepted authorization `code` useless, while `state` mitigates CSRF and tokens stay in `sessionStorage` for the active tab.

![Figure 3.3 – Cognito Managed Login screen](/fcj-workshop/images/5-workshop/26-0-cognito-managed-login.jpg)
*Figure 3.3 – Cognito Managed Login screen*

### 3.4. Configure API Gateway HTTP API
- Create HTTP API `imageguard-api` with an auto-deploying `$default` stage and routes `POST /upload-url` → `presign-upload` and `GET /images` → `get-images`.
- Attach a **JWT Authorizer** (`cognito-jwt`) with Issuer `https://cognito-idp.us-east-1.amazonaws.com/<USER_POOL_ID>` and Audience `<APP_CLIENT_ID>` to both routes.
- Restrict **CORS** to origin `https://<CLOUDFRONT_DOMAIN>`, headers `authorization, content-type`, and methods `GET, POST, OPTIONS`.
- Set default-route **throttling** to `rate = 10` and `burst = 20`.

> **Why configure API Gateway this way?** The JWT Authorizer verifies signatures against the User Pool public keys alongside `issuer` and `audience`, blocking forged or expired requests at the gateway (`401`) before Lambda runs or incurs charges (verified in tests E3 and E4 in Section 5.5). Restricting CORS to the CloudFront origin and throttling requests protects downstream compute and spend.

### 3.5. Connect All Components
- Add a **CORS** rule on `imageguard-images-<ACCOUNT_ID>` permitting `PUT, GET` from your CloudFront domain.
- Populate `config.js` with your Cognito domain, App Client ID, redirect URI, and API Gateway URL, re-upload it to S3, and run a CloudFront invalidation for `/*`.

![Figure 3.4 – End result: statistics, multi-image upload, status filters, and label search](/fcj-workshop/images/5-workshop/26-5-web-dashboard-dark.jpg)
*Figure 3.4 – End result: statistics, multi-image upload, status filters, and label search*

![Figure 3.5 – Image detail modal: labels, upload/processed timestamps, Image ID](/fcj-workshop/images/5-workshop/26-7-web-detail-dark.jpg)
*Figure 3.5 – Image detail modal: labels, upload/processed timestamps, Image ID*

> **Client-side UI capabilities:** Status filtering, client-side label search, sorting, popular tags, summary statistics, detail modal, multi-file upload, JPEG magic-byte verification, and VI/EN language switching (`?lang=en`) all execute in the browser on the `GET /images` payload without modifying the backend architecture.

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

> **Rationale for Alarms & 5 EMF Metrics:** Each alarm covers a distinct failure mode (Lambda errors, concurrency throttling, DLQ buildup, SNS notification failures tracked via `AlertPublishFailed`, and API 5xx spikes). Emitting `ImagesProcessed`, `ImagesApproved`, `ImagesRejected`, `ProcessingLatency`, and `AlertPublishFailed` via EMF JSON log lines avoids granting `cloudwatch:PutMetricData` or incurring extra API calls, while **OK** notifications confirm automatic recovery.

![Figure 4.1 – Seven alarms and dashboard provisioned via step4.sh](/fcj-workshop/images/5-workshop/28-1-step4-alarms-created.jpg)
*Figure 4.1 – Seven alarms and dashboard provisioned via step4.sh*

![Figure 4.2 – Alarm states: DLQ alarm in ALARM (test message present), all others OK](/fcj-workshop/images/5-workshop/28-2-step4-alarm-states.jpg)
*Figure 4.2 – Alarm states: DLQ alarm in ALARM (test message present), all others OK*

![Figure 4.3 – Custom EMF metrics under the ImageGuard namespace](/fcj-workshop/images/5-workshop/14-6-cloudwatch-metrics.png)
*Figure 4.3 – Custom EMF metrics under the ImageGuard namespace*

![Figure 4.4 – Log groups configured with 14-day retention](/fcj-workshop/images/5-workshop/14-7-log-retention.png)
*Figure 4.4 – Log groups configured with 14-day retention*

![Figure 4.5 – imageguard CloudWatch dashboard](/fcj-workshop/images/5-workshop/28-4-dashboard.png)
*Figure 4.5 – imageguard CloudWatch dashboard*
