---
title: "Kiểm thử và xử lý sự cố"
date: 2026-11-20
weight: 5
chapter: false
pre: " <b> 5.5. </b> "
---

## Kiểm thử chức năng (Functional Testing)

| Kịch bản kiểm thử | Cách thực hiện | Kết quả quan sát |
|---|---|---|
| **Ảnh hợp lệ** | Tải ảnh chai bia qua giao diện web | Trạng thái `APPROVED`, gắn nhãn *Alcohol · Beer · Bottle…*, hiển thị bình thường trong gallery |
| **Ảnh vi phạm** | Tải ảnh súng đồ chơi bằng gỗ | Trạng thái `REJECTED`, `blockedCategories = Violence`, quản trị viên nhận email cảnh báo qua SNS |
| **Định danh người dùng** | Kiểm tra item trong DynamoDB | `userId` là chuỗi UUID `sub` từ Cognito, không lưu hay lộ email |
| **Tính Idempotent** | Tải đè lên cùng một S3 key đã xử lý | Log ghi nhận `item_missing_or_already_processed`, kết quả cũ được giữ nguyên |
| **Upload ngoài luồng** | Tải trực tiếp file lên S3 không có item `PENDING` | Không tạo bản ghi mới và không gửi email cảnh báo |
| **Cơ chế DLQ** | Đổi `TABLE_NAME` sai và gọi Lambda bất đồng bộ | Sau 2 lần retry thất bại, message `RetriesExhausted` (`approximateInvokeCount: 3`) xuất hiện trong SQS DLQ |

![Gallery: ảnh APPROVED và REJECTED](/fcj-workshop-template/images/5-workshop/26-3-web-gallery.png)

{{% notice note %}}
**Ví dụ về False Positive:** ảnh súng đồ chơi bằng gỗ bị Rekognition nhận diện là *Violence/Weapons* với độ tin cậy ~97%. Trong thực tế, có thể giảm tỉ lệ chặn nhầm bằng cách nâng `MIN_MODERATION_CONFIDENCE` theo từng danh mục hoặc bổ sung bước quản trị viên duyệt lại (appeal) các ảnh `REJECTED`.
{{% /notice %}}

Phép thử DLQ đồng thời kiểm chứng nguyên tắc **IAM Least Privilege**: thông báo lỗi lưu trong DLQ ghi rõ `AccessDeniedException … not authorized to perform: dynamodb:UpdateItem on resource: …table/khong-ton-tai`, chứng minh IAM role chỉ cho phép ghi vào đúng bảng `imageguard-images`.

![Message trong DLQ: RetriesExhausted](/fcj-workshop-template/images/5-workshop/15-2b-dlq-message-body.jpg)

## Kiểm thử bảo mật (Security Testing)

| Hạng mục kiểm tra | Kỳ vọng | Kết quả thực tế |
|---|---|---|
| Gọi `GET /images` không kèm JWT | `401 Unauthorized` | ✅ Trả về `{"message":"Unauthorized"}` |
| Truy cập trực tiếp URL file trên bucket web | `403 AccessDenied` | ✅ Chỉ CloudFront được phép đọc qua OAC |
| Truy cập trực tiếp S3 Object URL của ảnh | `403 AccessDenied` | ✅ Bị chặn bởi Block Public Access |
| Mở lại Presigned URL sau 6 phút | Lỗi hết hạn | ✅ Trả về `Request has expired` |
| Truy cập qua giao thức `http://` | Tự động chuyển hướng HTTPS | ✅ Redirect sang `https://` |
| Kiểm tra HTTP Response Headers | Có đầy đủ Security Headers | ✅ Có HSTS, X-Frame-Options, X-Content-Type-Options, Referrer-Policy |
| Luồng đăng nhập OAuth | Bắt buộc dùng PKCE | ✅ URL xác thực chứa `code_challenge_method=S256` |

![API không có token trả 401](/fcj-workshop-template/images/5-workshop/27-1-api-no-token.jpg)

![Truy cập thẳng S3 bị từ chối](/fcj-workshop-template/images/5-workshop/27-2-s3-web-direct-denied.jpg)

Giao diện web được CloudFront phân phối từ edge location **SGN (TP.HCM)**, giúp tốc độ tải trang tại Việt Nam nhanh dù backend triển khai tại `us-east-1`.

## Các sự cố thực tế và cách khắc phục

### 1. Lỗi *"The policy failed legacy parsing"* khi tạo Inline Policy
- **Nguyên nhân:** Dư ký tự `>` sau Account ID trong ARN của DynamoDB (ví dụ `…:125022369745>:table/…`), khiến màn hình Review chỉ nhận diện được 5/6 service.
- **Khắc phục:** Account ID phải gồm đúng 12 chữ số liền nhau, không có dấu gạch ngang hay dấu `<` `>`; nên dùng tính năng Find & Replace để thay `<ACCOUNT_ID>` trước khi dán JSON.

### 2. Email cảnh báo đã gửi nhưng trạng thái trong DynamoDB vẫn là `PENDING`
- **Nguyên nhân:** Bản code đầu tiên gửi thông báo SNS **trước khi** thực hiện `UpdateItem` có điều kiện. Khi key không khớp với bản ghi đang chờ, DynamoDB từ chối cập nhật nhưng email vẫn bị bắn ra ngoài.
- **Khắc phục:** Đảo lại thứ tự thực thi — chỉ gọi `publish_alert` khi `UpdateItem` trả về thành công; nếu việc gửi SNS gặp lỗi, ghi log `ERROR` và phát metric `AlertPublishFailed` để CloudWatch Alarm cảnh báo mà không trigger retry vòng lặp.

![Email đến dù item vẫn PENDING](/fcj-workshop-template/images/5-workshop/t-1-bug-email-before-update.png)

### 3. CloudWatch Alarm chuyển sang trạng thái `ALARM` nhưng không gửi được email
- **Hiện tượng:** Lịch sử Alarm báo *"Failed to execute action"*, dù email cảnh báo `REJECTED` từ Lambda vẫn gửi bình thường.
- **Nguyên nhân:** Topic SNS đang bật mã hóa bằng AWS-managed key `alias/aws/sns`, vốn không cho phép dịch vụ CloudWatch Alarms gọi `kms:GenerateDataKey` trực tiếp.
- **Khắc phục:** Tắt mã hóa trên SNS topic (nội dung email chỉ gồm `userId`/`imageId`, không đính kèm ảnh hay link ảnh) → lịch sử Alarm chuyển sang *"Successfully executed action"*. Nếu bắt buộc phải mã hóa, cần dùng Customer Managed KMS Key (~1 USD/tháng) có key policy cấp quyền cho `cloudwatch.amazonaws.com`.

![Failed to execute action](/fcj-workshop-template/images/5-workshop/28-3a-bug-sns-kms-failed.jpg)

![Successfully executed action sau khi sửa](/fcj-workshop-template/images/5-workshop/28-3b-fix-sns-alarm-success.jpg)

### 4. Khác biệt giao diện AWS Console mới
- Trong Lambda Console mới, mục chọn role nằm ở **Custom settings → Additional settings → Custom execution role**.
- Trong CloudWatch Console mới, **Log groups** chuyển thành **Logs → Log Management**, còn **Logs Insights** đổi tên thành **Log Analytics**.
- Trang xem log stream mặc định chỉ hiển thị vài dòng cuối; nhấn **Load more** để xem đầy đủ dòng log `image_processed`.

### 5. Không thể sử dụng Amazon Bedrock do giới hạn Quota
Kế hoạch ban đầu dự kiến sử dụng **Amazon Nova Lite** trên Amazon Bedrock, nhưng toàn bộ tài khoản trong nhóm có quota bằng 0 và yêu cầu tăng hạn mức bị từ chối. Nhóm đã chuyển sang **Amazon Rekognition** — dịch vụ AI không bị chặn quota thử nghiệm, cung cấp sẵn API kiểm duyệt nội dung và dễ dự toán chi phí theo số ảnh.
