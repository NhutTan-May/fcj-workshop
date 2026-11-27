---
title: "Kiến trúc"
date: 2026-11-20
weight: 3
chapter: false
pre: " <b> 5.3. </b> "
---

![Sơ đồ kiến trúc ImageGuard](/fcj-workshop/images/2-proposal/imageguard-architecture.drawio.png)

## Luồng xử lý chi tiết

| Bước | Đường đi dữ liệu | Mô tả kỹ thuật |
|---|---|---|
| 1 | User → CloudFront → S3 (`imageguard-web`) | Người dùng truy cập trang web qua HTTPS; CloudFront đọc file tĩnh từ bucket private thông qua **Origin Access Control (OAC)** |
| 2 | User ↔ Cognito | Người dùng đăng nhập qua **Cognito Managed Login** (OAuth 2.0 Authorization Code + PKCE) và nhận về **JWT token** |
| 3 | User → API Gateway → Lambda `presign-upload` | Frontend gọi `POST /upload-url` kèm JWT; **JWT Authorizer** xác thực token, Lambda tạo bản ghi `PENDING` trong DynamoDB và trả về **Presigned PUT URL** (hết hạn sau 5 phút) |
| 4 | User → S3 (`imageguard-images`) | Trình duyệt tải ảnh trực tiếp lên S3 theo đường dẫn `uploads/{sub}/{uuid}.jpg` mà không đi qua API Gateway hay Lambda |
| 5 | S3 → Lambda `process-image` | Sự kiện **S3 Event `ObjectCreated:Put`** (giới hạn prefix `uploads/`, suffix `.jpg`) kích hoạt bất đồng bộ Lambda xử lý |
| 6 | `process-image` → Rekognition | Gọi `DetectModerationLabels` (ngưỡng ≥ 60%). Chỉ khi không vi phạm mới gọi tiếp `DetectLabels` (tối đa 10 nhãn, ngưỡng ≥ 80%) |
| 7 | `process-image` → DynamoDB | Thực hiện `UpdateItem` có điều kiện (`status = PENDING`) để chuyển sang `APPROVED` hoặc `REJECTED` |
| 8 | `process-image` → SNS | Nếu ảnh bị `REJECTED` **và** bước ghi DynamoDB thành công, hệ thống phát thông báo qua **Amazon SNS** tới email quản trị viên |
| 9 | `process-image` → SQS DLQ | Nếu quá trình xử lý vẫn lỗi sau 2 lần retry, sự kiện được đẩy vào **SQS Dead-Letter Queue** (lưu 14 ngày) |
| 10 | User → API Gateway → Lambda `get-images` | Frontend gọi `GET /images`; Lambda truy vấn DynamoDB theo `userId = sub` và chỉ cấp **Presigned GET URL** cho các ảnh `APPROVED` |
| 11–12 | Lambdas → CloudWatch → SNS | Toàn bộ log và metric EMF được đẩy về **CloudWatch**; khi vượt ngưỡng, Alarm gửi thông báo qua **SNS** |

## Chính sách kiểm duyệt nội dung

- **Danh mục bị chặn (cấp 1):** `Explicit`, `Violence`, `Visually Disturbing`, `Drugs & Tobacco`, `Hate Symbols`. Danh sách này được cấu hình qua biến môi trường `BLOCKED_CATEGORIES` nên có thể điều chỉnh linh hoạt mà không phải sửa mã nguồn.
- **Nhóm nhãn cho phép có giám sát:** `Alcohol` được phép hiển thị nhưng vẫn lưu lại nhãn kiểm duyệt trong DynamoDB để quản trị viên có thể rà soát khi cần.
- **Tối ưu lượt gọi AI:** Luồng xử lý luôn kiểm duyệt trước, gắn nhãn sau — những ảnh bị chặn ở bước `DetectModerationLabels` sẽ bỏ qua `DetectLabels` nhằm tiết kiệm chi phí.

## Thiết kế bảo mật

| Lớp bảo vệ | Cách triển khai trong ImageGuard |
|---|---|
| **Least Privilege** | Mỗi hàm Lambda sử dụng một IAM execution role độc lập, không dùng `AWSLambdaBasicExecutionRole` |
| **Lưu trữ riêng tư** | Cả hai bucket S3 đều bật **Block all public access**, tắt ACL, bật mã hóa mặc định SSE-S3 và có bucket policy ép buộc HTTPS |
| **Xác thực & Cô lập người dùng** | JWT Authorizer kiểm tra `issuer` (User Pool) và `audience` (App Client); `userId` luôn được trích xuất từ claim `sub` trong token, tuyệt đối không lấy từ request body |
| **Upload có kiểm soát** | Presigned PUT URL do backend ký, chỉ cho phép ghi vào đúng key được cấp phát, hết hạn sau 5 phút và ràng buộc `Content-Type: image/jpeg` |
| **Chống Confused Deputy** | Resource-based policy của Lambda ràng buộc điều kiện `aws:SourceArn` và `aws:SourceAccount` |
| **Chống Recursive Trigger** | S3 trigger chỉ lắng nghe prefix `uploads/` và role của `process-image` không có quyền ghi vào S3 |
| **Tính Idempotent** | Câu lệnh `UpdateItem` dùng điều kiện `status = PENDING`, ngăn chặn sự kiện trùng lặp hoặc ghi đè làm sai lệch kết quả |
| **Bảo vệ Frontend** | CloudFront dùng OAC, ép HTTPS, đính kèm `SecurityHeadersPolicy` (HSTS, X-Frame-Options…), và mã JavaScript chỉ dùng `textContent` để phòng tránh XSS |

### Ma trận phân quyền theo từng Lambda

| Hàm Lambda | Quyền được cấp (Allow) | Quyền bị chặn (Implicit Deny) |
|---|---|---|
| `imageguard-process-image` | `s3:GetObject` (`uploads/*`), 2 action Rekognition, `dynamodb:UpdateItem` (1 bảng), `sns:Publish` (1 topic), `sqs:SendMessage` (1 DLQ), ghi log của chính nó | `s3:PutObject`, `dynamodb:Query`, `dynamodb:PutItem` |
| `imageguard-presign-upload` | `s3:PutObject` (`uploads/*`), `dynamodb:PutItem` (1 bảng), ghi log của chính nó | `s3:GetObject`, `dynamodb:Query`, `dynamodb:UpdateItem` |
| `imageguard-get-images` | `dynamodb:Query` (1 bảng), `s3:GetObject` (`uploads/*`), ghi log của chính nó | `s3:PutObject`, `dynamodb:PutItem`, `dynamodb:UpdateItem` |

> **Lưu ý về Rekognition:** Các API `DetectModerationLabels` và `DetectLabels` không hỗ trợ giới hạn quyền theo ARN tài nguyên cụ thể nên bắt buộc khai báo `"Resource": "*"`. Để bù lại, policy chỉ mở đúng 2 action cần thiết và tuyệt đối không dùng `rekognition:*`.

## Sao lưu dữ liệu (Backup & Recovery)

Bảng DynamoDB `imageguard-images` được bật tính năng **Point-in-time recovery (PITR)**, cho phép khôi phục dữ liệu về bất kỳ thời điểm nào theo từng giây trong vòng **35 ngày gần nhất** nếu xảy ra sự cố xóa nhầm hoặc ghi sai dữ liệu. Kết hợp với độ bền lưu trữ **99,999999999%** của Amazon S3 đối với tệp ảnh gốc, hệ thống có thể phục hồi đồng bộ cả metadata lẫn ảnh khi cần.

```bash
aws dynamodb update-continuous-backups --table-name imageguard-images \
  --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true
# Kết quả trả về: ENABLED
```

## Vì sao chọn Region `us-east-1`?

- **Chi phí tối ưu nhất:** `us-east-1` (N. Virginia) có mức giá thấp nhất trong phần lớn các Region, đặc biệt đối với **Amazon Rekognition** — dịch vụ chiếm phần lớn chi phí hệ thống (5,91 / 8,37 USD/tháng, xem mục 5.6).
- **Đầy đủ dịch vụ cần thiết:** Toàn bộ các dịch vụ trong kiến trúc (Rekognition Content Moderation, Cognito Managed Login, HTTP API…) đều hỗ trợ đầy đủ tại `us-east-1`.
- **Độ trễ mạng không phải điểm nghẽn:** Dù độ trễ từ Việt Nam tới `us-east-1` khoảng 200 ms, giao diện web tĩnh đã được CloudFront phân phối từ edge location gần người dùng; bước tốn thời gian nhất là khâu AI phân tích ảnh (~0,5–1 giây).
- **Phương án triển khai thực tế tại Việt Nam:** Nếu đưa vào vận hành chính thức cho người dùng trong nước, chỉ cần chuyển sang Region Singapore (`ap-southeast-1`) qua biến môi trường để giảm độ trễ và đáp ứng yêu cầu lưu trữ dữ liệu khu vực mà không cần sửa mã nguồn.

## Lý do không đưa Lambda vào VPC

Trong kiến trúc ImageGuard, không có cơ sở dữ liệu hay máy chủ nội bộ nào yêu cầu đặt trong private subnet. Các hàm Lambda giao tiếp trực tiếp với S3, DynamoDB, Rekognition và SNS qua giao thức HTTPS và được xác thực bằng chữ ký IAM. Nếu đặt Lambda vào VPC, hệ thống phải gánh thêm chi phí NAT Gateway hoặc Interface VPC Endpoints (hàng chục USD/tháng) trong khi không mang lại giá trị bảo mật đáng kể cho mô hình này.
