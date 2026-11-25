---
title: "Các bài blogs đã đăng"
date: 2024-01-01
weight: 3
chapter: false
pre: " <b> 3. </b> "
---

Trong quá trình thực tập, em đã tìm hiểu và tổng hợp 3 bài viết chia sẻ kiến thức về các dịch vụ và tính năng mới của AWS:

### [Blog 1 - Amazon ECS Gateway: Giải pháp Ingress mạng quản lý toàn diện cho Container](3.1-Blog1/)
Giới thiệu **Amazon ECS Gateway**, tài nguyên cấp tài khoản giúp tự động hóa hoàn toàn việc cấp phát và quản lý hạ tầng mạng đầu vào (ALB, chứng chỉ ACM, DNS hostname, Security Group) cho các ứng dụng container trên ECS, hỗ trợ chia sẻ ALB lên tới 25 services và tách bạch trách nhiệm rõ ràng giữa Platform team và Application team.

### [Blog 2 - AWS Lambda Web Functions: Chạy trực tiếp ứng dụng web HTTP không máy chủ](3.2-Blog2/)
Tổng hợp tính năng **AWS Lambda Web Functions**, cho phép chạy trực tiếp các ứng dụng HTTP và web framework phổ biến (Express, Hono, Fastify, Next.js…) trên AWS Lambda mà không cần adapter, hỗ trợ xử lý đồng thời nhiều request, native HTTP streaming và cơ chế tính phí Active Compute tối ưu chi phí I/O.

### [Blog 3 - AWS Account Access Manager: Quản lý truy cập tập trung kết hợp linh hoạt IAM Role](3.3-Blog3/)
Giới thiệu **AWS Account Access Manager (AAM)**, giúp kết hợp quản trị truy cập tập trung qua AWS IAM Identity Center với khả năng tùy biến chi tiết của từng IAM Role trong từng tài khoản AWS, đồng thời đảm bảo truy vết đích danh người dùng trên CloudTrail thông qua *trusted identity propagation*.
