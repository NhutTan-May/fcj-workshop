---
title: "Demo Video"
date: 2026-11-20
weight: 8
chapter: false
pre: " <b> 5.8. </b> "
---

**Demo Video Link:** `[paste YouTube / Google Drive link after recording]`

## Walkthrough Script (5–7 Minutes)

| Timestamp | Demonstration Segment |
|---|---|
| `0:00` | Problem statement and ImageGuard architecture overview (Section 5.3) |
| `1:00` | Open the CloudFront web URL → sign up / sign in via Cognito Managed Login |
| `2:00` | Upload a beer bottle image → switches to **APPROVED** within seconds with detected labels |
| `2:45` | Upload the wooden toy gun → flagged **REJECTED**; show the SNS alert email and explain the false-positive example |
| `3:30` | Inspect DynamoDB item showing `userId = sub`, `moderationLabels`, and `labels` |
| `4:00` | Walk through the CloudWatch Dashboard (`APPROVED`/`REJECTED` counts, processing latency, alarm status) |
| `4:45` | Security checks: unauthenticated API call → `401`; direct S3 bucket access → `403 AccessDenied` |
| `5:30` | Review AWS Pricing Calculator breakdown (~USD 8.37/month) |
| `6:00` | Wrap-up and future improvements (admin review queue for `REJECTED` images, AWS WAF, Presigned POST size enforcement) |
