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

![Gallery: APPROVED and REJECTED images](/fcj-workshop/images/5-workshop/26-4-web-gallery-v2.jpg)

{{% notice note %}}
**False-positive observation:** the wooden toy gun triggered *Violence/Weapons* at ~97% confidence. In production, false positives can be tuned by raising per-category `MIN_MODERATION_CONFIDENCE` thresholds or introducing a human review workflow for `REJECTED` items.
{{% /notice %}}

The DLQ experiment also validated **IAM Least Privilege**: the captured failure payload reported `AccessDeniedException … not authorized to perform: dynamodb:UpdateItem on resource: …table/khong-ton-tai`, confirming IAM blocks writes to any table other than `imageguard-images`.

![DLQ message: RetriesExhausted](/fcj-workshop/images/5-workshop/15-2b-dlq-message-body.jpg)

## Success Criteria Verification (Proposal Section 3.2)

| Criterion | Target | Measured Result |
|---|---|---|
| Processing time per image | ≤ 10 seconds | ✅ Roughly **0.5–1 second** (`ProcessingLatency` metric on dashboard) |
| Violating images blocked accurately | ≥ 90% | ❌ **2/3 (67%)**: both firearm photos blocked, one smoking photo missed |
| False positive rate on valid images | ≤ 10% | ❌ **1/8 (12.5%)**: wooden toy gun blocked (`Violence/Weapons` 98%); toy water gun approved |
| Unauthenticated API call | Rejected (401) | ✅ `{"message":"Unauthorized"}` |
| Cross-user image access | Impossible | ✅ Real user gallery lists only objects matching the caller's `sub`, even with `test-user-001` items in the table |
| Violating image detected | Admin email alert | ✅ SNS alert delivered |
| Direct S3 bucket access | Blocked | ✅ `403 AccessDenied` |

Both accuracy metrics were measured across **12 images** (2 initial sample photos plus 10 Wikimedia Commons images) processed end-to-end via [`batch_test.sh`](/fcj-workshop/code/imageguard/scripts/batch_test.sh) on **26/11/2026**:

| Group | Test Image | Expected | Actual Result | Labels / Categories |
|---|---|---|---|---|
| Normal | Laptop on a desk | APPROVED | ✅ APPROVED | `Furniture`, `Table`, `Desk`, `Computer` |
| Normal | Bicycle | APPROVED | ✅ APPROVED | `Wheel`, `Bicycle`, `Transportation` |
| Normal | Pizza | APPROVED | ✅ APPROVED | `Food`, `Meal`, `Pizza` |
| Normal | Cat | APPROVED | ✅ APPROVED | `Animal`, `Abyssinian`, `Cat` |
| Alcohol | Beer bottle | APPROVED (`Alcohol` allowed) | ✅ APPROVED | `Alcohol`, `Beer`, `Bottle` |
| Tobacco | Person smoking holding a pickaxe | REJECTED | ❌ APPROVED | Moderation saw `Smoking` at 27% (< 60% threshold); `DetectLabels` mislabeled pickaxe as `Rifle` |
| Weapons | Handgun | REJECTED | ✅ REJECTED | `Violence` → `Weapons` |
| Weapons | Soldier holding a rifle | REJECTED | ✅ REJECTED | `Violence` → `Weapons`, `Weapon Violence` |
| Tricky | Toy water gun | APPROVED | ✅ APPROVED | `Toy`, `Water Gun` |
| Tricky | Halloween fake-blood makeup | Policy-dependent | REJECTED | `Violence` → `Blood & Gore` (excluded from both rates) |

**Observations & Next Improvements:**
- A uniform 60% threshold reliably flags clear weapons yet misses small cigarettes in wide shots; per-category thresholds (e.g., 25% for `Drugs & Tobacco`) can boost recall at the expense of extra false positives.
- Weapon-shaped toys and Halloween makeup remain ambiguous, reinforcing the need for a human-in-the-loop review queue for `REJECTED` items.
- Twelve images demonstrate practical behavior trends rather than statistically significant precision/recall benchmarks.

## Security Verification

| Security Check | Expected Behavior | Actual Result |
|---|---|---|
| Call `GET /images` without JWT | `401 Unauthorized` | ✅ Returned `{"message":"Unauthorized"}` |
| Direct access to web bucket file URL | `403 AccessDenied` | ✅ Blocked; readable only via CloudFront OAC |
| Direct access to image Object URL | `403 AccessDenied` | ✅ Blocked by S3 Block Public Access |
| Reuse Presigned URL after 6 minutes | Expired token error | ✅ `200` before expiry, `403 Request has expired` afterward |
| Plain `http://` request | Forced HTTPS redirect | ✅ `301 Moved Permanently` to `https://` |
| HTTP Response Headers | Security headers present | ✅ HSTS, X-Frame-Options, X-Content-Type-Options, Referrer-Policy verified |
| OAuth login flow | PKCE enforced | ✅ Authorization URL includes `code_challenge_method=S256` |

![API without a token returns 401](/fcj-workshop/images/5-workshop/27-1-api-no-token.jpg)

![Direct S3 web access is denied](/fcj-workshop/images/5-workshop/27-2-s3-web-direct-denied.jpg)

![Direct image Object URL is denied](/fcj-workshop/images/5-workshop/27-3-s3-image-direct-denied.jpg)

![HTTP to HTTPS redirect and expired presigned URL](/fcj-workshop/images/5-workshop/27-4-https-redirect-presigned-expired.jpg)

Static assets are served from CloudFront's **SGN (Ho Chi Minh City)** edge location, keeping frontend load times fast for Vietnamese users despite the backend residing in `us-east-1`.

## Troubleshooting Log

### 1. Inline Policy Error: *"The policy failed legacy parsing"*
- **Cause:** A stray `>` character remained after the Account ID in the DynamoDB ARN (`…:125022369745>:table/…`), causing the IAM Review page to show only 5 of 6 services.
- **Fix:** Ensure the Account ID is 12 continuous digits with no hyphens or angle brackets; run Find & Replace on `<ACCOUNT_ID>` before pasting JSON.

### 2. Alert Email Sent While DynamoDB Item Remained `PENDING`
- **Cause:** Early code published the SNS alert **before** running the conditional DynamoDB `UpdateItem`. When an upload key did not match the pending item, DynamoDB rejected the update while the email was already dispatched.
- **Fix:** Reorder operations so `publish_alert` executes only after `UpdateItem` returns `True`. If SNS publishing fails, log an `ERROR` and emit `AlertPublishFailed` for CloudWatch Alarms rather than triggering duplicate retries.

![The alert email was sent…](/fcj-workshop/images/5-workshop/t-1-bug-email-before-update.png)
*The alert email was sent…*

![…while the item was still PENDING](/fcj-workshop/images/5-workshop/t-2-bug-item-still-pending.png)
*…while the item was still PENDING*

### 3. CloudWatch Alarms Failed to Send Emails
- **Symptom:** Alarms transitioned to `ALARM` without delivering emails, and alarm history showed *"Failed to execute action"*—even though Lambda-originated `REJECTED` emails still arrived.
- **Cause:** The SNS topic used AWS-managed KMS encryption (`alias/aws/sns`), which does not authorize CloudWatch Alarms to call `kms:GenerateDataKey`.
- **Fix:** Disable encryption on `imageguard-alerts` (notifications only include `userId` and `imageId`, never images or links), restoring *"Successfully executed action"*. If encryption is required by policy, attach a Customer Managed Key (~USD 1/month) granting access to `cloudwatch.amazonaws.com`.

![Failed to execute action](/fcj-workshop/images/5-workshop/28-3a-bug-sns-kms-failed.jpg)

![Successfully executed action after the fix](/fcj-workshop/images/5-workshop/28-3b-fix-sns-alarm-success.jpg)

### 4. Updated AWS Console Navigation Differences
- In the updated Lambda Console, execution role selection appears under **Custom settings → Additional settings → Custom execution role**.
- In the updated CloudWatch Console, **Log groups** is listed under **Logs → Log Management**, and **Logs Insights** appears as **Log Analytics**.
- Log streams display only recent tail entries initially—click **Load more** to view the `image_processed` event entry.

### 5. Amazon Bedrock Unavailable Due to Zero Quota
The initial design targeted **Amazon Nova Lite** on Amazon Bedrock, but every team AWS account had a zero quota and quota increase requests were denied. Pivoting to **Amazon Rekognition** eliminated quota blockers while providing managed moderation/labeling APIs with predictable per-image pricing.
