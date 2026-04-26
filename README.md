# AWS

A collection of AWS architecture concepts, system design references, and service trade-off guides.

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
