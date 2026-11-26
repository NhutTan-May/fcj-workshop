---
title: "Hướng dẫn thực hành từng bước"
date: 2026-11-20
weight: 4
chapter: false
pre: " <b> 5.4. </b> "
---

Mã nguồn đầy đủ của toàn bộ hệ thống (3 Lambda, giao diện web tĩnh và các script tự động hóa): [static/code/imageguard](https://github.com/NhutTan-May/fcj-workshop/tree/main/static/code/imageguard). Trong quá trình thực hành, hãy thay thế `<ACCOUNT_ID>` bằng mã số tài khoản AWS gồm 12 chữ số của bạn.

## Step 1 – Khởi tạo các tài nguyên nền tảng

### 1.1. Tạo S3 bucket lưu trữ ảnh
Mở **S3 → Create bucket** và cấu hình:
- Tên bucket: `imageguard-images-<ACCOUNT_ID>`
- **Object Ownership:** ACLs disabled
- **Block Public Access:** bật toàn bộ **Block all public access**
- **Bucket Versioning:** Disable · **Encryption:** SSE-S3

Sau khi tạo xong bucket, cấu hình thêm 2 mục:
- **Management → Lifecycle rule**: tạo rule `delete-uploads-30d` áp dụng cho prefix `uploads/`, tự động xóa object sau **30 ngày**.
- **Permissions → Bucket policy**: thêm chính sách từ chối mọi request không đi qua HTTPS (`aws:SecureTransport = false`).

![S3 Block public access và bucket policy](/fcj-workshop/images/5-workshop/1-4-s3-permissions.png)

### 1.2. Tạo bảng Amazon DynamoDB
Vào **DynamoDB → Create table**:
- Table name: `imageguard-images`
- Partition key: `userId` (`String`)
- Sort key: `imageId` (`String`)
- Capacity mode: **On-demand**

### 1.3. Tạo Amazon SNS Topic và Amazon SQS DLQ
- **SNS:** tạo topic kiểu **Standard** tên `imageguard-alerts`, **tắt Encryption** (xem lý do ở mục [Xử lý sự cố](../5.5-testing/)). Tạo Email subscription trỏ về hộp thư quản trị viên và nhớ bấm **Confirm subscription** trong email xác nhận.
- **SQS:** tạo queue kiểu **Standard** tên `imageguard-process-dlq`, giữ **Message retention period = 14 ngày**, bật SSE-SQS mặc định.

### 1.4. Tạo IAM Role cho Lambda `process-image`
Mở **IAM → Roles → Create role**:
- Trusted entity: **AWS service → Lambda**
- Không đính kèm bất kỳ AWS managed policy nào.
- Đặt tên role: `imageguard-process-image-role`.
- Sau khi tạo role, chọn **Add permissions → Create inline policy** (tab JSON) và dán chính sách tối thiểu sau:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadUploadedImages",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::imageguard-images-<ACCOUNT_ID>/uploads/*"
    },
    {
      "Sid": "AnalyzeImagesWithRekognition",
      "Effect": "Allow",
      "Action": [
        "rekognition:DetectModerationLabels",
        "rekognition:DetectLabels"
      ],
      "Resource": "*"
    },
    {
      "Sid": "UpdateImageResults",
      "Effect": "Allow",
      "Action": "dynamodb:UpdateItem",
      "Resource": "arn:aws:dynamodb:us-east-1:<ACCOUNT_ID>:table/imageguard-images"
    },
    {
      "Sid": "SendModerationAlerts",
      "Effect": "Allow",
      "Action": "sns:Publish",
      "Resource": "arn:aws:sns:us-east-1:<ACCOUNT_ID>:imageguard-alerts"
    },
    {
      "Sid": "SendFailedEventsToDLQ",
      "Effect": "Allow",
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:us-east-1:<ACCOUNT_ID>:imageguard-process-dlq"
    },
    {
      "Sid": "WriteOwnLogsOnly",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": [
        "arn:aws:logs:us-east-1:<ACCOUNT_ID>:log-group:/aws/lambda/imageguard-process-image",
        "arn:aws:logs:us-east-1:<ACCOUNT_ID>:log-group:/aws/lambda/imageguard-process-image:*"
      ]
    }
  ]
}
```

Kiểm tra trang Review: phải hiển thị đủ **6 service** (S3, Rekognition, DynamoDB, SNS, SQS, CloudWatch Logs). Nếu gặp lỗi *"The policy failed legacy parsing"*, hãy kiểm tra lại định dạng `<ACCOUNT_ID>` (xem mục 5.5).

![Role chỉ có 1 inline policy](/fcj-workshop/images/5-workshop/5-4-iam-role-summary.png)

## Step 2 – Triển khai Lambda `process-image` và S3 Event Trigger

### 2.1. Tạo và cấu hình hàm Lambda
Vào **Lambda → Create function → Author from scratch**:
- Function name: `imageguard-process-image`
- Runtime: **Python 3.12** · Architecture: **arm64**
- **Additional settings → Custom execution role:** chọn `imageguard-process-image-role` · tắt Function URL.

![Tạo Lambda process-image](/fcj-workshop/images/5-workshop/6-1-lambda-create.png)

Dán mã nguồn từ [`process_image/lambda_function.py`](https://github.com/NhutTan-May/fcj-workshop/blob/main/static/code/imageguard/process_image/lambda_function.py) rồi nhấn **Deploy**. Ở tab **Configuration → General configuration**, đặt **Memory = 256 MB** và **Timeout = 30 giây**, sau đó thêm các biến môi trường:

| Biến môi trường | Giá trị |
|---|---|
| `TABLE_NAME` | `imageguard-images` |
| `BUCKET_NAME` | `imageguard-images-<ACCOUNT_ID>` |
| `TOPIC_ARN` | `arn:aws:sns:us-east-1:<ACCOUNT_ID>:imageguard-alerts` |
| `BLOCKED_CATEGORIES` | `Explicit,Violence,Visually Disturbing,Drugs & Tobacco,Hate Symbols` |
| `MIN_MODERATION_CONFIDENCE` | `60` |
| `MAX_LABELS` | `10` |
| `MIN_LABEL_CONFIDENCE` | `80` |

### 2.2. Các điểm kỹ thuật quan trọng trong mã nguồn

```python
# Tu dong retry theo exponential backoff khi gap throttling hoac loi 5xx
BOTO_CONFIG = Config(retries={"max_attempts": 5, "mode": "standard"})

# Chi chap nhan key dung cau truc uploads/{userId}/{imageId}.{ext} de chan path traversal
KEY_PATTERN = re.compile(r"^uploads/([A-Za-z0-9-]{1,64})/([A-Za-z0-9-]{1,64})\.(jpg|jpeg|png)$")

# UpdateItem co dieu kien: item phai ton tai va van o trang thai PENDING (idempotent)
ConditionExpression="attribute_exists(#uid) AND #st = :pending"

# Chi gui email SNS sau khi cap nhat DynamoDB thanh cong
if blocked and updated:
    publish_alert(user_id, image_id, key, blocked)
```

Hệ thống ghi metric thông qua **CloudWatch Embedded Metric Format (EMF)**: hàm in một dòng JSON có trường `_aws` vào stdout log để CloudWatch tự động trích xuất các metric `ImagesProcessed`, `ImagesApproved`, `ImagesRejected`, `ProcessingLatency` và `AlertPublishFailed` thuộc namespace `ImageGuard` mà không cần cấp quyền `cloudwatch:PutMetricData`.

### 2.3. Cấu hình Retry, Dead-Letter Queue và S3 Trigger
- Trong **Configuration → Asynchronous invocation**: đặt Maximum age of event = **1 giờ**, Retry attempts = **2**.
- Trong **Destinations → Add destination**: chọn **On failure** → **SQS queue** `imageguard-process-dlq`.
- Chọn **Add trigger → S3**: trỏ tới `imageguard-images-<ACCOUNT_ID>`, Event type **PUT**, Prefix `uploads/`, Suffix `.jpg`.

![Function overview: S3 → Lambda → SQS](/fcj-workshop/images/5-workshop/11-2-lambda-overview.png)

## Step 3 – Thiết lập Cognito, API Gateway và Frontend tĩnh

Bạn có thể cấu hình thủ công theo từng bước dưới đây hoặc chạy script [`setup.sh`](https://github.com/NhutTan-May/fcj-workshop/blob/main/static/code/imageguard/scripts/setup.sh) trực tiếp trong **AWS CloudShell** để tự động khởi tạo toàn bộ phần này trong khoảng 3 phút.

![Kết quả chạy setup.sh trong CloudShell](/fcj-workshop/images/5-workshop/23-0-cloudshell-setup-output.jpg)

### 3.1. Tạo hai hàm Lambda phục vụ API
Tạo 2 IAM role riêng theo nguyên tắc quyền tối thiểu, sau đó tạo 2 hàm Lambda **Python 3.12 (arm64)** với cấu hình **128 MB RAM**, **Timeout 10 giây**:

| Tên Lambda | Quyền trong Inline Policy | Biến môi trường |
|---|---|---|
| `imageguard-presign-upload` | `s3:PutObject` (`uploads/*`), `dynamodb:PutItem`, ghi log của chính nó | `BUCKET_NAME`, `TABLE_NAME`, `URL_EXPIRES_SECONDS=300` |
| `imageguard-get-images` | `dynamodb:Query`, `s3:GetObject` (`uploads/*`), ghi log của chính nó | `BUCKET_NAME`, `TABLE_NAME`, `URL_EXPIRES_SECONDS=300`, `MAX_ITEMS=50` |

Hàm `presign-upload` đọc `userId` từ `requestContext.authorizer.jwt.claims.sub`, tự sinh `imageId` (UUIDv4), ghi trước bản ghi `PENDING` vào DynamoDB rồi mới ký Presigned PUT URL. Vì link upload được ký bằng IAM Role của chính Lambda nên role này bắt buộc phải có quyền `s3:PutObject`.

### 3.2. Triển khai Website tĩnh với S3 + CloudFront (OAC)
1. Tạo bucket private `imageguard-web-<ACCOUNT_ID>` (bật Block Public Access, SSE-S3) và tải lên `index.html`, `app.js`, `config.js` từ [`static/code/imageguard/web`](https://github.com/NhutTan-May/fcj-workshop/tree/main/static/code/imageguard/web).
2. Vào **CloudFront → Create distribution**: chọn origin là bucket `imageguard-web-<ACCOUNT_ID>`, cấu hình **Origin Access Control (OAC)**, **Redirect HTTP to HTTPS**, cache policy `CachingOptimized`, response headers `SecurityHeadersPolicy`, Default root object `index.html`, không bật WAF.
3. Cập nhật Bucket Policy cho `imageguard-web-<ACCOUNT_ID>` để chỉ cho phép `cloudfront.amazonaws.com` truy cập với điều kiện `AWS:SourceArn` khớp ARN của distribution.

![Trang ImageGuard qua CloudFront](/fcj-workshop/images/5-workshop/21-4-web-first-load.jpg)

### 3.3. Cấu hình Amazon Cognito
Tạo User Pool `imageguard-users` hỗ trợ đăng nhập bằng **email** và cho phép người dùng tự đăng ký (self sign-up). Tạo App Client dạng **SPA (không tạo client secret)** với:
- Grant type: **Authorization code grant**
- OAuth scopes: `openid`, `email`
- Callback URL và Sign-out URL: `https://<CLOUDFRONT_DOMAIN>/`
- Khởi tạo Cognito Domain để bật giao diện **Managed Login**.

![Trang đăng nhập Cognito Managed Login](/fcj-workshop/images/5-workshop/26-0-cognito-managed-login.jpg)

### 3.4. Cấu hình Amazon API Gateway (HTTP API)
- Tạo HTTP API `imageguard-api` với stage `$default` (bật auto-deploy) và hai route: `POST /upload-url` → `presign-upload`, `GET /images` → `get-images`.
- Tạo **JWT Authorizer** `cognito-jwt` với Issuer `https://cognito-idp.us-east-1.amazonaws.com/<USER_POOL_ID>` và Audience `<APP_CLIENT_ID>`, sau đó gắn cho cả hai route.
- Cấu hình **CORS**: chỉ cho phép origin `https://<CLOUDFRONT_DOMAIN>`, header `authorization, content-type`, method `GET, POST, OPTIONS`.
- Bật **Throttling** trên default route: `rate = 10`, `burst = 20`.

### 3.5. Kết nối các thành phần
- Thêm cấu hình **CORS** cho bucket ảnh `imageguard-images-<ACCOUNT_ID>` cho phép method `PUT, GET` từ domain CloudFront.
- Điền đầy đủ thông số (`Cognito domain`, `client ID`, `redirect URI`, `API Gateway URL`) vào file `config.js`, tải lại lên bucket web và tạo **CloudFront Invalidation** `/*`.

## Step 4 – Thiết lập Giám sát trên CloudWatch

Chạy script [`step4.sh`](https://github.com/NhutTan-May/fcj-workshop/blob/main/static/code/imageguard/scripts/step4.sh) trong CloudShell hoặc tạo thủ công 7 CloudWatch Alarm sau trên Console:

| Tên Alarm | Metric | Ngưỡng kích hoạt (chu kỳ 5 phút) |
|---|---|---|
| `imageguard-{process-image,presign-upload,get-images}-errors` | `AWS/Lambda` → `Errors` | ≥ 1 |
| `imageguard-process-image-throttles` | `AWS/Lambda` → `Throttles` | ≥ 1 |
| `imageguard-dlq-not-empty` | `AWS/SQS` → `ApproximateNumberOfMessagesVisible` | ≥ 1 |
| `imageguard-alert-publish-failed` | `ImageGuard` → `AlertPublishFailed` | ≥ 1 |
| `imageguard-api-5xx` | `AWS/ApiGateway` → `5xx` | ≥ 5 |

Tất cả Alarm đều gửi thông báo về SNS topic `imageguard-alerts` ở cả hai trạng thái **ALARM** và **OK**. Dashboard `imageguard` tổng hợp số lượng ảnh `APPROVED`/`REJECTED`, thời gian xử lý trung bình, lỗi Lambda, lỗi API 5xx, số lượng message trong DLQ và trạng thái Alarm. Cuối cùng, cấu hình **Log retention = 14 ngày** cho cả 3 Log Group của Lambda.

![CloudWatch dashboard imageguard](/fcj-workshop/images/5-workshop/28-4-dashboard.png)
