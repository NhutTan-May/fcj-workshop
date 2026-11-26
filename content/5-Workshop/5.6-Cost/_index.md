---
title: "Cost"
date: 2026-11-20
weight: 6
chapter: false
pre: " <b> 5.6. </b> "
---

Cost modeling was built using the **AWS Pricing Calculator**: [view estimate](https://calculator.aws/#/estimate?id=5fa9c30d042e143fc2fd1f9387af56c6709e22f9).

**Workload assumptions:** `us-east-1` Region, **3,000 images/month** (~1 MB average, ~3% violation rate), 30,000 monthly gallery views/searches, evaluated at on-demand pricing **before Free Tier deductions**.

| AWS Service | Usage Profile | Est. Cost (USD/month) |
|---|---|---|
| Amazon Rekognition | 3,000 `DetectModerationLabels` + 2,910 `DetectLabels` calls | 5.91 |
| AWS Lambda | 3 `arm64` functions, 36,000 invocations × 500 ms × 256 MB | 0.07 |
| Amazon API Gateway (HTTP API) | 33,000 requests | 0.03 |
| Amazon DynamoDB (On-demand) | 6,000 writes, 30,000 reads, 0.01 GB storage | 0.01 |
| Amazon S3 | 3 GB storage, 3,000 PUTs, 30,000 GETs | 0.10 |
| Amazon CloudWatch | 5 EMF metrics + 7 alarms + 0.1 GB logs, 1 dashboard | 2.25 |
| CloudFront, Cognito, SNS, SQS | Within Free Tier allowances | 0.00 |
| **Total** | | **USD 8.37/month** (≈ USD 100/year) |

![AWS Pricing Calculator](/fcj-workshop/images/5-workshop/29-1-pricing-calculator-summary.jpg)

**Actual workshop spend:** under 300 test images incurred only **USD 0.30–0.60** in Rekognition charges, while all remaining services stayed within the AWS Free Tier.

## Applied Cost Optimization Techniques

- **100% Serverless:** avoids fixed EC2, NAT Gateway, or idle 24/7 infrastructure costs.
- **Screen first, tag second:** blocked images skip `DetectLabels`, preventing unnecessary AI API calls.
- **Direct S3 reads by Rekognition:** eliminates downloading image bytes inside Lambda, reducing memory and execution duration.
- **`arm64` Lambda runtime:** lowers compute cost by ~20% compared to `x86_64`.
- **API Gateway HTTP API:** ~70% cheaper than REST API for JWT-authenticated endpoints.
- **Bounded retention:** S3 Lifecycle expires uploads after 30 days, and CloudWatch Logs retain entries for 14 days.
- **Free Tier coverage for CloudWatch:** monitoring accounts for 27% of the pre-free-tier estimate, yet drops near zero within the 10-metric / 10-alarm / 3-dashboard Free Tier quota.
