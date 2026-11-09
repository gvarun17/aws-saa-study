# Day 11 — Disaster Recovery, AWS Backup, Migration, CloudWatch / CloudTrail / Config

**Domain weight:** mostly *Design Resilient Architectures* (26% — "RPO of 1 hour", "recover in another Region", "minimal downtime migration"). *Secure* (30%) via CloudTrail, Config, Backup Vault Lock and audit trails; *Cost-optimized* (20%) because every DR question is really "the cheapest strategy that still meets RPO/RTO"; *High-performing* (24%) via CloudWatch metrics and alarms that drive scaling.
**Time plan:** Learn 90 min · Lab 30 min · Drill 45 min · Review 30 min.

By the end of today you should be able to:

- Translate an RPO/RTO sentence into the cheapest of the four DR strategies, and name the AWS services that implement it.
- Design a centralised, immutable, cross-Region/cross-account backup setup with AWS Backup.
- Pick the right migration tool from one sentence: MGN, DMS (+ Schema Conversion), DataSync, Transfer Family, Storage Gateway, Snowball, Direct Connect.
- Do the "network vs truck" transfer-time math in your head.
- Tell CloudWatch, CloudTrail and Config apart instantly, and build the standard "detect → alert → remediate" pipelines with them.

---

## 1. RPO and RTO — the two numbers every DR question hangs on

```
          last good           disaster              service
          recovery point      strikes               restored
               │                 │                      │
  ─────────────●─────────────────✖──────────────────────●──────────► time
               └──── RPO ────────┘└──────── RTO ─────────┘
           data you can afford      downtime you can afford
               to lose
```

- **RPO (Recovery Point Objective):** maximum acceptable **data loss**, measured in time. Driven by *how often you copy data* (backup frequency, replication lag).
- **RTO (Recovery Time Objective):** maximum acceptable **downtime**. Driven by *how much is already running* in the recovery site and how automated failover is.

Lower RPO and RTO cost more, always. The exam gives you the numbers and a qualifier (usually *most cost-effective*), and you pick the **cheapest strategy that still meets both**.

> Trap: "Backups every 24 hours" can never meet an RPO of 1 hour, no matter how fast the restore is. Check RPO and RTO separately.

> Scope check: Multi-AZ (Day 1, Day 6) protects against **AZ failure** and is HA, not DR. DR on SAA usually means **Region failure**, corruption, or accidental/malicious deletion. Multi-AZ replicates a `DROP TABLE` instantly — only backups/point-in-time restore save you there.

---

## 2. The four DR strategies

```
 cost / complexity ───────────────────────────────────────────────────────►
 RPO/RTO       ◄─── hours ─────────── tens of min ─────── minutes ─────── ~0

 ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  ┌──────────────────┐
 │ BACKUP &     │  │ PILOT LIGHT  │  │ WARM STANDBY     │  │ MULTI-SITE       │
 │ RESTORE      │  │              │  │                  │  │ ACTIVE/ACTIVE    │
 │ data copied  │  │ data live-   │  │ full stack       │  │ full stack in    │
 │ to DR Region │  │ replicated;  │  │ running at small │  │ ≥2 Regions, all  │
 │ nothing      │  │ core (DB) on;│  │ scale; scale out │  │ serving traffic  │
 │ running      │  │ app servers  │  │ on failover      │  │                  │
 │              │  │ OFF (AMIs/IaC│  │                  │  │                  │
 │              │  │ ready)       │  │                  │  │                  │
 └──────────────┘  └──────────────┘  └──────────────────┘  └──────────────────┘
```

| Strategy | What runs in DR Region | Typical RPO | Typical RTO | Cost | AWS building blocks |
| --- | --- | --- | --- | --- | --- |
| **Backup & restore** | Nothing but stored backups | Hours (backup interval) | Hours (24h or less) | $ | AWS Backup cross-Region copy, EBS/RDS snapshot copy, S3 CRR, AMIs copied, CloudFormation/Terraform to rebuild |
| **Pilot light** | Data layer **live** (replica DB), core infra; compute **off** or zero | Seconds–minutes (replication) | Tens of minutes (launch + scale compute) | $$ | RDS cross-Region read replica / Aurora Global Database, DynamoDB global tables, S3 CRR, AMIs, ASG with desired = 0 |
| **Warm standby** | Complete, **functional** scaled-down copy taking no (or test) traffic | Seconds–minutes | Minutes (scale up, flip DNS) | $$$ | Same as pilot light + small ASG running, ALB in place, Route 53 failover |
| **Multi-site active/active** | Full production capacity, serving live traffic | Near zero | Near zero (seconds) | $$$$ | Aurora Global Database / DynamoDB global tables, Route 53 latency/weighted, Global Accelerator, ARC |

**How to tell pilot light from warm standby** (the most common confusion):
- *Pilot light:* the app tier **cannot serve requests** until you start it. "Keep the database replicated; launch servers only when needed."
- *Warm standby:* the app tier **could serve requests right now**, just at reduced capacity. "A minimum-size copy of the full environment is always running."

### Failover mechanics

- **Route 53 failover routing** + health checks (Day 9): primary record with health check, secondary record pointing at DR. Health checks can watch an endpoint or a CloudWatch alarm.
- **Amazon Application Recovery Controller (ARC):** readiness checks (is DR scaled and configured like prod?) and **routing controls** — highly available on/off switches for failover that don't depend on the impaired Region's control plane. ARC also offers **zonal shift** for moving traffic away from a bad AZ. Pick it for "reliable, controlled Region failover for a critical app".
- **Global Accelerator** (Day 9): static anycast IPs, shifts traffic between Regional endpoints in seconds, no DNS caching problem.
- **Data-layer promotion:**
  - RDS cross-Region read replica → **promote** (manual; replica becomes standalone, replication stops).
  - Aurora Global Database → **managed failover / switchover** to a secondary Region (typical RPO ~1 s, RTO around a minute; check current docs).
  - DynamoDB global tables → already multi-active; just route traffic.
  - S3 CRR → objects already there; **S3 Replication Time Control** gives an SLA-backed 15-minute replication target (99.99% of objects) for "predictable RPO for S3".

### AWS Elastic Disaster Recovery (DRS)

The managed answer for **servers** (on-premises, other clouds, or EC2 in another Region) that need low RPO/RTO **without re-architecting**.

```
 Source servers (on-prem / EC2 Region A)        DR Region (staging area subnet)
 ┌───────────────────────┐   continuous,        ┌──────────────────────────────┐
 │ AWS Replication Agent │── block-level ──────►│ lightweight replication      │
 │ (installed on each)   │   replication        │ servers + cheap EBS volumes  │
 └───────────────────────┘                      └──────────────┬───────────────┘
                                                  drill/recover │ launch
                                                               ▼
                                                 full-size recovery EC2 instances
```

- RPO: **seconds** (continuous replication). RTO: **minutes** (launch from latest or a point-in-time snapshot).
- Cost stays low because only a thin staging area runs until you recover — effectively a managed pilot light for servers.
- Supports non-disruptive **recovery drills** and **failback**.
- Exam cue: *"replicate on-premises servers to AWS for DR with RPO in seconds and RTO in minutes, minimal infrastructure"* → **Elastic Disaster Recovery**. (Same agent-based tech as MGN; DRS is for DR, MGN is for one-way migration.)

---

## 3. AWS Backup — centralised, policy-driven backups

Before AWS Backup, each service had its own snapshot/backup mechanism and schedule. AWS Backup puts one policy layer on top.

```
 Backup plan  ──(rules: schedule, window, lifecycle, copy-to)──►  Backup vault
     │                                                           (encrypted w/ KMS,
     ▼                                                            access policy,
 Resource assignment (by tag e.g. backup=daily, or ARNs)          optional Vault Lock)
     │                                                                 │
     ▼                                                                 ├──► copy to vault in
 EC2 · EBS · RDS · Aurora · DynamoDB · EFS · FSx · S3 ·                │    another Region
 Storage Gateway volumes · DocumentDB · Neptune · Redshift · …         └──► copy to vault in
 (check current docs for the full list)                                     another account
```

Key features and why they're on the exam:

| Feature | What it gives you | Exam cue |
| --- | --- | --- |
| **Backup plans + tag-based assignment** | One policy for many services; new resources tagged `backup=daily` are covered automatically | "Centralise and automate backups across services with least effort" |
| **Lifecycle** | Move recovery points to cold storage after N days, delete after M (cold storage has a minimum retention; check current docs) | "Keep backups 7 years cheaply" |
| **Cross-Region copy** | Copy recovery points to a vault in another Region | Backup & restore DR |
| **Cross-account copy** (with Organizations) | Copy to a vault in a separate backup/log-archive account | "Protect backups if the production account is compromised" |
| **Backup policies in Organizations** | Push backup plans to all member accounts from the management/delegated admin account | "Enforce backups across 50 accounts" |
| **Vault Lock** — governance mode | Retention rules that privileged users with special permissions can still change | Testing, softer control |
| **Vault Lock** — compliance mode | WORM: after the cooling-off period (minimum 72 h) **nobody — not even root or AWS — can delete recovery points early or remove the lock** | "Immutable backups", "ransomware", "regulators require backups cannot be deleted" |
| **Logically air-gapped vault** | Vault that is locked by default and can be shared to a recovery account via RAM | "Isolated recovery even if accounts are compromised" |
| **Continuous backup / PITR** | Point-in-time restore for supported services (e.g. RDS, S3, DynamoDB via its own PITR) | "Restore to any second in the last N days" |
| **Backup Audit Manager** | Frameworks/reports proving backups meet policy | "Prove to auditors backups comply" |
| **Restore testing** | Scheduled automated test restores | "Validate backups are restorable" |

Service-native features you should still recognise:

| Service | Native mechanism |
| --- | --- |
| EBS | Snapshots (incremental, stored in S3, Regional — copy for DR); **Data Lifecycle Manager** for EBS-only scheduling; Fast Snapshot Restore to avoid first-read latency |
| RDS | Automated backups (PITR, retention up to 35 days) + manual snapshots (kept until deleted); cross-Region automated backup replication |
| Aurora | Continuous backup to S3, PITR, **Backtrack** (MySQL-compatible — rewind in place, no restore) |
| DynamoDB | On-demand backups + **PITR** (any second, up to 35 days) |
| S3 | **Versioning** + CRR/SRR + **Object Lock** (compliance/governance) |
| EFS | AWS Backup (automatic backups on by default for new file systems via console) + EFS replication to another Region |

> Trap: "Automated RDS backups" are deleted when you delete the instance (unless you retain them). Manual snapshots stay. For long-term retention use manual snapshots or AWS Backup.

> Trap: AWS Backup Vault Lock vs S3 Object Lock — both WORM. Vault Lock protects **AWS Backup recovery points** across services; Object Lock protects **S3 objects**.

---

## 4. Migration — strategy first, then tools

### The 7 Rs (know what each means; the exam wants the verb)

| R | Meaning | Typical AWS answer |
| --- | --- | --- |
| **Retire** | Turn it off | — |
| **Retain** | Leave it on-prem for now | — |
| **Rehost** ("lift and shift") | Same server, now on EC2 | **AWS Application Migration Service (MGN)** |
| **Relocate** | Move a whole hypervisor estate unchanged | VMware Cloud on AWS (or similar) |
| **Replatform** ("lift, tinker, shift") | Small optimisations, no code rewrite | Self-managed DB → **RDS**; app → Elastic Beanstalk/containers |
| **Repurchase** | Move to SaaS | e.g. CRM → SaaS product |
| **Refactor / re-architect** | Rewrite cloud-native | Lambda, DynamoDB, microservices |

### Discover and plan

- **AWS Application Discovery Service:** inventory on-prem servers, utilisation and **dependencies** — agentless (VMware vCenter collector) or agent-based (deeper data: processes, network connections).
- **AWS Migration Hub** has historically been the single place to track migrations, and **AWS Transform** is AWS's newer agentic service for migration/modernisation planning. Service availability here has been changing — check current docs; SAA questions still reference Migration Hub as "track migration progress in one place".
- **Migration Evaluator:** build a business case / TCO from on-prem data.

### Servers: AWS Application Migration Service (MGN)

```
 On-prem / other cloud VM             AWS (target Region)
 ┌────────────────────┐   continuous   ┌───────────────────────┐      ┌──────────────┐
 │ Replication Agent  │── block-level ─►│ staging area: small   │─────►│ test launch  │
 └────────────────────┘   replication   │ replication servers + │      │ → cutover    │
                                        │ EBS                   │      │ (EC2)        │
                                        └───────────────────────┘      └──────────────┘
```

- Rehost with **minimal downtime**: replicate continuously, launch test instances non-disruptively, then cut over in minutes.
- Converts the boot volume to run natively on EC2. Successor to CloudEndure Migration and the retired Server Migration Service (SMS) — if you see SMS in an answer, it's legacy.
- Exam cue: *"migrate hundreds of VMware/physical servers to EC2 as-is with minimal downtime"* → **MGN**.

### Databases: AWS DMS + Schema Conversion

```
 Source DB ──► DMS replication instance (or DMS Serverless) ──► Target DB
 (on-prem,       │  1. full load                                  (RDS, Aurora,
  EC2, RDS,      │  2. CDC: ongoing changes from logs             Redshift,
  Oracle,        │     (keeps target in sync until cutover)       DynamoDB, S3,
  SQL Server…)   └────────────────────────────────────────────     OpenSearch, Kinesis…)
```

| Migration type | Example | Tools |
| --- | --- | --- |
| **Homogeneous** | MySQL → RDS/Aurora MySQL; PostgreSQL → Aurora PostgreSQL | DMS (or native dump/replication). No schema conversion needed |
| **Heterogeneous** | Oracle → Aurora PostgreSQL; SQL Server → MySQL | **DMS Schema Conversion** (in the DMS console) or the **AWS Schema Conversion Tool (SCT)** to convert schema/code, **then** DMS for the data |
| **Into a data warehouse** | Oracle/Teradata → Redshift | SCT (incl. extraction agents) + DMS |

- **Full load + CDC** = near-zero-downtime migration: the source stays live while DMS streams changes; cut over when lag is ~0.
- The source database **stays operational** during migration — that's the selling point.
- DMS can also do **continuous replication** (not just one-time), e.g. on-prem → AWS for reporting, or to S3/Kinesis as a change stream.
- For very large databases with limited bandwidth: load the bulk with **Snowball Edge** (DMS/SCT support this pattern), then use DMS CDC to catch up.

> Trap: DMS does **not** convert stored procedures, schema or application code for heterogeneous moves — Schema Conversion / SCT does. If the answer lists only DMS for Oracle → PostgreSQL, it's incomplete.

> Trap: Moving RDS MySQL → Aurora MySQL in AWS: an **Aurora read replica of the RDS instance**, then promote (Day 6) is simpler than DMS.

### Files and objects: DataSync, Transfer Family, Storage Gateway

| Service | Direction / pattern | Protocols | Pick it when |
| --- | --- | --- | --- |
| **AWS DataSync** | One-time or **scheduled** bulk copy **into, out of, or between** storage (on-prem ↔ AWS, AWS ↔ AWS, other clouds → AWS) | Agent reads NFS, SMB, HDFS, object storage; writes **S3, EFS, FSx** (all flavours) | "Migrate/sync 50 TB of NFS/SMB data online", "automate and verify transfer", "copy EFS to EFS in another Region" |
| **AWS Transfer Family** | External partners keep using file-transfer clients | **SFTP, FTPS, FTP, AS2** → S3 or EFS | "Partners upload via SFTP; we want to stop running an SFTP server" |
| **AWS Storage Gateway** (Day 5) | **Ongoing hybrid** access — on-prem apps keep a local mount/cache backed by AWS | S3 File Gateway (NFS/SMB), FSx File Gateway, Volume Gateway (iSCSI), Tape Gateway (VTL) | "On-prem apps need low-latency access to data stored in AWS", "replace tape backups" |

How DataSync works and why it beats `rsync`/scripts:
- Deploy an **agent** (VM on-prem or EC2) near the source; it talks to the DataSync service over TLS (internet, VPN, or **Direct Connect** via a VPC endpoint).
- Parallel, multi-threaded transfer; **incremental** (only changed data after the first run); built-in **data integrity verification**; **bandwidth throttling**; scheduling; preserves metadata/permissions where the target supports it.
- No agent needed for AWS-to-AWS transfers (e.g. S3 → EFS, EFS → FSx).

> Trap: DataSync = **move/sync data**. Storage Gateway = **keep using it from on-prem** afterwards. "Migrate then decommission the NAS" → DataSync. "Keep the file server but tier to the cloud" → File Gateway.

### Offline transfer: the Snow Family (and its status)

- **AWS Snowball Edge** — rugged appliance shipped to you; load data locally, ship back, AWS imports to S3. Variants: *Storage Optimized* (bulk transfer, edge storage) and *Compute Optimized* (edge compute in disconnected sites). Capacities vary by generation — check current docs.
- Status: AWS has been retiring Snow devices — **Snowmobile** and **Snowcone** are discontinued, and AWS has restricted Snowball availability for new customers while pointing to DataSync, Direct Connect and **AWS Data Transfer Terminal** (physical sites where you bring your own storage to a high-speed AWS connection). Check current docs for what you can order. **SAA-C03 still tests Snowball Edge** as the offline-bulk-transfer answer, so learn the pattern.
- Snowball imports to **S3** only. To land data in Glacier, import to S3 and use a **lifecycle rule** (trap).

### The transfer-time math

`time ≈ data (bits) ÷ usable bandwidth`. 1 TB ≈ 8,000 Gb (8 × 10¹² bits).

| Data | 100 Mbps | 1 Gbps | 10 Gbps |
| --- | --- | --- | --- |
| 1 TB | ~22 h | ~2.2 h | ~13 min |
| 10 TB | ~9 days | ~22 h | ~2.2 h |
| 100 TB | ~3 months | ~9 days | ~22 h |
| 1 PB | years | ~3 months | ~9 days |

(At 100% utilisation; real links rarely give you that — and the business still needs the link.) Rule of thumb from the exam: **if the network would take more than about a week, or the link can't be saturated, ship a device.** Also remember a new **Direct Connect** connection takes weeks to provision — wrong for "migrate within 1 week" if you don't already have one.

### Migration decision flow

```
 What are you moving?
 ├─ Whole servers/VMs, as-is ───────────────────────────► MGN
 ├─ Servers, but for DR (not migration) ────────────────► Elastic Disaster Recovery
 ├─ A database
 │    ├─ same engine ───────────────────────────────────► DMS (full load + CDC)
 │    └─ different engine ──────────────────────────────► Schema Conversion / SCT + DMS
 ├─ Files / objects
 │    ├─ online, one-time or scheduled ─────────────────► DataSync
 │    ├─ partners via SFTP/FTPS/AS2 ────────────────────► Transfer Family
 │    ├─ keep using on-prem, backed by AWS ─────────────► Storage Gateway
 │    └─ 10s of TB+ and slow/no network ────────────────► Snowball Edge
 └─ Need a dedicated, consistent network for all of it ─► Direct Connect (weeks to provision)
```

---

## 5. Monitoring and audit — the three you must never mix up

```
  "How is it PERFORMING?"         "WHO did WHAT, WHEN?"          "WHAT does it LOOK LIKE,
                                                                  and is that ALLOWED?"
  ┌──────────────────────┐        ┌──────────────────────┐       ┌──────────────────────┐
  │   Amazon CloudWatch  │        │    AWS CloudTrail    │       │      AWS Config      │
  │ metrics · logs ·     │        │ API call records:    │       │ resource config      │
  │ alarms · dashboards  │        │ identity, source IP, │       │ history + compliance │
  │                      │        │ time, params, result │       │ rules + remediation  │
  └──────────────────────┘        └──────────────────────┘       └──────────────────────┘
   CPU 95%, 5xx spike,             "Who terminated the              "Was this SG open to
   queue depth, app logs           instance?"                        0.0.0.0/0 last Tuesday?"
```

### 5a. Amazon CloudWatch

**Metrics**
- Namespaces (`AWS/EC2`, `AWS/SQS`…), dimensions (`InstanceId`, `AutoScalingGroupName`), statistics (Average, Sum, p99…).
- **EC2 basic monitoring = 5-minute** granularity (free); **detailed monitoring = 1-minute** (paid). Faster scaling reactions → detailed monitoring.
- **Not collected by default for EC2: memory utilisation, disk space used, per-process metrics** — the hypervisor can't see inside the guest. Install the **CloudWatch agent** (unified agent) to publish them as custom metrics and ship OS/app log files. *Huge* exam trap.
- **Custom metrics** via `PutMetricData`: standard resolution (1 min) or **high resolution (1 s)**.
- Retention rolls up automatically: high-res data kept for hours, 1-min for ~15 days, 5-min for ~63 days, 1-hour for 15 months.

**Alarms**
- States: `OK`, `ALARM`, `INSUFFICIENT_DATA`. Evaluate a metric (or **metric math**, or **anomaly detection** band) over N periods.
- Actions: **SNS** notification, **EC2 action** (stop, terminate, reboot, **recover** — moves the instance to new hardware on a failed *system* status check, keeping instance ID, private IP, EBS), **Auto Scaling** policy, **Systems Manager** OpsItem/incident.
- **Composite alarms**: combine alarms with AND/OR to cut noise ("page only if 5xx is high AND latency is high").
- Alarms are what Route 53 health checks can watch for private resources.

**Logs**
- Log groups (retention configurable — **default is never expire**, so set it for cost) → log streams.
- Sources: Lambda, API Gateway, VPC Flow Logs, CloudTrail, Route 53 Resolver query logs, ECS/EKS, the CloudWatch agent on EC2/on-prem.
- **Metric filters**: turn log patterns into metrics (count `ERROR`, count `ConsoleLogin` failures) → alarm on them.
- **Logs Insights**: interactive query language over log groups.
- **Subscription filters**: real-time stream to **Kinesis Data Streams, Amazon Data Firehose, Lambda** or OpenSearch (e.g. centralise logs to S3 via Firehose, or across accounts).
- **Export to S3** (`CreateExportTask`) is batch and not real-time — use subscription filters for near-real-time.
- Logs are encrypted at rest; you can associate a **KMS customer managed key** per log group.

**Other CloudWatch features to recognise**

| Feature | Use |
| --- | --- |
| **Dashboards** | Cross-account, cross-Region views |
| **Synthetics canaries** | Scripted checks of endpoints/UI flows from outside ("know the login page is broken before users do") |
| **RUM** | Real user monitoring for web apps |
| **Container Insights / Lambda Insights** | Curated metrics for ECS/EKS/Lambda |
| **Application Signals / X-Ray** | Distributed tracing and service maps — "find which microservice adds latency" → **X-Ray** |
| **Cross-account observability** | A monitoring account sees metrics/logs/traces from source accounts |

**Amazon EventBridge** (Day 8) is the reaction engine: AWS service events (EC2 state change, GuardDuty finding, CloudTrail-recorded API call, Health events, scheduled cron) → rule → target (Lambda, SNS, SQS, Step Functions, SSM Automation). "Run a Lambda every night" / "react when an instance is terminated" → EventBridge rule.

### 5b. AWS CloudTrail

Records **API calls** (console, CLI, SDK, and AWS services acting for you) — who (identity/role/session), when, from where (source IP, user agent), what (event name, request parameters), and the result.

| Concept | Details |
| --- | --- |
| **Event history** | On by default, free, **last 90 days of management events**, per Region, viewable/searchable in the console. No setup |
| **Trail** | Delivers events to an **S3 bucket** (and optionally **CloudWatch Logs**) for retention beyond 90 days, analysis and alerting. Can be **multi-Region** (recommended) and an **organization trail** (all accounts, created from the management/delegated admin account) |
| **Management events** | Control-plane ops: `RunInstances`, `CreateBucket`, `PutBucketPolicy`, `ConsoleLogin`, IAM changes. Logged by default |
| **Data events** | High-volume data-plane ops: **S3 object-level** (`GetObject`, `PutObject`, `DeleteObject`), Lambda `Invoke`, DynamoDB item-level, and more. **Off by default, extra cost**. "Who downloaded this object?" needs data events |
| **Insights events** | Detect unusual **API call volume or error rate** vs baseline ("sudden burst of `TerminateInstances`") |
| **Log file integrity validation** | SHA-256 digest files prove logs weren't modified or deleted after delivery |
| **CloudTrail Lake** | Managed, queryable (SQL) event data store with long retention — analysis without building Athena tables |

Protecting the audit trail (classic "secure the logs" question):

```
 Org trail (all accounts, all Regions)
     │  log file validation ON, SSE-KMS
     ▼
 S3 bucket in a separate LOG ARCHIVE account
     · bucket policy: CloudTrail may write; nobody in workload accounts may delete
     · Versioning + S3 Object Lock (WORM) · MFA delete
     · lifecycle → Glacier for long retention
 + SCP: Deny cloudtrail:StopLogging / DeleteTrail / UpdateTrail in member accounts
 + optional: CloudWatch Logs → metric filter → alarm on root login / IAM policy change
```

Querying: Event history for quick 90-day look-ups; **Athena** over the S3 trail (or CloudTrail Lake) for SQL across months/accounts.

> Trap: CloudTrail tells you **who changed** something; it does **not** evaluate compliance or show a resource's full configuration over time — that's Config. And it doesn't measure performance — that's CloudWatch.

> Trap: CloudTrail ≠ VPC Flow Logs. Flow Logs record **network traffic metadata** (IP, port, accept/reject); CloudTrail records **API calls**.

### 5c. AWS Config

Continuously **records resource configurations** and evaluates them against rules.

```
 Configuration recorder ──► configuration items (CIs) per resource change
        │                          │
        │                          ├──► Config timeline: what it looked like at any time,
        │                          │    linked to the CloudTrail event that changed it
        │                          └──► snapshots/history delivered to S3; changes to SNS
        ▼
 Config rules (managed, or custom: Lambda / Guard policy-as-code)
        │   evaluate on change or periodically
        ▼
 COMPLIANT / NON_COMPLIANT ──► Remediation: SSM Automation document (manual or AUTOMATIC)
                          └──► EventBridge → notify / custom action
```

| Concept | Details |
| --- | --- |
| **Regional** | Enable the recorder in each Region you use (and record global resources like IAM in one) |
| **Managed rules** | Hundreds, e.g. `restricted-ssh`, `s3-bucket-public-read-prohibited`, `encrypted-volumes`, `rds-multi-az-support`, `required-tags`, `cloudtrail-enabled`, `root-account-mfa-enabled` |
| **Custom rules** | Lambda-backed or **CloudFormation Guard** policy rules |
| **Remediation** | Attach **SSM Automation** runbooks, e.g. `AWS-DisableS3BucketPublicReadWrite`; automatic remediation with retries |
| **Conformance packs** | Bundle rules + remediations as one deployable unit (e.g. operational best practices for PCI DSS, CIS); deploy org-wide |
| **Aggregator** | Multi-account, multi-Region compliance view in one account |
| **Cost** | Charged per configuration item recorded and per rule evaluation — scope recording to needed resource types if cost matters (check current pricing) |

**Config is detective, SCPs are preventive.** "Prevent anyone from creating an unencrypted volume" → SCP/IAM condition (or EBS encryption-by-default). "Detect and auto-fix unencrypted volumes / open security groups" → Config rule + remediation.

### 5d. Supporting cast

| Service | One-liner |
| --- | --- |
| **AWS Health Dashboard** | AWS-side events affecting *your* resources (scheduled maintenance, service issues); integrate via EventBridge |
| **Systems Manager** | Ops hub: Session Manager, Patch Manager, Run Command, Automation runbooks (used by Config remediation), OpsCenter, Parameter Store |
| **Trusted Advisor** | Best-practice checks (cost, security, fault tolerance, performance, service limits) — Day 12 |
| **Service Quotas** | View quotas and request increases; CloudWatch alarm when usage approaches a quota |
| **VPC Flow Logs** | IP traffic metadata for VPC/subnet/ENI → CloudWatch Logs or S3 (Day 4) |

### 5e. Standard pipelines to memorise

| Requirement | Pipeline |
| --- | --- |
| Alert when the root user signs in | CloudTrail → **EventBridge rule** on `ConsoleLogin` with `userIdentity.type = Root` → SNS (or CloudTrail → CloudWatch Logs metric filter → alarm → SNS) |
| Auto-remediate public S3 buckets / open SSH | **Config rule** → automatic remediation with **SSM Automation** |
| Restart / recover an impaired instance automatically | CloudWatch alarm on `StatusCheckFailed_System` → **EC2 recover action** |
| Scale on memory usage | **CloudWatch agent** publishes memory → custom metric → target tracking / step scaling |
| Count application errors from logs and page on-call | Log group → **metric filter** → alarm → SNS |
| Stream all logs centrally in near-real-time | **Subscription filter** → Data Firehose → S3 (or OpenSearch) |
| Who deleted the S3 objects? | CloudTrail **data events** for S3 (must be enabled *before* it happened) |
| What did this security group look like last month and who changed it? | **Config** timeline (links to the CloudTrail event) |
| Make sure every account always has CloudTrail on | Organization trail + **SCP** denying `StopLogging`/`DeleteTrail` (+ Config rule `cloudtrail-enabled` to detect) |

---

## 6. Exam traps roundup

1. **Multi-AZ is not DR.** It doesn't protect against Region loss, corruption or deletion.
2. **Backup interval caps RPO.** Daily snapshots → RPO up to 24 h, whatever the restore speed.
3. **Pilot light = data live, compute off. Warm standby = everything on, small.** Read for "running but scaled down" vs "launch when needed".
4. **Snapshots and AMIs are Regional.** Backup & restore DR needs them **copied** to the DR Region (and the KMS key must exist there — multi-Region key or re-encrypt on copy, Day 10).
5. **Cross-Region read replica promotion is manual and one-way**; Aurora Global Database has managed failover with ~1 s RPO.
6. **DMS alone doesn't convert schemas** for heterogeneous moves → Schema Conversion / SCT.
7. **DataSync moves; Storage Gateway bridges.** Transfer Family is for SFTP/FTPS/FTP/AS2 clients.
8. **Snowball lands in S3**, not directly in Glacier.
9. **New Direct Connect takes weeks** — not a fix for "this week".
10. **EC2 memory/disk metrics need the CloudWatch agent.**
11. **CloudTrail management events are on (90 days free); data events are off** and must be enabled in advance.
12. **CloudTrail = who did it; Config = what it looked like / compliant?; CloudWatch = how it performs.**
13. **Config detects; SCPs prevent.** Config remediation runs SSM Automation.
14. **Log retention defaults to never expire** — set it to control cost.
15. **CloudWatch Logs export to S3 is batch**; use subscription filters for real time.

---

## 7. Exam decision table

| If the question says… | Pick |
| --- | --- |
| RPO/RTO in hours, lowest cost | **Backup & restore** (AWS Backup cross-Region copy, snapshot/AMI copy, IaC rebuild) |
| RPO minutes, RTO tens of minutes, "core services running, launch app servers on failover" | **Pilot light** |
| RTO minutes, "scaled-down but fully functional copy always running" | **Warm standby** |
| RPO/RTO near zero, traffic served from multiple Regions | **Multi-site active/active** (Aurora Global DB / DynamoDB global tables + Route 53 / Global Accelerator) |
| On-prem servers need DR in AWS, RPO seconds, RTO minutes, minimal infra | **AWS Elastic Disaster Recovery** |
| Relational DB in another Region, RPO ~1 s, failover in ~1 min | **Aurora Global Database** |
| Multi-Region, multi-active NoSQL | **DynamoDB global tables** |
| S3 replication with predictable 15-minute SLA | **S3 CRR + Replication Time Control** |
| Controlled, reliable Region failover with readiness checks | **Application Recovery Controller** (routing controls) + Route 53 |
| Centralise backups across RDS, EBS, EFS, DynamoDB… by tag | **AWS Backup** plan with tag-based assignment |
| Enforce backups across all accounts in the org | **AWS Backup policies** via Organizations |
| Backups must be immutable / ransomware-proof / not deletable even by root | **Backup Vault Lock (compliance mode)** + cross-account copy |
| Prove backup compliance to auditors | **Backup Audit Manager** |
| Rewind an Aurora MySQL DB a few minutes without restoring | **Aurora Backtrack** |
| Restore DynamoDB to any second in the last 35 days | **DynamoDB PITR** |
| Lift-and-shift many servers to EC2 with minimal downtime | **Application Migration Service (MGN)** |
| Discover on-prem servers and their dependencies | **Application Discovery Service** |
| Migrate DB, same engine, source stays online | **DMS** full load + CDC |
| Oracle/SQL Server → Aurora PostgreSQL/MySQL | **DMS Schema Conversion (or SCT) + DMS** |
| RDS MySQL → Aurora MySQL with minimal downtime | **Aurora read replica** of the RDS instance, then promote |
| Copy/sync NFS/SMB/HDFS data to S3/EFS/FSx online, scheduled, verified | **DataSync** |
| Partners must keep using SFTP; stop running SFTP servers | **Transfer Family** |
| On-prem apps keep accessing files, stored in S3 | **S3 File Gateway** |
| Replace physical tape backups | **Tape Gateway** |
| 100s of TB, limited bandwidth, weeks of transfer otherwise | **Snowball Edge** (check current availability) |
| Edge compute in a disconnected site | **Snowball Edge Compute Optimized** |
| EC2 memory / disk utilisation metrics | **CloudWatch agent** (custom metrics) |
| Faster (1-minute) EC2 metrics for scaling | **Detailed monitoring** |
| Turn log patterns into alarms | **CloudWatch Logs metric filter** + alarm |
| Query logs interactively | **CloudWatch Logs Insights** |
| Real-time log streaming to S3/OpenSearch/another account | **Subscription filter** → Data Firehose / Kinesis / Lambda |
| Alarm only when two conditions are both true | **Composite alarm** |
| Recover instance on underlying hardware failure, keep IP | CloudWatch alarm → **EC2 recover** action |
| Test the website's login flow every 5 minutes | **CloudWatch Synthetics canary** |
| Find the slow microservice | **AWS X-Ray** |
| Who made this API call / who deleted the resource | **CloudTrail** |
| Who read/deleted specific S3 objects | **CloudTrail data events** |
| Keep API logs > 90 days, all accounts and Regions | **Organization trail** (multi-Region) → S3 in log archive account |
| Prove log files weren't tampered with | **Log file integrity validation** |
| Detect unusual API call volume | **CloudTrail Insights** |
| SQL over CloudTrail events without building tables | **CloudTrail Lake** (or Athena over the S3 trail) |
| Resource configuration history / was this ever non-compliant | **AWS Config** |
| Detect and auto-remediate non-compliant resources | **Config rule + SSM Automation remediation** |
| Org-wide compliance dashboard across Regions | **Config aggregator** (+ conformance packs) |
| React to an AWS API event or service event automatically | **EventBridge rule** |
| Notified about AWS maintenance affecting my instances | **AWS Health** (+ EventBridge) |

---

## 8. Lab (30 min, free tier)

Goal: back up and restore with AWS Backup, build an alarm from a custom metric and a metric filter, answer "who did it?" with CloudTrail, and watch Config flag an open SSH rule. Run CLI steps from **CloudShell** in `ap-south-1`.

Cost notes (check current pricing): DynamoDB on-demand with a few items is effectively free; AWS Backup charges per GB-month of backup storage (a tiny table costs ~nothing). CloudWatch has a free tier for basic metrics, a few alarms and small log volumes. CloudTrail Event history is free. **AWS Config is not free** — it charges per configuration item and rule evaluation; recording only security groups for 15 minutes costs cents, but **stop the recorder at teardown**. MGN, DRS and DMS are console walkthroughs only today.

### Console + CLI steps

1. **Create something to protect** (DynamoDB, on-demand, tagged for backup):
   ```bash
   aws dynamodb create-table --table-name saa-day11 \
     --attribute-definitions AttributeName=pk,AttributeType=S \
     --key-schema AttributeName=pk,KeyType=HASH --billing-mode PAY_PER_REQUEST \
     --tags Key=backup,Value=daily
   aws dynamodb wait table-exists --table-name saa-day11
   for i in 1 2 3; do aws dynamodb put-item --table-name saa-day11 \
     --item "{\"pk\":{\"S\":\"order-$i\"},\"status\":{\"S\":\"paid\"}}"; done
   ```
2. **AWS Backup:** console → *Backup vaults* → note the `Default` vault and its KMS key. *Backup plans* → **Create backup plan** → *Build a new plan* → name `saa-day11`, rule `daily`, frequency *Daily*, retention 7 days. Look at (don't enable) **Copy to destination** (cross-Region / cross-account) and the cold-storage transition. Then **Assign resources** → *Include specific resource types* or *by tag* `backup = daily`. Also open *Vault* → **Vault Lock** → read the governance vs compliance options and **cancel** (compliance mode is irreversible after the cooling-off period).
3. **On-demand backup + restore** (no need to wait for the schedule):
   ```bash
   ACCT=$(aws sts get-caller-identity --query Account --output text)
   ROLE=arn:aws:iam::$ACCT:role/service-role/AWSBackupDefaultServiceRole
   # (the console creates this role the first time you assign resources with "Default role")
   JOB=$(aws backup start-backup-job --backup-vault-name Default \
     --resource-arn arn:aws:dynamodb:ap-south-1:$ACCT:table/saa-day11 \
     --iam-role-arn $ROLE --query BackupJobId --output text)
   aws backup describe-backup-job --backup-job-id $JOB --query '[State,RecoveryPointArn]'
   ```
   When `COMPLETED`, simulate a mistake and restore:
   ```bash
   aws dynamodb delete-item --table-name saa-day11 --key '{"pk":{"S":"order-2"}}'
   ```
   Console → *Protected resources* → `saa-day11` → pick the recovery point → **Restore** → new table name `saa-day11-restored`. When done: `aws dynamodb scan --table-name saa-day11-restored --select COUNT` → 3 items. Restores create a **new** resource; you then repoint the app — that's why "backup & restore" has the longest RTO.
4. **CloudWatch alarm from a custom metric:**
   ```bash
   TOPIC=$(aws sns create-topic --name saa-day11-alerts --query TopicArn --output text)
   aws sns subscribe --topic-arn $TOPIC --protocol email --notification-endpoint you@example.com
   # confirm the subscription email
   aws cloudwatch put-metric-alarm --alarm-name saa-day11-errors \
     --namespace SAA/Day11 --metric-name Errors --statistic Sum --period 60 \
     --evaluation-periods 1 --threshold 5 --comparison-operator GreaterThanOrEqualToThreshold \
     --treat-missing-data notBreaching --alarm-actions $TOPIC
   for i in $(seq 1 6); do aws cloudwatch put-metric-data --namespace SAA/Day11 \
     --metric-name Errors --value 1; done
   ```
   Within a couple of minutes the alarm goes `ALARM` and you get an email. This is exactly how the CloudWatch agent's memory metric would drive an alarm or scaling policy.
5. **Logs → metric filter → Logs Insights:**
   ```bash
   aws logs create-log-group --log-group-name /saa/day11/app
   aws logs put-retention-policy --log-group-name /saa/day11/app --retention-in-days 1
   aws logs create-log-stream --log-group-name /saa/day11/app --log-stream-name web-1
   aws logs put-metric-filter --log-group-name /saa/day11/app --filter-name errors \
     --filter-pattern '"ERROR"' --metric-transformations \
     metricName=AppErrors,metricNamespace=SAA/Day11,metricValue=1
   NOW=$(date +%s000)
   aws logs put-log-events --log-group-name /saa/day11/app --log-stream-name web-1 \
     --log-events timestamp=$NOW,message="INFO started" timestamp=$NOW,message="ERROR db timeout" \
                  timestamp=$NOW,message="ERROR db timeout"
   ```
   Console → *Logs Insights* → select `/saa/day11/app` → run
   `fields @timestamp, @message | filter @message like /ERROR/ | stats count() by bin(1m)`.
   Then *Metrics* → `SAA/Day11` → `AppErrors` shows the count. You just built the "page on log errors" pipeline.
6. **CloudTrail — who did it?** Console → CloudTrail → **Event history** → filter *Event name* = `DeleteItem`… nothing: DynamoDB item operations are **data events**, not logged by default. Now filter *Event name* = `CreateTable` → open it → see your identity, source IP, time and request parameters. Also look up `PutMetricAlarm`. Then open **Trails** → *Create trail* and read the *Data events* and *Insights events* sections — **cancel** (your 90-day history is enough for the lab).
7. **AWS Config — detect an open SSH rule:**
   - Console → Config → *Get started* (or *Settings* → *Edit*) → **Record specific resource types** → `AWS::EC2::SecurityGroup` only → create a new S3 bucket for delivery → use the service-linked role → *Next*. Add managed rule **`restricted-ssh`** → confirm.
   - ```bash
     VPC=$(aws ec2 describe-vpcs --filters Name=is-default,Values=true --query 'Vpcs[0].VpcId' --output text)
     SG=$(aws ec2 create-security-group --group-name saa-day11-open --description "bad SG" \
       --vpc-id $VPC --query GroupId --output text)
     aws ec2 authorize-security-group-ingress --group-id $SG --protocol tcp --port 22 --cidr 0.0.0.0/0
     ```
   - After a few minutes: *Rules* → `restricted-ssh` → `saa-day11-open` is **Noncompliant**. Open the resource → **Resource timeline**: the configuration change, with a link to the CloudTrail `AuthorizeSecurityGroupIngress` event. That link (Config "what" ↔ CloudTrail "who") is a favourite exam theme.
   - On the rule, open *Actions* → **Manage remediation** and look at the SSM Automation runbook list (e.g. `AWS-DisablePublicAccessForSecurityGroup`). Don't enable.
8. **Walkthroughs (no create):**
   - **Elastic Disaster Recovery** → *Settings* → *Replication settings template*: staging area subnet, replication server instance type, EBS type, point-in-time policy. Note how cheap the staging area is vs a warm standby.
   - **Application Migration Service** → *Source servers*: the agent install command and the *Launch test instances → Mark ready for cutover → Launch cutover instances* flow.
   - **DMS** → *Create migration task* form: *Migrate existing data* vs *Migrate existing data and replicate ongoing changes* (that's full load + CDC). Then **Schema conversion** in the left nav.
   - **DataSync** → *Create task*: source location types (NFS, SMB, HDFS, object storage, S3, EFS, FSx, other clouds) and the *Verify data* and *Bandwidth limit* options.

### Terraform variant

Covers the DynamoDB table, a Backup vault/plan/selection, SNS + alarm, log group + metric filter, and a Config recorder scoped to security groups with the `restricted-ssh` rule.

```hcl
terraform {
  required_providers { aws = { source = "hashicorp/aws" } }
}
provider "aws" { region = "ap-south-1" }

data "aws_caller_identity" "me" {}
locals { acct = data.aws_caller_identity.me.account_id }

# --- Thing to protect -------------------------------------------------------
resource "aws_dynamodb_table" "orders" {
  name         = "saa-day11-tf"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "pk"
  attribute {
    name = "pk"
    type = "S"
  }
  point_in_time_recovery { enabled = true }   # native PITR, any second up to 35 days
  tags = { backup = "daily" }
}

# --- AWS Backup -------------------------------------------------------------
resource "aws_backup_vault" "lab" {
  name          = "saa-day11-tf"
  force_destroy = true                        # lab only: lets destroy remove recovery points
}

resource "aws_backup_plan" "daily" {
  name = "saa-day11-tf"
  rule {
    rule_name         = "daily"
    target_vault_name = aws_backup_vault.lab.name
    schedule          = "cron(0 20 * * ? *)"  # 20:00 UTC = 01:30 IST
    lifecycle { delete_after = 7 }
    # copy_action { destination_vault_arn = "<vault ARN in DR Region or account>" }
  }
}

resource "aws_iam_role" "backup" {
  name = "saa-day11-backup"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{ Effect = "Allow", Action = "sts:AssumeRole",
                   Principal = { Service = "backup.amazonaws.com" } }]
  })
}

resource "aws_iam_role_policy_attachment" "backup" {
  role       = aws_iam_role.backup.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForBackup"
}

resource "aws_iam_role_policy_attachment" "restore" {
  role       = aws_iam_role.backup.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForRestores"
}

resource "aws_backup_selection" "by_tag" {
  name         = "backup-daily-tag"
  plan_id      = aws_backup_plan.daily.id
  iam_role_arn = aws_iam_role.backup.arn
  selection_tag {
    type  = "STRINGEQUALS"
    key   = "backup"
    value = "daily"
  }
}

# --- CloudWatch: SNS, alarm, logs, metric filter ----------------------------
resource "aws_sns_topic" "alerts" { name = "saa-day11-tf-alerts" }

resource "aws_cloudwatch_log_group" "app" {
  name              = "/saa/day11tf/app"
  retention_in_days = 1                       # default would be "never expire"
}

resource "aws_cloudwatch_log_metric_filter" "errors" {
  name           = "errors"
  log_group_name = aws_cloudwatch_log_group.app.name
  pattern        = "\"ERROR\""
  metric_transformation {
    name      = "AppErrors"
    namespace = "SAA/Day11TF"
    value     = "1"
  }
}

resource "aws_cloudwatch_metric_alarm" "app_errors" {
  alarm_name          = "saa-day11-tf-app-errors"
  namespace           = "SAA/Day11TF"
  metric_name         = "AppErrors"
  statistic           = "Sum"
  period              = 60
  evaluation_periods  = 1
  threshold           = 5
  comparison_operator = "GreaterThanOrEqualToThreshold"
  treat_missing_data  = "notBreaching"
  alarm_actions       = [aws_sns_topic.alerts.arn]
}

# --- AWS Config (scoped to security groups to keep cost tiny) ---------------
resource "aws_s3_bucket" "config" {
  bucket        = "saa-day11-config-${local.acct}"
  force_destroy = true
}

resource "aws_s3_bucket_policy" "config" {
  bucket = aws_s3_bucket.config.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      { Sid = "ConfigAclCheck", Effect = "Allow",
        Principal = { Service = "config.amazonaws.com" },
        Action = ["s3:GetBucketAcl", "s3:ListBucket"], Resource = aws_s3_bucket.config.arn },
      { Sid = "ConfigWrite", Effect = "Allow",
        Principal = { Service = "config.amazonaws.com" },
        Action = "s3:PutObject",
        Resource = "${aws_s3_bucket.config.arn}/AWSLogs/${local.acct}/Config/*",
        Condition = { StringEquals = { "s3:x-amz-acl" = "bucket-owner-full-control" } } }
    ]
  })
}

resource "aws_iam_service_linked_role" "config" {
  aws_service_name = "config.amazonaws.com"   # skip if it already exists in your account
}

resource "aws_config_configuration_recorder" "this" {
  name     = "default"
  role_arn = aws_iam_service_linked_role.config.arn
  recording_group {
    all_supported  = false
    resource_types = ["AWS::EC2::SecurityGroup"]
  }
}

resource "aws_config_delivery_channel" "this" {
  name           = "default"
  s3_bucket_name = aws_s3_bucket.config.id
  depends_on     = [aws_config_configuration_recorder.this, aws_s3_bucket_policy.config]
}

resource "aws_config_configuration_recorder_status" "this" {
  name       = aws_config_configuration_recorder.this.name
  is_enabled = true
  depends_on = [aws_config_delivery_channel.this]
}

resource "aws_config_config_rule" "restricted_ssh" {
  name = "restricted-ssh"
  source {
    owner             = "AWS"
    source_identifier = "INCOMING_SSH_DISABLED"   # the managed rule "restricted-ssh"
  }
  depends_on = [aws_config_configuration_recorder_status.this]
}

# --- A deliberately non-compliant SG to trigger the rule --------------------
data "aws_vpc" "default" { default = true }

resource "aws_security_group" "bad" {
  name        = "saa-day11-tf-open"
  description = "bad SG for Config demo"
  vpc_id      = data.aws_vpc.default.id
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

output "sns_topic" { value = aws_sns_topic.alerts.arn }
```

`terraform init && terraform apply`, subscribe your email to the topic, push a few `ERROR` lines into the log group, and check Config after a few minutes. If you already have a Config recorder in this Region (only one is allowed per Region) or the service-linked role exists, drop those resources or `terraform import` them.

### Teardown

Console/CLI path:
```bash
aws configservice stop-configuration-recorder --configuration-recorder-name default
aws configservice delete-config-rule --config-rule-name restricted-ssh
aws configservice delete-delivery-channel --delivery-channel-name default
aws configservice delete-configuration-recorder --configuration-recorder-name default
aws ec2 delete-security-group --group-id $SG
aws cloudwatch delete-alarms --alarm-names saa-day11-errors
aws logs delete-log-group --log-group-name /saa/day11/app
aws sns delete-topic --topic-arn $TOPIC
aws dynamodb delete-table --table-name saa-day11
aws dynamodb delete-table --table-name saa-day11-restored
# AWS Backup: delete the recovery point(s), then the plan
for RP in $(aws backup list-recovery-points-by-backup-vault --backup-vault-name Default \
  --query 'RecoveryPoints[?contains(ResourceArn, `saa-day11`)].RecoveryPointArn' --output text); do
  aws backup delete-recovery-point --backup-vault-name Default --recovery-point-arn $RP; done
```
Then delete the `saa-day11` backup plan in the console (remove its resource assignment first), and empty + delete the Config delivery bucket.

Terraform path: `terraform destroy` (stops the recorder, deletes the vault with `force_destroy`, bucket, alarm, table). Confirm in the Config console that **recording is off** — that's the only item here that keeps charging.

---

## 9. Practice questions

Answer all ten before opening the answers.

**Q1.** A company runs a three-tier application in `us-east-1`. For DR in `us-west-2` it requires an RPO of 15 minutes and an RTO of 4 hours, at the LOWEST cost. The database is Amazon RDS for MySQL. Which strategy meets the requirements?
A. Multi-site active/active with Route 53 latency routing
B. Pilot light: an RDS cross-Region read replica in `us-west-2`, AMIs and IaC templates copied; launch the app tier on failover
C. Warm standby with a full-capacity Auto Scaling group always running in `us-west-2`
D. Daily AWS Backup copies of the RDS instance to `us-west-2`

**Q2.** A company must protect backups of EBS, RDS and EFS from deletion by anyone, including administrators with full access to the production account, for seven years to satisfy regulators. What should a solutions architect do?
A. Enable S3 Object Lock on the bucket that stores EBS snapshots
B. Use AWS Backup with a vault configured with Vault Lock in compliance mode, and copy recovery points to a vault in a separate account
C. Create an SCP that denies `ec2:DeleteSnapshot`
D. Enable termination protection on all resources

**Q3.** A company is migrating 300 on-premises VMware virtual machines to Amazon EC2. It wants to rehost without changing the applications and with cutover downtime measured in minutes. Which service should be used?
A. AWS DataSync
B. AWS Database Migration Service
C. AWS Application Migration Service
D. AWS Snowball Edge

**Q4.** A company is moving a 5 TB Oracle database to Amazon Aurora PostgreSQL. The source must remain available during the migration, with minimal downtime at cutover. Which combination should be used? (Select TWO.)
A. AWS DMS Schema Conversion (or AWS SCT) to convert the schema and code objects
B. AWS DataSync to copy the Oracle data files
C. AWS DMS with a full load and ongoing replication (CDC) task
D. Aurora Backtrack
E. An Aurora read replica of the Oracle database

**Q5.** A media company needs to move 600 TB of archived video from its data center to Amazon S3 within three weeks. Its internet connection is 500 Mbps and is already 50% utilised during business hours. What is the MOST suitable approach?
A. AWS DataSync over the existing internet connection
B. Order a new 10 Gbps Direct Connect connection
C. Use multiple AWS Snowball Edge devices to ship the data
D. Use S3 Transfer Acceleration

**Q6.** An operations team needs to scale an Auto Scaling group based on memory utilisation of its EC2 instances. CloudWatch shows no memory metric. What should the team do?
A. Enable detailed monitoring on the instances
B. Install and configure the CloudWatch agent to publish memory as a custom metric, then create a scaling policy on it
C. Create a CloudTrail trail with data events
D. Use AWS Config to record instance memory

**Q7.** A security team must find out which IAM principal deleted several objects from an S3 bucket yesterday. CloudTrail Event history shows nothing relevant. What is the reason, and what should be done to answer this in the future?
A. Event history only covers 24 hours; create a trail
B. S3 object-level operations are data events, which aren't logged by default; enable S3 data events on a trail
C. CloudTrail doesn't support S3; use VPC Flow Logs
D. Enable S3 server access logging retroactively

**Q8.** A company wants to detect any security group that allows inbound SSH from `0.0.0.0/0` in all accounts and automatically remove the rule, with the LEAST operational overhead. What should be used?
A. A Lambda function on a schedule that scans all security groups
B. An AWS Config managed rule (`restricted-ssh`) deployed org-wide with automatic remediation using an SSM Automation runbook
C. Amazon GuardDuty
D. VPC Flow Logs with a metric filter

**Q9.** A company needs a relational database that serves reads in three Regions with low latency and can fail over to another Region with an RPO of about one second and an RTO of about a minute if the primary Region fails. Which solution should be used?
A. RDS Multi-AZ DB cluster
B. RDS cross-Region read replicas promoted manually
C. Amazon Aurora Global Database
D. DynamoDB global tables

**Q10.** A company's file server holds 40 TB on NFS. It wants to migrate the data to Amazon EFS over its existing Direct Connect link, run incremental syncs nightly until cutover, and verify data integrity — with no custom scripts. What should it use?
A. AWS Storage Gateway S3 File Gateway
B. AWS Transfer Family
C. AWS DataSync with an agent on premises and a scheduled task
D. `rsync` running on an EC2 instance

<details>
<summary><strong>Answers</strong></summary>

1. **B** — An RPO of 15 min rules out daily backups (D). An RTO of 4 h gives time to launch compute, so pilot light is enough; warm standby (C) and active/active (A) meet it but cost more.
2. **B** — Vault Lock compliance mode makes recovery points undeletable by anyone, root included, after the cooling-off period; a cross-account copy guards against a compromised account. An SCP (C) can be changed by the management account and doesn't cover RDS/EFS.
3. **C** — MGN does continuous block-level replication and fast cutover for lift-and-shift. DataSync (A) moves files, not bootable servers.
4. **A, C** — Heterogeneous move: Schema Conversion / SCT converts schema and code, then DMS full load + CDC moves data while the source stays live. You can't create an Aurora replica of Oracle (E).
5. **C** — 600 TB at ~250 Mbps usable is roughly 7 months. DataSync over that link (A) is far too slow; a new Direct Connect (B) takes weeks to provision; Transfer Acceleration (D) speeds long-distance paths but doesn't add bandwidth.
6. **B** — Memory is inside the guest OS; only the CloudWatch agent can publish it. Detailed monitoring (A) only makes the default metrics 1-minute.
7. **B** — `DeleteObject` is an S3 data event; trails must have data events enabled before the action. Event history covers 90 days of management events (A is wrong on both counts). Server access logs (D) can't be enabled retroactively either.
8. **B** — Config managed rule + automatic SSM remediation is native and deployable org-wide (conformance pack / org rule). A works but is custom code; GuardDuty (C) detects threats, not misconfigurations.
9. **C** — Aurora Global Database: up to several secondary Regions for reads, storage-level replication with ~1 s lag, managed cross-Region failover. Manual promotion (B) is slower and has higher RPO; DynamoDB (D) isn't relational.
10. **C** — DataSync: agent-based, NFS → EFS, schedulable incremental tasks, built-in verification, works over Direct Connect. File Gateway (A) is for ongoing hybrid access, not migration.

</details>

---

## 10. Flashcards (cover the right column)

| Prompt | Answer |
| --- | --- |
| RPO measures… | Acceptable data loss (time since last good copy) |
| RTO measures… | Acceptable downtime |
| Cheapest DR strategy | Backup & restore |
| DR strategy: data live, compute off | Pilot light |
| DR strategy: full stack running small | Warm standby |
| DR strategy with near-zero RPO/RTO | Multi-site active/active |
| Managed server DR, RPO seconds, RTO minutes | AWS Elastic Disaster Recovery |
| Aurora cross-Region DR with ~1 s RPO | Aurora Global Database |
| S3 replication with a 15-minute SLA | S3 Replication Time Control |
| Highly available failover switches + readiness checks | Application Recovery Controller |
| Immutable backups even root can't delete | AWS Backup Vault Lock — compliance mode |
| Vault Lock minimum cooling-off period | 72 hours |
| Enforce backup plans across accounts | AWS Backup policies (Organizations) |
| Rewind Aurora MySQL in place | Backtrack |
| Lift-and-shift servers to EC2 | Application Migration Service (MGN) |
| Find on-prem server dependencies | Application Discovery Service |
| DMS mode for minimal-downtime DB migration | Full load + CDC |
| Converts Oracle schema/code to PostgreSQL | DMS Schema Conversion / AWS SCT |
| Online scheduled NFS/SMB → S3/EFS/FSx copy with verification | DataSync |
| Managed SFTP/FTPS/AS2 endpoint into S3/EFS | Transfer Family |
| Snowball imports land in… | S3 (lifecycle to Glacier afterwards) |
| 100 TB over 1 Gbps takes about… | 9 days at full utilisation |
| EC2 basic vs detailed monitoring | 5 min vs 1 min |
| Memory/disk metrics on EC2 need… | CloudWatch agent |
| Highest custom metric resolution | 1 second |
| Default CloudWatch Logs retention | Never expire |
| Real-time log streaming from CloudWatch Logs | Subscription filter |
| Combine alarms with AND/OR | Composite alarm |
| Alarm action that moves instance to healthy hardware | EC2 recover |
| CloudTrail Event history covers | 90 days of management events, free |
| Logged by default: management or data events? | Management |
| Detect unusual API activity volume | CloudTrail Insights |
| Prove CloudTrail logs weren't altered | Log file integrity validation |
| Resource config history + compliance | AWS Config |
| Config auto-remediation runs… | SSM Automation runbooks |
| Multi-account Config view | Aggregator |
| Detective vs preventive: Config / SCP | Config detects, SCP prevents |

## 11. Before you close today

- [ ] Read sections 1–7 (focus: RPO/RTO → strategy, pilot light vs warm standby, Backup Vault Lock, the migration decision flow, transfer math, CloudWatch vs CloudTrail vs Config)
- [ ] Lab done and torn down (Config recorder **stopped and deleted**, SG, alarm, log group, SNS topic, both DynamoDB tables, recovery points, backup plan, Config bucket)
- [ ] 10 questions above + 20 bank questions on DR/Backup/migration/monitoring
- [ ] Misses logged in `miss-log.md` — still empty; Days 12–13 are built from it, so add every miss from Days 7–11 before tomorrow's Practice Exam 2

**Tomorrow (Day 12):** Analytics (Athena, Glue, EMR, Lake Formation, Redshift Spectrum, OpenSearch, QuickSight, Kinesis/Data Firehose recap) and cost tools (Cost Explorer, Budgets, Data Exports/CUR, Compute Optimizer, Trusted Advisor, Savings Plans) — then Practice Exam 2.
