---
title: "Proposal"
date: 2026-11-06
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

# ImageGuard – Automated AI Image Moderation and Labeling on AWS Serverless

{{% notice info %}}
**Status:** Proposal v2.0.
{{% /notice %}}

## 1. Overview

**ImageGuard** is a serverless web application built for classified-ad sites, community forums, and student clubs to **automatically moderate and tag user-uploaded images**. Whenever a user submits an image, **Amazon Rekognition** performs two tasks:

1. **Content moderation:** screens for unsafe or inappropriate material (violence, explicit imagery, weapons, illicit substances, etc.). Violating uploads are marked `REJECTED`, and administrators receive an email notification.
2. **Automatic labeling:** assigns descriptive tags to approved images (such as *Laptop*, *Bicycle*, or *Book*) so users can **search their uploaded images by label**.

Analysis results are persisted in **Amazon DynamoDB** and reflected in the web UI within seconds. Built on an **event-driven serverless architecture**, ImageGuard eliminates server management overhead and incurs charges strictly per request and image processed.

## 2. Problem Statement

- User-generated platforms (marketplaces, forums, student communities) handle high daily upload volumes. **Manual review** is slow, labor-intensive, and forces human moderators to inspect harmful imagery.
- Without automated screening, policy-violating images may become publicly visible, harming users and damaging platform trust.
- Uploaded photos rarely include **descriptions or tags**, making subsequent search and discovery difficult.

**Proposed solution:** automate both content screening and descriptive tagging immediately upon upload using managed AI, blocking unsafe images before exposure while indexing clean ones for fast lookup.

## 3. Objectives & Scope

### 3.1. Objectives

1. Deliver a fully automated pipeline (**Image Upload → AI Moderation → AI Labeling → Storage → Display & Search**) running end-to-end on AWS.
2. Enforce the **AWS Well-Architected Framework**, emphasizing the **Security** pillar (IAM Least Privilege, zero public buckets, no hard-coded credentials).
3. Implement centralized observability with **Amazon CloudWatch** (Logs, Metrics, Alarms, Dashboard) and email alerts via **Amazon SNS**.
4. Maintain monthly operating expenses around **USD 8–9/month** under the target workload profile (Section 6).
5. Publish a **bilingual step-by-step workshop guide** enabling reproducible deployment from scratch.

### 3.2. Success Criteria

| Criterion | Measurable Target |
|---|---|
| End-to-end latency per image (upload to result) | ≤ 10 seconds |
| Violating images blocked accurately on test dataset | ≥ 90% |
| Valid images falsely blocked (false positive rate) | ≤ 10% |
| Unauthenticated API requests | Rejected (HTTP 401) |
| Cross-user access (User A viewing User B's images/results) | Impossible |
| Violating image detected | Alert email delivered to administrator |
| Direct Internet access to S3 buckets | Denied (Block Public Access) |

### 3.3. Scope

| In Scope | Out of Scope |
|---|---|
| JPG/PNG images, ≤ 5 MB | Video and animated GIFs |
| Content moderation + automated object/scene labeling | **Facial recognition** (biometric and privacy-sensitive data) |
| Label-based filtering within the signed-in user's own images | System-wide full-text search |
| Sign-up / sign-in via Amazon Cognito | Dedicated admin portal and moderation appeal workflows |
| Single Region: **us-east-1** | Multi-Region active-active / Disaster Recovery |

## 4. Solution Architecture

![ImageGuard Architecture Diagram](/fcj-workshop/images/2-proposal/imageguard-architecture.drawio.png)

*Figure 1. ImageGuard architecture (designed in draw.io using AWS Architecture Icons). Arrow numbers correspond to Section 4.2. Colors: blue – upload flow; orange – AI processing flow; green – result retrieval flow; dashed grey – monitoring; dashed red – IAM.*

### 4.1. AWS Services and Selection Rationale

| Service | System Role | Selection Rationale |
|---|---|---|
| **Amazon S3** | (1) Private image bucket; (2) private static web hosting bucket | 99.999999999% durability, low storage cost, emits **S3 Events** to trigger processing |
| **Amazon CloudFront** | Serves the static frontend over HTTPS via **Origin Access Control (OAC)** | Keeps web S3 bucket private; free HTTPS; covered by Always Free tier (1 TB/month) |
| **Amazon Cognito** (User Pool) | Handles user registration, login, and **JWT issuance** | Managed identity service, free up to 10,000 MAU, avoids storing passwords in app code |
| **Amazon API Gateway** (HTTP API) | Exposes REST endpoints protected by a Cognito **JWT Authorizer** | ~70% cheaper than REST API (~USD 1 per million requests) with native JWT validation |
| **AWS Lambda** (Python 3.12) | 3 functions: `presign-upload`, `process-image`, `get-images` | Serverless pay-per-use compute with automatic scaling |
| **Amazon Rekognition** | Runs `DetectModerationLabels` and `DetectLabels` | Pre-trained managed vision models; reads directly from S3; usable without custom quota requests |
| **Amazon DynamoDB** (On-demand) | Persists moderation status and extracted labels per user | Serverless NoSQL with single-digit millisecond latency and 25 GB free storage |
| **Amazon SQS** | **Dead-Letter Queue (DLQ)** capturing exhausted processing failures | Prevents event loss and allows inspection/reprocessing |
| **Amazon CloudWatch** | Stores logs, built-in & EMF custom metrics, alarms, and dashboard | Centralized observability aligned with workshop standards |
| **Amazon SNS** | Dispatches violation notifications and CloudWatch alarm emails | Lightweight pub/sub messaging with 1,000 free email notifications/month |
| **AWS IAM** | Dedicated least-privilege execution role per Lambda function | Enforces strict security boundaries across all components |

**Why us-east-1:** offers the lowest baseline pricing and immediate availability of all required managed services. Although round-trip latency from Vietnam is ~200 ms, CloudFront edge locations (Hanoi and Ho Chi Minh City) serve the static site rapidly while AI inference dominates backend time. For a production rollout targeting Vietnamese end users, switching to Singapore (`ap-southeast-1`) requires only updating environment variables.

### 4.2. Data Flow

Numbers align with Figure 1 above.

| # | Source → Destination | Description |
|---|---|---|
| 1 | User → CloudFront → S3 `imageguard-web` | User loads the web app over HTTPS; CloudFront fetches static assets privately via **OAC** |
| 2 | User ↔ Cognito | User authenticates via Managed Login (Authorization Code + PKCE) and obtains a **JWT token** |
| 3 | User → API Gateway | Frontend calls `POST /upload-url` with the JWT; the **JWT Authorizer** validates the token |
| 3a | API Gateway → Lambda `presign-upload` | Invokes the upload-URL generator Lambda |
| 3b | `presign-upload` → DynamoDB | Inserts a `PENDING` record and returns a short-lived **Presigned URL** (5-minute TTL) for `uploads/{userId}/{imageId}.jpg` |
| 4 | User → S3 `imageguard-images` | Browser uploads the file **directly to S3** via Presigned URL, bypassing API Gateway and Lambda |
| 5 | S3 → Lambda `process-image` | **ObjectCreated** event (scoped to `uploads/` prefix) triggers `process-image` asynchronously |
| 6 | `process-image` → Rekognition | Invokes `DetectModerationLabels`; only clean images proceed to `DetectLabels` (up to 10 labels, confidence ≥ 80%) |
| 6a | Rekognition → S3 `imageguard-images` | Rekognition fetches image bytes directly from S3 |
| 7 | `process-image` → DynamoDB | Conditionally updates the record to `APPROVED` (with labels) or `REJECTED` (if blocked category confidence ≥ 60%) |
| 8 | `process-image` → SNS → Admin | Only when marked `REJECTED` **and** step 7 succeeded: sends an alert email to the administrator |
| 9 | `process-image` → SQS DLQ | After 2 failed Lambda retries, the failing event lands in the **DLQ** |
| 10 | User → API Gateway | Frontend requests `GET /images?label=...` with the JWT token |
| 10a | API Gateway → Lambda `get-images` | Invokes the query Lambda |
| 10b | `get-images` → DynamoDB | Queries **strictly by `userId` extracted from the JWT `sub` claim** and generates short-lived presigned GET URLs for `APPROVED` images |
| 11 | 3 Lambdas → CloudWatch | Emits structured logs and Embedded Metric Format (EMF) custom metrics |
| 12 | CloudWatch → SNS | Triggers email alerts on Lambda errors, non-empty DLQ, or API 5xx spikes |
| — | IAM → 3 Lambdas | Attaches a separate least-privilege **IAM execution role** to each Lambda (Section 4.3) |

### 4.3. Security Design (Security Pillar)

| Principle | ImageGuard Implementation |
|---|---|
| **Least Privilege** | Separate IAM roles per function. `process-image` gets `s3:GetObject` (`uploads/*`), 2 Rekognition actions, `dynamodb:UpdateItem` on **1 table**, `sns:Publish` on **1 topic**, and `sqs:SendMessage` on **1 DLQ**. `presign-upload` gets `s3:PutObject` + `dynamodb:PutItem`. `get-images` gets `dynamodb:Query` + `s3:GetObject` |
| **Rekognition Scope Constraint** | Because Rekognition image-analysis APIs do not support resource-level ARNs (`"Resource": "*"` required), permissions are tightly restricted to **only 2 specific actions** rather than `rekognition:*` |
| **Zero Public Storage** | Both buckets enable **Block Public Access**. Web assets flow solely through CloudFront OAC; user images flow solely through time-limited presigned URLs |
| **Auth & Tenant Isolation** | Every API call requires a valid Cognito JWT. `userId` is always derived server-side from the `sub` claim—never trusted from client request bodies |
| **No Hard-coded Secrets** | Lambdas authenticate via IAM Roles; bucket/table/topic names live in **environment variables**; `.gitignore` prevents credential leaks |
| **Encryption** | S3 and DynamoDB use server-side encryption at rest; all endpoints enforce HTTPS/TLS |
| **Input Validation** | Presigned URLs enforce `Content-Type`; Lambda checks file size (≤ 5 MB) and regex-validates S3 keys |
| **Privacy** | Facial recognition is excluded; no biometric identifiers are processed or stored |

> **Why keep Lambdas outside a VPC?** None of the dependencies (S3, DynamoDB, Rekognition, SNS) require private subnet placement—all are accessed over HTTPS with IAM authentication. Placing Lambdas in a VPC would require a NAT Gateway or VPC Interface Endpoints (adding tens of USD/month) with negligible security benefit here.

### 4.4. Alignment with AWS Well-Architected Framework

| Pillar | Implementation |
|---|---|
| Operational Excellence | 14-day CloudWatch Logs retention, CloudWatch Dashboard & Alarms, reproducible bilingual workshop docs |
| Security | Detailed in Section 4.3 |
| Reliability | Event-driven decoupling, 2 automatic Lambda async retries, SQS DLQ, explicit `PENDING/APPROVED/REJECTED` states |
| Performance Efficiency | Direct browser-to-S3 uploads via presigned URLs, zero-download Rekognition S3 reads, key-based DynamoDB queries |
| Cost Optimization | 100% pay-per-use serverless, **conditional labeling only for clean images**, 30-day S3 Lifecycle, 14-day log retention, AWS Budgets |
| Sustainability | Zero idle compute capacity; resources execute solely when images arrive |

### 4.5. Rationale for Choosing Amazon Rekognition

The original concept relied on **Amazon Bedrock** for document/invoice analysis, but initial account testing hit a zero quota and the quota increase request was denied. Switching to **Amazon Rekognition** preserved the event-driven serverless design while providing reliable out-of-the-box moderation and labeling without quota blockers (documented further in Troubleshooting).

## 5. Plan & Timeline

Internship period: 14/09/2026 – 04/12/2026. Target workshop submission: **19/11/2026**.

### 5.1. Team Roles

| Member | Primary Responsibilities |
|---|---|
| Member A | Team lead, system integration, IAM & security architecture |
| Member B | `process-image` Lambda, Rekognition integration, test dataset & accuracy evaluation |
| Member C | Cognito, API Gateway, `presign-upload` and `get-images` Lambdas |
| Member D | Static web UI, S3 + CloudFront (OAC) |
| Member E | CloudWatch, SNS, DLQ, cost estimation, clean-up guide |

*Every member is expected to understand and explain the end-to-end architecture, not just their assigned component.*

### 5.2. Weekly Timeline

| Week | Dates | Activities | Deliverables |
|---|---|---|---|
| Week 7 | 26/10 – 01/11 | Finalize topic, author Proposal v1.0; test Bedrock and discover zero quota | Proposal v1.0, incident log |
| Week 8 | 02/11 – 08/11 | Quota request rejected; pivot to Rekognition, author Proposal v2.0; build S3, DynamoDB, IAM roles, `process-image` + S3 Event | Proposal v2.0, working processing pipeline |
| Week 9 | 09/11 – 15/11 | Cognito, API Gateway, API Lambdas; static frontend + CloudFront; CloudWatch metrics/alarms/dashboard, SNS, DLQ; testing; Pricing Calculator; demo recording | End-to-end demo, video, cost estimate |
| Week 10 | 16/11 – 19/11 | Author bilingual `5-Workshop` docs (lab guide, testing, clean-up), capture screenshots, polish diagrams, peer review | **Workshop submission 19/11** |
| Week 10–12 | 20/11 – 04/12 | Address mentor feedback, finalize blogs and worklogs, clean up resources | Final internship report |

## 6. Cost Estimation

**Workload assumptions:** a small forum/classifieds platform processing **3,000 images/month** (~100/day, ~1 MB average, ~3% violation rate) with 30,000 monthly gallery views/searches in **us-east-1** (on-demand pricing **before Free Tier/credits**).

| Service | Usage Assumptions | Est. Monthly Cost (USD) |
|---|---|---|
| Amazon Rekognition | 3,000 `DetectModerationLabels` + ~2,910 `DetectLabels` × USD 0.001/image | 5.91 |
| AWS Lambda | 3 arm64 functions, 36,000 invocations × 500 ms × 256 MB | 0.07 (covered by Free Tier) |
| Amazon API Gateway (HTTP API) | 33,000 requests | 0.03 |
| Amazon DynamoDB (On-demand) | ~6,000 WRU + 30,000 RRU, < 1 GB | 0.01 |
| Amazon S3 | 3 GB (30-day lifecycle), 3,000 PUTs, 30,000 GETs | 0.10 |
| Amazon CloudWatch | 5 EMF custom metrics (1.50) + 7 alarms (0.70) + 0.1 GB logs (0.05); 1 dashboard | 2.25 (covered by Free Tier: 10 metrics, 10 alarms, 3 dashboards) |
| Amazon CloudFront | ~2 GB data transfer, 40,000 requests | 0.00 (Always Free) |
| Amazon Cognito | < 10,000 MAU | 0.00 |
| Amazon SNS, Amazon SQS | ~100 alerts/messages | 0.00 |
| **Total** | | **≈ USD 8.37/month (≈ USD 100/year)** |

During development and workshop demos (< 300 images), Rekognition spend is roughly USD 0.60 and fits well within account credits.

*AWS Pricing Calculator:* [https://calculator.aws/#/estimate?id=5fa9c30d042e143fc2fd1f9387af56c6709e22f9](https://calculator.aws/#/estimate?id=5fa9c30d042e143fc2fd1f9387af56c6709e22f9)

> **Post-deployment update (20/11/2026):** the initial estimate was USD 6.40/month. Adding 7 CloudWatch Alarms and 5 EMF custom metrics during implementation raised CloudWatch from USD 0.27 to USD 2.25 (effectively USD 0 within Free Tier limits). Image egress (~3 GB/month) fits inside the 100 GB/month Free Tier data transfer allowance.

### Cost Optimization Strategy

- **100% serverless:** zero EC2 instances, NAT Gateways, or idle 24/7 compute.
- **Screen first, label second:** rejected images skip `DetectLabels`, saving an AI call per blocked upload.
- **Direct S3 reads by Rekognition:** keeps Lambda execution brief (~2 s) at 256 MB memory.
- **S3 Lifecycle Policy:** expires uploaded images after 30 days (or transitions to Glacier if archiving is needed).
- **14-day CloudWatch Logs retention** instead of indefinite storage.
- **HTTP API** instead of REST API (~70% lower API Gateway cost).
- **AWS Budgets** alert at USD 10 plus full **Clean-up** post-evaluation.
- Tiered Rekognition pricing reduces per-image cost automatically beyond 1M images/month.

## 7. Risks & Mitigation

| Risk | Impact | Mitigation |
|---|---|---|
| False positives or missed violations by Rekognition | Poor UX or unsafe content slipping through | Externalize confidence threshold and blocked categories in env vars for tuning; measure accuracy on test set |
| New-account service quota restrictions (experienced with Bedrock) | AI pipeline blocked | Confirmed Rekognition availability on account; configured botocore exponential backoff retries |
| S3 → Lambda → S3 recursive trigger loop | Unexpected cost spike | Scope S3 trigger strictly to `uploads/` prefix; grant `process-image` read-only S3 access |
| Accidental credential exposure on GitHub | Account compromise | Rely exclusively on IAM Roles; enforce `.gitignore`; audit commits before pushing |
| Copyrighted or overly graphic test images | Legal/ethical concerns | Use team-owned or openly licensed images and mild moderation categories (alcohol, toy weapons) |
| Unplanned cloud charges | Credit depletion | AWS Budgets alert + ordered clean-up checklist |
| Tight schedule (~2 weeks remaining) | Missed deadline | Prioritize MVP before enhancements; enforce weekly deliverables |
| Varying AWS familiarity across members | Slow progress or Q&A gaps | Learn by building, run peer reviews, ensure every member can explain the full architecture |

## 8. Expected Outcomes

- A live end-to-end serverless web application: login → image upload → violating images blocked with admin email alert, valid images auto-labeled and searchable.
- Complete observability stack with CloudWatch Dashboard, Alarms, and SNS email notifications.
- A reproducible bilingual workshop guide covering setup, testing, troubleshooting, cost analysis, and resource clean-up.
- Practical team mastery of event-driven serverless design, IAM Least Privilege, managed AI integration, observability, cost control, and architectural pivoting under quota constraints.
