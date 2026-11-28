---
title: "Kiến trúc"
date: 2026-11-20
weight: 3
chapter: false
pre: " <b> 5.3. </b> "
---

![Sơ đồ kiến trúc ImageGuard](/fcj-workshop/images/2-proposal/imageguard-architecture.drawio.png)

Sơ đồ được chia thành 5 nhóm chức năng: **Frontend**, **Authentication & API**, **Compute**, **Storage & AI**, và **Notification & Monitoring**. Amazon CloudFront đặt ngoài khung Region vì là dịch vụ toàn cầu (global). Màu mũi tên phân biệt từng luồng xử lý: **xanh dương** (tải ảnh lên), **xanh lá** (xem ảnh), **cam** (phân tích AI), **xám nét đứt** (giám sát) và **đỏ nét đứt** (phân quyền IAM), tất cả vẽ theo chiều **gửi yêu cầu** (request).

## Luồng xử lý chi tiết (khớp số trên sơ đồ)

| # | Từ → Đến | Mô tả kỹ thuật |
|---|---|---|
| 1 | User → CloudFront → S3 (`imageguard-web`) | Trình duyệt tải trang web tĩnh (HTML/JS) qua HTTPS; CloudFront đọc bucket web private thông qua **Origin Access Control (OAC)** |
| 2 | User → Cognito | Đăng nhập qua **Cognito Managed Login** (OAuth 2.0 Authorization Code + PKCE) và nhận JWT (ID token + access token) |
| 3 | User → API Gateway | Frontend gọi `POST /upload-url` kèm access token; **JWT Authorizer** kiểm tra chữ ký, issuer và audience |
| 3a | API Gateway → `presign-upload` | Lambda trích xuất `userId` từ claim `sub` và sinh `imageId` (UUIDv4) |
| 3b | `presign-upload` → DynamoDB | Ghi trước bản ghi `PENDING`, sau đó trả về Presigned PUT URL (hết hạn 5 phút) cho key `uploads/{sub}/{imageId}.jpg` |
| 4 | User → S3 (`imageguard-images`) | Trình duyệt `PUT` ảnh trực tiếp lên S3 bằng Presigned URL; dữ liệu ảnh không đi qua API Gateway hay Lambda |
| 5 | S3 → `process-image` | **S3 Event** `ObjectCreated:Put` (prefix `uploads/`, suffix `.jpg`) kích hoạt **bất đồng bộ** hàm Lambda xử lý |
| 6 | `process-image` → Rekognition | Gọi `DetectModerationLabels` (ngưỡng ≥ 60%); chỉ khi hợp lệ mới gọi tiếp `DetectLabels` (tối đa 10 nhãn, ≥ 80%) |
| 6a | Rekognition → S3 | Rekognition đọc trực tiếp tệp ảnh từ S3 (`S3Object`), giúp Lambda không cần tải ảnh về |
| 7 | `process-image` → DynamoDB | Thực hiện `UpdateItem` **có điều kiện** (`status = PENDING`) để cập nhật sang `APPROVED` hoặc `REJECTED` |
| 8 | `process-image` → SNS → Admin | Chỉ gửi email cảnh báo khi ảnh bị `REJECTED` **và** bước cập nhật DynamoDB thành công |
| 9 | `process-image` → SQS DLQ | Nếu quá trình xử lý vẫn lỗi sau 2 lần retry, sự kiện được chuyển vào Dead-Letter Queue (lưu 14 ngày) |
| 10 | User → API Gateway | Gọi `GET /images` kèm access token (dùng chung đường mũi tên với bước 3) |
| 10a | API Gateway → `get-images` | Lambda trích xuất `userId` từ claim `sub` |
| 10b | `get-images` → DynamoDB | Truy vấn `Query` theo `userId`, sắp xếp ảnh mới nhất trước (tối đa 50 ảnh); chỉ ảnh `APPROVED` được ký Presigned GET URL (5 phút) |
| 10c | User → S3 (`imageguard-images`) | Trình duyệt hiển thị ảnh hợp lệ bằng Presigned GET URL |
| 11 | Lambdas → CloudWatch | Ghi log JSON có cấu trúc và 5 custom metric theo Embedded Metric Format (EMF) |
| 12 | CloudWatch → SNS | 7 Alarm gửi thông báo qua email khi chuyển trạng thái `ALARM` và khi hồi phục về `OK` |
| – | IAM → Lambdas | Mỗi hàm Lambda sở hữu một IAM Role riêng theo nguyên tắc Least Privilege |

## Căn cứ lựa chọn dịch vụ

| Dịch vụ | Vai trò | Lý do lựa chọn | Phương án đã cân nhắc |
|---|---|---|---|
| **Amazon S3** | Lưu ảnh & web tĩnh | Độ bền 99,999999999%, chi phí thấp, phát sinh S3 Event và hỗ trợ Lifecycle xóa ảnh cũ | EBS/EFS: phải gắn vào máy chủ, tốn chi phí cố định |
| **AWS Lambda** | Thực thi 3 hàm backend | Không quản lý server, tự mở rộng theo lưu lượng, trả phí theo lượt chạy | EC2/ECS: phải chạy 24/7 ngay cả khi không có ảnh |
| **Amazon Rekognition** | Kiểm duyệt & gắn nhãn | Có sẵn mô hình AI kiểm duyệt, không cần dữ liệu huấn luyện, tính phí theo ảnh | Amazon Bedrock: tài khoản mới có quota bằng 0 (mục 5.5); tự huấn luyện: tốn dữ liệu và GPU |
| **Amazon DynamoDB** | Lưu trạng thái & nhãn | Serverless, chế độ On-demand trả theo request, truy vấn theo `userId` nhanh, hỗ trợ ghi có điều kiện | Amazon RDS: phải chạy instance 24/7 và cấu hình VPC |
| **Amazon API Gateway (HTTP API)** | Cung cấp API cho web | Tích hợp sẵn JWT Authorizer cho Cognito, CORS, throttling; rẻ hơn REST API ~70% | REST API: dư thừa tính năng không cần thiết; Function URL: thiếu JWT Authorizer gốc |
| **Amazon Cognito** | Đăng ký & đăng nhập | Managed Login có sẵn giao diện, chuẩn OAuth 2.0 + PKCE, không tự lưu mật khẩu | Tự xây dựng Auth: rủi ro bảo mật và công sức bảo trì lớn |
| **Amazon CloudFront** | Phân phối giao diện web | Hỗ trợ HTTPS mặc định, cache tại edge gần người dùng, dùng OAC giữ bucket private | S3 Static Website Hosting: chỉ HTTP, bắt buộc bật public bucket |
| **Amazon SNS** | Gửi email cảnh báo | Triển khai nhanh chỉ với vài dòng code, dùng chung cho cả cảnh báo ảnh vi phạm và CloudWatch Alarms | Amazon SES: phù hợp gửi email cá nhân hóa số lượng lớn, đòi hỏi xác minh domain/sender |
| **Amazon SQS** | Dead-Letter Queue | Lưu trữ sự kiện lỗi trong 14 ngày để kiểm tra và xử lý lại, tránh mất dữ liệu | Không dùng DLQ: sự kiện bất đồng bộ sẽ bị hủy sau khi hết số lần retry |
| **Amazon CloudWatch** | Log, Metric, Alarm, Dashboard | Thu thập log mặc định từ Lambda; EMF tạo metric ngay từ log mà không tốn API call | Công cụ giám sát ngoài: phát sinh cài đặt và quản lý credential bổ sung |
| **AWS IAM** | Phân quyền bảo mật | Tách Role độc lập cho từng Lambda, giới hạn chặt tới từng tài nguyên | Dùng chung 1 Role: một hàm bị khai thác sẽ kéo theo toàn bộ quyền của hệ thống |

## Mô hình dữ liệu (DynamoDB `imageguard-images`)

| Thuộc tính | Kiểu dữ liệu | Ý nghĩa |
|---|---|---|
| `userId` | String – **Partition key** | Claim `sub` của Cognito; mỗi người dùng nằm trong một partition riêng nên `Query` chỉ trả dữ liệu của chính họ |
| `imageId` | String – **Sort key** | UUIDv4 do `presign-upload` sinh ra, đồng thời là tên tệp ảnh trên S3 |
| `status` | String | Vòng đời xử lý: `PENDING` → `APPROVED` hoặc `REJECTED` |
| `createdAt`, `processedAt` | String (ISO 8601) | Thời điểm cấp URL tải lên và thời điểm AI phân tích xong |
| `moderationLabels` | List | Danh sách nhãn kiểm duyệt từ Rekognition (tên nhãn, danh mục cha, độ tin cậy) |
| `labels` | List | Danh sách nhãn mô tả nội dung (tên, độ tin cậy), chỉ tạo với ảnh hợp lệ |
| `blockedCategories` | List | Danh mục cấp 1 khiến ảnh bị từ chối (ví dụ `Violence`) |
| `reason` | String | Mã lý do: `BLOCKED_CATEGORY`, `TOO_LARGE` (> 5 MB) hoặc `INVALID_IMAGE:<lỗi>` |

Không cần tạo Global Secondary Index (GSI) vì toàn bộ truy vấn của hệ thống đều lọc theo `userId` khớp trực tiếp Partition key.

## Chính sách kiểm duyệt nội dung

- **Danh mục bị chặn (cấp 1):** `Explicit`, `Violence`, `Visually Disturbing`, `Drugs & Tobacco`, `Hate Symbols`. Danh sách này cấu hình qua biến môi trường `BLOCKED_CATEGORIES` nên điều chỉnh được mà không phải sửa code.
- **Nhóm nhãn cho phép có giám sát:** `Alcohol` được phép hiển thị nhưng vẫn lưu lại nhãn kiểm duyệt trong DynamoDB để quản trị viên tra cứu khi cần.
- **Tối ưu lượt gọi AI:** Luồng xử lý luôn kiểm duyệt trước, gắn nhãn sau — những ảnh bị chặn ở bước `DetectModerationLabels` sẽ bỏ qua `DetectLabels` nhằm tiết kiệm chi phí.

## Thiết kế bảo mật

| Lớp bảo vệ | Cách triển khai trong ImageGuard |
|---|---|
| **Least Privilege** | Mỗi hàm Lambda sử dụng một IAM execution role độc lập, không dùng `AWSLambdaBasicExecutionRole` |
| **Lưu trữ riêng tư** | Cả hai bucket S3 đều bật **Block all public access**, tắt ACL, bật mã hóa mặc định SSE-S3 và có bucket policy ép buộc HTTPS |
| **Xác thực & Cô lập người dùng** | JWT Authorizer kiểm tra `issuer` (User Pool) và `audience` (App Client); `userId` luôn trích xuất từ claim `sub` trong token, không nhận từ request body |
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

## Đối chiếu AWS Well-Architected Framework

| Trụ cột | Biện pháp áp dụng trong ImageGuard |
|---|---|
| **Operational Excellence** | Log JSON có cấu trúc; 5 custom metric EMF (`ImagesProcessed`, `ImagesApproved`, `ImagesRejected`, `ProcessingLatency`, `AlertPublishFailed`); 7 Alarm và Dashboard `imageguard`; script `setup.sh`, `step4.sh` giúp tái triển khai đồng nhất; ngưỡng và danh mục chặn đặt trong biến môi trường |
| **Security** | Áp dụng đầy đủ các lớp ở mục *Thiết kế bảo mật*: Least Privilege cho từng Lambda, bucket private, JWT Authorizer, Presigned URL 5 phút, CloudFront OAC, chống Confused Deputy, không lưu access key trong mã nguồn |
| **Reliability** | Các dịch vụ managed (S3, Lambda, DynamoDB, SNS, SQS) tự động chạy đa vùng sẵn sàng (Multi-AZ); Lambda bất đồng bộ retry 2 lần trước khi đẩy vào DLQ; `boto3` retry 5 lần khi throttling; ghi có điều kiện đảm bảo idempotent; file hỏng được đánh dấu `REJECTED` thay vì gây treo hệ thống; DynamoDB bật PITR 35 ngày |
| **Performance Efficiency** | Upload thẳng lên S3 giúp API và Lambda không phải chuyển tiếp file ảnh; Rekognition đọc ảnh trực tiếp từ S3; Lambda chạy `arm64` (`process-image` 256 MB, 2 hàm API 128 MB); tự động scale với DynamoDB On-demand và cache frontend tại CloudFront edge |
| **Cost Optimization** | 100% Serverless không phát sinh tài nguyên chạy 24/7; kiểm duyệt trước gắn nhãn sau; dùng HTTP API thay REST API; kiến trúc `arm64`; S3 Lifecycle 30 ngày; CloudWatch Logs retention 14 ngày; dự toán AWS Pricing Calculator ~8,37 USD/tháng |
| **Sustainability** | Tài nguyên tính toán chỉ khởi chạy đúng lúc có ảnh tải lên và Lifecycle tự động dọn dữ liệu không còn nhu cầu lưu trữ |

## Các quyết định thiết kế quan trọng

- **Upload thẳng lên S3 thay vì gửi qua API Gateway:** Payload đồng bộ của Lambda giới hạn tối đa 6 MB (ảnh 5 MB mã hóa base64 sẽ vượt ngưỡng) và HTTP API tính mỗi 512 KB là một request. Dùng Presigned URL cho phép trình duyệt tải thẳng 1 tệp đúng key chỉ định trong 5 phút — vừa tiết kiệm chi phí vừa an toàn.
- **Tạo bản ghi `PENDING` trước khi cấp URL tải lên:** Giúp `process-image` chỉ xử lý những ảnh đã được API hợp lệ cấp phép; mọi tệp tải chui lên S3 không có item `PENDING` đều bị bỏ qua, đồng thời giao diện có thể hiển thị trạng thái *"Đang xử lý"* ngay lập tức.
- **Dùng Lambda bất đồng bộ + DLQ thay vì chèn thêm SQS ở giữa S3 và Lambda:** Ở quy mô hiện tại, cơ chế async retry (2 lần) kèm SQS DLQ đã đủ đảm bảo không mất sự kiện mà giữ kiến trúc gọn nhẹ; chèn thêm SQS trung gian sẽ phù hợp khi lưu lượng tăng đột biến ở giai đoạn mở rộng.
- **Chỉ gửi cảnh báo SNS sau khi ghi DynamoDB thành công:** Nếu gửi email trước khi cập nhật DB, khi bước ghi DB lỗi và Lambda retry sẽ gây spam email trùng trong khi trạng thái ảnh vẫn kẹt ở `PENDING` (sự cố thực tế đã gặp và khắc phục ở mục 5.5).
- **Chỉ cấp link xem cho ảnh `APPROVED`:** Ảnh vi phạm không bao giờ được cấp Presigned GET URL hiển thị lại, và email thông báo cho quản trị viên cũng chỉ chứa `s3Key`.

## Khả năng mở rộng và giới hạn

- **Mở rộng tự động:** Lambda tự động mở rộng số lượng thực thi đồng thời khi nhiều người tải ảnh cùng lúc; S3 và DynamoDB On-demand không cần dự phòng dung lượng trước.
- **Giới hạn vận hành cần lưu ý:** Hạn mức số request/giây của Rekognition và giới hạn concurrency của Lambda trên tài khoản. Khi chạm ngưỡng, `boto3` và Lambda tự retry trước khi đẩy sự kiện lỗi vào DLQ để Alarm `throttles` và `dlq-not-empty` cảnh báo quản trị viên.
- **Bảo vệ API:** Cấu hình throttling `10 request/giây` (burst `20`) giúp chặn spam request làm tăng chi phí Lambda.
- **Hướng mở rộng:** Bổ sung SQS làm bộ đệm giữa S3 và Lambda khi lưu lượng lớn, bật AWS WAF trên CloudFront và chuyển đổi script CLI sang AWS SAM / CloudFormation.

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
