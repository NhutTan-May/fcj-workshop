---
title: "Workshop Overview"
date: 2026-11-20
weight: 1
chapter: false
pre: " <b> 5.1. </b> "
---

## The Problem

Classified-ad platforms, forums, and student communities handle thousands of user-uploaded images every month. Manual screening is slow, costly, and exposes reviewers to disturbing content, while untagged images remain hard to search. **ImageGuard** automates both moderation and labeling: every uploaded photo is evaluated within **1–2 seconds**.

## What You Will Build

By completing this workshop, you will deploy a working end-to-end serverless system featuring:

- **Protected Web Frontend:** A static web app served over HTTPS via **Amazon CloudFront** (with OAC) and authenticated through **Amazon Cognito** (OAuth 2.0 Authorization Code + PKCE).
- **Direct-to-S3 Uploads:** Browsers upload images straight into a private S3 bucket using short-lived **Presigned URLs**, avoiding API Gateway and Lambda payload overhead.
- **Event-Driven AI Pipeline:** **S3 Events** asynchronously invoke the `process-image` Lambda to run `DetectModerationLabels`; only clean images proceed to `DetectLabels` before results are stored in **Amazon DynamoDB**.
- **Alerting & Fault Tolerance:** Violating images transition to `REJECTED` and trigger an **Amazon SNS** email to the admin, while failed invocations after 2 retries land safely in an **SQS Dead-Letter Queue (DLQ)**.
- **Full Observability:** Centralized logs (14-day retention), 5 custom EMF metrics, 7 CloudWatch Alarms, and 1 operational CloudWatch Dashboard.

## Key Learning Areas

| Area | Topics Covered |
|---|---|
| Event-Driven Architecture | S3 Event → asynchronous Lambda invocation, retry policy, and Dead-Letter Queue |
| Defense-in-Depth Security | Per-Lambda Least Privilege IAM roles, Presigned URLs, JWT Authorizer, CloudFront OAC, and confused-deputy prevention |
| Pre-trained Managed AI | Amazon Rekognition Content Moderation and Label Detection without model training |
| Cloud Operations | Embedded Metric Format (EMF) metrics, CloudWatch Alarms → SNS notifications, and Dashboards |
| Cost Engineering | AWS Pricing Calculator, S3 Lifecycle rules, log retention policies, and API Gateway HTTP API |

## Workshop Sections

1. [Prerequisites](../5.2-prerequisites/)
2. [Architecture & Security Design](../5.3-architecture/)
3. [Step-by-Step Lab Guide](../5.4-lab-guide/)
4. [Testing & Troubleshooting](../5.5-testing/)
5. [Cost Breakdown](../5.6-cost/)
6. [Resource Clean-up](../5.7-cleanup/)
7. [Demo Video](../5.8-demo/)
