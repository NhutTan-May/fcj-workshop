---
title: "Testing & Troubleshooting"
date: 2026-11-20
weight: 5
chapter: false
pre: " <b> 5.5. </b> "
---

## Functional Testing

| Test Case | Execution Method | Observed Outcome |
|---|---|---|
| **Valid image** | Upload a beer bottle photo through the web UI | Marked `APPROVED`, tagged with *Alcohol · Beer · Bottle…*, visible in the gallery |
| **Violating image** | Upload a wooden toy gun photo | Marked `REJECTED`, `blockedCategories = Violence`, SNS alert email delivered to admin |
| **User identity** | Inspect DynamoDB item attributes | `userId` stores the Cognito `sub` UUID rather than raw email addresses |
| **Idempotency** | Overwrite an already processed S3 key | Logs record `item_missing_or_already_processed`; finalized verdict remains unchanged |
| **Untracked upload** | Put an object directly to S3 without a `PENDING` record | No DynamoDB item created and no alert email triggered |
| **DLQ recovery path** | Misconfigure `TABLE_NAME` and invoke asynchronously | After 2 retries fail, SQS DLQ receives a `RetriesExhausted` payload (`approximateInvokeCount: 3`) |

![Gallery: APPROVED and REJECTED images](/fcj-workshop-template/images/5-workshop/26-3-web-gallery.png)

{{% notice note %}}
**False-positive observation:** the wooden toy gun triggered *Violence/Weapons* at ~97% confidence. In production, false positives can be tuned by raising per-category `MIN_MODERATION_CONFIDENCE` thresholds or introducing a human review workflow for `REJECTED` items.
{{% /notice %}}

The DLQ experiment also validated **IAM Least Privilege**: the captured failure payload reported `AccessDeniedException … not authorized to perform: dynamodb:UpdateItem on resource: …table/khong-ton-tai`, confirming IAM blocks writes to any table other than `imageguard-images`.

![DLQ message: RetriesExhausted](/fcj-workshop-template/images/5-workshop/15-2b-dlq-message-body.jpg)

## Security Verification

| Security Check | Expected Behavior | Actual Result |
|---|---|---|
| Call `GET /images` without JWT | `401 Unauthorized` | ✅ Returned `{"message":"Unauthorized"}` |
| Direct access to web bucket file URL | `403 AccessDenied` | ✅ Blocked; readable only via CloudFront OAC |
| Direct access to image Object URL | `403 AccessDenied` | ✅ Blocked by S3 Block Public Access |
| Reuse Presigned URL after 6 minutes | Expired token error | ✅ Returned `Request has expired` |
| Plain `http://` request | Forced HTTPS redirect | ✅ Automatically redirected to `https://` |
| HTTP Response Headers | Security headers present | ✅ HSTS, X-Frame-Options, X-Content-Type-Options, Referrer-Policy verified |
| OAuth login flow | PKCE enforced | ✅ Authorization URL includes `code_challenge_method=S256` |

![API without a token returns 401](/fcj-workshop-template/images/5-workshop/27-1-api-no-token.jpg)

![Direct S3 access is denied](/fcj-workshop-template/images/5-workshop/27-2-s3-web-direct-denied.jpg)

Static assets are served from CloudFront's **SGN (Ho Chi Minh City)** edge location, keeping frontend load times fast for Vietnamese users despite the backend residing in `us-east-1`.

## Troubleshooting Log

### 1. Inline Policy Error: *"The policy failed legacy parsing"*
- **Cause:** A stray `>` character remained after the Account ID in the DynamoDB ARN (`…:125022369745>:table/…`), causing the IAM Review page to show only 5 of 6 services.
- **Fix:** Ensure the Account ID is 12 continuous digits with no hyphens or angle brackets; run Find & Replace on `<ACCOUNT_ID>` before pasting JSON.

### 2. Alert Email Sent While DynamoDB Item Remained `PENDING`
- **Cause:** Early code published the SNS alert **before** running the conditional DynamoDB `UpdateItem`. When an upload key did not match the pending item, DynamoDB rejected the update while the email was already dispatched.
- **Fix:** Reorder operations so `publish_alert` executes only after `UpdateItem` returns `True`. If SNS publishing fails, log an `ERROR` and emit `AlertPublishFailed` for CloudWatch Alarms rather than triggering duplicate retries.

![Email arrived while the item was still PENDING](/fcj-workshop-template/images/5-workshop/t-1-bug-email-before-update.png)

### 3. CloudWatch Alarms Failed to Send Emails
- **Symptom:** Alarms transitioned to `ALARM` without delivering emails, and alarm history showed *"Failed to execute action"*—even though Lambda-originated `REJECTED` emails still arrived.
- **Cause:** The SNS topic used AWS-managed KMS encryption (`alias/aws/sns`), which does not authorize CloudWatch Alarms to call `kms:GenerateDataKey`.
- **Fix:** Disable encryption on `imageguard-alerts` (notifications only include `userId` and `imageId`, never images or links), restoring *"Successfully executed action"*. If encryption is required by policy, attach a Customer Managed Key (~USD 1/month) granting access to `cloudwatch.amazonaws.com`.

![Failed to execute action](/fcj-workshop-template/images/5-workshop/28-3a-bug-sns-kms-failed.jpg)

![Successfully executed action after the fix](/fcj-workshop-template/images/5-workshop/28-3b-fix-sns-alarm-success.jpg)

### 4. Updated AWS Console Navigation Differences
- In the updated Lambda Console, execution role selection appears under **Custom settings → Additional settings → Custom execution role**.
- In the updated CloudWatch Console, **Log groups** is listed under **Logs → Log Management**, and **Logs Insights** appears as **Log Analytics**.
- Log streams display only recent tail entries initially—click **Load more** to view the `image_processed` event entry.

### 5. Amazon Bedrock Unavailable Due to Zero Quota
The initial design targeted **Amazon Nova Lite** on Amazon Bedrock, but every team AWS account had a zero quota and quota increase requests were denied. Pivoting to **Amazon Rekognition** eliminated quota blockers while providing managed moderation/labeling APIs with predictable per-image pricing.
