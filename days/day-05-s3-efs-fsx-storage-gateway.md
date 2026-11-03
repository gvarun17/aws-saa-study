# Day 5 — S3, EFS, FSx & Storage Gateway

**Domain weight:** heavy in *Design Cost-Optimized Architectures* (20%) — storage classes and lifecycle are the single most common cost topic — plus *Design Secure Architectures* (30%) for S3 access control and encryption, *Design Resilient Architectures* (26%) for versioning, replication and Object Lock, and *High-Performing* (24%) for choosing the right file system.
**Time plan:** Learn 90 min · Lab 30 min · Drill 45 min · Review 30 min.

By the end of today you should be able to:

- Classify any storage requirement as block, file or object, and name the AWS service that fits.
- Pick the cheapest S3 storage class for an access pattern, and design a lifecycle policy without breaking minimum-duration or minimum-size rules.
- Secure a bucket end to end: Block Public Access, bucket policies, Object Ownership, encryption choice (SSE-S3 / SSE-KMS / DSSE-KMS / SSE-C), presigned URLs and CloudFront OAC.
- Explain versioning, replication (CRR/SRR, Batch Replication, RTC) and Object Lock well enough to answer "protect against deletion" and "compliance retention" questions.
- Choose between EFS, FSx for Windows File Server, FSx for Lustre, FSx for NetApp ONTAP and FSx for OpenZFS from a single scenario sentence.
- Pick the right Storage Gateway type (S3 File, Volume cached/stored, Tape) and the right bulk-transfer tool (DataSync, Transfer Family, Snow, Transfer Acceleration).

---

## 1. The storage landscape: block, file, object

```
                 ┌───────────────────────── How is it accessed? ─────────────────────────┐
                 │                                                                        │
         BLOCK (a disk)                    FILE (a shared folder)                 OBJECT (HTTP API)
     one instance mounts it          many clients mount it over NFS/SMB       GET/PUT whole objects by key
                 │                                  │                                     │
   EBS (Day 2) · instance store       EFS (Linux, NFS)                         Amazon S3
                                      FSx for Windows (SMB)                    (+ Glacier storage classes)
                                      FSx for Lustre (HPC)
                                      FSx for NetApp ONTAP (NFS+SMB+iSCSI)
                                      FSx for OpenZFS (NFS)
                 └──────────── Hybrid bridge to on-premises: Storage Gateway ─────────────┘
```

| | Block (EBS) | File (EFS / FSx) | Object (S3) |
| --- | --- | --- | --- |
| Access | Mounted as a disk by an OS | Mounted as a share by many clients | HTTPS API (`GetObject`, `PutObject`) |
| Concurrent clients | One instance (Multi-Attach is a narrow io1/io2 exception) | Thousands | Effectively unlimited |
| Scope | One AZ | EFS: Regional (multi-AZ) or One Zone; FSx: single- or multi-AZ deployments | Regional (Standard stores across ≥ 3 AZs) |
| Edit in place | Yes (byte-level) | Yes (POSIX/SMB semantics) | No — replace the whole object |
| Typical exam cue | "Database on EC2", "boot volume", "lowest latency for one instance" | "Shared file system", "multiple instances need the same files", "lift-and-shift file server" | "Unlimited, durable, cheap storage", "static website", "data lake", "backup/archive" |

> Exam reflex: if more than one instance must **read and write the same files**, the answer is a file system (EFS or FSx), never EBS and rarely S3 (S3 isn't a POSIX file system, though Mountpoint for Amazon S3 exists for read-heavy workloads).

---

## 2. Amazon S3 fundamentals

### Buckets, objects and keys

- A **bucket** lives in **one Region**; its **name is globally unique** (Day 1 trap).
- An **object** = key + data + metadata (+ version ID, tags). Objects are immutable — an "edit" is a new PUT.
- There are no real folders. `reports/2026/nov.csv` is a single key; the console shows `/`-separated **prefixes** as folders.
- **Durability:** designed for 11 nines (99.999999999%) across all classes. **Availability** differs per class (that's the knob that changes).
- **Consistency:** **strong read-after-write** for all PUTs, overwrites, deletes and LISTs — no "eventual consistency" caveats any more. If an answer choice tries to work around S3 eventual consistency, it's outdated.

### Size limits that drive design

| Rule | Value / guidance |
| --- | --- |
| Single PUT upload | Up to 5 GB |
| Multipart upload | Recommended above ~100 MB; **required** above 5 GB. Parts upload in parallel, retry individually |
| Maximum object size | 5 TB has long been the limit and is what the exam assumes; AWS has raised it — check current docs |
| Objects per bucket | Unlimited |

> Abandoned multipart uploads keep their parts — and you pay for them. Add a lifecycle rule: **AbortIncompleteMultipartUpload** after N days.

### Performance

- S3 scales per **prefix**: at least **3,500 PUT/COPY/POST/DELETE** and **5,500 GET/HEAD requests per second per prefix**. Spread hot keys over more prefixes to scale further (`logs/a1/…`, `logs/b7/…`).
- **Multipart upload** — parallel PUTs for large files.
- **Byte-range fetches** — parallel GETs of parts of a large object, or read just a header.
- **S3 Transfer Acceleration** — clients upload to the nearest CloudFront edge, then AWS's backbone carries data to the bucket. For **long-distance uploads** from users worldwide. Uses a distinct endpoint (`bucket.s3-accelerate.amazonaws.com`).
- **S3 Express One Zone** — a high-performance class in **directory buckets** in a single AZ, for single-digit-ms latency and very high request rates (ML training, analytics scratch). Single AZ → not for data you can't regenerate.
- **CloudFront** in front of S3 for read-heavy global delivery (Day 9).

> Querying inside objects: on the exam, "run SQL on data in S3 without loading it" → **Amazon Athena** (Day 12). S3 Select is no longer offered to new customers (check current docs) — don't build new designs on it.

---

## 3. S3 storage classes — the cost heart of the exam

| Class | AZs | Min storage duration | Min billable size | Retrieval | Use when |
| --- | --- | --- | --- | --- | --- |
| **S3 Standard** | ≥ 3 | — | — | ms, no fee | Frequently accessed, unknown-but-hot |
| **S3 Intelligent-Tiering** | ≥ 3 | — | Objects < 128 KB aren't monitored or auto-tiered (stay in frequent tier) | ms for the default tiers; no retrieval fees; small per-object monitoring fee | **Unknown or changing** access patterns |
| **S3 Standard-IA** | ≥ 3 | 30 days | 128 KB | ms, **per-GB retrieval fee** | Infrequent but needs fast access (backups, DR copies) |
| **S3 One Zone-IA** | **1** | 30 days | 128 KB | ms, per-GB fee | Infrequent, **re-creatable** data (secondary copies, thumbnails) |
| **S3 Glacier Instant Retrieval** | ≥ 3 | 90 days | 128 KB | **ms**, higher per-GB fee | Archive accessed ~once a quarter that still needs **instant** access (medical images, news archives) |
| **S3 Glacier Flexible Retrieval** | ≥ 3 | 90 days | (small per-object overhead) | Expedited **1–5 min**, Standard **3–5 h**, Bulk **5–12 h** | Archive, retrieval in minutes–hours is fine |
| **S3 Glacier Deep Archive** | ≥ 3 | 180 days | (small per-object overhead) | Standard **within 12 h**, Bulk **within 48 h** | Cheapest. 7–10-year compliance retention, tape replacement |
| **S3 Express One Zone** | **1** | — | — | Single-digit ms | Latency-critical, high request rate, single-AZ compute |

Storage price falls as you go down (roughly Standard → IA → Glacier IR → Flexible → Deep Archive); retrieval cost and latency rise. Check current pricing for actual numbers.

**Retrieving from Glacier Flexible/Deep Archive** is a **restore**: you call `RestoreObject`, S3 creates a **temporary copy** (for N days you choose) that you then GET. The object stays in its archive class. Provisioned capacity units guarantee Expedited retrievals are available when you need them.

### Intelligent-Tiering tiers

```
 Frequent Access ──30 days no access──► Infrequent Access ──90 days──► Archive Instant Access
   (automatic)                            (automatic)                     (automatic)
                                                 │ optional, opt-in per bucket/prefix/tag:
                                                 ├──90+ days──► Archive Access      (async restore, like Flexible)
                                                 └──180+ days─► Deep Archive Access (async restore, like Deep Archive)
 Any access to an object in an automatic tier moves it back to Frequent Access — with no retrieval fee.
```

**When Intelligent-Tiering beats lifecycle rules:** when you **can't predict** access. When you *can* predict ("logs are never read after 30 days"), a lifecycle rule is cheaper (no monitoring fee).

### Lifecycle policies

Two action types:

- **Transition** — move objects to a cheaper class after N days.
- **Expiration** — delete objects (or noncurrent versions, or expired delete markers) after N days.

```
 Standard ─► Intelligent-Tiering ─► Standard-IA ─► One Zone-IA ─► Glacier IR ─► Glacier Flexible ─► Deep Archive
   (transitions only flow "downhill" — you can't lifecycle an object back up to Standard)
```

Rules to know:

1. Objects must be in Standard **≥ 30 days** before a lifecycle transition to **Standard-IA or One Zone-IA**.
2. Moving into a class with a minimum duration and then out (or deleting) early bills the **remaining minimum** anyway. "Transition to Standard-IA at day 31, delete at day 45" → charged as if stored in IA for 30 days.
3. By default, lifecycle **doesn't transition objects smaller than 128 KB** (it would cost more, not less). Many tiny files → IA is a bad answer.
4. Rules can be scoped by **prefix** and/or **object tags**, and target **current** or **noncurrent** versions.

```json
{
  "Rules": [{
    "ID": "logs-tiering",
    "Status": "Enabled",
    "Filter": { "Prefix": "logs/" },
    "Transitions": [
      { "Days": 30,  "StorageClass": "STANDARD_IA" },
      { "Days": 90,  "StorageClass": "GLACIER_IR" },
      { "Days": 180, "StorageClass": "DEEP_ARCHIVE" }
    ],
    "Expiration": { "Days": 2555 },
    "NoncurrentVersionExpiration": { "NoncurrentDays": 30 },
    "AbortIncompleteMultipartUpload": { "DaysAfterInitiation": 7 }
  }]
}
```

**S3 Storage Class Analysis** observes access patterns on a bucket/prefix and recommends when to transition to Standard-IA. **S3 Storage Lens** gives organisation-wide usage and activity metrics with cost-saving recommendations (e.g. buckets with no lifecycle rule, incomplete multipart uploads).

> Traps:
> - "Data accessed unpredictably" → Intelligent-Tiering, not a lifecycle rule.
> - "Infrequently accessed, must be available immediately, must survive an AZ loss" → **Standard-IA**, not One Zone-IA.
> - "Accessed once or twice a year, retrieval in milliseconds" → **Glacier Instant Retrieval**.
> - "Retrieve within 12 hours, cheapest" → **Deep Archive**. "Within minutes, occasionally" → Glacier Flexible with **Expedited** (provisioned capacity if it must be guaranteed).
> - Storage classes are set **per object**, not per bucket.

---

## 4. Protecting data: versioning, Object Lock, replication

### Versioning

States: **Unversioned** (default) → **Enabled** → **Suspended**. Once enabled it can only be *suspended*, never removed.

```
PUT photo.jpg        → version v1
PUT photo.jpg        → version v2 (current); v1 becomes noncurrent
DELETE photo.jpg     → a DELETE MARKER becomes "current"; v1, v2 still exist  (GET returns 404)
DELETE marker itself → v2 is current again  ← this is how you "undelete"
DELETE ?versionId=v1 → v1 is permanently gone
```

- Protects against **accidental overwrite and deletion**. Pair it with a lifecycle rule expiring noncurrent versions, or costs creep up.
- **MFA Delete** — requires MFA to permanently delete a version or change versioning state. Only the **root user** can enable it, via CLI/API.

### S3 Object Lock (WORM)

Write-once-read-many. Requires **versioning** (enabled automatically when you turn Object Lock on). Locks apply to **object versions**.

| Mode / control | Who can delete or shorten retention? | Use |
| --- | --- | --- |
| **Governance mode** | Users with `s3:BypassGovernanceRetention` (plus the bypass header) | Protect from most users; admins can still fix mistakes; test retention settings |
| **Compliance mode** | **Nobody — not even the root user** — until retention expires. Retention can be extended, never shortened | SEC 17a-4 / FINRA-style regulatory retention |
| **Legal hold** | Anyone with `s3:PutObjectLegalHold` can remove it; no expiry date | Litigation holds, independent of retention period |

> "Must be impossible for anyone, including administrators and the root user, to delete records for 7 years" → **Object Lock in compliance mode** with a 7-year retention period. (S3 Glacier **Vault Lock** is the older vault-level equivalent for the Glacier service API.)

### Replication

```
 Source bucket (versioning ON) ──IAM role──► Destination bucket (versioning ON)
   ap-south-1                                  eu-west-1   = Cross-Region Replication (CRR)
                                               ap-south-1  = Same-Region Replication (SRR)
```

| Feature | Detail |
| --- | --- |
| Prerequisites | **Versioning on both buckets**; an IAM role S3 assumes; KMS permissions if objects use SSE-KMS |
| What replicates | **New** objects written after the rule is enabled, with metadata, tags, (optionally) Object Lock settings |
| Existing objects | **Not** replicated by a rule — use **S3 Batch Replication** |
| Deletes | Delete **markers** replicate only if you enable *delete marker replication*. Permanent deletes of a specific version are **never** replicated (protection against malicious deletes) |
| Chaining | Not transitive: A→B and B→C does not copy A's objects to C |
| Cross-account | Supported; set **owner override** so the destination account owns the replicas |
| Timing | Usually seconds–minutes; **S3 Replication Time Control (RTC)** gives an SLA (99.99% of objects within **15 minutes**) plus metrics |
| Storage class | Can replicate into a cheaper class (e.g. replicas into Standard-IA or Glacier) |

Use CRR for **compliance (copy in another Region)**, **lower latency** for users elsewhere, and **DR**. Use SRR for **log aggregation** into one bucket, or **prod → test** account copies.

**S3 Multi-Region Access Points** give one global endpoint that routes requests to the nearest of several replicated buckets, with failover controls — "single global S3 endpoint with active-active buckets".

---

## 5. Securing S3

### The layers

```
Request ─► Block Public Access (account + bucket) ─► Explicit DENY anywhere? ─► Bucket policy / access point policy
                                                                          ─► IAM identity policy
                                                                          ─► (ACLs — disabled by default)
                                                                          ─► KMS key policy (if SSE-KMS)
```

| Control | What it does | Exam notes |
| --- | --- | --- |
| **Block Public Access (BPA)** | Overrides any policy/ACL that would make data public. Four settings, at **account** and **bucket** level | **On by default** for new buckets. "Ensure no bucket in the account can ever be public" → account-level BPA (and an SCP denying changes to it) |
| **Bucket policy** | Resource-based JSON policy | Cross-account access, force HTTPS, force encryption, restrict to VPC endpoint / org / CloudFront |
| **IAM policies** | Identity-based | Same-account access |
| **Object Ownership** | *Bucket owner enforced* = **ACLs disabled**, bucket owner owns every object | The **default** for new buckets. ACLs are legacy — avoid them in answers unless the question insists |
| **S3 Access Points** | Named endpoints per application/team, each with its own policy; can be restricted to a VPC | "Hundreds of teams, one huge bucket policy is unmanageable" → access points |
| **Presigned URLs** | Time-limited URL signed with the creator's credentials | Let a user upload/download one object without AWS credentials. Valid up to the creator's credential lifetime (max 7 days with IAM user SigV4 keys) |
| **CORS** | Browser permission for cross-origin requests | "Website on domain A calls bucket B from JavaScript and the browser blocks it" → configure CORS on bucket B |

Classic bucket policy: force HTTPS and deny unencrypted-with-wrong-key uploads.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": ["arn:aws:s3:::acme-records", "arn:aws:s3:::acme-records/*"],
      "Condition": { "Bool": { "aws:SecureTransport": "false" } }
    },
    {
      "Sid": "RequireKmsEncryption",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::acme-records/*",
      "Condition": { "StringNotEquals": { "s3:x-amz-server-side-encryption": "aws:kms" } }
    }
  ]
}
```

### Encryption

All new objects are encrypted at rest **by default with SSE-S3**. Your choice is about **who controls the key** and **what's audited**.

| Option | Key managed by | Audit in CloudTrail? | Notes |
| --- | --- | --- | --- |
| **SSE-S3** | S3 (AES-256) | No per-key usage trail | Default, free, zero effort |
| **SSE-KMS** | AWS KMS (AWS managed `aws/s3` or **customer managed key**) | **Yes**, every key use | Key policy controls who can decrypt; rotation; separation of duties. Each request calls KMS → KMS request quotas and cost |
| **DSSE-KMS** | KMS, two independent layers of encryption | Yes | For compliance that mandates **dual-layer** encryption |
| **SSE-C** | **You** supply the key on every request; S3 doesn't store it | No | HTTPS required. You lose the key → you lose the data. AWS has been tightening SSE-C defaults for new buckets — check current docs |
| **Client-side** | You encrypt before upload | — | "S3 must never see plaintext" |

**S3 Bucket Keys** — S3 uses a short-lived bucket-level key derived from your KMS key, cutting **KMS requests (and cost) dramatically**. The answer to "SSE-KMS costs are high / KMS throttling on a busy bucket".

> Traps:
> - Reading an SSE-KMS object needs **`kms:Decrypt` on the key** as well as `s3:GetObject`. "User has s3:GetObject but gets AccessDenied" on an SSE-KMS object → missing KMS permission.
> - Cross-account access to SSE-KMS objects: the **AWS managed key `aws/s3` can't be shared** cross-account; use a customer managed key and grant the other account in its key policy.
> - Encryption **in transit** = HTTPS (`aws:SecureTransport`), not an S3 encryption setting.

### Serving content: static websites and CloudFront

- **S3 static website hosting:** HTTP-only website endpoint (`bucket.s3-website-<region>.amazonaws.com`), index/error documents, redirects. Requires the content to be publicly readable. No HTTPS on the website endpoint itself.
- **Better pattern:** private bucket + **CloudFront** with **Origin Access Control (OAC)** + bucket policy allowing only that distribution. Gives HTTPS, custom domain with ACM, caching, WAF. OAC replaces the legacy Origin Access Identity (OAI).

```json
{
  "Sid": "AllowCloudFrontOAC",
  "Effect": "Allow",
  "Principal": { "Service": "cloudfront.amazonaws.com" },
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::acme-site/*",
  "Condition": { "StringEquals": { "AWS:SourceArn": "arn:aws:cloudfront::111122223333:distribution/E1ABCDEF" } }
}
```

### Other S3 features you must recognise

| Feature | Exam cue |
| --- | --- |
| **Event notifications** → SNS, SQS, Lambda, or **EventBridge** | "When an image is uploaded, create a thumbnail" → S3 event → Lambda. Need advanced filtering / many targets → EventBridge |
| **S3 Inventory** | Scheduled CSV/Parquet list of objects + metadata (encryption status, replication status) — audit at scale |
| **S3 Batch Operations** | Run one action on billions of objects (copy, tag, restore, invoke Lambda, replicate existing objects) |
| **Requester Pays** | The downloader pays request and transfer costs — sharing large datasets |
| **Object tags** | Drive lifecycle filters, ABAC, and cost allocation |
| **S3 Storage Lens** | Org-wide storage usage dashboards and recommendations |
| **Access logs / CloudTrail data events** | Who accessed which object (CloudTrail data events are off by default) |

---

## 6. Moving data into AWS

| Tool | How it works | Pick it when |
| --- | --- | --- |
| **AWS DataSync** | Agent (VM on-prem) or agentless between AWS services; copies NFS/SMB/HDFS/object storage → S3, EFS, FSx; verifies integrity, schedules, throttles bandwidth | **Online** migration or recurring sync of file data over the network/DX — "migrate 50 TB of NFS shares to EFS with minimal effort" |
| **AWS Transfer Family** | Managed **SFTP/FTPS/FTP/AS2** endpoints backed by S3 or EFS | "Partners upload via SFTP and we don't want to run servers" |
| **S3 Transfer Acceleration** | Edge locations + AWS backbone | Users far from the bucket's Region uploading over the internet |
| **Multipart upload** | Parallel parts | Any large object; throughput and resilience |
| **AWS Snow Family** (Snowball Edge) | Ship a rugged device; load data locally; AWS imports to S3 | Tens–hundreds of TB with **limited bandwidth** or no network, or edge compute in disconnected sites. AWS has been restricting Snow devices for new customers — check current docs, but the exam still uses them |
| **Direct Connect** (Day 4) | Dedicated link | Ongoing, predictable, high-volume hybrid traffic |
| **Storage Gateway** (section 9) | On-prem appliance with a local cache | Ongoing **hybrid** access, not a one-off move |

Rule of thumb for "online vs offline": estimate transfer time = data size ÷ usable bandwidth. 100 TB over a 100 Mbps link is **months** → Snowball. Same data over 10 Gbps DX is about a day → DataSync.

---

## 7. Amazon EFS (Elastic File System)

Managed **NFS v4.1** file system for **Linux** (no Windows). Elastic — grows and shrinks automatically, pay for what you store.

```
                 ┌────────────────────── EFS file system (Regional) ─────────────────────┐
                 │  data stored redundantly across multiple AZs                         │
                 └───────▲───────────────────────▲──────────────────────────▲───────────┘
                  mount target (ENI)       mount target (ENI)          mount target (ENI)
                  AZ a, SG allows 2049     AZ b                         AZ c
                       ▲                        ▲                            ▲
                EC2 / ECS / EKS / Lambda   EC2 (ASG)                   on-prem via DX/VPN
```

- **One mount target per AZ** (an ENI in a subnet). Its security group must allow **TCP 2049 (NFS)** from clients.
- **Regional** (multi-AZ, default) vs **One Zone** (cheaper, single AZ — acceptable for dev or re-creatable data).
- Mountable concurrently by thousands of EC2 instances, **ECS/Fargate tasks, EKS pods and Lambda functions**, plus on-prem servers over DX/VPN.

| Setting | Options | Exam guidance |
| --- | --- | --- |
| **Throughput mode** | **Elastic** (scales automatically, pay per use) · **Provisioned** (fixed throughput independent of size) · **Bursting** (scales with stored size, burst credits) | Spiky/unknown → Elastic (the recommended default). Steady high throughput on a small file system → Provisioned. "Throughput drops as burst credits run out" → switch from Bursting |
| **Performance mode** | **General Purpose** (lowest latency, default) · Max I/O (legacy, higher latency) | General Purpose for nearly everything; check current docs before picking Max I/O |
| **Storage classes** | **Standard** · **Infrequent Access (IA)** · **Archive** (and One Zone variants) | **Lifecycle management** moves files not accessed for N days to IA/Archive — "cut EFS cost for rarely read files" |

Other features: encryption at rest (KMS, set at creation) and in transit (TLS via the EFS mount helper); **EFS access points** (enforce a POSIX user/group and root directory per application — great for multi-tenant containers and Lambda); **EFS Replication** to another Region or AZ; integration with **AWS Backup**.

> Traps:
> - EFS = **Linux/NFS**. Windows workloads with SMB → FSx for Windows File Server.
> - EFS is more expensive per GB than EBS or S3; use it because you need **shared** POSIX access, not as cheap bulk storage.
> - Mount fails / hangs → check the mount target SG allows 2049 from the client SG.

---

## 8. Amazon FSx — four managed file systems

FSx gives you fully managed, popular third-party file systems. The exam pattern is "match the scenario keyword to the flavour".

| | **FSx for Windows File Server** | **FSx for Lustre** | **FSx for NetApp ONTAP** | **FSx for OpenZFS** |
| --- | --- | --- | --- | --- |
| Protocols | **SMB**, NTFS, Windows ACLs | Lustre client (POSIX, Linux) | **NFS, SMB and iSCSI** (multi-protocol) | **NFS** |
| Clients | Windows (and Linux via SMB) | Linux HPC clusters | Linux, Windows, macOS, VMware Cloud on AWS | Linux, macOS, Windows NFS clients |
| Identity | **Active Directory** integration (AWS Managed Microsoft AD or self-managed) | POSIX | AD for SMB | POSIX |
| Signature features | DFS Namespaces, shadow copies (previous versions), data deduplication, Multi-AZ | **Sub-ms latency, hundreds of GB/s**; **links to an S3 bucket** (lazy-loads objects as files, writes back) | Snapshots, **FlexClone**, SnapMirror, dedupe/compression, auto-tiering to a capacity pool | Snapshots, clones, very low latency; ZFS compression |
| HA options | Single-AZ or **Multi-AZ** | **Scratch** (temporary, no replication, cheapest) vs **Persistent** (replicated within an AZ) | Single-AZ or Multi-AZ | Single-AZ or Multi-AZ |
| Exam cue | "Windows file server", "SMB", "Active Directory", "lift-and-shift home directories" | "HPC", "machine learning training", "financial modelling", "video rendering", "**process data that lives in S3** at high speed" | "**NetApp** on-prem", "**multi-protocol** (NFS and SMB)", "iSCSI", "migrate ONTAP with SnapMirror" | "migrate **ZFS** / Linux NFS file servers", "lift-and-shift NFS with low latency" |

```
FSx for Lustre linked to S3 (classic HPC pattern)

 s3://genomics-raw  ◄── data repository association ──►  FSx for Lustre (scratch)
                                                              ▲   ▲   ▲
                                                     1,000s of EC2 compute nodes (cluster placement group)
 Objects appear as files on first access; results exported back to S3; delete the file system when the job ends.
```

> Traps:
> - **Scratch** Lustre = short-term processing, data not replicated; **Persistent** = longer-term, replicated within the AZ.
> - For a Windows share that must survive an AZ failure → FSx for Windows **Multi-AZ**.
> - "Shared storage for Linux, simplest, elastic, pay per use" → EFS, not FSx. FSx wins when a **specific file system** (Windows/SMB, Lustre, ONTAP, ZFS) or its features are named.

---

## 9. AWS Storage Gateway — hybrid storage

A gateway runs on-prem (VM on VMware/Hyper-V/KVM, or a hardware appliance) or as an EC2 instance. It gives on-prem apps a **local protocol** and a **local cache**, and stores data durably in AWS.

| Gateway type | On-prem protocol | Data lands in | Exam cue |
| --- | --- | --- | --- |
| **S3 File Gateway** | **NFS / SMB** file shares | **S3 objects** (one file = one object; native S3 access from AWS too) | "On-prem apps write files; we want them in S3 for analytics/lifecycle"; "file share backed by S3 with local cache" |
| **Volume Gateway — cached** | **iSCSI** block volumes | Primary data in S3 (as volumes); **frequently accessed data cached** locally | "Expand on-prem block storage into AWS, keep hot data local" |
| **Volume Gateway — stored** | **iSCSI** | **Full dataset local**, asynchronously backed up to AWS as **EBS snapshots** | "Low-latency access to the *entire* dataset locally, with off-site backups" |
| **Tape Gateway** | **iSCSI virtual tape library (VTL)** | Virtual tapes in S3; archived tapes in **Glacier Flexible / Deep Archive** | "Replace physical tapes without changing backup software (Veeam, NetBackup, Commvault…)" |
| **FSx File Gateway** | SMB | FSx for Windows File Server | Low-latency on-prem access to an FSx share. **No longer offered to new customers** (check current docs) — prefer FSx directly or a different pattern |

```
On-prem server ──NFS/SMB──► S3 File Gateway (local cache) ──HTTPS / DX──► S3 bucket ──lifecycle──► Glacier
                                                                              └── Athena, Lambda, etc. read the same objects
```

> Traps:
> - "Cached" keeps **hot data** local; "stored" keeps **all data** local. Read the latency requirement carefully.
> - Storage Gateway is for **ongoing hybrid** use. A **one-time migration** → DataSync (online) or Snowball (offline).
> - Tape Gateway is the only answer that keeps existing tape-based **backup software unchanged**.

---

## 10. Putting it together: which storage?

| Need | Service |
| --- | --- |
| Boot volume / database on one EC2 instance | EBS (gp3 default, io2 for high IOPS — Day 2) |
| Temporary scratch, highest IOPS, data loss OK | Instance store |
| Shared Linux POSIX file system, elastic | **EFS** |
| Shared Windows SMB file system with AD | **FSx for Windows File Server** |
| HPC / ML on S3 data, extreme throughput | **FSx for Lustre** |
| NetApp features, NFS + SMB + iSCSI | **FSx for NetApp ONTAP** |
| Lift-and-shift ZFS/NFS servers | **FSx for OpenZFS** |
| Unlimited durable objects, data lake, static assets, backups | **S3** |
| Long-term archive, rare access | **S3 Glacier Flexible / Deep Archive** |
| On-prem apps need cloud-backed storage with local cache | **Storage Gateway** |

---

## 11. Exam traps roundup

1. **Versioning is required** for replication and Object Lock — on both buckets for replication.
2. Replication rules copy **new** objects only. Existing objects → **S3 Batch Replication**.
3. Lifecycle to Standard-IA/One Zone-IA needs **≥ 30 days** in Standard; early deletion still bills the minimum duration.
4. One Zone-IA and Express One Zone lose data if the AZ is destroyed — only for re-creatable data.
5. Compliance-mode Object Lock can't be bypassed by **anyone**, root included. Governance mode can be.
6. "Users can't access objects encrypted with SSE-KMS" → add `kms:Decrypt` (key policy/IAM), not more S3 permissions.
7. SSE-KMS at very high request rates → **S3 Bucket Keys**.
8. Serve private S3 content via CloudFront → **OAC** + bucket policy; keep Block Public Access on.
9. EFS = Linux/NFS; FSx for Windows = SMB/AD. Don't mix them up.
10. Storage Gateway types: files → S3 File Gateway; block/iSCSI → Volume Gateway; tapes → Tape Gateway.
11. Many small objects (< 128 KB): IA/Glacier IR and Intelligent-Tiering auto-tiering don't help much — minimum billable size and no monitoring.
12. "Unpredictable access" → Intelligent-Tiering; "known pattern" → lifecycle rule.

---

## 12. Exam decision table

| If the question says… | Pick |
| --- | --- |
| Access pattern unknown or changing; minimise cost; no retrieval delays | **S3 Intelligent-Tiering** |
| Infrequent access, millisecond retrieval, must survive AZ loss | **S3 Standard-IA** |
| Infrequent access, data is easily re-created, cheapest ms access | **S3 One Zone-IA** |
| Archive accessed ~quarterly, needs ms retrieval | **S3 Glacier Instant Retrieval** |
| Archive, retrieval in minutes to hours is OK | **S3 Glacier Flexible Retrieval** (Expedited for 1–5 min) |
| Cheapest storage, retrieval within 12–48 h, 7–10-year retention | **S3 Glacier Deep Archive** |
| Automatically move objects to cheaper classes as they age | **S3 Lifecycle** transitions |
| Clean up old versions / incomplete multipart uploads | Lifecycle **NoncurrentVersionExpiration** / **AbortIncompleteMultipartUpload** |
| Protect against accidental deletion or overwrite | **Versioning** (+ MFA Delete) |
| WORM, nobody (incl. root) can delete before date | **Object Lock — compliance mode** |
| WORM but privileged admins may override | **Object Lock — governance mode** |
| Copy objects to another Region for DR/compliance | **Cross-Region Replication** |
| Replicate objects that already exist | **S3 Batch Replication** |
| Replication with a 15-minute SLA | **S3 Replication Time Control** |
| Aggregate logs from many buckets in one Region | **Same-Region Replication** |
| One global endpoint, nearest of several replicated buckets | **S3 Multi-Region Access Points** |
| Let a user download/upload one object temporarily without AWS creds | **Presigned URL** |
| Serve private S3 content over HTTPS globally | **CloudFront + OAC** |
| Browser on another domain blocked calling the bucket | **CORS** configuration |
| Many teams/apps, bucket policy too complex | **S3 Access Points** |
| Audit every use of the encryption key | **SSE-KMS** (customer managed key) |
| Reduce KMS cost/throttling for SSE-KMS | **S3 Bucket Keys** |
| Two layers of encryption mandated | **DSSE-KMS** |
| Faster uploads from users across the world | **S3 Transfer Acceleration** (+ multipart) |
| Upload of multi-GB files fails partway | **Multipart upload** |
| Run SQL directly on files in S3 | **Athena** |
| Trigger processing when an object arrives | **S3 event notification → Lambda / SQS / SNS / EventBridge** |
| Shared file system for Linux EC2 across AZs | **EFS** (Regional) |
| Shared storage for Lambda or Fargate | **EFS** (with access points) |
| Reduce EFS cost for rarely read files | **EFS lifecycle management** → IA/Archive |
| Windows SMB shares with Active Directory | **FSx for Windows File Server** (Multi-AZ for HA) |
| HPC / ML, sub-ms, hundreds of GB/s, data in S3 | **FSx for Lustre** linked to S3 |
| NFS + SMB + iSCSI or NetApp migration | **FSx for NetApp ONTAP** |
| Migrate ZFS file servers | **FSx for OpenZFS** |
| On-prem file shares backed by S3 with local cache | **S3 File Gateway** |
| On-prem iSCSI, hot data local, rest in AWS | **Volume Gateway — cached** |
| On-prem iSCSI, all data local, backups in AWS | **Volume Gateway — stored** |
| Replace physical tapes, keep backup software | **Tape Gateway** |
| Online migration/sync of NFS/SMB data to S3/EFS/FSx | **DataSync** |
| Managed SFTP into S3 | **AWS Transfer Family** |
| 100s of TB, slow or no network | **Snowball Edge** |

---

## 13. Lab (30 min, free tier)

Goal: see versioning and delete markers with your own eyes, attach a lifecycle policy, enforce HTTPS + encryption with a bucket policy, and hand out a presigned URL.

**Cost note:** S3 storage and requests for a few small test files cost almost nothing, and new accounts get free-tier/credit coverage (check current free-tier terms). Don't enable replication RTC, don't upload large files, and delete everything at the end.

### Console steps

1. **S3** → *Create bucket* → name `saa-lab-<your-initials>-<random>`, Region of your choice. Keep **Block all public access ON**, **Object Ownership: ACLs disabled**, enable **Bucket Versioning**. Default encryption: **SSE-KMS** with the AWS managed key `aws/s3`, **Bucket Key enabled**.
2. Create a local file `notes.txt` containing `v1` and upload it. Edit it to `v2` and upload again. In the bucket, toggle **Show versions** — two version IDs.
3. **Delete** `notes.txt` (with *Show versions* off). Turn *Show versions* on: a **delete marker** is now the current version. Delete the **delete marker** → `notes.txt` is back with `v2`. That's how you undelete.
4. Click the object → *Properties* → confirm **Server-side encryption: SSE-KMS** and the KMS key ARN.
5. **Bucket policy** (*Permissions* → *Bucket policy*): paste the policy from section 5 with your bucket name (deny `aws:SecureTransport = false`; deny PutObject unless `aws:kms`). From CloudShell, test:
   ```bash
   B=saa-lab-xx-1234
   echo hello > a.txt
   aws s3 cp a.txt s3://$B/a.txt --sse aws:kms                  # works: header says aws:kms
   aws s3api put-object --bucket $B --key b.txt --body a.txt --server-side-encryption AES256   # AccessDenied
   aws s3 cp a.txt s3://$B/c.txt                                # AccessDenied too
   ```
   The second call is blocked because it asked for SSE-S3. The third is blocked because it sent **no** encryption header — `StringNotEquals` matches when the key is missing, even though default encryption would have used KMS. That's why this "require the header" policy is less common now that default bucket encryption exists. **Remove the `RequireKmsEncryption` statement** before continuing (keep the HTTPS one) so later uploads work.
6. **Presigned URL:**
   ```bash
   aws s3 presign s3://$B/notes.txt --expires-in 300
   ```
   Open the URL in a browser — it works for 5 minutes with no login. Try `https://$B.s3.amazonaws.com/notes.txt` without a signature → `AccessDenied` (BPA + no public policy).
7. **Lifecycle rule** (*Management* → *Create lifecycle rule*): name `lab-tiering`, scope prefix `logs/`. Transitions: current versions → Standard-IA at 30 days, Glacier Instant Retrieval at 90, Glacier Deep Archive at 180. Expire noncurrent versions after 30 days. Delete incomplete multipart uploads after 7 days. Notice the console **warns** if you try Standard-IA before day 30.
8. **Optional — event notification:** *Properties* → *Event notifications* → create one for `s3:ObjectCreated:*` with prefix `uploads/`, destination **EventBridge** (toggle *Send notifications to Amazon EventBridge: On*). Upload to `uploads/` and look for the event in an EventBridge rule's monitoring (or skip — Day 8 covers it).
9. **Optional — EFS:** create an EFS file system in the default VPC (Regional, Elastic throughput). Look at the **mount targets** (one per AZ) and their SG. Note *Lifecycle management* defaults. Delete it straight away if you don't mount it.
10. **Tear down:** *Empty* the bucket (this deletes **all versions and delete markers** — with versioning on, deleting "objects" alone isn't enough), then *Delete* the bucket. Delete the EFS file system if you created it.

### Terraform variant

```hcl
provider "aws" { region = "ap-south-1" }

resource "random_id" "suffix" { byte_length = 4 }

resource "aws_s3_bucket" "lab" {
  bucket        = "saa-lab-${random_id.suffix.hex}"
  force_destroy = true # lets `terraform destroy` delete all versions — lab only
}

resource "aws_s3_bucket_public_access_block" "lab" {
  bucket                  = aws_s3_bucket.lab.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_ownership_controls" "lab" {
  bucket = aws_s3_bucket.lab.id
  rule { object_ownership = "BucketOwnerEnforced" }
}

resource "aws_s3_bucket_versioning" "lab" {
  bucket = aws_s3_bucket.lab.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "lab" {
  bucket = aws_s3_bucket.lab.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms" # uses aws/s3 when no key ID is given
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "lab" {
  bucket     = aws_s3_bucket.lab.id
  depends_on = [aws_s3_bucket_versioning.lab]

  rule {
    id     = "lab-tiering"
    status = "Enabled"
    filter { prefix = "logs/" }

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }
    transition {
      days          = 90
      storage_class = "GLACIER_IR"
    }
    transition {
      days          = 180
      storage_class = "DEEP_ARCHIVE"
    }

    noncurrent_version_expiration { noncurrent_days = 30 }
    abort_incomplete_multipart_upload { days_after_initiation = 7 }
  }
}

data "aws_iam_policy_document" "lab" {
  statement {
    sid     = "DenyInsecureTransport"
    effect  = "Deny"
    actions = ["s3:*"]
    resources = [aws_s3_bucket.lab.arn, "${aws_s3_bucket.lab.arn}/*"]
    principals {
      type        = "*"
      identifiers = ["*"]
    }
    condition {
      test     = "Bool"
      variable = "aws:SecureTransport"
      values   = ["false"]
    }
  }
}

resource "aws_s3_bucket_policy" "lab" {
  bucket     = aws_s3_bucket.lab.id
  policy     = data.aws_iam_policy_document.lab.json
  depends_on = [aws_s3_bucket_public_access_block.lab]
}

resource "aws_s3_object" "notes" {
  bucket  = aws_s3_bucket.lab.id
  key     = "notes.txt"
  content = "v1"
}

output "bucket" { value = aws_s3_bucket.lab.bucket }
```

Run `terraform init && terraform apply`, change `content` to `"v2"` and apply again, then `aws s3api list-object-versions --bucket <bucket>` to see both versions. Generate a presigned URL with the CLI as in step 6.

**Optional stretch — CRR:** add a second provider alias in another Region, a destination bucket with versioning, an IAM role trusting `s3.amazonaws.com` (permissions: `s3:GetReplicationConfiguration`, `s3:ListBucket` on the source; `s3:GetObjectVersionForReplication`, `s3:GetObjectVersionAcl`, `s3:GetObjectVersionTagging` on source objects; `s3:ReplicateObject`, `s3:ReplicateDelete`, `s3:ReplicateTags` on destination objects; plus KMS permissions because the source uses SSE-KMS), and an `aws_s3_bucket_replication_configuration`. Upload a new object and watch it appear in the other Region — and notice that `notes.txt`, written before the rule, does **not** replicate.

**Tear down:** `terraform destroy` (`force_destroy = true` removes all versions). Confirm in the console that no `saa-lab-*` buckets remain.

---

## 14. Practice questions

Answer all ten before opening the answers.

**Q1.** A media company stores millions of objects in Amazon S3. Some objects are accessed heavily for a few days and then rarely; others are accessed unpredictably for years. The company wants to minimise storage cost without impacting performance or paying retrieval fees, and without analysing access patterns. Which storage class should be used?
A. S3 Standard-IA
B. S3 Intelligent-Tiering
C. S3 One Zone-IA
D. S3 Glacier Instant Retrieval

**Q2.** A financial services firm must store trade records in S3 for 7 years. Regulations require that no user, including administrators and the AWS account root user, can delete or overwrite the records during that period. What should a solutions architect do?
A. Enable versioning and MFA Delete on the bucket
B. Enable S3 Object Lock in governance mode with a 7-year retention period
C. Enable S3 Object Lock in compliance mode with a 7-year retention period
D. Use a bucket policy that denies `s3:DeleteObject` to all principals

**Q3.** A company runs a content management application on EC2 instances in an Auto Scaling group across three Availability Zones. All instances must read and write the same set of files with POSIX permissions. The storage must scale automatically. Which solution meets these requirements?
A. An EBS gp3 volume with Multi-Attach enabled
B. Amazon EFS with a mount target in each AZ
C. An S3 bucket mounted with an instance store cache
D. Amazon FSx for Windows File Server

**Q4.** A company is migrating a Windows-based application that uses SMB file shares and authenticates users through Microsoft Active Directory. The shares must remain available if an Availability Zone fails. Which solution requires the LEAST operational overhead?
A. Amazon EFS in Regional mode
B. EC2 Windows instances with EBS volumes in two AZs and DFS Replication
C. Amazon FSx for Windows File Server in a Multi-AZ deployment joined to AWS Managed Microsoft AD
D. Amazon FSx for Lustre with a persistent deployment

**Q5.** A research team runs a genomics pipeline on thousands of EC2 instances. Input data is stored in an S3 bucket. The compute jobs need a shared file system with sub-millisecond latency and very high throughput, and results must be written back to S3. The file system is only needed while jobs run. What should the team use?
A. Amazon EFS with Provisioned throughput
B. Amazon FSx for Lustre (scratch) linked to the S3 bucket
C. Amazon FSx for NetApp ONTAP
D. S3 Transfer Acceleration

**Q6.** A company backs up its on-premises servers to physical tapes using a commercial backup application. It wants to stop managing tapes and store backups in AWS for long-term retention at the lowest cost, without changing the backup application. What should a solutions architect recommend?
A. AWS DataSync to S3 Glacier Deep Archive
B. Storage Gateway Tape Gateway, archiving tapes to S3 Glacier Deep Archive
C. Storage Gateway Volume Gateway in stored mode
D. AWS Snowball Edge shipped monthly

**Q7.** A company hosts a static website in an S3 bucket. The security team requires that the bucket is not publicly accessible, that content is served over HTTPS with a custom domain, and that users can only access content through the CDN. What should be done?
A. Enable S3 static website hosting and a bucket policy allowing `s3:GetObject` to everyone
B. Create a CloudFront distribution with Origin Access Control and a bucket policy that allows only that distribution; keep Block Public Access enabled
C. Use S3 Transfer Acceleration with a custom domain
D. Generate presigned URLs for every object and publish them on the website

**Q8.** A company enabled S3 Cross-Region Replication from a bucket in `us-east-1` to a bucket in `eu-west-1`. New objects replicate, but the 40 million objects that existed before the rule was created do not. What is the MOST efficient way to replicate them?
A. Disable and re-enable versioning on the source bucket
B. Use S3 Batch Replication
C. Enable S3 Replication Time Control
D. Write a script that downloads and re-uploads each object

**Q9.** A manufacturing company has on-premises applications that write files to NFS shares. The company wants the files stored in Amazon S3 so that it can analyse them with Amazon Athena and apply lifecycle policies, while on-premises applications keep low-latency access to recently written files. Which solution should be used?
A. Amazon S3 File Gateway
B. Volume Gateway in cached mode
C. AWS DataSync on a daily schedule
D. Amazon FSx for Windows File Server

**Q10.** An application writes thousands of small objects per second to an S3 bucket encrypted with SSE-KMS using a customer managed key. The company sees KMS throttling errors and rising KMS costs. Encryption must still use the customer managed key. What should the solutions architect do?
A. Switch to SSE-S3
B. Enable S3 Bucket Keys for the bucket
C. Switch to client-side encryption with the same key
D. Request a KMS quota increase and add retries

<details>
<summary><strong>Answers</strong></summary>

1. **B** — Intelligent-Tiering moves objects between tiers automatically with no retrieval fees, ideal when patterns vary. Standard-IA and Glacier IR charge retrieval fees; One Zone-IA reduces resilience.
2. **C** — Compliance mode can't be overridden by anyone, root included, until retention ends. Governance mode (B) can be bypassed with a permission; MFA Delete (A) and a bucket policy (D) can be changed by an admin or root.
3. **B** — EFS is a shared, elastic, POSIX file system across AZs. EBS Multi-Attach (A) is single-AZ, io1/io2 only and needs a cluster-aware file system; FSx for Windows (D) is SMB.
4. **C** — FSx for Windows provides managed SMB with AD integration, and Multi-AZ covers an AZ failure. B works but is self-managed; EFS (A) doesn't support SMB/Windows.
5. **B** — FSx for Lustre is built for HPC throughput and latency, links natively to S3 for import/export, and scratch is cheapest for temporary jobs.
6. **B** — Tape Gateway presents a virtual tape library to the existing backup software and archives tapes to Glacier Deep Archive. The other options require changing the backup process.
7. **B** — CloudFront with OAC delivers HTTPS with an ACM certificate and a custom domain, while the bucket stays private. A makes the bucket public; D isn't practical for a website.
8. **B** — Batch Replication replicates existing objects (and failed replications) using the existing replication rule. RTC (C) only adds an SLA for new objects.
9. **A** — S3 File Gateway exposes NFS/SMB shares on-prem with a local cache and stores each file as an S3 object usable by Athena and lifecycle. Volume Gateway (B) stores block volumes, not readable objects.
10. **B** — Bucket Keys cut KMS requests sharply while still using the customer managed key. A and C abandon the requirement; D treats the symptom.

</details>

---

## 15. Flashcards (cover the right column)

| Prompt | Answer |
| --- | --- |
| S3 durability design target | 11 nines (99.999999999%) |
| S3 consistency model | Strong read-after-write for PUT, DELETE and LIST |
| Max size for a single PUT | 5 GB (multipart above that) |
| Request rate per prefix | 3,500 writes / 5,500 reads per second |
| Minimum duration: Standard-IA / One Zone-IA | 30 days |
| Minimum duration: Glacier IR / Glacier Flexible | 90 days |
| Minimum duration: Glacier Deep Archive | 180 days |
| Minimum billable object size for IA classes | 128 KB |
| Glacier Flexible retrieval tiers | Expedited 1–5 min · Standard 3–5 h · Bulk 5–12 h |
| Deep Archive retrieval | Standard within 12 h · Bulk within 48 h |
| Days in Standard before lifecycle to IA | At least 30 |
| What does deleting a versioned object create? | A delete marker |
| Who can enable MFA Delete? | Only the root user (CLI/API) |
| Object Lock mode nobody can override | Compliance |
| Replication prerequisite | Versioning on source and destination |
| Replicate existing objects | S3 Batch Replication |
| Replication with 15-minute SLA | S3 Replication Time Control |
| Default encryption for new S3 objects | SSE-S3 |
| Encryption with per-use audit trail | SSE-KMS |
| Cut SSE-KMS request cost | S3 Bucket Keys |
| Extra permission to read SSE-KMS objects | `kms:Decrypt` |
| Secure CloudFront → S3 access | Origin Access Control (OAC) |
| EFS protocol and port | NFS v4.1, TCP 2049 |
| EFS throughput mode for spiky workloads | Elastic |
| Windows SMB + AD managed file system | FSx for Windows File Server |
| HPC file system linked to S3 | FSx for Lustre |
| Multi-protocol NFS/SMB/iSCSI | FSx for NetApp ONTAP |
| Storage Gateway: files to S3 objects | S3 File Gateway |
| Storage Gateway: all data local, snapshots to AWS | Volume Gateway — stored |
| Storage Gateway: replace tapes | Tape Gateway |
| Online migration of NFS/SMB to EFS/S3/FSx | DataSync |
| Managed SFTP into S3 | AWS Transfer Family |

## 16. Before you close today

- [ ] Read sections 1–12; write the storage-class table (section 3) from memory, including minimum durations
- [ ] Lab done and torn down (bucket emptied of **all versions**, then deleted; no EFS left)
- [ ] 10 questions above + 20 bank questions on S3, EFS, FSx and Storage Gateway
- [ ] Misses logged in `miss-log.md` — especially storage-class, Object Lock mode or gateway-type mix-ups

**Tomorrow (Day 6):** Databases — RDS (Multi-AZ vs read replicas), Aurora (Global Database, Serverless), DynamoDB (capacity modes, DAX, Global Tables, streams), ElastiCache and Redshift.
