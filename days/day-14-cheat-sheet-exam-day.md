# Day 14 — Cheat Sheet, Exam-Day Logistics, Rest

**Domain weight:** all four. Secure 30% · Resilient 26% · High-performing 24% · Cost-optimized 20%.
**Time plan (about 3 hours, then stop):** Cheat-sheet pass 75 min · Account teardown audit 30 min · Rapid recall 20 min · Logistics 20 min · Rest of the day off.

**No new material today.** Every row below comes from a decision table in Days 1–13. If a row looks unfamiliar, open that day's lesson and read only that row's section. Don't start a new topic.

By the end of today you should be able to:

- Scan the whole syllabus in about 75 minutes and mark the rows you can't explain.
- Settle the ten most-confused pairs in a few seconds each.
- Run the constraint-first reading routine without thinking about it.
- Confirm your AWS account has no billable lab leftovers in any Region.
- Know what to bring, when to arrive or check in, and how to pace 65 questions in 130 minutes.

> **Miss log check:** `miss-log.md` has no entries yet. If you took Exams 1–3, spend 10 minutes now writing your misses there, one line each with a K/Q/C/O tag (see Day 13 §1). Then review those rows first in Parts A–N. If you didn't log them, start with the tables for your lowest Exam 3 domain (see the Day 13 scoring guide).

---

## 0. How to use this sheet (75 minutes)

```
 Pass 1 (45 min)  cover the Pick column → say the answer → uncover
                   ✓ = instant      ? = hesitated      ✗ = wrong
 Pass 2 (20 min)  re-read ONLY ? and ✗ rows; open that day's section if needed
 Pass 3 (10 min)  §15 confused pairs + §16 qualifier reflexes, out loud
 Then stop. Tomorrow morning: §16–§17 only.
```

The day numbers in each heading show where the full explanation lives.

---

## 1. Identity, accounts and governance (Days 1, 10, 13)

| If the question says… | Pick |
| --- | --- |
| App on EC2/Lambda/ECS needs AWS API access | IAM **role** (instance profile / execution role / task role) |
| Access keys in code, config files or an AMI | Always wrong. Replace with a role |
| Stop *everyone* in member accounts, root included, from doing X | **SCP** |
| Developers may create roles but must never exceed certain permissions | **Permissions boundary** |
| Another account needs one S3 bucket | **Bucket policy** with that account as principal (or a cross-account role) |
| Allow access only from my Organization | `aws:PrincipalOrgID` condition |
| Vendor assumes a role in your account | Trust policy with **`sts:ExternalId`** |
| Enforce MFA for sensitive actions | `aws:MultiFactorAuthPresent` condition |
| Permissions for hundreds of projects without writing new policies | **ABAC** (`aws:PrincipalTag` = `aws:ResourceTag`) |
| CI/CD (e.g. GitHub Actions) deploying without stored keys | **OIDC identity provider + role** |
| Workforce SSO into many AWS accounts | **IAM Identity Center** |
| Many accounts with best-practice guardrails, set up fast | **Control Tower** |
| Find resources shared publicly or outside the org | **IAM Access Analyzer** |
| Which permissions has this role actually used? | **Access Advisor** (last accessed) |
| Sign in to AWS apps with on-prem AD, nothing synced | **AD Connector** |
| App users sign up / sign in, MFA, social login | **Cognito user pool** |
| App users get temporary AWS credentials | **Cognito identity pool** |
| Each user may touch only their own S3 prefix / DynamoDB items | Identity pool role with `${cognito-identity.amazonaws.com:sub}` / **`dynamodb:LeadingKeys`** |
| SSO in front of an internal web app, no code changes | **ALB `authenticate-oidc` / `authenticate-cognito`** action |
| Employees reach internal apps without a VPN | **AWS Verified Access** |
| No bucket can ever be public, and admins can't undo that | **Account-level Block Public Access** + **SCP** denying `s3:PutAccountPublicAccessBlock` |
| Download AWS compliance reports for auditors | **AWS Artifact** |

---

## 2. EC2, purchasing and EBS (Day 2)

| If the question says… | Pick |
| --- | --- |
| Steady 24/7 for 1–3 years; may change family, Region, or move to Fargate/Lambda | **Compute Savings Plan** |
| Steady, one family, one Region, maximum discount | **EC2 Instance Savings Plan** or **Standard RI** (3-yr All Upfront) |
| Want RIs but instance type will change | **Convertible RI** |
| Interruptible batch/CI/rendering, lowest cost | **Spot** (diversified, **`price-capacity-optimized`**) |
| Short-term, unpredictable, can't be interrupted | **On-Demand** |
| Guaranteed capacity in one AZ for an event, no long commitment | **On-Demand Capacity Reservation** |
| Discount **and** capacity guarantee | Capacity Reservation + Savings Plan (or **zonal** RI) |
| Bring per-socket / per-core licences | **Dedicated Host** |
| Compliance: don't share hardware with other customers | **Dedicated Instance** |
| HPC / MPI, lowest node-to-node latency | **Cluster** placement group (+ EFA) |
| A few critical instances must not share hardware | **Spread** placement group |
| HDFS / Cassandra / Kafka, rack-aware isolation | **Partition** placement group |
| Fastest temporary storage, cache or scratch | **Instance store** |
| Default volume; tune IOPS without growing size | **gp3** |
| Mission-critical DB, sub-ms, highest IOPS and durability | **io2 Block Express** |
| Big sequential throughput, cheap | **st1** |
| Cold block data, cheapest | **sc1** |
| Shared block volume for a cluster-aware app in one AZ | **io2 Multi-Attach** |
| Move an EBS volume to another AZ | **Snapshot → new volume in the target AZ** |
| Move or DR an instance to another Region | **Copy AMI / snapshot** to that Region |
| Encrypt an existing unencrypted volume | Snapshot → **encrypted copy** → new volume → swap |
| All new volumes must be encrypted | **EBS encryption by default** (per Region) |
| Share an encrypted AMI with another account | **Customer managed KMS key**, share key + AMI |
| Scheduled EBS snapshots, simplest | **Data Lifecycle Manager** (AWS Backup for cross-service) |
| Restored volumes need full performance immediately | **Fast Snapshot Restore** |
| Keep snapshots for years, rare restore | **EBS Snapshots Archive** |
| Recover accidentally deleted snapshots/AMIs | **Recycle Bin** |
| Keep in-memory state across stop/start | **Hibernate** |
| Faster launches | **Golden AMI** (EC2 Image Builder) |
| Public IP must survive stop/start | **Elastic IP** |
| Rightsize from utilisation | **Compute Optimizer** |
| T instance slow under sustained load | Out of CPU credits → M/C family or **unlimited** mode |

---

## 3. Load balancing and Auto Scaling (Days 3, 13)

| If the question says… | Pick |
| --- | --- |
| HTTP(S), route by path or hostname; many microservices behind one entry point | **ALB** listener rules → one target group each |
| TCP/UDP, millions of req/s, ultra-low latency | **NLB** |
| Partners allow-list fixed IPs | **NLB** with Elastic IPs (or Global Accelerator) |
| Static IPs **and** path routing | **NLB → ALB** target, or Global Accelerator → ALB |
| TLS must terminate only on the instances | **NLB TCP:443 passthrough** |
| Third-party firewall appliances inline | **GWLB** + GWLB endpoints |
| Client IP over HTTP / at packet level | **`X-Forwarded-For`** (ALB) / **NLB** source IP preservation |
| HTTP → HTTPS | ALB **redirect** action |
| Many TLS domains on one LB | **SNI** + ACM certificates |
| Block SQLi / XSS at the LB | **AWS WAF** on the ALB |
| Canary / blue-green at the LB | ALB **weighted target groups** |
| Uneven load across AZs | **Cross-zone** load balancing |
| In-flight requests fail on scale-in/deploy | **Deregistration delay** |
| Unhealthy app instances not replaced | ASG health check type **ELB** |
| Keep CPU ~50%, least effort | **Target tracking** |
| Known spike at a known time | **Scheduled** scaling |
| Recurring daily/weekly pattern, scale ahead of it | **Predictive** scaling |
| Instances slow to initialise on scale-out | **Warm pool** (and/or golden AMI) |
| Run a script before termination | **Lifecycle hook** (Terminating:Wait) |
| Roll a new AMI across the fleet, no downtime | **Instance refresh** |
| Cheaper stateless fleet | **Mixed instances policy**: On-Demand base + Spot |
| Workers scale with SQS depth | Target tracking on **backlog per instance** |
| Self-healing single instance | ASG min = max = desired = 1 |
| Survive an AZ loss at peak with no scaling delay | **Static stability**: 3 AZs → provision 1.5 × peak |
| Load balance across Regions | **Route 53** or **Global Accelerator**, not ELB |

---

## 4. VPC and hybrid networking (Days 4, 7, 13)

| If the question says… | Pick |
| --- | --- |
| Private instances need internet for patches | **NAT gateway** in a public subnet, one per AZ for HA |
| IPv6 outbound-only | **Egress-only internet gateway** |
| Private access to S3/DynamoDB from inside the VPC; cut NAT cost | **Gateway endpoint** (+ route table association) |
| Private access to SQS/KMS/Secrets Manager/SSM/ECR; Lambda in VPC without NAT | **Interface endpoint** (PrivateLink) |
| On-prem or peered VPC reaching S3 privately | **S3 interface endpoint** (gateway endpoints don't work from outside the VPC) |
| Bucket accepts requests only from one endpoint/VPC | Bucket policy **Deny** unless `aws:SourceVpce` / `aws:SourceVpc` |
| Stop data leaving to outside buckets via the S3 endpoint | **Endpoint policy** listing approved buckets (or `aws:ResourceOrgID`) |
| Block a specific IP range | **NACL deny** |
| Allow app tier only from the ALB; DB only from the app tier | SG inbound **referencing the ALB SG / app SG** |
| NACL allows 443 in, responses never arrive | NACL **outbound ephemeral ports 1024–65535** |
| Two VPCs, cheap and simple | **VPC peering** (not transitive) |
| Many VPCs + on-prem, central routing and segmentation | **Transit Gateway** (route tables; share via RAM) |
| Offer a service to other VPCs/accounts, overlapping CIDRs | **NLB + PrivateLink endpoint service** |
| Many accounts launch into one central network | **VPC sharing** (RAM) |
| Quick, encrypted, over the internet | **Site-to-Site VPN** |
| Consistent latency, high bandwidth, private | **Direct Connect** |
| DX must be encrypted | **IPsec VPN over DX** or **MACsec** |
| Cheap DX failover | **Site-to-Site VPN backup** |
| Maximum hybrid resilience | **2 DX locations × redundant connections** |
| One DX to VPCs in several Regions | **Direct Connect gateway** |
| More VPN bandwidth than one tunnel | **TGW + ECMP** over several VPNs |
| Remote employees reach VPC resources | **Client VPN** |
| On-prem resolves private hosted zone names | **Route 53 Resolver inbound endpoint** |
| VPC resolves on-prem names | **Resolver outbound endpoint** + forwarding rule |
| Shell access, no bastion, no open ports | **Session Manager** (role `AmazonSSMManagedInstanceCore` + `ssm`, `ssmmessages`, `ec2messages` endpoints if no NAT) or **EC2 Instance Connect Endpoint** |
| Accepted/rejected traffic | **VPC Flow Logs** (query with Athena) |
| Inspect packet payloads | **Traffic Mirroring** |
| Why can't A reach B, without sending traffic | **Reachability Analyzer** |
| Central stateful inspection / domain filtering of egress | **AWS Network Firewall** |
| Block DNS lookups of malicious domains | **Route 53 Resolver DNS Firewall** |

---

## 5. S3, file storage and data transfer (Days 5, 11, 13)

| If the question says… | Pick |
| --- | --- |
| Unknown/changing access, no retrieval fees, no ops | **S3 Intelligent-Tiering** |
| Infrequent, ms retrieval, survives AZ loss | **S3 Standard-IA** |
| Infrequent, re-creatable, cheapest ms access | **S3 One Zone-IA** |
| ~Quarterly access, ms retrieval | **S3 Glacier Instant Retrieval** |
| Minutes to hours is fine | **Glacier Flexible Retrieval** (Expedited 1–5 min) |
| Cheapest, 12–48 h retrieval, multi-year retention | **Glacier Deep Archive** |
| Move objects to cheaper classes as they age | **Lifecycle** transitions |
| Clean up old versions / incomplete uploads | Lifecycle **NoncurrentVersionExpiration** / **AbortIncompleteMultipartUpload** |
| Protect against deletion or overwrite | **Versioning** (+ MFA Delete) |
| WORM, nobody (incl. root) deletes before the date | **Object Lock compliance** |
| WORM, admins can override | **Object Lock governance** + `s3:BypassGovernanceRetention` |
| Copy to another Region / predictable 15-min SLA | **CRR** / + **Replication Time Control** |
| Replicate objects that already exist | **S3 Batch Replication** |
| Aggregate logs from many buckets, one Region | **Same-Region Replication** |
| One global endpoint to the nearest replica bucket | **Multi-Region Access Points** |
| Temporary upload/download without AWS creds; large uploads through an API | **Presigned URL** |
| Private S3 content over HTTPS worldwide | **CloudFront + OAC** |
| Browser on another domain blocked | **CORS** |
| Bucket policy too complex for many teams | **S3 Access Points** |
| Audit every key use | **SSE-KMS** (customer managed key) |
| Cut SSE-KMS cost/throttling | **S3 Bucket Keys** |
| Two layers of encryption mandated | **DSSE-KMS** |
| Downloaders pay the transfer | **Requester Pays** |
| Faster uploads from around the world | **Transfer Acceleration** (+ multipart) |
| Multi-GB upload fails partway | **Multipart upload** |
| SQL directly on S3 files | **Athena** |
| Process each new object | **Event notification → Lambda / SQS / SNS / EventBridge** |
| Shared Linux files across AZs; storage for Lambda/Fargate | **EFS** (access points) |
| Cut EFS cost for cold files / bursty throughput | **EFS lifecycle** → IA/Archive / **Elastic throughput** |
| Windows SMB + Active Directory | **FSx for Windows File Server** (Multi-AZ) |
| HPC/ML, sub-ms, data in S3 | **FSx for Lustre** linked to S3 |
| NFS + SMB + iSCSI or NetApp | **FSx for NetApp ONTAP** |
| Migrate ZFS | **FSx for OpenZFS** |
| On-prem file shares backed by S3, local cache | **S3 File Gateway** |
| On-prem iSCSI: hot data local / all data local | **Volume Gateway cached** / **stored** |
| Replace physical tapes, keep backup software | **Tape Gateway** |
| Online migration/sync of NFS/SMB/HDFS to S3/EFS/FSx | **DataSync** |
| Partners keep SFTP; stop running SFTP servers | **Transfer Family** |
| 100s of TB, little or no bandwidth | **Snowball Edge** (check current availability) |

---

## 6. Databases and caching (Days 6, 13)

| If the question says… | Pick |
| --- | --- |
| Existing MySQL/PostgreSQL/Oracle/SQL Server, minimal changes | **RDS**, same engine |
| Need OS access, Oracle/SQL Server | **RDS Custom** (otherwise EC2) |
| Survive AZ failure, automatic failover | **RDS Multi-AZ** |
| Offload reads / reporting | **Read replicas** (Aurora: reader or **custom endpoint**) |
| Faster failover **and** readable standbys | **Multi-AZ DB cluster** (or Aurora) |
| Cheap relational DR in another Region | **Cross-Region read replica** or snapshot/backup copy |
| Relational, cross-Region RPO ~1 s, RTO ~1 min | **Aurora Global Database** |
| Spiky/intermittent relational load, no capacity planning | **Aurora Serverless v2** |
| Up to 15 replicas, fast failover, auto-growing storage | **Aurora** |
| Undo a bad write on Aurora MySQL in minutes, same cluster | **Backtrack** |
| Restore to a point in time | **PITR** → a *new* instance/cluster; repoint the app |
| Fast cheap copy for testing | **Aurora cloning** |
| Aurora I/O charges dominate | **Aurora I/O-Optimized** |
| Encrypt an existing RDS DB | Snapshot → **encrypted copy** → restore |
| Rotate DB credentials | **Secrets Manager** |
| DB login with IAM, no password | **IAM database authentication** |
| Lambda exhausting connections / smoother failover | **RDS Proxy** |
| Major version upgrade, minimal downtime | **RDS Blue/Green Deployments** |
| RDS MySQL → Aurora MySQL, minimal downtime | **Aurora read replica**, then promote |
| Key-value, massive scale, single-digit ms, serverless | **DynamoDB** |
| DynamoDB: spiky/unknown vs steady traffic | **On-demand** vs **provisioned + auto scaling** (+ reserved capacity) |
| Query by another attribute on an existing table | **GSI** |
| Microsecond reads, minimal code change | **DAX** |
| React to every item change | **DynamoDB Streams + Lambda** |
| Expire items automatically | **TTL** |
| Multi-Region active-active | **DynamoDB global tables** |
| Restore to any second (last 35 days) | **DynamoDB PITR** |
| Storage-heavy, rarely read table | **Standard-IA table class** |
| Cache query results with HA; sessions must survive node failure | **ElastiCache for Valkey / Redis OSS** with Multi-AZ auto-failover |
| Simplest multi-threaded cache, loss OK | **ElastiCache for Memcached** |
| Cache never stale after writes | **Write-through** |
| Leaderboard | Valkey/Redis OSS **sorted sets** |
| Sessions for a stateless web tier | **ElastiCache** or **DynamoDB** (TTL) |
| In-memory speed, durable primary DB | **MemoryDB** |
| Warehouse, complex SQL, TB–PB, BI | **Redshift** |
| Analytics on transactional data without ETL | **Zero-ETL** (Aurora/RDS/DynamoDB → Redshift) |
| MongoDB / graph / Cassandra | **DocumentDB** / **Neptune** / **Keyspaces** |

---

## 7. Serverless, messaging and streaming (Days 8, 13)

| If the question says… | Pick |
| --- | --- |
| Decouple tiers, absorb spikes | **SQS Standard** (+ ASG on queue depth) |
| Order + no duplicates | **SQS FIFO** (message group ID for parallelism) |
| Messages processed twice | Raise **visibility timeout** above processing time |
| Messages lost after a long consumer outage | Raise **retention** (default 4 days, max 14) |
| Poison message | **DLQ** + redrive policy |
| Empty receives / SQS cost | **Long polling** (≤ 20 s) |
| Payload bigger than SQS allows | **S3 + pointer** (Extended Client Library) |
| One event → several independent consumers | **SNS → multiple SQS** (fan-out) |
| Subscribers want only some messages | **SNS filter policy** (or EventBridge rules) |
| Email/SMS/mobile push | **SNS** |
| React to AWS service events | **EventBridge rule** |
| SaaS partner events, content routing, archive/replay | **EventBridge** (partner source, archive & replay) |
| Serverless schedule | **EventBridge Scheduler** |
| Real-time stream, per-key order, many consumers, replay | **Kinesis Data Streams** (on-demand = no shard management) |
| Stream into S3/Redshift/OpenSearch/Splunk, no code | **Amazon Data Firehose** (+ record format conversion to Parquet) |
| Windowed real-time analytics | **Managed Service for Apache Flink** |
| Existing Kafka | **Amazon MSK** (MSK Serverless for least ops) |
| JMS/AMQP/MQTT/RabbitMQ, no code change | **Amazon MQ** |
| Short event-driven compute, pay per use | **Lambda** |
| Longer than 15 min / needs GPU | **Fargate / AWS Batch / ECS on EC2** (Step Functions to orchestrate) |
| Cold starts | **Provisioned concurrency** (scheduled for a known spike); Java: **SnapStart** |
| Protect a downstream that allows only N calls | **Reserved concurrency = N** + SQS buffer (or event source max concurrency) |
| One bad message fails a whole batch | **`ReportBatchItemFailures`** |
| Lambda in a VPC needs internet | Private subnets + **NAT gateway** |
| REST API with caching, API keys, usage plans, WAF | **API Gateway REST API** |
| Cheap Lambda/HTTP proxy with JWT auth | **API Gateway HTTP API** |
| Real-time two-way / push to browsers | **WebSocket API** (AppSync for GraphQL subscriptions) |
| Internal-only API | **Private REST API** + interface endpoint |
| API that only writes to SQS/DynamoDB | **AWS service integration**, no Lambda |
| API validates user tokens | **Cognito user pool authorizer** |
| Multi-step workflow, retries, branching, human approval | **Step Functions Standard** (`.waitForTaskToken`) |
| High-volume short workflows | **Step Functions Express** |
| Millions of S3 objects in parallel | **Distributed Map** |
| GraphQL | **AppSync** |

---

## 8. Containers, DNS and edge (Day 9)

| If the question says… | Pick |
| --- | --- |
| Containers, no servers, least ops | **ECS on Fargate** |
| Already on Kubernetes / portability | **EKS** (Fargate or Auto Mode for least ops) |
| GPUs / specific instance types / high steady use | **ECS or EKS on EC2** with capacity providers |
| Cheapest interruptible tasks | **Fargate Spot** |
| Container calls S3/DynamoDB | **Task role** (EKS: Pod Identity / IRSA) |
| Can't pull image or read secret at startup | **Task execution role** / endpoints |
| Fargate private, no NAT, pull from ECR | **ecr.api + ecr.dkr interface endpoints + S3 gateway endpoint** (+ logs) |
| Shared storage across AZs | **EFS** volume |
| Inject a DB password | Task definition `secrets` from **Secrets Manager / Parameter Store** |
| Auto-rollback a failed ECS deployment | **Deployment circuit breaker** with rollback |
| Big batch jobs with queues, retries, Spot | **AWS Batch** |
| Apex domain → ALB/CloudFront | **Route 53 alias A record** |
| 10% to a new version via DNS | **Weighted** |
| Lowest latency across Regions | **Latency** |
| Country must go to a specific Region | **Geolocation** (+ default record) |
| Shift geographic share with a bias | **Geoproximity** |
| Active-passive at DNS | **Failover** + health check |
| Health-check a private resource | **CloudWatch alarm–based health check** |
| Several healthy IPs, no LB | **Multivalue answer** |
| Route by client IP range | **IP-based** routing |
| Controlled multi-Region switch-over | **Route 53 ARC** routing controls |
| Global users, slow content, reduce origin load | **CloudFront** |
| Private bucket only via CloudFront | **OAC** + bucket policy (`AWS:SourceArn`); + KMS key policy if SSE-KMS |
| Users must not bypass CloudFront to the ALB | **VPC origins**, or SG with **CloudFront prefix list** + secret header |
| Premium: one file / many files | **Signed URL** / **signed cookies** |
| Block countries | **CloudFront geo restriction** (WAF geo match at ALB) |
| Protect one form field end to end | **Field-level encryption** |
| Light header/URL rewrite, cheapest | **CloudFront Functions** |
| Edge logic needing network/body/origin events | **Lambda@Edge** |
| Static content HA across Regions | **Origin group** (S3 + CRR replica) |
| Better cache hit ratio | Trim cache key, **Origin Shield**, longer TTLs, versioned names |
| Cheaper CloudFront, users mostly NA/EU | **Price Class 100** |
| CloudFront custom-domain certificate | **ACM in `us-east-1`** |
| Static IPs + multi-Region; UDP gaming/IoT, fast failover | **Global Accelerator** |

---

## 9. Encryption and security services (Days 10, 13)

| If the question says… | Pick |
| --- | --- |
| Full control of key policy, audit, disable | **KMS customer managed key** |
| Encrypt with least effort | AWS managed key / service default |
| Encrypt large data with KMS | **Envelope encryption** (`GenerateDataKey`) |
| Rotate yearly, no app change | **Automatic rotation** on a CMK |
| Decrypt in another Region without re-encrypting | **Multi-Region keys** |
| Single-tenant HSM, PKCS#11 | **CloudHSM** (≥ 2 AZs) |
| AWS integration but keys in your HSMs | **KMS custom key store (CloudHSM)** |
| Key material outside AWS | **External key store** |
| Key usable only via one service | **`kms:ViaService`** |
| Free public TLS with auto-renewal | **ACM** |
| Internal TLS / mTLS | **AWS Private CA** |
| Alert before an imported cert expires | ACM expiry events → **EventBridge** / Config rule |
| Secret replicated to DR Region | **Secrets Manager replication** |
| Config + encrypted strings, free, no rotation | **Parameter Store** SecureString |
| SQLi, XSS, bots | **AWS WAF** (managed rules, Bot Control) |
| Throttle noisy IPs | **WAF rate-based rule** |
| Basic DDoS, free | **Shield Standard** + CloudFront |
| DDoS response team, cost protection | **Shield Advanced** |
| Same WAF/SG/Shield policy in every account, incl. new ones | **Firewall Manager** |
| Compromised instances, crypto-mining, odd API calls | **GuardDuty** |
| Scan S3 uploads for malware | **GuardDuty Malware Protection for S3** |
| CVE scanning of EC2, ECR, Lambda | **Inspector** |
| PII in S3 | **Macie** |
| Aggregate findings + CIS/FSBP/PCI checks | **Security Hub** |
| Root cause of a finding | **Detective** |
| Central security data lake, standard schema | **Security Lake** |
| Auto-remediate a finding | **EventBridge → Lambda / SSM Automation** |
| Patch OS fleets on a schedule | **SSM Patch Manager** + maintenance windows |

---

## 10. DR, backup and migration (Days 7, 11, 13)

```
 cost ▲                                               multi-site active/active
      │                                    warm standby      (RPO/RTO ≈ 0)
      │                     pilot light      (RTO minutes)
      │  backup & restore   (RPO minutes,
      │  (RPO/RTO hours)     RTO tens of min)
      └──────────────────────────────────────────────────────────► faster recovery
```

| If the question says… | Pick |
| --- | --- |
| RPO/RTO hours, lowest cost | **Backup & restore** (Backup cross-Region copy, AMI/snapshot copy, IaC) |
| Core data running, launch app servers on failover | **Pilot light** |
| Scaled-down full copy always running; RTO minutes, RPO seconds, cheapest | **Warm standby** |
| Near-zero RPO/RTO, serve from several Regions | **Multi-site active/active** |
| On-prem servers DR to AWS, RPO seconds, RTO minutes | **Elastic Disaster Recovery** |
| Cheap Regional DR for EC2, RTO hours | Copy **AMIs/snapshots** (DLM or Backup); launch on demand |
| Controlled Region failover with readiness checks | **Application Recovery Controller** + Route 53 |
| Central backups across services by tag | **AWS Backup** plan, tag-based assignment |
| Enforce backups across the org | **Backup policies** (Organizations) |
| Immutable / ransomware-proof / survives account compromise | **Vault Lock (compliance)** + **cross-account copy** |
| Prove backup compliance | **Backup Audit Manager** |
| Lift-and-shift many servers, minimal downtime | **Application Migration Service (MGN)** |
| Discover servers and dependencies | **Application Discovery Service** |
| Same-engine DB migration, source stays online | **DMS** full load + CDC |
| Oracle/SQL Server → Aurora | **DMS Schema Conversion (or SCT) + DMS** |
| Edge compute at a disconnected site | **Snowball Edge Compute Optimized** |

---

## 11. Monitoring, audit and compliance (Day 11)

| If the question says… | Pick |
| --- | --- |
| EC2 memory/disk metrics | **CloudWatch agent** |
| 1-minute EC2 metrics | **Detailed monitoring** |
| Log pattern → alarm | **Metric filter** + alarm |
| Query logs interactively | **Logs Insights** |
| Stream logs to S3/OpenSearch/another account | **Subscription filter** → Firehose / Kinesis / Lambda |
| Alarm when two conditions are both true | **Composite alarm** |
| Hardware failure, keep IP | Alarm → **EC2 recover** |
| Test the login flow every 5 min | **Synthetics canary** |
| Find the slow microservice | **X-Ray** |
| Logs kept forever by accident | **Log group retention** |
| Who called this API / deleted this resource | **CloudTrail** |
| Who read/deleted specific S3 objects | **CloudTrail data events** |
| > 90 days, all accounts and Regions | **Organization trail** → S3 in the log archive account |
| Prove logs weren't tampered with | **Log file integrity validation** |
| Unusual API volume | **CloudTrail Insights** |
| SQL over CloudTrail without building tables | **CloudTrail Lake** (or Athena on the trail) |
| Config history / was this ever non-compliant | **AWS Config** |
| Detect + auto-fix non-compliance | **Config rule + SSM Automation** |
| Org-wide compliance view | **Config aggregator** (+ conformance packs) |
| Notified about AWS maintenance on my resources | **AWS Health** (+ EventBridge) |

---

## 12. Analytics (Day 12)

| If the question says… | Pick |
| --- | --- |
| Ad hoc SQL on S3, serverless | **Athena** |
| Cheaper/faster Athena | **Parquet/ORC + partitions + compression**; CTAS |
| Cap each team's Athena scan | **Workgroups** with data usage limits |
| SQL over DynamoDB/RDS/on-prem without moving data | **Athena federated query** |
| PB warehouse, many concurrent BI users | **Redshift** (+ concurrency scaling) |
| Intermittent warehouse use, no cluster ops | **Redshift Serverless** |
| Join Redshift with rarely used history in S3 | **Redshift Spectrum** |
| Share live Redshift data, no copies | **Redshift data sharing** |
| Central schema repository | **Glue Data Catalog** |
| Discover schema and new partitions | **Glue crawler** |
| Serverless ETL, process only new data | **Glue ETL** + **job bookmarks** |
| Visual no-code data prep | **Glue DataBrew** |
| Row/column permissions; cross-account table sharing | **Lake Formation** (LF-Tags) |
| Migrate Hadoop/Spark, full control | **EMR** (Spot task nodes, transient clusters) |
| Spark/Hive without clusters | **EMR Serverless** (or Glue) |
| Full-text search, log dashboards | **OpenSearch Service** (Serverless for least ops) |
| BI dashboards, row-level security | **QuickSight** (SPICE) |

---

## 13. Cost tools (Days 12, 13)

| If the question says… | Pick |
| --- | --- |
| Estimate before building | **Pricing Calculator** |
| Alert on actual/forecast spend | **AWS Budgets** |
| Stop instances / apply SCP when over budget | **Budget actions** |
| Spend spikes without fixed thresholds | **Cost Anomaly Detection** |
| Spend by service/account/tag; SP/RI recommendations | **Cost Explorer** |
| Hourly line-item billing in SQL | **Data Exports (CUR 2.0) → S3 → Athena** |
| Cost per department/project | **Cost allocation tags** (activate in payer) / **Cost Categories** |
| Rightsize EC2/EBS/Lambda/ASG | **Compute Optimizer** |
| Idle LBs, unattached EIPs, open SGs, quotas in one list | **Trusted Advisor** |
| Cut NAT cost from S3/DynamoDB traffic | **Gateway endpoints** |
| Steady base + interruptible burst | **Savings Plan** base + **Spot** in a mixed instances policy |

---

## 14. Cross-service patterns (Day 7)

| If the question says… | Pick |
| --- | --- |
| Private EC2, no NAT, reads S3 | **Role** + **S3 gateway endpoint** + route table association |
| HA web app, relational, minimal management | **ALB + ASG (≥ 2 AZs) + RDS Multi-AZ** (or Aurora) |
| EC2 → RDS without stored password | **IAM DB auth** with the instance role |
| Users lose sessions on scale-in | Sessions in **ElastiCache** or **DynamoDB** |
| Share encrypted AMI/snapshot cross-account | **CMK** shared via key policy + share the AMI/snapshot |
| Cross-Region DB DR, RPO seconds | **Aurora Global Database** / **DynamoDB global tables** |

---

## 15. The most-confused pairs: one discriminator each (Days 7, 13)

| Pair | Decides it |
| --- | --- |
| RDS Multi-AZ vs read replica | Sync + auto-failover (HA) vs async + readable (scale reads, cross-Region DR) |
| Multi-AZ instance vs Multi-AZ DB cluster | Cluster gives readable standbys + faster failover |
| Gateway vs interface endpoint | S3/DynamoDB from inside the VPC (free, route table) vs everything else or from outside the VPC (ENI, billed) |
| SG vs NACL | Stateful allow-only, instance vs stateless allow+deny, subnet |
| SCP vs permissions boundary | Whole account incl. root vs one user/role |
| ALB vs NLB | L7 paths/hosts/WAF/auth vs L4 TCP/UDP/static IP/PrivateLink |
| EFS vs FSx | Linux NFS shared across AZs vs Windows SMB/AD, Lustre HPC, ONTAP multi-protocol |
| DAX vs ElastiCache | DynamoDB-only, no code change vs any source, sessions, leaderboards |
| GSI vs LSI | Add any time, new partition key vs creation-only, same partition key, strong consistency |
| Savings Plans vs RIs | Spend commitment, flexible vs instance attributes, zonal RI = capacity |
| Standard-IA vs Intelligent-Tiering | Known infrequent vs unknown pattern |
| Spread vs partition placement | Individual instances vs groups on separate racks |
| Object Lock governance vs compliance | Bypassable vs not, even by root |
| Aurora Global DB vs DynamoDB global tables | Relational, one writer vs key-value, multi-writer |
| File Gateway vs DataSync | Ongoing hybrid access vs migrate/sync |
| Backtrack vs PITR | Rewind in place (Aurora MySQL) vs restore to a new DB |
| Resolver inbound vs outbound | On-prem → AWS names vs AWS → on-prem names |
| Signed URL vs signed cookies | One file vs many files |
| Task role vs task execution role | App's AWS calls vs pulling image, logs, secrets at start |
| Kinesis Data Streams vs Data Firehose | Custom consumers, replay vs managed delivery to a destination |
| SQS vs SNS vs EventBridge | Queue (pull, buffer) vs push fan-out vs event routing with rules, SaaS, replay |
| Secrets Manager vs Parameter Store | Rotation, replication vs cheap config with no rotation |
| Shield Standard vs Advanced | Free automatic vs DRT, cost protection |
| GuardDuty vs Inspector vs Macie | Threats vs vulnerabilities vs sensitive data |
| CloudTrail vs Config | Who did what vs what the resource looked like and whether it complied |
| DRS vs MGN | Ongoing DR vs one-way migration |
| Pilot light vs warm standby | Data only, start servers on failover vs everything running at small scale |

---

## 16. Reading routine and qualifier reflexes (Days 7, 13)

**On every question:**

1. Read the **last sentence first**: the qualifier, and how many answers to pick.
2. Write the hard constraints as single words: *on-prem*, *SMB*, *RPO 5 s*, *no code change*, *Kafka*.
3. Cross out every option that breaks a constraint. You're usually left with two.
4. Compare the last two **only** on the qualifier.
5. Change an answer only if a word in the stem proves the first pick wrong.

| Qualifier | Reflex |
| --- | --- |
| *LEAST operational overhead* | Managed/serverless over EC2; built-in feature over custom Lambda |
| *MOST cost-effective* | Smallest thing that meets every requirement: Spot, SP, lifecycle, gateway endpoint |
| *Highly available* | ≥ 2 AZs |
| *Fault tolerant / no data loss* | Synchronous replication, durable storage |
| *Minimal changes / lift-and-shift* | Same engine, same protocol, rehost |
| *Most secure* | Private subnets + endpoints, roles, CMK, least privilege, deny conditions |
| *Real-time / lowest latency* | Cache, instance store, io2, cluster placement, Global Accelerator |

**Choose two:** the right pair is usually two halves that work together (a control plus a guard on it, a buffer plus a limiter), not two ways to do the same thing.

**Classic traps:** gateway endpoints from on-prem · Lambda over 15 minutes · ACM certificate for CloudFront outside `us-east-1` · NACL missing ephemeral ports · "read replica" for automatic HA · Multi-AZ standby for reads (instance deployment) · SCPs granting permissions (they only cap them) · Glacier Flexible when "milliseconds" is in the stem · access keys anywhere.

---

## 17. Exam-day logistics — Friday 2026-11-13

Policies change. Check your AWS Certification account and the Pearson VUE confirmation email for current rules on ID, check-in windows, breaks and rescheduling. Where this section says "check", don't rely on memory.

### Tonight (Day 14)

- [ ] Open the confirmation email. Note the **exact time (IST)**, **delivery mode** (test centre or online/OnVUE) and, for a centre, the address and travel time.
- [ ] **ID:** a valid government-issued photo ID with a signature. The name must match your certification account exactly. Check the current ID rules for India.
- [ ] **Language accommodation:** if English isn't your first language, AWS has offered an **ESL +30 minutes** accommodation. It usually has to be on your account *before* the booking. Check today whether you can still add it to tomorrow's appointment.
- [ ] **Online only:** run the Pearson VUE **system test** on the same computer, network and room you'll use tomorrow. Close VPNs and corporate security agents that could block the browser. Have a phone hotspot ready as a backup connection.
- [ ] **Online only:** clear the desk and the wall behind you. No second monitor (or unplug it), no papers, no watch, no headphones.
- [ ] Set two alarms. Decide what you'll eat and when.
- [ ] **No new questions after 8 pm.** Read §15 and §16 once, then stop.

### Tomorrow morning

```
 T−3 h   light breakfast · read §16 only (5 min) · no new practice questions
 T−60    leave for the centre (allow for traffic)      │  online: restroom, water, desk clear
 T−30    arrive / start online check-in                │  photos of ID + room, face photo
 T−15    lockers (centre) · proctor greeting (online)  │  phone out of reach
 T−0     NDA screen → 65 questions, 130 min (+30 if ESL accommodation is applied)
```

- **Test centre:** arrive early (check how early your centre asks). Everything goes in a locker. You'll get a note board or scratch materials.
- **Online:** check-in usually opens about 30 minutes before. Use a built-in whiteboard instead of paper. Stay in camera view, don't read questions aloud, and don't look away from the screen for long. Unscheduled breaks may end the exam, so check the current break policy.

### Pacing (130 minutes, 65 questions)

| Clock (elapsed) | Be at question | Notes |
| --- | --- | --- |
| 0:35 | ~20 | First pass ≈ 1.6–1.8 min each |
| 1:10 | ~40 | Flag anything past 3 min. Pick your best guess first, *then* flag |
| 1:50 | 65 | First pass done with ~20 min in hand |
| 1:50–2:05 | flagged ones | Change only with a stem word as proof |
| 2:05–2:10 | — | Make sure nothing is blank. No penalty for guessing |

- About 15 of the 65 questions are unscored, and you can't tell which ones. A strange question may be one of them, so don't let it cost you time.
- Multiple-response questions tell you how many answers to pick. Pick exactly that many.
- If you're stuck, reset with §16 step 2: list the constraints, then eliminate.

### After you submit

- You'll answer a short survey. Then you may see a provisional pass/fail on screen.
- The official result and score report arrive in your AWS Certification account, usually within a few business days.
- **Passed:** the certification is valid for 3 years. Check the benefits page for current perks, such as a discount on your next exam.
- **Didn't pass:** the score report breaks your results down by domain. There's a waiting period before a retake (check current policy). Use the domain breakdown and the Days 7/12/13 scoring guides to pick the 2–3 days to redo.

---

## 18. Teardown audit (30 min, free) — make sure nothing from Days 1–13 is still billing

There's no new lab today. Instead, check that every lab resource is gone in **every Region**, because a leftover NAT gateway or Elastic IP keeps billing after the exam.

### Console

1. **Billing and Cost Management → Bills**: look at this month's charges by service. Any line item that isn't zero is a lead.
2. **Resource Explorer** (turn it on with the aggregator index if prompted) or **Tag Editor**: search *All Regions*, all resource types.
3. **EC2 Global View**: instances, volumes, Elastic IPs, security groups in every Region.
4. Check by hand the services that bill while idle: **NAT gateways**, **Elastic IPs** (all public IPv4 addresses bill now), **load balancers**, **interface endpoints**, **RDS/Aurora** instances and snapshots, **Redshift Serverless** workgroups, **Kinesis** streams, **OpenSearch** domains, **KMS** customer managed keys (schedule deletion), **Secrets Manager** secrets, **Direct Connect / VPN** connections, **Transit Gateway** attachments.
5. **S3**: empty and delete lab buckets, including versions and delete markers. Buckets with Object Lock in governance mode need the bypass, as on Day 13.
6. **Security services with trials** (GuardDuty, Inspector, Macie, Security Hub, Config recorder, Detective): turn off the ones you don't want once the free trials end.
7. **Budgets**: keep (or create) a **zero-spend budget** with email alerts, so you'll see any leftover charge.

### CLI sweep (read-only)

```bash
for r in $(aws ec2 describe-regions --query 'Regions[].RegionName' --output text); do
  echo "== $r"
  aws ec2 describe-instances --region "$r" \
    --filters Name=instance-state-name,Values=pending,running,stopped \
    --query 'Reservations[].Instances[].InstanceId' --output text
  aws ec2 describe-nat-gateways --region "$r" --filter Name=state,Values=available \
    --query 'NatGateways[].NatGatewayId' --output text
  aws ec2 describe-addresses --region "$r" --query 'Addresses[].PublicIp' --output text
  aws ec2 describe-volumes --region "$r" --query 'Volumes[].VolumeId' --output text
  aws elbv2 describe-load-balancers --region "$r" --query 'LoadBalancers[].LoadBalancerName' --output text
  aws ec2 describe-vpc-endpoints --region "$r" --query 'VpcEndpoints[].VpcEndpointId' --output text
  aws rds describe-db-instances --region "$r" --query 'DBInstances[].DBInstanceIdentifier' --output text
done
aws s3 ls                      # buckets are global in the listing
```

Anything listed that you created during the course: delete it, then run the sweep again until it's clean (the default VPC's resources are fine).

### Terraform variant

```bash
# In each lab directory you used on Days 1–13
for d in ~/aws-saa-labs/day-*/; do
  echo "== $d"; (cd "$d" && terraform state list 2>/dev/null)
done
# Anything listed → cd into that directory and:
terraform destroy
```

`terraform state list` printing nothing means that lab is gone. Versioned or Object Lock buckets may block `destroy`. Empty them first, as in that day's teardown.

**Teardown done when:** the sweep prints no lab resources, every lab's `terraform state list` is empty, and the bill page shows nothing new accruing.

---

## 19. Rapid recall — 10 questions from earlier days (10 min)

Untimed. These repeat earlier decision rows, phrased the way the exam phrases them.

1. A company must let EC2 instances in a private subnet with no NAT read from S3, and nothing may cross the internet. What should it do, at the lowest cost?
2. An on-premises data centre connects over Direct Connect and must reach S3 privately. Which endpoint type works?
3. A web app's NACL allows inbound TCP 443, the security group allows 443, but clients time out. What is missing?
4. An RDS MySQL database must survive an AZ failure with automatic failover and no data loss. Which option?
5. A workload's access pattern is unknown and changes over time. The company wants the lowest storage cost with no retrieval fees and no lifecycle tuning. Which storage class?
6. A Lambda function overwhelms a third-party API that allows only 10 concurrent calls. With the least effort, what should be configured?
7. Users must reach a private S3 bucket only through CloudFront. What should be configured?
8. A company needs RPO in seconds and RTO in minutes for a Regional DR at the lowest cost that still meets this. Which strategy?
9. Auditors require proof of who deleted specific S3 objects last month. Which feature should have been enabled?
10. Steady compute for three years. The company may move from EC2 to Fargate partway through. Which purchase option?

<details>
<summary>Answers</summary>

1. **S3 gateway endpoint** + route table association, with an instance role. It's free and keeps traffic off the internet (Days 4, 7).
2. **S3 interface endpoint.** Gateway endpoints can't be reached from outside the VPC (Days 4, 13).
3. **NACL outbound ephemeral ports 1024–65535.** NACLs are stateless, so the replies need their own rule (Day 4).
4. **RDS Multi-AZ.** Synchronous standby with automatic failover. A read replica is asynchronous (Day 6).
5. **S3 Intelligent-Tiering.** It handles unknown patterns and has no retrieval fees (Day 5).
6. **Reserved concurrency = 10**, ideally with SQS buffering in front (Days 8, 13).
7. **CloudFront OAC** + a bucket policy that allows the CloudFront service principal with `AWS:SourceArn` = the distribution (Day 9).
8. **Warm standby.** Pilot light has a longer RTO, and active/active costs more (Days 11, 13).
9. **CloudTrail data events for S3.** Management events don't record object-level calls (Day 11).
10. **Compute Savings Plan.** It covers EC2, Fargate and Lambda (Day 2).

</details>

---

## 20. Flashcards — the last-night set (cover the right column)

| Prompt | Answer |
| --- | --- |
| Exam format | 65 questions (50 scored), 130 min, pass 720/1000 |
| Domain weights | Secure 30 · Resilient 26 · Performing 24 · Cost 20 |
| "Highly available" means | ≥ 2 AZs |
| Can SCPs grant permissions? | No. They only cap them (root included) |
| Gateway endpoints exist for | S3 and DynamoDB only |
| NACL reply ports | 1024–65535 outbound |
| ACM Region for CloudFront | `us-east-1` |
| Lambda max duration | 15 min |
| SQS retention default / max | 4 / 14 days |
| Messages processed twice | Visibility timeout too short |
| Sync HA for RDS | Multi-AZ |
| Rewind Aurora MySQL in place | Backtrack |
| Microsecond DynamoDB reads | DAX |
| Multi-writer multi-Region NoSQL | DynamoDB global tables |
| Unknown access pattern | Intelligent-Tiering |
| WORM, nobody can delete | Object Lock compliance |
| Windows + AD file shares | FSx for Windows File Server |
| HPC with S3 data | FSx for Lustre |
| Static IPs + global | Global Accelerator |
| Many premium files | CloudFront signed cookies |
| Threats / CVEs / PII | GuardDuty / Inspector / Macie |
| Who did it / what it looked like | CloudTrail / Config |
| Cheapest DR | Backup & restore |
| On-prem servers DR, RPO seconds | Elastic Disaster Recovery |
| Flexible compute commitment | Compute Savings Plan |
| Spot allocation strategy | `price-capacity-optimized` |

---

## 21. Before you close today

- [ ] Misses from Exams 1–3 logged in `miss-log.md` (it's still empty) and the go/no-go line written at the top
- [ ] §§1–14 Pass 1 done; every ? and ✗ row re-read in Pass 2
- [ ] §15 pairs and §16 reflexes said out loud once
- [ ] Teardown audit clean in every Region; zero-spend budget in place
- [ ] Rapid recall: 10/10, or the misses re-read
- [ ] Confirmation email checked: time (IST), mode, address or system test, ID name matches
- [ ] ESL +30 accommodation checked, if it applies to you
- [ ] Bag or desk ready; alarms set; nothing new after 8 pm; in bed on time

**Tomorrow:** exam day, 2026-11-13. Read §16 at breakfast, trust your first pass, and flag and move on past 3 minutes.
