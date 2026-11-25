---
title: "Workshop"
date: 2026-09-14
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# ImageGuard – Hệ thống kiểm duyệt và gắn nhãn ảnh tự động trên AWS Serverless

Workshop này hướng dẫn bạn xây dựng hoàn chỉnh ứng dụng **ImageGuard** trên kiến trúc **AWS Serverless**: người dùng đăng nhập qua giao diện web, tải ảnh lên; hệ thống sử dụng **Amazon Rekognition** để tự động kiểm duyệt nội dung (bạo lực, nội dung nhạy cảm, chất cấm, biểu tượng thù ghét…), gắn nhãn mô tả cho ảnh hợp lệ phục vụ tìm kiếm, đồng thời chặn ảnh vi phạm và gửi email cảnh báo đến quản trị viên.

Toàn bộ kiến trúc vận hành theo mô hình **hướng sự kiện (event-driven)** tại Region **us-east-1**, không cần quản lý máy chủ và có chi phí ước tính khoảng **8,37 USD/tháng** cho kịch bản 3.000 ảnh.

| Thông tin | Chi tiết |
|---|---|
| **Thời gian thực hiện** | 3 – 4 giờ |
| **Cấp độ** | 200 – Intermediate |
| **Dịch vụ AWS chính** | Amazon S3, AWS Lambda, Amazon Rekognition, Amazon DynamoDB, Amazon API Gateway, Amazon Cognito, Amazon CloudFront, Amazon SNS, Amazon SQS, Amazon CloudWatch, AWS IAM |
| **Mã nguồn tham khảo** | [static/code/imageguard](https://github.com/NhutTan-May/fcj-workshop-template/tree/main/static/code/imageguard) |

{{% children /%}}
