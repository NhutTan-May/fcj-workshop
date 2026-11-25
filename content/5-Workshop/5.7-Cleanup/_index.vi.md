---
title: "Dọn dẹp tài nguyên"
date: 2026-11-20
weight: 7
chapter: false
pre: " <b> 5.7. </b> "
---

Sau khi hoàn tất kiểm thử và chấm điểm, hãy xóa toàn bộ tài nguyên theo đúng thứ tự bên dưới (vì một số dịch vụ có ràng buộc phụ thuộc, ví dụ CloudFront phải tắt trước khi xóa) để tránh phát sinh chi phí ngoài ý muốn.

{{% notice warning %}}
Thao tác xóa tài nguyên **không thể hoàn tác**. Hãy lưu lại ảnh chụp màn hình, log hoặc dữ liệu cần cho báo cáo trước khi thực hiện.
{{% /notice %}}

## 1. Vô hiệu hóa và xóa CloudFront (thực hiện trước, mất khoảng 10 phút)

Mở **CloudFront → Distributions → `imageguard-web` → Disable**, chờ trạng thái chuyển sang *Disabled* rồi nhấn **Delete**. Tiếp theo vào **Security → Origin access** để xóa Origin Access Control `imageguard-web-oac`.

## 2. Xóa các tài nguyên còn lại bằng AWS CloudShell

Chạy lần lượt các lệnh sau trong **AWS CloudShell** (`us-east-1`):

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

# 4. S3 Buckets (xoa toan bo object truoc khi xoa bucket)
aws s3 rb s3://imageguard-images-$ACC --force
aws s3 rb s3://imageguard-web-$ACC --force

# 5. DynamoDB, SNS, SQS
aws dynamodb delete-table --table-name imageguard-images
aws sns delete-topic --topic-arn arn:aws:sns:us-east-1:$ACC:imageguard-alerts
aws sqs delete-queue --queue-url https://sqs.us-east-1.amazonaws.com/$ACC/imageguard-process-dlq

# 6. IAM Roles (phai xoa inline policy truoc khi xoa role)
for r in process-image presign-upload get-images; do
  aws iam delete-role-policy --role-name imageguard-$r-role --policy-name imageguard-$r-policy
  aws iam delete-role --role-name imageguard-$r-role
done
```

## 3. Kiểm tra xác nhận sau khi dọn dẹp

- Kiểm tra lại trên Console các dịch vụ **S3, Lambda, DynamoDB, API Gateway, Cognito, SNS, SQS, CloudWatch** để đảm bảo không còn tài nguyên nào mang tiền tố `imageguard`.
- Theo dõi mục **Billing → Bills** trong những ngày tiếp theo để chắc chắn không còn phát sinh chi phí.
- Tùy chọn: gỡ group `imageguard-dev-2` và policy `AWSCloudShellFullAccess` khỏi IAM User nếu không tiếp tục dùng.
