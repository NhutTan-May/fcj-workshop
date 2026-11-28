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

![Gallery: ảnh APPROVED và REJECTED](/fcj-workshop/images/5-workshop/26-6-web-library-dark.jpg)

{{% notice note %}}
**Ví dụ về False Positive:** ảnh súng đồ chơi bằng gỗ bị Rekognition nhận diện là *Violence/Weapons* với độ tin cậy ~97%. Trong thực tế, có thể giảm tỉ lệ chặn nhầm bằng cách nâng `MIN_MODERATION_CONFIDENCE` theo từng danh mục hoặc bổ sung bước quản trị viên duyệt lại (appeal) các ảnh `REJECTED`.
{{% /notice %}}

Phép thử DLQ đồng thời kiểm chứng nguyên tắc **IAM Least Privilege**: thông báo lỗi lưu trong DLQ ghi rõ `AccessDeniedException … not authorized to perform: dynamodb:UpdateItem on resource: …table/khong-ton-tai`, chứng minh IAM role chỉ cho phép ghi vào đúng bảng `imageguard-images`.

![Message trong DLQ: RetriesExhausted](/fcj-workshop/images/5-workshop/15-2b-dlq-message-body.jpg)

## Đối chiếu tiêu chí hoàn thành (Proposal mục 3.2)

| Tiêu chí | Mục tiêu | Kết quả đo thực tế |
|---|---|---|
| Thời gian xử lý 1 ảnh | ≤ 10 giây | ✅ Khoảng **0,5–1 giây** (đo trên metric `ProcessingLatency` của Dashboard) |
| Ảnh vi phạm bị chặn đúng | ≥ 90% | ❌ **2/3 (67%)**: chặn đúng 2 ảnh vũ khí, bỏ sót 1 ảnh hút thuốc |
| Ảnh hợp lệ bị chặn nhầm | ≤ 10% | ❌ **1/8 (12,5%)**: súng gỗ đồ chơi bị chặn (`Violence/Weapons` 98%), trong khi súng nước đồ chơi được duyệt |
| Gọi API khi chưa đăng nhập | Bị từ chối (401) | ✅ `{"message":"Unauthorized"}` |
| Người dùng A xem ảnh của B | Không thể | ✅ Gallery chỉ trả về ảnh của đúng `sub` đang đăng nhập, dù bảng chứa cả dữ liệu `test-user-001` |
| Phát hiện ảnh vi phạm | Gửi email quản trị viên | ✅ Đã nhận email cảnh báo từ SNS |
| Truy cập trực tiếp bucket S3 | Bị chặn | ✅ `403 AccessDenied` |

Hai tỉ lệ độ chính xác trên được đo trên **12 ảnh** (gồm 2 ảnh thử ban đầu và 10 ảnh lấy từ Wikimedia Commons) chạy qua pipeline thực tế bằng script [`batch_test.sh`](/fcj-workshop/code/imageguard/scripts/batch_test.sh) vào ngày **26/11/2026**:

| Nhóm | Ảnh kiểm thử | Kỳ vọng | Kết quả thực tế | Nhãn / Danh mục ghi nhận |
|---|---|---|---|---|
| Thường | Laptop trên bàn | APPROVED | ✅ APPROVED | `Furniture`, `Table`, `Desk`, `Computer` |
| Thường | Xe đạp | APPROVED | ✅ APPROVED | `Wheel`, `Bicycle`, `Transportation` |
| Thường | Pizza | APPROVED | ✅ APPROVED | `Food`, `Meal`, `Pizza` |
| Thường | Mèo | APPROVED | ✅ APPROVED | `Animal`, `Abyssinian`, `Cat` |
| Rượu bia | Chai bia | APPROVED (`Alcohol` được cho qua) | ✅ APPROVED | `Alcohol`, `Beer`, `Bottle` |
| Thuốc lá | Người hút thuốc cầm cuốc | REJECTED | ❌ APPROVED | Moderation chỉ đạt `Smoking` 27% (< ngưỡng 60%); `DetectLabels` nhận nhầm cái cuốc là `Rifle` |
| Vũ khí | Súng ngắn | REJECTED | ✅ REJECTED | `Violence` → `Weapons` |
| Vũ khí | Lính cầm súng trường | REJECTED | ✅ REJECTED | `Violence` → `Weapons`, `Weapon Violence` |
| Dễ nhầm | Súng nước đồ chơi | APPROVED | ✅ APPROVED | `Toy`, `Water Gun` |
| Dễ nhầm | Hóa trang máu giả Halloween | Tùy chính sách | REJECTED | `Violence` → `Blood & Gore` (không tính vào 2 tỉ lệ trên) |

**Nhận xét và hướng cải thiện:**
- Ngưỡng chung 60% phát hiện tốt vũ khí rõ nét, nhưng bỏ sót hành vi hút thuốc khi điếu thuốc chiếm diện tích nhỏ trong khung hình. Có thể tách ngưỡng riêng theo từng danh mục (ví dụ hạ xuống 25% cho `Drugs & Tobacco`) và chấp nhận đánh đổi thêm một ít false positive.
- Đồ chơi có hình dáng giống vũ khí hoặc hóa trang máu giả rất dễ gây phân loại sai; do đó nên bổ sung hàng đợi kiểm duyệt lại bởi quản trị viên đối với các ảnh `REJECTED` thay vì chặn cứng.
- Tập 12 ảnh giúp nhìn ra xu hướng hành vi của mô hình trong thực tế nhưng chưa đủ lớn để kết luận độ chính xác thống kê.

## Kiểm thử bảo mật (Security Testing)

| Hạng mục kiểm tra | Kỳ vọng | Kết quả thực tế |
|---|---|---|
| Gọi `GET /images` không kèm JWT | `401 Unauthorized` | ✅ Trả về `{"message":"Unauthorized"}` |
| Truy cập trực tiếp URL file trên bucket web | `403 AccessDenied` | ✅ Chỉ CloudFront được phép đọc qua OAC |
| Truy cập trực tiếp S3 Object URL của ảnh | `403 AccessDenied` | ✅ Bị chặn bởi Block Public Access |
| Mở lại Presigned URL sau 6 phút | Lỗi hết hạn | ✅ Trả về `200` khi còn hạn và `403 Request has expired` sau khi hết hạn |
| Truy cập qua giao thức `http://` | Tự động chuyển hướng HTTPS | ✅ Trả về `301 Moved Permanently` sang `https://` |
| Kiểm tra HTTP Response Headers | Có đầy đủ Security Headers | ✅ Có HSTS, X-Frame-Options, X-Content-Type-Options, Referrer-Policy |
| Luồng đăng nhập OAuth | Bắt buộc dùng PKCE | ✅ URL xác thực chứa `code_challenge_method=S256` |

![API không có token trả 401](/fcj-workshop/images/5-workshop/27-1-api-no-token.jpg)

![Truy cập thẳng S3 web bị từ chối](/fcj-workshop/images/5-workshop/27-2-s3-web-direct-denied.jpg)

![Mở thẳng Object URL ảnh bị từ chối](/fcj-workshop/images/5-workshop/27-3-s3-image-direct-denied.jpg)

![HTTP chuyển HTTPS và presigned URL hết hạn](/fcj-workshop/images/5-workshop/27-4-https-redirect-presigned-expired.jpg)

Giao diện web được CloudFront phân phối từ edge location **SGN (TP.HCM)**, giúp tốc độ tải trang tại Việt Nam nhanh dù backend triển khai tại `us-east-1`.

## Kiểm thử kịch bản lỗi (Error-Case Testing)

Chạy script [`error_test.sh`](/fcj-workshop/code/imageguard/scripts/error_test.sh) trong CloudShell ngày **28/11/2026**. Các bài test `E1`–`E3` gọi trực tiếp Lambda `imageguard-presign-upload` qua `aws lambda invoke` với đầu vào không hợp lệ; `E4` gửi đồng thời 100 request tới API; `E5` tải lên một tệp văn bản đổi phần mở rộng thành `.jpg`.

| # | Kịch bản kiểm thử | Kết quả kỳ vọng | Kết quả quan sát |
|---|---|---|---|
| `E1` | Yêu cầu upload URL với `contentType: image/png` | Từ chối `400` | ✅ `400` – *"Chi chap nhan image/jpeg"* |
| `E2` | Request body không phải JSON hợp lệ | Từ chối `400` | ✅ `400` – *"Body phai la JSON"* |
| `E3` | Không có JWT (thiếu claim `sub`) | Từ chối `401` | ✅ `401` – *"Unauthorized"* |
| `E4` | Gửi song song 100 request `GET /images` không có token | Chặn tại cổng, không gọi Lambda | ✅ 100/100 trả `401` ngay tại JWT Authorizer, Lambda không bị invoke nên không tốn chi phí (ngưỡng throttling 10 req/s, burst 20 chưa kích hoạt trong kịch bản không token này) |
| `E5` | Tệp văn bản đổi tên thành `.jpg` | Không treo `PENDING`, Lambda không lỗi | ✅ Item chuyển `REJECTED` với lý do `INVALID_IMAGE:InvalidImageFormatException`; Lambda kết thúc bình thường và không đẩy vào DLQ |

![Kết quả 5 test kịch bản lỗi trong CloudShell](/fcj-workshop/images/5-workshop/30-1-error-cases.jpg)
*Hình: Kết quả chạy `error_test.sh` – E1, E2 trả 400; E3 và E4 trả 401; E5 tệp không hợp lệ bị đánh dấu REJECTED*

## Các sự cố thực tế và cách khắc phục

### 1. Lỗi *"The policy failed legacy parsing"* khi tạo Inline Policy
- **Nguyên nhân:** Dư ký tự `>` sau Account ID trong ARN của DynamoDB (ví dụ `…:125022369745>:table/…`), khiến màn hình Review chỉ nhận diện được 5/6 service.
- **Khắc phục:** Account ID phải gồm đúng 12 chữ số liền nhau, không có dấu gạch ngang hay dấu `<` `>`; nên dùng tính năng Find & Replace để thay `<ACCOUNT_ID>` trước khi dán JSON.

### 2. Email cảnh báo đã gửi nhưng trạng thái trong DynamoDB vẫn là `PENDING`
- **Nguyên nhân:** Bản code đầu tiên gửi thông báo SNS **trước khi** thực hiện `UpdateItem` có điều kiện. Khi key không khớp với bản ghi đang chờ, DynamoDB từ chối cập nhật nhưng email vẫn bị bắn ra ngoài.
- **Khắc phục:** Đảo lại thứ tự thực thi — chỉ gọi `publish_alert` khi `UpdateItem` trả về thành công; nếu việc gửi SNS gặp lỗi, ghi log `ERROR` và phát metric `AlertPublishFailed` để CloudWatch Alarm cảnh báo mà không trigger retry vòng lặp.

![Email cảnh báo vẫn được gửi…](/fcj-workshop/images/5-workshop/t-1-bug-email-before-update.png)
*Email cảnh báo vẫn được gửi…*

![…trong khi item vẫn ở trạng thái PENDING](/fcj-workshop/images/5-workshop/t-2-bug-item-still-pending.png)
*…trong khi item vẫn ở trạng thái PENDING*

### 3. CloudWatch Alarm chuyển sang trạng thái `ALARM` nhưng không gửi được email
- **Hiện tượng:** Lịch sử Alarm báo *"Failed to execute action"*, dù email cảnh báo `REJECTED` từ Lambda vẫn gửi bình thường.
- **Nguyên nhân:** Topic SNS đang bật mã hóa bằng AWS-managed key `alias/aws/sns`, vốn không cho phép dịch vụ CloudWatch Alarms gọi `kms:GenerateDataKey` trực tiếp.
- **Khắc phục:** Tắt mã hóa trên SNS topic (nội dung email chỉ gồm `userId`/`imageId`, không đính kèm ảnh hay link ảnh) → lịch sử Alarm chuyển sang *"Successfully executed action"*. Nếu bắt buộc phải mã hóa, cần dùng Customer Managed KMS Key (~1 USD/tháng) có key policy cấp quyền cho `cloudwatch.amazonaws.com`.

![Failed to execute action](/fcj-workshop/images/5-workshop/28-3a-bug-sns-kms-failed.jpg)

![Successfully executed action sau khi sửa](/fcj-workshop/images/5-workshop/28-3b-fix-sns-alarm-success.jpg)

### 4. Khác biệt giao diện AWS Console mới
- Trong Lambda Console mới, mục chọn role nằm ở **Custom settings → Additional settings → Custom execution role**.
- Trong CloudWatch Console mới, **Log groups** chuyển thành **Logs → Log Management**, còn **Logs Insights** đổi tên thành **Log Analytics**.
- Trang xem log stream mặc định chỉ hiển thị vài dòng cuối; nhấn **Load more** để xem đầy đủ dòng log `image_processed`.

### 5. Không thể sử dụng Amazon Bedrock do giới hạn Quota
Kế hoạch ban đầu dự kiến sử dụng **Amazon Nova Lite** trên Amazon Bedrock, nhưng toàn bộ tài khoản trong nhóm có quota bằng 0 và yêu cầu tăng hạn mức bị từ chối. Nhóm đã chuyển sang **Amazon Rekognition** — dịch vụ AI không bị chặn quota thử nghiệm, cung cấp sẵn API kiểm duyệt nội dung và dễ dự toán chi phí theo số ảnh.
