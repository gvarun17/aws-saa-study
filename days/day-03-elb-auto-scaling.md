# Day 3 — Elastic Load Balancing (ALB / NLB / GWLB) & EC2 Auto Scaling

**Domain weight:** mostly *Design Resilient Architectures* (26%) and *Design High-Performing Architectures* (24%), with a slice of *Cost-Optimized* (20%) — right-sizing capacity is a cost decision.
**Time plan:** Learn 90 min · Lab 30 min · Drill 45 min · Review 30 min.

By the end of today you should be able to:

- Pick ALB, NLB or GWLB from one or two phrases in a question (layer, protocol, static IP, appliances).
- Explain how health checks, deregistration delay and cross-zone load balancing change what users see during failures and deployments.
- Design a multi-AZ Auto Scaling group behind a load balancer, and explain why `min ≥ 2` across ≥ 2 AZs is the HA baseline.
- Choose the right scaling policy — target tracking, step, simple, scheduled, predictive — and size cooldowns/warm-up correctly.
- Use lifecycle hooks, warm pools, instance refresh and mixed-instances policies to solve boot-time, deployment and cost problems.

---

## 1. Why load balancers exist (and how ELB actually works)

A load balancer gives clients **one stable DNS name** while spreading traffic across a changing fleet of targets, and it **stops sending traffic to unhealthy targets**. Those two jobs — decoupling and health-based routing — are what make "stateless app tier in an Auto Scaling group" the default resilient pattern on the exam.

```
                       Client
                         │  DNS: my-alb-123.ap-south-1.elb.amazonaws.com
                         ▼
          ┌─────────── Load balancer (Regional, managed) ───────────┐
          │  LB node in AZ a        LB node in AZ b        AZ c ... │  ← one node (ENI) per enabled subnet
          └───────┬────────────────────────┬────────────────────────┘
                  │ Listener (port + protocol) → Rules → Target group
                  ▼                        ▼
           ┌─ Target group (health checks, stickiness, algorithm) ─┐
           │  i-aaa (AZ a)   i-bbb (AZ a)   i-ccc (AZ b)            │
           └────────────────────────────────────────────────────────┘
                  ▲
                  └── registered/deregistered automatically by the Auto Scaling group
```

The moving parts:

| Part | What it is | Exam relevance |
| --- | --- | --- |
| **Load balancer nodes** | AWS-managed ENIs in each subnet you enable (pick **one subnet per AZ**, ≥ 2 AZs for ALB) | Enable every AZ your targets live in, or targets in a missing AZ get no traffic |
| **Listener** | Protocol + port the LB accepts (e.g. HTTPS:443) | TLS certificates attach here (from ACM) |
| **Rules** (ALB) | Conditions → actions (forward, redirect, fixed response, authenticate) | Content-based routing questions |
| **Target group** | A set of targets + health check + routing settings | One ASG can attach to several TGs; one TG can serve several rules |
| **Targets** | Instances, IP addresses, Lambda functions (ALB), an ALB (NLB) | IP targets = on-prem servers over VPN/DX, or containers |

**Scheme:** *internet-facing* (public IPs on nodes, in public subnets) or *internal* (private IPs only — for tier-to-tier traffic). Your targets can — and should — sit in **private subnets** even behind an internet-facing LB.

**Elastic Load Balancing scales itself.** You don't size an ALB. It's Regional and multi-AZ by construction. (For a sudden, enormous spike, AWS used to recommend "pre-warming"; today ALB/NLB scale automatically, and you can request **LCU reservation** for known events — check current docs.)

> Trap: an ALB's IP addresses **change** over time. Clients must use the DNS name. If a question needs fixed IPs for allow-listing, the answer is **NLB** (static IP / Elastic IP per AZ) or **Global Accelerator** (two static anycast IPs) — not ALB on its own.

---

## 2. The four load balancer types

| | **ALB** (Application) | **NLB** (Network) | **GWLB** (Gateway) | CLB (Classic) |
| --- | --- | --- | --- | --- |
| OSI layer | 7 | 4 | 3 (network gateway) | 4/7 (legacy) |
| Protocols | HTTP, HTTPS, gRPC, WebSockets | TCP, UDP, TLS, TCP_UDP | IP (all traffic), encapsulated in **GENEVE on port 6081** | HTTP/S, TCP, SSL |
| Routing on | Path, host, headers, HTTP method, query string, source IP | Port / protocol only | Flow hash (5-tuple) to appliances | Basic |
| Targets | Instance, IP, **Lambda** | Instance, IP, **ALB** | Instance, IP (appliances) | Instance |
| Static IP | No (use NLB in front, or Global Accelerator) | **Yes** — one per AZ, optionally your **Elastic IPs** | N/A | No |
| Client source IP seen by target | Via **`X-Forwarded-For`** header | **Preserved** (instance targets; configurable for IP targets) | Preserved (traffic is transparent) | X-Forwarded-For |
| Performance | High; adds a little latency | **Millions of req/s, ultra-low latency**, handles sudden spikes | Line-rate for appliance fleets | — |
| Security groups | Yes | Yes (must be associated **at creation**; otherwise none) | No | Yes |
| TLS termination | Yes (SNI, multiple certs, mTLS) | Yes (TLS listener, SNI) | No | Yes (one cert) |
| Cross-zone default | **On** (no inter-AZ charge) | **Off** (inter-AZ data charges if enabled) | **Off** | — |
| Typical use | Web apps, microservices, containers, path-based APIs | Gaming, IoT, VoIP, financial, non-HTTP TCP/UDP, PrivateLink | Inline firewalls/IDS/IPS from vendors | Don't pick for new designs |

### ALB in depth — layer 7 features you're tested on

- **Content-based routing.** Rules evaluate in priority order; the default rule catches everything else.
  - `/api/*` → API target group, `/images/*` → image target group (**path-based**)
  - `orders.example.com` → orders TG, `users.example.com` → users TG (**host-based**) — one ALB fronting many microservices
  - Header (`User-Agent` contains `Mobile`), HTTP method, query string (`?version=beta`), source IP CIDR
- **Actions:** `forward` (to one or **weighted** multiple TGs — blue/green and canary at the LB), `redirect` (HTTP → HTTPS, the classic one), `fixed-response` (maintenance page, 404 for unknown paths), `authenticate-oidc` / `authenticate-cognito` (offload login to an IdP before traffic reaches your app).
- **Lambda targets:** the ALB invokes the function and translates the HTTP request into an event. Serverless HTTP endpoint without API Gateway (but without API Gateway features like throttling keys, usage plans, request validation).
- **TLS:** certificates from **ACM** (free public certs, auto-renewing). **SNI** lets one HTTPS listener serve many domains with different certs. **Mutual TLS** (client certificates) is supported on ALB via trust stores.
- **Containers:** IP targets + dynamic port mapping let many ECS tasks on one host register on different ports (Day 9).
- **Algorithms:** round robin (default), **least outstanding requests** (good when request cost varies), weighted random (with optional *automatic target weights* that back off from anomalous targets).

```json
// ALB listener rule: path-based forward with a 90/10 canary split
{
  "Priority": 10,
  "Conditions": [{ "Field": "path-pattern", "Values": ["/api/*"] }],
  "Actions": [{
    "Type": "forward",
    "ForwardConfig": {
      "TargetGroups": [
        { "TargetGroupArn": "arn:aws:elasticloadbalancing:...:targetgroup/api-v1/...", "Weight": 90 },
        { "TargetGroupArn": "arn:aws:elasticloadbalancing:...:targetgroup/api-v2/...", "Weight": 10 }
      ]
    }
  }]
}
```

### NLB in depth — layer 4 and the "static IP" answer

- Operates on connections, not requests. It doesn't look inside HTTP — no path routing, no cookies, no redirects.
- **One static IP per AZ**, or bring your own **Elastic IPs** → partners can allow-list fixed addresses.
- **Source IP preservation** — the target sees the real client IP in the packet, so target security groups / NACLs must allow the *client* CIDRs, not just the NLB. (Common trap: "targets reject traffic after migrating to NLB.")
- **TLS listener** offloads TLS at layer 4 while keeping the NLB's performance.
- **ALB as a target of NLB:** the pattern for "we need static IPs *and* path-based routing" — NLB (static IPs/EIPs) → ALB-type target group → ALB rules.
- **PrivateLink:** to expose your service privately to other VPCs/accounts as a VPC **endpoint service**, you put it behind an **NLB** (Day 4).
- UDP support → DNS, syslog, gaming, VoIP, IoT.

### GWLB in depth — "insert a third-party appliance in the traffic path"

```
 Internet ─► IGW ─► (ingress route table) ─► GWLB endpoint ─► GWLB ─► firewall appliances (ASG)
                                                                  ◄──── inspected traffic returns
                                         ◄── GWLB endpoint ◄──────────
                     app subnets ◄────────┘
```

- Combines a **transparent network gateway** (single entry/exit for traffic) with a **load balancer** that distributes flows across a fleet of virtual appliances (Palo Alto, Fortinet, Check Point, etc.) and health-checks them.
- Traffic is steered there by **route tables** pointing at a **Gateway Load Balancer endpoint** (a PrivateLink-style endpoint) — often in a central inspection VPC.
- Uses **GENEVE (UDP 6081)** so appliances see the original packet unchanged.
- Exam cue: "inspect all traffic with third-party firewall/IDS/IPS appliances, scale them, keep them highly available" → **GWLB**. If the question says *managed* firewall with no third-party appliance → **AWS Network Firewall** (Day 4/10).

### CLB

Previous generation. On the exam it appears as a wrong answer, or in "migrate from CLB to get path-based routing/Lambda targets" → ALB.

---

## 3. Health checks, draining and stickiness

### Health checks

Configured **per target group**. The LB probes each target; only healthy targets receive traffic.

| Setting | Meaning | Typical tuning |
| --- | --- | --- |
| Protocol / port / path | e.g. HTTP `/health` on traffic port | Point at a lightweight endpoint that checks the app (and maybe its critical dependencies) |
| Healthy / unhealthy threshold | Consecutive successes/failures to flip state | Lower unhealthy threshold → faster failover, more flapping risk |
| Interval / timeout | How often / how long to wait | Shorter interval → faster detection |
| Success codes (ALB) | e.g. `200` or `200-299` | Make sure redirects (`301`) aren't treated as failures unless intended |

**Fail open:** if *every* target in a TG is unhealthy, the ALB/NLB routes to all of them anyway (better than serving nothing). Don't be surprised by this in a "why is traffic still reaching broken instances" question.

### Health checks in Auto Scaling — the most-tested interaction

An ASG uses **EC2 status checks** by default. That only catches a broken VM — not a hung web server. Turn on **ELB health checks** on the ASG so instances the load balancer marks unhealthy get **terminated and replaced**.

```
App process crashes on i-bbb
   ├─ ALB health check fails ─► ALB stops routing to i-bbb   (always)
   └─ ASG health check type?
         ├─ EC2 only   ─► i-bbb stays "InService" forever, wasting capacity  ✗
         └─ ELB        ─► ASG marks i-bbb unhealthy ─► terminates ─► launches replacement  ✓
```

Pair it with a **health check grace period** long enough for boot + app start, or new instances get killed before they're ready (a classic "instances keep getting terminated right after launch" question).

### Deregistration delay (connection draining)

When a target is deregistered (scale-in, deployment), the LB stops sending **new** requests but lets **in-flight** requests finish for up to the deregistration delay (**default 300 s**, configurable 0–3600 s). Short-lived HTTP requests → lower it to speed up deployments; long uploads/websockets → raise it.

### Slow start (ALB)

New targets get a linearly increasing share of traffic over a configured window — for apps with cold caches or JIT warm-up.

### Sticky sessions (session affinity)

| LB | Mechanism |
| --- | --- |
| ALB | **Duration-based** cookie generated by the LB (`AWSALB`), or **application-based** using your app's cookie |
| NLB | Source-IP affinity |
| GWLB | Flow stickiness (5-tuple / 3-tuple / 2-tuple) |

Stickiness causes **uneven load** and loses sessions when a target dies. The exam's preferred fix for "users lose their session/cart when an instance is terminated" is to make the app **stateless** and put session state in **ElastiCache** or **DynamoDB** — stickiness is the "minimal changes" answer, not the best-architecture one.

### Cross-zone load balancing

```
Cross-zone OFF                         Cross-zone ON
AZ a node: 50% → 2 targets (25% each)  AZ a node: 50% ─┐
AZ b node: 50% → 8 targets (6.25% ea)  AZ b node: 50% ─┴► spread over all 10 (10% each)
```

Without it, each LB node only sends to targets in its own AZ, so **uneven target counts per AZ** give uneven load per target. ALB: on by default, no inter-AZ data charge. NLB/GWLB: off by default; enabling it incurs inter-AZ data transfer charges (check current pricing). Fix for "instances in one AZ are overloaded" → enable cross-zone and/or balance the ASG across AZs.

### Other details worth recognising

- **Idle timeout** (ALB default 60 s) — raise it for long-polling or slow backends that otherwise see `504`s.
- **HTTP 5xx meanings:** `502` bad gateway (target sent a bad/closed response), `503` no healthy/registered targets, `504` target didn't respond within the idle timeout.
- **Access logs** → S3 (per-request detail); **CloudWatch metrics** like `RequestCountPerTarget`, `TargetResponseTime`, `HTTPCode_Target_5XX_Count`, `HealthyHostCount`.
- **WAF** attaches to **ALB** (and CloudFront, API Gateway, AppSync, Cognito) — not to NLB. "Block SQL injection at the load balancer" → ALB + AWS WAF.
- **Shield Standard** protects all ELBs automatically; Shield Advanced adds DDoS response and cost protection (Day 10).

---

## 4. EC2 Auto Scaling — how an ASG thinks

An Auto Scaling group keeps the **number of healthy instances equal to desired capacity**, within `min ≤ desired ≤ max`, spread across the AZs/subnets you give it.

```
               ┌──────────── Auto Scaling group ────────────┐
Launch         │ min 2   desired 4   max 10                  │
template ────► │ subnets: private-a, private-b, private-c    │──► registers into ALB target group(s)
(AMI, type,    │ health check: ELB, grace 300 s              │
 SGs, role,    │ policies: target tracking CPU 50%           │◄── CloudWatch alarms / metrics
 user data)    └─────────────────────────────────────────────┘
```

Three jobs, all on the exam:

1. **Self-healing** — replace instances that fail health checks (resilience; works even with a fixed size `min = max = desired`).
2. **AZ balance** — spread across AZs; after an AZ outage, launch in the remaining AZs; **rebalance** when the AZ returns.
3. **Elasticity** — change desired capacity with scaling policies (performance + cost).

> "Make a single EC2 instance self-healing without a load balancer" → ASG with `min = max = desired = 1`. It's free (you pay for the instances), but it won't keep the same IP or instance store data.

### Launch templates

Define *what* to launch: AMI, instance type(s), key pair, security groups, IAM instance profile, user data, EBS, IMDSv2, tags, network settings. They're **versioned** (`$Latest`, `$Default`, or a specific number).

**Launch configurations** are the deprecated predecessor (immutable, no versions, no mixed instances, no newer instance types). On the exam, "use a launch template" is the right modernisation answer.

### Capacity, AZs and the HA baseline

- For HA: **≥ 2 AZs** and **min ≥ 2** (so one instance survives an AZ loss).
- For "must keep N instances serving even if an AZ fails": size so that the **remaining AZs** still hold N. Example: need 4 always → 3 AZs × 2 = 6 (lose one AZ, 4 remain), or 2 AZs × 4 = 8.
- An ASG is **Regional within one VPC**; it spans AZs, not Regions.

### Scaling policies

| Policy | How it works | Use when | Exam cue |
| --- | --- | --- | --- |
| **Target tracking** | You pick a metric and a target (e.g. CPU 50%, `ALBRequestCountPerTarget` 1000); AWS creates and manages the alarms, scaling out and in proportionally | Default for most workloads | "Keep average CPU around X%", "simplest", "least operational overhead" |
| **Step scaling** | CloudWatch alarm with step adjustments by breach size (+1 at 60–70%, +3 above 80%); keeps evaluating during warm-up | Need different responses to small vs large breaches | "Scale more aggressively the higher the metric goes" |
| **Simple scaling** | One alarm → one adjustment → wait for **cooldown** (default 300 s) before any further action | Legacy; rarely the best answer | Appears as the inferior option |
| **Scheduled** | Set min/max/desired at a time or cron | Known patterns: business hours, a Monday 9 am spike, a sale | "Every weekday at 8 am traffic jumps" |
| **Predictive** | ML forecast from history (needs at least a day of data, better with ~2 weeks) → launches capacity **ahead** of the forecast load; can run in *forecast only* mode first | Recurring daily/weekly cycles with slow-booting instances | "Cyclical traffic, scale before the load arrives" |

Notes that decide questions:

- You can combine policies: **predictive or scheduled for the baseline + target tracking for surprises**. When several policies fire, the ASG picks the one giving the **largest** capacity.
- **Instance warm-up** (target tracking/step) stops a just-launched instance's metrics from triggering more scaling until it's ready. **Cooldown** is the simple-scaling equivalent.
- **Scale on the right metric.** CPU isn't always it:
  - Web tier → `ALBRequestCountPerTarget` or CPU.
  - Queue workers → **SQS backlog per instance** (`ApproximateNumberOfMessagesVisible ÷ running instances`) published as a custom metric, then target-track it. "Workers fall behind the queue" → this.
  - Memory isn't a default EC2 metric — publish it with the **CloudWatch agent**.
- Target tracking scales in more gently than out (so as not to overreact); you can disable scale-in on a policy if another mechanism handles it.

```bash
# Target tracking on requests per target (CLI)
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name web-asg \
  --policy-name rps-1000 \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ALBRequestCountPerTarget",
      "ResourceLabel": "app/web-alb/abc123/targetgroup/web-tg/def456"
    },
    "TargetValue": 1000
  }'
```

### Termination: who goes first on scale-in?

Default termination policy, roughly:

1. Pick the **AZ with the most instances** (keep AZs balanced).
2. Within it, prefer instances that align with the allocation strategy (On-Demand vs Spot mix).
3. Then instances using the **oldest launch template/configuration**.
4. Then the instance **closest to the next billing hour** (tie-break), then random.

Custom policies exist (`OldestInstance`, `NewestInstance`, `OldestLaunchTemplate`, `AllocationStrategy`, or a Lambda for custom logic). **Instance scale-in protection** stops specific instances (e.g. a long-running job) from being chosen.

### Lifecycle hooks

Pause an instance during launch or termination so you can run custom actions.

```
Scale-out:  Pending ──► Pending:Wait ──(your action)──► Pending:Proceed ──► InService
Scale-in:   Terminating ──► Terminating:Wait ──(your action)──► Terminating:Proceed ──► Terminated
```

- Hook notification → **EventBridge** / SNS / SQS → Lambda or SSM Automation does the work → calls `complete-lifecycle-action` (or the timeout expires; default **1 hour**, adjustable, with `CONTINUE` or `ABANDON` as the default result).
- Launch-side uses: install/configure software, pull config, register with an external system, warm caches *before* the instance receives traffic.
- Terminate-side uses: **copy logs to S3**, drain work, deregister from a license server, take a final snapshot. Exam cue: "logs are lost when instances scale in" → **termination lifecycle hook**.

### Warm pools

A pool of **pre-initialised** instances kept **Stopped** (pay only for EBS), **Hibernated** (RAM preserved on EBS) or **Running**, ready to join the ASG quickly. Answer for "instances take 10+ minutes to boot/initialise and scale-out is too slow" when you don't want to over-provision running capacity. (Alternatives the exam also accepts in the right context: bake software into a **golden AMI** to cut boot time; scheduled/predictive scaling to start earlier.)

### Instance refresh — rolling deployments for an ASG

Change the launch template (new AMI), then start an **instance refresh**: the ASG replaces instances in batches, respecting **minimum healthy percentage** (and optionally a max healthy percentage to launch before terminating), with **checkpoints**, **skip matching** (don't replace instances already on the target config), and **automatic rollback** on failure or CloudWatch alarm. "Roll out a new AMI to the fleet with no downtime and minimal effort" → instance refresh.

### Mixed instances policy (cost optimisation)

One ASG, many instance types, a blend of purchase options:

```
On-Demand base capacity: 2        ← always On-Demand (covered by Savings Plans/RIs)
Above base: 20% On-Demand / 80% Spot
Instance types: m7i.large, m6i.large, m7g.large*, c7i.large, ...   (*Graviton needs an arm64 AMI)
Spot allocation strategy: price-capacity-optimized
Capacity Rebalancing: on        ← proactively replace Spot instances at elevated interruption risk
```

- More instance types/AZs = deeper Spot pools = fewer interruptions.
- **Attribute-based instance type selection** (vCPU/memory ranges) lets AWS pick matching types — including new ones — automatically.
- Ties back to Day 2: Spot for fault-tolerant stateless tiers; On-Demand base for steady state.

### Other ASG features to recognise

| Feature | What it's for |
| --- | --- |
| **Standby** | Take an instance out of service for troubleshooting without terminating it (still counted, no traffic) |
| **Suspend processes** (`Launch`, `Terminate`, `HealthCheck`, `ReplaceUnhealthy`, `AZRebalance`, `AddToLoadBalancer`…) | Temporarily stop specific ASG behaviours, e.g. during maintenance |
| **Instance maintenance policy** | Control whether replacements launch before or after terminating (availability vs cost during refresh/rebalancing) |
| **Default instance warm-up** | One warm-up setting for all policies and refreshes |
| **SNS / EventBridge notifications** | Alert on launch, terminate, failures |
| **AWS Auto Scaling scaling plans / Application Auto Scaling** | Scale *other* resources: ECS services, DynamoDB capacity, Aurora replicas, Spot Fleet, Lambda provisioned concurrency |

---

## 5. Putting it together — the reference web architecture

```
                       Route 53 (alias record → ALB)
                                  │
                         [optional: CloudFront + WAF]
                                  │
        ┌──────────────── VPC ────┼─────────────────────────────┐
        │  Public subnets  ┌──────▼───────┐  (AZ a, AZ b)        │
        │                  │     ALB      │  SG: 443 from world  │
        │                  └──┬────────┬──┘                      │
        │  Private subnets    │        │                         │
        │   ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐          │
        │   │ EC2 a1 │ │ EC2 a2 │ │ EC2 b1 │ │ EC2 b2 │  ← ASG   │
        │   └────────┘ └────────┘ └────────┘ └────────┘  SG: 80 from ALB SG only
        │         session state → ElastiCache / DynamoDB         │
        │         data → RDS Multi-AZ (Day 6)                     │
        └────────────────────────────────────────────────────────┘
```

Security group chaining is the detail examiners love: the **instance SG allows the app port only from the ALB's security group ID**, not from `0.0.0.0/0`. Instances need no public IPs.

Route 53 points at the ALB with an **alias** record (works at the zone apex, free queries to AWS resources) — Day 9.

### When ELB isn't the front door

| Need | Front door |
| --- | --- |
| Global users, cache static content, reduce origin load | **CloudFront** in front of the ALB |
| Two static anycast IPs, fast failover across Regions, non-HTTP global traffic | **Global Accelerator** → ALB/NLB endpoints in several Regions |
| Serverless REST API with throttling, API keys, usage plans, request validation | **API Gateway** (Day 8) |
| Multi-Region active-passive at DNS level | **Route 53 failover** routing between Regional ALBs |

ELB is **Regional**. "Load balance across Regions" is never "one ALB" — it's Route 53 or Global Accelerator.

---

## 6. Exam traps roundup

- ALB = HTTP smarts, **no static IP**. NLB = static IP, TCP/UDP, extreme performance, **source IP preserved**.
- "Path-based / host-based routing", "redirect HTTP to HTTPS", "authenticate users at the LB", "Lambda behind a load balancer" → **ALB**.
- "Third-party virtual appliances", "GENEVE", "bump-in-the-wire inspection" → **GWLB**.
- "Expose my service to other VPCs via PrivateLink" → **NLB** (endpoint service).
- WAF goes on **ALB**, not NLB.
- ASG default health check is **EC2 only** — enable **ELB** health checks to replace app-level failures.
- "Instances terminated immediately after launch" → health check **grace period** too short (or failing health check path).
- "Users logged out when an instance is removed" → externalise sessions (ElastiCache/DynamoDB); stickiness is the stop-gap.
- "Scale on queue depth" → custom **backlog-per-instance** metric + target tracking.
- "Known spike every Monday 9 am" → **scheduled**; "recurring pattern, slow boot" → **predictive** and/or **warm pool**.
- "Capture logs before instances terminate" → **termination lifecycle hook**.
- Launch configurations are deprecated — **launch templates**.
- An ASG and an ELB are both **Regional**; neither spans Regions.

---

## 7. Exam decision table

| If the question says… | Pick |
| --- | --- |
| HTTP/HTTPS app, route by URL path or hostname | **ALB** with listener rules |
| Many microservices behind one entry point | ALB with host/path rules → one target group each |
| TCP/UDP, millions of requests per second, ultra-low latency | **NLB** |
| Partners must allow-list fixed IPs | **NLB** with Elastic IPs (or Global Accelerator) |
| Static IPs **and** path-based routing | **NLB → ALB** (ALB-type target group), or Global Accelerator → ALB |
| Inspect traffic with third-party firewall appliances | **GWLB** + GWLB endpoints |
| Share a service privately with other VPCs/accounts | **NLB** + PrivateLink endpoint service |
| Backend must see the client's real IP over HTTP | Read **`X-Forwarded-For`** (ALB) |
| Backend must see the client IP at the packet level | **NLB** (source IP preservation) |
| Redirect HTTP → HTTPS | ALB listener **redirect** action |
| Many TLS domains on one LB | ALB/NLB with **SNI** + ACM certificates |
| Users must log in before reaching the app, minimal code | ALB **authenticate-oidc / Cognito** action |
| Block SQL injection / XSS at the load balancer | **AWS WAF** on the ALB |
| Canary / blue-green at the load balancer | ALB **weighted target groups** |
| Uneven load because AZs have different instance counts | Enable **cross-zone** load balancing |
| In-flight requests fail during scale-in/deploys | Tune **deregistration delay** |
| Unhealthy app instances aren't replaced | ASG health check type **ELB** |
| Keep CPU at ~50% with least effort | **Target tracking** policy |
| Predictable spike at a known time | **Scheduled** scaling |
| Recurring daily/weekly pattern, scale before it arrives | **Predictive** scaling |
| Instances take too long to initialise when scaling out | **Warm pool** (and/or golden AMI) |
| Run a script / copy logs before an instance is terminated | **Lifecycle hook** (Terminating:Wait) |
| Roll a new AMI across the fleet with no downtime | **Instance refresh** |
| Cut cost on a stateless, fault-tolerant fleet | **Mixed instances policy** with Spot + On-Demand base |
| Worker fleet should scale with SQS queue depth | Target tracking on **backlog per instance** |
| Self-healing single instance | ASG with min = max = desired = 1 |
| Load balance across Regions | **Route 53** routing or **Global Accelerator** — not ELB |

---

## 8. Lab (30 min, free tier)

Goal: build ALB + ASG across two AZs, watch self-healing and scaling, and use a path-based rule.

**Free tier note:** ALB hours and LCUs have historically been included in the free tier for eligible accounts, and AWS changed free-tier terms for newer accounts — **check your Billing console / current docs**. Use the *Free tier eligible* instance type, keep the lab short, and tear down at the end. This lab uses the **default VPC** so instances get public IPs for package installs (in production they'd be in private subnets behind NAT — Day 4).

### Console steps

1. **Security groups** (EC2 → Security Groups):
   - `lab-alb-sg`: inbound HTTP 80 from `0.0.0.0/0`.
   - `lab-web-sg`: inbound HTTP 80 **from source `lab-alb-sg`** (the SG ID, not a CIDR). No SSH.
2. **Launch template** `lab-web-lt`: Amazon Linux 2023, free-tier type, `lab-web-sg`, the Day 1 instance profile (for Session Manager), **IMDSv2 required**, user data:
   ```bash
   #!/bin/bash
   dnf install -y httpd
   TOKEN=$(curl -sX PUT http://169.254.169.254/latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
   ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
   AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone)
   echo "<h1>$ID in $AZ</h1>" > /var/www/html/index.html
   echo ok > /var/www/html/health
   systemctl enable --now httpd
   ```
3. **Target group** `lab-web-tg`: type *Instances*, HTTP 80, health check path `/health`, healthy threshold 2, interval 10 s. Under attributes, set **deregistration delay to 30 s**.
4. **ALB** `lab-alb`: internet-facing, IPv4, default VPC, select **two AZs**, `lab-alb-sg`, listener HTTP:80 → forward to `lab-web-tg`.
5. **Auto Scaling group** `lab-web-asg`: launch template `lab-web-lt`, the same two AZs' subnets, attach to `lab-web-tg`, **turn on Elastic Load Balancing health checks**, grace period 120 s, desired 2 / min 2 / max 4, target tracking **Average CPU 50%**.
6. Open the ALB DNS name and refresh a few times — the instance ID and AZ alternate. That's round robin with cross-zone on.
7. **Self-healing:** in Session Manager on one instance run `sudo systemctl stop httpd`. Watch the target go *unhealthy* in the TG within ~30 s, and the ASG **Activity** tab terminate and replace it. (Try the same with the ASG health check set to EC2 only, and notice it never gets replaced.)
8. **Path rule:** add a listener rule: path `/maintenance*` → **fixed response** 503 "Back soon". Visit `/maintenance`. Add another rule: path `/old*` → **redirect** to `/`.
9. **Scale out:** on one instance run `sudo dnf install -y stress-ng && stress-ng --cpu 0 --timeout 600s`, or simply edit the ASG's desired capacity to 3 and watch the new instance register. Look at the CloudWatch alarms target tracking created for you (don't edit them — AWS manages them).
10. **Tear down (in this order):** delete the ASG (terminates its instances) → delete the ALB → delete the target group → delete the launch template → delete both security groups. Check EC2 → Instances for leftovers.

### Terraform variant

```hcl
provider "aws" { region = "ap-south-1" }

data "aws_vpc" "default" { default = true }

data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
  filter {
    name   = "availability-zone"
    values = ["ap-south-1a", "ap-south-1b"]
  }
}

data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

resource "aws_security_group" "alb" {
  name   = "lab-alb-sg"
  vpc_id = data.aws_vpc.default.id
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "web" {
  name   = "lab-web-sg"
  vpc_id = data.aws_vpc.default.id
  ingress {
    from_port       = 80
    to_port         = 80
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]   # SG chaining: only the ALB can reach the app
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_lb" "web" {
  name               = "lab-alb"
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = data.aws_subnets.default.ids
}

resource "aws_lb_target_group" "web" {
  name                 = "lab-web-tg"
  port                 = 80
  protocol             = "HTTP"
  vpc_id               = data.aws_vpc.default.id
  deregistration_delay = 30
  health_check {
    path                = "/health"
    interval            = 10
    healthy_threshold   = 2
    unhealthy_threshold = 2
  }
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.web.arn
  port              = 80
  protocol          = "HTTP"
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.web.arn
  }
}

resource "aws_lb_listener_rule" "maintenance" {
  listener_arn = aws_lb_listener.http.arn
  priority     = 10
  condition {
    path_pattern { values = ["/maintenance*"] }
  }
  action {
    type = "fixed-response"
    fixed_response {
      content_type = "text/plain"
      message_body = "Back soon"
      status_code  = "503"
    }
  }
}

resource "aws_launch_template" "web" {
  name_prefix            = "lab-web-lt-"
  image_id               = data.aws_ami.al2023.id
  instance_type          = "t3.micro"
  vpc_security_group_ids = [aws_security_group.web.id]
  iam_instance_profile { name = "day1-ssm-profile" }   # from Day 1, or create one
  metadata_options { http_tokens = "required" }
  user_data = base64encode(file("${path.module}/user-data.sh"))   # the script from step 2
}

resource "aws_autoscaling_group" "web" {
  name                      = "lab-web-asg"
  min_size                  = 2
  desired_capacity          = 2
  max_size                  = 4
  vpc_zone_identifier       = data.aws_subnets.default.ids
  target_group_arns         = [aws_lb_target_group.web.arn]
  health_check_type         = "ELB"     # replace instances the ALB marks unhealthy
  health_check_grace_period = 120

  launch_template {
    id      = aws_launch_template.web.id
    version = "$Latest"
  }

  instance_refresh {
    strategy = "Rolling"
    preferences { min_healthy_percentage = 50 }
  }
}

resource "aws_autoscaling_policy" "cpu50" {
  name                   = "cpu-50"
  autoscaling_group_name = aws_autoscaling_group.web.name
  policy_type            = "TargetTrackingScaling"
  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value = 50
  }
}

output "url" { value = "http://${aws_lb.web.dns_name}" }
```

Experiment: change the user data (e.g. add "v2" to the page) and `terraform apply` — a new launch template version plus the `instance_refresh` block triggers a rolling replacement. Refresh the URL and watch v1 and v2 mix, then settle on v2.

**Teardown:** `terraform destroy`. Confirm no instances, load balancers or target groups remain.

---

## 9. Practice questions

Answer all ten before opening the answers.

**Q1.** A company runs a web application on EC2 instances in an Auto Scaling group behind an Application Load Balancer. Sometimes the application process hangs; the instances pass EC2 status checks but return errors, and they are never replaced. What should a solutions architect do?
A. Increase the ALB idle timeout
B. Configure the Auto Scaling group to use Elastic Load Balancing health checks
C. Enable cross-zone load balancing
D. Add a step scaling policy on CPU utilization

**Q2.** A financial services company exposes a TCP-based trading API. Its partners require a small set of fixed IP addresses to allow-list in their firewalls, and the API needs extremely low latency. Which solution meets these requirements?
A. Application Load Balancer with an Elastic IP address
B. Network Load Balancer with an Elastic IP address in each Availability Zone
C. Classic Load Balancer
D. Amazon CloudFront distribution with the instances as origin

**Q3.** A company is consolidating three microservices — `/orders`, `/users` and `/catalog` — that currently each have their own load balancer. It wants a single entry point with the LEAST operational overhead. What should it do?
A. Use one Network Load Balancer with a listener per service port
B. Use one Application Load Balancer with path-based listener rules forwarding to a target group per service
C. Use Route 53 weighted routing between the three existing load balancers
D. Use a Gateway Load Balancer

**Q4.** A security team must route all inbound and outbound VPC traffic through a fleet of third-party virtual firewall appliances that must scale and remain highly available. What should a solutions architect recommend?
A. AWS WAF on an Application Load Balancer
B. A Network Load Balancer in front of the appliances
C. A Gateway Load Balancer with Gateway Load Balancer endpoints and updated route tables
D. Security groups and network ACLs

**Q5.** An e-commerce site's traffic increases sharply every weekday at 9:00 AM. Instances take about 8 minutes to become ready, so customers see slow responses each morning until scaling catches up. Which solution is MOST effective?
A. Simple scaling policy with a shorter cooldown
B. Scheduled scaling action that raises capacity shortly before 9:00 AM on weekdays
C. Enable cross-zone load balancing
D. Increase the health check grace period

**Q6.** A company runs worker instances in an Auto Scaling group that process messages from an Amazon SQS queue. During peaks, the queue grows and processing falls behind even though CPU on the workers stays moderate. How should the group be scaled?
A. Target tracking on average CPU utilization of 40%
B. Target tracking on a custom metric of SQS messages visible divided by the number of InService instances
C. Scheduled scaling every hour
D. Step scaling on ALB request count

**Q7.** Application logs on EC2 instances in an Auto Scaling group are lost when the group scales in. The company wants to keep the logs with minimal changes to the application. What should a solutions architect do?
A. Enable termination protection on all instances
B. Add a lifecycle hook for instance termination that triggers a script to copy logs to Amazon S3 before the instance terminates
C. Increase the deregistration delay to 3600 seconds
D. Use a warm pool

**Q8.** Users of a web application report that their shopping carts are emptied when the Auto Scaling group terminates instances. The company wants the most resilient and scalable fix. What should the architect do?
A. Enable sticky sessions on the Application Load Balancer
B. Store session data in Amazon ElastiCache or Amazon DynamoDB and make the web tier stateless
C. Enable scale-in protection on all instances
D. Store sessions on an instance store volume

**Q9.** An Application Load Balancer is enabled in two Availability Zones. AZ a has 2 instances and AZ b has 6 instances. The 2 instances in AZ a are overloaded while AZ b's are underused. Cross-zone load balancing has been explicitly disabled on the target group. What is the simplest fix?
A. Switch to a Network Load Balancer
B. Re-enable cross-zone load balancing on the target group
C. Add a third Availability Zone
D. Enable sticky sessions

**Q10.** A company wants to cut the cost of a stateless, fault-tolerant web fleet in an Auto Scaling group while always keeping a baseline of 4 instances that are never interrupted. What should it configure?
A. All Spot Instances with the lowest-price allocation strategy
B. A mixed instances policy with an On-Demand base capacity of 4, Spot for capacity above the base across multiple instance types, using the price-capacity-optimized strategy
C. Dedicated Hosts for all instances
D. A launch configuration with Spot pricing

<details>
<summary><strong>Answers</strong></summary>

1. **B** — The ALB already stops routing to the hung instances, but the ASG only replaces what *its* health check type says is unhealthy. ELB health checks close that gap.
2. **B** — NLB supports one Elastic IP per AZ, handles TCP at very low latency. ALBs can't have Elastic IPs (A); CloudFront (D) is for HTTP(S) and doesn't give a small fixed IP set.
3. **B** — Path-based rules on one ALB with a target group per service is the textbook microservices pattern. NLB (A) can't route by path.
4. **C** — GWLB is designed for transparent, scalable third-party appliance insertion via route tables and GWLB endpoints. WAF (A) is a layer 7 web filter, not a network firewall fleet.
5. **B** — The spike is predictable in time; scheduled scaling launches capacity early enough to absorb the 8-minute boot. (Predictive scaling or a warm pool could also help, but neither is offered here; simple scaling (A) still reacts too late.)
6. **B** — The work is queue-bound, not CPU-bound; backlog per instance is the AWS-recommended metric for SQS workers.
7. **B** — A termination lifecycle hook (Terminating:Wait) gives time to ship logs. Termination protection (A) doesn't apply to ASG scale-in (that's scale-in protection) and would stop scaling anyway. (Streaming logs continuously with the CloudWatch agent is also valid in real life.)
8. **B** — Externalising session state is the resilient answer; stickiness (A) still loses carts when the sticky instance dies.
9. **B** — With cross-zone on, each ALB node spreads traffic across all 8 healthy targets regardless of AZ.
10. **B** — Mixed instances policy: On-Demand base for the uninterruptible baseline, diversified Spot above it with price-capacity-optimized. Launch configurations (D) are deprecated and don't support mixed instances.

</details>

---

## 10. Flashcards (cover the right column)

| Prompt | Answer |
| --- | --- |
| ALB OSI layer / NLB / GWLB | 7 / 4 / 3 |
| Which LB supports Elastic IPs? | NLB |
| Which LB can target Lambda? | ALB |
| Which LB can target an ALB? | NLB |
| GWLB encapsulation protocol and port | GENEVE, UDP 6081 |
| LB required for a PrivateLink endpoint service | NLB (GWLB for appliances) |
| How an ALB target learns the client IP | `X-Forwarded-For` header |
| Cross-zone default: ALB / NLB | On / Off |
| Default deregistration delay | 300 seconds |
| ALB default idle timeout | 60 seconds |
| `503` from an ALB usually means | No healthy/registered targets |
| `504` from an ALB usually means | Target didn't respond in time |
| Can AWS WAF attach to an NLB? | No — ALB, CloudFront, API Gateway, AppSync, Cognito |
| ASG default health check type | EC2 status checks |
| Simple scaling default cooldown | 300 seconds |
| Policy for "keep metric at X" | Target tracking |
| Policy for a known-time spike | Scheduled |
| Policy that forecasts from history | Predictive |
| Metric to scale SQS workers | Backlog per instance (custom metric) |
| First step of default termination policy | Choose the AZ with the most instances |
| Lifecycle hook default timeout | 1 hour |
| Pre-initialised stopped instances for fast scale-out | Warm pool |
| Rolling AMI update for an ASG | Instance refresh |
| On-Demand baseline + Spot above | Mixed instances policy (On-Demand base capacity) |
| Replacement for launch configurations | Launch templates |
| Do ELB or ASG span Regions? | No — both Regional |

## 11. Before you close today

- [ ] Read sections 1–7; redraw the ALB/NLB/GWLB comparison table from memory
- [ ] Lab done and torn down (ASG → ALB → target group → launch template → SGs)
- [ ] 10 questions above + 20 bank questions on ELB/Auto Scaling
- [ ] Misses logged in `miss-log.md`
- [ ] Quick Day 2 recap: which purchasing option fits the On-Demand base in today's mixed instances policy, and why Spot suits the rest?

**Tomorrow (Day 4):** VPC networking — subnets, route tables, IGW/NAT, security groups vs NACLs, VPC endpoints and PrivateLink, peering vs Transit Gateway, and Site-to-Site VPN vs Direct Connect.
