---
title: "Architecture"
date: 2026-11-20
weight: 3
chapter: false
pre: " <b> 5.3. </b> "
---

![ImageGuard Architecture Diagram](/fcj-workshop/images/2-proposal/imageguard-architecture.drawio.png)

## End-to-End Request Flow

| Step | Flow Path | Technical Description |
|---|---|---|
| 1 | User → CloudFront → S3 (`imageguard-web`) | Users load the web app over HTTPS; CloudFront fetches static files from the private S3 bucket via **Origin Access Control (OAC)** |
| 2 | User ↔ Cognito | Users sign in via **Cognito Managed Login** (OAuth 2.0 Authorization Code + PKCE) and obtain a **JWT token** |
| 3 | User → API Gateway → Lambda `presign-upload` | The frontend calls `POST /upload-url` with the JWT; the **JWT Authorizer** validates it, and the Lambda creates a `PENDING` record in DynamoDB before returning a 5-minute **Presigned PUT URL** |
| 4 | User → S3 (`imageguard-images`) | The browser uploads image bytes directly to `uploads/{sub}/{uuid}.jpg` in S3 without routing through API Gateway or Lambda |
| 5 | S3 → Lambda `process-image` | An **S3 `ObjectCreated:Put` event** (prefix `uploads/`, suffix `.jpg`) asynchronously triggers `process-image` |
| 6 | `process-image` → Rekognition | Runs `DetectModerationLabels` (≥ 60% confidence). Only clean images proceed to `DetectLabels` (up to 10 labels, ≥ 80% confidence) |
| 7 | `process-image` → DynamoDB | Executes a conditional `UpdateItem` (`status = PENDING`) transitioning the item to `APPROVED` or `REJECTED` |
| 8 | `process-image` → SNS | Only when `REJECTED` **and** the DynamoDB conditional write succeeds does the function publish an alert to **Amazon SNS** |
| 9 | `process-image` → SQS DLQ | Events that continue failing after 2 async retries are sent to the **SQS Dead-Letter Queue** (14-day retention) |
| 10 | User → API Gateway → Lambda `get-images` | The web app calls `GET /images`; `get-images` queries DynamoDB by `userId = sub` and signs short-lived **Presigned GET URLs** exclusively for `APPROVED` items |
| 11–12 | Lambdas → CloudWatch → SNS | Logs and EMF custom metrics flow into **CloudWatch**; triggered alarms notify administrators via **SNS** |

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
