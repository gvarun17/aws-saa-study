# Day 12 — Analytics, Cost Tools + Practice Exam 2

**Domain weight:** analytics sits mostly in *Design High-Performing Architectures* (24%) — "select high-performing data ingestion and transformation solutions" is its own task statement. Cost tools are the core of *Design Cost-Optimized Architectures* (20%). The practice exam covers all four domains in exam proportion, drawing on Days 1–12 with extra weight on Days 8–12 (Exam 1 only covered Days 1–6).
**Time plan (≈ 5 h — long day on purpose):** Learn 75 min · Lab 30 min · **Practice Exam 2: 130 min, timed, no notes** · Score & log misses 45 min.

By the end of today you should be able to:

- Trace a data lake end to end — ingest (Kinesis, Data Firehose, MSK, DMS) → store (S3) → catalog (Glue) → govern (Lake Formation) → query/process (Athena, Redshift, EMR) → visualise (QuickSight) — and name the service at each step.
- Pick between Athena, Redshift (provisioned, Serverless, Spectrum), EMR, Glue ETL and OpenSearch from the shape of the workload.
- Explain *why* Parquet + partitioning + compression make Athena faster and cheaper, and how Lake Formation's column/row permissions differ from S3 bucket policies.
- Match every cost tool to its job: Cost Explorer, Budgets (and budget actions), Cost Anomaly Detection, Data Exports (CUR 2.0), cost allocation tags, Compute Optimizer, Trusted Advisor, Savings Plans/RIs, Pricing Calculator.
- Sit Practice Exam 2 at ~2 min/question, score it by domain, and decide what Day 13 must fix.

> **Miss-log check:** `miss-log.md` is still empty, so there are no logged patterns to target yet. Today's exam is your second data point — **log every miss and lucky guess tonight**, including any you remember from Exam 1 and Days 8–11. Day 13's gap-closing lesson and the go/no-go call are built from that file.

---

## 1. The analytics pipeline: one picture, every service

```
 SOURCES                INGEST                      STORE + CATALOG               PROCESS / QUERY             CONSUME
 ───────                ──────                      ───────────────               ───────────────             ───────
 Apps, clicks,   ──►  Kinesis Data Streams  ─┐
 IoT, logs            (real-time, replayable)│     ┌───────────────────┐       ┌─ Athena (serverless SQL) ──► QuickSight
                                             ├──►  │  Amazon S3        │       │                              dashboards
 Logs, metrics   ──►  Amazon Data Firehose ──┤     │  raw/ → clean/ →  │◄──────┼─ Redshift + Spectrum ─────► BI / SQL
                      (near-real-time, no    │     │  curated/ (Parquet│       │                               clients
                       code, buffers to S3)  │     │  partitioned)     │       ├─ EMR / EMR Serverless ────► ML, Spark
 Kafka apps      ──►  Amazon MSK  ───────────┘     └────────┬──────────┘       │   (Spark, Hive, Presto)
                                                            │                  │
 RDS / on-prem   ──►  DMS (full load + CDC) ──────────────► │                  └─ Glue ETL jobs (Spark,
 databases            or zero-ETL → Redshift                │                     serverless) ──► back to S3
                                                    ┌───────▼───────────┐
                                                    │ Glue Data Catalog │  ◄── crawlers infer schema
                                                    │ (Hive metastore)  │
                                                    └───────┬───────────┘
                                                    Lake Formation: who can see which DB / table / column / row
 Search / logs   ──►  OpenSearch Ingestion / Firehose ──►  Amazon OpenSearch Service ──► OpenSearch Dashboards
```

Why this shape? **Storage and compute are decoupled.** Data lives once in S3 (cheap, 11 nines, any format), and *many* engines read it through one shared catalog. You pay for compute only when you query. That's the reason "data lake on S3" is the answer to most "store everything, analyse it many ways, lowest cost" questions.

---

## 2. Ingestion and streaming (recap from Day 8 + new pieces)

| Service | Latency | You manage | Replay? | Typical exam cue |
| --- | --- | --- | --- | --- |
| **Kinesis Data Streams** | Real-time (~sub-second to ~1 s) | Shards (provisioned) or nothing (on-demand mode); consumers | **Yes** — retention 24 h default, extendable (check current docs for max) | "Multiple consumers read the same stream", "replay", "ordering per key", "custom real-time processing" |
| **Amazon Data Firehose** | Near-real-time (buffered by size/time) | Nothing — fully managed, auto-scales | No | "Load streaming data into S3 / Redshift / OpenSearch / Splunk with **no code** and least ops"; optional Lambda transform, format conversion to **Parquet/ORC**, dynamic partitioning |
| **Amazon MSK** (+ MSK Serverless) | Real-time | Brokers (provisioned) or nothing (Serverless) | Yes | "Existing Apache **Kafka** workload", "Kafka APIs", "migrate Kafka with minimal code changes" |
| **Amazon Managed Service for Apache Flink** (formerly Kinesis Data Analytics) | Real-time | Application code (Flink/SQL/Studio notebooks) | — | "Real-time aggregation / windowed analytics / anomaly detection on a stream" |
| **AWS DMS** | Continuous (CDC) | Replication instance or DMS Serverless | — | "Replicate an operational DB into the data lake / warehouse continuously" |
| **Zero-ETL integrations** | Near-real-time | Nothing | — | "Aurora/RDS/DynamoDB data available in **Redshift** for analytics without building pipelines" |

**How Firehose works and why it's the "least ops" answer:** producers `PutRecord` (or point Data Streams/MSK/CloudWatch Logs/EventBridge at it) → Firehose buffers until a size or time threshold → optionally invokes a Lambda to transform each batch → optionally converts JSON to Parquet/ORC using a Glue table schema → writes to the destination. For Redshift it stages in S3 and issues `COPY`. Failed records go to an S3 error prefix. No shards, no consumers, no servers.

> **Trap — "real-time" vs "near-real-time".** If the question needs sub-second processing or multiple independent consumers, Firehose is wrong (it buffers, and a record goes to one destination). If it just needs data *landed* in S3/Redshift/OpenSearch with least effort, Firehose is right and Data Streams + custom consumers is over-engineering.

> **Trap — Firehose is not a stream store.** You can't re-read data from Firehose. If replay is required, put Kinesis Data Streams (or MSK) in front, with Firehose as one of its consumers.

---

## 3. Storage, catalog and governance: S3 + Glue + Lake Formation

### Data-lake layout that the exam rewards

```
s3://acme-lake/
  raw/clicks/                 ← as ingested (JSON, CSV)
  clean/clicks/               ← deduped, typed
  curated/clicks/year=2026/month=11/day=10/part-0001.snappy.parquet
                ▲ Hive-style partitions          ▲ columnar + compressed
```

Why each choice matters:

| Choice | Effect | Why |
| --- | --- | --- |
| **Columnar format** (Parquet, ORC) | 30–90% less data scanned (typical) | Query engines read only the columns referenced |
| **Compression** (Snappy, ZSTD, GZIP) | Less storage, less scanned | Fewer bytes from S3 |
| **Partitioning** (`year=/month=/day=`) | Engines skip whole prefixes | `WHERE day='10'` reads one folder, not the table |
| **Right-sized files** (~128 MB–1 GB, not millions of tiny files) | Faster queries | Fewer S3 GETs and less per-file overhead |
| **Open table format** (Apache Iceberg) | ACID updates/deletes, time travel, schema evolution on S3 | Athena, Glue, EMR, Redshift all read Iceberg |

### AWS Glue — the serverless data-integration service

| Component | What it does | Cue |
| --- | --- | --- |
| **Data Catalog** | Central metadata store (databases, tables, schemas, partitions); Hive-metastore compatible. Used by Athena, Redshift Spectrum, EMR, Lake Formation | "Central schema/metadata repository for the data lake" |
| **Crawlers** | Scan S3/JDBC sources, infer schema, create/update catalog tables and partitions on a schedule | "Automatically discover schema", "new partitions arrive daily" |
| **ETL jobs** | Serverless Spark (or Python shell) jobs; **job bookmarks** process only new data | "Serverless ETL", "convert CSV to Parquet", "no cluster to manage" |
| **Glue Studio / DataBrew** | Visual job authoring / no-code data prep with 250+ transforms | "Analysts clean data without writing code" → DataBrew |
| **Glue Data Quality** | Rule-based checks on data | "Validate data quality in the pipeline" |
| **Workflows / triggers** | Orchestrate crawlers and jobs | Simple Glue-only orchestration (Step Functions or MWAA for broader) |

### AWS Lake Formation — permissions for the lake

S3 bucket policies work at the **object/prefix** level. Analysts think in **tables and columns**. Lake Formation bridges that: you register S3 locations, and Lake Formation grants `SELECT` on databases, tables, **columns**, **rows** (data filters) and **cells** — enforced when Athena, Redshift Spectrum, EMR or Glue read through the catalog.

```
             ┌───────────── Lake Formation ─────────────┐
 Analyst ──► │ GRANT SELECT (order_id, total)           │ ──► Athena returns only
 (role)      │   ON curated.orders                      │     those columns, only
             │   WHERE region = 'IN'   (row filter)     │     region IN rows
             └──────────────────────────────────────────┘
```

- **LF-Tags** (tag-based access control) scale permissions: tag columns `pii=true` and grant on tags rather than on hundreds of tables.
- **Cross-account sharing** of catalog tables (uses AWS RAM under the hood) — the answer for "central data lake account shares curated tables with consumer accounts without copying data".
- Blueprints can ingest from databases into the lake.

> **Trap:** "Restrict analysts to specific *columns* of data in S3" → Lake Formation, **not** a bucket policy (bucket policies can't see columns) and not "split into separate buckets" (high ops).

---

## 4. Query and processing engines

### Amazon Athena — serverless SQL on S3

- Presto/Trino-based SQL directly over S3 using the Glue Data Catalog. No infrastructure; **pay per data scanned** (check current docs for price) — or provisioned capacity for predictable heavy use.
- **Make it cheap/fast:** Parquet/ORC, compression, partitions (or **partition projection** for huge partition counts), `CTAS` / `INSERT INTO` to convert data.
- **Workgroups** separate teams, set per-query and per-workgroup **data-scanned limits**, and route results to separate S3 locations — the answer for "control Athena costs per team".
- **Federated query** (Lambda-based connectors) queries DynamoDB, RDS, Redshift, CloudWatch Logs, on-prem JDBC sources in place.
- Athena also runs **Apache Spark** notebooks; and queries ALB/CloudFront/VPC Flow/CloudTrail logs stored in S3 — a classic "ad hoc analysis of logs in S3" answer.

```sql
-- Convert raw CSV to partitioned Parquet in one statement (CTAS)
CREATE TABLE curated.clicks
WITH (format = 'PARQUET', write_compression = 'SNAPPY',
      external_location = 's3://acme-lake/curated/clicks/',
      partitioned_by = ARRAY['day'])
AS SELECT user_id, page, ts, date_format(ts, '%Y-%m-%d') AS day
FROM raw.clicks;
```

### Amazon Redshift — the data warehouse (Day 6 recap + analytics pieces)

| Feature | What / why | Cue |
| --- | --- | --- |
| **RA3 nodes + managed storage** | Compute and storage scale separately; hot data cached on local SSD | "Warehouse grows, compute doesn't" |
| **Redshift Serverless** | Capacity in RPUs, auto-scales, pay for use | "Intermittent/unpredictable warehouse usage, no cluster management" |
| **Redshift Spectrum** | Query S3 data (via Glue Catalog) **from Redshift SQL** and join with local tables | "Join warehouse tables with infrequently used historical data in S3 without loading it" |
| **Concurrency scaling** | Transient extra clusters for bursts of queries | "Many concurrent BI users at month-end" |
| **Data sharing** | Live, read-only access to data across clusters/accounts/Regions, no copies | "Share warehouse data with another team's cluster without ETL" |
| **Zero-ETL / federated query** | Data from Aurora/RDS/DynamoDB lands in Redshift automatically / query live operational DBs | "Analytics on transactional data without pipelines" |
| **Materialized views, sort/dist keys, AQUA-style caching** | Query performance | "Speed up repeated dashboards" |
| **Snapshots** (automated, cross-Region copy) | Backup/DR | "Redshift DR in another Region" |

**Athena vs Redshift vs Spectrum** — the most-tested split:

| Need | Pick | Why |
| --- | --- | --- |
| Ad hoc, infrequent SQL on S3; no infra; pay per query | **Athena** | Serverless, zero standing cost |
| Complex, frequent, high-concurrency BI over structured data, sub-second dashboards | **Redshift** (provisioned or Serverless) | MPP, columnar local storage, caching, workload management |
| Already on Redshift, need to include S3 data | **Redshift Spectrum** | Extends warehouse queries to the lake |

### Amazon EMR — managed big-data frameworks

- Managed **Spark, Hive, Presto/Trino, HBase, Flink, Hadoop**. Deployment options: **EMR on EC2** (most control), **EMR on EKS** (share a Kubernetes cluster), **EMR Serverless** (no cluster sizing).
- Node types on EC2: **primary** (coordinates), **core** (runs tasks *and* stores HDFS data — don't use Spot for critical core nodes), **task** (compute only — **ideal for Spot**).
- **EMRFS** reads/writes S3 directly, so clusters can be **transient**: spin up, process, terminate; data stays in S3.
- Cue: "existing Hadoop/Spark jobs on-prem, migrate with minimal changes", "petabyte-scale processing with custom frameworks", "full control over the cluster".

**Glue ETL vs EMR:** Glue = serverless Spark for ETL with least ops; EMR = broad framework choice, tuning control, long-running clusters, or lift-and-shift of Hadoop ecosystems.

### Amazon OpenSearch Service — search and log analytics

- Managed OpenSearch (Elasticsearch-compatible fork) with **OpenSearch Dashboards**. Deploy as domains (Multi-AZ with standby for HA) or **OpenSearch Serverless** (collections; no capacity planning).
- Storage tiers: hot → **UltraWarm** → **cold** for cheaper retention of older logs.
- Cue: "full-text search", "fuzzy/relevance search on product catalog", "log analytics with near-real-time dashboards", "search DynamoDB data" (stream changes via DynamoDB Streams / zero-ETL into OpenSearch).
- Ingestion: Data Firehose, **OpenSearch Ingestion** (managed Data Prepper), CloudWatch Logs subscription.

> **Trap:** OpenSearch is not a primary database and not a general SQL warehouse. "Search" or "log analytics dashboards" → OpenSearch; "SQL BI reporting" → Redshift/Athena + QuickSight.

### Amazon QuickSight — serverless BI

- Dashboards, ad hoc visuals, ML insights, natural-language Q&A. Connects to Athena, Redshift, RDS/Aurora, S3, OpenSearch, SaaS sources.
- **SPICE** — in-memory engine; import data for fast dashboards without hammering the source.
- **Row-level security** and column-level security; pay-per-user/session pricing (check current docs).
- Naming note: AWS has folded QuickSight into its **Amazon Quick** suite (as *Quick Sight*). SAA-C03 questions still say **QuickSight** — same answer.

### Other names to recognise

| Service | One-liner |
| --- | --- |
| **AWS Data Exchange** | Subscribe to third-party datasets delivered into S3/Redshift |
| **Amazon MWAA** | Managed Apache Airflow for pipeline orchestration |
| **AWS Glue DataBrew** | No-code visual data preparation |
| **Amazon SageMaker (Unified Studio / lakehouse)** | AWS's umbrella for analytics + ML workspaces; rarely the SAA answer unless ML is named |

---

## 5. Cost-management tools: which one does what

```
 PLAN ─────────────► WATCH ──────────────────► ANALYSE ───────────────► OPTIMISE ──────────────► COMMIT
 Pricing Calculator  Budgets (+ actions)       Cost Explorer            Compute Optimizer         Savings Plans
                     Cost Anomaly Detection    Data Exports (CUR 2.0)   Trusted Advisor           Reserved Instances
                     Free Tier alerts          + Athena / QuickSight    Cost Optimization Hub     (RDS, ElastiCache,
                                               Cost allocation tags     S3 Storage Lens           Redshift, OpenSearch)
                                               Cost Categories
```

| Tool | What it does | Exam cue |
| --- | --- | --- |
| **AWS Pricing Calculator** | Estimate cost of an architecture *before* building | "Estimate monthly cost of a proposed design" |
| **AWS Budgets** | Thresholds on **cost, usage, RI/SP utilization or coverage**; alerts on *actual* or *forecasted* spend via email/SNS/Chatbot | "Alert when forecasted spend exceeds $X" |
| **Budget actions** | When a budget is breached, **automatically** apply an IAM policy or **SCP**, or stop EC2/RDS instances (optionally with approval) | "Automatically prevent further provisioning when the budget is exceeded" |
| **Cost Anomaly Detection** | ML baselines per service/account/tag/cost category; alerts on **unusual spikes** with root-cause hints | "Detect unexpected spend without setting fixed thresholds" |
| **Cost Explorer** | Visualise/filter/group up to ~13 months of history and forecast; RI/SP **recommendations**, utilization and coverage reports; rightsizing recommendations | "Analyse spend trends by service/account/tag", "which Savings Plan should we buy?" |
| **Data Exports (CUR 2.0) / Cost and Usage Report** | The most granular billing data (hourly, per resource, with tags) delivered to **S3**; query with **Athena**, visualise with **QuickSight** | "Most detailed billing data", "custom chargeback analysis" |
| **Cost allocation tags** | User-defined (e.g. `CostCenter`) or AWS-generated tags; **must be activated in the Billing console of the management (payer) account** before they appear in Cost Explorer/CUR; not retroactive | "Break down cost by department/project" |
| **Cost Categories** | Rules that map accounts/tags/services into business groupings | "Group costs into business units across many accounts" |
| **AWS Billing Conductor** | Custom pro forma rates and billing groups for showback/chargeback | "Resell/chargeback with custom rates" |
| **AWS Compute Optimizer** | ML rightsizing from CloudWatch utilisation for **EC2, ASGs, EBS, Lambda, ECS on Fargate, RDS/Aurora** and more (check current docs) | "Rightsize based on actual utilisation" |
| **AWS Trusted Advisor** | Best-practice checks: cost, performance, security, fault tolerance, **service limits (quotas)**, operational excellence. Core checks for all; **full set requires Business support or higher** (check current support plan names) | "Idle load balancers, underutilised instances, unassociated EIPs, open SGs, approaching quotas" |
| **Cost Optimization Hub** | One place aggregating savings recommendations across accounts/Regions | "Consolidated view of all optimisation recommendations in the org" |
| **S3 Storage Lens** | Org-wide S3 usage/activity metrics and recommendations | "Find buckets with no lifecycle rules / incomplete multipart uploads across the org" |
| **Service Quotas** | View/request quota increases; CloudWatch alarms on usage | "Proactively avoid hitting limits" |

> **Trap — Budgets vs Anomaly Detection vs Cost Explorer.** Budgets = *you* choose the threshold. Anomaly Detection = *ML* decides what's abnormal. Cost Explorer = *you* look and analyse; it alerts nothing.

> **Trap — tags.** Tagging resources is not enough; tags appear in billing only after they're **activated** as cost allocation tags in the payer account, and only from activation onward.

### Commitment discounts (Day 2 recap, cost-domain view)

| Option | Flexibility | Covers | Use when |
| --- | --- | --- | --- |
| **Compute Savings Plans** | Any instance family, size, OS, tenancy, **Region**; also **Fargate and Lambda** | EC2, Fargate, Lambda | Steady spend but architecture may change |
| **EC2 Instance Savings Plans** | One instance **family in one Region**; any size/OS/AZ | EC2 | Stable family, max discount for SPs |
| **Standard RIs** | Fixed type (size flexibility within family for Linux regional RIs); can sell on Marketplace | EC2 | Very stable; also capacity reservation if zonal |
| **Convertible RIs** | Exchange for other types | EC2 | Stable spend, may change family |
| **Database RIs** | Per engine/class | RDS, Aurora, ElastiCache, Redshift, OpenSearch, MemoryDB, DynamoDB (reserved capacity) | Steady DB baselines |
| **Database Savings Plans** (launched Dec 2025) | Spend commitment across several managed DB services | Check current docs for covered services/terms | Newer than the exam guide — recognise it, but RIs remain the classic SAA answer for databases |
| **SageMaker Savings Plans** | SageMaker usage | ML | ML-specific |
| **On-Demand Capacity Reservations** | Reserve capacity in an AZ, **no discount by itself** (pair with SPs) | EC2 | "Guarantee capacity for an event" |

Multi-account rule: in Organizations with consolidated billing, RI and SP discounts **share across accounts by default** (sharing can be turned off per account).

### Cost-optimisation patterns that show up as answers

| Pattern | Saves on |
| --- | --- |
| S3 Gateway endpoint instead of NAT for S3/DynamoDB traffic | NAT processing charges |
| Lifecycle rules + Intelligent-Tiering; abort incomplete multipart uploads | S3 storage |
| Spot for stateless/batch/EMR task nodes; mixed instances policy | Compute |
| Graviton (arm64) instances / Lambda | Price-performance |
| gp2 → gp3 | EBS |
| Turn off non-prod at night (Instance Scheduler / EventBridge + Lambda) | Idle compute |
| CloudFront in front of S3/ALB | Data transfer out, origin load |
| Keep traffic in one AZ where HA allows; use VPC endpoints | Data transfer |
| Athena on Parquet/partitioned data; workgroup limits | Query scan cost |
| Serverless (Lambda, Fargate, Aurora Serverless v2, DynamoDB on-demand) for spiky/idle workloads | Idle capacity |

---

## 6. Exam traps roundup

1. **Kinesis Data Streams vs Data Firehose** — replay/multiple consumers/sub-second → Streams; "load into S3/Redshift/OpenSearch, no code" → Firehose.
2. **Kinesis vs MSK** — "Kafka" anywhere in the question → MSK.
3. **Athena vs Redshift** — ad hoc + serverless + S3 → Athena; heavy repeated BI with concurrency → Redshift.
4. **Spectrum vs loading data** — "infrequently accessed history in S3, join with warehouse" → Spectrum, not `COPY` everything.
5. **Glue vs EMR** — "serverless ETL, least ops" → Glue; "existing Hadoop/Spark, custom frameworks, cluster control" → EMR.
6. **Lake Formation vs bucket policy** — columns/rows/table-level grants → Lake Formation.
7. **OpenSearch** = search & log analytics, not a warehouse.
8. **QuickSight** = BI dashboards; **CloudWatch dashboards** = operational metrics. Different audiences.
9. **Athena costs** — biggest lever is scanning less (Parquet, partitions, compression), not "bigger Athena".
10. **Budgets vs Anomaly Detection** — fixed threshold vs ML-detected spike.
11. **CUR/Data Exports** is the most granular billing data; **Cost Explorer** is for visual analysis and recommendations.
12. **Cost allocation tags** must be activated in the management account; not retroactive.
13. **Trusted Advisor full checks** need Business support or higher; **Compute Optimizer** is the rightsizing specialist.
14. **Capacity Reservations** guarantee capacity but give no discount by themselves.
15. **Compute SP vs EC2 Instance SP** — "may move to Fargate/Lambda or change Region" → Compute SP.

---

## 7. Exam decision table

| If the question says… | Pick |
| --- | --- |
| Load streaming data into S3 / Redshift / OpenSearch / Splunk with no code, least ops | **Amazon Data Firehose** |
| Convert streaming JSON to Parquet before landing in S3 | **Data Firehose** record format conversion (Glue schema) |
| Real-time stream, multiple consumers, replay, per-key ordering | **Kinesis Data Streams** |
| Existing Apache Kafka, minimal code changes | **Amazon MSK** (MSK Serverless for least ops) |
| Real-time windowed aggregations / anomaly detection on a stream | **Managed Service for Apache Flink** |
| Ad hoc SQL on data in S3, serverless, pay per query | **Athena** |
| Reduce Athena cost / speed up queries | **Parquet/ORC + partitioning + compression**; CTAS |
| Limit how much data each team's Athena queries can scan | **Athena workgroups** with data usage limits |
| Query DynamoDB/RDS/on-prem with SQL without moving data | **Athena federated query** |
| Petabyte-scale data warehouse, complex BI, many concurrent users | **Redshift** (+ concurrency scaling) |
| Warehouse with unpredictable/intermittent usage, no cluster ops | **Redshift Serverless** |
| Join Redshift tables with rarely used history in S3 | **Redshift Spectrum** |
| Share live Redshift data with another account/cluster, no copies | **Redshift data sharing** |
| Aurora data available in Redshift for analytics without ETL | **Aurora zero-ETL integration with Redshift** |
| Central metadata/schema repository for the lake | **Glue Data Catalog** |
| Automatically discover schema and new partitions | **Glue crawler** |
| Serverless ETL (CSV → Parquet), process only new data | **Glue ETL job** with **job bookmarks** |
| Analysts prepare data visually, no code | **Glue DataBrew** |
| Column/row-level permissions on the data lake; share tables cross-account | **Lake Formation** (LF-Tags for scale) |
| Migrate on-prem Hadoop/Spark with minimal change; full cluster control | **EMR** (Spot task nodes for cost; transient clusters with EMRFS) |
| Spark/Hive without managing clusters | **EMR Serverless** (or Glue for ETL) |
| Full-text search, log analytics dashboards | **OpenSearch Service** (Serverless for least ops) |
| BI dashboards for business users, row-level security | **QuickSight** (SPICE for speed) |
| Estimate cost before building | **Pricing Calculator** |
| Alert when actual/forecast spend exceeds a threshold | **AWS Budgets** |
| Automatically stop instances / apply SCP when budget is exceeded | **Budget actions** |
| Detect unusual spend spikes without fixed thresholds | **Cost Anomaly Detection** |
| Analyse spend by service/account/tag; SP/RI recommendations | **Cost Explorer** |
| Most granular hourly billing data, query with SQL | **Data Exports (CUR 2.0) → S3 → Athena** |
| Cost per department/project | **Cost allocation tags** (activated in payer account) / **Cost Categories** |
| Rightsize EC2/EBS/Lambda/ASG from utilisation | **Compute Optimizer** |
| Idle LBs, unassociated EIPs, open SGs, quota limits — one checklist | **Trusted Advisor** |
| Steady compute, may change family/Region or move to Fargate/Lambda | **Compute Savings Plan** |
| Guarantee capacity in an AZ for an event (no long commitment) | **On-Demand Capacity Reservation** |

---

## 8. Lab (30 min, near-zero cost)

Goal: build a tiny data lake — S3 → Glue Data Catalog → Athena — see partitioning and Parquet cut the data scanned, then set a budget with an alert.

> **Cost note:** Athena bills per data scanned (with a per-query minimum) and Glue crawlers bill per DPU-hour; with a few KB of data this is cents at most. Budgets alerts are free for the first few budgets (check current docs). New AWS accounts now use a credit-based free plan — check your account's free-tier status before starting.

### Console + CLI steps

1. **Create a bucket and sample data** (CloudShell):
   ```bash
   B=day12-lake-$RANDOM; aws s3 mb s3://$B
   for d in 2026-11-08 2026-11-09 2026-11-10; do
     for i in $(seq 1 200); do
       echo "{\"user_id\":$i,\"page\":\"/p$((i%7))\",\"ms\":$((RANDOM%900))}"
     done > clicks.json
     aws s3 cp clicks.json s3://$B/raw/clicks/day=$d/clicks.json   # partition value lives in the path
   done
   echo $B
   ```
2. **Athena setup:** Athena console → *Settings* → query result location `s3://<bucket>/athena-results/`. Create a **workgroup** `day12` and set a *per-query data usage limit* (e.g. 10 MB) — you've just met the cost-control cue.
3. **Catalog it (two options):**
   - *Crawler:* Glue → Crawlers → source `s3://<bucket>/raw/clicks/`, new IAM role, target database `day12` → run. Note it creates table `clicks` with partition column `day`.
   - *Or DDL in Athena (no crawler cost):*
     ```sql
     CREATE DATABASE day12;
     CREATE EXTERNAL TABLE day12.clicks (user_id int, page string, ms int)
     PARTITIONED BY (day string)
     ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
     LOCATION 's3://<bucket>/raw/clicks/';
     MSCK REPAIR TABLE day12.clicks;   -- loads the three partitions
     ```
4. **Compare scan sizes** (watch *Data scanned* under each result):
   ```sql
   SELECT page, avg(ms) FROM day12.clicks GROUP BY page;                    -- scans all partitions
   SELECT page, avg(ms) FROM day12.clicks WHERE day='2026-11-10' GROUP BY page; -- one partition
   ```
5. **Convert to Parquet with CTAS**, then rerun the query against `clicks_parquet` and compare:
   ```sql
   CREATE TABLE day12.clicks_parquet
   WITH (format='PARQUET', write_compression='SNAPPY',
         external_location='s3://<bucket>/curated/clicks/', partitioned_by=ARRAY['day'])
   AS SELECT user_id, page, ms, day FROM day12.clicks;
   SELECT page, avg(ms) FROM day12.clicks_parquet WHERE day='2026-11-10' GROUP BY page;
   ```
6. **Budget:** Billing → *Budgets* → *Create budget* → cost budget, monthly, **$5**, alert at 80% of **forecasted** spend to your email. Look at (don't create) **Budget actions** to see the "apply SCP / stop EC2" options.
7. **Look around (read-only, 3 min):** Cost Explorer → group by *Service*; *Cost Anomaly Detection* → see the default AWS services monitor; Billing → *Cost allocation tags*; *Compute Optimizer* opt-in page; *Trusted Advisor* → cost checks.

### Terraform variant

```hcl
provider "aws" { region = "ap-south-1" }

resource "random_id" "s" { byte_length = 3 }

resource "aws_s3_bucket" "lake" {
  bucket        = "day12-lake-${random_id.s.hex}"
  force_destroy = true
}

resource "aws_s3_object" "clicks" {
  for_each = toset(["2026-11-08", "2026-11-09", "2026-11-10"])
  bucket   = aws_s3_bucket.lake.id
  key      = "raw/clicks/day=${each.key}/clicks.json"
  content  = join("\n", [for i in range(1, 201) :
    jsonencode({ user_id = i, page = "/p${i % 7}", ms = (i * 37) % 900 })])
}

resource "aws_glue_catalog_database" "db" { name = "day12" }

resource "aws_glue_catalog_table" "clicks" {
  name          = "clicks"
  database_name = aws_glue_catalog_database.db.name
  table_type    = "EXTERNAL_TABLE"
  partition_keys {
    name = "day"
    type = "string"
  }
  storage_descriptor {
    location      = "s3://${aws_s3_bucket.lake.id}/raw/clicks/"
    input_format  = "org.apache.hadoop.mapred.TextInputFormat"
    output_format = "org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat"
    ser_de_info { serialization_library = "org.openx.data.jsonserde.JsonSerDe" }
    columns {
      name = "user_id"
      type = "int"
    }
    columns {
      name = "page"
      type = "string"
    }
    columns {
      name = "ms"
      type = "int"
    }
  }
}

resource "aws_athena_workgroup" "wg" {
  name          = "day12"
  force_destroy = true
  configuration {
    enforce_workgroup_configuration = true
    bytes_scanned_cutoff_per_query  = 10485760   # 10 MB guardrail
    result_configuration {
      output_location = "s3://${aws_s3_bucket.lake.id}/athena-results/"
    }
  }
}

resource "aws_budgets_budget" "monthly" {
  name         = "day12-monthly"
  budget_type  = "COST"
  limit_amount = "5"
  limit_unit   = "USD"
  time_unit    = "MONTHLY"
  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "FORECASTED"
    subscriber_email_addresses = ["you@example.com"]   # change me
  }
}
```

After `terraform apply`, run `MSCK REPAIR TABLE day12.clicks;` in the `day12` workgroup, then steps 4–5 above.

### Teardown

- Athena: `DROP TABLE day12.clicks_parquet; DROP TABLE day12.clicks; DROP DATABASE day12;` then delete the `day12` workgroup.
- Glue: delete the crawler and its IAM role (if you used one).
- S3: `aws s3 rb s3://$B --force` (removes raw, curated and Athena results).
- Budgets: delete the $5 budget, or keep it — it's a useful guardrail.
- Terraform: `terraform destroy`.

---

## 9. Practice questions (lesson topics — 15 min, untimed)

Do these before the exam to warm up on today's material.

**Q1.** A company ingests clickstream events and must deliver them to Amazon S3 in Apache Parquet format, partitioned by date, for querying with Athena. The team wants the LEAST operational overhead. What should the architect use?
A. Kinesis Data Streams with a fleet of EC2 consumers that write Parquet files
B. Amazon Data Firehose with record format conversion and dynamic partitioning
C. Amazon MSK with Kafka Connect on EC2
D. AWS Glue crawlers running every minute

**Q2.** Analysts run ad hoc SQL queries a few times a week against 20 TB of CSV log files in S3. Query costs are high and queries are slow. Which actions reduce cost the MOST? *(Choose two.)*
A. Convert the data to Apache Parquet with compression
B. Partition the data by date and filter on the partition column
C. Move the data to S3 Glacier Flexible Retrieval
D. Increase the Athena query timeout
E. Load the data into Amazon RDS

**Q3.** A Redshift cluster holds two years of sales data. Finance occasionally needs to join it with seven years of older history stored as Parquet in S3, without loading that history into the cluster. What should the architect use?
A. Redshift Spectrum with the history registered in the Glue Data Catalog
B. Redshift COPY of the history into a new table
C. Amazon EMR with Hive
D. Amazon OpenSearch Service

**Q4.** A central data lake account must let analysts in several business-unit accounts query curated tables, but each unit may see only certain columns and only rows for its own region. Data must not be copied. What should the architect use?
A. S3 bucket policies granting each account access to specific prefixes
B. AWS Lake Formation with column-level permissions, row-level data filters and cross-account grants
C. Separate copies of the tables in each account via S3 Replication
D. IAM policies with `s3:prefix` conditions

**Q5.** A company runs dozens of Apache Spark and Hive jobs on an on-premises Hadoop cluster and wants to move them to AWS with minimal code changes while reducing cost for the compute-heavy, interruption-tolerant stages. What should the architect recommend?
A. Rewrite the jobs as AWS Lambda functions
B. Amazon EMR on EC2 reading from S3 via EMRFS, with Spot Instances for task nodes
C. Amazon Redshift with stored procedures
D. AWS Glue DataBrew

**Q6.** A product team needs relevance-ranked, typo-tolerant search over a product catalog stored in DynamoDB. What should the architect add?
A. A DynamoDB global secondary index on product name
B. Amazon Athena federated query
C. Amazon OpenSearch Service fed by DynamoDB Streams (or zero-ETL integration)
D. Amazon QuickSight

**Q7.** A company wants to be alerted when a sudden, unusual increase in spend occurs in any account, without having to define thresholds for each service. Which tool should it use?
A. AWS Budgets
B. AWS Cost Anomaly Detection
C. AWS Pricing Calculator
D. AWS Trusted Advisor

**Q8.** A company tags resources with `CostCenter`, but the tag does not appear as a filter in Cost Explorer. What is the MOST likely cause?
A. Tags are not supported by Cost Explorer
B. The tag has not been activated as a cost allocation tag in the management account's Billing console
C. The resources must be recreated after tagging
D. Cost Explorer only supports AWS-generated tags

**Q9.** In a sandbox account, developers must be prevented from launching new resources automatically once monthly spend exceeds $1,000. Which solution meets this with the LEAST custom code?
A. A CloudWatch billing alarm that emails the team
B. An AWS Budgets budget with a budget action that applies a restrictive SCP to the sandbox account
C. A daily Lambda function that reads Cost Explorer and terminates instances
D. Trusted Advisor cost checks

**Q10.** A company's EC2 usage is steady at about $10,000/month, but it plans to migrate several workloads to AWS Fargate and Lambda over the next year and may move Regions. Which purchase gives the best discount while preserving that flexibility?
A. EC2 Instance Savings Plans
B. Standard Reserved Instances
C. Compute Savings Plans
D. On-Demand Capacity Reservations

<details>
<summary><strong>Answers</strong></summary>

1. **B** — Firehose converts JSON to Parquet using a Glue schema and writes date-partitioned prefixes, with nothing to manage. A and C need custom consumers/connectors; crawlers catalog data, they don't ingest it.
2. **A, B** — Athena bills by data scanned. Columnar + compression and partition pruning cut the bytes read. Glacier classes can't be queried directly by Athena; timeouts and RDS don't help.
3. **A** — Spectrum queries external S3 tables from Redshift and joins them with local data. COPY loads everything (cost, storage); EMR and OpenSearch aren't warehouse joins.
4. **B** — Lake Formation grants at column/row level and shares catalog tables cross-account without copying. Bucket policies only understand object prefixes.
5. **B** — EMR runs Spark/Hive as-is; EMRFS lets clusters be transient over S3; task nodes hold no HDFS data, so Spot interruptions are safe.
6. **C** — OpenSearch provides full-text, relevance and fuzzy search. A GSI only supports exact key/range lookups.
7. **B** — Anomaly Detection uses ML baselines and needs no fixed thresholds. Budgets need you to set a threshold.
8. **B** — User-defined tags must be activated as cost allocation tags in the payer account and only show data from activation onward.
9. **B** — Budget actions can automatically apply an IAM policy or SCP (or stop instances). A only notifies; C is custom code.
10. **C** — Compute Savings Plans apply across EC2 families and Regions and to Fargate and Lambda. EC2 Instance SPs and Standard RIs are tied to EC2 (and a family/Region).

</details>

---

## 10. Practice Exam 2 — 65 questions, 130 minutes

**Rules:** one sitting, timer on, no notes, no console, no searching. ~2 minutes per question; past 3 minutes, pick your best guess, note the number, move on. Questions marked *(Choose two.)* need both picks correct to score. Write answers 1–65 on paper before opening the key.

Domain mix matches the real exam: 20 Secure · 17 Resilient · 15 High-performing · 13 Cost-optimized, shuffled. Content draws on Days 1–12, weighted toward Days 8–12.

**Pacing checkpoints:** Q22 by ~0:45 · Q44 by ~1:30 · Q65 by ~2:00 · last 10 minutes for flagged questions.

**Q1.** A finance team needs hourly, resource-level billing data that includes cost allocation tags, so it can build custom chargeback reports using SQL. What should the architect set up?
A. AWS Cost Explorer with daily granularity
B. AWS Budgets reports emailed weekly
C. AWS Trusted Advisor cost optimization checks
D. AWS Data Exports (CUR 2.0) delivered to Amazon S3 and queried with Amazon Athena

**Q2.** A company stores regulated documents in S3. It must control the key policy, see every use of the encryption key in CloudTrail, and be able to disable the key immediately if needed. Which encryption option meets these requirements with the LEAST operational overhead?
A. SSE-S3
B. SSE-KMS with a customer managed key
C. SSE-C with keys stored in the application
D. Client-side encryption with keys stored in a configuration file

**Q3.** A web tier submits orders directly to a backend fleet. During flash sales the backend is overwhelmed and some orders are lost. Orders must never be lost and the backend should scale with demand. What should the architect do?
A. Put an Amazon SQS queue between the tiers and scale the backend Auto Scaling group on queue depth (backlog per instance)
B. Replace the backend with larger instance types
C. Publish orders to an SNS topic subscribed by the backend instances over HTTP
D. Enable cross-zone load balancing on the backend load balancer

**Q4.** A multiplayer game uses UDP and serves players worldwide from two Regions. It needs two static anycast IP addresses that players' clients can allow-list, and failover between Regions within seconds. What should the architect use?
A. Amazon CloudFront with two origins
B. Route 53 latency-based routing
C. AWS Global Accelerator with endpoint groups in both Regions
D. An Application Load Balancer in each Region

**Q5.** A public web application behind an Application Load Balancer is receiving SQL injection attempts and a flood of requests from a few IP addresses. What should the architect implement?
A. AWS WAF web ACL on the ALB with AWS managed rule groups (SQL injection) and a rate-based rule
B. AWS Shield Standard
C. Amazon GuardDuty
D. Network ACL rules on the public subnets for each attacking IP as it appears

**Q6.** Users upload images to an S3 bucket a few hundred times a day at unpredictable times; each image needs a thumbnail generated within seconds. Which solution is MOST cost-effective?
A. An Auto Scaling group of EC2 instances polling the bucket every minute
B. An Amazon EMR cluster running continuously
C. An S3 event notification that invokes an AWS Lambda function
D. Amazon ECS on EC2 with Reserved Instances

**Q7.** A company runs an EC2-based application with an RDS database. For DR in a second Region, it can tolerate an RPO of 1 hour and an RTO of 8 hours, and wants the lowest possible cost. What should the architect choose?
A. Multi-site active-active across both Regions
B. Warm standby with a scaled-down fleet running in the DR Region
C. Pilot light with an always-running cross-Region read replica and pre-provisioned load balancer
D. Backup and restore using AWS Backup with hourly backups copied to the DR Region

**Q8.** Five different applications consume the same Kinesis data stream. Each needs low-latency reads, and they are throttling each other on read throughput. What should the architect do?
A. Register each application as an enhanced fan-out consumer
B. Replace the stream with Amazon Data Firehose
C. Increase the data retention period
D. Move the data to an SQS FIFO queue

**Q9.** A security team wants automatic detection of EC2 instances that communicate with known command-and-control IP addresses or perform cryptocurrency mining, across all accounts, with no agents to manage. Which service should it enable?
A. Amazon Inspector
B. Amazon GuardDuty with a delegated administrator in Organizations
C. Amazon Macie
D. AWS Config

**Q10.** Several teams share an AWS account and run Amazon Athena queries. One team's poorly written queries scanned many terabytes last month. The company wants to cap the data each team's queries can scan. What should the architect do?
A. Create an IAM policy that denies `athena:StartQueryExecution` after business hours
B. Create an AWS Budgets alert for Athena
C. Create an Athena workgroup per team with per-query and per-workgroup data usage limits
D. Migrate all data to Amazon Redshift

**Q11.** A company hosts its website on EC2 behind an ALB in one Region. If the ALB becomes unhealthy, users must be sent to a static "maintenance" site hosted on S3. What should the architect configure?
A. Route 53 weighted routing with weights 100/0
B. Route 53 failover routing: primary record for the ALB with a health check, secondary record for the S3 website
C. Route 53 geolocation routing
D. A CloudFront distribution with geo restriction

**Q12.** A mobile app must let customers sign up and sign in with email or social identity providers, and then upload photos directly to their own prefix in an S3 bucket using temporary AWS credentials. What should the architect use?
A. An IAM user for each customer
B. IAM Identity Center with permission sets
C. AWS Secrets Manager storing per-customer access keys
D. Amazon Cognito user pools for sign-in and Cognito identity pools to obtain temporary credentials scoped by identity

**Q13.** A team is containerising a dozen microservices and does not want to provision, patch or scale any servers or clusters of instances. Which compute option should the architect choose?
A. Amazon ECS on AWS Fargate
B. Amazon ECS on EC2 with an Auto Scaling group capacity provider
C. Amazon EKS with self-managed node groups
D. Docker on EC2 instances launched from a custom AMI

**Q14.** A consumer application processes messages from an SQS standard queue. A few malformed messages fail every time, are retried endlessly and delay other work. What should the architect do?
A. Increase the queue's visibility timeout to 12 hours
B. Switch to an SQS FIFO queue
C. Configure a dead-letter queue with a redrive policy and an appropriate `maxReceiveCount`
D. Reduce the message retention period to 1 minute

**Q15.** A company wants HTTPS on its public ALB using a certificate that renews automatically with no manual steps. What should the architect do?
A. Request a public certificate in AWS Certificate Manager and attach it to the ALB's HTTPS listener
B. Generate a self-signed certificate and upload it to IAM
C. Store the certificate in Secrets Manager and rotate it with Lambda
D. Use a KMS key on the ALB listener

**Q16.** A company runs hundreds of Lambda functions and suspects many are configured with far more memory than they use. It wants data-driven recommendations with the least effort. What should it use?
A. AWS Trusted Advisor service limit checks
B. AWS Compute Optimizer
C. AWS Cost Anomaly Detection
D. Amazon CloudWatch Logs Insights queries written per function

**Q17.** Application logs must be searchable in near-real-time on dashboards by the operations team. The company wants a managed ingestion path with no servers or custom code. What should the architect do?
A. Stream logs with Amazon Data Firehose into Amazon OpenSearch Service and use OpenSearch Dashboards
B. Write logs to S3 and query them with Athena once a day
C. Load logs into Amazon RDS for PostgreSQL
D. Use Kinesis Data Streams with custom consumers on EC2 writing to Redshift

**Q18.** A data lake in S3 contains a customer table with PII columns. Marketing analysts using Athena must query the table but must never see the `email` and `phone` columns. What should the architect implement?
A. An S3 bucket policy that denies access to the PII columns
B. AWS Lake Formation column-level permissions on the table for the analysts' role
C. A separate S3 bucket for each column
D. S3 Object Lock on the PII objects

**Q19.** A static website is served through CloudFront from an S3 bucket in us-east-1. The company wants CloudFront to serve content from a replica bucket in us-west-2 automatically if the primary origin returns errors. What should the architect configure?
A. Route 53 failover records for the two buckets
B. Two CloudFront distributions with weighted routing
C. A CloudFront origin group with the primary and replica buckets and failover on 5xx/4xx status codes
D. S3 Transfer Acceleration

**Q20.** An application in Account A must decrypt data that was encrypted with a customer managed KMS key in Account B. What is required? 
A. Only an IAM policy in Account A allowing `kms:Decrypt`
B. Only a key policy statement in Account B allowing Account A
C. A KMS grant created in Account A
D. A key policy in Account B allowing Account A to use the key, and an IAM policy in Account A allowing the principal `kms:Decrypt` on that key ARN

**Q21.** A read-heavy REST API on Amazon API Gateway returns product details that change once an hour. Backend Lambda costs and latency are high. What should the architect do FIRST?
A. Enable API Gateway stage caching with a TTL
B. Increase Lambda memory
C. Move the backend to EC2
D. Switch to a WebSocket API

**Q22.** An insurance claim workflow has several steps: validate, run a fraud check, wait for a human adjuster's approval (which can take days), then pay. Each step must retry on failure and the state must be visible. What should the architect use?
A. A chain of Lambda functions invoking each other synchronously
B. AWS Step Functions Standard workflow with a task token for the human approval step
C. An SQS queue per step with cron jobs on EC2
D. AWS Step Functions Express workflow

**Q23.** A company serves large static media files to users worldwide directly from an S3 bucket. Data transfer costs are high and users far from the Region complain of slow downloads. What is the MOST cost-effective improvement?
A. Enable S3 Transfer Acceleration for downloads
B. Replicate the bucket to every Region
C. Move the files to EFS
D. Serve the files through Amazon CloudFront with an origin access control to the bucket

**Q24.** A company uses AWS Organizations and must keep an audit trail of all API activity in every account, in a central place that member account administrators cannot alter, and must be able to prove log files weren't modified. Which actions meet these requirements? *(Choose two.)*
A. Create an organization trail in CloudTrail that delivers to an S3 bucket in a dedicated log archive account
B. Enable CloudTrail log file integrity validation
C. Rely on the 90-day CloudTrail Event history in each account
D. Enable AWS Config in each account
E. Create a CloudWatch dashboard in each account

**Q25.** A Redshift cluster performs well most of the month, but at month-end hundreds of analysts run reports at once and queries queue for minutes. What should the architect enable?
A. Redshift Spectrum
B. A larger single node type permanently
C. Concurrency scaling
D. Cross-Region snapshot copy

**Q26.** After a ransomware incident at a peer company, a CISO requires that backups cannot be deleted or have their retention shortened by anyone, including the root user, for one year. What should the architect configure?
A. AWS Backup with a backup vault protected by Vault Lock in compliance mode
B. S3 Versioning on the backup bucket
C. An SCP that denies `backup:DeleteRecoveryPoint`
D. RDS automated backups with 35-day retention

**Q27.** A company's public-facing application has been the target of large DDoS attacks. It wants 24/7 access to the AWS Shield Response Team, protection against scaling charges caused by attacks, and advanced attack visibility. What should it subscribe to?
A. AWS Shield Standard
B. AWS Shield Advanced
C. AWS WAF Bot Control only
D. Amazon GuardDuty

**Q28.** A startup wants an email alert when its AWS spend is *forecast* to exceed $2,000 this month, so it can act before the bill arrives. What should it use?
A. AWS Cost Explorer saved report
B. AWS Cost Anomaly Detection
C. AWS Budgets with a forecasted-spend alert
D. Trusted Advisor

**Q29.** When an order is placed, three independent services (inventory, email, analytics) must each process it in parallel, and each must be able to fall behind without affecting the others. What should the architect use?
A. A single SQS queue read by all three services
B. An SNS topic with three SQS queue subscriptions, one per service
C. Synchronous calls from the order service to each service
D. A Kinesis data stream with one shard

**Q30.** A bank processes account transactions. For each account, transactions must be processed exactly once and in order, but different accounts can be processed in parallel. What should the architect use?
A. SQS standard queue with long polling
B. SNS standard topic
C. SQS FIFO queue using the account ID as the message group ID
D. Amazon EventBridge default bus

**Q31.** A company must discover and classify personally identifiable information across thousands of S3 buckets and receive findings when sensitive data is found in a bucket that is publicly accessible. Which service should it use?
A. Amazon Macie
B. Amazon Inspector
C. AWS Security Hub only
D. Amazon Detective

**Q32.** A production Amazon RDS for PostgreSQL instance has run at a steady size 24/7 for two years and will continue for three more. What is the MOST cost-effective purchasing option?
A. Compute Savings Plans
B. Amazon RDS Reserved Instances for the instance class
C. EC2 Spot Instances
D. On-Demand Capacity Reservations

**Q33.** A CloudFront distribution must add security headers to every response and rewrite a few URLs at the edge. The logic is simple, must run at very high request volume and should be as cheap as possible. What should the architect use?
A. Lambda@Edge on origin request
B. An ALB listener rule
C. AWS WAF custom rules
D. CloudFront Functions

**Q34.** A company runs 200 on-premises VMware servers. It needs DR to AWS with an RPO of seconds and an RTO of minutes, using continuous block-level replication with low-cost staging resources until failover. What should the architect use?
A. AWS Backup for on-premises VMware
B. AWS DataSync
C. AWS Elastic Disaster Recovery
D. AWS Application Migration Service

**Q35.** A company wants continuous scanning of its EC2 instances and container images in Amazon ECR for software vulnerabilities and unintended network exposure. Which service should it use?
A. Amazon GuardDuty
B. Amazon Macie
C. AWS Shield
D. Amazon Inspector

**Q36.** A two-person data team must convert daily CSV drops in S3 to Parquet and process only the new files each day. They do not want to manage any clusters. What should the architect recommend?
A. AWS Glue ETL job with job bookmarks enabled, triggered daily
B. Amazon EMR on EC2 with a long-running cluster
C. An EC2 cron job running a Python script
D. Amazon Redshift UNLOAD

**Q37.** A company has two years of steady EC2 and Fargate usage and wants to know how much hourly Savings Plans commitment to buy for the best savings. Which tool provides this recommendation directly?
A. AWS Pricing Calculator
B. AWS Cost Explorer Savings Plans recommendations
C. AWS Budgets
D. AWS Service Quotas

**Q38.** A company has GuardDuty, Inspector, Macie and Config across 60 accounts. The security team wants a single place to aggregate findings and check accounts against the AWS Foundational Security Best Practices standard. What should it enable?
A. Amazon Detective
B. AWS Audit Manager
C. AWS Security Hub (CSPM) with a delegated administrator and cross-Region aggregation
D. CloudWatch cross-account dashboards

**Q39.** An ECS service on Fargate runs behind an ALB. The company needs the service to remain available if an Availability Zone fails. Which actions meet this? *(Choose two.)*
A. Configure the service with subnets in at least two AZs
B. Set the service's desired count high enough that the remaining AZs can carry the load, with the ALB health checks replacing failed tasks
C. Place all tasks in one AZ to reduce latency
D. Use a cluster placement group
E. Use a single task with a larger CPU size

**Q40.** Whenever an item in a DynamoDB table is updated, a downstream service must be notified within seconds with the item's old and new values. What should the architect use?
A. A scheduled Lambda that scans the table every minute
B. DynamoDB Streams with `NEW_AND_OLD_IMAGES`, triggering a Lambda function
C. DynamoDB Accelerator (DAX)
D. CloudTrail data events for DynamoDB

**Q41.** Workloads in private subnets must only reach a short list of approved internet domains. All other outbound traffic must be blocked and logged, with stateful inspection. What should the architect deploy?
A. Security groups with domain names in outbound rules
B. Network ACLs with deny rules
C. AWS Network Firewall with a domain allow-list rule group, routed between the private subnets and the NAT gateway
D. VPC Flow Logs

**Q42.** A company with 400 S3 buckets across 30 accounts wants an organization-wide view to find buckets without lifecycle rules and with large amounts of incomplete multipart upload storage. What should it use?
A. S3 Storage Lens with an organization-level dashboard
B. S3 Inventory on one bucket
C. AWS Config rules
D. AWS Cost Anomaly Detection

**Q43.** A company must migrate a 4 TB on-premises Oracle database to Amazon Aurora PostgreSQL. Downtime during cutover must be minutes. Which approach should the architect use? *(Choose two.)*
A. Convert the schema and code with DMS Schema Conversion (or AWS SCT)
B. Use AWS DMS with full load plus ongoing change data capture (CDC) until cutover
C. Take an export file and import it over a weekend
D. Use AWS DataSync to copy the database files
E. Use Snowball Edge to ship the data files

**Q44.** Business users' QuickSight dashboards query Athena directly. Dashboards are slow to load and Athena costs are rising as more users open them. What should the architect do?
A. Import the dataset into QuickSight SPICE with a scheduled refresh
B. Increase Athena's query timeout
C. Replace QuickSight with CloudWatch dashboards
D. Move the data to S3 Glacier Instant Retrieval

**Q45.** A media company must ensure its video content served via CloudFront cannot be viewed from two specific countries due to licensing. What should the architect configure with the LEAST effort?
A. NACLs on the origin VPC
B. CloudFront geographic restrictions (deny list) for those countries
C. Route 53 geolocation records
D. Security groups on the origin ALB

**Q46.** An S3-triggered Lambda function (asynchronous invocation) occasionally fails after all retries. The team needs failed events captured, with the error details, for later reprocessing. What should the architect configure?
A. Increase the function timeout to 15 minutes
B. Reserved concurrency of 1
C. Provisioned concurrency
D. An on-failure destination (SQS queue or SNS topic) for the function's asynchronous invocations

**Q47.** A company runs a large number of batch-processing ECS tasks that can be safely interrupted and restarted. It wants to minimise compute cost without managing EC2 instances. What should it use?
A. Fargate with a Compute Savings Plan only
B. Fargate Spot capacity provider
C. ECS on EC2 Dedicated Hosts
D. AWS Lambda with provisioned concurrency

**Q48.** A REST API on API Gateway must accept requests only from users who have signed in to the company's Amazon Cognito user pool. What should the architect configure with the LEAST custom code?
A. A Lambda authorizer that validates JWTs manually
B. IAM authorization with long-lived access keys distributed to users
C. A Cognito user pool authorizer on the API methods
D. API keys with usage plans

**Q49.** A company deploys its application in us-east-1 and ap-south-1. Users should be routed to whichever Region gives them the lowest network latency, and away from a Region if it is unhealthy. What should the architect configure?
A. Route 53 latency-based routing records with health checks
B. Route 53 simple routing
C. Route 53 geolocation routing without health checks
D. A single ALB with cross-zone load balancing

**Q50.** A single EC2 instance runs a legacy licensed application that can't be clustered. If the underlying host hardware fails, the instance should come back automatically with the same private IP, instance ID and EBS volumes. What should the architect configure?
A. An Auto Scaling group with min/max of 1
B. A CloudWatch alarm on `StatusCheckFailed_System` with the EC2 recover action (or the default automatic recovery)
C. A Route 53 health check
D. An EBS snapshot every hour

**Q51.** Security wants any S3 bucket that becomes publicly readable to be detected and automatically corrected, with a compliance history kept. What should the architect use?
A. CloudTrail Insights
B. Amazon Inspector
C. Trusted Advisor
D. An AWS Config managed rule for S3 public read with automatic remediation via an SSM Automation runbook

**Q52.** An application in us-east-1 reads database credentials from AWS Secrets Manager. In a Regional outage, the DR copy of the application in us-west-2 must read the same secret, kept in sync. What should the architect do with the LEAST effort?
A. Store the secret in a DynamoDB global table
B. Recreate the secret manually in us-west-2 during failover
C. Enable Secrets Manager secret replication to us-west-2
D. Put the secret in a Lambda environment variable in both Regions

**Q53.** A Kinesis data stream in provisioned mode returns `ProvisionedThroughputExceededException`. Metrics show one shard is far busier than the rest because most records share the same partition key. What should the architect do?
A. Increase the stream's retention period
B. Switch consumers to enhanced fan-out
C. Use Amazon Data Firehose as a producer
D. Choose a higher-cardinality partition key so records spread evenly across shards

**Q54.** Objects written to an S3 bucket in eu-west-1 must be replicated to eu-central-1, and the business needs a predictable replication time with metrics and an SLA (most objects within 15 minutes). What should the architect configure?
A. S3 Cross-Region Replication with S3 Replication Time Control enabled
B. S3 Same-Region Replication
C. A lifecycle rule
D. AWS DataSync scheduled hourly

**Q55.** A regulated company requires that encryption keys are kept in a single-tenant hardware security module that only the company's own administrators control, and that AWS cannot access. What should the architect use?
A. AWS KMS with an AWS managed key
B. AWS KMS with a customer managed key
C. AWS CloudHSM
D. SSE-S3

**Q56.** A data warehouse is used heavily for two days at month-end and barely at all otherwise. The team wants to stop paying for idle capacity and avoid managing clusters. What should the architect recommend?
A. A provisioned Redshift cluster with 3-year Reserved Nodes
B. Amazon Redshift Serverless
C. Amazon RDS for MySQL
D. Amazon EMR on EC2 running 24/7

**Q57.** A company runs a self-managed Apache Kafka cluster on premises that dozens of applications use. It wants to move to AWS without changing application code and with the least ongoing broker management. What should the architect recommend?
A. Amazon Kinesis Data Streams
B. Amazon SQS
C. Amazon MSK Serverless
D. Amazon EventBridge

**Q58.** An internal API on API Gateway must be reachable only from within the company's VPC, never from the internet. What should the architect configure?
A. A private REST API with an interface VPC endpoint for `execute-api` and a resource policy allowing only that endpoint
B. A regional API with an API key
C. An edge-optimised API with AWS WAF
D. A public HTTP API with a security group

**Q59.** An event-driven system on Amazon EventBridge needs the ability to re-process the last seven days of order events after a bug in a consumer is fixed. What should the architect configure?
A. A dead-letter queue on each rule target
B. An EventBridge archive on the event bus, then replay the archived events
C. CloudWatch Logs retention of seven days
D. An SQS FIFO queue

**Q60.** Customers must be able to download their invoices from a private S3 bucket for 10 minutes after clicking a link in the web app, without making the bucket public or giving them AWS credentials. What should the architect do?
A. Make the invoice prefix public-read
B. Generate S3 presigned URLs with a 10-minute expiry from the application's role
C. Create an IAM user per customer
D. Enable S3 Transfer Acceleration

**Q61.** A new API needs simple JWT authorisation and proxying to Lambda, with the lowest cost per request. Advanced features like request validation, usage plans and API keys are not needed. What should the architect choose?
A. API Gateway REST API
B. Application Load Balancer with Lambda targets and Cognito
C. API Gateway HTTP API
D. AWS AppSync

**Q62.** A CloudFront distribution has a low cache hit ratio. Investigation shows the cache key includes all query strings, all cookies and the `User-Agent` header, though the origin returns the same content regardless. What should the architect do?
A. Increase the origin's instance size
B. Create a cache policy that includes only the query strings/headers/cookies the origin actually varies on, and forward any others through an origin request policy only if needed
C. Disable compression
D. Add more edge locations

**Q63.** A global e-commerce platform must keep serving reads and writes from either of two Regions with near-zero RTO and RPO if one Region fails. The data model is key-value. What should the architect build?
A. Warm standby with an Aurora cross-Region read replica
B. Backup and restore with AWS Backup
C. Pilot light with RDS snapshots copied hourly
D. Active-active multi-site in both Regions with DynamoDB global tables and Route 53 latency routing with health checks

**Q64.** Auditors require a record of every `GetObject` and `PutObject` call on one S3 bucket containing financial reports, including the caller's identity. What should the architect enable?
A. CloudTrail data events for that bucket
B. CloudTrail management events only
C. VPC Flow Logs
D. AWS Config

**Q65.** A company runs many x86 Lambda functions written in Python that are CPU-bound. It wants to lower Lambda cost with no code changes. What should the architect try FIRST?
A. Move the functions to EC2 On-Demand
B. Switch the functions to the arm64 (Graviton) architecture and test
C. Add provisioned concurrency to every function
D. Increase the memory of every function to the maximum

<details>
<summary><strong>Answer key — open only after all 65 are answered</strong></summary>

Each line gives the answer, the domain and source day, and why it wins.

1. **D** · *Cost-optimized · Day 12* — Data Exports/CUR is the most granular billing data, landed in S3 for Athena. Cost Explorer is visual analysis, not raw hourly line items.
2. **B** · *Secure · Day 10* — A customer managed key gives you the key policy, CloudTrail visibility of each use, and enable/disable control. SSE-S3 keys are invisible to you; SSE-C and client-side shift key management to you.
3. **A** · *Resilient · Day 8* — SQS buffers orders durably; scaling on backlog per instance matches capacity to demand. SNS over HTTP isn't a durable buffer for slow consumers.
4. **C** · *High-performing · Day 9* — Global Accelerator gives static anycast IPs, supports UDP, and fails over between endpoint groups in seconds. CloudFront is HTTP(S); DNS-based routing is subject to client caching.
5. **A** · *Secure · Day 10* — WAF managed rules block SQLi; rate-based rules throttle abusive IPs. Shield Standard is L3/L4; GuardDuty detects, it doesn't block; manual NACLs don't scale.
6. **C** · *Cost-optimized · Day 8* — Event-driven Lambda bills only per invocation, nothing when idle. Polling fleets and standing clusters pay for idle time.
7. **D** · *Resilient · Day 11* — An RTO measured in hours and an RPO of one hour fit backup and restore, the cheapest strategy. Pilot light and above pay for standing resources the requirements don't need.
8. **A** · *High-performing · Day 8* — Enhanced fan-out gives each consumer its own dedicated read throughput per shard with push delivery. Firehose delivers to destinations, not to five custom apps.
9. **B** · *Secure · Day 10* — GuardDuty analyses VPC Flow Logs, DNS and CloudTrail for threats like C2 traffic and crypto mining, agentless, org-wide via delegated admin. Inspector finds vulnerabilities, not active threats.
10. **C** · *Cost-optimized · Day 12* — Workgroups enforce data-scanned limits per query and per workgroup. Budgets only alert after the fact.
11. **B** · *Resilient · Day 9* — Failover routing uses the primary's health check to switch to the secondary. Weighted routing is for splitting traffic; failover routing is the purpose-built active-passive pattern.
12. **D** · *Secure · Day 10* — User pools authenticate app users (including social IdPs); identity pools exchange tokens for temporary, scoped AWS credentials. Identity Center is for the workforce.
13. **A** · *High-performing · Day 9* — Fargate runs tasks without managing any instances. Every other option leaves servers to patch and scale.
14. **C** · *Resilient · Day 8* — A DLQ isolates poison messages after `maxReceiveCount` failures. A huge visibility timeout just delays retries; FIFO doesn't fix bad messages.
15. **A** · *Secure · Day 9/10* — ACM public certificates renew automatically when attached to integrated services like ALB. Self-signed certs aren't trusted; KMS isn't used for TLS certificates.
16. **B** · *Cost-optimized · Day 12* — Compute Optimizer gives memory recommendations for Lambda from utilisation history. Per-function log queries are manual effort.
17. **A** · *High-performing · Day 12* — Firehose to OpenSearch is a fully managed, near-real-time ingest path with dashboards. Daily Athena isn't near-real-time; custom consumers add ops.
18. **B** · *Secure · Day 12* — Lake Formation enforces column-level permissions for Athena. Bucket policies operate on objects, not columns.
19. **C** · *Resilient · Day 9* — CloudFront origin groups fail over to a secondary origin on configured status codes. Route 53 can't target buckets behind one distribution this way.
20. **D** · *Secure · Day 10* — Cross-account KMS needs both sides: the key policy in B must allow Account A, and A's IAM policy must allow the principal to use the key.
21. **A** · *High-performing · Day 8* — Stage caching serves repeated reads from the cache for the TTL, cutting backend calls and latency. More Lambda memory still runs every request.
22. **B** · *Resilient · Day 8* — Standard workflows run up to a year, have built-in retries and visual state; task tokens pause for human approval. Express workflows max out at 5 minutes.
23. **D** · *Cost-optimized · Day 9* — CloudFront caches at the edge, speeds downloads, and data transfer from CloudFront is typically cheaper than direct S3 egress. Transfer Acceleration is for uploads and adds cost.
24. **A, B** · *Secure · Day 11* — An organization trail into a log archive account centralises logs out of member admins' reach; integrity validation proves files weren't changed. Event history is per account and only 90 days.
25. **C** · *High-performing · Day 12* — Concurrency scaling adds transient capacity for bursts of concurrent queries. Spectrum queries S3; a permanently larger cluster pays for month-end all month.
26. **A** · *Resilient · Day 11* — Vault Lock compliance mode makes recovery points immutable, even for root, once the cooling-off period ends. SCPs don't apply to the management account and can be changed.
27. **B** · *Secure · Day 10* — Shield Advanced adds the Shield Response Team, cost protection for attack-driven scaling, and advanced visibility. Standard is automatic, basic L3/L4 protection.
28. **C** · *Cost-optimized · Day 12* — Budgets supports alerts on forecasted spend against a threshold you set. Anomaly Detection flags unusual spikes, not a fixed budget.
29. **B** · *High-performing · Day 8* — SNS-to-SQS fan-out gives each service its own queue, so each consumes at its own pace. A shared queue delivers each message to only one consumer.
30. **C** · *Resilient · Day 8* — FIFO preserves order and deduplicates per message group; separate groups (accounts) process in parallel. Standard queues are at-least-once and unordered.
31. **A** · *Secure · Day 10* — Macie discovers and classifies sensitive data in S3 and flags public buckets. Inspector scans compute for vulnerabilities.
32. **B** · *Cost-optimized · Day 2/12* — RDS Reserved Instances discount a steady instance. Compute Savings Plans don't cover RDS; Spot isn't for databases; Capacity Reservations give no discount.
33. **D** · *High-performing · Day 9* — CloudFront Functions run lightweight header/URL logic at viewer request/response at very high scale and low cost. Lambda@Edge is for heavier logic or network calls.
34. **C** · *Resilient · Day 11* — Elastic Disaster Recovery does continuous block replication to a low-cost staging area with launch in minutes. MGN is for migration, not ongoing DR.
35. **D** · *Secure · Day 10* — Inspector continuously scans EC2, ECR images and Lambda for CVEs and network reachability. GuardDuty detects threats; Macie finds sensitive data.
36. **A** · *High-performing · Day 12* — Glue ETL is serverless Spark; job bookmarks track processed files so only new data is handled. EMR and EC2 add cluster/server management.
37. **B** · *Cost-optimized · Day 12* — Cost Explorer generates Savings Plans purchase recommendations from your usage history. The Pricing Calculator estimates new designs.
38. **C** · *Secure · Day 10* — Security Hub aggregates findings from GuardDuty, Inspector, Macie and Config and runs standards checks like FSBP across accounts and Regions. Detective investigates; Audit Manager collects audit evidence.
39. **A, B** · *Resilient · Day 9* — Spread tasks across AZs and size the service so surviving AZs carry the load; the service scheduler and ALB health checks replace failed tasks. One AZ or one task is a single point of failure.
40. **B** · *High-performing · Day 6/8* — Streams capture item-level changes with old and new images, and Lambda reacts within seconds. Scans are slow and expensive; DAX is a cache.
41. **C** · *Secure · Day 4/10* — Network Firewall does stateful inspection and domain (SNI/Host) allow-listing with logging. SGs and NACLs filter by IP/port, not domain.
42. **A** · *Cost-optimized · Day 5/12* — Storage Lens gives org-wide usage and activity metrics, including incomplete multipart bytes and lifecycle coverage. Inventory is per bucket.
43. **A, B** · *Resilient · Day 11* — Heterogeneous migration needs schema conversion, then DMS full load + CDC keeps the target in sync until a short cutover. Exports and shipping files mean long downtime.
44. **A** · *High-performing · Day 12* — SPICE serves dashboards from memory and refreshes on a schedule, so Athena runs once per refresh, not per viewer.
45. **B** · *Secure · Day 9* — CloudFront geo restriction blocks countries at the edge with one setting. Route 53 geolocation only changes DNS answers and is easy to bypass.
46. **D** · *Resilient · Day 8* — On-failure destinations capture the event plus error details after retries are exhausted. Timeouts and concurrency settings don't capture failures.
47. **B** · *Cost-optimized · Day 9* — Fargate Spot gives a large discount for interruption-tolerant tasks with no instances to manage. Savings Plans discount less than Spot for this pattern.
48. **C** · *Secure · Day 10* — A Cognito user pool authorizer validates tokens natively. A Lambda authorizer works but is custom code; API keys aren't authentication.
49. **A** · *High-performing · Day 9* — Latency routing sends users to the lowest-latency Region; health checks remove an unhealthy Region from answers.
50. **B** · *Resilient · Day 11* — EC2 recovery keeps the same instance ID, private IP, EBS volumes and metadata on new hardware. An ASG would launch a *new* instance with a new ID.
51. **D** · *Secure · Day 11* — Config evaluates and records compliance history and can auto-remediate with SSM Automation. Trusted Advisor flags but doesn't remediate or keep history.
52. **C** · *Resilient · Day 10/11* — Secrets Manager replicates secrets (and rotations) to other Regions natively. Manual recreation fails the "kept in sync" requirement.
53. **D** · *High-performing · Day 8* — A hot partition key concentrates writes on one shard; a high-cardinality key spreads them. Fan-out helps reads, not writes.
54. **A** · *Resilient · Day 5/11* — Replication Time Control adds the 15-minute SLA and replication metrics. SRR stays in-Region; lifecycle doesn't replicate.
55. **C** · *Secure · Day 10* — CloudHSM gives dedicated, single-tenant HSMs under your exclusive control. KMS keys are managed in AWS's multi-tenant HSM fleet.
56. **B** · *Cost-optimized · Day 12* — Redshift Serverless scales with demand and charges for compute used, with no cluster to manage. Reserved Nodes pay for idle capacity all month.
57. **C** · *High-performing · Day 12* — MSK is Kafka API compatible, so apps don't change; Serverless removes broker sizing. Kinesis and SQS would require code changes.
58. **A** · *Secure · Day 8/10* — Private APIs are reachable only through an interface endpoint, and the resource policy restricts to it. API keys aren't access control.
59. **B** · *Resilient · Day 8* — EventBridge archives store events and replay re-sends them to the bus. DLQs only hold failed deliveries.
60. **B** · *Secure · Day 5* — Presigned URLs grant time-limited access to one object using the signer's permissions. Public prefixes and per-customer IAM users are insecure or unmanageable.
61. **C** · *Cost-optimized · Day 8* — HTTP APIs are cheaper per request and support JWT authorizers and Lambda proxying natively. REST APIs add features not needed here.
62. **B** · *High-performing · Day 9* — A lean cache key raises the hit ratio; origin request policies can still forward values the origin needs without fragmenting the cache.
63. **D** · *Resilient · Day 6/11* — Active-active with DynamoDB global tables gives multi-Region writes and near-zero RTO/RPO. Warm standby and below take time to fail over.
64. **A** · *Secure · Day 11* — Object-level S3 API calls are data events, which aren't logged by default. Management events cover control-plane calls only.
65. **B** · *Cost-optimized · Day 12* — Graviton (arm64) Lambda typically gives better price-performance; pure Python usually runs unchanged (test native dependencies). Provisioned concurrency and maximum memory add cost.

</details>

---

## 11. Scoring guide

### Step 1 — raw score

Count correct answers (a *Choose two* question scores only if both picks are right).

| Raw score | % | Read it as | Next move |
| --- | --- | --- | --- |
| 56–65 | 86%+ | Exam-ready on this material | Light Day 13; focus on any repeat misses |
| 52–55 | 80–85% | Meets the go/no-go bar for this exam | Need one more 80%+ (Exam 3 tomorrow) to book |
| 46–51 | 70–79% | Borderline | Day 13 at full depth on your weakest domain; Exam 3 decides |
| < 46 | < 70% | Below the README bar | Per the README, **push the exam back a week**; re-read the decision tables of your two weakest domains |

Scaled scoring (720/1000) isn't a straight percentage. 80% on practice exams is the safety margin, not a conversion.

### Step 2 — score by domain

| Domain (exam weight) | Questions | Count | Your score | 80% bar |
| --- | --- | --- | --- | --- |
| Secure (30%) | 2, 5, 9, 12, 15, 18, 20, 24, 27, 31, 35, 38, 41, 45, 48, 51, 55, 58, 60, 64 | 20 | __ / 20 | 16 |
| Resilient (26%) | 3, 7, 11, 14, 19, 22, 26, 30, 34, 39, 43, 46, 50, 52, 54, 59, 63 | 17 | __ / 17 | 14 |
| High-performing (24%) | 4, 8, 13, 17, 21, 25, 29, 33, 36, 40, 44, 49, 53, 57, 62 | 15 | __ / 15 | 12 |
| Cost-optimized (20%) | 1, 6, 10, 16, 23, 28, 32, 37, 42, 47, 56, 61, 65 | 13 | __ / 13 | 11 |

### Step 3 — map weak domains back to the lessons

| Weak domain | Re-read first | Then |
| --- | --- | --- |
| Secure | Day 10 §§ on KMS (key policies, cross-account), WAF/Shield, the GuardDuty/Inspector/Macie/Security Hub split, Cognito | Day 11 CloudTrail (org trail, data events) and Config; Day 12 §3 Lake Formation |
| Resilient | Day 11 §1–3 (RPO/RTO → strategy, DRS, Backup Vault Lock), Day 8 (SQS DLQ, FIFO, Step Functions, EventBridge archive) | Day 9 Route 53 routing policies and CloudFront origin groups |
| High-performing | Day 8 (Kinesis shards, fan-out, SNS→SQS), Day 9 (Global Accelerator vs CloudFront, CloudFront Functions, cache keys) | Day 12 §2–4 (Firehose, Athena, Redshift features, Glue, MSK) |
| Cost-optimized | Day 12 §5 (cost tools table, commitment discounts) | Day 2 §3 purchasing options; Day 5 storage classes; Day 8 HTTP vs REST API |

### Step 4 — log every miss, then compare with Exam 1

Add one line per miss *and* lucky guess to `miss-log.md`, for example:

```markdown
| 12 | Athena vs Redshift | Redshift | Athena | "Ad hoc, few times a week, S3, serverless" = Athena; Redshift is for heavy repeated BI |
```

Also add a score line at the top: `Exam 2 (Day 12): __/65 — S __/20 · R __/17 · P __/15 · C __/13`.

Then compare with Exam 1: a domain under its 80% bar **on both exams** is your Day 13 priority, and any discriminator that appears twice in the log (e.g. Streams vs Firehose, Budgets vs Anomaly Detection) is a pattern Day 13 will drill.

---

## 12. Flashcards (cover the right column)

| Prompt | Answer |
| --- | --- |
| Load streams into S3/Redshift/OpenSearch, no code | Amazon Data Firehose |
| Replay + multiple consumers on a stream | Kinesis Data Streams |
| Dedicated read throughput per consumer on Kinesis | Enhanced fan-out |
| Hot shard on Kinesis | Higher-cardinality partition key |
| "Kafka" in the question | Amazon MSK (Serverless for least ops) |
| Real-time windowed analytics on a stream | Managed Service for Apache Flink |
| Serverless SQL on S3, pay per scan | Athena |
| Three levers to cut Athena cost | Columnar format, partitioning, compression |
| Cap Athena scan per team | Workgroups with data usage limits |
| SQL across DynamoDB/RDS/on-prem in place | Athena federated query |
| Redshift + S3 data without loading | Redshift Spectrum |
| Month-end query bursts on Redshift | Concurrency scaling |
| Intermittent warehouse, no cluster | Redshift Serverless |
| Live Redshift data to another account | Data sharing |
| Central schema store for the lake | Glue Data Catalog |
| Discover schema/partitions automatically | Glue crawler |
| Glue: process only new files | Job bookmarks |
| No-code data prep | Glue DataBrew |
| Column/row permissions on the lake | Lake Formation |
| EMR node type safe for Spot | Task nodes |
| Full-text / relevance search | OpenSearch Service |
| Fast BI dashboards in memory | QuickSight SPICE |
| Estimate a design's cost | Pricing Calculator |
| Alert on forecast spend vs threshold | AWS Budgets |
| Auto-apply SCP / stop EC2 when over budget | Budget actions |
| ML-detected spend spike | Cost Anomaly Detection |
| Most granular billing data | Data Exports (CUR 2.0) → S3 → Athena |
| Tag not showing in Cost Explorer | Activate the cost allocation tag in the payer account |
| Rightsize EC2/EBS/Lambda/ASG | Compute Optimizer |
| Full Trusted Advisor checks need | Business support or higher |
| Org-wide S3 usage & lifecycle gaps | S3 Storage Lens |
| SP that covers Fargate and Lambda | Compute Savings Plans |
| Steady RDS instance discount | RDS Reserved Instances |
| Capacity guarantee, no discount | On-Demand Capacity Reservation |

## 13. Before you close today

- [ ] Sections 1–7 read (focus: pipeline diagram, Streams vs Firehose vs MSK, Athena vs Redshift vs Spectrum, Glue vs EMR, Lake Formation, the cost tools table)
- [ ] Lab done and torn down (Athena tables, database and workgroup; Glue crawler + role if used; S3 bucket; budget optional)
- [ ] Warm-up: 10 lesson questions
- [ ] Practice Exam 2 taken in one 130-minute sitting, **then** scored
- [ ] Score line `Exam 2 (Day 12): …` and every miss + lucky guess logged in `miss-log.md` — Day 13 is built from this file
- [ ] Go/no-go check: under 70% → push the exam back a week (README rule); 80%+ → one more 80%+ tomorrow books 2026-11-13

**Tomorrow (Day 13):** gap-closing lesson aimed at your miss-log patterns, Practice Exam 3, and the go/no-go decision for 2026-11-13.
