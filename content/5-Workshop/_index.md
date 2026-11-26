---
title: "Workshop"
date: 2026-09-14
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# ImageGuard – Automated AI Image Moderation and Labeling on AWS Serverless

This hands-on workshop walks you through building **ImageGuard** from the ground up on **AWS Serverless**: authenticated users upload images from a web interface, **Amazon Rekognition** screens out inappropriate content (violence, explicit imagery, illicit substances, hate symbols, etc.), clean images receive automatic descriptive tags for search, and violating uploads are blocked with an email alert to the administrator.

The entire platform follows an **event-driven serverless architecture** deployed in **us-east-1**, requiring zero server maintenance and costing approximately **USD 8.37/month** for a 3,000-image monthly workload.

| Detail | Value |
|---|---|
| **Estimated duration** | 3 – 4 hours |
| **Level** | 200 – Intermediate |
| **Core AWS services** | Amazon S3, AWS Lambda, Amazon Rekognition, Amazon DynamoDB, Amazon API Gateway, Amazon Cognito, Amazon CloudFront, Amazon SNS, Amazon SQS, Amazon CloudWatch, AWS IAM |
| **Source code** | [static/code/imageguard](https://github.com/NhutTan-May/fcj-workshop/tree/main/static/code/imageguard) |

{{% children /%}}
