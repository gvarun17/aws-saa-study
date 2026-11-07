# Day 9 — Containers & the Edge: ECS, EKS, Fargate, Route 53, CloudFront, Global Accelerator

**Domain weight:** mainly *Design High-Performing Architectures* (24% — "global users", "reduce latency", "scale containers") and *Design Resilient Architectures* (26% — DNS failover, multi-Region, origin failover). *Secure* (30%) shows up through OAC, signed URLs/cookies, task roles and geo restriction; *Cost* (20%) through Fargate vs EC2, Fargate Spot, CloudFront price classes and egress savings.
**Time plan:** Learn 90 min · Lab 30 min · Drill 45 min · Review 30 min.

By the end of today you should be able to:

- Choose between ECS on EC2, ECS on Fargate, EKS, Lambda and AWS Batch from one scenario sentence — and wire up the right IAM role (task role vs task execution role).
- Get containers in private subnets pulling from ECR without a NAT gateway, and scale an ECS service on load.
- Pick any Route 53 routing policy, explain alias vs CNAME, and design health-check-driven failover (including for private resources).
- Put CloudFront in front of S3 and ALB securely (OAC, VPC origins, prefix lists), control caching, and choose signed URLs vs signed cookies and CloudFront Functions vs Lambda@Edge.
- Decide between CloudFront, Global Accelerator, S3 Transfer Acceleration and Route 53 latency routing for a "global performance" question.

---

## 1. The mental model: one request, front to back

Most questions today are about *where in this path* a feature lives.

```
 User (Mumbai)
   │  1. DNS: "app.example.com?"
   ▼
 Route 53 ──────── routing policy picks a target (latency / failover / geo / weighted)
   │  2. answer: d111.cloudfront.net  (or GA anycast IPs, or an ALB)
   ▼
 Edge location (nearest PoP)
   ├─ CloudFront: cache HIT → respond from edge (static + cacheable dynamic, HTTP/S only)
   │              cache MISS → regional edge cache → origin
   └─ Global Accelerator: no cache; proxies TCP/UDP onto the AWS backbone to the best healthy Region
   │
   ▼  3. AWS global network (not the public internet)
 Region ap-south-1
   └─ ALB (public subnets) ──► ECS service: tasks on Fargate (private subnets, awsvpc ENIs)
                                   │ task role → DynamoDB / S3
                                   └ images from ECR via VPC endpoints
```

Three rules to keep in your head:

1. **DNS (Route 53) decides *where* to go; it can't make the trip faster.** Clients and resolvers cache answers for the TTL, so DNS failover is never instant.
2. **CloudFront makes HTTP(S) faster by caching *at the edge*.** It also hides and protects the origin.
3. **Global Accelerator makes any TCP/UDP traffic faster by *routing* over AWS's network**, with two static IPs and failover in seconds — but it caches nothing.

---

## 2. Containers on AWS

### The landscape

| Option | You manage | Pick it when |
| --- | --- | --- |
| **ECS on EC2** | EC2 instances (AMI, patching, scaling the cluster) | You need GPUs/specific instance types, very high steady utilisation, daemon-style agents, or cheapest $/vCPU with RIs/Savings Plans |
| **ECS on Fargate** | Nothing below the task — no servers | "Run containers without managing servers", **least operational overhead** |
| **EKS** (on EC2 nodes, Fargate, or Auto Mode) | Kubernetes objects; nodes unless Fargate/Auto Mode | Team already uses **Kubernetes**, needs portability, Helm/operators, open-source K8s tooling |
| **AWS Lambda** (incl. container images up to 10 GB) | Nothing | Event-driven, short (≤ 15 min), spiky — Day 8 |
| **AWS Batch** | Job definitions/queues; Batch runs ECS/EKS/Fargate for you | Large-scale batch jobs (rendering, genomics, simulations) with queues and retries |
| **ECS Anywhere / EKS Anywhere / EKS Hybrid Nodes** | Your on-prem hardware | Same orchestration on premises |

> Exam reflex: *"containers" + "no servers to manage" → **Fargate***. *"Kubernetes" anywhere in the question → **EKS***. ECS vs EKS is almost never decided on performance — it's decided on "do they already use Kubernetes?"

### Amazon ECR (Elastic Container Registry)

- Private (and public) registry for OCI/Docker images, stored durably in S3 behind the scenes.
- **Image scanning:** basic, or **enhanced** scanning via Amazon Inspector (OS + language packages, continuous).
- **Lifecycle policies:** expire untagged/old images (cost control).
- **Replication:** cross-Region and cross-account (multi-Region deployments, DR).
- **Pull-through cache:** cache images from public registries (Docker Hub, ECR Public, Quay…) in your private ECR.
- Image tag immutability: prevent `:latest`-style overwrites.

**Classic trap — ECR from private subnets with no NAT:** tasks fail with `CannotPullContainerError`. Image *layers* live in S3, so you need **all** of:

| Endpoint | Type | Why |
| --- | --- | --- |
| `com.amazonaws.<region>.ecr.api` | Interface | ECR API calls (auth token) |
| `com.amazonaws.<region>.ecr.dkr` | Interface | Docker registry protocol |
| `com.amazonaws.<region>.s3` | **Gateway** | Image layers (free gateway endpoint) |
| `com.amazonaws.<region>.logs` | Interface | awslogs driver → CloudWatch Logs |
| `secretsmanager` / `ssm` | Interface | If the task definition injects secrets |

### ECS building blocks

```
Cluster (logical grouping, per Region)
 ├─ Capacity: Fargate / Fargate Spot / EC2 Auto Scaling group(s) via capacity providers
 └─ Service "web" (desired count 4, spread across AZs, registered in an ALB target group)
      └─ Task  = running copy of a Task Definition
           └─ Task Definition (versioned JSON): image(s), CPU/memory, ports,
              env, secrets, logging, volumes, taskRoleArn, executionRoleArn, networkMode
```

- **Task definition** = the blueprint (like a launch template for containers). One task can hold several containers (sidecars: log router, proxy).
- **Service** = keeps N tasks running, replaces failed ones, integrates with ELB, does rolling or blue/green (CodeDeploy) deployments, with a deployment **circuit breaker** that can roll back failed deployments automatically.
- **Standalone task / scheduled task** = run-to-completion jobs, e.g. triggered by EventBridge Scheduler.

### The two IAM roles (most-tested ECS fact)

| Role | Used by | Typical permissions |
| --- | --- | --- |
| **Task execution role** (`executionRoleArn`) | The **ECS agent / Fargate** *before and around* your code | Pull from ECR, write to CloudWatch Logs, read Secrets Manager / Parameter Store values to inject |
| **Task role** (`taskRoleArn`) | **Your application code** inside the container | `dynamodb:PutItem`, `s3:GetObject`, `sqs:SendMessage`… |

"The app in the container gets AccessDenied calling S3" → fix the **task role**. "The task can't start: can't pull image / can't fetch secret" → fix the **execution role** (or networking). Never use the EC2 instance profile to give containers app permissions — every task on that host would inherit it.

```json
{
  "family": "web",
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "512", "memory": "1024",
  "executionRoleArn": "arn:aws:iam::111122223333:role/ecsTaskExecutionRole",
  "taskRoleArn":      "arn:aws:iam::111122223333:role/web-app-role",
  "containerDefinitions": [{
    "name": "web",
    "image": "111122223333.dkr.ecr.ap-south-1.amazonaws.com/web:1.4.2",
    "portMappings": [{ "containerPort": 8080 }],
    "secrets": [{ "name": "DB_PASSWORD",
                  "valueFrom": "arn:aws:secretsmanager:ap-south-1:111122223333:secret:prod/db-AbCdEf" }],
    "logConfiguration": { "logDriver": "awslogs",
      "options": { "awslogs-group": "/ecs/web", "awslogs-region": "ap-south-1", "awslogs-stream-prefix": "web" } }
  }]
}
```

### Networking modes

| Mode | What you get | Notes |
| --- | --- | --- |
| **awsvpc** | Each task gets its **own ENI, private IP and security groups** | **Required for Fargate**; recommended everywhere. ALB target type = **ip**. Watch ENI limits per instance on EC2 (ENI trunking helps). |
| **bridge** | Docker bridge on the host; **dynamic host port mapping** | EC2 launch type only. ALB target type = instance; many copies of the same container per host on random ports. |
| **host** | Container uses host's network directly | One container per port per host. |

> Trap: "security groups at the **task** level" → awsvpc. "Many tasks of one service on a single EC2 host behind an ALB" → dynamic port mapping (bridge) or awsvpc with ip targets.

### Scaling ECS — two layers

```
   Service auto scaling (Application Auto Scaling)          Cluster capacity (EC2 launch type only)
   ─────────────────────────────────────────────            ───────────────────────────────────────
   target tracking: avg CPU 60%, or                         Capacity provider with MANAGED SCALING
   ALBRequestCountPerTarget = 1000, or                      grows/shrinks the ASG so pending tasks
   step scaling on SQS ApproximateNumberOfMessagesVisible   have somewhere to run
          │ changes desiredCount                                     │
          └──────────────► more tasks ─── need room? ────────────────┘
                          (Fargate: AWS finds room — nothing to scale)
```

- With **Fargate**, only the *service* layer exists — another reason it's "least overhead".
- **Fargate Spot:** spare capacity at a steep discount, **can be interrupted with a 2-minute warning** (SIGTERM). Mix via capacity provider strategy, e.g. `base: 2 on FARGATE, weight 1 FARGATE : 3 FARGATE_SPOT`. Good for stateless, fault-tolerant tasks; bad for "must not be interrupted".
- **Cost:** Fargate charges per vCPU-second and GB-second of the task size; covered by **Compute Savings Plans** (Day 2). Graviton (ARM64) tasks are cheaper — check current pricing.

### Storage for containers

| Need | Answer |
| --- | --- |
| Scratch space | Task ephemeral storage (Fargate: 20 GiB default, configurable higher — check docs for the max) |
| **Shared, persistent** files across many tasks/AZs | **Amazon EFS** volume in the task definition (works with Fargate and EC2) |
| Per-task block volume (e.g. high-IOPS scratch DB) | **EBS volume attached at task launch** (supported for ECS, incl. Fargate) |
| Windows file shares | FSx for Windows File Server (EC2 launch type) |
| Container state that must survive task replacement | Never local disk — EFS, S3, or a database |

### Service-to-service communication

- **ALB** for north–south HTTP traffic (path/host routing to different services).
- **ECS Service Connect** (built on AWS Cloud Map) or **Cloud Map** DNS service discovery for east–west calls between services by name.
- AWS App Mesh is being retired — prefer Service Connect for new designs (check current docs).

### Amazon EKS in one screen

- AWS runs the **Kubernetes control plane** (API servers, etcd) across 3 AZs; you pay a per-cluster hourly fee plus compute.
- **Data plane options:**

| Option | What it is | Use when |
| --- | --- | --- |
| Managed node groups | AWS-managed ASGs of EC2 nodes, easy upgrades | Default for EC2-based nodes |
| Self-managed nodes | Your own ASGs/AMIs | Custom AMIs, special needs |
| **Fargate profiles** | Pods matching a namespace/label run on Fargate | No nodes to manage (no DaemonSets, no GPUs) |
| **EKS Auto Mode** | AWS manages nodes, scaling (Karpenter-based), core add-ons | Least-overhead Kubernetes on EC2 |
| Karpenter | Open-source node autoscaler, provisions right-sized nodes fast | Efficient scaling, Spot mixes |

- **Pod-level IAM:** **EKS Pod Identity** (newer, simpler) or **IRSA** (IAM Roles for Service Accounts via OIDC). Same lesson as the ECS task role: never rely on the node's instance role for app permissions.
- Load balancing: **AWS Load Balancer Controller** creates ALBs (Ingress) and NLBs (Service type LoadBalancer).
- Storage: EBS CSI driver (per-pod, single AZ), EFS CSI driver (shared, multi-AZ).
- Scaling: Horizontal Pod Autoscaler (pods) + Cluster Autoscaler/Karpenter (nodes).

---

## 3. Amazon Route 53

Route 53 is AWS's global DNS: domain registration, authoritative DNS (hosted zones), health checks, and Resolver for VPC/hybrid DNS (Day 4). Its SLA targets **100% availability** for DNS query answering (check the current SLA page).

### Hosted zones and records

| Hosted zone | Answers queries from | Use |
| --- | --- | --- |
| **Public** | The internet | `www.example.com` → CloudFront/ALB |
| **Private** | Only VPCs you associate (needs `enableDnsHostnames` + `enableDnsSupport`) | `db.internal.example.com` → RDS endpoint; can be associated cross-account |

Records to know: **A** (IPv4), **AAAA** (IPv6), **CNAME** (name → name), **MX**, **TXT**, **NS**, **SOA**, **CAA** (which CAs may issue certificates), **SRV**, plus Route 53's **alias**.

### Alias vs CNAME (tested every exam)

| | **Alias** (Route 53 extension) | **CNAME** |
| --- | --- | --- |
| Zone apex (`example.com`) | **Yes** | **No** — DNS standard forbids it |
| Targets | AWS resources: ELB, CloudFront, API Gateway, S3 **website** endpoint, Elastic Beanstalk, Global Accelerator, VPC interface endpoints, App Runner, another record in the same zone | Any hostname |
| Query cost | **Free** for queries to AWS resources | Charged |
| TTL | Inherited from target (you can't set it) | You set it |
| Health | "Evaluate target health" uses the target's health (e.g. ALB with no healthy targets) | Needs a separate health check |
| Record type | A / AAAA (it returns IPs) | CNAME |

> Trap: alias **cannot** point to an EC2 instance's public DNS name. Use an A record to an Elastic IP, or put the instance behind an ALB.

### Routing policies

| Policy | How it decides | Exam cue | Health checks? |
| --- | --- | --- | --- |
| **Simple** | One record, possibly several IPs returned in random order | One resource, no logic | No (on the record) |
| **Weighted** | Proportion by weight (e.g. 90/10; weight 0 = no traffic) | Canary / blue-green / A-B testing at DNS | Yes |
| **Latency** | Region with lowest measured latency *from the resolver* | "Best performance for global users" across Regions | Yes |
| **Failover** | Primary while healthy, else secondary | Active-passive DR | **Required** on primary |
| **Geolocation** | Continent / country / US state of the user | Compliance, localisation, licensing ("EU users must hit EU") — **add a Default record** or unmatched users get no answer | Yes |
| **Geoproximity** | Distance to resources, adjustable with a **bias** to grow/shrink a Region's area | "Shift more traffic to one Region based on geography" (historically via Traffic Flow; check current console) | Yes |
| **Multivalue answer** | Up to 8 healthy records, random | Simple client-side load spreading with health checks — *not* a substitute for ELB | Yes |
| **IP-based** | Client's source IP CIDR (CIDR collections) | Route specific ISPs/networks to specific endpoints | Yes |

> **Latency vs geolocation:** latency = *fastest*; geolocation = *where the user is*, regardless of speed. "Must", "regulation", "language", "content rights" → geolocation.

Policies combine in trees: e.g. **latency** between Regions, each pointing to a **weighted** or **failover** set. Route 53 **Traffic Flow** gives a visual editor for these trees.

### Health checks

| Type | Checks | Use |
| --- | --- | --- |
| **Endpoint** | HTTP/HTTPS/TCP from health checkers in multiple Regions; optional string match on the first bytes of the body | Public endpoints |
| **Calculated** | Combines child health checks (AND/OR/at least N) | "Healthy if 2 of 3 Regions are up" |
| **CloudWatch alarm** | State of a CloudWatch alarm | **Private resources** (health checkers live on the internet and can't reach your VPC), or any custom metric (e.g. DynamoDB throttles) |

Defaults to know (check docs for options): 30-second interval (10-second "fast" option), failure threshold of 3. Health checkers must be allowed through your security groups / NACLs. A health check can trigger a CloudWatch alarm → SNS.

### Active-passive failover

```
 example.com  (failover routing)
   ├─ PRIMARY  alias → ALB ap-south-1      + evaluate target health / health check
   └─ SECONDARY alias → ALB ap-southeast-1  (warm standby)  or  S3 static "sorry" site
        ▲
   Route 53 stops returning PRIMARY once its health check fails;
   clients switch after their cached answer's TTL expires → keep TTLs low (e.g. 60 s).
```

### Other Route 53 features

- **Domain registration** and transfer; **DNSSEC** signing for public zones (protects against spoofed responses).
- **Route 53 Resolver**: inbound/outbound endpoints for hybrid DNS, DNS Firewall for filtering outbound queries (Day 4).
- **Route 53 Application Recovery Controller (ARC):** readiness checks and highly available **routing controls** for manual, controlled multi-Region failover — the "we want a deliberate, reliable switch-over, not automatic DNS health" answer.
- Query logging to CloudWatch Logs / S3.

---
## 4. Amazon CloudFront

CloudFront is a global CDN: hundreds of edge locations plus **regional edge caches** (a larger mid-tier cache between edges and your origin). It serves static and dynamic content, APIs, video and WebSockets over HTTP/1.1, HTTP/2 and HTTP/3.

### How a request flows

```
Viewer ──► Edge location ── HIT ──► response (fast, no origin cost)
               │ MISS
               ▼
        Regional edge cache ── HIT ──► back to edge, cached there too
               │ MISS
               ▼
        (optional) Origin Shield — one extra central cache layer in a chosen Region
               │
               ▼
        Origin: S3 bucket · ALB · EC2 · API Gateway · Lambda function URL ·
                MediaStore/MediaPackage · any HTTP server (incl. on-prem)
```

Even uncacheable dynamic requests get faster: TLS terminates near the user and CloudFront keeps warm, persistent connections to the origin over the AWS network.

### Distributions, origins and behaviors

- A **distribution** has one or more **origins** and an ordered list of **cache behaviors** matched by path pattern:

| Path pattern | Origin | Caching |
| --- | --- | --- |
| `/static/*` | S3 bucket (OAC) | Long TTL, CachingOptimized |
| `/api/*` | ALB | CachingDisabled, forward auth headers |
| `Default (*)` | ALB | Short TTL |

- **Cache key** = what makes two requests "the same object". Configured by a **cache policy**: which headers, cookies and query strings are part of the key, plus min/default/max TTL.
- **Origin request policy** = what extra values are *forwarded* to the origin **without** being in the cache key.
- **Response headers policy** = add CORS/security headers (HSTS, X-Frame-Options) without code.

> **Cache hit ratio trap:** forwarding *all* headers/cookies/query strings into the cache key kills the hit ratio. "Improve cache hit ratio" → include only the values the origin actually varies on; use Origin Shield for many-edge, single-origin workloads.

- **TTL:** origin `Cache-Control`/`Expires` headers, bounded by the policy's min/max TTL.
- **Invalidation** (`/images/*`) removes objects early — has a monthly free allowance, then a per-path charge (check current pricing). For frequent deployments, **versioned file names** (`app.3f9c.js`) are cheaper and instant.

### Securing the origin

**S3 origin — Origin Access Control (OAC):**

- Bucket stays private (Block Public Access on). CloudFront signs requests to S3 with SigV4; the bucket policy allows only `cloudfront.amazonaws.com` with `AWS:SourceArn` = your distribution (snippet in Day 5).
- OAC supports **SSE-KMS** (add the CloudFront service principal to the **KMS key policy**), all Regions, and PUT/POST. It replaces the legacy **OAI**, which doesn't support SSE-KMS.
- Trap: OAC works with the **S3 REST endpoint**. An S3 **static website endpoint** is a *custom* origin — it must be public, and OAC can't be used. If you need website-style redirects on a private bucket, use the REST origin + CloudFront Functions for URL rewriting.

**ALB / EC2 / custom origin — stop people bypassing CloudFront:**

| Technique | How |
| --- | --- |
| **CloudFront VPC origins** | Origin is an ALB/NLB/EC2 in **private subnets**; no public exposure at all (newer feature — the strongest answer where offered) |
| Managed prefix list | ALB security group allows inbound only from `com.amazonaws.global.cloudfront.origin-facing` |
| Secret custom header | CloudFront adds `X-Origin-Verify: <secret>`; ALB listener rule (or WAF on the ALB) rejects requests without it; rotate via Secrets Manager |

**Viewer side:**

- **HTTPS:** viewer protocol policy *Redirect HTTP to HTTPS*. Custom domain certificate from **ACM must be in `us-east-1`** for CloudFront (an ALB's certificate lives in the ALB's own Region). SNI is the default (no extra cost).
- **Signed URLs vs signed cookies** (private content; signed with a key pair in a **trusted key group**):

| | Signed URL | Signed cookies |
| --- | --- | --- |
| Scope | **One file** per URL | **Many files** (e.g. everything under `/premium/*`) |
| URL changes? | Yes | **No** — existing URLs keep working |
| Typical use | Download link for one installer/report | Subscriber area, HLS video (many segment files) |

> Don't confuse with **S3 presigned URLs**: those grant temporary access to an S3 object directly (no CloudFront), with the signer's IAM permissions. "Content served through CloudFront, restricted to paying users" → CloudFront signed URL/cookies.

- **Geo restriction:** allow-list or block-list countries at the distribution (403 for blocked viewers). Finer-grained rules → AWS WAF geo match.
- **Field-level encryption:** encrypt specific POST fields (e.g. card number) at the edge with a public key so only a specific backend component can decrypt.
- **AWS WAF** attaches to the distribution (Web ACL in `us-east-1` scope "CloudFront"); **Shield Standard** is automatic and free. Day 10 covers both.

### High availability: origin groups

An **origin group** = primary + secondary origin. CloudFront fails over to the secondary when the primary returns configured status codes (e.g. 500/502/503/504/403/404) or times out. Works for **GET, HEAD and OPTIONS** only — not writes. Classic pattern: S3 bucket in Region A (primary) + replica bucket in Region B via CRR (secondary).

### Edge compute: CloudFront Functions vs Lambda@Edge

| | **CloudFront Functions** | **Lambda@Edge** |
| --- | --- | --- |
| Runtime | JavaScript (lightweight runtime) | Node.js, Python |
| Runs at | All edge locations | Regional edge caches |
| Events | **Viewer request / viewer response** only | Viewer request/response **and origin request/response** |
| Execution time | Sub-millisecond budget | Seconds (viewer events shorter than origin events — check docs) |
| Network / body access | **No** network calls, no request body | **Yes** — call other services, read body |
| Scale & cost | Millions of req/s, very cheap | Fewer req/s, costlier |
| Deploy | In CloudFront | Authored in **`us-east-1`**, replicated globally |
| Use for | Header manipulation, URL rewrites/redirects, cache-key normalisation, simple JWT/token validation, A/B cookie | Auth calling an IdP, image resizing, origin selection based on a lookup, generating responses from other services |

CloudFront Functions can read a **CloudFront KeyValueStore** for small lookup tables (redirect maps, feature flags) without network calls.

> Rule: "lightweight, high volume, lowest cost, headers/URLs" → **CloudFront Functions**. "Needs network access, body, or origin-facing events" → **Lambda@Edge**.

### Cost levers

- **Price classes:** All edge locations · Price Class 200 · Price Class 100 (cheapest Regions only). Fewer edges = lower cost, slightly higher latency for some users.
- Data transfer from AWS origins **to CloudFront is free**; CloudFront → internet is often cheaper than EC2/S3 → internet, and caching cuts origin compute. "Reduce data transfer costs for global downloads" → CloudFront.
- CloudFront has an always-free monthly allowance, and AWS has introduced flat-rate pricing plans — check current pricing before relying on numbers.

### Logging and monitoring

Standard access logs (S3/CloudWatch Logs/Firehose), **real-time logs** via Kinesis Data Streams, CloudWatch metrics (`4xxErrorRate`, `5xxErrorRate`, `CacheHitRate` with additional metrics enabled).

---

## 5. AWS Global Accelerator

```
 Clients worldwide ──► 2 static ANYCAST IPv4 addresses (same IPs announced from every edge)
                           │ enter at the nearest edge location
                           ▼  AWS global network (avoids congested internet hops)
                  ┌────────┴────────┐
   Endpoint group ap-south-1     Endpoint group eu-west-1       ← traffic dial 0–100% per Region
   ALB / NLB / EC2 / EIP          ALB / NLB / EC2 / EIP          ← weights per endpoint
   health checks ─► unhealthy endpoint/Region removed in seconds (no DNS TTL to wait out)
```

- **Listeners**: TCP and/or UDP ports. Works for **non-HTTP** protocols (gaming, VoIP, MQTT, custom TCP).
- **Static IPs**: two anycast IPv4 addresses (bring-your-own IP supported; dual-stack adds IPv6). Never change, even when you change Regions or endpoints → perfect for **firewall allow-lists** and clients that hard-code IPs.
- **Fast failover**: health-check-based, no client DNS caching involved.
- **Traffic dials** (per Region) and **endpoint weights**: blue/green or gradual Regional migration.
- **Client IP preservation** for ALB and EC2 endpoints.
- **Custom routing accelerators**: deterministically map users to specific EC2 instances/ports (e.g. multiplayer game sessions).
- No caching, no edge compute, no HTTP-level features — put CloudFront in front for those.

### CloudFront vs Global Accelerator vs friends

| Need | CloudFront | Global Accelerator | Route 53 latency | S3 Transfer Acceleration |
| --- | --- | --- | --- | --- |
| Protocols | HTTP/HTTPS, WebSocket | **TCP / UDP** (any app) | Any (it's DNS) | HTTP(S) to S3 |
| Caching | **Yes** | No | No | No |
| Static IPs | No | **Yes — 2 anycast** | No | No |
| Failover speed | Origin groups (per request) | **Seconds** | Bound by DNS TTL | n/a |
| Path | Edge → AWS network | Edge → AWS network | Public internet | Edge → AWS network |
| Best for | Websites, APIs, video, downloads | Games, IoT, VoIP, allow-listed IPs, instant Regional failover | Cheap multi-Region routing | **Long-distance uploads** to one bucket |

---

## 6. Putting it together: a global, resilient container app

```
                   Route 53  www.example.com  ── alias ──► CloudFront (+ WAF, ACM cert in us-east-1)
                                                             │ /static/*  → S3 (OAC)   [origin group: S3 replica in 2nd Region]
                                                             │ /api/*     → origin "api.example.com"
                                                             ▼
                   Route 53  api.example.com  latency routing + health checks
                       ├─► ALB ap-south-1     → ECS Fargate service (3 AZs) → DynamoDB global table
                       └─► ALB eu-west-1      → ECS Fargate service (3 AZs) → DynamoDB global table
                   ECR replication keeps images in both Regions.
```

Swap the `api` latency record for **Global Accelerator** if the question adds "static IPs", "non-HTTP", or "failover must not depend on DNS caching".

---

## 7. Exam traps roundup

1. **Task role ≠ task execution role.** App permissions → task role. Pull image/logs/secrets → execution role.
2. **Fargate requires awsvpc**; ALB target type must be **ip**.
3. **ECR in private subnets** needs `ecr.api` + `ecr.dkr` interface endpoints **and the S3 gateway endpoint** (layers live in S3).
4. **Fargate Spot** can be interrupted — not for "must complete without interruption".
5. **CNAME can't be at the zone apex**; alias can, and alias queries to AWS resources are free.
6. **Alias can't target an EC2 DNS name.**
7. **Geolocation needs a default record**; latency ≠ geolocation.
8. **Health checkers can't reach private resources** → CloudWatch alarm–based health check.
9. **DNS failover isn't instant** (TTL caching). Need seconds → Global Accelerator.
10. **ACM certificate for CloudFront must be in us-east-1.** Lambda@Edge is authored in us-east-1, too.
11. **OAC doesn't work with S3 website endpoints**; OAI doesn't support SSE-KMS.
12. **Origin failover is for GET/HEAD/OPTIONS only.**
13. **One file → signed URL; many files / don't change URLs → signed cookies.** S3 presigned URL bypasses CloudFront.
14. **CloudFront caches; Global Accelerator doesn't.** GA gives static IPs; CloudFront doesn't.
15. **S3 Transfer Acceleration** is for *uploads* to a bucket over long distances — not for serving downloads (CloudFront).

---

## 8. Exam decision table

| If the question says… | Pick |
| --- | --- |
| Run containers, no servers to manage, least operational overhead | **ECS on Fargate** |
| Company already runs Kubernetes / wants open-source K8s tooling, portability | **EKS** (Fargate profiles or Auto Mode for least overhead) |
| Containers needing GPUs or specific instance types, high steady utilisation | **ECS (or EKS) on EC2** with capacity providers |
| Cheapest for fault-tolerant, interruptible container tasks | **Fargate Spot** (or Spot capacity provider) |
| Container app needs to call S3/DynamoDB | **ECS task role** (EKS: Pod Identity / IRSA) |
| Task fails to pull image or read secret at startup | Fix **task execution role** / VPC endpoints |
| Fargate in private subnets, no NAT, pull from ECR | **Interface endpoints ecr.api + ecr.dkr + S3 gateway endpoint** (+ logs) |
| Shared persistent storage for containers across AZs | **Amazon EFS** volume |
| Inject DB password into container securely | Task definition `secrets` from **Secrets Manager / Parameter Store** |
| Scale ECS service on queue depth / requests | **Service auto scaling** (target tracking / step) via Application Auto Scaling |
| Large batch jobs with queues, retries, Spot | **AWS Batch** |
| Point `example.com` (apex) at ALB/CloudFront | **Route 53 alias A record** |
| Send 10% of traffic to new version via DNS | **Weighted** routing |
| Lowest latency across multiple Regions via DNS | **Latency** routing |
| Users in a country must be served by a specific Region (compliance/language) | **Geolocation** (+ default record) |
| Shift a Region's share of geographic traffic with a bias | **Geoproximity** |
| Active-passive DR at DNS level | **Failover** routing + health check |
| Health-check a private resource / custom metric | **CloudWatch alarm–based health check** |
| Return several healthy IPs, no load balancer | **Multivalue answer** |
| Route by client IP range / ISP | **IP-based** routing |
| Deliberate, controlled multi-Region switch-over | **Route 53 ARC** routing controls |
| Global users, static/dynamic HTTP content slow, reduce origin load | **CloudFront** |
| Private S3 bucket readable only via CloudFront | **OAC** + bucket policy (`AWS:SourceArn`) — + KMS key policy if SSE-KMS |
| Users must not bypass CloudFront to hit the ALB | **VPC origins**, or SG with **CloudFront prefix list** + secret header |
| Premium content: one file | **CloudFront signed URL** |
| Premium content: many files, keep URLs | **CloudFront signed cookies** |
| Block countries from content | **CloudFront geo restriction** (or WAF geo match) |
| Protect a sensitive form field end to end | **Field-level encryption** |
| Lightweight header/URL rewrite at high scale, cheapest | **CloudFront Functions** |
| Edge logic needing network calls/body/origin events | **Lambda@Edge** |
| Static content HA across Regions | **Origin group** (S3 + CRR replica) |
| Improve cache hit ratio | Trim cache key; **Origin Shield**; longer TTLs; versioned file names |
| Lower CloudFront cost, users mostly in NA/EU | **Price Class 100** |
| Static IPs for allow-listing + multi-Region | **Global Accelerator** |
| UDP/TCP gaming or IoT, global, low jitter, fast failover | **Global Accelerator** |
| Users worldwide upload large files to one S3 bucket | **S3 Transfer Acceleration** (+ multipart upload) |

---
## 9. Lab (30 min, free tier)

Goal: serve a **private** S3 bucket through CloudFront with OAC, watch caching work, add security headers with a CloudFront Function, and invalidate. Everything here fits CloudFront's and S3's free allowances for a lab's traffic (check current free-tier terms). Route 53 hosted zones and Fargate are **not** free, so they're optional extensions at the end.

### Console steps

1. **S3 → Create bucket** `day09-site-<yourname>` in `ap-south-1`. Leave **Block all public access ON**. Upload an `index.html`:
   ```html
   <h1>Day 9 — served by CloudFront v1</h1>
   ```
2. Open the object's **Object URL** in a browser → `AccessDenied`. Good: the bucket is private.
3. **CloudFront → Create distribution** (if the console offers pricing plans, choose the free / pay-as-you-go option and skip paid add-ons):
   - Origin domain: pick the bucket's **REST** endpoint (`…s3.ap-south-1.amazonaws.com`, *not* the website endpoint).
   - Origin access: **Origin access control settings** → *Create new OAC* (sign requests).
   - Viewer protocol policy: **Redirect HTTP to HTTPS**. Allowed methods: GET, HEAD.
   - Cache policy: **CachingOptimized**. Default root object: `index.html`.
   - WAF: **do not enable** (it costs money; Day 10 covers it). Price class: *North America and Europe* or *…Asia* — note how this trades cost for reach.
4. After creation, CloudFront shows a banner with the **bucket policy** to copy. Paste it into S3 → bucket → Permissions → Bucket policy. Read it: principal `cloudfront.amazonaws.com`, condition `AWS:SourceArn` = this distribution.
5. Wait for *Deployed* (a few minutes). Then:
   ```bash
   D=dxxxxxxxxxxxx.cloudfront.net
   curl -sI https://$D/ | grep -iE 'x-cache|age|via'
   curl -sI https://$D/ | grep -iE 'x-cache|age|via'
   ```
   First call: `X-Cache: Miss from cloudfront`. Second: `Hit from cloudfront` with an `Age` header.
6. **CloudFront → Functions → Create function** `day09-security-headers` (runtime cloudfront-js-2.0):
   ```javascript
   function handler(event) {
     var response = event.response;
     var h = response.headers;
     h['strict-transport-security'] = { value: 'max-age=63072000; includeSubDomains' };
     h['x-content-type-options']    = { value: 'nosniff' };
     h['x-frame-options']           = { value: 'DENY' };
     h['x-lesson']                  = { value: 'day-09' };
     return response;
   }
   ```
   **Test** it with a viewer-response event in the console, **Publish**, then associate it with the default behavior on **Viewer response**. After it deploys, `curl -sI https://$D/` shows the new headers. (A *response headers policy* could do the same with no code — note both options.)
7. Change `index.html` to `v2` and upload. `curl https://$D/` still shows **v1** — cached. Create an **invalidation** for `/index.html`, wait, and fetch again → v2. Note why versioned file names are the better long-term habit.
8. Optional: **Geo restriction** → block your own country, wait for deploy, `curl -sI` → `403`. Remove the restriction.
9. Optional (small cost — check pricing): if you own a domain with a Route 53 hosted zone, request an **ACM certificate in us-east-1**, add `www` as an alternate domain name on the distribution, and create an **alias A record** to the distribution.
10. Optional (small cost — check Fargate pricing): ECS → create a Fargate cluster, run one task of `public.ecr.aws/nginx/nginx:latest` in a public subnet with a public IP and an SG allowing 80 from your IP. Open the task's ENI to see awsvpc in action. **Stop the task** straight after.

### Terraform variant

```hcl
terraform {
  required_providers {
    aws    = { source = "hashicorp/aws" }
    random = { source = "hashicorp/random" }
  }
}

provider "aws" { region = "ap-south-1" }

resource "random_id" "suffix" { byte_length = 4 }

resource "aws_s3_bucket" "site" {
  bucket        = "day09-site-${random_id.suffix.hex}"
  force_destroy = true
}

resource "aws_s3_bucket_public_access_block" "site" {
  bucket                  = aws_s3_bucket.site.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_object" "index" {
  bucket       = aws_s3_bucket.site.id
  key          = "index.html"
  content      = "<h1>Day 9 — served by CloudFront v1</h1>"
  content_type = "text/html"
}

resource "aws_cloudfront_origin_access_control" "oac" {
  name                              = "day09-oac-${random_id.suffix.hex}"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

resource "aws_cloudfront_function" "headers" {
  name    = "day09-security-headers-${random_id.suffix.hex}"
  runtime = "cloudfront-js-2.0"
  publish = true
  code    = <<-EOT
    function handler(event) {
      var response = event.response;
      var h = response.headers;
      h['strict-transport-security'] = { value: 'max-age=63072000; includeSubDomains' };
      h['x-content-type-options']    = { value: 'nosniff' };
      h['x-lesson']                  = { value: 'day-09' };
      return response;
    }
  EOT
}

data "aws_cloudfront_cache_policy" "optimized" { name = "Managed-CachingOptimized" }

resource "aws_cloudfront_distribution" "cdn" {
  enabled             = true
  default_root_object = "index.html"
  price_class         = "PriceClass_100"

  origin {
    origin_id                = "s3-site"
    domain_name              = aws_s3_bucket.site.bucket_regional_domain_name
    origin_access_control_id = aws_cloudfront_origin_access_control.oac.id
  }

  default_cache_behavior {
    target_origin_id       = "s3-site"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    cache_policy_id        = data.aws_cloudfront_cache_policy.optimized.id

    function_association {
      event_type   = "viewer-response"
      function_arn = aws_cloudfront_function.headers.arn
    }
  }

  restrictions {
    geo_restriction { restriction_type = "none" }
  }

  viewer_certificate { cloudfront_default_certificate = true }
}

data "aws_iam_policy_document" "oac" {
  statement {
    sid       = "AllowCloudFrontOAC"
    actions   = ["s3:GetObject"]
    resources = ["${aws_s3_bucket.site.arn}/*"]
    principals {
      type        = "Service"
      identifiers = ["cloudfront.amazonaws.com"]
    }
    condition {
      test     = "StringEquals"
      variable = "AWS:SourceArn"
      values   = [aws_cloudfront_distribution.cdn.arn]
    }
  }
}

resource "aws_s3_bucket_policy" "oac" {
  bucket     = aws_s3_bucket.site.id
  policy     = data.aws_iam_policy_document.oac.json
  depends_on = [aws_s3_bucket_public_access_block.site]
}

output "url" { value = "https://${aws_cloudfront_distribution.cdn.domain_name}" }
```

```bash
terraform init && terraform apply
curl -sI "$(terraform output -raw url)" | grep -iE 'x-cache|x-lesson'
aws cloudfront create-invalidation --distribution-id <id> --paths "/index.html"
```

### Teardown

- **Console:** CloudFront → distribution → **Disable**, wait until it shows disabled/deployed, then **Delete**. Delete the CloudFront Function (disassociate first) and the OAC. Empty and delete the S3 bucket. Delete any ACM certificate, Route 53 records/hosted zone, ECS task/cluster and SGs you created in the optional steps.
- **Terraform:** `terraform destroy` (the distribution takes several minutes to disable and delete — let it finish).
- Check **Billing → Bills** tomorrow for anything unexpected.

---

## 10. Practice questions

Answer all ten before opening the answers.

**Q1.** A company is containerising a web application. The operations team is small and does not want to patch, scale or manage servers or clusters of instances. The containers must run across multiple Availability Zones behind a load balancer. Which solution has the LEAST operational overhead?
A. Amazon ECS on EC2 instances in an Auto Scaling group with a capacity provider
B. Amazon ECS with the Fargate launch type behind an Application Load Balancer
C. Self-managed Kubernetes on EC2 instances
D. Amazon EC2 instances running Docker, registered with a Network Load Balancer

**Q2.** An application running as an Amazon ECS task on Fargate receives `AccessDenied` when it tries to write items to a DynamoDB table. The task starts successfully and its logs appear in CloudWatch Logs. What should the solutions architect do?
A. Add DynamoDB permissions to the task execution role
B. Add DynamoDB permissions to the IAM role specified as the task role in the task definition
C. Store an IAM user's access keys as environment variables in the task definition
D. Attach the DynamoDB permissions to the instance profile of the cluster

**Q3.** ECS tasks on Fargate run in private subnets that have no route to a NAT gateway or internet gateway. Tasks fail with `CannotPullContainerError` when pulling images from Amazon ECR in the same Region. Which combination resolves this with the LEAST cost? (Select the best single answer.)
A. Add a NAT gateway to each private subnet
B. Create interface VPC endpoints for `ecr.api` and `ecr.dkr`, and a gateway VPC endpoint for Amazon S3
C. Create an interface VPC endpoint for `ecr.dkr` only
D. Move the images to Amazon ECR Public

**Q4.** A company runs dozens of microservices on a self-managed Kubernetes cluster on premises, using Helm charts and Kubernetes operators. It wants to move to AWS with minimal changes to its deployment tooling while offloading control plane management. What should it use?
A. Amazon ECS on Fargate
B. AWS App Runner
C. Amazon EKS
D. AWS Elastic Beanstalk with the Docker platform

**Q5.** A company hosts its website behind an Application Load Balancer. It wants users to reach the site at `example.com` (the zone apex) and `www.example.com`, with DNS queries as cost-efficient as possible. Which Route 53 configuration should be used?
A. A CNAME record for `example.com` pointing to the ALB DNS name
B. An A record for `example.com` containing the ALB's current IP addresses
C. Alias A records for `example.com` and `www.example.com` targeting the ALB
D. An MX record for `example.com` pointing to the ALB

**Q6.** A media company streams content licensed only for viewers in specific countries. Users in Germany must always be served from `eu-central-1` and users in Japan from `ap-northeast-1`, regardless of network latency. Users from all other countries should go to `us-east-1`. Which Route 53 routing policy meets these requirements?
A. Latency routing
B. Geolocation routing with records for Germany and Japan, plus a default record pointing to `us-east-1`
C. Weighted routing
D. Multivalue answer routing

**Q7.** A company runs an application in `us-east-1` with a warm standby in `us-west-2`. The application's health can only be determined from a metric inside a private VPC. The company wants Route 53 to fail over automatically when the primary is unhealthy. What should the architect configure?
A. Failover routing with a Route 53 health check that targets the private IP address of the primary
B. Failover routing with a Route 53 health check based on a CloudWatch alarm for the metric
C. Weighted routing with weights 100 and 0 and no health checks
D. Latency routing between both Regions

**Q8.** A company serves files from a private S3 bucket that uses SSE-KMS with a customer managed key. It wants to deliver the files globally through CloudFront and ensure the bucket cannot be accessed directly. What should the architect do?
A. Make the bucket public and enable the S3 static website endpoint as the CloudFront origin
B. Use an Origin Access Identity (OAI) and a bucket policy granting the OAI read access
C. Use Origin Access Control (OAC), a bucket policy allowing the CloudFront service principal with the distribution's ARN as `AWS:SourceArn`, and allow CloudFront to use the KMS key in the key policy
D. Generate S3 presigned URLs for every object and cache them in CloudFront

**Q9.** A company needs to add security headers to every response and redirect legacy URLs to new paths for a website that handles millions of requests per second through CloudFront. The logic needs no network access. Which solution is the MOST cost-effective?
A. Lambda@Edge on origin response
B. CloudFront Functions on viewer request and viewer response
C. An AWS Lambda function behind API Gateway in each Region
D. Custom middleware on the origin EC2 instances

**Q10.** A gaming company runs UDP-based game servers on EC2 behind Network Load Balancers in three Regions. Players' firewalls must allow-list a small fixed set of IP addresses, and if a Region fails, players must be redirected in seconds without waiting for DNS caches to expire. What should the architect use?
A. Amazon CloudFront with the NLBs as origins
B. Route 53 latency routing with health checks and a 60-second TTL
C. AWS Global Accelerator with an endpoint group per Region containing the NLBs
D. Elastic IP addresses on each NLB with Route 53 multivalue answer routing

<details>
<summary><strong>Answers</strong></summary>

1. **B** — Fargate removes servers and cluster capacity management entirely; ECS services spread tasks across AZs behind an ALB. A still means managing EC2/ASGs; C and D are the most overhead.
2. **B** — App code uses the **task role**. The execution role (A) is only for the agent — image pulls, logs, secrets — which already work. C uses long-term keys; D doesn't exist on Fargate and would over-share on EC2.
3. **B** — ECR needs both interface endpoints, and image layers come from S3 via the (free) gateway endpoint. C misses `ecr.api` and S3. A works but NAT gateways cost more per hour and per GB.
4. **C** — EKS runs upstream Kubernetes, so Helm charts and operators keep working; AWS manages the control plane. ECS (A) would mean rewriting deployments.
5. **C** — Alias records work at the apex (CNAME can't, A), track the ALB's changing IPs (B would break), and alias queries to AWS resources are free.
6. **B** — "Regardless of latency" + countries = geolocation. The default record catches everyone else (without it, they'd get no answer). Latency (A) could send Germans elsewhere.
7. **B** — Route 53 health checkers are on the internet and can't reach private IPs (A). A CloudWatch alarm–based health check reflects the internal metric and drives failover routing.
8. **C** — OAC supports SSE-KMS (with the key policy allowing CloudFront) and locks the bucket to one distribution. OAI (B) doesn't support SSE-KMS; website endpoints (A) must be public.
9. **B** — CloudFront Functions are built for lightweight header/URL logic at viewer events, at massive scale and the lowest price. Lambda@Edge (A) works but costs more and is overkill without network needs.
10. **C** — Global Accelerator gives two static anycast IPs, supports UDP, and fails over by health checks in seconds, independent of DNS TTLs. CloudFront (A) doesn't do UDP; B and D depend on DNS caching.

</details>

---

## 11. Flashcards (cover the right column)

| Prompt | Answer |
| --- | --- |
| Least-overhead way to run containers | ECS on Fargate |
| Role for app code in an ECS task | Task role |
| Role for pulling images, writing logs, fetching secrets | Task execution role |
| Network mode required by Fargate | awsvpc (ALB target type ip) |
| ECR endpoints for private subnets with no NAT | ecr.api + ecr.dkr (interface) + S3 (gateway) (+ logs) |
| Fargate Spot interruption warning | 2 minutes (SIGTERM) |
| Shared persistent storage for Fargate tasks | Amazon EFS |
| Pod-level IAM on EKS | EKS Pod Identity or IRSA |
| "Already uses Kubernetes" | EKS |
| Can a CNAME be at the zone apex? | No — use alias |
| Can alias target an EC2 public DNS name? | No |
| Cost of alias queries to AWS resources | Free |
| "Fastest Region for the user" policy | Latency |
| "User's country decides" policy | Geolocation (+ default record) |
| Policy with a bias to grow a Region's area | Geoproximity |
| Up to 8 healthy records returned | Multivalue answer |
| Health-check a private resource | CloudWatch alarm–based health check |
| Region for CloudFront's ACM certificate | us-east-1 |
| Lock S3 to CloudFront (modern) | Origin Access Control (OAC) |
| Does OAC work with S3 website endpoints? | No — REST endpoint only |
| Private content, many files, keep URLs | Signed cookies |
| CloudFront origin failover methods | GET, HEAD, OPTIONS |
| Edge logic with network access / origin events | Lambda@Edge |
| Header/URL rewrite, cheapest, highest scale | CloudFront Functions |
| Extra central cache layer to protect origin | Origin Shield |
| Cheapest CloudFront price class | Price Class 100 |
| Static anycast IPs + TCP/UDP + fast failover | Global Accelerator (2 IPs) |
| Shift % of traffic per Region in GA | Traffic dial |
| Faster long-distance uploads to S3 | S3 Transfer Acceleration |

## 12. Before you close today

- [ ] Read sections 1–8 (focus: the two ECS roles, alias vs CNAME, routing policies table, CloudFront vs Global Accelerator)
- [ ] Lab done and torn down (distribution disabled + deleted, function, OAC, bucket; any optional Route 53/ACM/ECS resources)
- [ ] 10 questions above + 20 bank questions on ECS/EKS/Route 53/CloudFront/Global Accelerator
- [ ] Misses logged in `miss-log.md` — the log is still empty, so add any misses from Days 7–9 now; Days 12–13 are built from it

**Tomorrow (Day 10):** Security services — KMS (key types, envelope encryption, key policies), Secrets Manager vs Parameter Store, WAF and Shield, GuardDuty, Inspector, Macie, Security Hub, Detective, and Amazon Cognito.
