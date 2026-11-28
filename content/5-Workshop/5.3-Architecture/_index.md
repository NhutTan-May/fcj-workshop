---
title: "Architecture"
date: 2026-11-20
weight: 3
chapter: false
pre: " <b> 5.3. </b> "
---

![ImageGuard Architecture Diagram](/fcj-workshop/images/2-proposal/imageguard-architecture.drawio.png)

The diagram is organized into five logical groups: **Frontend**, **Authentication & API**, **Compute**, **Storage & AI**, and **Notification & Monitoring**. Amazon CloudFront sits outside the AWS Region box because it is a global edge service. Arrow colors distinguish data flows: **blue** (image upload), **green** (image retrieval), **orange** (AI analysis), **dashed grey** (monitoring), and **dashed red** (IAM permissions), with every arrow drawn in the **request** direction.

## End-to-End Request Flow (Matches Diagram Numbers)

| # | Source → Destination | Technical Description |
|---|---|---|
| 1 | User → CloudFront → S3 (`imageguard-web`) | Browser loads static assets (HTML/JS) over HTTPS; CloudFront reads the private web bucket via **Origin Access Control (OAC)** |
| 2 | User → Cognito | Signs in via **Cognito Managed Login** (OAuth 2.0 Authorization Code + PKCE) and receives JWTs (ID + access tokens) |
| 3 | User → API Gateway | Calls `POST /upload-url` with the access token; the **JWT Authorizer** validates signature, issuer, and audience |
| 3a | API Gateway → `presign-upload` | Extracts `userId` from the `sub` claim and generates a UUIDv4 `imageId` |
| 3b | `presign-upload` → DynamoDB | Creates a `PENDING` item and returns a 5-minute Presigned PUT URL for `uploads/{sub}/{imageId}.jpg` |
| 4 | User → S3 (`imageguard-images`) | Browser `PUT`s the image directly to S3 via the Presigned URL, bypassing API Gateway and Lambda |
| 5 | S3 → `process-image` | **S3 Event** `ObjectCreated:Put` (prefix `uploads/`, suffix `.jpg`) triggers `process-image` **asynchronously** |
| 6 | `process-image` → Rekognition | Invokes `DetectModerationLabels` (≥ 60% threshold); only clean images continue to `DetectLabels` (up to 10 labels, ≥ 80%) |
| 6a | Rekognition → S3 | Rekognition reads image objects directly from S3 (`S3Object`) without Lambda downloading image bytes |
| 7 | `process-image` → DynamoDB | Runs a **conditional** `UpdateItem` (`status = PENDING`) transitioning the item to `APPROVED` or `REJECTED` |
| 8 | `process-image` → SNS → Admin | Publishes an email alert only when marked `REJECTED` **and** the DynamoDB update succeeded |
| 9 | `process-image` → SQS DLQ | Events still failing after 2 async retries are routed to the Dead-Letter Queue (14-day retention) |
| 10 | User → API Gateway | Calls `GET /images` with the access token (shares arrow with step 3) |
| 10a | API Gateway → `get-images` | Extracts `userId` from the `sub` claim |
| 10b | `get-images` → DynamoDB | Executes `Query` by `userId`, sorted newest-first (max 50 items), and signs 5-minute Presigned GET URLs solely for `APPROVED` items |
| 10c | User → S3 (`imageguard-images`) | Browser fetches approved photos directly from S3 using the Presigned GET URL |
| 11 | Lambdas → CloudWatch | Emits structured JSON logs and 5 custom metrics via Embedded Metric Format (EMF) |
| 12 | CloudWatch → SNS | 7 alarms send email alerts on `ALARM` transitions and upon returning to `OK` |
| – | IAM → Lambdas | Assigns a dedicated least-privilege IAM execution role to each Lambda function |

## Service Selection & Alternatives Considered

| AWS Service | System Role | Selection Rationale | Alternatives Considered |
|---|---|---|---|
| **Amazon S3** | Image & static web storage | 99.999999999% durability, low cost, native S3 Events, Lifecycle expiration | EBS/EFS: requires attached compute instances and higher baseline cost |
| **AWS Lambda** | Runs 3 backend functions | Zero server management, automatic concurrency scaling, pay-per-use billing | EC2/ECS: incurs 24/7 compute charges even during zero traffic |
| **Amazon Rekognition** | Moderation & labeling | Pre-trained content moderation models requiring zero training data, per-image pricing | Amazon Bedrock: zero quota on new accounts (Section 5.5); custom models: require labeled data and GPUs |
| **Amazon DynamoDB** | Status & label storage | Serverless On-demand billing, fast `userId` key queries, conditional updates | Amazon RDS: runs 24/7 instances and requires VPC networking |
| **Amazon API Gateway (HTTP API)** | Web API layer | Native Cognito JWT Authorizer, CORS, throttling; ~70% cheaper than REST API | REST API: unnecessary extra features; Function URL: lacks native JWT Authorizer |
| **Amazon Cognito** | Sign-up & authentication | Managed Login UI, OAuth 2.0 + PKCE, avoids storing credentials in app code | Custom Auth: higher security risk and maintenance burden |
| **Amazon CloudFront** | Static web delivery | Built-in HTTPS, edge caching close to users, OAC keeps S3 buckets private | S3 static website hosting: HTTP-only and requires public bucket access |
| **Amazon SNS** | Alert notifications | Simple email delivery in a few lines of code, shared by Lambda alerts and CloudWatch Alarms | Amazon SES: better for high-volume per-user emails, requires sender/domain verification |
| **Amazon SQS** | Dead-Letter Queue | Retains failing events for 14 days for inspection and replay without data loss | No DLQ: async Lambda failures are dropped once retries exhaust |
| **Amazon CloudWatch** | Logs, metrics, alarms, dashboard | Native Lambda logging destination; EMF generates metrics directly from logs with zero extra API calls | Third-party APM: adds setup overhead and external credential management |
| **AWS IAM** | Access governance | Isolates each Lambda with minimal resource-scoped permissions | Single shared role: compromising one function exposes permissions of all three |

## Data Model (DynamoDB `imageguard-images`)

| Attribute | Type | Purpose |
|---|---|---|
| `userId` | String – **Partition key** | Cognito `sub` claim; isolates each user's partition so `Query` returns only their own uploads |
| `imageId` | String – **Sort key** | UUIDv4 generated by `presign-upload`, matching the S3 object filename |
| `status` | String | Processing lifecycle: `PENDING` → `APPROVED` or `REJECTED` |
| `createdAt`, `processedAt` | String (ISO 8601) | Upload URL creation timestamp and AI completion timestamp |
| `moderationLabels` | List | Rekognition moderation labels (name, parent category, confidence) |
| `labels` | List | Content tags (name, confidence), populated only for approved images |
| `blockedCategories` | List | Top-level categories triggering rejection (e.g., `Violence`) |
| `reason` | String | Rejection reason code: `BLOCKED_CATEGORY`, `TOO_LARGE` (> 5 MB), or `INVALID_IMAGE:<error>` |

No Global Secondary Index (GSI) is required because every query filters directly on `userId` (the partition key).

## Moderation Policy

- **Blocked top-level categories:** `Explicit`, `Violence`, `Visually Disturbing`, `Drugs & Tobacco`, and `Hate Symbols` — stored in the `BLOCKED_CATEGORIES` environment variable so rules can be updated without redeploying code.
- **Permitted monitored category:** `Alcohol` is allowed in the gallery while its moderation metadata remains recorded in DynamoDB for auditing.
- **Cost-aware ordering:** Moderation always runs before labeling; blocked uploads skip `DetectLabels` entirely.

## Security Design

| Layer | ImageGuard Implementation |
|---|---|
| **Least Privilege** | Each Lambda receives its own custom inline IAM role rather than broad managed policies such as `AWSLambdaBasicExecutionRole` |
| **Private Storage** | Both S3 buckets enforce **Block all public access**, disable ACLs, apply default SSE-S3 encryption, and enforce HTTPS via bucket policy |
| **Identity & Tenant Isolation** | The API Gateway JWT Authorizer validates the Cognito `issuer` and `audience`; `userId` is always extracted from the verified `sub` claim, never client input |
| **Controlled Uploads** | Backend-signed Presigned PUT URLs restrict target key, enforce `Content-Type: image/jpeg`, and expire after 5 minutes |
| **Confused Deputy Defense** | Lambda resource-based policies lock triggers using `aws:SourceArn` and `aws:SourceAccount` |
| **Recursive Loop Prevention** | S3 triggers match only the `uploads/` prefix, and `process-image` holds zero S3 write permissions |
| **Idempotency** | Conditional `UpdateItem` (`status = PENDING`) guarantees duplicate S3 events or overwrites cannot alter finalized verdicts |
| **Web Security** | CloudFront enforces HTTPS via OAC, attaches `SecurityHeadersPolicy` (HSTS, X-Frame-Options, etc.), and frontend JS renders text via `textContent` to mitigate XSS |

### Per-Lambda Permission Matrix

| Lambda Function | Allowed Actions | Denied by Omission |
|---|---|---|
| `imageguard-process-image` | `s3:GetObject` (`uploads/*`), 2 Rekognition actions, `dynamodb:UpdateItem` (1 table), `sns:Publish` (1 topic), `sqs:SendMessage` (1 DLQ), own log group | `s3:PutObject`, `dynamodb:Query`, `dynamodb:PutItem` |
| `imageguard-presign-upload` | `s3:PutObject` (`uploads/*`), `dynamodb:PutItem` (1 table), own log group | `s3:GetObject`, `dynamodb:Query`, `dynamodb:UpdateItem` |
| `imageguard-get-images` | `dynamodb:Query` (1 table), `s3:GetObject` (`uploads/*`), own log group | `s3:PutObject`, `dynamodb:PutItem`, `dynamodb:UpdateItem` |

> **Note on Rekognition:** `DetectModerationLabels` and `DetectLabels` do not support resource-level ARN scoping, so `"Resource": "*"` is required. To preserve least privilege, only those two exact actions are permitted—never `rekognition:*`.

## AWS Well-Architected Framework Mapping

| Pillar | Measures Applied in ImageGuard |
|---|---|
| **Operational Excellence** | Structured JSON logs; 5 EMF custom metrics (`ImagesProcessed`, `ImagesApproved`, `ImagesRejected`, `ProcessingLatency`, `AlertPublishFailed`); 7 alarms and the `imageguard` dashboard; reproducible `setup.sh` and `step4.sh` scripts; moderation thresholds externalized in environment variables |
| **Security** | Full defense-in-depth under *Security Design*: per-Lambda Least Privilege, private buckets, JWT Authorizer, 5-minute Presigned URLs, OAC, confused-deputy protection, zero hard-coded secrets |
| **Reliability** | Multi-AZ managed services (S3, Lambda, DynamoDB, SNS, SQS); 2 async Lambda retries before DLQ routing; 5 `boto3` retries on throttling; idempotent conditional writes; corrupt files marked `REJECTED` gracefully; 35-day DynamoDB PITR |
| **Performance Efficiency** | Direct browser-to-S3 uploads keep API and Lambda payloads small; Rekognition reads directly from S3; `arm64` Lambdas (`process-image` 256 MB, API Lambdas 128 MB); auto-scaling with On-demand DynamoDB and CloudFront edge caching |
| **Cost Optimization** | 100% serverless with zero 24/7 idle compute; screen before labeling; HTTP API over REST API; `arm64`; 30-day S3 Lifecycle; 14-day log retention; USD 8.37/month Pricing Calculator estimate |
| **Sustainability** | Compute runs strictly on demand when images arrive, while S3 Lifecycle automatically purges expired objects |

## Key Architectural Decisions

- **Direct-to-S3 uploads vs. proxying through API Gateway:** Synchronous Lambda invocations cap payloads at 6 MB (exceeded by base64-encoded 5 MB images), and HTTP API meters every 512 KB chunk as a billable request. Presigned PUT URLs restrict browsers to one server-chosen key for 5 minutes—both cheaper and safer.
- **Writing `PENDING` before signing the upload URL:** Ensures `process-image` processes only API-authorized uploads; stray S3 objects lacking a `PENDING` record are ignored, while the UI can show `"Processing"` immediately.
- **Async Lambda + DLQ vs. an SQS queue between S3 and Lambda:** At workshop scale, async Lambda's built-in 2 retries plus an SQS DLQ prevent lost events with fewer moving parts; inserting a buffering SQS queue remains a clean future step for spiky workloads.
- **Publishing SNS alerts only after DynamoDB confirms the update:** Publishing before the conditional write previously caused duplicate/spurious emails when DB updates failed or raced (caught and fixed in Section 5.5).
- **Issuing viewing URLs exclusively for `APPROVED` items:** Blocked photos are never exposed back to the browser, and admin emails contain only `s3Key` rather than image links.

## Scalability & Limits

- **Elastic scaling:** Lambda spins up concurrent executions automatically during upload spikes, while S3 and On-demand DynamoDB scale without capacity pre-provisioning.
- **Operational quotas:** Account-level Rekognition transactions-per-second and Lambda concurrency caps. Exceeding quotas triggers `boto3` retries, Lambda async retries, and eventual DLQ capture—surfaced via the `throttles` and `dlq-not-empty` alarms.
- **API protection:** Route throttling (`10 req/s`, burst `20`) guards backend concurrency and spend against abusive traffic.
- **Growth roadmap:** Insert an SQS buffer between S3 and Lambda for burst smoothing, enable AWS WAF on CloudFront, and migrate CLI scripts to AWS SAM/CloudFormation.

## Data Backup & Recovery

The `imageguard-images` DynamoDB table enables **Point-in-time recovery (PITR)**, allowing second-level restoration to any moment within the preceding **35 days** in case of accidental deletion or bad writes. Paired with Amazon S3's **99.999999999%** durability for source image objects, both metadata and raw assets can be restored consistently.

```bash
aws dynamodb update-continuous-backups --table-name imageguard-images \
  --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true
# Expected output: ENABLED
```

## Why Choose Region `us-east-1`?

- **Lowest baseline pricing:** `us-east-1` (N. Virginia) typically offers the lowest rates across AWS services—including **Amazon Rekognition**, which drives most of the monthly bill (USD 5.91 out of USD 8.37/month; see Section 5.6).
- **Full service availability:** Every component required by the workshop (Rekognition Content Moderation, Cognito Managed Login, HTTP API, etc.) is immediately available in `us-east-1`.
- **Network latency is not the bottleneck:** Although round-trip latency from Vietnam to `us-east-1` is roughly 200 ms, static assets are cached at CloudFront edge nodes close to users, while backend latency is dominated by AI inference (~0.5–1 second).
- **Production rollout path for Vietnam:** Moving to Singapore (`ap-southeast-1`) for lower latency and regional data residency requires only updating environment variables—no code changes needed.

## Why Lambda Is Not Placed Inside a VPC

ImageGuard has no private RDS database or internal server requiring private subnet isolation. Every AWS API call (S3, DynamoDB, Rekognition, SNS) runs over HTTPS with IAM SigV4 authentication. Moving Lambdas into a VPC would introduce NAT Gateway or Interface VPC Endpoint charges (tens of USD per month) without meaningful security gain for this serverless pattern.
