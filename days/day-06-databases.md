# Day 6 — RDS, Aurora, DynamoDB, ElastiCache & Redshift

**Domain weight:** heavy in *Design Resilient Architectures* (26%) — Multi-AZ vs read replicas vs Global Database is a top-five exam topic — and *Design High-Performing Architectures* (24%) for picking the right engine, caching and scaling reads. Also *Design Secure Architectures* (30%) for encryption, IAM auth and Secrets Manager, and *Design Cost-Optimized Architectures* (20%) for capacity modes, Serverless and Reserved Instances.
**Time plan:** Learn 90 min · Lab 30 min · Drill 45 min · Review 30 min.

By the end of today you should be able to:

- Classify a workload as relational OLTP, key-value, in-memory, analytics (OLAP), document, graph or time-series, and name the AWS database that fits.
- Explain the difference between RDS Multi-AZ (instance and DB cluster), read replicas and cross-Region replicas — what each protects against and what each does *not* do.
- Explain how Aurora's shared storage works and use it to answer failover-time, replica-count and Global Database questions; know when Aurora Serverless v2 is the answer.
- Design a DynamoDB table: keys, capacity mode, GSI vs LSI, DAX, Streams, TTL, Global Tables and backups.
- Pick a caching strategy (lazy loading vs write-through, TTL) and choose between ElastiCache for Valkey/Redis OSS, Memcached and MemoryDB.
- Recognise when Redshift (or Redshift Serverless / Spectrum) is the answer, and when it is a trap.

---

## 1. Choosing a database: the first decision

Most database questions are decided in the first sentence. Read for the **data shape** and **access pattern** before the qualifier.

```
                     ┌──────────── What does the workload look like? ────────────┐
                     │                                                            │
  Relational, joins, ACID,           Key-value / document,        Analytics over TBs–PBs,
  existing SQL app (OLTP)            single-digit ms at any       complex aggregations (OLAP)
           │                         scale, simple key lookups              │
           ▼                                   │                            ▼
   RDS  or  Aurora                             ▼                        Redshift
   (lift-and-shift → RDS;              DynamoDB                  (ad hoc SQL on S3 → Athena, Day 12)
    cloud-native / HA / scale → Aurora)        │
                                        microsecond reads? → DAX
           │
   Repeated reads of the same data, sub-ms, sessions, leaderboards → ElastiCache (Valkey / Redis OSS / Memcached)
   Need that in-memory speed but as a *durable primary* database → MemoryDB
```

| Data / need | Service | Why |
| --- | --- | --- |
| Relational OLTP, standard engine, minimal change from on-prem | **Amazon RDS** | Managed MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, Db2 |
| Relational, higher performance/availability, many read replicas, fast failover, cross-Region | **Amazon Aurora** | MySQL/PostgreSQL-compatible, cloud-native storage |
| Key-value or document, massive scale, serverless, single-digit ms | **Amazon DynamoDB** | Fully managed, no servers, scales horizontally |
| In-memory cache / session store | **Amazon ElastiCache** | Sub-millisecond reads, offloads the database |
| Data warehouse, BI, columnar analytics | **Amazon Redshift** | MPP columnar SQL |
| MongoDB-compatible documents | **Amazon DocumentDB** | Managed, MongoDB API compatible |
| Graph (social networks, fraud rings, recommendations) | **Amazon Neptune** | Gremlin / openCypher / SPARQL |
| Apache Cassandra (CQL) workloads | **Amazon Keyspaces** | Serverless Cassandra-compatible |
| Time-series (IoT metrics, telemetry) | **Amazon Timestream** | Purpose-built time-series (check current docs for which Timestream offerings are open to new customers) |
| Full-text search, log analytics | **Amazon OpenSearch Service** | Search indexes, dashboards |
| Immutable, verifiable ledger | Historically **QLDB** — now discontinued; AWS points to Aurora PostgreSQL for ledger use cases. Older exam questions may still name QLDB. |

> Trap: "**relational**" + "**unpredictable traffic**" + "**least operational overhead**" → Aurora Serverless v2, not DynamoDB. DynamoDB is only right if the data model is key-value and the app can change. "**Minimal code changes**" for an existing MySQL/PostgreSQL app → RDS or Aurora, never DynamoDB.

---

## 2. Amazon RDS fundamentals

RDS runs a database engine on a managed instance. AWS handles provisioning, OS and engine patching (in your maintenance window), backups, monitoring and failover. You still choose the **instance class**, **storage**, **engine version**, **parameter groups** and **network placement**.

**What you cannot do on standard RDS:** SSH to the host or install OS-level agents. If a scenario needs OS access or custom software on Oracle/SQL Server, the answer is **RDS Custom**; otherwise, self-managed on EC2.

### Networking

```
VPC
├── DB subnet group (subnets in ≥ 2 AZs — required, even for Single-AZ)
│     ├── private subnet AZ-a ── primary
│     └── private subnet AZ-b ── standby (Multi-AZ)
└── Security group on the DB: inbound 3306/5432 ONLY from the app tier's SG
```

- Put databases in **private subnets**; "Publicly accessible: No".
- Security groups reference the **app SG**, not CIDRs — the classic exam answer for "allow only the web tier to reach the database".
- Apps connect to the **endpoint DNS name**, never an IP: failover changes the IP behind the name.

### Storage

| Type | Use |
| --- | --- |
| General Purpose SSD (gp3) | Default; baseline IOPS/throughput, adjustable independently of size |
| Provisioned IOPS SSD (io1/io2 Block Express) | I/O-intensive OLTP with consistent latency |
| Magnetic | Legacy — never the answer |

- **Storage autoscaling** grows storage automatically when free space is low (up to a max you set). Answer for "database occasionally runs out of storage, minimise admin."
- Storage can grow but **cannot shrink** in place.

### Backups and restore

| Feature | Detail |
| --- | --- |
| **Automated backups** | Daily snapshot + transaction logs. Retention 1–35 days (0 disables). Enables **point-in-time restore** (PITR) to any second within retention, typically up to the last ~5 minutes. Deleted with the instance unless you retain them. |
| **Manual snapshots** | Kept until **you** delete them. Use for long-term retention beyond 35 days, or before risky changes. |
| **Restore** | Always creates a **new** DB instance with a new endpoint. You can't restore "into" an existing instance. |
| **Cross-Region** | Copy snapshots to another Region, or enable cross-Region automated backup replication — for DR. |
| **AWS Backup** | Central policy-based backups across RDS, Aurora, DynamoDB, EBS, EFS… (Day 11) |

> Trap: "keep backups for 7 years for compliance" → automated backups max out at 35 days. Use **manual snapshots** or **AWS Backup** with a long retention rule (optionally exported to S3).

### Encryption (memorise the rules)

- Enable at **creation**, using KMS. Covers storage, automated backups, snapshots, read replicas and logs.
- You **cannot** encrypt an existing unencrypted DB in place. Path: **snapshot → copy snapshot with encryption enabled → restore from the encrypted copy** → switch the app over.
- Read replicas of an encrypted DB are encrypted. You can't have an encrypted replica of an unencrypted primary.
- Cross-Region replica / snapshot copy of an encrypted DB uses a KMS key **in the destination Region**.
- **In transit:** TLS. Force it with a parameter (`rds.force_ssl=1` for PostgreSQL/SQL Server; `require_secure_transport=ON` for MySQL/MariaDB).

### Authentication and credentials

| Option | When |
| --- | --- |
| Master user password managed in **Secrets Manager** (RDS-managed, automatic rotation) | Default best answer for "rotate DB credentials automatically" |
| **IAM database authentication** | MySQL, MariaDB, PostgreSQL. App gets a short-lived token (valid 15 min) with `rds-db:connect`. No password stored. Good with roles on EC2/Lambda. |
| Kerberos (AWS Managed Microsoft AD) | SQL Server, Oracle, MySQL, PostgreSQL with corporate AD |
| SSM Parameter Store SecureString | Works, but no built-in rotation — Secrets Manager wins for "rotate" |

### Other RDS features to recognise

| Feature | Exam use |
| --- | --- |
| **RDS Proxy** | Pools and shares connections. Answer for "Lambda functions exhaust DB connections", "too many connections" errors, and "reduce failover time" (proxy keeps client connections open and reroutes). Can enforce IAM auth and pulls credentials from Secrets Manager. Lives in your VPC. |
| **Blue/Green Deployments** | Staging copy kept in sync via replication; switch over in about a minute for major version upgrades or schema changes with minimal downtime |
| **Performance Insights / CloudWatch Database Insights** | Find the SQL causing load |
| **Enhanced Monitoring** | OS-level metrics (per-process CPU/memory) at up to 1-second granularity — CloudWatch's default RDS metrics are hypervisor-level |
| **Event notifications** | RDS events → SNS / EventBridge |
| **Reserved Instances** | 1- or 3-year commitment for steady DB workloads — the cost answer for "runs 24/7 for years" |
| **Stop instance** | Up to 7 days; it automatically starts again after that. For long idle periods: snapshot and delete. |

---

## 3. RDS high availability and read scaling

This is the section the exam tests hardest. Two different tools, solving two different problems.

```
        Multi-AZ (instance)                         Read replicas
   ┌──────────── Region ────────────┐      ┌──────────────── Region ─────────────────┐
   │  AZ-a              AZ-b         │      │  AZ-a            AZ-b         AZ-c       │
   │ ┌───────┐  sync  ┌─────────┐   │      │ ┌───────┐ async ┌───────┐   ┌───────┐   │
   │ │Primary│──────► │ Standby │   │      │ │Primary│──────►│Replica│   │Replica│   │
   │ └───┬───┘        └─────────┘   │      │ └───┬───┘   └──►└───────┘   └───────┘   │
   │     │   (not readable)          │      │     │                 ▲  read-only        │
   └─────┼───────────────────────────┘      └─────┼─────────────────┼──────────────────┘
         │ one endpoint; DNS flips                 │ writes           │ reads (own endpoints)
        App                                       App ───────────────┘
                                                   └── async ──► Replica in another Region (DR)
```

| | **Multi-AZ DB instance** | **Multi-AZ DB cluster** | **Read replica** |
| --- | --- | --- | --- |
| Purpose | **Availability** (survive AZ/instance failure) | Availability **and** some read scaling | **Read scaling** (and DR if cross-Region) |
| Replication | **Synchronous** | Semi-synchronous (commit after ≥ 1 reader acknowledges) | **Asynchronous** (replica lag) |
| Readable standby? | **No** | Yes — 2 readable standbys | Yes |
| Engines | All RDS engines | MySQL, PostgreSQL | MySQL, MariaDB, PostgreSQL, Oracle, SQL Server, Db2 (feature details vary — check docs) |
| Failover | Automatic, DNS CNAME flips; typically 60–120 s | Automatic, typically under ~35 s | **Manual promotion** — becomes a standalone DB; replication breaks |
| Across Regions? | No (one Region) | No | **Yes** (cross-Region read replica) |
| Count | 1 standby | 1 writer + 2 readers in 3 AZs | Up to 15 for MySQL/MariaDB/PostgreSQL (check docs for other engines) |
| Cost | Pay for the standby | Pay for 3 instances | Pay for each replica (+ cross-Region transfer) |

What triggers Multi-AZ automatic failover: AZ outage, primary instance or storage failure, instance type change, OS patching, manual "reboot with failover".

Key points:

- **Multi-AZ is not a scaling tool.** The standby of a Multi-AZ DB *instance* serves no traffic. "Reduce read load on the primary" → read replicas (or ElastiCache).
- **Read replicas are not an HA tool** (in-Region). Promotion is manual and the app must point to a new endpoint. But a read replica *can itself* be Multi-AZ.
- Applications must send reads to the **replica endpoint** — RDS does not split reads and writes for you. (Aurora's reader endpoint load-balances across replicas; RDS Proxy can help too.)
- Reports/analytics queries slowing the production DB → run them against a **read replica**.
- Same-Region replica data transfer is free; cross-Region costs transfer.
- **Converting Single-AZ to Multi-AZ** is an online modification: RDS takes a snapshot, builds the standby, syncs — no downtime (some I/O latency).

> Trap: "highly available **and** offload read traffic, MySQL" → Multi-AZ **plus** read replicas (two features), *or* a Multi-AZ **DB cluster**, *or* Aurora. Pick the one that matches the qualifier: least changes on RDS → add both; best performance → Aurora.

---

## 4. Amazon Aurora

Aurora keeps the MySQL or PostgreSQL wire protocol but replaces the storage layer. That one design choice explains almost every Aurora exam fact.

### How the storage works

```
                     Aurora DB cluster (one Region)
   ┌───────────────────────────────────────────────────────────────────┐
   │   Writer (primary)       Aurora Replica        Aurora Replica       │
   │     AZ-a                    AZ-b                  AZ-c              │
   │       │ writes redo log        │ reads               │ reads        │
   │  ═════╧════════════════════════╧═════════════════════╧══════════    │
   │       Shared cluster volume — 6 copies across 3 AZs (2 per AZ)      │
   │       grows automatically; you pay for what you use                 │
   └───────────────────────────────────────────────────────────────────┘
```

- Data is stored **six ways across three AZs**. Writes need 4 of 6 copies; reads need 3 of 6. So Aurora tolerates losing an entire AZ (2 copies) without losing write availability, and losing an AZ plus one more copy without losing read availability.
- Storage is **shared** between the writer and all replicas. Replicas don't replay a full copy of the data — they read the same volume — so replica lag is typically **milliseconds** (often under 100 ms).
- Storage **auto-grows** in 10 GiB increments up to a large maximum (128 TiB on most versions — check current docs). No storage provisioning.
- Storage is self-healing; continuous incremental backup to S3. Backtrack and cloning build on this.

### Replicas, failover and endpoints

| Item | Aurora |
| --- | --- |
| Replicas per cluster | Up to **15** Aurora Replicas, across AZs |
| Failover | Automatic to a replica, typically **under 30 seconds**. You set **failover priority tiers** (0–15) to control which replica is promoted. With **no** replicas, Aurora recreates the writer, which takes longer. |
| Replica auto scaling | Adds/removes replicas based on CPU or connections |
| **Cluster (writer) endpoint** | Always points at the current writer — use for writes |
| **Reader endpoint** | Load-balances connections across replicas — use for reads |
| **Custom endpoints** | A named subset of instances, e.g. larger replicas for analytics queries |
| Instance endpoints | Direct to one instance — rarely the right answer |

> Trap: "Analytics queries on Aurora slow down the reporting replicas used by the app" → create a **custom endpoint** pointing at dedicated larger replicas for analytics.

### Aurora features to recognise

| Feature | What it does | Exam cue |
| --- | --- | --- |
| **Aurora Global Database** | One primary Region + read-only secondary Regions. Storage-level replication, typically **under 1 second** lag. Managed **switchover** (planned) or **failover** (unplanned) promotes a secondary in about a minute. Optional **write forwarding** from secondaries. | "Cross-Region DR with RPO of seconds and RTO of about a minute", "low-latency global reads" |
| **Aurora Serverless v2** | Capacity in **Aurora Capacity Units (ACUs)**, scales in fine-grained steps in seconds between a min and max you set. Can be mixed with provisioned instances in one cluster. Supports Multi-AZ, replicas and Global Database. Recent versions can scale to 0 ACUs and auto-pause when idle (check docs for engine versions). | "Unpredictable / spiky / intermittent relational workload", "dev/test that's idle most of the time", "least capacity management" |
| **Backtrack** (Aurora MySQL) | Rewinds the cluster in place to a point in time, in minutes, no restore | "Undo an accidental `DELETE` quickly without restoring a new cluster" |
| **Cloning** | Copy-on-write clone — fast, initially no extra storage | "Create a copy of prod for testing quickly and cheaply" |
| **Aurora I/O-Optimized** | Storage configuration with no per-I/O charges (higher instance/storage price) | "I/O costs are a large share of the Aurora bill" |
| **Aurora Machine Learning, zero-ETL to Redshift** | Call SageMaker/Bedrock from SQL; replicate to Redshift without pipelines | "Near-real-time analytics on transactional data without building ETL" |
| **Database Activity Streams** | Near-real-time stream of DB activity to Kinesis for auditing | "Monitor database activity for compliance" |
| **Aurora DSQL** | Distributed, serverless, PostgreSQL-compatible, active-active multi-Region | Newer service — recognise the name; not yet a core SAA-C03 topic |

### RDS vs Aurora

| | RDS (MySQL/PostgreSQL) | Aurora |
| --- | --- | --- |
| Storage | EBS volume per instance; you size it (autoscaling optional) | Shared, auto-growing cluster volume, 6 copies / 3 AZs |
| Read replicas | Up to 15, async, each with its own storage | Up to 15, shared storage, ms lag |
| Failover | Multi-AZ instance: ~1–2 min | Typically < 30 s |
| Cross-Region | Cross-Region read replica (async, promote manually) | **Global Database** (< 1 s lag, managed failover) |
| Serverless | No | Serverless v2 |
| Price | Lower per instance | Higher per-instance price (check current pricing), but often cheaper at scale since replicas don't duplicate storage |
| Engines | Six engines | MySQL and PostgreSQL only |

**Migrating to Aurora:** from RDS MySQL/PostgreSQL, create an **Aurora read replica** of the RDS instance and promote it (minimal downtime), or restore a snapshot. From on-premises or other engines: **AWS DMS** (+ Schema Conversion for heterogeneous moves) — Day 11.

---

## 5. Amazon DynamoDB

DynamoDB is a serverless key-value and document database. No instances, no patching, no storage sizing. It scales by **partitioning** data by the partition key, so design is all about keys.

### Data model

```
Table: Orders
┌──────────────────────┬─────────────────────┬────────┬────────┬──────────┐
│ Partition key (PK)    │ Sort key (SK)        │ total  │ status │ ...      │
│  customerId           │  orderDate#orderId   │        │        │ (any     │
├──────────────────────┼─────────────────────┼────────┼────────┤  other   │
│  C#1001               │  2026-11-01#O-77     │  42.10 │ SHIPPED│  attrs,  │
│  C#1001               │  2026-11-03#O-91     │  18.00 │ PENDING│  schema- │
│  C#2002               │  2026-10-30#O-12     │  99.99 │ SHIPPED│  less)   │
└──────────────────────┴─────────────────────┴────────┴────────┴──────────┘
  hash(PK) → partition        items with same PK stored together, sorted by SK
```

- **Primary key:** partition key alone, or partition key + sort key (composite). Must be unique.
- **Item** = row; max item size **400 KB**. Larger blobs → store in **S3**, keep the S3 key in DynamoDB.
- `GetItem` (one item by full key), `Query` (one partition key, optional sort-key condition — efficient), `Scan` (reads the whole table — slow and expensive; a "Scan is slow" question wants a GSI or a better key).
- **High-cardinality partition keys** (userId, orderId) spread load. Low-cardinality keys (status, date) create **hot partitions** — throttling even when total capacity looks fine.

### Consistency and capacity

| | Detail |
| --- | --- |
| Reads | **Eventually consistent** by default; **strongly consistent** on request (base table and LSI only, same Region) |
| Transactions | `TransactWriteItems` / `TransactGetItems` — ACID across multiple items/tables; cost 2× normal |
| **RCU** (provisioned) | 1 strongly consistent read/s of up to 4 KB, or 2 eventually consistent reads/s |
| **WCU** (provisioned) | 1 write/s of up to 1 KB |

Worked example: 50 strongly consistent reads/s of 6 KB items → each read rounds up to 8 KB = 2 RCU → **100 RCU**. Eventually consistent → **50 RCU**.

| Capacity mode | How it bills | Pick when |
| --- | --- | --- |
| **On-demand** | Per request; no capacity planning; scales instantly to prior peaks and beyond | New/unknown/spiky traffic, "least operational overhead", pay-per-use |
| **Provisioned** (+ **auto scaling**) | Per RCU/WCU-hour; auto scaling adjusts to a target utilisation | Steady, predictable traffic; cheaper at sustained load; can buy **reserved capacity** |

You can switch modes (with a limit on how often — check docs). Throttling (`ProvisionedThroughputExceededException`) on provisioned → enable auto scaling, switch to on-demand, or fix a hot key.

**Table classes:** Standard, and **Standard-IA** (lower storage price, higher request price) for tables that are mostly stored, rarely read.

### Secondary indexes

| | **Local Secondary Index (LSI)** | **Global Secondary Index (GSI)** |
| --- | --- | --- |
| Keys | **Same** partition key, different sort key | **Any** partition key and sort key |
| When created | **Only at table creation** | Any time |
| Consistency | Strong or eventual | **Eventual only** |
| Capacity | Shares the table's | Its own (provisioned) — a throttled GSI can throttle table writes |
| Limits | 5 per table; 10 GB per partition-key value | 20 per table by default (check docs) |
| Pick when | Alternate sort order within one customer/partition, needs strong reads | Query by a completely different attribute (e.g. by `status` or `email`) |

> Trap: "Need to query by a new attribute on an existing table" → **GSI** (LSIs can't be added later).

### DAX, Streams, TTL

- **DynamoDB Accelerator (DAX):** in-memory, write-through cache *in front of* DynamoDB. **Microsecond** reads, API-compatible (minimal code change — swap the client). Runs as a cluster in your VPC. Caches eventually consistent reads only; strongly consistent reads pass through. Answer for "read-heavy DynamoDB, microsecond latency, minimal changes." (ElastiCache is for general-purpose caching or for RDS.)
- **DynamoDB Streams:** ordered, per-item change log (INSERT/MODIFY/REMOVE), kept **24 hours**. Trigger **Lambda** for event-driven processing (send email on new order, update an aggregate, replicate to OpenSearch). Need longer retention or multiple consumers → **Kinesis Data Streams for DynamoDB**.
- **Time to Live (TTL):** set an epoch-seconds attribute; DynamoDB deletes expired items automatically at no extra write cost (deletion is background and not instant). Answer for "expire session data / old records automatically".

### Global Tables

```
   us-east-1                 eu-west-1                 ap-south-1
 ┌───────────┐  ◄──────►  ┌───────────┐  ◄──────►  ┌───────────┐
 │ Orders    │  async,    │ Orders    │            │ Orders    │
 │ read+write│  usually   │ read+write│            │ read+write│
 └───────────┘  < 1 s     └───────────┘            └───────────┘
        multi-Region, multi-active (every replica accepts writes)
```

- **Multi-active**: users write to their nearest Region. Default mode is **eventually consistent** across Regions, conflicts resolved **last writer wins**.
- A **multi-Region strong consistency** mode now also exists (with Region-count and feature constraints — check current docs). For the exam, the classic answer is "Global Tables = multi-Region active-active".
- Built on Streams; replication is managed for you.
- Answer for "globally distributed users need low-latency reads **and writes**", or "multi-Region active-active key-value". (Aurora Global Database = one writer Region.)

### Backups, security and integrations

| Feature | Detail |
| --- | --- |
| **Point-in-time recovery (PITR)** | Continuous backups, restore to any second within the recovery period (up to 35 days, configurable). Restore creates a **new table**. |
| **On-demand backups** | Full backups kept until deleted; no performance impact. Also via AWS Backup (cross-Region/cross-account copy, long retention). |
| **Export to S3** | Full or incremental export from PITR data, no RCU consumed → analyse with Athena. |
| **Import from S3** | Bulk load into a new table. |
| **Encryption at rest** | **Always on**: AWS owned key (default), AWS managed key or customer managed KMS key. |
| **VPC access** | **Gateway VPC endpoint** (free) — like S3. Interface endpoints also exist. |
| **Fine-grained access** | IAM condition `dynamodb:LeadingKeys` restricts users to items whose partition key equals their ID (common with Cognito identities). |
| **Zero-ETL** | To Amazon OpenSearch Service (search) and Redshift (analytics). |
| **Resource-based policies** | Cross-account access to tables/streams without assuming roles. |

---

## 6. Amazon ElastiCache and MemoryDB

### Why cache

A cache stores frequently read results in memory so the database isn't asked again. It cuts latency (sub-millisecond) and database load (often cheaper than scaling the DB). It also requires **application changes** — your code has to talk to the cache. That's the key difference from DAX (drop-in for DynamoDB) and read replicas (just a new endpoint).

### Caching strategies

```
Lazy loading (cache-aside)                     Write-through
  read ─► cache hit? ─yes─► return               write ─► DB ─► and update cache
              │no                                read  ─► cache (always populated)
              ▼
          read DB ─► put in cache (with TTL) ─► return
```

| Strategy | Pros | Cons |
| --- | --- | --- |
| **Lazy loading / cache-aside** | Only requested data cached; cache failure isn't fatal | Cache miss = 3 round trips; data can be **stale** |
| **Write-through** | Cache always fresh | Every write costs more; caches data that may never be read |
| **TTL** (combine with either) | Bounds staleness; evicts cold data | Choose TTL vs freshness trade-off |

Eviction: when memory is full, keys are evicted (LRU by default for Valkey/Redis OSS). Constant evictions → scale up or out.

### Engines

| | **Valkey / Redis OSS** | **Memcached** |
| --- | --- | --- |
| Data types | Strings, hashes, lists, sets, **sorted sets**, streams, geospatial | Simple key-value strings |
| Replication / Multi-AZ | **Yes** — primary + up to 5 replicas per shard, **automatic failover** | **No** replication |
| Persistence / backup | **Yes** — snapshots (and AOF) | **No** — data lost on node failure |
| Sharding | Cluster mode enabled (up to many shards) | Yes — client-side across nodes |
| Threads | Mostly single-threaded command execution | **Multi-threaded** |
| Pub/Sub, Lua, transactions | Yes | No |
| Security | In-transit + at-rest encryption, AUTH / RBAC, IAM auth | SASL auth, encryption in transit (check docs) |
| Pick when | HA, durability, **leaderboards** (sorted sets), sessions, pub/sub, complex types | Simplest pure cache, multi-threaded, can lose data |

- **Valkey** is the open-source fork of Redis that AWS now leads with; it's compatible with Redis OSS clients and priced lower than Redis OSS on ElastiCache (check current pricing). Exam questions may still say "ElastiCache for Redis" — treat it as the same family.
- **ElastiCache Serverless:** no node sizing; scales automatically; Multi-AZ by default. "Cache with least operational overhead / unpredictable load" → Serverless.
- **Session store:** making web tiers **stateless** by moving HTTP sessions to ElastiCache (or DynamoDB) is a classic Auto Scaling enabler (Day 3 sticky sessions are the alternative, and usually the weaker answer).

### MemoryDB

**Amazon MemoryDB** is Valkey/Redis OSS-compatible but **durable**: every write goes to a Multi-AZ transaction log before it's acknowledged. Use it as the **primary database** when you need in-memory speed *and* durability (e.g. microsecond reads, single-digit-ms writes for a gaming backend). ElastiCache is a **cache** in front of another database.

---

## 7. Amazon Redshift

Redshift is a columnar, massively parallel processing (MPP) **data warehouse**. It's built for scanning and aggregating huge datasets, not for thousands of small transactions.

```
           SQL clients / BI (QuickSight)
                     │ JDBC/ODBC
               ┌─────▼──────┐
               │ Leader node │  parses, plans, aggregates results
               └─────┬──────┘
       ┌─────────────┼─────────────┐
 ┌─────▼────┐  ┌─────▼────┐  ┌─────▼────┐
 │ Compute  │  │ Compute  │  │ Compute  │   columnar storage, slices run in parallel
 └─────┬────┘  └─────┬────┘  └─────┬────┘
       └──── RA3: Redshift Managed Storage (S3-backed; scale compute and storage separately)
                     │
             Redshift Spectrum ──► query data in S3 directly (external tables, Glue Data Catalog)
```

| Feature | Exam cue |
| --- | --- |
| **Columnar + compression + MPP** | Why it's fast for `SUM`/`GROUP BY` over billions of rows |
| **RA3 nodes + managed storage** | Scale compute independently of storage |
| **Redshift Serverless** | No cluster management; pay for compute used (RPUs). "Intermittent analytics / least admin" |
| **Redshift Spectrum** | Query exabytes in S3 without loading; join with warehouse tables |
| **Concurrency Scaling** | Adds transient capacity for bursts of concurrent queries |
| **Workload Management (WLM)** | Prioritise queues — e.g. short dashboards vs long ETL |
| **Data sharing** | Share live data between clusters/accounts without copying |
| **Zero-ETL integrations** | From Aurora, RDS, DynamoDB (and others) into Redshift without pipelines |
| **Snapshots** | Automated + manual, stored in S3. **Cross-Region snapshot copy** for DR (KMS-encrypted clusters need a snapshot copy grant in the destination Region). |
| **Multi-AZ** (RA3) | Deploy across two AZs for higher availability (check docs for constraints) |
| **Enhanced VPC routing** | Forces COPY/UNLOAD traffic through your VPC (for endpoints, NACLs, flow logs) |
| Loading | `COPY` from S3 (parallel, the fast way); streaming ingestion from Kinesis/MSK; Amazon Data Firehose into Redshift |

**Redshift vs Athena (preview of Day 12):** Redshift for a persistent warehouse with frequent, complex, fast BI queries. **Athena** for occasional, serverless SQL on files in S3, pay per data scanned.

> Trap: Redshift is never the answer for OLTP ("process customer orders"), and RDS is never the answer for "analyse petabytes with complex queries."

---

## 8. Exam traps roundup

1. **Multi-AZ ≠ read scaling.** The Multi-AZ *instance* standby isn't readable.
2. **Read replica ≠ HA.** Async, manual promotion, new endpoint (in-Region). Cross-Region replica = DR.
3. **Encrypting an existing unencrypted RDS DB** → snapshot → encrypted copy → restore. Not "modify the instance".
4. **Backups beyond 35 days** → manual snapshots or AWS Backup.
5. **Lambda + RDS "too many connections"** → RDS Proxy.
6. **Rotate DB passwords automatically** → Secrets Manager (not Parameter Store, not a cron Lambda).
7. **Aurora fastest failover** → have at least one Aurora Replica in another AZ (with none, recovery is slower). Use priority tiers to choose the target.
8. **Cross-Region RPO of seconds, RTO ~1 min, relational** → Aurora Global Database. Multi-Region **active-active writes**, key-value → DynamoDB Global Tables.
9. **LSIs only at table creation; GSIs are eventually consistent.**
10. **DynamoDB items > 400 KB** → S3 + pointer.
11. **DAX for DynamoDB; ElastiCache for RDS / general-purpose.** DAX needs no app logic change; ElastiCache does.
12. **Memcached has no replication or persistence.** HA/durable cache → Valkey/Redis OSS. Durable primary DB in memory → MemoryDB.
13. **Restores create new resources** (RDS instance, Aurora cluster, DynamoDB table) — update the endpoint/table name. Exception: Aurora **Backtrack** is in place.
14. **Security groups, not NACLs or IPs,** are the standard way to lock a DB to the app tier.
15. **Stopped RDS restarts after 7 days** — for long pauses, snapshot and delete (or Aurora Serverless v2 scaling to zero).

---

## 9. Exam decision table

| If the question says… | Pick |
| --- | --- |
| Existing MySQL/PostgreSQL/Oracle/SQL Server app, move to AWS with minimal changes | **Amazon RDS** (same engine) |
| Need OS/host access to the DB server, Oracle or SQL Server | **RDS Custom** (otherwise EC2) |
| Survive an AZ failure, automatic failover, RDS | **RDS Multi-AZ** |
| Reduce read load / reporting queries slowing the primary | **Read replicas** (Aurora: reader/custom endpoint) |
| HA + readable standbys, MySQL/PostgreSQL, faster failover than Multi-AZ instance | **Multi-AZ DB cluster** (or Aurora) |
| DR copy of a relational DB in another Region, low cost | **Cross-Region read replica** or cross-Region snapshot/backup copy |
| Relational, cross-Region RPO ~1 s, RTO ~1 min, global reads | **Aurora Global Database** |
| Relational, unpredictable/spiky/intermittent load, least capacity management | **Aurora Serverless v2** |
| Up to 15 low-lag replicas, failover < 30 s, auto-growing storage | **Aurora** |
| Quickly undo a bad change on Aurora MySQL without restoring | **Aurora Backtrack** |
| Fast, cheap copy of a production Aurora DB for testing | **Aurora cloning** |
| Encrypt an existing unencrypted RDS DB | Snapshot → **copy with encryption** → restore |
| Rotate DB credentials automatically | **Secrets Manager** |
| DB login without passwords using IAM roles | **IAM database authentication** |
| Lambda exhausting DB connections / reduce failover disruption | **RDS Proxy** |
| Major version upgrade with minimal downtime | **RDS Blue/Green Deployments** |
| Key-value, millions of requests/s, single-digit ms, serverless | **DynamoDB** |
| DynamoDB, unknown/spiky traffic, no capacity planning | **On-demand** capacity |
| DynamoDB, steady predictable traffic, lowest cost | **Provisioned** + auto scaling (+ reserved capacity) |
| Query DynamoDB by a different attribute on an existing table | **GSI** |
| DynamoDB microsecond reads, minimal code change | **DAX** |
| React to every DynamoDB item change | **DynamoDB Streams + Lambda** |
| Automatically delete expired items | **DynamoDB TTL** |
| Multi-Region active-active reads and writes | **DynamoDB Global Tables** |
| Restore DynamoDB to a second before an accidental write | **PITR** |
| Users may only access items with their own user ID as key | IAM condition **`dynamodb:LeadingKeys`** |
| Cache DB query results, sub-ms, with HA and backups | **ElastiCache for Valkey / Redis OSS** |
| Simplest multi-threaded cache, data loss acceptable | **ElastiCache for Memcached** |
| Real-time leaderboard | **ElastiCache (Valkey/Redis OSS) sorted sets** |
| Stateless web tier — where to put sessions | **ElastiCache** or **DynamoDB** |
| In-memory speed but durable primary database | **MemoryDB** |
| Data warehouse, complex SQL over TB–PB, BI dashboards | **Redshift** |
| Warehouse queries that also need data in S3 without loading it | **Redshift Spectrum** |
| Analytics on transactional data without building ETL | **Zero-ETL** integration (Aurora/RDS/DynamoDB → Redshift) |
| MongoDB workload | **DocumentDB** |
| Graph relationships | **Neptune** |
| Cassandra workload | **Keyspaces** |

---

## 10. Lab (30 min, free tier)

Goal: build a DynamoDB table with a GSI and TTL, feel the Query-vs-Scan difference, then launch a small RDS instance and walk through the encryption and Multi-AZ settings without paying for them.

**Cost note:** DynamoDB on-demand with a handful of items costs almost nothing. A Single-AZ `db.t4g.micro` (or `db.t3.micro`) RDS instance with 20 GB gp2/gp3 has historically been free-tier eligible, but free-tier terms changed for newer accounts — **check current free-tier/credit terms** before launching. Do **not** enable Multi-AZ, read replicas, Aurora, ElastiCache or Redshift in this lab — none are free. Delete everything at the end.

### Console steps

**Part A — DynamoDB (15 min)**

1. **DynamoDB** → *Create table* → name `saa-orders`, partition key `customerId` (String), sort key `orderId` (String). *Table settings: Customize* → capacity mode **On-demand**. Create.
2. *Indexes* → *Create index* → partition key `status` (String), sort key `orderId`, name `status-index`. (A GSI added *after* creation — you could not do this with an LSI.)
3. *Additional settings* → **Time to Live** → attribute `expiresAt`. **Point-in-time recovery** → turn on.
4. From **CloudShell**, add items:
   ```bash
   T=saa-orders
   NOW=$(date +%s); SOON=$((NOW+600))
   aws dynamodb put-item --table-name $T --item '{"customerId":{"S":"C1"},"orderId":{"S":"O1"},"status":{"S":"SHIPPED"},"total":{"N":"42"}}'
   aws dynamodb put-item --table-name $T --item '{"customerId":{"S":"C1"},"orderId":{"S":"O2"},"status":{"S":"PENDING"},"total":{"N":"18"}}'
   aws dynamodb put-item --table-name $T --item "{\"customerId\":{\"S\":\"C2\"},\"orderId\":{\"S\":\"O3\"},\"status\":{\"S\":\"PENDING\"},\"total\":{\"N\":\"99\"},\"expiresAt\":{\"N\":\"$SOON\"}}"
   ```
5. **Query vs Scan** — compare `ConsumedCapacity`:
   ```bash
   aws dynamodb query --table-name $T --key-condition-expression "customerId = :c" \
     --expression-attribute-values '{":c":{"S":"C1"}}' --return-consumed-capacity TOTAL
   aws dynamodb scan --table-name $T --return-consumed-capacity TOTAL
   aws dynamodb query --table-name $T --index-name status-index --key-condition-expression "#s = :p" \
     --expression-attribute-names '{"#s":"status"}' --expression-attribute-values '{":p":{"S":"PENDING"}}'
   ```
   With three items the numbers are tiny, but the Scan reads *everything*; the GSI query finds all `PENDING` orders without one. (`status` is a reserved word — hence `#s`.)
6. Try a strongly consistent read on the GSI: add `--consistent-read` to the last command → `ValidationException`. GSIs are eventually consistent only.
7. Look at *Exports and streams* — note **DynamoDB Streams** (24 h) vs **Kinesis Data Streams** options. Don't enable.

**Part B — RDS walkthrough (15 min)**

8. **RDS** → *Create database* → *Standard create* → **PostgreSQL** → template **Free tier** (if shown; otherwise *Dev/Test*, Single-AZ).
9. Before creating, read each section and note the exam-relevant settings:
   - *Availability and durability*: **Single-AZ DB instance**, **Multi-AZ DB instance**, **Multi-AZ DB cluster** — the three options from section 3. Keep **Single-AZ**.
   - *Credentials*: choose **Managed in AWS Secrets Manager**.
   - *Instance*: `db.t4g.micro`. *Storage*: gp3, 20 GiB; note **Enable storage autoscaling**.
   - *Connectivity*: default VPC, **Public access: No**, new security group `saa-db-sg`.
   - *Database authentication*: note **Password and IAM database authentication**.
   - *Additional configuration*: backup retention (1–35 days), **Encryption enabled** (default, `aws/rds` key). Note you can't change encryption after creation.
   - Turn **off** Performance Insights/Database Insights extras and Enhanced Monitoring to stay free.
10. Create. While it's provisioning (5–10 min), open **Secrets Manager** and find the RDS-managed secret with rotation configured.
11. Once *Available*: *Actions* → **Take snapshot** `saa-snap-1`. Then on the snapshot → *Actions* → look at **Copy snapshot** — note the KMS key choice and destination Region (don't run it). This is the "encrypt an unencrypted DB" path.
12. *Actions* on the DB → look at **Create read replica** and **Modify → Multi-AZ** (don't apply; both cost money).
13. **Tear down:** delete the RDS instance — **uncheck** *Create final snapshot*, **uncheck** *Retain automated backups*, acknowledge. Delete snapshot `saa-snap-1`. Delete the DynamoDB table `saa-orders`. Delete the `saa-db-sg` security group after the DB is gone. The RDS-managed secret is deleted with the instance — confirm in Secrets Manager.

### Terraform variant

```hcl
provider "aws" { region = "ap-south-1" }

# ---------- DynamoDB ----------
resource "aws_dynamodb_table" "orders" {
  name         = "saa-orders"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "customerId"
  range_key    = "orderId"

  attribute {
    name = "customerId"
    type = "S"
  }
  attribute {
    name = "orderId"
    type = "S"
  }
  attribute {
    name = "status"
    type = "S"
  }

  global_secondary_index {
    name            = "status-index"
    hash_key        = "status"
    range_key       = "orderId"
    projection_type = "ALL"
  }

  ttl {
    attribute_name = "expiresAt"
    enabled        = true
  }

  point_in_time_recovery { enabled = true }

  # Encryption at rest is always on; omit server_side_encryption to use the AWS owned key.
}

# ---------- RDS PostgreSQL (Single-AZ, lab only) ----------
data "aws_vpc" "default" { default = true }

resource "aws_security_group" "db" {
  name   = "saa-db-sg"
  vpc_id = data.aws_vpc.default.id
  # No ingress in the lab. In real life: ingress 5432 from the app tier's SG:
  # ingress { from_port = 5432, to_port = 5432, protocol = "tcp", security_groups = [aws_security_group.app.id] }
}

resource "aws_db_instance" "lab" {
  identifier                  = "saa-lab-pg"
  engine                      = "postgres"
  instance_class              = "db.t4g.micro"
  allocated_storage           = 20
  max_allocated_storage       = 50          # storage autoscaling ceiling
  storage_type                = "gp3"
  storage_encrypted           = true        # can't be turned on later
  username                    = "saaadmin"
  manage_master_user_password = true        # password lives in Secrets Manager, rotated by RDS
  iam_database_authentication_enabled = true
  multi_az                    = false       # true = synchronous standby in another AZ (costs extra)
  publicly_accessible         = false
  vpc_security_group_ids      = [aws_security_group.db.id]
  backup_retention_period     = 1           # 0 disables automated backups / PITR
  deletion_protection         = false       # set true in production
  skip_final_snapshot         = true        # lab only
  delete_automated_backups    = true
}

# A read replica would be:
# resource "aws_db_instance" "replica" {
#   identifier          = "saa-lab-pg-replica"
#   replicate_source_db = aws_db_instance.lab.identifier
#   instance_class      = "db.t4g.micro"
#   skip_final_snapshot = true
# }

output "db_endpoint"   { value = aws_db_instance.lab.endpoint }
output "db_secret_arn" { value = aws_db_instance.lab.master_user_secret[0].secret_arn }
```

Run `terraform init && terraform apply`, do steps 4–7 against the table, then **`terraform destroy`** and confirm in the console that the DB instance, its automated backups, the table and the security group are gone.

---

## 11. Practice questions

Answer all ten before opening the answers.

**Q1.** A company runs a MySQL database on Amazon RDS. The database must remain available if an Availability Zone fails, with automatic failover and no data loss. Which solution meets these requirements?
A. Create a read replica in another AZ and promote it if the primary fails
B. Enable Multi-AZ on the RDS DB instance
C. Take automated snapshots every 5 minutes and restore in another AZ
D. Migrate the database to DynamoDB

**Q2.** An e-commerce site on Amazon RDS for PostgreSQL is slowing down because analysts run heavy reporting queries during business hours. The company wants to isolate reporting without changing the main application. What should a solutions architect do?
A. Enable Multi-AZ and point analysts at the standby
B. Create a read replica and direct reporting queries to its endpoint
C. Increase the instance size of the primary
D. Add an ElastiCache cluster in front of the database

**Q3.** A company's Amazon RDS for MySQL database was created without encryption. A new compliance rule requires encryption at rest. What is the correct approach with minimal data loss?
A. Modify the DB instance and enable encryption
B. Create an encrypted read replica and promote it
C. Take a snapshot, copy the snapshot with encryption enabled using a KMS key, and restore a new DB instance from the encrypted copy
D. Enable default EBS encryption for the account

**Q4.** A serverless application uses hundreds of concurrent AWS Lambda functions that connect to an Amazon Aurora PostgreSQL cluster. The database is reporting too many connections and occasional errors during failover. Which solution requires the LEAST code change?
A. Increase the `max_connections` parameter
B. Place Amazon RDS Proxy between Lambda and Aurora
C. Migrate to DynamoDB
D. Add more Aurora Replicas

**Q5.** A global company runs its order system on Aurora MySQL in `us-east-1`. It needs a disaster recovery plan with an RPO of about 1 second and an RTO of about 1 minute, plus low-latency reads for users in Europe. What should a solutions architect recommend?
A. Cross-Region read replicas of RDS MySQL
B. Aurora Global Database with a secondary Region in Europe
C. Daily snapshots copied to `eu-west-1`
D. DynamoDB Global Tables

**Q6.** A team has a DynamoDB table keyed by `orderId`. They now need to retrieve all orders for a given `customerEmail` efficiently. The table is already in production. What should they do?
A. Add a local secondary index on `customerEmail`
B. Add a global secondary index with `customerEmail` as the partition key
C. Use a Scan with a filter expression
D. Recreate the table with `customerEmail` as the partition key

**Q7.** A gaming application stores player profiles in DynamoDB. Reads are extremely heavy and the company wants microsecond response times without rewriting data-access logic. What is the MOST suitable solution?
A. ElastiCache for Memcached with lazy loading
B. DynamoDB Accelerator (DAX)
C. Increase read capacity units
D. DynamoDB Global Tables

**Q8.** A company wants a real-time leaderboard for an online game that ranks millions of players by score, with high availability and automatic failover. Which solution is the best fit?
A. Amazon RDS for MySQL with an `ORDER BY` index
B. ElastiCache for Memcached
C. ElastiCache for Valkey (or Redis OSS) with Multi-AZ, using sorted sets
D. Amazon Redshift

**Q9.** A startup is building a new relational application. Traffic is unpredictable: near zero overnight, with sudden spikes during marketing campaigns. The team wants to minimize database capacity management and pay only for what it uses, while keeping SQL and joins. Which option fits best?
A. Amazon RDS for MySQL with provisioned Reserved Instances
B. Aurora Serverless v2
C. DynamoDB on-demand
D. Amazon Redshift Serverless

**Q10.** A company stores user session data in DynamoDB. Sessions must be removed automatically 24 hours after the last activity, without consuming write capacity for deletes or running scheduled jobs. What should a solutions architect do?
A. Run a daily Lambda function that scans and deletes old items
B. Enable DynamoDB TTL on an attribute holding the expiry time in epoch seconds, updated on each activity
C. Use DynamoDB Streams to delete items after 24 hours
D. Create a GSI on `lastActivity` and delete via Query

<details>
<summary><strong>Answers</strong></summary>

1. **B** — Multi-AZ uses synchronous replication and automatic DNS failover, so no committed data is lost. A read replica (A) is async and needs manual promotion; snapshot restores (C) lose data and take time.
2. **B** — A read replica offloads reporting to its own endpoint without touching the app's writes. The Multi-AZ instance standby (A) is not readable; ElastiCache (D) needs app changes and doesn't help ad hoc reports.
3. **C** — Encryption can't be enabled on an existing instance (A), and you can't create an encrypted replica from an unencrypted primary (B). Snapshot → encrypted copy → restore is the documented path.
4. **B** — RDS Proxy pools connections and holds them during failover; the app only changes its endpoint. Raising `max_connections` (A) just moves the limit and wastes memory.
5. **B** — Aurora Global Database replicates at the storage layer with typically under 1 s lag and promotes a secondary in about a minute; secondaries also serve local reads. Snapshots (C) give hours of RPO; DynamoDB (D) means rewriting the app.
6. **B** — GSIs can use a different partition key and can be added to existing tables. LSIs (A) must share the table's partition key and can only be created with the table; Scan (C) is slow and costly.
7. **B** — DAX is an API-compatible in-memory cache for DynamoDB with microsecond reads; the app swaps the client. ElastiCache (A) requires custom caching logic.
8. **C** — Sorted sets make ranking O(log N) and native; Valkey/Redis OSS supports replicas and automatic Multi-AZ failover. Memcached (B) has neither sorted sets nor replication.
9. **B** — Aurora Serverless v2 scales capacity automatically in fine steps and keeps full SQL. DynamoDB (C) isn't relational; Redshift (D) is for analytics; Reserved Instances (A) assume steady load.
10. **B** — TTL deletes expired items in the background at no write-capacity cost. Updating the expiry on activity gives "24 hours after last activity". A and D cost capacity and need jobs; Streams (C) only report changes.

</details>

---

## 12. Flashcards (cover the right column)

| Prompt | Answer |
| --- | --- |
| RDS Multi-AZ instance replication type | Synchronous |
| Is the Multi-AZ instance standby readable? | No |
| Read replica replication type | Asynchronous |
| Multi-AZ DB cluster shape | 1 writer + 2 readable standbys in 3 AZs (MySQL/PostgreSQL) |
| Max automated backup retention, RDS | 35 days |
| Encrypt an existing unencrypted RDS DB | Snapshot → copy with encryption → restore |
| Does an RDS restore overwrite the existing DB? | No — creates a new instance/endpoint |
| Fix "too many connections" from Lambda | RDS Proxy |
| Rotate DB credentials automatically | Secrets Manager |
| IAM DB auth token lifetime | 15 minutes |
| Aurora storage copies / AZs | 6 copies across 3 AZs |
| Max Aurora Replicas | 15 |
| Typical Aurora failover time | Under 30 seconds |
| Aurora endpoint for load-balanced reads | Reader endpoint |
| Aurora cross-Region, ~1 s RPO | Global Database |
| Aurora capacity unit | ACU (Serverless v2) |
| Rewind Aurora MySQL in place | Backtrack |
| Max DynamoDB item size | 400 KB |
| 1 RCU = | 1 strongly consistent (or 2 eventually consistent) reads/s of up to 4 KB |
| 1 WCU = | 1 write/s of up to 1 KB |
| Index that can be added after table creation | GSI |
| Index that supports strongly consistent reads | LSI |
| DynamoDB Streams retention | 24 hours |
| DynamoDB microsecond cache | DAX |
| DynamoDB multi-Region active-active | Global Tables |
| DynamoDB auto-expire items | TTL |
| DynamoDB VPC endpoint type (free) | Gateway endpoint |
| Cache with no replication or persistence | Memcached |
| Cache data structure for leaderboards | Sorted sets (Valkey/Redis OSS) |
| Durable in-memory primary database | MemoryDB |
| Query S3 data from Redshift without loading | Redshift Spectrum |
| Redshift DR to another Region | Cross-Region snapshot copy |
| MongoDB-compatible / graph / Cassandra | DocumentDB / Neptune / Keyspaces |

## 13. Before you close today

- [ ] Read sections 1–9; redraw the Multi-AZ vs read replica vs Aurora comparison (sections 3–4) from memory
- [ ] Do the RCU worked example again with 10 KB items, both consistency modes
- [ ] Lab done and torn down (RDS instance deleted with no final snapshot or retained backups; manual snapshot deleted; DynamoDB table deleted; security group deleted)
- [ ] 10 questions above + 20 bank questions on RDS, Aurora, DynamoDB, ElastiCache and Redshift
- [ ] Misses logged in `miss-log.md` — especially Multi-AZ vs read replica, GSI vs LSI, and DAX vs ElastiCache mix-ups

**Tomorrow (Day 7):** Review of Days 1–6 and your first full 65-question timed practice exam, with a domain-scoring guide.
