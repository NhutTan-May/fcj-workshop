---
title: "Clean-up"
date: 2026-11-20
weight: 7
chapter: false
pre: " <b> 5.7. </b> "
---

Once testing and grading are complete, delete all provisioned resources in the order below (respecting service dependencies such as disabling CloudFront before deletion) to eliminate ongoing charges.

{{% notice warning %}}
Resource deletion is **permanent and irreversible**. Export any screenshots, logs, or metrics needed for your report before running these steps.
{{% /notice %}}

## 1. Disable and Delete CloudFront (Run First, ~10 Minutes)

Navigate to **CloudFront → Distributions → `imageguard-web` → Disable**, wait until the status changes to *Disabled*, and click **Delete**. Afterward, open **Security → Origin access** and remove `imageguard-web-oac`.

## 2. Remove Remaining Resources via AWS CloudShell

Execute the following commands in **AWS CloudShell** (`us-east-1`):

```bash
ACC=$(aws sts get-caller-identity --query Account --output text)
export AWS_REGION=us-east-1 AWS_PAGER=""

# 1. Monitoring
aws cloudwatch delete-dashboards --dashboard-names imageguard
aws cloudwatch delete-alarms --alarm-names $(aws cloudwatch describe-alarms --alarm-name-prefix imageguard --query 'MetricAlarms[].AlarmName' --output text)

# 2. API Gateway + Cognito
API_ID=$(aws apigatewayv2 get-apis --query "Items[?Name=='imageguard-api'].ApiId | [0]" --output text)
aws apigatewayv2 delete-api --api-id "$API_ID"
POOL_ID=$(aws cognito-idp list-user-pools --max-results 60 --query "UserPools[?Name=='imageguard-users'].Id | [0]" --output text)
aws cognito-idp delete-user-pool-domain --user-pool-id "$POOL_ID" --domain "imageguard-$ACC"
aws cognito-idp delete-user-pool --user-pool-id "$POOL_ID"

# 3. Lambda + Log Groups
for f in process-image presign-upload get-images; do
  aws lambda delete-function --function-name imageguard-$f
  aws logs delete-log-group --log-group-name /aws/lambda/imageguard-$f
done

# 4. S3 Buckets (empty objects then remove bucket)
aws s3 rb s3://imageguard-images-$ACC --force
aws s3 rb s3://imageguard-web-$ACC --force

# 5. DynamoDB, SNS, SQS
aws dynamodb delete-table --table-name imageguard-images
aws sns delete-topic --topic-arn arn:aws:sns:us-east-1:$ACC:imageguard-alerts
aws sqs delete-queue --queue-url https://sqs.us-east-1.amazonaws.com/$ACC/imageguard-process-dlq

# 6. IAM Roles (delete inline policy before deleting role)
for r in process-image presign-upload get-images; do
  aws iam delete-role-policy --role-name imageguard-$r-role --policy-name imageguard-$r-policy
  aws iam delete-role --role-name imageguard-$r-role
done
```

## 3. Post-Cleanup Verification

- Confirm across **S3, Lambda, DynamoDB, API Gateway, Cognito, SNS, SQS, and CloudWatch** that no `imageguard` resources remain.
- Review **Billing → Bills** over the next few days to verify zero recurring spend.
- Optional: detach `imageguard-dev-2` and `AWSCloudShellFullAccess` from your IAM user when finished.
