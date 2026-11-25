---
title: "Video demo"
date: 2026-11-20
weight: 8
chapter: false
pre: " <b> 5.8. </b> "
---

**Link Video Demo:** `[dán link YouTube / Google Drive sau khi quay]`

## Kịch bản trình bày (5–7 phút)

| Mốc thời gian | Nội dung demo |
|---|---|
| `0:00` | Giới thiệu bài toán thực tế và sơ đồ kiến trúc ImageGuard (mục 5.3) |
| `1:00` | Mở giao diện web qua CloudFront → đăng ký / đăng nhập bằng Cognito Managed Login |
| `2:00` | Tải ảnh chai bia → hệ thống xử lý sau vài giây, trả về **APPROVED** kèm các nhãn mô tả |
| `2:45` | Tải ảnh súng đồ chơi → trả về **REJECTED**, mở hộp thư kiểm tra email cảnh báo SNS (giải thích ví dụ false positive) |
| `3:30` | Kiểm tra bảng DynamoDB: cho thấy `userId` lưu theo `sub`, cùng `moderationLabels` và `labels` |
| `4:00` | Mở CloudWatch Dashboard: xem số lượng ảnh APPROVED/REJECTED, thời gian xử lý và trạng thái Alarm |
| `4:45` | Kiểm chứng bảo mật: gọi API không có token → `401`; mở trực tiếp S3 Object URL → `403 AccessDenied` |
| `5:30` | Trình bày bảng dự toán chi phí trên AWS Pricing Calculator (~8,37 USD/tháng) |
| `6:00` | Tổng kết kết quả và đề xuất hướng phát triển (trang admin duyệt lại ảnh REJECTED, tích hợp AWS WAF, dùng Presigned POST để giới hạn dung lượng upload) |
