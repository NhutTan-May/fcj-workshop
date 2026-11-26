---
title: "Prerequisites"
date: 2026-11-20
weight: 2
chapter: false
pre: " <b> 5.2. </b> "
---

## AWS Account & Permissions

1. Prepare an AWS account with an **AWS Budgets** alert configured to notify you if monthly spend exceeds **USD 10**.
2. **Never use the Root user** for lab work: enable MFA on Root, store its credentials securely, and do not generate Root access keys.
3. Create an **MFA-enabled IAM User** and assign it to two IAM Groups (each group supports up to 10 attached managed policies):

| IAM Group | Attached Managed Policies |
|---|---|
| `imageguard-dev` | `AmazonS3FullAccess`, `AmazonDynamoDBFullAccess`, `AmazonSNSFullAccess`, `AmazonSQSFullAccess`, `AWSLambda_FullAccess`, `IAMFullAccess`, `CloudWatchFullAccessV2`, `AmazonRekognitionReadOnlyAccess`, `AWSCloudShellFullAccess` |
| `imageguard-dev-2` | `AmazonCognitoPowerUser`, `AmazonAPIGatewayAdministrator`, `CloudFrontFullAccess` |

![IAM group policies configuration](/fcj-workshop/images/5-workshop/0-1-iam-group-policies.png)

{{% notice warning %}}
`IAMFullAccess` permits privilege escalation. It is acceptable in this sandboxed workshop so you can create dedicated Lambda execution roles; in production environments, constrain it using an **IAM Permission Boundary**.
{{% /notice %}}

## Region & Tooling

- **Target Region:** **US East (N. Virginia) – `us-east-1`**. Verify the Region selector in the top-right corner of the AWS Console before each step.
- **Tooling:** A modern browser for the AWS Console. Steps 3 and 4 can be automated directly inside **AWS CloudShell** (AWS CLI pre-installed), requiring zero local setup.
- **Account ID:** Copy your 12-digit AWS Account ID (digits only, no dashes) and replace `<ACCOUNT_ID>` wherever shown in policies and commands.

## Sample Test Images

Prepare 2–3 **JPEG** images under 5 MB each:
- **Valid image:** e.g., a beer bottle photo (Rekognition detects *Alcohol*, which our moderation policy permits).
- **Flagged image (false-positive demo):** e.g., a wooden toy gun (Rekognition flags *Violence/Weapons* at ~97% confidence, ideal for testing `REJECTED` handling and SNS alerts safely).

## Quick Rekognition Readiness Check

Amazon Rekognition does not require a manual quota increase for workshop usage. To verify readiness, open **Amazon Rekognition → Content moderation → Upload** any sample photo in the Console and confirm that moderation labels appear.
