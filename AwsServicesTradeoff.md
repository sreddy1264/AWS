# AWS Services Trade-offs — Decision Guide

> Every AWS service choice is a trade-off. This guide covers the most common decisions you face when hosting an application on AWS, with clear **when to pick what** guidance.

---

## Table of Contents

1. [Compute — Lambda vs EC2 vs Fargate](#1-compute--lambda-vs-ec2-vs-fargate)
2. [Containers — ECS vs EKS vs Fargate](#2-containers--ecs-vs-eks-vs-fargate)
3. [Database — DynamoDB vs RDS vs Aurora](#3-database--dynamodb-vs-rds-vs-aurora)
4. [Messaging — SQS vs SNS vs EventBridge](#4-messaging--sqs-vs-sns-vs-eventbridge)
5. [API Layer — API Gateway vs ALB vs AppSync](#5-api-layer--api-gateway-vs-alb-vs-appsync)
6. [Load Balancers — ALB vs NLB vs CLB](#6-load-balancers--alb-vs-nlb-vs-clb)
7. [Storage — S3 vs EBS vs EFS](#7-storage--s3-vs-ebs-vs-efs)
8. [Caching — ElastiCache Redis vs Memcached vs DAX](#8-caching--elasticache-redis-vs-memcached-vs-dax)
9. [Streaming — Kinesis vs SQS vs MSK](#9-streaming--kinesis-vs-sqs-vs-msk)
10. [CDN & Edge — CloudFront vs Without CDN](#10-cdn--edge--cloudfront-vs-without-cdn)
11. [Orchestration — Step Functions vs SQS + Lambda](#11-orchestration--step-functions-vs-sqs--lambda)
12. [Auth — Cognito vs Custom Auth vs IAM](#12-auth--cognito-vs-custom-auth-vs-iam)
13. [Observability — CloudWatch vs X-Ray vs OpenTelemetry](#13-observability--cloudwatch-vs-x-ray-vs-opentelemetry)
14. [DNS Routing — Route 53 Routing Policies](#14-dns-routing--route-53-routing-policies)
15. [High Availability — Multi-AZ vs Multi-Region](#15-high-availability--multi-az-vs-multi-region)
16. [Search — OpenSearch vs RDS Full-Text vs Kendra](#16-search--opensearch-vs-rds-full-text-vs-kendra)
17. [Secrets — Secrets Manager vs Parameter Store vs Env Vars](#17-secrets--secrets-manager-vs-parameter-store-vs-env-vars)
18. [Deployment — CodeDeploy vs ECS Rolling vs Blue/Green](#18-deployment--codedeploy-vs-ecs-rolling-vs-bluegreen)
19. [Quick Decision Cheat Sheet](#19-quick-decision-cheat-sheet)

---

## 1. Compute — Lambda vs EC2 vs Fargate

### At a Glance

| Factor | Lambda | EC2 | Fargate |
|--------|--------|-----|---------|
| **Startup time** | Cold start: 100ms–3s | Minutes to provision | 30s–2min |
| **Max runtime** | 15 minutes | Unlimited | Unlimited |
| **Max memory** | 10 GB | Up to 24 TB | 120 GB |
| **Scaling** | Instant, automatic | Auto Scaling Groups (slower) | Task-level, fast |
| **Server management** | None | Full OS/patching | None |
| **Pricing model** | Per invocation + duration | Per hour (reserved/spot) | Per vCPU/memory/second |
| **Idle cost** | $0 | Paying even when idle | $0 when no tasks running |
| **Concurrency** | 1000 default (soft limit) | Depends on instance size | Depends on cluster |
| **State** | Stateless | Stateful possible | Stateless by default |

### Pick Lambda when
- Traffic is **spiky or unpredictable** — pay only for what you use
- Tasks complete in **< 15 minutes**
- You want **zero infrastructure** management
- Building **event-driven** pipelines (S3 triggers, SQS consumers, API Gateway backends)
- Running **scheduled jobs** (cron-style via EventBridge)

### Pick EC2 when
- You need **long-running processes** (daemons, WebSocket servers)
- Application has **specific OS or GPU** requirements (ML training, video encoding)
- You need **predictable, consistent** performance — no cold starts
- Running **licensed software** that charges per server (Oracle, Windows)
- Need **persistent local storage** (EBS volumes)

### Pick Fargate when
- You already have **Docker containers** and want serverless container execution
- Tasks run **longer than 15 min** but you don't want to manage EC2
- Need **more isolation** than Lambda (separate container per task)
- Batch jobs or **background workers** that are containerized

### The Core Trade-off

```
Lambda:   Low ops overhead ✅  |  Cold starts ⚠️  |  15-min limit ❌
EC2:      Full control ✅      |  Pay always ❌   |  Ops burden ❌
Fargate:  No servers ✅        |  Slower start ⚠️ |  Container overhead ⚠️
```

---

## 2. Containers — ECS vs EKS vs Fargate

### At a Glance

| Factor | ECS | EKS | ECS on Fargate |
|--------|-----|-----|----------------|
| **Orchestrator** | AWS proprietary | Kubernetes | AWS proprietary |
| **Control plane cost** | Free | $0.10/hr (~$73/mo) | Free |
| **Learning curve** | Low | High (K8s expertise) | Low |
| **Portability** | AWS lock-in | K8s standard (portable) | AWS lock-in |
| **Ecosystem** | AWS-native only | Helm, CNCF tools, vast | AWS-native only |
| **Node management** | You manage EC2 | You manage EC2 (or use Managed Nodes) | AWS manages |
| **Networking** | VPC native, simple | Complex (CNI plugins) | VPC native |
| **Auto-scaling** | Service Auto Scaling | HPA / KEDA | Service Auto Scaling |
| **Multi-cluster** | Manual setup | Easy with K8s tooling | Manual setup |

### Pick ECS when
- Team is **new to containers** and doesn't have K8s expertise
- You're **all-in on AWS** and don't need portability
- Simpler setup is more valuable than flexibility
- Small to **mid-size teams** where K8s overhead isn't justified

### Pick EKS when
- Team already has **Kubernetes experience**
- You need **multi-cloud portability** (run same workloads on GKE/AKS)
- Using **Helm charts**, service meshes (Istio, Linkerd), or CNCF ecosystem tools
- **Large-scale** microservices where K8s advanced scheduling pays off
- Regulatory needs require **on-prem + cloud** hybrid (K8s runs everywhere)

### Pick Fargate (with ECS or EKS) when
- You want **serverless container execution** — no EC2 nodes to patch/manage
- Workloads are **bursty** — Fargate scales fast without pre-provisioning
- Dev/staging environments where **cost at idle** matters
- Running **batch jobs** or one-off tasks

### The Core Trade-off

```
ECS:            Simple + AWS-native    |  Vendor lock-in    |  Limited ecosystem
EKS:            Portable + powerful    |  $73/mo minimum    |  Steep learning curve
Fargate:        Zero node management   |  Higher per-unit cost  |  Less control
```

> **Rule of thumb:** Start with ECS Fargate. Migrate to EKS only when you hit a concrete wall (K8s-specific tooling, multi-cloud requirement, or team already knows K8s).

---

## 3. Database — DynamoDB vs RDS vs Aurora

### At a Glance

| Factor | DynamoDB | RDS (MySQL/Postgres) | Aurora |
|--------|----------|----------------------|--------|
| **Type** | NoSQL (key-value / document) | Relational (SQL) | Relational (SQL) — AWS-optimized |
| **Schema** | Schema-less, flexible | Strict schema | Strict schema |
| **Scaling** | Automatic, unlimited | Manual vertical/read replicas | Auto-scales storage; read replicas |
| **Performance** | Single-digit ms at any scale | Predictable, can degrade under load | 5x MySQL / 3x Postgres throughput |
| **Max storage** | Unlimited | 64 TB | 128 TB (auto-grows) |
| **Pricing** | Per RCU/WCU or on-demand | Per instance/hour | Per instance/hour + I/O |
| **ACID transactions** | Limited (across ≤25 items) | Full ACID | Full ACID |
| **Joins** | Not supported | Full JOINs | Full JOINs |
| **Global distribution** | Global Tables (multi-region) | Cross-region read replicas | Aurora Global Database |
| **Serverless option** | Yes (always) | No | Aurora Serverless v2 |
| **Idle cost** | $0 (on-demand mode) | Paying even at idle | Paying even at idle |

### Pick DynamoDB when
- Access patterns are **known and simple** (lookup by user ID, session, etc.)
- Need **massive scale** with consistent low latency (millions of req/sec)
- Data model is **hierarchical or document-like** (user profiles, game state, IoT events)
- You want **true serverless** — no instance to size or manage
- Building **global apps** — DynamoDB Global Tables replicate across regions in <1s

### Pick RDS (MySQL / Postgres) when
- Data has **complex relationships** — JOINs are critical to your queries
- You need **full SQL** expressiveness (window functions, CTEs, aggregations)
- Team is **familiar with SQL** — migration from existing relational DB
- Application does **reporting / analytics** alongside transactions
- **Budget is tight** — RDS is simpler and cheaper to start than Aurora

### Pick Aurora when
- You need **RDS but with higher throughput** (5x MySQL, 3x Postgres)
- Mission-critical apps where **failover speed** matters (Aurora fails over in <30s vs RDS ~60–120s)
- Need **read scalability** — up to 15 Aurora read replicas vs 5 for RDS
- Want **Aurora Serverless v2** — scales compute up/down instantly to 0 (great for variable workloads)
- **Multi-region** with Aurora Global Database (< 1s replication lag)

### The Core Trade-off

```
DynamoDB:   Infinite scale ✅  |  No JOINs ❌        |  Access patterns upfront ⚠️
RDS:        Full SQL ✅        |  Manual scaling ⚠️  |  Vertical limit ⚠️
Aurora:     SQL + fast ✅      |  Higher cost ❌      |  AWS lock-in ⚠️
```

> **Decision rule:**
> - Simple lookups + massive scale → **DynamoDB**
> - Complex queries + existing SQL team → **RDS Postgres**
> - High throughput SQL + global + HA → **Aurora**

---

## 4. Messaging — SQS vs SNS vs EventBridge

### At a Glance

| Factor | SQS | SNS | EventBridge |
|--------|-----|-----|-------------|
| **Pattern** | Queue (point-to-point) | Pub/Sub (fan-out) | Event bus (routing rules) |
| **Consumers** | One consumer per message | Many subscribers | Many targets via rules |
| **Message retention** | Up to 14 days | No retention — fire and forget | No retention (archive optional) |
| **Ordering** | FIFO queue option | No ordering | No ordering |
| **Delivery guarantee** | At-least-once (exactly-once with FIFO) | At-least-once | At-least-once |
| **Filtering** | No built-in | Subscription filter policies | Rich content-based routing rules |
| **Sources** | AWS services + your app | AWS services + your app | 200+ AWS services + SaaS (Stripe, GitHub…) |
| **Max message size** | 256 KB | 256 KB | 256 KB |
| **Throughput** | Unlimited | Unlimited | Unlimited |
| **Dead-letter queue** | Yes | Yes (to SQS) | Yes (to SQS) |

### Pick SQS when
- You need to **decouple** a producer from a slow consumer (work queue)
- **Rate-limiting** downstream services — consumer processes at its own pace
- Tasks that need **retry logic** — failed messages go to DLQ
- **Exactly-once processing** matters — use SQS FIFO
- **One consumer** should handle each message (order processing, payment jobs)

### Pick SNS when
- One event needs to **fan out to multiple systems** simultaneously
- Sending **notifications** — email, SMS, push, HTTP endpoints
- **Broadcasting** the same message to Lambda + SQS + HTTP all at once
- SNS → SQS pattern: SNS fans out, SQS buffers for each consumer

### Pick EventBridge when
- Routing events based on **content / payload rules** (e.g., `order.status == "paid"`)
- Consuming events from **SaaS tools** (Stripe webhooks, GitHub events) natively
- Building **event-driven microservices** that react to AWS service state changes
- **Scheduled events** (EventBridge Scheduler replaces cron on Lambda)
- You want **schema registry** — define and version event shapes

### The Core Trade-off

```
SQS:          Durable queue ✅     |  One consumer per msg  |  No fan-out
SNS:          Fan-out ✅           |  No retention ❌       |  Limited routing
EventBridge:  Rich routing ✅      |  No retention ❌       |  Slightly higher latency
```

> **Common pattern:** `SNS → SQS` (fan-out + durability) or `EventBridge → SQS → Lambda` (routing + buffering + processing)

---

## 5. API Layer — API Gateway vs ALB vs AppSync

### At a Glance

| Factor | API Gateway (REST/HTTP) | ALB | AppSync |
|--------|------------------------|-----|---------|
| **Protocol** | HTTP / REST / WebSocket | HTTP / HTTPS / gRPC | GraphQL / WebSocket |
| **Best for** | Lambda backends, auth, throttling | EC2 / ECS / container backends | GraphQL APIs, real-time subscriptions |
| **Auth built-in** | Cognito, IAM, Lambda authorizer | None (use your app) | Cognito, IAM, OIDC, Lambda |
| **Throttling** | Per-method rate limits | None | Per-resolver |
| **Caching** | Yes (response cache) | No | Yes (per-resolver) |
| **Cost** | $3.50/million REST calls | $0.008/LCU-hour | $4/million query/mutation |
| **WebSocket** | Yes | No | Yes (subscriptions) |
| **VPC integration** | Via VPC Link | Native | No |
| **gRPC** | No | Yes | No |

### Pick API Gateway when
- Backend is **Lambda** (tight native integration)
- Need **request throttling, API keys**, usage plans per customer
- Want built-in **auth with Cognito** or custom Lambda authorizer
- Public-facing API that needs **WAF integration** per route

### Pick ALB when
- Backend is **EC2 or ECS containers**
- Need **gRPC** or WebSocket at scale
- Want **path / host routing** to multiple microservices
- **Lower cost** at high volume — ALB is cheaper than API Gateway at millions of req

### Pick AppSync when
- Building a **GraphQL API** — subscriptions, real-time data, offline sync
- Mobile/web clients need **real-time updates** (chat, live dashboards)
- Complex **data aggregation** from multiple sources in one query

### The Core Trade-off

```
API Gateway:  Lambda-native + auth ✅  |  Expensive at scale ❌  |  REST/HTTP only
ALB:          Cheap + flexible ✅      |  No built-in auth ❌    |  No serverless routing
AppSync:      GraphQL + real-time ✅   |  GraphQL-only ❌        |  Higher complexity
```

---

## 6. Load Balancers — ALB vs NLB vs CLB

### At a Glance

| Factor | ALB | NLB | CLB (Classic) |
|--------|-----|-----|---------------|
| **OSI Layer** | Layer 7 (HTTP/S) | Layer 4 (TCP/UDP) | Layer 4/7 (legacy) |
| **Protocol** | HTTP, HTTPS, WebSocket, gRPC | TCP, UDP, TLS | HTTP, HTTPS, TCP |
| **Routing** | Path, host, header, query-string | IP + port only | Basic round-robin |
| **Latency** | ~1ms added | Ultra-low (~microseconds) | Higher |
| **Static IP** | No (DNS only) | Yes — Elastic IPs | No |
| **TLS termination** | Yes | Yes (passthrough or terminate) | Yes |
| **WebSocket** | Yes | Yes | No |
| **Target types** | EC2, ECS, Lambda, IP | EC2, ECS, IP | EC2 only |
| **Use case** | Web apps, microservices | Gaming, IoT, ultra-low latency | Legacy (avoid) |

### Pick ALB when
- Hosting **web applications** — HTTP/HTTPS traffic
- Need **path-based routing** (`/api` → service A, `/static` → service B)
- Backend targets include **Lambda**
- Need **header / query string routing** for A/B testing

### Pick NLB when
- Need **static IP addresses** (firewall whitelisting by IP)
- **Ultra-low latency** or high-throughput non-HTTP (gaming, VoIP, IoT)
- **TCP/UDP** protocols (databases, custom protocols)
- Need to **preserve client source IP** (NLB passes it through; ALB replaces it)

### Pick CLB — Never, use ALB or NLB instead (CLB is legacy).

---

## 7. Storage — S3 vs EBS vs EFS

### At a Glance

| Factor | S3 | EBS | EFS |
|--------|----|----|-----|
| **Type** | Object storage | Block storage (disk) | Network file system (NFS) |
| **Access** | HTTP API | Single EC2 instance | Multiple EC2 instances simultaneously |
| **Latency** | Milliseconds | Sub-millisecond | Low (but higher than EBS) |
| **Max size** | Unlimited | 64 TB per volume | Unlimited |
| **Durability** | 11 nines (99.999999999%) | 99.999% | 99.999999999% |
| **Pricing** | $0.023/GB | $0.08–0.10/GB | $0.30/GB (standard) |
| **Use case** | Files, media, backups, static sites | Boot volumes, databases, single-app storage | Shared filesystems, CMS, home directories |
| **Lifecycle policies** | Yes (auto-move to Glacier) | No | Yes |

### Pick S3 when
- Storing **images, videos, PDFs**, any unstructured files
- Hosting a **static website**
- **Data lake** or analytics source (Athena, Redshift Spectrum)
- **Backups and archives** (use S3 Glacier for long-term cheap storage)
- Sharing files **across regions** or publicly

### Pick EBS when
- **EC2 boot volume** (required)
- **Database storage** on EC2 (MySQL on EC2 needs EBS)
- Single-instance app needing **fast local-disk-like access**
- Low-latency **random read/write** workloads

### Pick EFS when
- **Multiple EC2 instances** must share the same files simultaneously
- Running a **CMS (WordPress)** across a fleet of web servers
- **Container workloads** where multiple tasks need shared persistent storage
- Shared **home directories** or build artifact caches

---

## 8. Caching — ElastiCache Redis vs Memcached vs DAX

### At a Glance

| Factor | Redis | Memcached | DAX |
|--------|-------|-----------|-----|
| **Data structures** | Strings, lists, sets, sorted sets, hashes, streams | Strings only | DynamoDB-native |
| **Persistence** | Yes (AOF / RDB snapshots) | No | No |
| **Replication** | Yes (primary + replicas) | No | Yes |
| **Pub/Sub** | Yes | No | No |
| **Clustering** | Yes | Yes (simpler) | Yes |
| **Use case** | Sessions, leaderboards, rate-limiting, pub/sub | Simple object cache, high throughput | DynamoDB read-through cache |
| **Failover** | Automatic with replication | Manual | Automatic |

### Pick Redis when
- Storing **user sessions** (survive restarts with persistence)
- Building **leaderboards** (sorted sets)
- **Rate limiting** / token bucket algorithms
- **Pub/Sub** messaging between services
- Need **data persistence** — cache should survive a restart

### Pick Memcached when
- Simple **key-value object cache** with no need for persistence
- Need **multi-threaded** cache that scales with CPU cores
- Willing to accept data loss on restart — pure ephemeral cache

### Pick DAX when
- Using **DynamoDB** and need **microsecond read latency** (down from single-digit ms)
- **Read-heavy DynamoDB tables** — DAX is a fully managed write-through cache in front of DynamoDB
- You don't want to change your DynamoDB SDK calls (DAX is API-compatible)

---

## 9. Streaming — Kinesis vs SQS vs MSK

### At a Glance

| Factor | Kinesis Data Streams | SQS | MSK (Managed Kafka) |
|--------|---------------------|-----|---------------------|
| **Pattern** | Ordered stream, multiple consumers | Queue, one consumer | Ordered stream, multiple consumer groups |
| **Retention** | 1–365 days | Up to 14 days | Configurable (unlimited with tiered) |
| **Ordering** | Per shard, guaranteed | FIFO queue option | Per partition, guaranteed |
| **Replay** | Yes — re-read old data | No — consumed once | Yes — re-read old data |
| **Throughput** | 1 MB/s write per shard | Unlimited | Very high (topic partitions) |
| **Latency** | ~200ms | ~0ms–minutes | ~5ms |
| **Management** | Semi-managed (shard management) | Fully managed | Semi-managed (broker management) |
| **Ecosystem** | AWS-native only | AWS-native | Kafka ecosystem (Kafka Streams, ksqlDB) |
| **Cost** | Per shard-hour | Per million messages | Per broker-hour |

### Pick Kinesis when
- **Real-time analytics** — clickstream, metrics, log aggregation
- Need **ordered processing** with multiple consumers reading the same stream
- Want **replay** — reprocess historical events
- AWS-native stack, no Kafka expertise on team

### Pick SQS when
- Simple **work queue** — tasks to be processed once
- No need for replay or ordering (or acceptable with FIFO)
- Consumer count is small — **one system processes each message**

### Pick MSK (Kafka) when
- Team has **Kafka expertise** and needs the full ecosystem
- Need **Kafka Streams** or **ksqlDB** for stream processing
- **Multi-region / multi-cloud** Kafka topic mirroring
- Extremely high throughput needing **Kafka's partition model**

---

## 10. CDN & Edge — CloudFront vs Without CDN

### Should You Use CloudFront?

| Scenario | Use CloudFront | Skip CloudFront |
|----------|---------------|-----------------|
| Serving static assets (JS, CSS, images) | ✅ Yes — massive latency reduction | ❌ Slow for global users |
| Global user base | ✅ Yes — edge locations in 450+ cities | ❌ All traffic hits your origin |
| DDoS protection | ✅ Yes — Shield Standard is free with CF | ❌ Origin is directly exposed |
| Dynamic API responses | ⚠️ Maybe — CF can forward with low cache TTL | ✅ Fine to skip |
| Internal / intranet app | ❌ No benefit | ✅ Skip it |
| Low traffic / single region | ❌ Adds complexity | ✅ Keep it simple |

### CloudFront Cache Behaviors Trade-off

```
Short TTL (seconds):   Fresh content ✅  |  More origin requests, higher cost ❌
Long TTL (hours/days): Cheap + fast ✅   |  Stale content risk ❌  → Use cache invalidation on deploy
```

---

## 11. Orchestration — Step Functions vs SQS + Lambda

### At a Glance

| Factor | Step Functions | SQS + Lambda |
|--------|---------------|--------------|
| **Visibility** | Visual workflow map, execution history | No built-in visibility |
| **Error handling** | Built-in retry, catch, timeout per state | Manual DLQ + retry logic |
| **Cost** | $0.025 per 1000 state transitions | SQS + Lambda costs separately |
| **Complexity** | Low for complex flows | High — you wire retries manually |
| **Long-running** | Up to 1 year | Lambda 15-min limit |
| **Parallel execution** | Built-in parallel branches | Manual fan-out pattern |
| **Debugging** | Full execution history in console | CloudWatch Logs only |

### Pick Step Functions when
- **Multi-step workflows** with retries, timeouts, and error handling per step
- **Human approval** workflows (wait for callback)
- Need **visual audit trail** of every execution
- Long-running processes that span days/weeks

### Pick SQS + Lambda when
- Simple **one-step async task** (send email, resize image)
- Cost sensitivity — Step Functions charge per state transition
- High-volume **simple fan-out** where SF transitions would be expensive

---

## 12. Auth — Cognito vs Custom Auth vs IAM

| Factor | Cognito | Custom Auth (your own) | IAM |
|--------|---------|----------------------|-----|
| **For** | End users (B2C / B2B) | End users | AWS services / machines |
| **Setup** | Low — managed service | High — build it yourself | Low — policy documents |
| **Federation** | Google, Facebook, SAML, OIDC | Whatever you build | IAM Identity Center (SSO) |
| **MFA** | Built-in | Build yourself | Built-in |
| **Cost** | Free up to 50k MAU, then per MAU | Your infra cost | Free |
| **Customization** | Limited UI, Lambda triggers for logic | Full control | N/A |

### Pick Cognito when
- Building a **customer-facing app** needing sign-up / sign-in / social login
- Don't want to build auth from scratch
- Need **JWT tokens** consumed by API Gateway or AppSync

### Build Custom Auth when
- Cognito's **limitations are blockers** (complex flows, full UI control, complex attribute mapping)
- You need **advanced session management** Cognito doesn't support

### Use IAM when
- **Service-to-service** auth within AWS (Lambda calling DynamoDB, EC2 calling S3)
- **Never use long-lived access keys** in code — use IAM roles instead

---

## 13. Observability — CloudWatch vs X-Ray vs OpenTelemetry

| Tool | What it gives you | Best for |
|------|------------------|---------|
| **CloudWatch Metrics** | Time-series numbers (CPU, RPS, errors) | Alarms, dashboards, auto-scaling triggers |
| **CloudWatch Logs** | Log storage + search + Insights queries | Debug errors, query logs with SQL-like syntax |
| **CloudWatch Alarms** | Trigger notifications / actions on metric thresholds | PagerDuty, SNS alerts, auto-scaling |
| **X-Ray** | Distributed trace map across services | Find which service adds latency in a chain |
| **OpenTelemetry (ADOT)** | Vendor-neutral traces + metrics | Multi-cloud or when you want portability |

### Core Trade-off

```
CloudWatch only:  Simple + integrated ✅  |  No traces ❌  |  Log search is slow ⚠️
X-Ray:            Traces ✅               |  AWS-only ⚠️   |  Limited custom spans
OpenTelemetry:    Portable ✅             |  More setup ❌  |  Future-proof
```

> Start with CloudWatch Logs + Metrics. Add X-Ray when debugging latency across Lambda / API Gateway chains. Add OpenTelemetry if you plan multi-cloud.

---

## 14. DNS Routing — Route 53 Routing Policies

| Policy | How it works | Use case |
|--------|-------------|---------|
| **Simple** | Returns one record, no logic | Single-region app, no failover |
| **Weighted** | Split traffic by % across records | A/B testing, blue/green, canary deployments |
| **Latency** | Routes to region with lowest latency to user | Multi-region app, performance optimization |
| **Failover** | Primary + standby — health check driven | Disaster recovery |
| **Geolocation** | Route based on user's country/continent | Data sovereignty, localized content |
| **Geoproximity** | Route based on geographic distance + bias | Fine-grained traffic shifting between regions |
| **IP-based** | Route based on client IP CIDR | ISP-specific routing, on-prem integration |
| **Multivalue** | Returns multiple IPs, health-checked | Client-side load balancing, basic HA |

---

## 15. High Availability — Multi-AZ vs Multi-Region

| Factor | Multi-AZ | Multi-Region |
|--------|----------|-------------|
| **Protects against** | AZ failure (data center fire, power outage) | Region failure (rare but catastrophic) |
| **Failover time** | Seconds to minutes (automatic) | Minutes (Route 53 health check + failover) |
| **Data replication** | Synchronous (no data loss) | Asynchronous (small data loss possible) |
| **Complexity** | Low — AWS manages it | High — you manage data sync, failover logic |
| **Cost** | ~2x (standby instance) | ~2–3x (full stack in second region) |
| **RTO / RPO** | RTO: seconds / RPO: 0 | RTO: minutes / RPO: seconds to minutes |
| **Required for** | Production workloads | Mission-critical / financial / healthcare |

### The Core Trade-off

```
Single-AZ:    Cheapest ✅        |  Any AZ outage = downtime ❌
Multi-AZ:     HA for most ✅     |  2x cost ⚠️                |  Same-region only
Multi-Region: Maximum HA ✅      |  3x cost + complexity ❌   |  Data sync challenges
```

---

## 16. Search — OpenSearch vs RDS Full-Text vs Kendra

| Factor | OpenSearch (Elasticsearch) | RDS Full-Text Search | Kendra |
|--------|---------------------------|---------------------|--------|
| **Type** | Distributed search engine | SQL `LIKE` / `tsvector` | AI-powered enterprise search |
| **Query power** | Full-text, fuzzy, facets, aggregations | Basic text matching | Natural language questions |
| **Scaling** | Horizontal — add nodes | Vertical — bigger instance | Fully managed |
| **Cost** | Per node/hour | Included in RDS | $1/hour minimum (expensive) |
| **Setup** | Moderate (index management) | Simple | Low (SaaS-like) |
| **Use case** | Log analytics, product search, autocomplete | Simple `WHERE name LIKE '%query%'` | Enterprise document/knowledge search |

---

## 17. Secrets — Secrets Manager vs Parameter Store vs Env Vars

| Factor | Secrets Manager | SSM Parameter Store | Environment Variables |
|--------|----------------|--------------------|-----------------------|
| **Auto-rotation** | Yes (built-in for RDS, Redshift) | No | No |
| **Encryption** | KMS always | KMS (SecureString tier) | Plaintext (bad practice) |
| **Audit** | CloudTrail | CloudTrail | None |
| **Cost** | $0.40/secret/month | Free (standard) / $0.05 (advanced) | Free |
| **Max size** | 64 KB | 4 KB (standard) / 8 KB (advanced) | Varies |
| **Use case** | DB passwords, API keys needing rotation | App config, feature flags, non-rotating secrets | Local dev only |

### Rule
- **Rotating DB passwords** → Secrets Manager
- **App config / feature flags** → Parameter Store (free)
- **Never** store secrets in environment variables in production (they appear in logs, crash reports)

---

## 18. Deployment — CodeDeploy vs ECS Rolling vs Blue/Green

| Strategy | Downtime | Rollback speed | Risk | Use case |
|----------|----------|---------------|------|----------|
| **In-place (rolling)** | No (gradual) | Slow — re-deploy old version | Medium | Low-stakes apps, quick deploys |
| **Blue/Green** | Zero | Instant — flip DNS back | Low | Production APIs, any user-facing app |
| **Canary** | Zero | Fast — shift traffic back | Lowest | High-risk changes, gradual confidence |
| **All-at-once** | Yes (brief) | N/A | Highest | Dev/test environments only |

### Pick Blue/Green when
- **Zero-downtime** deployments are required
- Need **instant rollback** on failure — just flip the load balancer
- Running stateless services (EC2, ECS, Lambda with traffic shifting)

### Pick Canary when
- Releasing a **high-risk change** and want real traffic validation at 5% before full rollout
- Lambda aliases or CodeDeploy canary deployments make this simple

---

## 19. Quick Decision Cheat Sheet

```
COMPUTE
  Spiky/event-driven, < 15 min           → Lambda
  Long-running, OS control needed        → EC2
  Dockerized, no node management         → Fargate

CONTAINERS
  New to containers, AWS-only            → ECS
  K8s expertise, multi-cloud needed      → EKS
  Serverless containers                  → ECS/EKS + Fargate

DATABASE
  Simple lookups, massive scale          → DynamoDB
  Complex SQL joins, existing SQL team   → RDS Postgres
  High-throughput SQL, global, HA        → Aurora

MESSAGING
  Work queue, one consumer               → SQS
  Fan-out to multiple systems            → SNS  (or SNS → SQS)
  Content-based routing, SaaS events     → EventBridge

LOAD BALANCER
  Web/HTTP app, path routing             → ALB
  Static IP, TCP/UDP, ultra-low latency  → NLB
  Legacy                                 → Avoid CLB

STORAGE
  Files, media, backups, static site     → S3
  EC2 boot volume, single-app disk       → EBS
  Shared filesystem across EC2 fleet     → EFS

CACHING
  Sessions, leaderboards, pub/sub        → Redis (ElastiCache)
  Simple ephemeral object cache          → Memcached
  DynamoDB read acceleration             → DAX

STREAMING
  Real-time analytics, replay needed     → Kinesis
  Simple async work queue                → SQS
  Kafka ecosystem, team knows Kafka      → MSK

SEARCH
  Product search, autocomplete, logs     → OpenSearch
  Simple text filter on small dataset    → RDS full-text
  Enterprise document/knowledge search   → Kendra

SECRETS
  Rotating DB passwords                  → Secrets Manager
  App config, feature flags              → SSM Parameter Store
  Never in production                    → Environment variables

HA STRATEGY
  Standard production                    → Multi-AZ
  Mission-critical, 99.99%+ uptime       → Multi-Region
  Dev/test                               → Single-AZ is fine
```

---

> **Golden Rule:** Choose the simplest service that meets your requirements. Complexity compounds — every added service is an extra failure mode, cost line, and operational burden. Add services when you hit a concrete wall, not in anticipation of future scale.
