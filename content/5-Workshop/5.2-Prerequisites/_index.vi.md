---
title: "Chuẩn bị"
date: 2026-11-20
weight: 2
chapter: false
pre: " <b> 5.2. </b> "
---

## Tài khoản AWS và Phân quyền

1. Chuẩn bị một tài khoản AWS đã cấu hình **AWS Budgets** gửi cảnh báo khi chi phí vượt quá **10 USD**.
2. **Tuyệt đối không dùng tài khoản Root** cho thao tác thực hành: chỉ dùng Root để bật MFA, sau đó tạo IAM User riêng và không tạo access key cho Root.
3. Tạo một IAM User đã bật **MFA** và gắn vào 2 IAM Group (do giới hạn mỗi group tối đa 10 managed policy):

| IAM Group | Danh sách Managed Policy |
|---|---|
| `imageguard-dev` | `AmazonS3FullAccess`, `AmazonDynamoDBFullAccess`, `AmazonSNSFullAccess`, `AmazonSQSFullAccess`, `AWSLambda_FullAccess`, `IAMFullAccess`, `CloudWatchFullAccessV2`, `AmazonRekognitionReadOnlyAccess`, `AWSCloudShellFullAccess` |
| `imageguard-dev-2` | `AmazonCognitoPowerUser`, `AmazonAPIGatewayAdministrator`, `CloudFrontFullAccess` |

![Group IAM với các policy](/fcj-workshop/images/5-workshop/0-1-iam-group-policies.png)

{{% notice warning %}}
Policy `IAMFullAccess` cho phép người dùng tự mở rộng quyền (privilege escalation). Trong phạm vi lab cá nhân, quyền này cần thiết để tạo role riêng cho từng Lambda; khi triển khai ở môi trường production, hãy giới hạn bằng **IAM Permission Boundary**.
{{% /notice %}}

## Region và Công cụ sử dụng

- **Region thực hành:** **US East (N. Virginia) – `us-east-1`**. Hãy luôn kiểm tra góc trên bên phải AWS Console trước mỗi bước cấu hình.
- **Công cụ:** Trình duyệt web truy cập AWS Console. Từ Step 3 đến Step 4, bạn có thể chạy script tự động ngay trên **AWS CloudShell** (tích hợp sẵn AWS CLI) mà không cần cài đặt công cụ trên máy cá nhân.
- **Account ID:** Ghi lại mã số tài khoản AWS gồm 12 chữ số liền nhau (không dấu gạch ngang). Trong các bước hướng dẫn, hãy thay thế chuỗi `<ACCOUNT_ID>` bằng mã số này.

## Bộ ảnh dùng để kiểm thử

Chuẩn bị sẵn 2–3 ảnh định dạng **JPEG** dung lượng dưới 5 MB:
- **Ảnh hợp lệ (có ngữ cảnh nhẹ):** ví dụ ảnh chai bia (Rekognition sẽ nhận diện nhãn *Alcohol* nhưng chính sách hệ thống vẫn cho phép hiển thị).
- **Ảnh kích hoạt từ chối (false positive minh họa):** ví dụ ảnh súng đồ chơi bằng gỗ (Rekognition có thể nhận diện nhầm là *Violence/Weapons* với độ tin cậy ~97%, rất phù hợp để kiểm chứng luồng từ chối và email cảnh báo).

## Kiểm tra quyền dùng Amazon Rekognition

Ở quy mô thử nghiệm của workshop, Amazon Rekognition không cần gửi yêu cầu mở quota. Để xác nhận tài khoản hoạt động bình thường, hãy mở **Amazon Rekognition → Content moderation → Upload** một ảnh mẫu bất kỳ; nếu Console trả về kết quả phân tích là bạn đã sẵn sàng thực hành.
