---
title: "Chi phí"
date: 2026-11-20
weight: 6
chapter: false
pre: " <b> 5.6. </b> "
---

Bảng dự toán được lập bằng **AWS Pricing Calculator**: [xem chi tiết bản ước tính](https://calculator.aws/#/estimate?id=5fa9c30d042e143fc2fd1f9387af56c6709e22f9).

**Giả định tải:** Region `us-east-1`, xử lý **3.000 ảnh/tháng** (trung bình 1 MB/ảnh, tỉ lệ vi phạm ~3%), 30.000 lượt xem/tìm kiếm mỗi tháng, tính theo giá on-demand **trước khi trừ Free Tier**.

| Dịch vụ AWS | Thông số ước tính | Chi phí (USD/tháng) |
|---|---|---|
| Amazon Rekognition | 3.000 `DetectModerationLabels` + 2.910 `DetectLabels` | 5,91 |
| AWS Lambda | 3 function `arm64`, 36.000 lượt gọi × 500 ms × 256 MB | 0,07 |
| Amazon API Gateway (HTTP API) | 33.000 request | 0,03 |
| Amazon DynamoDB (On-demand) | 6.000 lượt ghi, 30.000 lượt đọc, 0,01 GB lưu trữ | 0,01 |
| Amazon S3 | 3 GB lưu trữ, 3.000 PUT, 30.000 GET | 0,10 |
| Amazon CloudWatch | 5 custom metric EMF + 7 alarm + 0,1 GB log, 1 dashboard | 2,25 |
| CloudFront, Cognito, SNS, SQS | Nằm trong hạn mức miễn phí | 0,00 |
| **Tổng cộng** | | **8,37 USD/tháng** (≈ 100 USD/năm) |

![AWS Pricing Calculator](/fcj-workshop-template/images/5-workshop/29-1-pricing-calculator-summary.jpg)

**Chi phí thực tế khi chạy workshop:** với dưới 300 ảnh thử nghiệm, chi phí Rekognition chỉ khoảng **0,30 – 0,60 USD**, toàn bộ các dịch vụ còn lại đều nằm gọn trong Free Tier.

## Các biện pháp tối ưu chi phí đã áp dụng

- **100% Serverless:** loại bỏ hoàn toàn EC2, NAT Gateway hay tài nguyên chạy thường trực 24/7.
- **Kiểm duyệt trước, gắn nhãn sau:** ảnh bị từ chối ở bước kiểm duyệt sẽ bỏ qua lệnh gọi `DetectLabels`, giảm chi phí API AI.
- **Đọc trực tiếp từ S3:** Rekognition lấy ảnh thẳng từ S3 nên Lambda không cần tải dữ liệu về, giúp rút ngắn thời gian chạy và tiết kiệm RAM.
- **Kiến trúc `arm64` cho Lambda:** chi phí compute thấp hơn khoảng 20% so với `x86_64`.
- **HTTP API thay cho REST API:** tiết kiệm khoảng 70% chi phí API Gateway.
- **Vòng đời dữ liệu rõ ràng:** S3 Lifecycle xóa ảnh sau 30 ngày và CloudWatch Logs giữ log trong 14 ngày.
- **Tận dụng Free Tier cho CloudWatch:** phần giám sát chiếm 27% tổng chi phí ước tính, nhưng thực tế gần như bằng 0 khi nằm trong ngưỡng miễn phí (10 metric, 10 alarm, 3 dashboard).
