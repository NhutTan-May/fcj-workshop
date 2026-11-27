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

![Hình 1.1 – Đặt tên bucket, chọn ACLs disabled](/fcj-workshop/images/5-workshop/1-1-s3-create-general.png)
*Hình 1.1 – Đặt tên bucket, chọn ACLs disabled*

![Hình 1.2 – Block all public access, tắt Versioning, mã hóa SSE-S3](/fcj-workshop/images/5-workshop/1-2-s3-create-security.png)
*Hình 1.2 – Block all public access, tắt Versioning, mã hóa SSE-S3*

![Hình 1.3 – Lifecycle rule xóa ảnh trong uploads/ sau 30 ngày](/fcj-workshop/images/5-workshop/1-3-s3-lifecycle.png)
*Hình 1.3 – Lifecycle rule xóa ảnh trong uploads/ sau 30 ngày*

![Hình 1.4 – Tab Permissions: Block public access On và bucket policy bắt HTTPS](/fcj-workshop/images/5-workshop/1-4-s3-permissions.png)
*Hình 1.4 – Tab Permissions: Block public access On và bucket policy bắt HTTPS*

### 1.2. Tạo bảng Amazon DynamoDB
Vào **DynamoDB → Create table**:
- Table name: `imageguard-images`
- Partition key: `userId` (`String`)
- Sort key: `imageId` (`String`)
- Capacity mode: **On-demand**

![Hình 1.5 – Bảng imageguard-images: userId / imageId, chế độ On-demand](/fcj-workshop/images/5-workshop/2-1-dynamodb.png)
*Hình 1.5 – Bảng imageguard-images: userId / imageId, chế độ On-demand*

### 1.3. Tạo Amazon SNS Topic và Amazon SQS DLQ
- **SNS:** tạo topic kiểu **Standard** tên `imageguard-alerts`, **tắt Encryption** (xem lý do ở mục [Xử lý sự cố](../5.5-testing/)). Tạo Email subscription trỏ về hộp thư quản trị viên và nhớ bấm **Confirm subscription** trong email xác nhận.
- **SQS:** tạo queue kiểu **Standard** tên `imageguard-process-dlq`, giữ **Message retention period = 14 ngày**, bật SSE-SQS mặc định.

![Hình 1.6 – Tạo SNS topic Standard imageguard-alerts](/fcj-workshop/images/5-workshop/3-1-sns-create-topic.png)
*Hình 1.6 – Tạo SNS topic Standard imageguard-alerts*

![Hình 1.7 – Email xác nhận đăng ký nhận cảnh báo](/fcj-workshop/images/5-workshop/3-2-sns-confirm-email.png)
*Hình 1.7 – Email xác nhận đăng ký nhận cảnh báo*

![Hình 1.8 – Subscription ở trạng thái Confirmed](/fcj-workshop/images/5-workshop/3-3-sns-confirmed.png)
*Hình 1.8 – Subscription ở trạng thái Confirmed*

![Hình 1.9 – Tạo SQS Standard imageguard-process-dlq, giữ message 14 ngày](/fcj-workshop/images/5-workshop/4-1-sqs-create.png)
*Hình 1.9 – Tạo SQS Standard imageguard-process-dlq, giữ message 14 ngày*

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

![Hình 1.10 – Trusted entity: AWS service Lambda](/fcj-workshop/images/5-workshop/5-1-iam-role-trusted.png)
*Hình 1.10 – Trusted entity: AWS service Lambda*

![Hình 1.11 – Không gắn managed policy nào](/fcj-workshop/images/5-workshop/5-2-iam-role-no-managed.png)
*Hình 1.11 – Không gắn managed policy nào*

![Hình 1.12 – Inline policy dạng JSON](/fcj-workshop/images/5-workshop/5-3-iam-inline-policy.png)
*Hình 1.12 – Inline policy dạng JSON*

![Hình 1.13 – Role chỉ có đúng 1 inline policy](/fcj-workshop/images/5-workshop/5-4-iam-role-summary.png)
*Hình 1.13 – Role chỉ có đúng 1 inline policy*

## Step 2 – Triển khai Lambda `process-image` và S3 Event Trigger

### 2.1. Tạo và cấu hình hàm Lambda
Vào **Lambda → Create function → Author from scratch**:
- Function name: `imageguard-process-image`
- Runtime: **Python 3.12** · Architecture: **arm64**
- **Additional settings → Custom execution role:** chọn `imageguard-process-image-role` · tắt Function URL.

![Hình 2.1 – Tạo Lambda: Python 3.12, ARM64, dùng role có sẵn, tắt Function URL](/fcj-workshop/images/5-workshop/6-1-lambda-create.png)
*Hình 2.1 – Tạo Lambda: Python 3.12, ARM64, dùng role có sẵn, tắt Function URL*

Dán mã nguồn từ [`process_image/lambda_function.py`](https://github.com/NhutTan-May/fcj-workshop/blob/main/static/code/imageguard/process_image/lambda_function.py) rồi nhấn **Deploy**. Ở tab **Configuration → General configuration**, đặt **Memory = 256 MB** và **Timeout = 30 giây**, sau đó thêm các biến môi trường:

![Hình 2.2 – Dán code và Deploy thành công](/fcj-workshop/images/5-workshop/7-1-lambda-code-deployed.png)
*Hình 2.2 – Dán code và Deploy thành công*

![Hình 2.3 – Memory 256 MB, Timeout 30 giây](/fcj-workshop/images/5-workshop/8-1-lambda-general-config.png)
*Hình 2.3 – Memory 256 MB, Timeout 30 giây*

| Biến môi trường | Giá trị |
|---|---|
| `TABLE_NAME` | `imageguard-images` |
| `BUCKET_NAME` | `imageguard-images-<ACCOUNT_ID>` |
| `TOPIC_ARN` | `arn:aws:sns:us-east-1:<ACCOUNT_ID>:imageguard-alerts` |
| `BLOCKED_CATEGORIES` | `Explicit,Violence,Visually Disturbing,Drugs & Tobacco,Hate Symbols` |
| `MIN_MODERATION_CONFIDENCE` | `60` |
| `MAX_LABELS` | `10` |
| `MIN_LABEL_CONFIDENCE` | `80` |

![Hình 2.4 – 7 biến môi trường của process-image](/fcj-workshop/images/5-workshop/9-1-lambda-env-vars.png)
*Hình 2.4 – 7 biến môi trường của process-image*

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

![Hình 2.5 – Asynchronous invocation: tối đa 1 giờ, thử lại 2 lần](/fcj-workshop/images/5-workshop/10-1-lambda-async-config.png)
*Hình 2.5 – Asynchronous invocation: tối đa 1 giờ, thử lại 2 lần*

![Hình 2.6 – Destination On failure → SQS DLQ](/fcj-workshop/images/5-workshop/10-2-lambda-destination.png)
*Hình 2.6 – Destination On failure → SQS DLQ*

![Hình 2.7 – S3 trigger: event PUT, prefix uploads/, suffix .jpg](/fcj-workshop/images/5-workshop/11-1-s3-trigger-config.png)
*Hình 2.7 – S3 trigger: event PUT, prefix uploads/, suffix .jpg*

![Hình 2.8 – Function overview: S3 → Lambda → SQS](/fcj-workshop/images/5-workshop/11-2-lambda-overview.png)
*Hình 2.8 – Function overview: S3 → Lambda → SQS*

![Hình 2.9 – Resource-based policy có aws:SourceArn và aws:SourceAccount (chống confused deputy)](/fcj-workshop/images/5-workshop/11-3-lambda-resource-policy.png)
*Hình 2.9 – Resource-based policy có aws:SourceArn và aws:SourceAccount (chống confused deputy)*

### 2.4. Kiểm tra nhanh pipeline xử lý (trước khi dựng Web)
1. Trong **DynamoDB → Explore items → Create item**, tạo hai bản ghi `PENDING`: `userId` = `test-user-001`, `imageId` = `img-001` (và `img-002`) kèm thuộc tính String `status` = `PENDING`.
2. Trên **S3**, tạo thư mục `uploads/test-user-001/` và tải ảnh chai bia dưới tên `img-001.jpg`, ảnh súng đồ chơi dưới tên `img-002.jpg`.
3. Chờ khoảng 10 giây rồi nhấn **Explore items → Run** để kiểm tra kết quả cập nhật.

![Hình 2.10 – Các item thử nghiệm ở trạng thái PENDING](/fcj-workshop/images/5-workshop/12-1-dynamodb-pending.png)
*Hình 2.10 – Các item thử nghiệm ở trạng thái PENDING*

![Hình 2.11 – Ảnh thử nghiệm trong uploads/test-user-001/](/fcj-workshop/images/5-workshop/13-1-s3-uploaded.png)
*Hình 2.11 – Ảnh thử nghiệm trong uploads/test-user-001/*

![Hình 2.12 – Kết quả: img-001 APPROVED, img-002 REJECTED](/fcj-workshop/images/5-workshop/14-1-dynamodb-result.png)
*Hình 2.12 – Kết quả: img-001 APPROVED, img-002 REJECTED*

![Hình 2.13 – Ảnh hợp lệ: danh sách nhãn (labels)](/fcj-workshop/images/5-workshop/14-2-item-approved-labels.png)
*Hình 2.13 – Ảnh hợp lệ: danh sách nhãn (labels)*

![Hình 2.14 – Ảnh vi phạm: blockedCategories = Violence](/fcj-workshop/images/5-workshop/14-3-item-rejected.png)
*Hình 2.14 – Ảnh vi phạm: blockedCategories = Violence*

![Hình 2.15 – Email cảnh báo quản trị viên nhận được](/fcj-workshop/images/5-workshop/14-4-sns-alert-email.png)
*Hình 2.15 – Email cảnh báo quản trị viên nhận được*

![Hình 2.16 – Log image_processed trong CloudWatch Logs](/fcj-workshop/images/5-workshop/14-5-cloudwatch-log.png)
*Hình 2.16 – Log image_processed trong CloudWatch Logs*

### 2.5. Kiểm thử cơ chế Dead-Letter Queue (cố ý tạo lỗi)
1. Tạm đổi biến môi trường `TABLE_NAME` thành một tên bảng không tồn tại (ví dụ `khong-ton-tai`).
2. Vào tab **Test**, chọn Invocation type **Asynchronous** và gửi một sự kiện S3 mẫu trỏ tới ảnh có sẵn.
3. Chờ khoảng 3 phút (1 lần chạy đầu + 2 lần retry) rồi vào **SQS → `imageguard-process-dlq` → Poll for messages**.
4. Khôi phục `TABLE_NAME` về lại `imageguard-images`.

![Hình 2.17 – Tạm đổi TABLE_NAME để gây lỗi](/fcj-workshop/images/5-workshop/15-0-env-table-name-wrong.jpg)
*Hình 2.17 – Tạm đổi TABLE_NAME để gây lỗi*

![Hình 2.18 – Gọi Lambda bất đồng bộ từ tab Test](/fcj-workshop/images/5-workshop/15-1-lambda-async-invoke.jpg)
*Hình 2.18 – Gọi Lambda bất đồng bộ từ tab Test*

![Hình 2.19 – DLQ nhận 1 message sau khi hết lượt thử lại](/fcj-workshop/images/5-workshop/15-2a-dlq-messages-list.jpg)
*Hình 2.19 – DLQ nhận 1 message sau khi hết lượt thử lại*

![Hình 2.20 – Trả TABLE_NAME về giá trị đúng](/fcj-workshop/images/5-workshop/15-3-env-restored.jpg)
*Hình 2.20 – Trả TABLE_NAME về giá trị đúng*

Chi tiết nội dung message trong DLQ được phân tích ở [mục 5.5](../5.5-testing/).

## Step 3 – Thiết lập Cognito, API Gateway và Frontend tĩnh

Bạn có thể cấu hình thủ công theo từng bước dưới đây hoặc chạy script [`setup.sh`](https://github.com/NhutTan-May/fcj-workshop/blob/main/static/code/imageguard/scripts/setup.sh) trực tiếp trong **AWS CloudShell** để tự động khởi tạo toàn bộ phần này trong khoảng 3 phút.

![Hình 3.1 – Kết quả chạy setup.sh trong CloudShell](/fcj-workshop/images/5-workshop/23-0-cloudshell-setup-output.jpg)
*Hình 3.1 – Kết quả chạy setup.sh trong CloudShell*

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

![Hình 3.2 – Trang ImageGuard qua CloudFront](/fcj-workshop/images/5-workshop/21-5-web-first-load-v2.jpg)
*Hình 3.2 – Trang ImageGuard qua CloudFront*

### 3.3. Cấu hình Amazon Cognito
Tạo User Pool `imageguard-users` hỗ trợ đăng nhập bằng **email** và cho phép người dùng tự đăng ký (self sign-up). Tạo App Client dạng **SPA (không tạo client secret)** với:
- Grant type: **Authorization code grant**
- OAuth scopes: `openid`, `email`
- Callback URL và Sign-out URL: `https://<CLOUDFRONT_DOMAIN>/`
- Khởi tạo Cognito Domain để bật giao diện **Managed Login**.

![Hình 3.3 – Trang đăng nhập Cognito Managed Login](/fcj-workshop/images/5-workshop/26-0-cognito-managed-login.jpg)
*Hình 3.3 – Trang đăng nhập Cognito Managed Login*

### 3.4. Cấu hình Amazon API Gateway (HTTP API)
- Tạo HTTP API `imageguard-api` với stage `$default` (bật auto-deploy) và hai route: `POST /upload-url` → `presign-upload`, `GET /images` → `get-images`.
- Tạo **JWT Authorizer** `cognito-jwt` với Issuer `https://cognito-idp.us-east-1.amazonaws.com/<USER_POOL_ID>` và Audience `<APP_CLIENT_ID>`, sau đó gắn cho cả hai route.
- Cấu hình **CORS**: chỉ cho phép origin `https://<CLOUDFRONT_DOMAIN>`, header `authorization, content-type`, method `GET, POST, OPTIONS`.
- Bật **Throttling** trên default route: `rate = 10`, `burst = 20`.

### 3.5. Kết nối các thành phần
- Thêm cấu hình **CORS** cho bucket ảnh `imageguard-images-<ACCOUNT_ID>` cho phép method `PUT, GET` từ domain CloudFront.
- Điền đầy đủ thông số (`Cognito domain`, `client ID`, `redirect URI`, `API Gateway URL`) vào file `config.js`, tải lại lên bucket web và tạo **CloudFront Invalidation** `/*`.

![Hình 3.4 – Kết quả cuối: người dùng đăng nhập, tải ảnh lên và xem trạng thái APPROVED / REJECTED](/fcj-workshop/images/5-workshop/26-4-web-gallery-v2.jpg)
*Hình 3.4 – Kết quả cuối: người dùng đăng nhập, tải ảnh lên và xem trạng thái APPROVED / REJECTED*

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

![Hình 4.1 – Tạo 7 alarm và dashboard bằng step4.sh](/fcj-workshop/images/5-workshop/28-1-step4-alarms-created.jpg)
*Hình 4.1 – Tạo 7 alarm và dashboard bằng step4.sh*

![Hình 4.2 – Trạng thái alarm: DLQ ALARM (còn message thử nghiệm), các alarm khác OK](/fcj-workshop/images/5-workshop/28-2-step4-alarm-states.jpg)
*Hình 4.2 – Trạng thái alarm: DLQ ALARM (còn message thử nghiệm), các alarm khác OK*

![Hình 4.3 – Custom metric (EMF) trong namespace ImageGuard](/fcj-workshop/images/5-workshop/14-6-cloudwatch-metrics.png)
*Hình 4.3 – Custom metric (EMF) trong namespace ImageGuard*

![Hình 4.4 – Log group giữ log 14 ngày](/fcj-workshop/images/5-workshop/14-7-log-retention.png)
*Hình 4.4 – Log group giữ log 14 ngày*

![Hình 4.5 – CloudWatch dashboard imageguard](/fcj-workshop/images/5-workshop/28-4-dashboard.png)
*Hình 4.5 – CloudWatch dashboard imageguard*
