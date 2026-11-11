# Day 13 — Gap-Closing + Practice Exam 3

**Domain weight:** all four, in exam proportion — *Secure* 30% · *Resilient* 26% · *High-performing* 24% · *Cost-optimized* 20%. Today's lesson goes after the **second layer** of confusion: not "which service?" (Day 7 §6 covered the top ten pairs) but "which *setting*, *mode* or *variant* of the service?" That's where most people who pass the first-layer questions still lose points.
**Time plan (≈ 4 h 45 min):** Gap lesson 75 min · Lab 30 min · **Practice Exam 3: 130 min, timed, no notes** · Score + go/no-go 45 min.

By the end of today you should be able to:

- Classify every miss you make into one of four failure types and apply the matching fix, so the last 48 hours go to the right place.
- Pick the right **variant** within a service family: Multi-AZ instance vs Multi-AZ cluster vs Aurora replica vs Global Database; Secrets Manager vs Parameter Store; gateway vs interface endpoint vs PrivateLink endpoint service; Object Lock governance vs compliance.
- Pick the right **knob** on a service you already know: SQS visibility timeout vs retention vs long polling vs DLQ; Lambda reserved vs provisioned concurrency; ALB deregistration delay; EFS throughput mode; Spot allocation strategy.
- Explain the less-obvious **layering** of access control: NACL ephemeral ports, ABAC tags, external IDs, endpoint policies, and why ACM certificates for CloudFront live in `us-east-1`.
- Sit Practice Exam 3, score it by domain, and make the **go/no-go call** for 2026-11-13 using the rule: 80%+ on two full practice exams → sit the exam.

> **Miss-log check:** `miss-log.md` still has **no entries** in the repo — no score lines from Exams 1 or 2 and no logged misses. So today can't be tailored to *your* specific patterns. Instead, sections 2–9 target the second-layer confusions that most often survive into the last week of SAA prep, and the exam is built around them. **Two things to do before you start:** (1) if you scored Exams 1 and 2 on paper, add the score lines and misses to `miss-log.md` now — then re-read only the sections below that match them; (2) after Exam 3 tonight, log everything. The go/no-go call in §16 needs your Exam 2 and Exam 3 scores side by side.

---

## 1. Diagnose before you study: the four ways a question is lost

With two days left, reading more material is rarely the highest-value move. Fixing *how* you miss is. Take every miss from Exams 1–2 (and tonight's Exam 3) and tag it with one of these:

| Type | What happened | Tell-tale sign in your notes | Fix (time-boxed) |
| --- | --- | --- | --- |
| **K — Knowledge gap** | You didn't know a feature existed or what it does | "Never heard of Backtrack / Resolver endpoints" | Read the one decision-table row + the explanation. Make one flashcard. 5 min each. |
| **Q — Qualifier miss** | Both your pick and the answer work; the answer fits *MOST cost-effective* / *LEAST overhead* better | "Mine would work too…" | Re-read the stem, underline the qualifier, say why yours loses on it. Drill §10. |
| **C — Constraint missed** | You overlooked one hard requirement (no internet, existing Kafka, must be SMB, RPO seconds) | "Didn't notice it said *on-premises* / *Windows* / *within 5 minutes*" | Before reading options: list every hard constraint in the stem. Eliminate by constraint first. |
| **O — Overthinking / changed answer** | First instinct was right, you talked yourself out of it, or you imagined requirements not in the stem | "Changed from B to D" | Rule: only change an answer if you can point to a word in the stem that proves the first pick wrong. |

**How to read the totals:**

- Mostly **K** → you're in content mode. Spend Day 13's lesson + Day 14's cheat sheet on the rows you missed; don't re-read whole days.
- Mostly **Q** or **C** → content is fine; your *reading* is leaking points. These are the cheapest points to recover — drill §10 and slow down on the stem, not the options.
- Mostly **O** → trust the first pass. Flag, move on, and only revisit flagged ones with fresh eyes at the end.

Add a column to your miss log if it helps: `| Day | Topic | I picked | Correct | Why | Type |`.

---

## 2. Gap cluster A — private connectivity: which *kind* of private?

Day 4 taught each option; the misses come from mixing up **who initiates**, **which direction**, and **from where**.

```
                    ┌───────────── Your VPC ─────────────┐
 On-prem ──DX/VPN──►│  Interface endpoint (ENI, priv IP) │──► AWS service (S3, SQS, Secrets Manager, KMS, …)
                    │  ✔ reachable from on-prem/peered   │
                    │                                    │
                    │  Gateway endpoint (route target)   │──► S3 / DynamoDB only
                    │  ✘ NOT reachable from on-prem,     │
                    │    peered VPCs or via TGW          │
                    │                                    │
 Consumer VPC ─────►│  Interface endpoint ──PrivateLink──┼──► Provider's NLB ──► provider's service
 (other account,    │  (consumer creates it)             │    ("endpoint service" — provider creates it)
  overlapping CIDR  └────────────────────────────────────┘
  is fine)
```

| You need… | Answer | Why not the near-miss |
| --- | --- | --- |
| Restrict *which buckets* instances can reach through the S3 gateway endpoint (stop data exfiltration to a personal bucket) | **VPC endpoint policy** listing allowed bucket ARNs (or `aws:ResourceOrgID` = your org) | A bucket policy only controls *your* buckets — it can't stop writes to someone else's bucket |
| Lambda in private subnets must call Secrets Manager / SQS / KMS without NAT | **Interface endpoint** for that service, private DNS on | Gateway endpoints exist only for S3 and DynamoDB |
| Offer your service to 200 customer VPCs, one-way, overlapping CIDRs | **PrivateLink endpoint service** (NLB or GWLB in front) | Peering/TGW need non-overlapping CIDRs and open whole networks |
| Two VPCs, lots of traffic, cheapest | **VPC peering** (no hourly or per-GB processing charge; normal data-transfer rates apply) | Transit Gateway adds per-attachment-hour and per-GB processing charges — worth it at scale, not for two VPCs |
| Many VPCs + on-prem, transitive routing | **Transit Gateway** | Peering is non-transitive: A↔B and B↔C does **not** give A↔C |
| On-prem must resolve names in a Route 53 **private** hosted zone | **Route 53 Resolver inbound endpoint** (on-prem DNS forwards to it) | *Outbound* endpoint + forwarding rule is the other direction: VPC resolving on-prem names |

> **Trap — inbound vs outbound Resolver endpoints.** Name them from the VPC's point of view: queries coming **in** to the VPC from on-prem = **inbound**. Queries going **out** of the VPC to on-prem DNS = **outbound** (with a forwarding rule, shareable via RAM).

Endpoint policy that pins an S3 gateway endpoint to approved buckets:

```json
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": "*",
    "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::acme-app-data", "arn:aws:s3:::acme-app-data/*",
      "arn:aws:s3:::amazonlinux-2-repos-ap-south-1/*"
    ]
  }]
}
```

Note the second bucket: if instances pull OS packages from Amazon-hosted repos through the endpoint, a too-tight policy breaks `dnf update`. Real-world gotcha, occasional exam distractor.

---

## 3. Gap cluster B — database HA and DR variants

Day 6 and Day 11 covered these one by one. Put side by side, the discriminators are **sync vs async**, **readable or not**, **same Region or cross-Region**, and **how fast**.

| Variant | Replication | Standby readable? | Failover | Scope | Pick when the stem says… |
| --- | --- | --- | --- | --- | --- |
| **RDS Multi-AZ DB instance** | Synchronous, 1 standby | **No** | Automatic, typically 1–2 min | 2 AZs | "Highly available", "automatic failover", standby not needed for reads |
| **RDS Multi-AZ DB cluster** (MySQL, PostgreSQL) | Semi-sync, 2 standbys in 3 AZs | **Yes** (reader endpoint) | Automatic, typically faster (AWS quotes under ~35 s — check current docs) | 3 AZs | "Faster failover **and** readable standbys", "lower write latency than Multi-AZ instance" |
| **RDS read replica** | Async | Yes | **Manual promote** | Same or cross-Region | "Offload reads", "cross-Region DR copy you can promote" |
| **Aurora replicas** (up to 15) | Shared cluster storage (6 copies / 3 AZs) | Yes | Automatic to a replica by **priority tier** | Region | "Aurora", "fast failover", "scale reads" |
| **Aurora custom endpoint** | — | — | — | — | "Send reporting queries to *specific* larger replicas, keep them off the others" |
| **Aurora Global Database** | Storage-level, typically ~1 s lag | Yes, secondary Regions | Managed cross-Region failover/switchover, ~1 min | Multi-Region | "RPO ~seconds, RTO ~minute, relational, cross-Region" |
| **Aurora Backtrack** (Aurora MySQL only, enabled on the cluster) | — | — | Rewinds the **same** cluster in place | — | "Undo a bad `DELETE` in minutes without restoring a new cluster" |
| **Point-in-time restore** (RDS/Aurora) | — | — | Creates a **new** instance/cluster | — | "Restore to 14:05 yesterday", retention 1–35 days |

> **Trap — PITR vs Backtrack.** PITR always creates a *new* DB (new endpoint → app config change). Backtrack rewinds the existing Aurora MySQL cluster in place, quickly, within its configured window. If the stem says "fastest", "same endpoint", "Aurora MySQL" → Backtrack. If it's RDS or Aurora PostgreSQL → PITR.

> **Trap — "Multi-AZ" doesn't scale reads** — except the *Multi-AZ DB cluster* variant, whose two standbys *are* readable. The exam now uses both phrasings; read the noun after "Multi-AZ".

### Caching variants (ElastiCache)

| Question | Answer |
| --- | --- |
| Session store / cache must survive a node or AZ failure | **ElastiCache for Valkey or Redis OSS** with replicas + **Multi-AZ auto-failover** (Memcached has no replication) |
| Real-time leaderboard, ranking, counters | Valkey/Redis **sorted sets** |
| Simple, multithreaded, horizontally sharded cache; losing it is fine | Memcached |
| Cached data must **never** be stale after a write | **Write-through** (update cache on every DB write) |
| Cache only what's actually read; tolerate a miss penalty and some staleness | **Lazy loading** (cache-aside) + **TTL** |

### Purpose-built databases — the one-liners

| Stem says… | Pick |
| --- | --- |
| "MongoDB-compatible", "JSON documents", migrating Mongo workloads | **Amazon DocumentDB** |
| "Relationships", "friends-of-friends", "fraud rings", "knowledge graph" | **Amazon Neptune** |
| "Cassandra (CQL)" | **Amazon Keyspaces** |
| "Relational, spiky / unpredictable, scale in fine increments, no capacity planning" | **Aurora Serverless v2** |
| "Key-value, single-digit ms at any scale" | DynamoDB |
| "Search, relevance, log analytics" | OpenSearch Service |

---

## 4. Gap cluster C — messaging knobs (SQS, Lambda, SNS, EventBridge, MQ)

Most messaging misses aren't "SQS vs SNS" — they're "which SQS *setting*". Learn the symptom → knob mapping.

```
Producer ──► [ SQS queue ] ──ReceiveMessage──► Consumer ──DeleteMessage──► gone
               │   ▲                               │
               │   └── message reappears after ────┘  (if not deleted within the
               │       VisibilityTimeout                visibility timeout)
               │
               ├── after maxReceiveCount receives ──► DLQ (redrive policy)
               └── after MessageRetentionPeriod ────► deleted (default 4 days, max 14)
```

| Symptom in the stem | Knob |
| --- | --- |
| Messages processed **twice**; processing takes longer than the visibility timeout | **Raise the visibility timeout** above max processing time (or extend it with `ChangeMessageVisibility` heartbeats). For Lambda, AWS recommends a queue visibility timeout of at least 6× the function timeout. |
| Consumer outage of ~10 days; messages vanished | **Raise `MessageRetentionPeriod`** (default 4 days, max 14 days) |
| High SQS bill from empty `ReceiveMessage` responses | **Long polling** (`WaitTimeSeconds` up to 20) |
| A few bad messages block or loop forever | **DLQ** with `maxReceiveCount`; redrive back once fixed |
| Lambda batch of 10: one bad message makes all 10 retry | **`ReportBatchItemFailures`** — return only the failed message IDs |
| Exactly-once processing, ordering per customer | **FIFO** queue, `MessageGroupId` = customer, dedup ID or content-based dedup (5-minute window) |
| Downstream system allows only N concurrent calls | **Reserved concurrency = N** on the consuming Lambda (also guarantees it N); buffer the rest in SQS |
| No cold-start latency at a known 09:00 spike | **Provisioned concurrency** + Application Auto Scaling **scheduled** scaling |

> **Trap — reserved vs provisioned concurrency.** *Reserved* = a cap and a guarantee of how many can run (no extra charge, no pre-warming). *Provisioned* = pre-initialised environments that remove cold starts (extra charge). "Limit" → reserved. "Latency / cold start" → provisioned.

### Choosing the integration service (beyond Day 8's table)

| Stem says… | Pick | Near-miss and why it loses |
| --- | --- | --- |
| "Existing app uses **JMS / AMQP / MQTT / STOMP**, migrate with minimal code changes" | **Amazon MQ** (ActiveMQ or RabbitMQ) | SQS/SNS need code rewrites to their APIs |
| "Events from a **SaaS partner** (e.g., a helpdesk or payments SaaS)", "route by event content to many targets", "schema registry", "archive & replay" | **EventBridge** (partner event source) | SNS filter policies work on attributes/body but no SaaS partner sources, archive or schema registry |
| "Run a task every hour / at a time" with least ops | **EventBridge Scheduler** → Lambda / ECS task | A cron EC2 instance is ops + idle cost |
| "Process millions of S3 objects in parallel, serverless" | **Step Functions Distributed Map** | One Lambda can't run past 15 minutes; a hand-built fan-out needs coordination code |
| "Real-time two-way messages to browsers (chat, live scores)" | **API Gateway WebSocket API** (or AppSync subscriptions for GraphQL) | REST/HTTP APIs are request-response only |
| "Streaming throughput unknown / spiky, don't want to manage shards" | **Kinesis Data Streams on-demand mode** | Provisioned mode needs shard math and resharding |

---

## 5. Gap cluster D — encryption, keys and secrets: the variants

| Question | Answer | Why |
| --- | --- | --- |
| Config values + a few secrets, **no rotation**, lowest cost | **SSM Parameter Store** (standard tier, `SecureString`) | Standard parameters have no storage charge; Secrets Manager charges per secret per month (check current pricing) |
| DB credentials must **rotate automatically**, cross-Region replicas | **Secrets Manager** | Built-in rotation (managed rotation for RDS/Aurora/Redshift/DocumentDB) and native replication |
| Encrypt in Region A, decrypt in Region B **without re-encrypting** or cross-Region KMS calls | **KMS multi-Region keys** (primary + replicas share key material and key ID) | A single-Region key can't be used outside its Region |
| SSE-KMS on a high-request bucket → KMS throttling / big KMS bill | **S3 Bucket Keys** | S3 uses a short-lived bucket-level data key, cutting KMS requests dramatically |
| Third-party SaaS assumes a role in your account; prevent the confused-deputy problem | Trust policy condition on **`sts:ExternalId`** | The external ID is unique per customer, so another customer of the vendor can't trick the vendor into using your role |
| HTTPS certificate for a **CloudFront** distribution | **ACM certificate in `us-east-1`** | CloudFront is global and reads certificates only from N. Virginia. ALB certificates are in the ALB's own Region. |
| TLS must terminate **only** on the instances (end-to-end, LB must not decrypt) | **NLB with a TCP listener on 443** (TLS passthrough) | ALB always terminates TLS (it can re-encrypt to targets, but it sees plaintext) |
| Encrypt an existing **unencrypted** EBS volume | Snapshot → **copy snapshot with encryption** (or create volume from snapshot with encryption on) → swap volume | You can't flip encryption on an existing volume in place |

```bash
# Encrypt an existing volume: snapshot → encrypted volume → swap
aws ec2 create-snapshot --volume-id vol-0abc --description pre-encrypt
aws ec2 create-volume --snapshot-id snap-0def --availability-zone ap-south-1a \
  --encrypted --kms-key-id alias/app-ebs --volume-type gp3
# stop instance → detach old → attach new as same device → start
```

> **Trap — "enable EBS encryption by default"** (Day 2/7) only affects **new** volumes. Existing ones still need the snapshot-copy route.

---

## 6. Gap cluster E — access-control layering you might be shaky on

### NACL ephemeral ports (the stateless trap)

```
Client 203.0.113.9:51544 ──► web 10.0.1.10:443     inbound NACL must allow 443
Client 203.0.113.9:51544 ◄── web 10.0.1.10:443     outbound NACL must allow 1024–65535 to 0.0.0.0/0
                                                    (the client's ephemeral port)
```

Security groups are stateful (return traffic is automatic); NACLs aren't. If a stem says "the NACL allows inbound 443 and the site still times out", the answer is an **outbound ephemeral-port rule**. The same applies in reverse for instances making outbound calls: the *inbound* NACL needs the ephemeral range for replies.

### ABAC — permissions that scale with tags

Instead of one policy per project, one policy that matches the caller's tag to the resource's tag:

```json
{
  "Effect": "Allow",
  "Action": ["ec2:StartInstances", "ec2:StopInstances"],
  "Resource": "*",
  "Condition": {
    "StringEquals": { "aws:ResourceTag/project": "${aws:PrincipalTag/project}" }
  }
}
```

Pick ABAC when the stem says "hundreds of projects", "new projects weekly", "don't want to update policies each time". Pair with an SCP or tag policy that stops people retagging resources into other projects.

### The other layers, one line each

| Layer | One-line rule |
| --- | --- |
| **ALB authentication** (`authenticate-oidc` / `authenticate-cognito` listener actions) | Put SSO in front of an internal web app **without changing app code** (HTTPS listener required) |
| **Object Lock governance** | Blocks deletes/overwrites, but principals with `s3:BypassGovernanceRetention` can override — "admins can delete in exceptional cases" |
| **Object Lock compliance** | Nobody, including root, can shorten or remove retention — "regulator", "no one, ever" |
| **S3 Block Public Access (account level)** + **SCP denying `s3:PutAccountPublicAccessBlock`** | Prevent any bucket in the account being made public, and stop admins switching that off |
| **AD Connector** | Directory *proxy* — users sign in to AWS apps with on-prem AD credentials, nothing cached or synced in AWS |
| **AWS Managed Microsoft AD** | A real AD in AWS (trusts with on-prem possible) — when AD-aware apps must run in AWS even if on-prem is unreachable |
| **AWS Firewall Manager** | Apply WAF rules, Shield Advanced, security groups, Network Firewall policies **across all accounts in the org**, auto-applied to new resources |
| **Amazon Detective** | *Investigate* a GuardDuty finding — graph of related entities and activity over time. GuardDuty detects; Detective explains. |
| **GuardDuty Malware Protection for S3** | Scan newly uploaded objects for malware, tag results, no agents |
| **Systems Manager Patch Manager** + maintenance windows | Patch OS fleets on a schedule with patch baselines and compliance reporting. Inspector finds the CVEs; Patch Manager fixes them. |
| **CloudFront signed URLs vs signed cookies** | One file per link → signed URL. Many files (a subscriber library, HLS segments) → signed cookies |

---

## 7. Gap cluster F — resilience by design (not by service)

### Static stability: the N+1 AZ math

"Must handle peak with **no scaling delay** even if an AZ fails" means *pre-provision* enough in the surviving AZs.

```
Peak needs 6 instances.
  2 AZs: lose 1 → survivor must hold 6     → 6 + 6 = 12 instances
  3 AZs: lose 1 → 2 survivors hold 6 (3 each) → 3 + 3 + 3 = 9 instances  ← cheaper
```

More AZs → less over-provisioning. This is why "spread across three AZs" often beats two on *both* resilience and cost.

### Deployment safety knobs

| Stem says… | Knob |
| --- | --- |
| In-flight requests fail when instances/tasks are removed | **Deregistration delay** (connection draining) on the target group — default 300 s |
| A bad ECS deployment keeps failing health checks; roll back automatically | **ECS deployment circuit breaker** with rollback enabled |
| Shift 10% of traffic to the new version first | ALB **weighted target groups**, or CodeDeploy canary/linear (Lambda/ECS) |
| Route 53 must health-check a resource with **only a private IP** | **CloudWatch alarm–based health check** (Route 53 health checkers run on the internet) |

### Backup and data-protection variants

| Need | Answer |
| --- | --- |
| Automated EBS snapshots/AMIs on a schedule with retention, nothing else | **Amazon Data Lifecycle Manager** (simplest), or AWS Backup if many services |
| Backups must survive compromise of the production **account** | **AWS Backup cross-account copy** to a vault in a separate (e.g., backup/log-archive) account in the org; add Vault Lock |
| Snapshots kept for years, restored rarely | **EBS Snapshots Archive** tier (cheaper storage, hours-to-days restore, minimum retention — check current docs) |
| Replicate **existing** objects after enabling CRR | **S3 Batch Replication** (CRR only replicates new objects) |
| 15-minute replication SLA | S3 **Replication Time Control** (Exam 2) |

### DR strategy from RTO/RPO (Day 11 recap — still the #1 resilience question type)

| RTO | RPO | Strategy |
| --- | --- | --- |
| Hours–a day | Hours | Backup & restore |
| Tens of minutes–hours | Minutes | Pilot light (data live, compute off) |
| **Minutes** | **Seconds** | **Warm standby** (scaled-down full stack running) |
| Near zero | Near zero | Multi-site active-active |

> **Trap:** pilot light vs warm standby. Both replicate data continuously. If the stem needs the app **serving within minutes**, compute must already be running → warm standby. Pilot light still has to launch and scale compute.

---

## 8. Gap cluster G — performance variants

| Stem says… | Pick | Why |
| --- | --- | --- |
| gp2 volume sized huge only to get IOPS | **gp3** with IOPS/throughput provisioned independently of size | gp2 IOPS scale with size (3 IOPS/GiB); gp3 decouples them and costs less per GiB |
| EFS throughput unpredictable, bursty, don't want to size it | **EFS Elastic throughput** | Bursting depends on stored size; Provisioned pays for a fixed level |
| Multi-protocol NFS **and** SMB, snapshots, clones, migrating from on-prem NetApp | **FSx for NetApp ONTAP** | FSx Windows is SMB-only; EFS is NFS-only |
| Download very large S3 objects faster | Parallel **byte-range GETs** (and multipart upload for writes) | S3 scales per prefix and per connection; parallelism uses it |
| Thousands of batch jobs with dependencies, queues, Spot, containers | **AWS Batch** | Manages the compute environment and job scheduling for you |
| Heavy reporting queries must run only on two large Aurora replicas, away from the replicas serving the app | Aurora **custom endpoint** over those two replicas | The default reader endpoint balances across *all* replicas |
| Java Lambda functions need lower cold-start latency at no extra idle cost | **Lambda SnapStart** (check current docs for supported runtimes and pricing) | Provisioned concurrency also removes cold starts but bills while idle |

---

## 9. Gap cluster H — cost variants

| Stem says… | Pick | Why it beats the near-miss |
| --- | --- | --- |
| Steady use of **one instance family in one Region** for 3 years, maximum discount | **EC2 Instance Savings Plans** | Bigger discount than Compute SP; less flexible (Compute SP for Fargate/Lambda/family changes) |
| Spot fleet: lowest price **and** fewest interruptions | **`price-capacity-optimized`** allocation strategy | `lowest-price` chases cheap pools that get reclaimed most |
| Partners download your large public dataset; they should pay transfer | **S3 Requester Pays** | Requester must be an authenticated AWS account |
| Dev/test instances idle nights and weekends | Stop them on a schedule (**EventBridge Scheduler** / Instance Scheduler on AWS) | ~70% of hours are nights/weekends |
| CloudWatch Logs growing forever | Set **log group retention**; export old logs to S3 if needed | Default retention is *never expire* |
| DynamoDB table where storage dominates cost, items read rarely | **DynamoDB Standard-IA table class** | Lower storage price, higher per-request price |
| Aurora bill dominated by I/O charges | **Aurora I/O-Optimized** cluster configuration | No per-I/O charges; AWS suggests it when I/O exceeds ~25% of Aurora spend |
| Accessed once a quarter, needs milliseconds when it is | **S3 Glacier Instant Retrieval** | Cheaper storage than Standard-IA for rarely read data; retrieval fees and a 90-day minimum apply |
| SSE-KMS request charges high | **S3 Bucket Keys** | Fewer KMS calls |
| Two VPCs exchanging a lot of data | **VPC peering** rather than Transit Gateway | No TGW processing charge |

---

## 10. Qualifier drills — fix the "Q" and "C" misses

Three stems, same scenario, three different right answers. Do this exercise once out loud; it rewires how you read.

> *A company processes uploaded images. Each takes ~30 s of CPU. Volume is a few thousand a day, arriving in unpredictable bursts.*

| Qualifier | Answer | Why the others lose |
| --- | --- | --- |
| …with the **LEAST operational overhead** | S3 event → **Lambda** | Nothing to patch or scale |
| …and each job now needs **25 minutes and a GPU** | S3 event → SQS → **AWS Batch** (or ECS on EC2 GPU instances) | Lambda caps at 15 min and has no GPU |
| …**MOST cost-effective** at a steady 50,000 jobs/hour, 24/7, for 3 years | SQS → **EC2 ASG with Savings Plans + Spot** | At constant high utilisation, per-invocation pricing loses to committed instances |

**Constraint-first reading routine (do it on every Exam 3 question):**

1. Read the **last sentence** first — it holds the qualifier and how many answers to pick.
2. Read the stem and write the hard constraints as single words: *on-prem*, *SMB*, *RPO 5 s*, *no code change*, *Kafka*.
3. Cross out any option that violates a constraint. You usually get down to two.
4. Compare the last two **only** on the qualifier.

**Multi-response (*Choose two*) tips:** the two answers are usually *complementary halves* (a control + a guard on that control; a buffer + a limiter), not two alternative ways to do the same thing. If two options are near-duplicates, at most one is right.

---

## 11. Exam decision table — the gap edition

Second-layer picks only. Days 1–12 hold the first-layer tables; Day 14 merges them all.

| If the question says… | Pick |
| --- | --- |
| Stop instances copying data to buckets outside the company through the S3 endpoint | **Endpoint policy** allowing only approved buckets (or `aws:ResourceOrgID`) |
| Lambda in a VPC, no NAT, must call Secrets Manager / SQS / KMS | **Interface VPC endpoint** for that service |
| Expose a service privately to many customer VPCs, overlapping CIDRs | **PrivateLink endpoint service** behind an NLB |
| On-prem resolves Route 53 private zone names | **Resolver inbound endpoint** |
| VPC resolves on-prem names | **Resolver outbound endpoint** + forwarding rule |
| Faster RDS failover **and** readable standbys | **Multi-AZ DB cluster** deployment |
| Route reporting to specific Aurora replicas | **Aurora custom endpoint** |
| Undo a bad write on Aurora MySQL in minutes, same cluster | **Backtrack** |
| Restore RDS to a point in time | **PITR** → new instance, repoint app |
| Session cache must survive node/AZ failure | **ElastiCache (Valkey/Redis OSS)** with Multi-AZ auto-failover |
| Cache must never serve stale data after writes | **Write-through** |
| Leaderboard | Valkey/Redis **sorted sets** |
| MongoDB-compatible / graph / Cassandra | **DocumentDB** / **Neptune** / **Keyspaces** |
| Spiky relational workload, no capacity planning | **Aurora Serverless v2** |
| Messages processed twice | Raise **visibility timeout** above processing time |
| Messages lost after a long consumer outage | Raise **retention period** (max 14 days) |
| Empty receives driving cost | **Long polling** |
| One bad message fails a whole Lambda batch | **`ReportBatchItemFailures`** |
| Protect a downstream that allows only N concurrent calls | **Reserved concurrency = N** + SQS buffer |
| Remove cold starts at a known spike | **Provisioned concurrency** with scheduled scaling |
| JMS / AMQP / MQTT app, minimal changes | **Amazon MQ** |
| SaaS partner events, content routing, archive/replay | **EventBridge** |
| Millions of S3 objects processed in parallel, serverless | **Step Functions Distributed Map** |
| Real-time push to browsers | **API Gateway WebSocket API** |
| Unpredictable stream throughput, no shard management | **Kinesis on-demand** |
| Non-rotating config + secrets, cheapest | **Parameter Store** SecureString |
| Rotating DB credentials | **Secrets Manager** |
| Same key usable in two Regions | **KMS multi-Region keys** |
| KMS cost/throttling from S3 SSE-KMS | **S3 Bucket Keys** |
| Vendor assumes a role in your account | Trust policy with **`sts:ExternalId`** |
| CloudFront custom domain certificate | **ACM in `us-east-1`** |
| TLS terminates only on the instances | **NLB TCP:443 passthrough** |
| Encrypt an existing EBS volume | **Snapshot → encrypted copy/volume → swap** |
| NACL allows 443 in, site still times out | **Outbound ephemeral ports 1024–65535** |
| Permissions for hundreds of projects without new policies | **ABAC** (`aws:PrincipalTag` = `aws:ResourceTag`) |
| SSO in front of an internal web app, no code changes | **ALB `authenticate-oidc` / Cognito action** |
| Retention that admins can override in emergencies | **Object Lock governance** + `s3:BypassGovernanceRetention` |
| No bucket can ever be public; admins can't undo it | **Account-level Block Public Access** + **SCP** denying `s3:PutAccountPublicAccessBlock` |
| Sign in to AWS apps with on-prem AD, no sync | **AD Connector** |
| WAF/SG policies across every account automatically | **Firewall Manager** |
| Investigate the root cause of a GuardDuty finding | **Detective** |
| Scan uploads for malware | **GuardDuty Malware Protection for S3** |
| Patch OS fleets on a schedule | **SSM Patch Manager** + maintenance windows |
| Many protected files per subscriber | **CloudFront signed cookies** |
| Survive an AZ loss at peak with no scaling delay | **Static stability**: pre-provision so surviving AZs carry peak (3 AZs → 1.5× peak) |
| In-flight requests fail on scale-in/deploy | **Deregistration delay** |
| Auto-rollback failed ECS deployments | **Deployment circuit breaker** with rollback |
| Health-check a private-IP-only resource in Route 53 | **CloudWatch alarm–based health check** |
| Scheduled EBS snapshots, simplest | **Data Lifecycle Manager** |
| Backups survive account compromise | **AWS Backup cross-account copy** + Vault Lock |
| Replicate existing objects | **S3 Batch Replication** |
| RTO minutes, RPO seconds, cheapest | **Warm standby** |
| Decouple IOPS from volume size | **gp3** |
| Bursty EFS throughput | **EFS Elastic throughput** |
| NFS + SMB + NetApp features | **FSx for NetApp ONTAP** |
| Thousands of containerised batch jobs with queues and Spot | **AWS Batch** |
| One family, one Region, max discount | **EC2 Instance Savings Plans** |
| Spot with fewest interruptions at low price | **`price-capacity-optimized`** |
| Data consumers pay for downloads | **Requester Pays** |
| Rarely-read DynamoDB data dominating storage | **Standard-IA table class** |
| Aurora I/O charges dominate | **Aurora I/O-Optimized** |
| Quarterly access, ms retrieval | **S3 Glacier Instant Retrieval** |
| Years-long snapshot retention, rare restore | **EBS Snapshots Archive** |
| Logs kept forever by accident | **Log group retention** setting |

---

## 12. Lab (30 min, free tier) — feel three knobs you'll be asked about

Goal: watch an **SQS visibility timeout** redeliver a message and a **FIFO dedup ID** swallow a duplicate; prove that **Object Lock governance** blocks a version delete until you use the bypass permission; and use the **IAM policy simulator** to see a permissions boundary cap an identity policy — all without launching anything billable. Run CLI steps from **CloudShell** in `ap-south-1`.

### Console + CLI steps

**Part A — SQS knobs (10 min)**

```bash
REGION=ap-south-1
STD=$(aws sqs create-queue --queue-name lab13-std --query QueueUrl --output text)
FIFO=$(aws sqs create-queue --queue-name lab13.fifo \
  --attributes FifoQueue=true,ContentBasedDeduplication=false --query QueueUrl --output text)

# 1. Visibility timeout: receive without deleting, wait it out, receive again
aws sqs send-message --queue-url "$STD" --message-body "order-1"
aws sqs receive-message --queue-url "$STD" --visibility-timeout 5 \
  --attribute-names ApproximateReceiveCount --query 'Messages[0].Attributes'
sleep 6
aws sqs receive-message --queue-url "$STD" --wait-time-seconds 10 \
  --attribute-names ApproximateReceiveCount --query 'Messages[0].Attributes'
# → ApproximateReceiveCount is now "2": the same message came back because it wasn't deleted.
#   That's the "processed twice" symptom. Fix = visibility timeout > processing time.

# 2. FIFO dedup: same dedup ID twice within 5 minutes → one message
for i in 1 2; do
  aws sqs send-message --queue-url "$FIFO" --message-body "pay-42" \
    --message-group-id acct-42 --message-deduplication-id pay-42
done
aws sqs get-queue-attributes --queue-url "$FIFO" \
  --attribute-names ApproximateNumberOfMessages
# → "1". Do the same send twice on $STD (no dedup) and you'd see 2.
```

Note `--wait-time-seconds 10` in step 1: that's **long polling** — the call waits for a message instead of returning empty immediately.

**Part B — Object Lock governance (10 min)**

```bash
B=lab13-lock-$RANDOM$RANDOM
aws s3api create-bucket --bucket "$B" --region $REGION \
  --create-bucket-configuration LocationConstraint=$REGION \
  --object-lock-enabled-for-bucket          # also turns versioning on
aws s3api put-object-lock-configuration --bucket "$B" --object-lock-configuration \
  '{"ObjectLockEnabled":"Enabled","Rule":{"DefaultRetention":{"Mode":"GOVERNANCE","Days":1}}}'

echo "quarterly report" > report.txt
aws s3 cp report.txt "s3://$B/report.txt"   # the CLI adds the integrity checksum Object Lock requires
V=$(aws s3api list-object-versions --bucket "$B" --prefix report.txt \
     --query 'Versions[0].VersionId' --output text)

aws s3 rm "s3://$B/report.txt"               # succeeds! it only adds a delete marker
aws s3api delete-object --bucket "$B" --key report.txt --version-id "$V"
# → AccessDenied: the locked VERSION can't be deleted
aws s3api delete-object --bucket "$B" --key report.txt --version-id "$V" --bypass-governance-retention
# → succeeds, because your admin identity has s3:BypassGovernanceRetention.
#   In COMPLIANCE mode this would fail for everyone, including root, until retention expires.
```

Two exam lessons in one: Object Lock protects **versions**, not the key name (a delete marker still "hides" the object), and **governance** is bypassable by design.

**Part C — Permissions boundary in the simulator (5 min)**

```bash
cat > identity.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":["s3:*","iam:*"],"Resource":"*"}]}
EOF
cat > boundary.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":["s3:Get*","s3:List*"],"Resource":"*"}]}
EOF
aws iam simulate-custom-policy \
  --policy-input-list file://identity.json \
  --permissions-boundary-policy-input-list file://boundary.json \
  --action-names s3:GetObject s3:PutObject iam:CreateUser \
  --query 'EvaluationResults[].[EvalActionName,EvalDecision]' --output table
# → GetObject allowed; PutObject and CreateUser implicitDeny.
#   Effective permissions = identity ∩ boundary. The boundary granted nothing by itself.
```

Console alternative: **IAM → Policy simulator** (the standalone tool) lets you paste the same policies.

**Part D — Read the result (5 min).** Write one sentence per part in your notes: which knob, what symptom it fixes, and the near-miss knob it's confused with.

### Terraform variant

```hcl
terraform {
  required_providers {
    aws    = { source = "hashicorp/aws", version = "~> 5.0" }
    random = { source = "hashicorp/random", version = "~> 3.0" }
  }
}
provider "aws" { region = "ap-south-1" }

resource "random_id" "s" { byte_length = 4 }

resource "aws_sqs_queue" "std" {
  name                       = "lab13-std"
  visibility_timeout_seconds = 30
  message_retention_seconds  = 345600   # 4 days (default); max 1209600 = 14 days
  receive_wait_time_seconds  = 20       # long polling on by default for every consumer
}

resource "aws_sqs_queue" "fifo" {
  name                        = "lab13.fifo"
  fifo_queue                  = true
  content_based_deduplication = false
}

resource "aws_s3_bucket" "lock" {
  bucket              = "lab13-lock-${random_id.s.hex}"
  object_lock_enabled = true
}

resource "aws_s3_bucket_object_lock_configuration" "lock" {
  bucket = aws_s3_bucket.lock.id
  rule {
    default_retention {
      mode = "GOVERNANCE"
      days = 1
    }
  }
}

output "std_url"  { value = aws_sqs_queue.std.url }
output "fifo_url" { value = aws_sqs_queue.fifo.url }
output "bucket"   { value = aws_s3_bucket.lock.bucket }
```

`terraform init && terraform apply`, then run Parts A–C against the outputs (`STD=$(terraform output -raw std_url)` etc.). The policy simulator needs no Terraform.

### Teardown

Locked versions block bucket deletion, so remove them with the bypass first — **before** `terraform destroy` if you used Terraform.

```bash
# Delete every version and delete marker, bypassing governance retention
for q in Versions DeleteMarkers; do
  aws s3api list-object-versions --bucket "$B" --output text \
    --query "$q[].[Key,VersionId]" | while read -r k v; do
      [ "$k" = "None" ] && continue
      aws s3api delete-object --bucket "$B" --key "$k" --version-id "$v" --bypass-governance-retention
  done
done
aws s3api delete-bucket --bucket "$B"

aws sqs delete-queue --queue-url "$STD"
aws sqs delete-queue --queue-url "$FIFO"
rm -f report.txt identity.json boundary.json
# Terraform users: run the version loop with B=$(terraform output -raw bucket), then terraform destroy
```

Check: S3 console shows no `lab13-lock-*` bucket; SQS console shows no `lab13*` queues.

---

## 13. Warm-up: ten gap questions (15 min, untimed)

Do these before the timed exam. Each targets one gap cluster; tag any miss with K/Q/C/O from §1.

**Q1.** VPC A is peered with VPC B, and VPC B is peered with VPC C. Instances in A cannot reach instances in C. What is the simplest fix for these three VPCs?
A. Add a route in A's route table pointing C's CIDR at the A–B peering connection
B. Create a peering connection between A and C and add routes on both sides
C. Enable DNS resolution on the A–B peering connection
D. Replace the peering connections with a Site-to-Site VPN

**Q2.** Applications in a peered VPC need to read objects from S3 through an S3 gateway endpoint that lives in the central VPC. The requests fail. What should the architect do?
A. Add the peered VPC's CIDR to the gateway endpoint policy
B. Create an S3 interface endpoint in the central VPC (or a gateway endpoint in the peered VPC itself)
C. Enable transitive routing on the peering connection
D. Attach an internet gateway to the peered VPC

**Q3.** A worker takes up to 4 minutes to process each SQS message. The queue's visibility timeout is 30 seconds. Some orders are being shipped twice. What should the architect change?
A. Enable long polling
B. Add a dead-letter queue
C. Increase the visibility timeout to longer than the maximum processing time
D. Increase the message retention period

**Q4.** An application needs an RDS for MySQL password that rotates every 30 days without any custom code. Which service should store it?
A. SSM Parameter Store standard tier
B. AWS Secrets Manager with managed rotation
C. An encrypted S3 object
D. AWS KMS

**Q5.** A security team must block all traffic from one malicious /24 range to every instance in a subnet. What should they use?
A. A security group rule denying the range
B. A network ACL deny rule with a lower rule number than the allow rules
C. AWS WAF on the instances
D. A route table blackhole route

**Q6.** An RDS for PostgreSQL instance is CPU-bound from dashboard read queries. The company also wants protection from an AZ failure. *(Choose two.)*
A. Enable Multi-AZ (DB instance) deployment
B. Add a read replica and point the dashboards at it
C. Increase the backup retention period
D. Enable Performance Insights
E. Use a larger standby instance

**Q7.** An HPC simulation needs a high-throughput, sub-millisecond file system that lazily loads a 50 TB input dataset from S3 and writes results back. The file system is needed only for each two-day run. What should the architect choose?
A. Amazon EFS with Max I/O
B. FSx for Lustre scratch file system linked to the S3 bucket
C. FSx for Windows File Server
D. EBS io2 volumes with Multi-Attach

**Q8.** Order events must go to 15 different targets based on fields inside the event body, and the team wants to archive events and replay them after bug fixes. Which service fits with the least custom code?
A. Amazon SNS with one topic per target
B. Amazon EventBridge with rules and an archive
C. Amazon SQS with a consumer that routes messages
D. Kinesis Data Streams with 15 consumers

**Q9.** A company runs EC2, Fargate and Lambda and expects to shift between them over the next three years. It wants a commitment discount that follows the workload. What should it buy?
A. Standard Reserved Instances
B. EC2 Instance Savings Plans
C. Compute Savings Plans
D. Convertible Reserved Instances

**Q10.** A 2-minute cleanup script must run every hour against a DynamoDB table. Which option has the LEAST operational overhead?
A. A t3.micro instance with a cron job
B. EventBridge Scheduler invoking a Lambda function
C. An ECS service running continuously with a sleep loop
D. A Step Functions Standard workflow with a one-hour Wait state looping forever

<details>
<summary><strong>Answers</strong></summary>

1. **B** — Peering is non-transitive; A needs its own peering with C. For three VPCs that's simpler than a Transit Gateway; routes through B (A) don't work.
2. **B** — Gateway endpoints can't be used from outside their VPC (peering, TGW, VPN, DX). An interface endpoint has a private IP that's reachable across the peering.
3. **C** — The message becomes visible again while still being processed, so another worker takes it. The timeout must exceed processing time (plus idempotent processing as a backstop).
4. **B** — Secrets Manager has managed rotation for RDS. Parameter Store has no built-in rotation.
5. **B** — Only NACLs have deny rules; they're evaluated lowest number first. Security groups are allow-only.
6. **A, B** — Multi-AZ gives automatic failover; a read replica offloads reads. A Multi-AZ *DB instance* standby isn't readable.
7. **B** — FSx for Lustre links to S3, lazy-loads objects, exports results, and scratch file systems suit short-lived runs.
8. **B** — EventBridge rules match on event content and fan out to many targets; archive + replay is built in.
9. **C** — Compute Savings Plans apply across EC2 (any family/Region), Fargate and Lambda.
10. **B** — Serverless scheduling, no servers, pay per run. The Step Functions loop works but is contrived and runs for ever.

</details>

---

## 14. Practice Exam 3 — 65 questions, 130 minutes

**Rules:** one sitting, timer on, no notes, no console, no searching. ~2 minutes per question; past 3 minutes, pick your best guess, note the number, move on. Questions marked *(Choose two.)* need both picks correct to score. Write answers 1–65 on paper before opening the key. Use the constraint-first routine from §10 on every question.

Domain mix matches the real exam: 20 Secure · 17 Resilient · 15 High-performing · 13 Cost-optimized, shuffled. Content draws on Days 1–13, weighted toward the second-layer variants in today's lesson. No stem repeats Exams 1 or 2.

**Pacing checkpoints:** Q22 by ~0:45 · Q44 by ~1:30 · Q65 by ~2:00 · last 10 minutes for flagged questions.

**Q1.** A database server on EC2 uses a 2 TiB gp2 volume. Only 300 GiB is used; the volume was sized that large purely to get 6,000 IOPS. What is the MOST cost-effective change that keeps the same performance?
A. Move to an io2 volume of 300 GiB with 6,000 provisioned IOPS
B. Move to an st1 volume
C. Move to a gp3 volume sized for the data, with 6,000 IOPS provisioned
D. Add an instance store volume and RAID it with the gp2 volume

**Q2.** Instances in private subnets reach S3 through a gateway VPC endpoint. The security team is worried that a compromised instance could copy data to an attacker-owned S3 bucket in another account. What should the architect do?
A. Add a bucket policy to the company's buckets that denies access from outside the VPC
B. Attach a VPC endpoint policy that allows access only to the company's approved buckets
C. Remove the gateway endpoint and route S3 traffic through a NAT gateway
D. Enable S3 Block Public Access on the company's buckets

**Q3.** A fulfilment service reads from an SQS standard queue. Processing each message takes up to 3 minutes, and the queue's visibility timeout is 30 seconds. Customers occasionally receive duplicate shipments. What is the MOST likely fix?
A. Switch to long polling with a 20-second wait time
B. Reduce the message retention period
C. Configure a dead-letter queue with `maxReceiveCount` of 1
D. Increase the visibility timeout to more than the maximum processing time and make processing idempotent

**Q4.** A product catalogue on Aurora is fronted by ElastiCache. After a price update, customers sometimes see the old price for several minutes. The business requires that cached prices are never stale after an update. What should the architect implement?
A. Write-through caching: update the cache whenever the database is written
B. Lazy loading with a longer TTL
C. Increase the number of cache nodes
D. Move the cache to DAX

**Q5.** An application on EC2 needs about 40 configuration values, three of which are API keys. None of them needs automatic rotation. The company wants encrypted storage at the lowest cost. What should the architect use?
A. AWS Secrets Manager for all 40 values
B. An S3 bucket with SSE-KMS and a JSON file
C. SSM Parameter Store standard parameters, using SecureString for the API keys
D. AWS AppConfig with a hosted configuration and a Lambda extension

**Q6.** A company runs RDS for PostgreSQL in a Multi-AZ DB instance deployment. It now wants failover to complete faster and wants to use the standby capacity for read queries, without changing database engines. What should the architect recommend?
A. Add a cross-Region read replica
B. Move to a Multi-AZ DB cluster deployment with two readable standbys
C. Migrate to DynamoDB
D. Enable RDS Proxy

**Q7.** An Aurora MySQL cluster has four replicas. Two are large instances intended for heavy reporting; two smaller ones serve the application's reads. Reporting queries currently land on all four replicas via the reader endpoint, slowing the application. What should the architect do?
A. Create a custom endpoint that includes only the two large replicas and point reporting at it
B. Give the large replicas the highest failover priority
C. Use the cluster (writer) endpoint for reporting
D. Create an Aurora Global Database for reporting

**Q8.** A bucket uses SSE-KMS with a customer managed key. A data pipeline writes millions of small objects per hour, and the KMS bill is high with occasional throttling. Requirements for the customer managed key remain. What should the architect do?
A. Enable S3 Bucket Keys on the bucket
B. Switch to SSE-S3
C. Request a KMS quota increase and keep everything else the same
D. Switch to client-side encryption

**Q9.** A company encrypts data in `ap-south-1` with KMS and replicates it to `ap-southeast-1` for DR. In a failover, applications in `ap-southeast-1` must decrypt the data locally without calling the original Region and without re-encrypting it. What should the architect use?
A. A separate customer managed key in each Region with the same alias
B. The AWS managed key `aws/s3` in both Regions
C. CloudHSM clusters in both Regions
D. A KMS multi-Region primary key with a replica key in `ap-southeast-1`

**Q10.** A web tier must handle a peak of 12 instances and must keep handling peak with no scaling delay if any single Availability Zone fails. Which design meets this with the FEWEST instances?
A. 12 instances in one AZ with an Auto Scaling group
B. 12 instances in each of two AZs
C. 6 instances in each of three AZs
D. 4 instances in each of three AZs

**Q11.** A company must ensure that no S3 bucket in a member account can ever be made public, and that account administrators cannot reverse this control. *(Choose two.)*
A. Enable S3 Block Public Access at the account level
B. Enable default encryption on all buckets
C. Use Amazon Macie to alert on public buckets
D. Attach an SCP that denies `s3:PutAccountPublicAccessBlock`
E. Enable S3 Versioning on all buckets

**Q12.** A support team uses a SaaS helpdesk that integrates with Amazon EventBridge. When a high-priority ticket is created, a Lambda function and a Step Functions workflow must start, based on fields in the ticket. What is the simplest design?
A. Poll the SaaS API every minute from an EC2 instance
B. Use the SaaS partner event source in EventBridge with a rule that matches priority and targets Lambda and Step Functions
C. Have the SaaS call an SNS topic over email
D. Stream the SaaS data into Kinesis Data Streams and write a consumer

**Q13.** A company will run m7g instances in `ap-south-1` steadily for three years. It does not expect to change instance family or Region, and wants the largest discount with flexibility to change instance size and OS. What should it buy?
A. Compute Savings Plans
B. Spot Instances
C. On-Demand Capacity Reservations
D. EC2 Instance Savings Plans for the m7g family in `ap-south-1`

**Q14.** A Lambda function consumes an SQS queue in batches of 10. When one message in a batch fails, all 10 are retried, causing duplicate work and delays. What should the architect configure?
A. Enable `ReportBatchItemFailures` on the event source mapping and return only the failed message IDs
B. Set the batch size to 1
C. Disable the dead-letter queue
D. Increase the function's timeout

**Q15.** An internal web application on EC2 behind an ALB has no authentication. The company wants employees to sign in through its OIDC-compliant identity provider before reaching the application, without changing application code. What should the architect do?
A. Add a Lambda authorizer
B. Put API Gateway in front of the ALB
C. Configure an `authenticate-oidc` action on the ALB's HTTPS listener rule
D. Enable AWS WAF with a rate-based rule

**Q16.** A social network needs to find friends-of-friends and detect fraud rings across billions of relationships with low-latency traversals. Which database should the architect choose?
A. Amazon Neptune
B. Amazon DynamoDB with GSIs
C. Amazon Redshift
D. Amazon RDS for MySQL with self-joins

**Q17.** User sessions are stored in an ElastiCache cluster. When a cache node failed last week, all users were logged out. Sessions must survive a node or AZ failure with minimal changes. What should the architect do?
A. Move to ElastiCache for Memcached with more nodes
B. Store sessions on the EC2 instances' instance store
C. Enable sticky sessions on the ALB
D. Use ElastiCache for Valkey (or Redis OSS) with a replica in another AZ and Multi-AZ automatic failover

**Q18.** A public subnet's network ACL allows inbound TCP 443 from 0.0.0.0/0, and the outbound rules allow only TCP 443 to 0.0.0.0/0. The web servers' security group allows inbound 443. Users' HTTPS connections time out. What should the architect change?
A. Add an inbound security group rule for ephemeral ports
B. Add an outbound NACL rule allowing TCP 1024–65535 to 0.0.0.0/0
C. Add an outbound security group rule for port 443
D. Replace the NACL with a second security group

**Q19.** A batch analytics fleet uses Spot Instances across many instance types. It is being interrupted too often. The company wants low prices but fewer interruptions. Which allocation strategy should it use?
A. `lowest-price`
B. `diversified` with only one instance type
C. `price-capacity-optimized`
D. On-Demand for all instances

**Q20.** A genomics tool must download 50 GB objects from S3 to EC2 as fast as possible. Single-stream downloads are slow. What should the architect recommend?
A. Enable S3 Transfer Acceleration
B. Move the objects to S3 Glacier Instant Retrieval
C. Use a larger EBS volume
D. Download each object with parallel byte-range GET requests

**Q21.** A company has hundreds of projects. Engineers must be able to start and stop only the EC2 instances that belong to their own project. New projects appear weekly, and the team doesn't want to write new policies for each one. What should the architect implement?
A. ABAC: tag principals and instances with `project`, and allow actions when `aws:ResourceTag/project` equals `${aws:PrincipalTag/project}`
B. One IAM group per project with its own policy
C. A separate AWS account per project
D. A resource-based policy on each instance

**Q22.** An application runs on EC2 instances with only private IP addresses. The team wants Route 53 failover routing to switch to a secondary Region when the primary application is unhealthy. Route 53 health checks can't reach the private endpoints. What should the architect do?
A. Assign Elastic IPs to every instance
B. Create a CloudWatch alarm on the application's health metric and use a Route 53 health check based on that alarm
C. Use a simple routing policy with two records
D. Use a TCP health check against the private IPs

**Q23.** A company is adding a custom domain (`www.example.com`) with HTTPS to a CloudFront distribution whose origin is an ALB in `ap-south-1`. The certificate was requested in ACM in `ap-south-1`, but it doesn't appear in the CloudFront console. What should the architect do?
A. Import the certificate into IAM
B. Export the ACM certificate and upload it to the ALB
C. Enable SNI on the ALB
D. Request (or import) the certificate in ACM in `us-east-1` and attach it to the distribution

**Q24.** A research institute publishes a 500 TB public dataset in S3. Partner organisations download it often, and the institute wants the downloaders to pay the data transfer charges. What should the architect configure?
A. A CloudFront distribution
B. S3 Requester Pays on the bucket
C. S3 Intelligent-Tiering
D. Presigned URLs with long expiry

**Q25.** After a credential compromise at another company, a CISO requires that backups of production workloads remain recoverable even if the entire production account is compromised. The company uses AWS Organizations and AWS Backup. What should the architect do?
A. Increase backup retention in the production account
B. Enable MFA on the production root user
C. Configure AWS Backup to copy recovery points to a backup vault in a separate account in the organization, and lock that vault with Vault Lock
D. Store backups in a second Region of the same account

**Q26.** A Lambda-backed API sees a predictable surge every weekday at 09:00. During the first minutes, cold starts cause latency SLA breaches. What should the architect do?
A. Increase the function's memory to the maximum
B. Set reserved concurrency to the expected peak
C. Configure provisioned concurrency and use Application Auto Scaling scheduled actions to raise it before 09:00
D. Move the function into a VPC

**Q27.** A third-party monitoring SaaS needs to assume a role in the company's account to read CloudWatch metrics. The vendor serves many customers. How should the architect prevent another of the vendor's customers from tricking the vendor into accessing this account?
A. Give the vendor an IAM user with access keys
B. Require a unique `sts:ExternalId` in the role's trust policy
C. Restrict the role with a permissions boundary
D. Require MFA in the trust policy

**Q28.** A company wants daily snapshots of all EBS volumes tagged `backup=true`, kept for 14 days, with the least effort. It uses no other services that need backup. What should the architect use?
A. Amazon Data Lifecycle Manager policy targeting the tag
B. A cron job on an EC2 instance calling `create-snapshot`
C. S3 lifecycle rules
D. AWS Elastic Disaster Recovery

**Q29.** A sports app must push live score updates to hundreds of thousands of connected browsers and let users send reactions back, with no servers to manage. What should the architect use?
A. API Gateway REST API with polling every second
B. Amazon SQS with one queue per user
C. An NLB in front of EC2 instances running a socket server
D. API Gateway WebSocket API with Lambda integrations

**Q30.** Development and test EC2 instances run 24/7 but are used only 09:00–19:00 on weekdays. What is the MOST cost-effective action with little effort?
A. Stop the instances outside working hours on a schedule (e.g., EventBridge Scheduler or Instance Scheduler on AWS)
B. Buy 3-year Reserved Instances for them
C. Move them to Dedicated Hosts
D. Convert them to Spot Instances that run 24/7

**Q31.** GuardDuty raised a finding that an EC2 instance's IAM credentials were used from an unusual IP address. The security team needs to understand the related activity — which other resources and API calls were involved over the past weeks — without building queries. What should they use?
A. AWS Config timeline
B. CloudTrail event history
C. Amazon Detective
D. Amazon Inspector

**Q32.** A company's RTO is 10 minutes and its RPO is under a minute for a web application with a relational database. It wants the lowest cost that meets these targets. Which DR strategy should it use?
A. Backup and restore
B. Pilot light with compute launched during failover
C. Multi-site active-active
D. Warm standby: a scaled-down copy of the full stack running in the DR Region with continuous database replication

**Q33.** A company with 80 accounts in AWS Organizations must ensure that a standard set of AWS WAF rules is applied to every ALB and CloudFront distribution, including ones created in the future. What should the architect use?
A. AWS Firewall Manager security policies
B. AWS Config rules in each account
C. A CloudFormation template shared with each team
D. Shield Standard

**Q34.** A company runs several MongoDB applications on self-managed EC2 clusters and wants a managed, MongoDB-compatible database with minimal application changes. What should the architect choose?
A. Amazon DynamoDB
B. Amazon DocumentDB
C. Amazon Keyspaces
D. Amazon RDS for PostgreSQL with JSONB

**Q35.** During scale-in events, users of an ALB-fronted application see failed requests from instances that are removed mid-request. What should the architect configure?
A. Cross-zone load balancing
B. A shorter health check interval
C. An appropriate deregistration delay on the target group so in-flight requests complete
D. Sticky sessions

**Q36.** A team notices its CloudWatch Logs storage cost grows every month. Logs are needed for 30 days for troubleshooting; there is no other requirement. What should the architect do?
A. Move logs to CloudTrail
B. Increase the log ingestion batch size
C. Create a metric filter on each log group
D. Set a 30-day retention policy on the log groups

**Q37.** Employees sign in with on-premises Active Directory credentials. The company is deploying Amazon WorkSpaces and wants users to use their existing AD credentials, with no directory data cached or synchronised into AWS. What should the architect use?
A. AWS Managed Microsoft AD with a one-way trust
B. Simple AD
C. Amazon Cognito user pool
D. AD Connector

**Q38.** A Lambda function triggered by an API calls a partner's API, which allows at most 50 concurrent requests. During spikes the partner returns errors and requests are lost. The company needs requests to be kept and processed as fast as the partner allows. *(Choose two.)*
A. Increase the function's memory
B. Send requests to an SQS queue and have the function consume from it
C. Enable provisioned concurrency of 500
D. Use CloudFront in front of the partner API
E. Set reserved concurrency of 50 on the consuming function

**Q39.** A research team must run tens of thousands of containerised simulation jobs with dependencies between them, using Spot capacity where possible, without building its own scheduler. What should the architect use?
A. AWS Batch with a Spot compute environment and job queues
B. A long-running ECS service
C. Lambda functions chained with SNS
D. An EC2 Auto Scaling group with a cron script

**Q40.** A legal team requires that contract documents in S3 cannot be deleted or overwritten for one year. A small group of security administrators must still be able to remove a document in exceptional, approved cases. What should the architect configure?
A. Object Lock in compliance mode with a one-year retention
B. MFA Delete on the bucket
C. Object Lock in governance mode with a one-year retention, granting `s3:BypassGovernanceRetention` only to the administrators
D. A bucket policy denying `s3:DeleteObject` for everyone

**Q41.** A DynamoDB table stores five years of order history. Items older than three months are rarely read, and storage is now about 80% of the table's cost. Which change is MOST cost-effective with no code changes?
A. Switch the table to on-demand capacity
B. Change the table class to DynamoDB Standard-IA
C. Add DAX
D. Create a GSI on the order date

**Q42.** A company is migrating an on-premises Java application that uses Apache ActiveMQ via JMS. It wants a managed broker and no code changes to messaging. What should the architect choose?
A. Amazon MQ for ActiveMQ
B. Amazon SQS FIFO
C. Amazon SNS
D. Amazon Kinesis Data Streams

**Q43.** A software provider wants to offer its API, running behind an NLB in its VPC, privately to hundreds of customer VPCs in other accounts. Many customers use overlapping CIDR ranges. Traffic must not traverse the internet. What should the architect do?
A. VPC peering with each customer
B. Create a VPC endpoint service (AWS PrivateLink) for the NLB; customers create interface endpoints
C. A Transit Gateway shared via RAM
D. Site-to-Site VPN connections to each customer

**Q44.** A mobile game needs a real-time global leaderboard with millions of players, returning a player's rank and the top 100 in milliseconds. What should the architect use?
A. RDS with an `ORDER BY` query and an index
B. Amazon Redshift
C. ElastiCache for Valkey or Redis OSS using sorted sets
D. S3 with Athena

**Q45.** On-premises servers connected through Direct Connect must resolve DNS names in a Route 53 private hosted zone associated with a VPC. What should the architect configure?
A. A Route 53 public hosted zone with the same records
B. A Resolver outbound endpoint and a forwarding rule
C. DHCP options sets pointing to on-prem DNS
D. A Route 53 Resolver inbound endpoint in the VPC, with on-prem DNS servers forwarding the zone to it

**Q46.** An Aurora PostgreSQL cluster runs an I/O-heavy workload. Analysis shows I/O charges make up well over a quarter of the cluster's total cost. What should the architect do to lower cost with predictable pricing?
A. Move to RDS for PostgreSQL Single-AZ
B. Add more Aurora replicas
C. Switch the cluster to the Aurora I/O-Optimized configuration
D. Enable Performance Insights

**Q47.** Lambda functions run in private subnets with no NAT gateway. They must retrieve database credentials from Secrets Manager. What should the architect do?
A. Create an interface VPC endpoint for Secrets Manager in the VPC and enable private DNS
B. Create a gateway VPC endpoint for Secrets Manager
C. Move the functions out of the VPC
D. Store the credentials in environment variables

**Q48.** A SaaS application uses a relational database whose load varies sharply and unpredictably during the day, from near idle to very high. The company wants MySQL compatibility, automatic scaling in fine-grained increments, and no capacity planning. What should the architect choose?
A. RDS for MySQL on the largest instance
B. Aurora provisioned cluster with Auto Scaling replicas only
C. DynamoDB on-demand
D. Aurora Serverless v2

**Q49.** An SQS queue buffers events for a downstream system that is sometimes offline for maintenance for up to 10 days. After a recent 6-day outage, many events had disappeared. The queue uses default settings. What should the architect change?
A. Enable long polling
B. Increase the message retention period to 14 days
C. Add a DLQ with `maxReceiveCount` 5
D. Convert the queue to FIFO

**Q50.** A security audit found that a fleet of 300 EC2 instances is missing many OS security patches. The company needs automated patching on a weekly schedule with compliance reporting. What should the architect use?
A. Amazon Inspector
B. AWS Config managed rules
C. Amazon GuardDuty
D. AWS Systems Manager Patch Manager with patch baselines and a maintenance window

**Q51.** Regulations require EBS snapshots to be retained for seven years. They are almost never restored, and a restore time of a day or more is acceptable. What is the MOST cost-effective option?
A. Move the snapshots to the EBS Snapshots Archive tier
B. Keep them in the standard snapshot tier
C. Copy snapshots to another Region
D. Convert snapshots to AMIs

**Q52.** A content management system on EFS has highly spiky, unpredictable throughput: quiet most of the day, then large bursts. The file system is small, so burst credits run out and the app slows. The team doesn't want to size throughput manually. What should the architect do?
A. Switch to Provisioned throughput at the peak level
B. Switch the file system to Elastic throughput
C. Move to EBS gp3
D. Move to EFS One Zone

**Q53.** A company enabled S3 Cross-Region Replication on an existing bucket with 200 million objects. New objects replicate, but the existing objects do not. What should the architect do?
A. Re-upload all objects manually
B. Enable versioning on the destination bucket
C. Run S3 Batch Replication for the existing objects
D. Enable S3 Replication Time Control

**Q54.** A video-training site gives paying subscribers access to a library of thousands of private video files served through CloudFront. Users shouldn't need a new link for every file. What should the architect use?
A. CloudFront signed URLs for each file
B. CloudFront signed cookies
C. S3 presigned URLs
D. CloudFront geo restriction

**Q55.** Two VPCs in the same Region exchange several terabytes of data per day. No other VPCs or on-prem networks need to connect. What is the MOST cost-effective way to connect them?
A. AWS Transit Gateway
B. Site-to-Site VPN between the VPCs
C. Route traffic over the internet through NAT gateways
D. VPC peering

**Q56.** A company is migrating an on-premises NetApp storage system that serves both Linux (NFS) and Windows (SMB) clients and relies on snapshots and clones. It wants a managed AWS service with the same capabilities. What should the architect choose?
A. Amazon FSx for NetApp ONTAP
B. Amazon EFS
C. FSx for Windows File Server
D. S3 File Gateway

**Q57.** Compliance requires that TLS traffic be decrypted only on the application instances — not on any load balancer — while still distributing traffic across instances in multiple AZs. What should the architect use?
A. An ALB with an HTTPS listener and HTTPS target groups
B. CloudFront with origin protocol policy HTTPS only
C. An NLB with a TCP listener on port 443 forwarding to the instances
D. An ALB with an HTTP listener

**Q58.** A team deploys an ECS service on Fargate. A recent bad image failed health checks repeatedly, and the deployment kept trying for hours until an engineer intervened. The team wants failed deployments to roll back automatically. What should it enable?
A. The ECS deployment circuit breaker with rollback
B. A larger desired count
C. Fargate Spot
D. ECS Service Connect

**Q59.** A company must process 3 million images in S3 with a Lambda function, in parallel, coordinating retries and tracking failures, without writing a custom fan-out framework. What should the architect use?
A. One Lambda function that loops over all objects
B. AWS Step Functions with a Distributed Map state over the S3 objects
C. S3 event notifications on the existing objects
D. An SQS FIFO queue with one message group

**Q60.** An EC2 instance's data volume was created unencrypted. Policy now requires it to be encrypted with a customer managed KMS key. What should the architect do?
A. Enable EBS encryption by default in the Region
B. Modify the volume and set Encrypted to true
C. Snapshot the volume, create an encrypted volume from the snapshot with the customer managed key, and swap it in
D. Enable encryption on the instance's root device only

**Q61.** Hundreds of workers poll an SQS queue continuously. Most `ReceiveMessage` calls return no messages, and the SQS request bill is high. What should the architect do?
A. Enable long polling by setting `ReceiveMessageWaitTimeSeconds` (up to 20 seconds)
B. Convert to a FIFO queue
C. Reduce the visibility timeout
D. Add a DLQ

**Q62.** A developer ran a faulty `UPDATE` on an Aurora MySQL cluster (created with Backtrack enabled) 20 minutes ago. The team wants to return the database to its prior state as quickly as possible, keeping the same cluster endpoint. What should the architect do?
A. Restore from the latest automated snapshot
B. Backtrack the cluster to a time just before the faulty statement
C. Perform a point-in-time restore to a new cluster
D. Promote an Aurora replica

**Q63.** An IoT platform ingests telemetry into Kinesis Data Streams. Volume swings unpredictably by 10× during the day, and the team keeps getting throttled while resharding. It wants to stop managing capacity. What should the architect do?
A. Add more consumers using enhanced fan-out
B. Replace the stream with SQS standard
C. Switch the stream to on-demand capacity mode
D. Increase the retention period

**Q64.** A company accepts file uploads from customers into an S3 bucket. Security wants every new object scanned for malware automatically, with results available to downstream processing, and no agents or servers to manage. What should the architect use?
A. Amazon Macie
B. Amazon Inspector
C. AWS Config rules
D. Amazon GuardDuty Malware Protection for S3

**Q65.** Audit evidence files must be kept for five years. They are read about once a quarter, and when they are needed they must be available in milliseconds. What is the MOST cost-effective storage class?
A. S3 Standard
B. S3 Glacier Deep Archive
C. S3 Glacier Instant Retrieval
D. S3 One Zone-IA

<details>
<summary><strong>Answer key — open only after all 65 are answered</strong></summary>

Each line gives the answer, the domain and source day, and why it wins.

1. **C** · *Cost-optimized · Day 2/13* — gp3 provisions IOPS independently of size, so a right-sized volume keeps 6,000 IOPS for less. io2 costs more per IOPS; st1 is HDD.
2. **B** · *Secure · Day 4/13* — The endpoint policy controls which buckets traffic through the endpoint can reach, including other accounts' buckets. Your bucket policies can't restrict access to someone else's bucket.
3. **D** · *Resilient · Day 8/13* — The message becomes visible again after 30 s while still being processed, so a second worker takes it. Visibility timeout must exceed processing time; idempotency covers the remaining at-least-once cases.
4. **A** · *High-performing · Day 6/13* — Write-through updates the cache on every write, so reads never see an old value. A longer TTL makes staleness worse; DAX is DynamoDB-only.
5. **C** · *Secure · Day 10/13* — Parameter Store standard parameters are free to store, and SecureString encrypts with KMS. Secrets Manager charges per secret and its rotation isn't needed.
6. **B** · *Resilient · Day 6/13* — A Multi-AZ DB cluster has two readable standbys and typically fails over faster than a Multi-AZ DB instance, on the same engine. A cross-Region replica is for DR, not faster local failover.
7. **A** · *High-performing · Day 6/13* — A custom endpoint load-balances only across the replicas you choose. Failover priority doesn't route queries; the writer endpoint would put reporting on the primary.
8. **A** · *Cost-optimized · Day 10/13* — Bucket Keys sharply cut S3-to-KMS requests while keeping the customer managed key. SSE-S3 drops the CMK requirement.
9. **D** · *Secure · Day 10/13* — Multi-Region keys share key material and key ID, so ciphertext from one Region decrypts in the other locally. Separate keys with the same alias are different keys.
10. **C** · *Resilient · Day 3/13* — With three AZs, losing one leaves two that must hold 12 → 6 each → 18 total. Two AZs needs 24; D leaves only 8 after an AZ loss.
11. **A, D** · *Secure · Day 5/13* — Account-level Block Public Access overrides bucket settings; the SCP stops anyone in the account, including admins and root, from turning it off. Macie only detects.
12. **B** · *High-performing · Day 8/13* — EventBridge partner event sources deliver SaaS events onto a bus; a rule matches on content and targets both services. Polling adds servers and latency.
13. **D** · *Cost-optimized · Day 2/12* — EC2 Instance Savings Plans give the larger discount for one family in one Region, with size and OS flexibility. Compute SP is more flexible but discounts less.
14. **A** · *Resilient · Day 8/13* — Partial batch responses delete successful messages and retry only the failed ones. Batch size 1 works but sacrifices throughput and cost efficiency.
15. **C** · *Secure · Day 3/13* — ALB listener rules can authenticate users with any OIDC IdP (or Cognito) before forwarding, with no app changes. Lambda authorizers belong to API Gateway.
16. **A** · *High-performing · Day 13* — Neptune is a graph database built for relationship traversals. Relational self-joins and key-value lookups degrade with traversal depth.
17. **D** · *Resilient · Day 6/13* — Valkey/Redis OSS replication with Multi-AZ failover promotes a replica when a node fails. Memcached doesn't replicate; sticky sessions don't protect data.
18. **B** · *Secure · Day 4/13* — NACLs are stateless; responses go to the client's ephemeral port, which outbound 443 doesn't allow. Security groups are stateful and already fine.
19. **C** · *Cost-optimized · Day 2/13* — `price-capacity-optimized` picks pools with the most spare capacity and low prices, cutting interruptions. `lowest-price` chases pools more likely to be reclaimed.
20. **D** · *High-performing · Day 5/13* — Parallel byte-range GETs use many connections to S3 at once. Transfer Acceleration helps long-distance transfers over the internet, not in-Region EC2 downloads.
21. **A** · *Secure · Day 1/13* — ABAC uses one policy that compares principal and resource tags, so new projects need only tags. Per-project groups or accounts grow with every project.
22. **B** · *Resilient · Day 9/13* — Route 53 health checkers run on the internet; for private resources, health checks can monitor a CloudWatch alarm instead.
23. **D** · *Secure · Day 9/13* — CloudFront uses ACM certificates only from `us-east-1`. The ALB's own certificate stays in `ap-south-1`.
24. **B** · *Cost-optimized · Day 5/13* — Requester Pays makes the requester's AWS account pay request and data transfer charges. CloudFront still bills the bucket owner.
25. **C** · *Resilient · Day 11/13* — A cross-account copy puts recovery points outside the blast radius of the production account; Vault Lock stops deletion. Same-account copies fall with the account.
26. **C** · *High-performing · Day 8/13* — Provisioned concurrency keeps environments initialised; scheduled scaling raises it before the known spike. Reserved concurrency caps, it doesn't pre-warm.
27. **B** · *Secure · Day 1/13* — The external ID, unique per customer, blocks the confused-deputy problem for third-party role assumption. Access keys are long-lived and worse.
28. **A** · *Resilient · Day 2/13* — Data Lifecycle Manager creates and expires tag-targeted EBS snapshots on a schedule with no code. Cron on EC2 adds a server to run.
29. **D** · *High-performing · Day 8/13* — WebSocket APIs keep two-way connections open and push messages through the `@connections` API. Polling a REST API is wasteful and slow.
30. **A** · *Cost-optimized · Day 2/13* — Running ~50 of 168 hours a week saves roughly 70% of compute. RIs commit you to paying for all hours; Spot can be interrupted and still runs 24/7.
31. **C** · *Secure · Day 13* — Detective builds a behaviour graph from CloudTrail, VPC Flow Logs and GuardDuty findings for investigation. Event history means manual searching.
32. **D** · *Resilient · Day 11/13* — Minutes of RTO need compute already running; warm standby scales up a live copy. Pilot light must launch compute first; active-active costs more than needed.
33. **A** · *Secure · Day 13* — Firewall Manager centrally applies WAF policies across accounts in the organization, including to new resources. Per-account Config rules only detect.
34. **B** · *High-performing · Day 13* — DocumentDB is MongoDB-compatible, so apps keep their drivers and queries. DynamoDB and Keyspaces need rewrites.
35. **C** · *Resilient · Day 3/13* — The deregistration delay lets in-flight requests finish before a target is removed. Health-check tuning doesn't affect draining.
36. **D** · *Cost-optimized · Day 11/13* — Log groups default to never expire; a 30-day retention deletes older events automatically.
37. **D** · *Secure · Day 13* — AD Connector proxies authentication to on-prem AD without caching or syncing directory data. Managed Microsoft AD is a separate directory in AWS.
38. **B, E** · *Resilient · Day 8/13* — SQS durably buffers requests; reserved concurrency of 50 caps the consumer at the partner's limit. Provisioned concurrency of 500 would make the overload worse.
39. **A** · *High-performing · Day 13* — AWS Batch manages job queues, dependencies and Spot compute environments. Everything else needs a hand-built scheduler.
40. **C** · *Secure · Day 5/13* — Governance mode blocks deletes but lets explicitly permitted principals bypass. Compliance mode can't be bypassed by anyone, so the exception would be impossible.
41. **B** · *Cost-optimized · Day 6/13* — Standard-IA lowers storage price for tables where storage dominates cost, with no code changes. On-demand changes throughput pricing, not storage.
42. **A** · *Resilient · Day 8/13* — Amazon MQ runs ActiveMQ with JMS, so code stays the same. SQS and SNS require their own APIs.
43. **B** · *Secure · Day 4/13* — PrivateLink exposes one service one-way through interface endpoints; overlapping CIDRs don't matter. Peering and TGW need unique CIDRs.
44. **C** · *High-performing · Day 13* — Sorted sets maintain rank in memory with O(log N) updates and fast range queries. SQL `ORDER BY` across millions per request is slow.
45. **D** · *Resilient · Day 4/13* — Inbound endpoints let on-prem resolvers forward queries into the VPC's Route 53 Resolver. Outbound is the opposite direction.
46. **C** · *Cost-optimized · Day 13* — I/O-Optimized removes per-I/O charges (higher instance and storage price) — AWS suggests it when I/O is more than ~25% of Aurora spend.
47. **A** · *Secure · Day 4/10* — An interface endpoint gives private access to Secrets Manager without NAT. Gateway endpoints exist only for S3 and DynamoDB; environment variables expose the secret.
48. **D** · *High-performing · Day 6/13* — Aurora Serverless v2 scales capacity in fine increments within seconds and is MySQL-compatible. Largest-instance RDS overpays at idle.
49. **B** · *Resilient · Day 8/13* — Default retention is 4 days, so a 6-day outage deletes messages; the maximum is 14 days. DLQs handle failures, not expiry.
50. **D** · *Secure · Day 13* — Patch Manager applies patch baselines on maintenance-window schedules and reports compliance. Inspector finds vulnerabilities but doesn't patch.
51. **A** · *Cost-optimized · Day 13* — Snapshots Archive is a cheaper tier for long-retention, rarely restored snapshots with a longer restore time. Standard tier costs more per GB-month.
52. **B** · *High-performing · Day 5/13* — Elastic throughput scales automatically with the workload's demand, with no credits or manual sizing. Provisioned at peak pays for peak all day.
53. **C** · *Resilient · Day 5/13* — Replication rules apply only to new objects; Batch Replication replicates existing ones. RTC is an SLA for new objects.
54. **B** · *Secure · Day 9/13* — Signed cookies grant access to many files under a path with one credential. Signed URLs are per file.
55. **D** · *Cost-optimized · Day 4/13* — Peering has no hourly or per-GB processing fee; you pay only data transfer. Transit Gateway adds attachment-hour and per-GB processing charges.
56. **A** · *High-performing · Day 5/13* — FSx for NetApp ONTAP supports NFS and SMB with ONTAP snapshots and clones. FSx Windows is SMB only; EFS is NFS only.
57. **C** · *Secure · Day 3/13* — An NLB TCP listener passes TLS through untouched, so only instances decrypt. An ALB always terminates TLS, even when re-encrypting.
58. **A** · *Resilient · Day 9/13* — The circuit breaker detects deployments that can't reach steady state and rolls back to the last working one.
59. **B** · *High-performing · Day 8/13* — Distributed Map iterates over S3 objects with many parallel child executions, retries and failure tracking built in. A single Lambda is capped at 15 minutes.
60. **C** · *Secure · Day 2/13* — Encryption can't be turned on for an existing volume; create an encrypted volume from a snapshot. Encryption by default only affects new volumes.
61. **A** · *Cost-optimized · Day 8/13* — Long polling waits for messages instead of returning empty, cutting the number of billed requests.
62. **B** · *Resilient · Day 13* — Backtrack rewinds the same Aurora MySQL cluster in minutes, keeping its endpoint. PITR and snapshot restores create new clusters.
63. **C** · *High-performing · Day 12/13* — On-demand mode scales stream capacity automatically. SQS loses ordering and replay; fan-out helps reads, not writes.
64. **D** · *Secure · Day 13* — GuardDuty Malware Protection for S3 scans new objects and can tag results for downstream processing. Macie finds sensitive data, not malware.
65. **C** · *Cost-optimized · Day 5/13* — Glacier Instant Retrieval suits quarterly access with millisecond retrieval at lower storage cost than Standard-IA. Deep Archive takes hours.

</details>

---

## 15. Scoring guide

### Step 1 — raw score

Count correct answers (a *Choose two* question scores only if both picks are right).

| Raw score | % | Read it as |
| --- | --- | --- |
| 56–65 | 86%+ | Exam-ready on this material |
| 52–55 | 80–85% | Meets the go/no-go bar for this exam |
| 46–51 | 70–79% | Borderline |
| < 46 | < 70% | Below the README bar |

Scaled scoring (720/1000) isn't a straight percentage. 80% on practice exams is the safety margin, not a conversion.

### Step 2 — score by domain

| Domain (exam weight) | Questions | Count | Your score | 80% bar |
| --- | --- | --- | --- | --- |
| Secure (30%) | 2, 5, 9, 11, 15, 18, 21, 23, 27, 31, 33, 37, 40, 43, 47, 50, 54, 57, 60, 64 | 20 | __ / 20 | 16 |
| Resilient (26%) | 3, 6, 10, 14, 17, 22, 25, 28, 32, 35, 38, 42, 45, 49, 53, 58, 62 | 17 | __ / 17 | 14 |
| High-performing (24%) | 4, 7, 12, 16, 20, 26, 29, 34, 39, 44, 48, 52, 56, 59, 63 | 15 | __ / 15 | 12 |
| Cost-optimized (20%) | 1, 8, 13, 19, 24, 30, 36, 41, 46, 51, 55, 61, 65 | 13 | __ / 13 | 11 |

### Step 3 — tag and log every miss

For each miss *and* lucky guess, add a line to `miss-log.md` with the failure type from §1:

```markdown
| 13 | Backtrack vs PITR | C (PITR) | B (Backtrack) | "Same endpoint, fastest, Aurora MySQL" = Backtrack; PITR makes a new cluster | K |
```

Add the score line at the top: `Exam 3 (Day 13): __/65 — S __/20 · R __/17 · P __/15 · C __/13`.

### Step 4 — map weak spots to tomorrow's cheat sheet

| Weak domain | Re-read tonight (≤ 20 min) | Tomorrow |
| --- | --- | --- |
| Secure | §§5–6 above; Day 10 KMS + security-service split | Day 14 cheat sheet: Days 1, 4, 10 tables |
| Resilient | §§3, 4, 7 above; Day 11 RTO/RPO table | Day 14: Days 3, 6, 8, 11 tables |
| High-performing | §§3, 4, 8 above; Day 9 edge services | Day 14: Days 5, 8, 9, 12 tables |
| Cost-optimized | §9 above; Day 12 cost tools | Day 14: Days 2, 5, 12 tables |

---

## 16. Go/no-go for 2026-11-13

The README rule: **book (keep) 2026-11-13 only if you scored 80%+ on two full timed practice exams by today.** Your three data points are Exam 1 (Day 7, Days 1–6 content only), Exam 2 (Day 12) and Exam 3 (today). Exams 2 and 3 cover the whole syllabus, so weigh them most.

| Your results | Decision | What to do |
| --- | --- | --- |
| **Exam 2 ≥ 80% and Exam 3 ≥ 80%** | **GO** | Sit the exam on 2026-11-13. Day 14 is the cheat sheet and rest — no new material, no more full exams. |
| One of Exams 2/3 ≥ 80%, the other 70–79%, **Exam 1 ≥ 80%**, and no domain below 70% on Exam 3 | **GO, with care** — two 80%+ exams meets the rule, but Exam 1 covered only Days 1–6, so it's weaker evidence | Keep the date. Spend Day 14's review time on the weakest domain's tables first. |
| Only one exam ≥ 80% in total, the others 70–79% | **NO-GO (short delay)** | Reschedule by about 3–7 days. Take one more full practice exam (e.g., the free official AWS Skill Builder practice set or a bank exam) and sit the real one after a second 80%+. |
| Exam 3 < 70%, or any domain < 60% on both Exams 2 and 3 | **NO-GO (one week)** | Per the README, push back a week. Re-do that domain's day lessons and decision tables; take two more full practice exams. |

**Domain floor:** even with two 80%+ totals, a single domain under 70% on *both* recent exams is a risk — that domain is 20–30% of the real exam. Keep the date, but make that domain's tables the first thing you review on Day 14.

**Trend check:** rising scores (Exam 1 → 2 → 3) on similar difficulty are a good sign; a drop on Exam 3 usually points to Q/C/O misses (reading), not missing knowledge — check your §1 tags before deciding.

**Rescheduling logistics:** if you need to move the date, do it **today or first thing tomorrow**. AWS exams booked through Pearson VUE have a cutoff before the appointment after which you can't reschedule or cancel without losing the fee — check the current AWS Certification rescheduling policy in your certification account. Don't leave this until the evening of Day 14.

Write the decision in one line at the top of `miss-log.md`: `Go/no-go (Day 13): GO / NO-GO — reason`.

---

## 17. Flashcards (cover the right column)

| Prompt | Answer |
| --- | --- |
| Four miss types | Knowledge · Qualifier · Constraint · Overthinking |
| Stop exfiltration via the S3 gateway endpoint | Endpoint policy with allowed buckets |
| Gateway endpoint reachable from on-prem/peered VPC? | No — use an interface endpoint |
| On-prem resolves private hosted zone | Resolver **inbound** endpoint |
| VPC resolves on-prem names | Resolver **outbound** endpoint + rule |
| Readable standbys + faster RDS failover | Multi-AZ DB **cluster** |
| Send reporting to chosen Aurora replicas | Custom endpoint |
| Rewind Aurora MySQL in place | Backtrack |
| PITR result | A **new** DB instance/cluster |
| Cache never stale after writes | Write-through |
| Cache must survive node failure | Valkey/Redis OSS + Multi-AZ failover |
| Leaderboard | Sorted sets |
| MongoDB-compatible | DocumentDB |
| Graph | Neptune |
| Messages processed twice | Raise visibility timeout |
| Default / max SQS retention | 4 days / 14 days |
| Cut empty receives | Long polling (≤ 20 s) |
| One bad message fails Lambda batch | `ReportBatchItemFailures` |
| Cap Lambda at N concurrent | Reserved concurrency |
| Remove cold starts at known time | Provisioned concurrency + scheduled scaling |
| JMS / AMQP app, no code change | Amazon MQ |
| SaaS events + content routing + replay | EventBridge |
| Parallel processing of millions of S3 objects | Step Functions Distributed Map |
| Two-way real-time browser messaging | API Gateway WebSocket API |
| No-rotation secrets, cheapest | Parameter Store SecureString |
| Same KMS key in two Regions | Multi-Region keys |
| Cut SSE-KMS request cost | S3 Bucket Keys |
| Third-party role assumption guard | `sts:ExternalId` |
| ACM cert Region for CloudFront | `us-east-1` |
| TLS only on instances | NLB TCP passthrough |
| NACL outbound rule for web replies | TCP 1024–65535 |
| Tag-matched permissions | ABAC |
| SSO in front of ALB app | `authenticate-oidc` action |
| Bypassable retention | Object Lock governance |
| AD auth, nothing stored in AWS | AD Connector |
| Org-wide WAF rules | Firewall Manager |
| Investigate GuardDuty finding | Detective |
| Patch OS on schedule | SSM Patch Manager |
| Many private files per user via CloudFront | Signed cookies |
| 3 AZs, survive one, peak P | Provision 1.5 × P |
| Health-check private resource in Route 53 | CloudWatch alarm-based health check |
| Replicate existing S3 objects | Batch Replication |
| RTO minutes, RPO seconds, cheapest | Warm standby |
| Spot with fewer interruptions | `price-capacity-optimized` |
| Downloaders pay transfer | Requester Pays |
| Storage-heavy, rarely-read DynamoDB | Standard-IA table class |
| Aurora I/O > ~25% of spend | I/O-Optimized |
| Quarterly reads, ms retrieval | Glacier Instant Retrieval |

## 18. Before you close today

- [ ] (If you have them) Exam 1 and Exam 2 score lines and misses added to `miss-log.md`
- [ ] Sections 1–11 read (focus: the variant and knob tables in §§2–9, §10 drill)
- [ ] Lab done and torn down (SQS queues, Object Lock bucket emptied with bypass and deleted, local JSON files removed)
- [ ] Warm-up: 10 gap questions
- [ ] Practice Exam 3 taken in one 130-minute sitting, **then** scored by domain
- [ ] Every miss + lucky guess logged with its K/Q/C/O tag, plus the `Exam 3 (Day 13)` score line
- [ ] **Go/no-go decided** and written at the top of `miss-log.md`; rescheduled today if it's a no-go
- [ ] Bed on time — sleep consolidates more than one extra hour of reading

**Tomorrow (Day 14):** one-page cheat sheet of every decision table from Days 1–13, exam-day logistics, and rest — no new material.
