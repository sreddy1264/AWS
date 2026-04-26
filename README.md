# AWS

A collection of AWS architecture concepts, system design references, service trade-off guides, and deployable CloudFormation templates.

---

## Templates

### [Event-Driven Order Processing — CloudFormation Template](templates/event-driven-order-processing.yaml)

A production-grade event-driven e-commerce order processing system using EventBridge, SNS, SQS, Lambda, and DynamoDB. Built for real production workloads: idempotency, DLQs, partial batch failure, least-privilege IAM, CloudWatch alarms.

**Architecture:**

```
API → OrderProducerLambda → EventBridge (custom bus: com.ecommerce)
                                │
              ┌─────────────────┼──────────────────┐
              ▼                 ▼                   ▼
      order.created      order.cancelled     payment.processed
      order.cancelled                        payment.failed
              │                 │                   │
              ▼                 ▼                   ▼
      SNS: OrderEventsTopic            SNS: PaymentEventsTopic
              │                                     │
    ┌─────────┼───────────┐              ┌──────────┴──────────┐
    ▼         ▼           ▼              ▼                     ▼
PaymentQ  FulfillQ  NotifQ+AnalyticsQ  FulfillQ           NotifQ+AnalyticsQ
    │         │           │              │                     │
    ▼         ▼           ▼              ▼                     ▼
 Lambda    Lambda      Lambda         Lambda               Lambda
  +DLQ      +DLQ        +DLQ           (same)               (same)
```

**Services and justification:**

| Service | Role | Why chosen over alternative |
|---------|------|----------------------------|
| **EventBridge** | Content-based routing | Filters on any payload field (detail-type, source). SNS alone can't route on payload content. 200+ AWS service integrations. |
| **SNS** | Fan-out | One EB rule → one SNS topic → N SQS queues atomically. Adding a consumer = new SNS subscription, no EB rule change. |
| **SQS Standard** | Durable buffering | Survives Lambda downtime. At-least-once + code idempotency = effectively exactly-once. FIFO rejected: adds 3K msg/s limit, no real benefit when Lambda handles idempotency. |
| **DynamoDB on-demand** | State + idempotency | Conditional PutItem (`attribute_not_exists`) is atomic. TTL auto-expires idempotency records at 24h. $0 at idle. |
| **Lambda** | Event processing | Scales to 0 when idle. Each queue consumer independently throttleable via `ReservedConcurrentExecutions`. |

**What's included:**

| Category | Resources |
|----------|-----------|
| Event routing | EventBridge custom bus + 3 rules (order.created, order.cancelled, payment events) |
| Fan-out | 2 SNS topics + 7 subscriptions |
| Durable queues | 4 SQS queues + 4 DLQs + 4 queue policies |
| Consumers | 5 Lambda functions, each with its own IAM role |
| Idempotency | DynamoDB conditional write on SQS MessageId per function |
| State store | DynamoDB single-table with 2 GSIs + PITR + Streams |
| Failure handling | Partial batch failure (`ReportBatchItemFailures` ESM) + DLQ + CloudWatch alarms |
| Observability | 8 CloudWatch alarms (DLQ depth, Lambda errors, throttles, queue backlog) |
| Security | One IAM role per Lambda, no wildcard resources, SQS/SNS SSE |

**Deploy:**

```bash
aws cloudformation deploy \
  --template-file templates/event-driven-order-processing.yaml \
  --stack-name ecommerce-order-processing-dev \
  --parameter-overrides AppName=ecommerce Environment=dev AlertEmail=you@example.com \
  --capabilities CAPABILITY_NAMED_IAM
```

**Parameters:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `AppName` | `ecommerce` | Prefix for all resource names |
| `Environment` | `dev` | `dev` / `staging` / `prod` — affects log retention, message retention, alarm thresholds |
| `AlertEmail` | `ops@example.com` | Email for CloudWatch alarm SNS notifications |
| `MaxReceiveCount` | `3` | SQS retry attempts before DLQ (prod: set to 5) |
| `PaymentConcurrencyLimit` | `5` | Reserved concurrency on payment Lambda — caps gateway call rate |
| `SQSVisibilityTimeout` | `300` | Must be ≥ 6× Lambda timeout (30s × 6 = 180s minimum) |

---

### [Simple App — CloudFormation Template](templates/simple-app.yaml)

A production-ready, serverless CloudFormation template for a simple web application. Every service choice is justified against AWS Well-Architected Framework principles.

**Architecture:**

```
Browser → CloudFront (CDN + HTTPS + DDoS)
            ├── /          → S3 (static frontend / React SPA)
            └── /api/*     → API Gateway HTTP API → Lambda → DynamoDB
                                    ↑
                              Cognito JWT auth (API GW validates — no extra Lambda call)
```

**Services and justification:**

| Service | Why chosen | Idle cost |
|---------|-----------|-----------|
| **Lambda** | Bursty/unpredictable traffic — pay per invocation, zero idle cost vs EC2 ~$15/month minimum | $0 |
| **API Gateway HTTP API** | $1/million requests (3.5× cheaper than REST API), 6ms overhead vs 30ms, native JWT auth | $0 |
| **DynamoDB on-demand** | Simple key-value access patterns, no JOINs needed, scales to zero, PITR free | $0 |
| **S3 + CloudFront** | ~$0/month for a 10 MB SPA bundle; global CDN + free HTTPS + Shield Standard DDoS | ~$0 |
| **Cognito User Pool** | Free up to 50K MAU; JWT integrates natively with API Gateway; no auth code to maintain | $0 |
| **CloudWatch** | Native metrics/logs from all services; 5 GB/month free tier; 30-day retention auto-purge | $0 |

**Estimated cost at scale:**

| Traffic | Estimated monthly cost |
|---------|----------------------|
| Idle / dev | ~$0 |
| 100K API calls | ~$0.10 |
| 1M API calls | ~$5–10 |
| 10M API calls | ~$40–60 |

**Deploy:**

```bash
# Create stack
aws cloudformation deploy \
  --template-file templates/simple-app.yaml \
  --stack-name my-app-dev \
  --parameter-overrides AppName=my-app Environment=dev \
  --capabilities CAPABILITY_NAMED_IAM

# Deploy frontend (after building your React/Vue/etc app)
aws s3 sync ./dist s3://$(aws cloudformation describe-stacks \
  --stack-name my-app-dev \
  --query 'Stacks[0].Outputs[?OutputKey==`FrontendBucket`].OutputValue' \
  --output text) --delete

# Invalidate CloudFront cache
aws cloudfront create-invalidation \
  --distribution-id $(aws cloudformation describe-stacks \
    --stack-name my-app-dev \
    --query 'Stacks[0].Outputs[?OutputKey==`AppUrl`].OutputValue' \
    --output text | cut -d/ -f3) \
  --paths "/*"
```

**Parameters:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `AppName` | `my-app` | Prefix for all resource names |
| `Environment` | `dev` | `dev` or `prod` — affects throttle limits, log retention, CloudFront price class |
| `LambdaMemoryMB` | `256` | 128 / 256 / 512 / 1024 MB |

**What's included in the template:**
- IAM role with least-privilege DynamoDB access (no wildcards)
- DynamoDB single-table design with GSI, PITR, and encryption at rest
- Lambda with inline placeholder handler (replace with your code)
- API Gateway HTTP API with Cognito JWT authorizer
- Public `/health` route + protected `/me`, `/items` routes
- S3 private bucket with Origin Access Control (OAC) for CloudFront
- CloudFront with two origins (S3 for frontend, API GW for `/api/*`)
- SPA routing: 403/404 → `index.html` so React Router handles URLs
- CloudWatch alarms: Lambda error rate and API p99 latency

---

## Concepts

### 1. [AWS Services Trade-off Decision Guide](conepts/AwsServicesTradeoff.md)

A comprehensive decision guide covering the most common AWS service choices with clear **when to pick what** guidance.

| Topic | Services Compared |
|-------|------------------|
| Compute | Lambda vs EC2 vs Fargate |
| Containers | ECS vs EKS vs Fargate |
| Database | DynamoDB vs RDS vs Aurora |
| Messaging | SQS vs SNS vs EventBridge |
| API Layer | API Gateway vs ALB vs AppSync |
| Load Balancers | ALB vs NLB vs CLB |
| Storage | S3 vs EBS vs EFS |
| Caching | ElastiCache Redis vs Memcached vs DAX |
| Streaming | Kinesis vs SQS vs MSK |
| CDN & Edge | CloudFront vs Without CDN |
| Orchestration | Step Functions vs SQS + Lambda |
| Auth | Cognito vs Custom Auth vs IAM |
| Observability | CloudWatch vs X-Ray vs OpenTelemetry |
| DNS Routing | Route 53 Routing Policies |
| High Availability | Multi-AZ vs Multi-Region |
| Search | OpenSearch vs RDS Full-Text vs Kendra |
| Secrets | Secrets Manager vs Parameter Store vs Env Vars |
| Deployment | CodeDeploy vs ECS Rolling vs Blue/Green |

Includes a **Quick Decision Cheat Sheet** covering all categories in one place.

---

### 2. [AWS Website Hosting — Architecture Flowchart](conepts/aws-architecture-flowchart.md)

A Mermaid architecture diagram of the full AWS website hosting stack, tracing a request from user to data layer.

**Flow:** User → Route 53 → CloudFront → WAF + Shield → ALB → Compute (EC2/ECS/Lambda) → Data (RDS, DynamoDB, S3, ElastiCache) → Observability (CloudWatch, X-Ray, IAM)

Includes:
- Step-by-step request journey table with each service's role
- Common architecture patterns (Static Site, API Backend, Full-stack, Hybrid)
- Mermaid diagram with styled subgraphs for Compute, Data, and Observability layers

---

### 3. [Large-Scale Application Hosting on AWS — System Design Reference](conepts/large-scale-system-design-aws.md)

A full system design reference for architecting a globally distributed video streaming platform at scale — **250M+ users, 20M concurrent streams, 5M requests/second**.

**Covers:**

| Section | Details |
|---------|---------|
| Requirements & Scale Targets | Functional/non-functional requirements, scale numbers |
| Full Architecture Diagram | End-to-end Mermaid diagram across all layers |
| Edge & DNS Layer | Route 53 config, CloudFront cache strategy, custom ISP CDN rationale |
| Security Layer | WAF rules, Shield Advanced, GuardDuty auto-remediation |
| API Gateway Layer | HTTP API vs REST API, AppSync for GraphQL mobile clients |
| Microservices Compute | ECS Fargate vs EC2, auto-scaling config per service |
| Async Messaging | MSK (Kafka) vs Kinesis, SQS FIFO for exactly-once, EventBridge for scheduling |
| Data Layer | DynamoDB for user state, Aurora for billing, Redis, OpenSearch, S3 data lake |
| Video Processing | Spot GPU encoding pipeline, S3 storage tiering strategy |
| ML Platform | SageMaker recommendation system, pre-computed + real-time hybrid |
| Observability | Metrics, logs, traces — CloudWatch, X-Ray, Kinesis Firehose |
| Capacity Estimation | Storage (600 PB video), compute (2,000 Fargate tasks at peak), egress |
| Service Trade-off Decisions | Decision matrix for every major service choice |
| Resilience & DR | Failure mode analysis, circuit breaker pattern |
| Cost Architecture | Monthly cost breakdown ($3.5M–$7M), 6 cost optimization levers |
| Scaling Playbook | Pre-scaling for traffic spikes, zero-downtime deployment pipeline |

---

### 4. [Netflix on AWS — Real-World Architecture Deep Dive](conepts/netflix-aws.md)

A deep dive into Netflix's actual AWS architecture — how they serve **250M subscribers**, stream **700,000+ hours per minute**, and run **1,000+ microservices** across 190+ countries.

**Covers:**

| Section | Details |
|---------|---------|
| Scale Numbers | EC2 instances, CDN nodes, events/day, API RPS |
| High-Level Architecture Diagram | Full Mermaid diagram across Edge, Gateway, Compute, Data, Encoding, Observability, Resilience |
| Request Flow — Press Play | Sequence diagram: DNS → CloudFront → WAF → Zuul → Streaming Service → EVCache → DynamoDB → Open Connect CDN |
| Service-by-Service Breakdown | Route 53, CloudFront, Open Connect (custom CDN), Zuul API Gateway, EC2, Kafka, DynamoDB, Cassandra, EVCache, S3, SageMaker |
| Data Architecture | Write path (Kafka → Flink → S3 → Redshift) and Read path (EVCache → DynamoDB → OpenSearch) |
| Video Encoding Pipeline | 1,200+ encode jobs per title, Spot GPU fleet, VMAF quality control, AV1 adoption |
| Resilience Engineering | Chaos Monkey / Gorilla / Kong, Multi-Region Active architecture with Route 53 failover |
| Observability Stack | Atlas, Mantis, Hollow, CloudWatch, X-Ray, Kayenta canary analysis |
| Security Architecture | mTLS (Metatron), Shield Advanced, WAF, KMS encryption, Secrets Manager |
| Trade-offs Netflix Made | Build vs Buy decisions for CDN, API Gateway, container scheduler, metrics, caching |
| Key Lessons | 8 architectural principles derived from Netflix's 15 years on AWS |
