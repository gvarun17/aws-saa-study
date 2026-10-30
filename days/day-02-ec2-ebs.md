# Day 2 — EC2, EBS, Instance Store, AMIs & Purchasing Options

**Domain weight:** *Design High-Performing Architectures* (24%) and *Design Cost-Optimized Architectures* (20%), with a solid slice of *Resilient* (26%) — snapshots, AMIs and recovery.
**Time plan:** Learn 90 min · Lab 30 min · Drill 45 min · Review 30 min.

By the end of today you should be able to:

- Read an instance type name (`c7g.2xlarge`) and match an instance family to a workload.
- Pick the cheapest purchasing option that still meets a scenario's constraints (On-Demand, Savings Plans, Reserved Instances, Spot, Dedicated, Capacity Reservations).
- Choose the right placement group for latency, availability or big-data scenarios.
- Choose between EBS volume types, instance store and (preview) EFS, and explain what survives stop/terminate.
- Use snapshots and AMIs to back up, encrypt, copy across Regions/accounts and recover instances.
- Spot the classic traps: EBS is AZ-locked, instance store is ephemeral, shared encrypted AMIs need the KMS key shared too.

---

## 1. How EC2 actually works

An EC2 instance is a virtual machine running on the **Nitro System**: a lightweight hypervisor plus dedicated Nitro cards that offload networking, EBS storage I/O and security. That offload is *why* modern instances get near bare-metal performance, why EBS is "network storage" that still feels local, and why features like EBS encryption, ENA networking and many newer instance types require Nitro.

```
┌──────────────────────── Physical host (in one AZ) ────────────────────────┐
│  ┌──────────────┐  ┌──────────────┐                                       │
│  │ Instance A   │  │ Instance B   │   ◄── Nitro hypervisor (thin)         │
│  │ vCPU / RAM   │  │ vCPU / RAM   │                                       │
│  └──────┬───────┘  └──────┬───────┘                                       │
│         │ local NVMe       │                                              │
│   ┌─────▼──────┐   ┌───────▼────────┐   ┌──────────────┐                  │
│   │ Instance   │   │ Nitro card:    │   │ Nitro card:  │                  │
│   │ store      │   │ VPC networking │   │ EBS (NVMe    │──── network ───► EBS volumes
│   │ (ephemeral)│   │ (ENA, SGs)     │   │ over fabric) │                  (same AZ)
│   └────────────┘   └────────────────┘   └──────────────┘                  │
└───────────────────────────────────────────────────────────────────────────┘
```

Key consequences:

- **Instance store** is physically on the host → blazing fast, but gone when the instance moves to another host (stop/start, hibernate, terminate, host failure).
- **EBS** is reached over the network → survives the instance, but is pinned to **one AZ**.
- An instance lives in **one subnet → one AZ**. High availability always means *several* instances in *several* AZs (Auto Scaling, Day 3).

### Instance lifecycle

```
 pending ──► running ──► stopping ──► stopped ──► (start) ──► pending …
                │  ▲                      │
          reboot│  │                      └── billing for compute stops; EBS still billed
                ▼  │
             (same host, instance store kept)
                │
                └──► shutting-down ──► terminated (EBS root deleted by default)
```

| Action | Same host? | Instance store data | EBS data | Public IPv4 (non-Elastic) | Private IP |
| --- | --- | --- | --- | --- | --- |
| Reboot | Yes | Kept | Kept | Kept | Kept |
| Stop / start | Usually **no** | **Lost** | Kept | **Changes** | Kept |
| Hibernate / resume | Usually no | **Lost** | Kept (RAM saved to root EBS) | Changes | Kept |
| Terminate | — | Lost | Root deleted by default; extra volumes kept by default | Released | Released |

> Trap: "The public IP changes every time the instance is stopped and started" → attach an **Elastic IP** (or better, put it behind a load balancer / use DNS). Note that AWS now charges for *all* public IPv4 addresses, in use or not — check current pricing.

**`DeleteOnTermination`**: `true` by default for the root volume, `false` for additional volumes attached at launch. If a question says "data on the root volume must be retained after termination," set the root volume's `DeleteOnTermination` to `false`.

**Hibernate**: saves RAM to the (encrypted) EBS root volume and restores it on start — the answer for "long app warm-up / in-memory caches must be preserved across stops." Requires an encrypted EBS root large enough for RAM, supported instance families and is time-limited; check current docs for RAM and duration limits.

**User data** runs once at first boot (as root) — bootstrap scripts. **Instance metadata** (`169.254.169.254`, use IMDSv2) exposes instance ID, role credentials, etc. Both came up on Day 1.

---

## 2. Instance types and families

Reading a name: `c7gn.2xlarge`

```
c   7   g   n   .  2xlarge
│   │   │   │       └── size (vCPU/RAM scale roughly ×2 each step)
│   │   │   └── extra capability (n = network-optimized, d = local NVMe instance store, e = extra RAM/storage, z = high frequency)
│   │   └── processor (g = AWS Graviton/Arm, a = AMD, i = Intel)
│   └── generation (higher = newer, usually better price-performance)
└── family (c = compute)
```

| Family | Letters | Optimised for | Typical exam workloads |
| --- | --- | --- | --- |
| General purpose | **M**, **T** (burstable), Mac | Balanced CPU/RAM | Web/app servers, small DBs, dev/test |
| Compute optimized | **C** | High CPU per GiB RAM | Batch, media transcoding, HPC, gaming servers, scientific modelling, ML inference on CPU |
| Memory optimized | **R**, **X**, **z**, high-memory **u** | High RAM per vCPU | In-memory DBs/caches, SAP HANA, real-time big-data analytics |
| Storage optimized | **I** (NVMe SSD), **D** (dense HDD), **Im/Is** | Huge local IOPS/throughput | NoSQL (Cassandra), OLTP needing local NVMe, data warehousing, distributed file systems, Kafka |
| Accelerated computing | **P** (training), **G** (graphics/inference), **Inf** (Inferentia), **Trn** (Trainium), **F** (FPGA) | GPUs / custom silicon | ML training & inference, video rendering, genomics |
| HPC | **Hpc** | Tightly coupled HPC at scale | CFD, weather simulation |

Mnemonic: **C**ompute, **R**AM, **I**/O, **G**raphics, **P**arallel training, **T**iny/bursty, **M**iddle-of-the-road.

### Burstable (T family)

T instances earn **CPU credits** while below their baseline and spend them when bursting.

- **Standard mode:** when credits run out, CPU is throttled to baseline.
- **Unlimited mode:** can burst indefinitely; you pay for surplus credits.
- Exam use: "spiky, mostly idle" workloads (dev boxes, low-traffic sites). If a question describes a T instance that is *constantly* at 100% and slow → it's out of credits; move to M/C or enable unlimited.

### Graviton

AWS Graviton (Arm) instances (`g` suffix) typically offer better price-performance than comparable x86 — AWS markets up to ~40%; check current claims. Exam cue: "improve price-performance with minimal changes" for interpreted/managed runtimes (Java, Python, Node, containers built multi-arch) → Graviton.

### Networking add-ons you should recognise

| Feature | What it is | Exam cue |
| --- | --- | --- |
| **ENA** (Elastic Network Adapter) | Enhanced networking via SR-IOV; up to very high bandwidth on large instances | "Higher packets per second, lower latency, no extra cost" |
| **EFA** (Elastic Fabric Adapter) | ENA + OS-bypass for MPI/NCCL | Tightly coupled HPC / distributed ML training |
| **ENI** (Elastic Network Interface) | Virtual NIC; can be moved between instances in the same AZ | Failover by moving an ENI (and its IPs/MAC) to a standby instance; management networks |
| **Elastic IP** | Static public IPv4 | Fixed public IP that survives stop/start; remap to standby on failure |

---

## 3. Purchasing options — the cost-optimization core

| Option | Commitment | Discount vs On-Demand* | Interruptible? | Best for |
| --- | --- | --- | --- | --- |
| **On-Demand** | None, per-second (Linux) | 0% | No | Short, unpredictable, can't be interrupted; new workloads before you know the baseline |
| **Compute Savings Plan** | $/hour for 1 or 3 yrs | Up to ~66% | No | Steady spend that may change family, size, Region, OS, tenancy — **also covers Fargate and Lambda** |
| **EC2 Instance Savings Plan** | $/hour for 1 or 3 yrs, locked to a family in a Region | Up to ~72% | No | Steady use of one family in one Region; size/OS/AZ can change |
| **Standard Reserved Instance** | 1 or 3 yrs, specific instance attributes | Up to ~72% | No | Steady-state; can be **sold** on the RI Marketplace |
| **Convertible RI** | 1 or 3 yrs, can exchange for different attributes | Up to ~66% | No | Steady-state but expect to change instance family |
| **Spot** | None | Up to ~90% | **Yes — 2-minute warning** | Fault-tolerant, stateless, flexible: batch, CI, rendering, big data, containers, ASG burst capacity |
| **Dedicated Instance** | None (or RI) | Premium | No | Hardware not shared with other AWS accounts (compliance) |
| **Dedicated Host** | On-Demand or reservation | Premium | No | **BYOL** licensing per socket/core/VM (Windows Server, SQL Server, Oracle), host affinity, visibility of physical cores |
| **On-Demand Capacity Reservation** | None (cancel any time); pay whether used or not | 0% (combine with SPs/regional RIs for discount) | No | **Guaranteed capacity** in a specific AZ for a known event/DR |
| **Capacity Blocks for ML** | Reserve GPU instances for a future date range | Varies | No | "Need GPU instances for a 2-week training run next month" |

\* Discount percentages are AWS's "up to" marketing figures and change; check current docs. On the exam, the **ranking** matters, not the exact numbers: Spot > (EC2 Instance SP ≈ Standard RI) > (Compute SP ≈ Convertible RI) > On-Demand.

Payment options for RIs/SPs: **All Upfront** (biggest discount) > **Partial Upfront** > **No Upfront**. 3-year > 1-year.

### Savings Plans vs Reserved Instances

Savings Plans are the modern default: you commit to **$/hour of spend**, and AWS applies the discount automatically to matching usage. RIs commit to **instance attributes**.

```
Flexibility ▲
            │  Compute Savings Plan   (any family, size, Region, OS, tenancy; EC2 + Fargate + Lambda)
            │  Convertible RI         (exchange for other attributes)
            │  EC2 Instance SP        (any size/OS/AZ within one family + Region)
            │  Standard RI            (fixed; regional RIs flex size within family on Linux)
            └──────────────────────────────────────────────► Discount
```

**Zonal vs regional RIs** — a frequent trap:

| | Regional RI | Zonal RI |
| --- | --- | --- |
| Discount applies | Any AZ in the Region | That AZ only |
| **Capacity reservation** | **No** | **Yes** |
| Size flexibility | Yes (Linux, default tenancy) | No |

Savings Plans **never reserve capacity**. If a question needs both a discount *and* guaranteed capacity → **On-Demand Capacity Reservation + Savings Plan** (or a zonal RI).

### Spot in depth

- You pay the current Spot price (no bidding wars any more; a max price is optional and defaults to On-Demand price).
- AWS can reclaim with a **2-minute interruption notice** (instance metadata + EventBridge). An earlier **rebalance recommendation** signal may also arrive.
- Interruption behaviour: **terminate** (default), **stop** or **hibernate** (for persistent requests).
- **Spot blocks** (fixed-duration Spot) are retired — any answer relying on them is wrong.
- Design for it: stateless workers, checkpointing to S3/EFS, SQS-driven jobs, mixed-instance Auto Scaling groups.

**Allocation strategies** (EC2 Fleet / Spot Fleet / Auto Scaling groups):

| Strategy | Behaviour | When |
| --- | --- | --- |
| **price-capacity-optimized** | Pools with most capacity, then lowest price | **AWS's recommended default** for most Spot workloads |
| **capacity-optimized** | Deepest pools → fewest interruptions | Interruption-sensitive (long jobs) |
| **lowest-price** | Cheapest pools | Short jobs; higher interruption risk |
| **diversified** | Spread across all pools | Long-running, availability-focused fleets |

Pattern you'll see constantly — an Auto Scaling group with a **mixed instances policy**: On-Demand base capacity (covered by a Savings Plan) for the steady baseline + Spot for the variable portion, across several instance types and AZs.

```json
"MixedInstancesPolicy": {
  "InstancesDistribution": {
    "OnDemandBaseCapacity": 2,
    "OnDemandPercentageAboveBaseCapacity": 20,
    "SpotAllocationStrategy": "price-capacity-optimized"
  },
  "LaunchTemplate": {
    "LaunchTemplateSpecification": { "LaunchTemplateName": "web", "Version": "$Latest" },
    "Overrides": [ {"InstanceType": "m7g.large"}, {"InstanceType": "m6g.large"}, {"InstanceType": "c7g.large"} ]
  }
}
```

### Dedicated Instances vs Dedicated Hosts

| | Dedicated Instance | Dedicated Host |
| --- | --- | --- |
| Physical isolation from other accounts | Yes | Yes |
| You see sockets/cores, control placement | No | **Yes** |
| BYOL per-socket/per-core licences | No | **Yes** |
| Host affinity (instance stays on same host across stop/start) | No | Yes |
| Billing | Per instance | Per host |

Exam rule: **licensing** → Dedicated **Host**. "Must not share hardware" with no licensing detail → Dedicated **Instance** (cheaper, simpler).

### The cost decision flow

```
Can it tolerate interruption? ── yes ─► Spot (diversify types/AZs)
          │ no
Runs steadily for 1–3 years? ── no ─► On-Demand (maybe + Capacity Reservation for a known event)
          │ yes
Need flexibility across family/Region/compute service? ── yes ─► Compute Savings Plan
          │ no
Fixed family in one Region ─► EC2 Instance Savings Plan (or Standard RI)
Need guaranteed AZ capacity too ─► + On-Demand Capacity Reservation (or zonal RI)
Licensing per core/socket ─► Dedicated Host
```

Tools: **AWS Compute Optimizer** recommends right-sized instance types from CloudWatch metrics; **Cost Explorer** gives RI/SP purchase recommendations (Day 12).

---

## 4. Placement groups

Placement groups influence *where* instances land on hardware. Free to use.

```
CLUSTER (1 AZ)              SPREAD (per AZ: max 7 running)     PARTITION (per AZ: up to 7 partitions)
┌──── same rack area ───┐   Rack1  Rack2  Rack3  …            P1 (rack set)  P2           P3
│ [i][i][i][i][i][i][i] │   [i]    [i]    [i]                 [i][i][i]      [i][i][i]    [i][i][i]
│  low-latency, high-bw │   each instance on distinct hw      partitions don't share racks;
└───────────────────────┘                                     instances see their partition number
```

| Type | Gives you | Cost | Use when |
| --- | --- | --- | --- |
| **Cluster** | Lowest latency, highest per-flow throughput between instances | Single AZ = single point of failure | HPC, tightly coupled MPI jobs, distributed ML training (pair with EFA) |
| **Spread** | Each instance on distinct hardware | **Max 7 running instances per AZ per group** | Small number of critical instances that must not fail together (e.g. primary/secondary DB nodes, ZooKeeper quorum) |
| **Partition** | Groups of instances isolated from other groups' hardware; topology visible to the app | Up to 7 partitions per AZ | Large distributed, replica-aware systems: **HDFS, HBase, Cassandra, Kafka** |

Tips:

- Cluster: launch all instances in one request with the same type to reduce *insufficient capacity* errors; if it fails, stop and start all of them again.
- Spread = "few, critical, isolated." Partition = "many, distributed, rack-aware."

---

## 5. EBS — Elastic Block Store

EBS volumes are network-attached block devices, **replicated within one AZ** (not across AZs). One volume attaches to one instance at a time (except io1/io2 Multi-Attach), in the **same AZ**.

### Volume types

| Type | Media | Performance model | Boot? | Use for |
| --- | --- | --- | --- | --- |
| **gp3** | SSD | Baseline 3,000 IOPS + 125 MiB/s regardless of size; provision more IOPS/throughput **independently of size** | Yes | **Default** for almost everything: boot, dev/test, most DBs |
| **gp2** | SSD | IOPS tied to size (3 IOPS/GiB), small volumes burst to 3,000 | Yes | Legacy — migrate to gp3 (usually cheaper, decoupled performance) |
| **io2 Block Express** | SSD | Provisioned IOPS, sub-millisecond latency, highest durability (99.999%) | Yes | **Mission-critical, I/O-intensive DBs** (Oracle, SQL Server, SAP HANA), needs > gp3 max IOPS |
| **io1** | SSD | Provisioned IOPS (older) | Yes | Legacy PIOPS |
| **st1** | HDD | Throughput-optimized, burst by MB/s | **No** | Big sequential: log processing, data warehouses, Kafka, ETL |
| **sc1** | HDD | Cold, lowest cost | **No** | Infrequently accessed large sequential data, lowest cost block storage |

Maximum IOPS/throughput/size per type change over time (AWS has raised gp3 and io2 maximums) — **check current docs** for exact ceilings. What you must know is the shape:

- "Need more IOPS but not more capacity" → **gp3** (raise IOPS independently) or **io2** if beyond gp3's ceiling / need sub-ms consistent latency.
- "Sequential throughput, large files, cheapest" → **st1**; "rarely accessed, cheapest" → **sc1**.
- HDD types **can't be boot volumes**.
- Instance-level limits also apply: a big volume on a small instance is capped by the instance's EBS bandwidth (look for **EBS-optimized** / Nitro).

```bash
# Create a gp3 volume with extra IOPS and throughput, decoupled from size
aws ec2 create-volume --availability-zone ap-south-1a \
  --volume-type gp3 --size 100 --iops 6000 --throughput 250 --encrypted
```

### Elastic Volumes

You can **increase size**, **change type** (e.g. gp2 → gp3, gp3 → io2) and **change IOPS/throughput** on a live, attached volume with no downtime. You **cannot shrink** a volume (create a smaller one and copy data). After growing, extend the partition/filesystem inside the OS. There's a cooldown between modifications on the same volume — check current docs.

### Multi-Attach

- **io1/io2 only**, same AZ, multiple Nitro instances (check current docs for the max).
- The application/filesystem must be **cluster-aware** (e.g. a clustered FS); standard ext4/XFS will corrupt.
- Exam cue: "shared block storage for a clustered application in one AZ." If it says shared **file** storage across AZs → **EFS** (Linux) or **FSx** (Day 5), not Multi-Attach.

### Encryption

- AES-256 via **KMS**; encrypts data at rest, in transit between instance and volume, all snapshots, and volumes created from those snapshots. Minimal performance impact (handled by Nitro).
- **Encryption by default** is an account-level, **per-Region** setting — the answer to "ensure all new EBS volumes are encrypted."
- **You can't encrypt an existing unencrypted volume in place.** Path:

```
Unencrypted volume ─► snapshot ─► copy snapshot with encryption (KMS key) ─► create volume ─► attach/swap
                         (or create an encrypted volume directly from the unencrypted snapshot)
```

### Snapshots

- **Point-in-time, incremental** backups stored in AWS-managed S3 (you don't see the bucket). Deleting an older snapshot is safe — only blocks no longer referenced are removed.
- Snapshots are **regional**: create a volume from a snapshot in **any AZ in that Region** — *this is how you move an EBS volume to another AZ*. Copy the snapshot to move it to another Region.
- Can be taken while the volume is in use; for consistency, pause I/O or use application-consistent tools (e.g. AWS Backup with VSS on Windows).
- Volumes restored from snapshots load blocks lazily from S3 → first-read latency. **Fast Snapshot Restore (FSR)** removes this warm-up (costs extra, per AZ).
- **EBS Snapshots Archive**: much cheaper tier for snapshots kept long-term and rarely restored; restore takes hours (up to ~72 h); minimum retention period applies.
- **Recycle Bin**: retention rules that let you recover accidentally deleted snapshots/AMIs.
- Automate with **Amazon Data Lifecycle Manager (DLM)** (EBS snapshots & EBS-backed AMIs) or **AWS Backup** (centralised, cross-service, cross-account/cross-Region — Day 11).

```bash
aws ec2 create-snapshot --volume-id vol-0abc --description "pre-upgrade"
aws ec2 copy-snapshot --source-region ap-south-1 --source-snapshot-id snap-0abc \
  --region eu-west-1 --encrypted --kms-key-id alias/dr-key
```

---

## 6. Instance store

Physical NVMe/SSD/HDD disks on the host (instance types with a `d`, plus I/D families).

| | Instance store | EBS |
| --- | --- | --- |
| Location | Physically on the host | Network-attached, same AZ |
| Performance | Highest IOPS / lowest latency (millions of IOPS on storage-optimized types) | High, provisioned |
| Persistence | **Lost on stop, hibernate, terminate, or host/disk failure**; survives reboot | Independent of instance lifecycle |
| Snapshots | No (you copy data yourself) | Yes |
| Resize / detach | No — fixed by instance type | Yes |
| Cost | Included in the instance price | Billed separately per GB(+IOPS/throughput) |

Use instance store for **buffers, caches, scratch data, temp files**, or data **replicated across a fleet** (Cassandra/Kafka nodes, HDFS). Never for the only copy of anything.

> Trap: "A company stores session data on instance store and loses it after stopping the instance for maintenance." Expected — move it to ElastiCache/DynamoDB or EBS.

### Storage at a glance (EFS/FSx in depth on Day 5)

| Need | Pick |
| --- | --- |
| Boot disk / single-instance block storage | EBS gp3 |
| Highest IOPS, data can be lost | Instance store |
| Sub-ms, very high IOPS, durable, single AZ | EBS io2 Block Express |
| Shared POSIX file system across many instances **and AZs** | EFS |
| Shared Windows (SMB) file system | FSx for Windows File Server |
| Object storage, unlimited, 11 9s durability | S3 |

---

## 7. AMIs — Amazon Machine Images

An AMI = template for the root volume (+ block device mappings + launch permissions). For EBS-backed AMIs it's essentially **snapshots + metadata**.

- AMIs are **regional**. To launch in another Region, **copy the AMI** (copies its snapshots). Launch it in **any AZ** of its Region.
- **Sharing:** add launch permissions for account IDs, OUs or the whole Organization (or make public — rarely right). The AMI stays owned by you; the other account can copy it to own a copy.
- **Encrypted AMIs:** must be encrypted with a **customer managed KMS key**, and you must **share the key** (key policy grant) with the target account. AMIs encrypted with the default `aws/ebs` AWS managed key **cannot be shared** — re-copy with a CMK first.
- Creating an AMI reboots the instance by default for filesystem consistency (`--no-reboot` skips it at the cost of consistency).
- **Golden AMI** pattern: bake OS hardening, agents and dependencies in → faster Auto Scaling launches (less user-data work). Automate with **EC2 Image Builder** (build, test, distribute across Regions/accounts on a schedule).
- **Instance-store-backed AMIs** (legacy): root on instance store; can't stop, only reboot/terminate.
- AMI deprecation / deregistration protection / Recycle Bin exist for lifecycle hygiene.

```
Golden AMI pipeline (EC2 Image Builder)
  base AMI ─► build (patch, install agents) ─► test ─► distribute (Regions, accounts) ─► launch template version
```

Boot-time trade-off (an exam favourite): **"Reduce instance launch time for Auto Scaling"** → pre-baked AMI (+ optionally EC2 **warm pools**, Day 3) instead of long user-data scripts.

---

## 8. Resilience on a single instance

Even with Auto Scaling as the real answer, know the single-instance tools:

| Mechanism | What it does |
| --- | --- |
| **Status checks** | *System* status check (AWS host/network issue) vs *instance* status check (your OS/config). |
| **Simplified automatic recovery** | On by default for supported instances: on a system status check failure, AWS recovers the instance onto new hardware. |
| **CloudWatch alarm → EC2 recover action** | Same idea, explicitly configured. Recovered instance keeps **instance ID, private IPs, Elastic IPs, instance metadata and EBS volumes**. Instance store data is lost; instances with instance store volumes generally aren't eligible. |
| **Auto Scaling group of min=max=1** | Self-healing across AZs (replaces a failed instance, possibly in another AZ) — but new instance ID/IP; pair with EIP re-association or a load balancer. |
| **Stop/start** | Moves the instance to new hardware (for scheduled host retirement events). |

---

## 9. Exam decision table

| If the question says… | Pick |
| --- | --- |
| Steady 24/7 workload for 1–3 years, may change instance family or move to Fargate/Lambda | **Compute Savings Plan** |
| Steady workload, one instance family, one Region, maximum discount | **EC2 Instance Savings Plan** or **Standard RI** (3-yr, All Upfront) |
| Steady workload but instance type will change; want RIs | **Convertible RI** |
| Batch / CI / rendering / stateless workers that can be interrupted, lowest cost | **Spot** (diversified, price-capacity-optimized) |
| Short-term, unpredictable, cannot be interrupted | **On-Demand** |
| Guaranteed capacity in a specific AZ for a known event, no long-term commitment | **On-Demand Capacity Reservation** |
| Discount *and* capacity guarantee | Capacity Reservation + Savings Plan (or **zonal** RI) |
| Bring your own per-socket/per-core licence | **Dedicated Host** |
| Compliance: don't share physical hardware with other customers | **Dedicated Instance** |
| Lowest latency between nodes, HPC, MPI | **Cluster** placement group (+ EFA) |
| A handful of critical instances must not share hardware | **Spread** placement group |
| Hadoop/HDFS, Cassandra, Kafka — rack-aware fault isolation | **Partition** placement group |
| Highest-performance temporary storage / cache / scratch | **Instance store** |
| Default boot/data volume; tune IOPS without growing size | **gp3** |
| Mission-critical DB needing sub-ms latency and highest IOPS/durability | **io2 Block Express** |
| Large sequential throughput (logs, ETL), cheap | **st1** |
| Cold, infrequently accessed block data, cheapest | **sc1** |
| Shared block volume for a cluster-aware app in one AZ | **io2 Multi-Attach** |
| Move an EBS volume to another AZ | **Snapshot → create volume in target AZ** |
| Move / DR an instance to another Region | **Copy AMI (or snapshot) to that Region** |
| Encrypt an existing unencrypted volume | Snapshot → **copy with encryption** → new volume |
| All new EBS volumes must be encrypted | **EBS encryption by default** (per Region) |
| Share an encrypted AMI with another account | Encrypt with a **customer managed KMS key** and share the key + AMI |
| Automate EBS snapshot schedules/retention | **Data Lifecycle Manager** (or AWS Backup for cross-service) |
| Restored volumes must deliver full performance immediately | **Fast Snapshot Restore** |
| Keep snapshots for years at lowest cost, rare restore | **EBS Snapshots Archive** |
| Recover accidentally deleted snapshots/AMIs | **Recycle Bin** |
| Preserve in-memory state across stop/start | **Hibernate** |
| Speed up instance launches | **Golden AMI** (EC2 Image Builder) |
| Public IP must not change after stop/start | **Elastic IP** |
| Right-size instances based on utilisation | **AWS Compute Optimizer** |
| T instance slow under sustained load | Out of CPU credits → move to M/C or **unlimited** mode |

---

## 10. Lab (30 min, free tier)

Goal: feel the AZ boundary of EBS, the snapshot/AMI escape hatch, and live volume changes.

**Note on free tier:** stick to the instance type the console labels *Free tier eligible* (typically `t3.micro` or `t2.micro`) and small gp3 volumes. Free-tier terms differ by account age — check your Billing console.

### Console steps

1. **Launch** an Amazon Linux 2023 instance in AZ **a** (e.g. `ap-south-1a`), free-tier type, IMDSv2 required, the Day 1 instance role (Session Manager access), no key pair, no inbound rules. Add user data:
   ```bash
   #!/bin/bash
   echo "built at $(date)" > /etc/built-at
   ```
2. **Create a 1 GiB gp3 volume** in the *same* AZ, encrypted. Attach it as `/dev/sdf`. In Session Manager:
   ```bash
   lsblk                                      # find the NVMe name, e.g. nvme1n1
   sudo mkfs -t xfs /dev/nvme1n1
   sudo mkdir /data && sudo mount /dev/nvme1n1 /data
   echo "hello from AZ a" | sudo tee /data/proof.txt
   ```
3. **Try the trap:** create another volume in AZ **b** and try attaching it to your instance → the instance doesn't appear in the list. EBS is AZ-locked. Delete that volume.
4. **Elastic Volumes:** modify the 1 GiB volume → size 2 GiB, IOPS 3000 → 4000 (or just size). Watch state go `modifying → optimizing`. Then:
   ```bash
   sudo xfs_growfs /data && df -h /data      # filesystem now sees 2 GiB, no downtime
   ```
5. **Snapshot** the data volume. When complete, **Create volume from snapshot** in AZ **b** — this is how you cross AZs. (Optional: note the *Fast snapshot restore* option and leave it off; it costs extra.)
6. **Create an AMI** from the instance (Actions → Image and templates → Create image). Note it reboots by default. Then **Copy AMI** to a second Region and look at the snapshots it created there.
7. **Stop and start** the instance; compare the public IPv4 before/after (it changes) and confirm `/etc/built-at` and the root disk content persisted. (User data didn't re-run — check the timestamp.)
8. **Tear down** (in both Regions): terminate the instance; delete the data volume and the AZ-b volume; **deregister** both AMIs; **delete** all snapshots (deregistering an AMI does *not* delete its snapshots).

### Terraform variant

```hcl
provider "aws" { region = "ap-south-1" }

data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

resource "aws_instance" "lab" {
  ami                  = data.aws_ami.al2023.id
  instance_type        = "t3.micro"
  availability_zone    = "ap-south-1a"
  iam_instance_profile = "day1-ssm-profile"   # from Day 1, or create one
  user_data            = "#!/bin/bash\necho \"built at $(date)\" > /etc/built-at"

  metadata_options { http_tokens = "required" }
  root_block_device {
    volume_type           = "gp3"
    encrypted             = true
    delete_on_termination = true
  }
}

resource "aws_ebs_volume" "data" {
  availability_zone = aws_instance.lab.availability_zone   # must match the instance's AZ
  size              = 1
  type              = "gp3"
  encrypted         = true
}

resource "aws_volume_attachment" "data" {
  device_name = "/dev/sdf"
  volume_id   = aws_ebs_volume.data.id
  instance_id = aws_instance.lab.id
}

resource "aws_ebs_snapshot" "data" {
  volume_id   = aws_ebs_volume.data.id
  description = "day2-lab"
}

resource "aws_ebs_volume" "restored_in_b" {
  availability_zone = "ap-south-1b"
  snapshot_id       = aws_ebs_snapshot.data.id
  type              = "gp3"
}
```

Try changing `aws_ebs_volume.data.size` to `2` and re-applying — Terraform modifies it in place (Elastic Volumes). Try changing `availability_zone` on `restored_in_b` — Terraform must **replace** it, because volumes can't move between AZs.

**Teardown:** `terraform destroy`, then check the EC2 console for leftover snapshots/AMIs from the console steps.

---

## 11. Practice questions

Answer all ten before opening the answers.

**Q1.** A company runs a nightly batch job that processes files from S3 for about 3 hours. The job checkpoints progress to S3 and can resume if interrupted. The company wants the LOWEST cost. Which option should it use?
A. On-Demand Instances
B. Spot Instances
C. Standard Reserved Instances
D. Dedicated Hosts

**Q2.** A company runs a steady fleet of EC2 instances 24/7. Over the next 3 years it plans to migrate parts of the workload to AWS Fargate and AWS Lambda and may change instance families. Which purchasing option gives the GREATEST savings while allowing this flexibility?
A. EC2 Instance Savings Plan
B. Standard Reserved Instances
C. Compute Savings Plan
D. Spot Instances

**Q3.** A company must run a large sales event next month in a single AZ and needs to be certain that 50 `m7i.xlarge` instances can be launched during the event. It does not want a long-term commitment. What should a solutions architect recommend?
A. Regional Reserved Instances
B. An EC2 Instance Savings Plan
C. An On-Demand Capacity Reservation in that AZ for the event period
D. Spot Instances with a capacity-optimized allocation strategy

**Q4.** A company is migrating an application licensed per physical CPU core and must keep using its existing licences. Which EC2 option meets this requirement?
A. Dedicated Instances
B. Dedicated Hosts
C. On-Demand Instances with default tenancy
D. A cluster placement group

**Q5.** A genomics company runs a tightly coupled HPC simulation using MPI across 32 instances. It needs the lowest possible network latency and highest throughput between nodes. Which solution should be used?
A. Spread placement group across three AZs
B. Partition placement group
C. Cluster placement group with Elastic Fabric Adapter-enabled instances
D. Instances in different Regions connected with VPC peering

**Q6.** An application on EC2 uses a 500 GiB gp2 volume. Monitoring shows the volume is constantly hitting its IOPS limit, but only 120 GiB is used. The company wants more IOPS at the LOWEST cost. What should the architect do?
A. Increase the gp2 volume to 2 TiB
B. Modify the volume to gp3 and provision the required IOPS
C. Change to an st1 volume
D. Move data to instance store

**Q7.** A company has an unencrypted EBS volume attached to a production instance. A new policy requires all data at rest to be encrypted. What is the correct approach?
A. Enable encryption on the existing volume from the console
B. Enable EBS encryption by default; existing volumes will be encrypted automatically
C. Create a snapshot, copy the snapshot with encryption enabled, create a volume from it and replace the original
D. Use an S3 bucket policy to enforce encryption

**Q8.** A company wants to share an AMI with a partner account. The AMI's snapshots are encrypted with the AWS managed key `aws/ebs`. The partner cannot launch instances from it. What should the company do?
A. Make the AMI public
B. Copy the AMI, re-encrypting with a customer managed KMS key, share that key with the partner account and share the new AMI
C. Share the `aws/ebs` key with the partner account
D. Disable encryption on the AMI

**Q9.** A data processing application on EC2 needs temporary scratch space with the highest possible IOPS. Data can be regenerated if lost. Which storage is MOST cost-effective?
A. io2 Block Express volume
B. EBS gp3 volume with maximum IOPS
C. Instance store volumes on a storage-optimized instance
D. Amazon EFS with Max I/O

**Q10.** A company runs a three-node Apache Cassandra cluster per AZ on EC2, with replicas spread so that a single rack failure must not take down multiple replicas. The application must know which rack-group each node is in. Which placement strategy should be used?
A. Cluster placement group
B. Spread placement group
C. Partition placement group
D. No placement group; use an Auto Scaling group

<details>
<summary><strong>Answers</strong></summary>

1. **B** — Interruptible, checkpointed, time-flexible batch = Spot. RIs (C) need a 1–3 year commitment for a 3-hour nightly job.
2. **C** — Only Compute Savings Plans apply across EC2 families, Regions, **Fargate and Lambda**. EC2 Instance SP and Standard RIs are locked to EC2 family/attributes.
3. **C** — Capacity Reservations guarantee AZ capacity with no term commitment. Regional RIs and Savings Plans give discounts but **no** capacity guarantee; Spot can be reclaimed.
4. **B** — Per-core/per-socket BYOL needs visibility of physical cores → Dedicated Hosts. Dedicated Instances isolate hardware but don't expose sockets/cores.
5. **C** — Cluster placement group (single AZ, low latency) plus EFA for OS-bypass MPI traffic. Spread/partition optimise for failure isolation, not latency.
6. **B** — gp2 IOPS scale with size; gp3 decouples IOPS from capacity and is generally cheaper per GiB. Growing gp2 (A) pays for unused space; st1 (C) is HDD.
7. **C** — You can't encrypt an existing volume in place; encryption-by-default (B) only affects *new* volumes.
8. **B** — AMIs encrypted with the AWS managed `aws/ebs` key can't be shared because you can't modify that key's policy. Re-encrypt with a CMK and share the key.
9. **C** — Instance store is included in the instance price and delivers the highest IOPS; perfect for regenerable scratch data.
10. **C** — Partition placement groups isolate groups of racks and expose the partition number to the app — designed for Cassandra/HDFS/Kafka.

</details>

---

## 12. Flashcards (cover the right column)

| Prompt | Answer |
| --- | --- |
| Is an EBS volume regional or zonal? | Zonal — one AZ |
| How do you move an EBS volume to another AZ? | Snapshot → create volume in the new AZ |
| Are EBS snapshots regional or zonal? | Regional (copy to move across Regions) |
| Can you shrink an EBS volume? | No — only grow/change type/IOPS |
| EBS types that can't be boot volumes | st1, sc1 (HDD) |
| Which EBS types support Multi-Attach? | io1 / io2 (same AZ, cluster-aware FS) |
| gp3's big advantage over gp2 | IOPS/throughput provisioned independently of size |
| Instance store data survives… | Reboot only (lost on stop, hibernate, terminate, failure) |
| Default `DeleteOnTermination` for root / extra volumes | true / false |
| Spot interruption notice | 2 minutes |
| AWS-recommended Spot allocation strategy | price-capacity-optimized |
| Which Savings Plan covers Fargate and Lambda? | Compute Savings Plan |
| Which RI type reserves capacity? | Zonal RI (regional RIs don't) |
| Do Savings Plans reserve capacity? | No — use On-Demand Capacity Reservations |
| BYOL per-core licensing | Dedicated Host |
| Spread placement group limit | 7 running instances per AZ per group |
| Placement group for Kafka/HDFS/Cassandra | Partition |
| Placement group for HPC | Cluster (single AZ) |
| AMI scope | Regional — copy to use in another Region |
| Can you share an AMI encrypted with `aws/ebs`? | No — re-encrypt with a customer managed key and share the key |
| Tool to automate EBS snapshot schedules | Data Lifecycle Manager (or AWS Backup) |
| Removes first-read latency on restored volumes | Fast Snapshot Restore |
| Preserve RAM across stop/start | Hibernate |
| Service to build golden AMIs | EC2 Image Builder |
| Right-sizing recommendations | AWS Compute Optimizer |

## 13. Before you close today

- [ ] Read sections 1–9; redraw the purchasing-option decision flow from memory
- [ ] Lab done and torn down in **both** Regions (instances, volumes, AMIs, snapshots)
- [ ] 10 questions above + 20 bank questions on EC2/EBS/purchasing options
- [ ] Misses logged in `miss-log.md`
- [ ] Quick Day 1 recap: re-answer Day 1 Q2, Q3 and Q9 without looking

**Tomorrow (Day 3):** Elastic Load Balancing (ALB vs NLB vs GWLB), target groups, health checks, stickiness, cross-zone, TLS, and EC2 Auto Scaling policies, lifecycle hooks and warm pools.
