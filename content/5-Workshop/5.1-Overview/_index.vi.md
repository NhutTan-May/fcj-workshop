---
title: "Tổng quan Workshop"
date: 2026-11-20
weight: 1
chapter: false
pre: " <b> 5.1. </b> "
---

## Bối cảnh bài toán

Các trang rao vặt, diễn đàn hoặc cộng đồng sinh viên thường tiếp nhận hàng nghìn lượt đăng ảnh mỗi tháng. Việc kiểm duyệt thủ công vừa chậm chạp, tốn nhân lực, vừa khiến người kiểm duyệt phải tiếp xúc trực tiếp với hình ảnh tiêu cực. **ImageGuard** giải quyết bài toán này bằng quy trình tự động hoàn toàn: mỗi ảnh sau khi tải lên được phân tích và trả kết quả trong vòng **1–2 giây**.

## Kiến thức & thành quả đạt được

Sau khi hoàn thành bài thực hành, bạn sẽ xây dựng được một hệ thống hoàn chỉnh với các thành phần:

- **Frontend bảo mật:** Website tĩnh phân phối qua **Amazon CloudFront** (HTTPS, OAC) và tích hợp đăng nhập **Amazon Cognito** theo chuẩn OAuth 2.0 Authorization Code + PKCE.
- **Upload trực tiếp lên S3:** Trình duyệt tải ảnh thẳng lên bucket private qua **Presigned URL** (thời hạn 5 phút), giúp dữ liệu ảnh không phải đi qua API Gateway hay Lambda.
- **Xử lý AI hướng sự kiện:** **S3 Event** kích hoạt bất đồng bộ hàm Lambda `process-image` để chạy `DetectModerationLabels`; chỉ những ảnh hợp lệ mới gọi tiếp `DetectLabels`, sau đó cập nhật kết quả vào **Amazon DynamoDB**.
- **Cảnh báo & chống mất dữ liệu:** Ảnh vi phạm được gắn trạng thái `REJECTED` kèm email cảnh báo qua **Amazon SNS**. Các sự kiện lỗi sau 2 lần retry sẽ được chuyển vào **SQS Dead-Letter Queue (DLQ)**.
- **Giám sát tập trung:** Theo dõi toàn diện qua **Amazon CloudWatch** với Log Group (retention 14 ngày), 5 custom metric theo chuẩn EMF, 7 Alarm và 1 Dashboard trực quan.

## Trọng tâm kỹ thuật

| Lĩnh vực | Nội dung thực hành |
|---|---|
| Kiến trúc hướng sự kiện | Kết nối S3 Event → Lambda bất đồng bộ, cấu hình retry và Dead-Letter Queue |
| Bảo mật nhiều lớp | Áp dụng IAM Least Privilege cho từng Lambda, Presigned URL, JWT Authorizer, CloudFront OAC và chống confused deputy |
| AI không cần tự huấn luyện | Tích hợp Amazon Rekognition Content Moderation và Label Detection qua SDK |
| Vận hành & Giám sát | Ghi custom metric bằng Embedded Metric Format (EMF), cấu hình CloudWatch Alarm → SNS và Dashboard |
| Tối ưu chi phí | Sử dụng AWS Pricing Calculator, S3 Lifecycle, giới hạn log retention và HTTP API |

## Lộ trình thực hiện

1. [Chuẩn bị môi trường](../5.2-prerequisites/)
2. [Kiến trúc & Thiết kế bảo mật](../5.3-architecture/)
3. [Hướng dẫn triển khai từng bước](../5.4-lab-guide/)
4. [Kiểm thử & Xử lý sự cố](../5.5-testing/)
5. [Ước tính & Tối ưu chi phí](../5.6-cost/)
6. [Dọn dẹp tài nguyên](../5.7-cleanup/)
7. [Video Demo](../5.8-demo/)
