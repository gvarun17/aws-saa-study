# Day 7 — Review of Days 1–6 + Practice Exam 1

**Domain weight:** all four domains, in exam proportion — *Design Secure Architectures* (30%), *Design Resilient Architectures* (26%), *Design High-Performing Architectures* (24%), *Design Cost-Optimized Architectures* (20%). Today covers only Days 1–6 material (IAM, EC2/EBS, ELB/Auto Scaling, VPC, storage, databases), which is well over half of what the real exam draws on.
**Time plan (≈ 4.5 h — today runs long on purpose):** Review 60 min · Lab 30 min · Warm-up 15 min · **Practice exam 130 min, timed, no notes** · Score & log misses 45 min.

By the end of today you should be able to:

- Draw the reference three-tier architecture from memory and name which Day 1–6 service sits at each layer and why.
- Answer the four cross-cutting questions for any service from Days 1–6: *What AZ/Region scope is it? How is it made highly available? How is it encrypted (and can you encrypt it after the fact)? How is it backed up and restored?*
- Separate the ten most-confused pairs (Multi-AZ vs read replica, gateway vs interface endpoint, SG vs NACL, SCP vs permissions boundary, ALB vs NLB, EFS vs FSx, DAX vs ElastiCache, GSI vs LSI, Savings Plans vs RIs, Standard-IA vs Intelligent-Tiering).
- Sit a full 65-question timed exam and pace it at ~2 minutes per question.
- Score yourself by domain, log every miss in `miss-log.md`, and know which day to re-read for each weak domain.

> **How to use today.** Do sections 1–6 as review, then the lab and warm-up. Take a break. Then do the practice exam in **one sitting, 130 minutes, no notes, no console**. Only open the answer key after you've answered all 65. Your miss-log entries from today drive Days 12 and 13.

---

## 1. The six-day map: one architecture, every service

Almost every SAA question is a variation on one picture. If you can place a service on this diagram, you can usually eliminate two answers immediately.

```
                       Users (worldwide)
                             │
                 Route 53 (Day 9) / CloudFront (Day 9) ──► S3 static assets via OAC (Day 5)
                             │
┌──────────────────────────── Region ── VPC 10.0.0.0/16 (Day 4) ─────────────────────────────┐
│   IGW                                                                                      │
│    │      ┌──────────── AZ a ────────────┐        ┌──────────── AZ b ────────────┐         │
│  Public   │ ALB node      NAT GW (AZ a)  │        │ ALB node      NAT GW (AZ b)  │  Day 3/4│
│  subnets  │   │  SG: 443 from 0.0.0.0/0  │        │   │                          │         │
│           └───┼──────────────────────────┘        └───┼──────────────────────────┘         │
│  Private  ┌───▼──────────────────────────┐        ┌───▼──────────────────────────┐         │
│  app      │ EC2 in ASG (Day 2/3)         │        │ EC2 in ASG                   │         │
│  subnets  │ role via instance profile    │        │ SG: app port from ALB SG     │  Day 1/2│
│           │ IMDSv2 · gp3 root · EFS mnt  │        │ sessions → ElastiCache/DDB   │  Day 5/6│
│           └───┬──────────────────────────┘        └───┬──────────────────────────┘         │
│  Private  ┌───▼──────────────────────────┐        ┌───▼──────────────────────────┐         │
│  data     │ RDS/Aurora writer            │◄─sync─►│ Multi-AZ standby / Aurora    │  Day 6  │
│  subnets  │ SG: 5432 from app SG         │        │ replica · ElastiCache replica│         │
│           └──────────────────────────────┘        └──────────────────────────────┘         │
│                                                                                            │
│  Gateway endpoints: S3, DynamoDB (route table)   Interface endpoints: SSM, KMS, Secrets…   │
│  Flow Logs → S3 · NACLs on subnets · Transit Gateway / DX / VPN to on-prem (Day 4)         │
└────────────────────────────────────────────────────────────────────────────────────────────┘
   Guardrails above it all: Organizations + SCPs, IAM Identity Center, Access Analyzer (Day 1)
```

Read it bottom-up as a checklist when a question says "design a highly available, secure web application":

| Layer | HA move | Security move | Cost move |
| --- | --- | --- | --- |
| Edge | CloudFront in front, S3 for static | OAC, Block Public Access, HTTPS (`aws:SecureTransport`) | Offload requests from EC2 to cache |
| Load balancer | ALB/NLB spans ≥ 2 AZs automatically | WAF on ALB; TLS with ACM; SG on ALB | One ALB, many target groups (host/path rules) |
| Compute | ASG across ≥ 2 AZs, ELB health checks | Role not keys, IMDSv2, private subnets, Session Manager | Savings Plans for base, Spot for burst, right-size |
| Egress | NAT gateway **per AZ** | No inbound path from internet to private subnets | Gateway endpoints for S3/DynamoDB instead of NAT |
| Data | RDS Multi-AZ / Aurora replicas / DynamoDB (Regional) | SG from app SG only; KMS at rest; Secrets Manager | Serverless/on-demand when spiky; RIs when steady |
| Org | Multi-account isolation | SCPs, Identity Center, Access Analyzer | Consolidated billing shares RI/SP discounts |

---

## 2. Cross-cutting question 1 — what is the blast radius? (scope)

Half of "which design survives X" questions are just scope questions. Know what each resource lives in.

| Scope | Resources (Days 1–6) |
| --- | --- |
| **Global** | IAM, Organizations/SCPs, Identity Center, Route 53, CloudFront; S3 bucket *namespace* |
| **Region** | VPC, S3 bucket (data), ALB/NLB, ASG, EFS (Regional), DynamoDB table, AMIs, EBS snapshots, KMS keys, Aurora cluster storage, Transit Gateway, gateway endpoints |
| **AZ** | Subnet, EC2 instance, EBS volume, NAT gateway, interface endpoint ENI (one per subnet you pick), RDS Single-AZ instance, EFS One Zone, S3 One Zone-IA / Express One Zone, FSx Single-AZ, cluster placement group |
| **Host / hardware** | Instance store (lost on stop, terminate, or host failure) |

**Consequences the exam tests:**

- Anything single-AZ needs a twin in another AZ for HA — two NAT gateways, two subnets per tier, Multi-AZ DB.
- To move an AZ-scoped thing to another AZ you go through a **Regional** artifact: EBS volume → snapshot → new volume; instance → AMI → new instance.
- To move a Region-scoped thing to another Region you **copy** it: AMI copy, snapshot copy, S3 CRR, cross-Region read replica, Aurora Global Database, DynamoDB Global Tables, KMS multi-Region keys.
- An ALB or ASG never spans Regions — cross-Region traffic steering is Route 53 or Global Accelerator (Day 9).

---

## 3. Cross-cutting question 2 — how is it made highly available?

| Service | Default | How you make it HA | Common wrong answer |
| --- | --- | --- | --- |
| EC2 | One instance, one AZ | ASG across ≥ 2 AZs behind an ELB; for one instance: ASG min=max=1 for self-healing | "Bigger instance" or "cluster placement group" |
| EBS | Replicated **within** one AZ | Snapshots (Regional); app-level replication | "Multi-Attach for HA" (same AZ only) |
| ELB | Multi-AZ by design (one node per enabled AZ) | Enable ≥ 2 AZs; cross-zone for even spread | Two load balancers per AZ |
| NAT gateway | Redundant **within** one AZ | One per AZ + per-AZ route tables | One NAT for all AZs |
| VPN / DX | VPN: 2 tunnels; DX: one link | DX at **two locations** with redundant connections; VPN as cheap backup | Two DX connections on one device |
| S3 | ≥ 3 AZs (except One Zone classes) | Already HA; CRR for Region DR | "Replicate to another AZ" |
| EFS | Regional (multi-AZ) | Mount target per AZ | EBS Multi-Attach as shared FS |
| FSx Windows / ONTAP | Single-AZ or Multi-AZ chosen at creation | Pick **Multi-AZ** | EFS (Linux NFS only) |
| RDS | Single-AZ unless chosen | **Multi-AZ** (sync standby) or Multi-AZ DB cluster | Read replica "for HA" |
| Aurora | Storage: 6 copies across 3 AZs | Add an Aurora Replica in another AZ for fast failover; Global Database for Regions | "Enable Multi-AZ" checkbox thinking |
| DynamoDB | Regional, multi-AZ | Already HA; Global Tables for multi-Region active-active | Read replicas (that's RDS) |
| ElastiCache (Valkey/Redis OSS) | Depends on config | Replicas + Multi-AZ automatic failover | Memcached for HA (no replication) |

The pattern: **managed regional services (S3, DynamoDB, EFS, ELB) are HA out of the box; anything you "place" (instances, NAT, Single-AZ DBs, FSx Single-AZ) needs a second copy.**

---

## 4. Cross-cutting question 3 — how is it encrypted, and can you fix it later?

| Service | At rest | Encrypt something that's already unencrypted | In transit |
| --- | --- | --- | --- |
| EBS | KMS; **encryption by default** per Region | Snapshot → **copy with encryption** → new volume (or launch from encrypted AMI copy) | Encrypted between instance and volume when the volume is encrypted |
| AMI sharing | Snapshots encrypted with KMS | Re-encrypt with a **customer managed key**, share key + AMI | — |
| S3 | SSE-S3 applied by default to new objects; SSE-KMS, DSSE-KMS, SSE-C options | Re-copy objects (in place or via **S3 Batch Operations**) with the new encryption | Enforce HTTPS with `aws:SecureTransport` |
| RDS | KMS, chosen **at creation** | Snapshot → **copy with encryption** → restore new instance | SSL/TLS; enforce via parameter group |
| Aurora | KMS, chosen at creation | Restore from snapshot with a KMS key (new cluster) | TLS |
| DynamoDB | **Always** encrypted; AWS owned, AWS managed, or customer managed key | Nothing to fix — you can switch the key type | HTTPS endpoints |
| EFS | KMS, chosen at creation | New encrypted file system + copy (DataSync) | TLS via the EFS mount helper |
| ElastiCache | At-rest and in-transit options set on the cluster | Typically a new cluster from a backup (check current docs for in-place options per engine) | TLS option |

Two things to memorise:

1. **"Encrypt existing" almost always means snapshot/copy → new resource.** The only "flip a switch" answers are EBS encryption-by-default (future volumes only) and S3 default encryption (future objects only).
2. **SSE-KMS access needs KMS permissions.** If a role has `s3:GetObject` but still gets AccessDenied on SSE-KMS objects, add `kms:Decrypt` (key policy and IAM). AWS managed keys (`aws/ebs`, `aws/s3`) **cannot be shared cross-account** — use a customer managed key.

---

## 5. Cross-cutting question 4 — how is it backed up and restored?

| Service | Native backup | Retention ceiling (check current docs) | Restore lands… |
| --- | --- | --- | --- |
| EBS | Incremental snapshots; **Data Lifecycle Manager**; Snapshots Archive; Recycle Bin | Your policy | As a **new volume** (any AZ in the Region); Fast Snapshot Restore for full performance immediately |
| EC2 | AMIs (point at snapshots) | Your policy | New instance |
| S3 | **Versioning**, replication, Object Lock; AWS Backup | Your policy | Previous version is restored in place; replicas in the target bucket |
| RDS | Automated backups (PITR) + manual snapshots | Automated: up to **35 days**; manual: until you delete | **New DB instance** — new endpoint |
| Aurora | Continuous backup + snapshots; **Backtrack** (Aurora MySQL); cloning | Automated: up to 35 days | New cluster, except Backtrack which rewinds **in place** |
| DynamoDB | **PITR** (up to 35 days) + on-demand backups | Your policy for on-demand | **New table** |
| EFS / FSx | AWS Backup / FSx backups | Your policy | New file system (or file-level restore with AWS Backup for EFS) |
| Everything | **AWS Backup**: central plans, cross-Region and cross-account copy, Vault Lock | Plan-defined (years) | Per service |

**Rule:** "retain beyond 35 days" → manual snapshots or **AWS Backup**. "Central, policy-driven backups across services and accounts" → **AWS Backup**. Restores create new resources, so the app must be repointed (except S3 versions and Aurora Backtrack).

---

## 6. The ten most-confused pairs

These ten pairs account for a large share of misses on first practice exams. For each, the *one discriminator* that decides it:

| Pair | Discriminator | Pick A when… | Pick B when… |
| --- | --- | --- | --- |
| **RDS Multi-AZ** vs **read replica** | Sync + automatic failover vs async + readable | "AZ failure", "automatic failover", "no data loss" | "offload reads", "reporting", "cross-Region DR copy" |
| **Gateway** vs **interface** endpoint | Route-table target (S3/DynamoDB, free) vs ENI with private IP (most services, billed) | Traffic from **inside** the VPC to S3/DynamoDB, cut NAT cost | Any other service, or S3 from **on-prem / peered VPC** |
| **Security group** vs **NACL** | Stateful, allow-only, instance level vs stateless, allow+deny, subnet level | "Allow from the ALB / app tier" (SG referencing) | "Block this IP range", "deny" |
| **SCP** vs **permissions boundary** | Caps a whole account (incl. root) vs caps one user/role | "All accounts / OU / even root" | "Developers can create roles but not exceed X" |
| **ALB** vs **NLB** | Layer 7 (paths, hosts, headers, WAF, auth) vs layer 4 (TCP/UDP, static IP, extreme scale) | HTTP routing, WAF, OIDC auth, Lambda targets | UDP, static/Elastic IPs, PrivateLink, preserve client IP |
| **EFS** vs **FSx** | Linux NFS, elastic vs purpose-built file systems | Linux, shared across AZs, Lambda/Fargate | Windows/SMB/AD (FSx Windows), HPC + S3 (Lustre), NetApp/multi-protocol (ONTAP) |
| **DAX** vs **ElastiCache** | DynamoDB-only, API-compatible vs general cache needing app logic | DynamoDB + microseconds + minimal code change | RDS/Aurora or any other source; sessions; leaderboards |
| **GSI** vs **LSI** | Any time, different partition key, eventually consistent vs creation-only, same partition key, can be strongly consistent | Existing table, query by another attribute | New table, alternate sort order within the same partition, need strong consistency |
| **Savings Plans** vs **RIs** | Spend commitment across compute types vs instance-attribute commitment | Compute SP: flexibility across family/Region/Fargate/Lambda | Standard RI / EC2 Instance SP: fixed family, maximum discount; zonal RI for capacity |
| **Standard-IA** vs **Intelligent-Tiering** | Known infrequent pattern vs unknown/changing pattern | "Accessed monthly, known", fine with retrieval fee | "Unpredictable", "no retrieval fees", "no ops" |

Bonus pairs worth one more look: **spread vs partition** placement group (individual instances vs groups of instances on separate racks); **instance store vs EBS** (ephemeral/fastest vs persistent); **Object Lock governance vs compliance** (can vs can't be bypassed); **Aurora Global Database vs DynamoDB Global Tables** (relational single-writer vs key-value multi-writer); **S3 File Gateway vs DataSync** (ongoing hybrid access vs migration/sync).

### Qualifier → answer reflexes

| Qualifier in the question | Reflex |
| --- | --- |
| *LEAST operational overhead* | Managed / serverless over self-managed EC2; built-in feature over custom Lambda |
| *MOST cost-effective* | Smallest thing that meets every requirement; Spot/SP/lifecycle/gateway endpoint |
| *Highly available* | ≥ 2 AZs |
| *Fault tolerant / no data loss* | Synchronous replication (Multi-AZ), durable storage |
| *Minimal changes / lift-and-shift* | Same engine on RDS, same protocol (FSx Windows for SMB), rehost on EC2 |
| *Most secure* | Private subnets + endpoints, roles, KMS CMK, least privilege, deny conditions |
| *Real-time / lowest latency* | Caching (DAX/ElastiCache), instance store, io2, cluster placement, Global Accelerator |

---

## 7. Exam decision table — the cross-day edition

Each day's lesson has its own table; this one is the "two services from different days must work together" version.

| If the question says… | Pick |
| --- | --- |
| EC2 in a private subnet with no NAT must read S3 | **Role** (instance profile) + **S3 gateway endpoint** + route table association |
| Bucket readable only from my VPC | Bucket policy **Deny** unless `aws:SourceVpce` = my endpoint (or `aws:SourceVpc`) |
| Private instances need shell access, no internet, no bastion | **Session Manager**: role with `AmazonSSMManagedInstanceCore` + interface endpoints `ssm`, `ssmmessages`, `ec2messages` |
| App tier must accept traffic only from the load balancer | App SG inbound **from the ALB's SG** |
| DB must accept traffic only from the app tier | DB SG inbound on the DB port **from the app SG** |
| EC2 connects to RDS without a stored password | **IAM database authentication** using the instance role |
| Credentials must rotate automatically | **Secrets Manager** with rotation |
| Encrypt an existing unencrypted EBS volume or RDS DB | **Snapshot → encrypted copy → new resource** |
| Share an encrypted AMI or snapshot with another account | **Customer managed KMS key** shared via key policy + share the AMI/snapshot |
| Stateless web tier behind ALB, users lose sessions on scale-in | Sessions in **ElastiCache** or **DynamoDB** (with TTL) |
| Shared files for an ASG spread over three AZs (Linux) | **EFS** |
| Expose an internal service to a partner VPC with an overlapping CIDR | **NLB + PrivateLink** endpoint service |
| On-prem must reach S3 privately over DX/VPN | **S3 interface endpoint** (gateway endpoints don't work from outside the VPC) |
| Highly available web app, relational data, minimal management | **ALB + ASG (≥ 2 AZs) + RDS Multi-AZ** (or Aurora) |
| Regional DR for an EC2 app at low cost, RTO hours | Copy **AMIs/snapshots** to the DR Region (DLM or AWS Backup); launch on demand |
| Cross-Region DR for the database, RPO seconds | **Aurora Global Database** (relational) / **DynamoDB Global Tables** (key-value) |
| Cut NAT gateway cost from S3/DynamoDB traffic | **Gateway endpoints** |
| Cut compute cost for a steady base + interruptible burst | **Savings Plan** for the base, **Spot** in a mixed instances policy for the rest |
| Enforce a rule in every account, including root | **SCP** |

---

## 8. Lab (30 min, free tier) — private S3 access through a gateway endpoint

Goal: wire together four days in one small build — an **IAM role** (Day 1) on an **EC2 instance** with IMDSv2 (Day 2), a **gateway VPC endpoint** and route (Day 4), and an **S3 bucket policy** keyed on `aws:SourceVpce` (Day 5) — and watch explicit deny win (Day 1) when the request takes a different path.

Gateway endpoints are free. Use a free-tier-eligible instance type for your account (check current free-tier terms) and stay in one Region; the steps use `ap-south-1`.

```
  CloudShell / your laptop ──► S3 public endpoint ──► bucket policy: DENY (no vpce) ✗
  EC2 (default VPC) ──► route pl-xxxx → vpce-xxxx ──► bucket policy: vpce matches  ✓
```

### Console steps

1. **S3:** create bucket `saa-day7-<random>` (Block Public Access on, default SSE-S3). Upload a small `hello.txt`.
2. **VPC → Endpoints → Create endpoint:** service `com.amazonaws.ap-south-1.s3`, type **Gateway**, VPC = default VPC, select the default VPC's **main route table**. Note the endpoint ID (`vpce-…`). Open the route table and find the new route to a **prefix list** (`pl-…`) targeting the endpoint.
3. **IAM:** create role `saa-day7-ec2` for EC2 with `AmazonSSMManagedInstanceCore` plus an inline policy allowing `s3:GetObject` on `arn:aws:s3:::saa-day7-<random>/*` and `s3:ListBucket` on the bucket ARN.
4. **EC2:** launch an Amazon Linux 2023 instance in the default VPC with that role, **IMDSv2 required**, a security group with **no inbound rules**.
5. **Bucket policy:** add the deny below (replace bucket name and endpoint ID). It only denies `GetObject`, so you can still manage and delete the bucket afterwards.
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [{
       "Sid": "DenyGetUnlessViaMyEndpoint",
       "Effect": "Deny",
       "Principal": "*",
       "Action": "s3:GetObject",
       "Resource": "arn:aws:s3:::saa-day7-<random>/*",
       "Condition": { "StringNotEquals": { "aws:SourceVpce": "vpce-0123456789abcdef0" } }
     }]
   }
   ```
6. **Test from the instance** (connect with Session Manager — no SSH, no open port):
   ```bash
   aws s3 cp s3://saa-day7-<random>/hello.txt - --region ap-south-1   # works: path is via the endpoint
   ```
7. **Test from CloudShell** (or your laptop) as your admin user:
   ```bash
   aws s3 cp s3://saa-day7-<random>/hello.txt -                       # AccessDenied: explicit deny wins over your admin allow
   ```
8. **Break it on purpose:** in the endpoint, remove the main route table association. Retry step 6 → **AccessDenied**: traffic now leaves via the internet gateway with no `aws:SourceVpce`. Re-associate the route table and it works again. That is the whole "gateway endpoint + bucket policy" exam pattern.

### Terraform variant

```hcl
terraform {
  required_providers {
    aws    = { source = "hashicorp/aws", version = "~> 5.0" }
    random = { source = "hashicorp/random", version = "~> 3.0" }
  }
}

provider "aws" { region = "ap-south-1" }

data "aws_vpc" "default" { default = true }

data "aws_route_table" "main" {
  vpc_id = data.aws_vpc.default.id
  filter {
    name   = "association.main"
    values = ["true"]
  }
}

data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}

data "aws_ssm_parameter" "al2023" {
  name = "/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64"
}

# ---------- Day 4: gateway endpoint ----------
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = data.aws_vpc.default.id
  service_name      = "com.amazonaws.ap-south-1.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = [data.aws_route_table.main.id]
}

# ---------- Day 5: bucket + deny-unless-endpoint policy ----------
resource "random_id" "suffix" { byte_length = 4 }

resource "aws_s3_bucket" "lab" {
  bucket        = "saa-day7-${random_id.suffix.hex}"
  force_destroy = true
}

resource "aws_s3_object" "hello" {
  bucket  = aws_s3_bucket.lab.id
  key     = "hello.txt"
  content = "reached via the gateway endpoint\n"
}

data "aws_iam_policy_document" "bucket" {
  statement {
    sid       = "DenyGetUnlessViaMyEndpoint"
    effect    = "Deny"
    actions   = ["s3:GetObject"]
    resources = ["${aws_s3_bucket.lab.arn}/*"]
    principals {
      type        = "*"
      identifiers = ["*"]
    }
    condition {
      test     = "StringNotEquals"
      variable = "aws:SourceVpce"
      values   = [aws_vpc_endpoint.s3.id]
    }
  }
}

resource "aws_s3_bucket_policy" "lab" {
  bucket = aws_s3_bucket.lab.id
  policy = data.aws_iam_policy_document.bucket.json
}

# ---------- Day 1: role + instance profile ----------
data "aws_iam_policy_document" "trust" {
  statement {
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["ec2.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "ec2" {
  name               = "saa-day7-ec2"
  assume_role_policy = data.aws_iam_policy_document.trust.json
}

resource "aws_iam_role_policy_attachment" "ssm" {
  role       = aws_iam_role.ec2.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

resource "aws_iam_role_policy" "s3_read" {
  role = aws_iam_role.ec2.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      { Effect = "Allow", Action = "s3:ListBucket", Resource = aws_s3_bucket.lab.arn },
      { Effect = "Allow", Action = "s3:GetObject",  Resource = "${aws_s3_bucket.lab.arn}/*" }
    ]
  })
}

resource "aws_iam_instance_profile" "ec2" {
  name = "saa-day7-ec2"
  role = aws_iam_role.ec2.name
}

# ---------- Day 2/4: instance, no inbound ----------
resource "aws_security_group" "no_inbound" {
  name   = "saa-day7-no-inbound"
  vpc_id = data.aws_vpc.default.id
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "lab" {
  ami                    = data.aws_ssm_parameter.al2023.value
  instance_type          = "t3.micro" # use a free-tier-eligible type for your account
  subnet_id              = data.aws_subnets.default.ids[0]
  vpc_security_group_ids = [aws_security_group.no_inbound.id]
  iam_instance_profile   = aws_iam_instance_profile.ec2.name
  metadata_options {
    http_tokens = "required" # IMDSv2
  }
  tags = { Name = "saa-day7" }
}

output "bucket"      { value = aws_s3_bucket.lab.id }
output "endpoint_id" { value = aws_vpc_endpoint.s3.id }
output "instance_id" { value = aws_instance.lab.id }
```

`terraform init && terraform apply`, then run steps 6–8 above (`aws ssm start-session --target <instance_id>` works too if you have the Session Manager plugin).

### Teardown

- Terraform: `terraform destroy`.
- Console: terminate the instance → delete the bucket policy → empty and delete the bucket → delete the gateway endpoint (its route disappears with it) → delete the security group → delete role `saa-day7-ec2` and its instance profile.
- Check the route table no longer has a `pl-…` route and no instances are running.

---

## 9. Warm-up: ten integration questions (15 min, untimed)

Every question below needs two days' knowledge at once — the same muscle the exam uses. Do these *before* the timed exam; they're not counted in your score.

**W1.** An application on EC2 instances in a private subnet reads objects from an S3 bucket in the same Region using an instance role. To save money, the team deleted the subnet's NAT gateway route, and the application now times out reaching S3. What is the MOST cost-effective fix?
A. Add access keys for an IAM user to the instances so they can sign requests directly
B. Create an S3 interface endpoint in every AZ and disable private DNS
C. Move the instances to a public subnet and assign Elastic IP addresses
D. Create an S3 gateway VPC endpoint and associate it with the private subnet's route table

**W2.** A stateless web tier runs in an Auto Scaling group behind an ALB. To cut cost, the team wants most capacity on Spot while keeping the site available when Spot capacity is reclaimed. Which configuration is BEST?
A. Spot Instances in a cluster placement group
B. A launch template with a single Spot instance type in one AZ
C. A mixed instances policy with an On-Demand base capacity, Spot above it, several instance types across multiple AZs, the price-capacity-optimized allocation strategy and Capacity Rebalancing
D. All instances on Spot with a maximum price set to the On-Demand price

**W3.** Users of an e-commerce site behind an ALB lose their shopping carts whenever the Auto Scaling group scales in. Carts are stored in instance memory. The company wants carts to expire automatically after 24 hours. What should the architect do?
A. Enable ALB sticky sessions with a 24-hour duration
B. Store carts in a DynamoDB table keyed by session ID with a TTL attribute set to 24 hours
C. Store carts on each instance's EBS root volume
D. Disable scale-in on the Auto Scaling group

**W4.** An application tier in private subnets connects to an RDS for PostgreSQL instance in separate private data subnets. Auto Scaling changes the application instances' IP addresses constantly. How should database access be restricted?
A. Allow inbound TCP 5432 from 0.0.0.0/0 and rely on the private subnet
B. Use a bucket-style resource policy on the RDS instance
C. Add NACL rules for each application instance IP
D. Allow inbound TCP 5432 on the database security group with the application tier's security group as the source

**W5.** Partner Account B uploads files into a bucket owned by Account A using a bucket policy. Account A's users cannot read some uploaded objects because Account B owns them. What should Account A configure?
A. Set S3 Object Ownership to Bucket owner enforced on the bucket
B. Require Account B to use SSE-C
C. Enable S3 Versioning
D. Enable Requester Pays

**W6.** Web servers in an Auto Scaling group across three AZs each write user-uploaded images to a local EBS volume, so images uploaded to one server are missing on the others. What is the simplest fix?
A. Mount a shared Amazon EFS file system on all instances
B. Enable Multi-Attach on the gp3 volumes
C. Take hourly EBS snapshots and restore them on every instance
D. Use instance store volumes and sync them nightly

**W7.** An application on EC2 must connect to an Amazon RDS for MySQL database without storing any database password on the instance or in code. Which approach meets this requirement?
A. Store the password in the instance user data
B. Use a database user with an empty password restricted by security group
C. Enable IAM database authentication and let the instance role generate authentication tokens
D. Store the password in an S3 object encrypted with SSE-S3

**W8.** A company must encrypt an existing unencrypted EBS data volume and an existing unencrypted RDS instance. What do the two procedures have in common?
A. Both require creating a snapshot, making an encrypted copy, and creating a new resource from that copy
B. Both require exporting data to S3 and re-importing it
C. Both can be encrypted in place by modifying the resource
D. Both are encrypted automatically once EBS encryption by default is enabled

**W9.** A SaaS provider must offer its API, running on EC2 behind a load balancer, to customers' VPCs privately. Several customers use the same CIDR range as the provider. Which design works?
A. A Transit Gateway shared with every customer
B. Put the service behind a Network Load Balancer and publish it as a PrivateLink endpoint service; customers create interface endpoints
C. VPC peering with each customer
D. A public ALB restricted with security groups to customer IPs

**W10.** A company is migrating 80 TB from an on-premises NFS server to Amazon EFS over an existing Direct Connect link and wants to keep the two in sync until cutover with minimal scripting. What should it use?
A. AWS Snowball Edge shipped weekly
B. rsync from a cron job on an EC2 instance
C. AWS DataSync with an agent on premises, scheduled tasks to EFS
D. S3 File Gateway

<details>
<summary><strong>Warm-up answers</strong></summary>

1. **D** · *Secure · Day 1+4+5* — A gateway endpoint adds a prefix-list route to S3 at no charge. Credentials weren't the problem (the role is fine); an interface endpoint works but costs money; public subnets weaken security.
2. **C** · *Cost-optimized · Day 2+3* — Diversifying types and AZs and keeping an On-Demand base protects availability; Capacity Rebalancing replaces at-risk Spot instances early. One type/AZ or a cluster group concentrates interruption risk.
3. **B** · *Resilient · Day 3+6* — Externalising state makes instances disposable; DynamoDB TTL handles expiry for free. Stickiness still loses carts when the instance terminates; EBS is per instance.
4. **D** · *Secure · Day 4+6* — Security group referencing tracks membership, not IPs. NACL per-IP rules break with scaling; RDS has no resource-based access policy for network access.
5. **A** · *Secure · Day 1+5* — Bucket owner enforced disables ACLs and makes the bucket owner own every object regardless of uploader.
6. **A** · *Resilient · Day 2+5* — EFS is a Regional shared NFS file system. Multi-Attach is io1/io2 only, single-AZ, and needs a cluster-aware file system.
7. **C** · *Secure · Day 1+6* — IAM DB auth issues short-lived tokens from the instance's role credentials — no stored password. (Secrets Manager is the answer when a password must exist and rotate.)
8. **A** · *Secure · Day 2+6* — Encryption at rest is fixed at creation for EBS volumes and RDS instances. EBS encryption by default only affects new volumes and never RDS.
9. **B** · *High-performing · Day 3+4* — PrivateLink needs no routing between VPCs, so overlapping CIDRs don't matter, and endpoint services are fronted by an NLB. Peering and TGW require non-overlapping CIDRs.
10. **C** · *Resilient · Day 4+5* — DataSync is the managed, scheduled, verified online transfer service for NFS/SMB → EFS/FSx/S3. File Gateway is for ongoing hybrid access to S3, not EFS migration.

</details>

---

## 10. Practice Exam 1 — 65 questions, 130 minutes

**Rules:** one sitting, timer on, no notes, no console, no searching. ~2 minutes per question; if you pass 3 minutes, pick your best guess, note the number to revisit, and move on. Questions marked *(Choose two.)* need both correct to score. Write your answers on paper (1–65) before opening the key.

Domain mix matches the real exam's weighting: 20 Secure · 17 Resilient · 15 High-performing · 13 Cost-optimized — shuffled, just like the real thing. All content is from Days 1–6.

**Pacing checkpoints:** Q22 by ~0:45 · Q44 by ~1:30 · Q65 by ~2:00 · last 10 minutes for flagged questions.

**Q1.** Finance suspects that many of the company's EC2 instances are over-provisioned and wants recommendations based on actual utilisation history. Which service should the architect use?
A. Cost allocation tags
B. AWS CloudTrail
C. AWS Config
D. AWS Compute Optimizer

**Q2.** A machine learning team repeatedly reads a 200 TB training dataset stored in S3 from a fleet of GPU instances. It needs sub-millisecond latency and hundreds of GB/s of aggregate throughput. Which storage should the architect add?
A. Amazon FSx for Lustre linked to the S3 bucket
B. Amazon EFS
C. Amazon S3 File Gateway
D. Amazon FSx for Windows File Server

**Q3.** An application on EC2 connects to Amazon RDS for PostgreSQL using a password kept in a configuration file. Security requires the password to rotate automatically every 30 days with minimal custom code. What should the architect do?
A. Store the credentials in AWS Secrets Manager with managed rotation enabled and have the application retrieve them at runtime
B. Store the password as a SecureString parameter in Systems Manager Parameter Store
C. Rebuild the AMI with a new password every 30 days
D. Schedule a custom Lambda function to edit the configuration file on each instance

**Q4.** A Windows-based line-of-business application needs shared file storage accessed over SMB, integrated with Active Directory, and available if an Availability Zone fails. Which service should the architect choose?
A. Amazon EFS with Regional storage
B. Amazon EBS io2 volumes with Multi-Attach
C. Amazon FSx for Lustre
D. Amazon FSx for Windows File Server with a Multi-AZ deployment

**Q5.** A stateless web fleet in an Auto Scaling group always needs at least 4 instances. Capacity above that handles spikes and can tolerate interruptions. Which configuration is MOST cost-effective?
A. All instances as Dedicated Instances
B. All instances On-Demand, with Reserved Instances covering the peak
C. A mixed instances policy with an On-Demand base capacity of 4 and Spot Instances above that, across multiple instance types
D. All instances as Spot Instances

**Q6.** A monitoring Lambda function in Account B must call ec2:DescribeInstances in Account A. What is the MOST secure way to grant access?
A. Add a resource-based policy to the EC2 instances in Account A granting Account B access
B. Create an IAM role in Account A with the required permissions and a trust policy that trusts the Lambda execution role in Account B; the function calls sts:AssumeRole
C. Create an IAM user in Account A and store its access keys in the function's environment variables
D. Share the EC2 instances with Account B using AWS RAM

**Q7.** Thousands of concurrent AWS Lambda invocations each open a connection to an Aurora PostgreSQL cluster, causing 'too many connections' errors. What should the architect do with the LEAST code change?
A. Increase the Lambda functions' memory
B. Convert the cluster to Multi-AZ
C. Put Amazon RDS Proxy between the functions and the cluster
D. Add an ElastiCache cluster to store connections

**Q8.** An S3 bill keeps growing from storage that doesn't appear in object listings. The team finds many failed multipart uploads from unreliable clients. What should the architect do?
A. Add a lifecycle rule with AbortIncompleteMultipartUpload after a few days
B. Enable S3 Versioning
C. Move the bucket to S3 Intelligent-Tiering
D. Enable S3 Object Lock

**Q9.** Each new instance in an Auto Scaling group runs a 15-minute initialisation before it can serve traffic. Load is unpredictable, so scale-out is too slow. What should the architect do?
A. Configure a warm pool of pre-initialised instances for the Auto Scaling group
B. Add scheduled scaling actions
C. Replace the ALB with an NLB
D. Increase the health check grace period

**Q10.** A production Amazon RDS for MySQL DB instance was created unencrypted. Compliance now requires encryption at rest. What should the architect do?
A. Snapshot the instance, copy the snapshot with encryption using a KMS key, and restore a new DB instance from the encrypted copy
B. Modify the DB instance and enable encryption at rest
C. Enable EBS encryption by default in the Region
D. Create an encrypted read replica of the instance and promote it

**Q11.** A global retailer runs a MySQL-compatible relational database in us-east-1. It needs a disaster recovery copy in eu-west-1 with an RPO of about 1 second and an RTO of about 1 minute, and wants European users to read locally. What should it use?
A. Daily automated snapshot copies to eu-west-1
B. Amazon Aurora Global Database with a secondary cluster in eu-west-1
C. An RDS for MySQL cross-Region read replica in eu-west-1
D. Amazon DynamoDB global tables

**Q12.** A company has 40 VPCs across several accounts plus a VPN to its data center. Full-mesh VPC peering has become unmanageable, and the company needs central routing with network segmentation. What should the architect use?
A. A Direct Connect gateway
B. AWS Transit Gateway shared through AWS RAM, with separate Transit Gateway route tables per segment
C. More VPC peering connections with route summaries
D. AWS PrivateLink between every pair of VPCs

**Q13.** A DynamoDB Orders table uses CustomerId as the partition key and OrderDate as the sort key. A new feature must efficiently query all orders for a given ProductId. The table already holds production data. What should the architect do?
A. Run a Scan with a filter expression on ProductId
B. Add a local secondary index with ProductId as the sort key
C. Add a DAX cluster
D. Add a global secondary index with ProductId as the partition key

**Q14.** Private subnets in AZ-a and AZ-b both send 0.0.0.0/0 to a single NAT gateway in AZ-a. During an AZ-a outage, instances in AZ-b lost internet access. How should the architect fix this?
A. Add a second default route to the internet gateway in the private route tables
B. Create a NAT gateway in a public subnet in each AZ and route each private subnet to the NAT gateway in its own AZ
C. Attach a second Elastic IP address to the existing NAT gateway
D. Replace the NAT gateway with a NAT instance

**Q15.** Users of a document management application sometimes overwrite or delete files stored in S3 by mistake. The company must be able to recover earlier versions while controlling storage growth. What should the architect do?
A. Enable S3 Versioning and add a lifecycle rule that expires noncurrent versions after a retention period
B. Enable Cross-Region Replication to a second bucket
C. Move all objects to S3 Intelligent-Tiering
D. Enable Object Lock in compliance mode

**Q16.** Development and test teams use MySQL-compatible databases for a few irregular hours each day. The company wants to minimise cost and capacity management. What should the architect recommend?
A. Amazon Aurora Serverless v2
B. Amazon RDS for MySQL on large instances with Multi-AZ
C. Amazon Redshift Serverless
D. MySQL self-managed on EC2 Spot Instances

**Q17.** A data lake in S3 has unpredictable access patterns: some objects are hot for weeks, then never touched again. The company wants the lowest storage cost without retrieval fees or manual analysis. Which storage class should it use?
A. S3 Standard-IA
B. S3 Intelligent-Tiering
C. A lifecycle rule to S3 Glacier Flexible Retrieval after 30 days
D. S3 One Zone-IA

**Q18.** Objects in a bucket are encrypted with SSE-KMS using a customer managed key. A role with s3:GetObject on the bucket receives AccessDenied when downloading objects. What should the architect do?
A. Allow the role kms:Decrypt on the KMS key (in the key policy, or in IAM where the key policy permits IAM policies)
B. Disable S3 Block Public Access on the bucket
C. Add s3:GetObjectAcl to the role's policy
D. Enable S3 Bucket Keys on the bucket

**Q19.** Company policy states that every new EBS volume created in an account must be encrypted, regardless of who or what creates it. Which solution requires the LEAST operational effort?
A. Use instance store volumes instead of EBS
B. A Lambda function that encrypts volumes after they are created
C. An AWS Config rule that reports unencrypted volumes
D. Enable EBS encryption by default in each Region the account uses

**Q20.** On-premises servers connected through a Direct Connect private virtual interface must upload data to Amazon S3 without traversing the public internet. What should the architect configure?
A. A NAT gateway in the VPC
B. An S3 gateway VPC endpoint and on-premises routes pointing to it
C. An S3 interface VPC endpoint, addressed from on premises by its endpoint-specific DNS name
D. S3 Transfer Acceleration

**Q21.** An application behind an ALB runs in an Auto Scaling group. Some instances hang and return HTTP 500 errors, the ALB marks them unhealthy, but the Auto Scaling group never replaces them. What should the architect change?
A. Enable cross-zone load balancing
B. Increase the group's desired capacity
C. Set the Auto Scaling group's health check type to ELB, with an appropriate health check grace period
D. Replace the ALB with an NLB

**Q22.** A company must share an AMI with a partner account. The AMI's EBS snapshots are encrypted with the AWS managed key aws/ebs. What must the architect do?
A. Make the AMI public
B. Copy the AMI, re-encrypting it with a customer managed KMS key; grant the partner account use of that key in its key policy; share the copied AMI
C. Share the AMI directly; AWS shares the aws/ebs key automatically
D. Edit the aws/ebs key policy to allow the partner account

**Q23.** A company uses AWS Organizations. No one in any member account — including the account root user — may stop or delete AWS CloudTrail trails. What is the MOST effective control?
A. An SCP attached to the organization root that denies cloudtrail:StopLogging and cloudtrail:DeleteTrail
B. An AWS Config rule that re-enables any stopped trail
C. A permissions boundary on every IAM role
D. An IAM policy in each account denying those actions to all users and roles

**Q24.** Three EC2 instances run a quorum-based coordination service. To reduce the risk of simultaneous failure, each instance must run on distinct underlying hardware. Which option meets this requirement?
A. Launch the instances on a single Dedicated Host
B. Launch the instances in a spread placement group
C. Launch the instances in the same subnet with enhanced networking
D. Launch the instances in a cluster placement group

**Q25.** A company connects its data center to a VPC with AWS Direct Connect. A new policy requires all traffic between them to be encrypted in transit. The company wants to keep using the existing connection. What should the architect do?
A. Replace Direct Connect with AWS Client VPN for all servers
B. Nothing — Direct Connect traffic is encrypted by default
C. Enable encryption on the VPC's route tables
D. Run an AWS Site-to-Site VPN (IPsec) over the Direct Connect connection

**Q26.** An RDS for MySQL primary runs in one AZ with a read replica in another AZ used for reporting. The company now requires automatic failover with no data loss if the primary's AZ fails. What should the architect do?
A. Configure the read replica to promote itself automatically
B. Add two more read replicas in other AZs
C. Take snapshots every hour and copy them to another AZ
D. Enable Multi-AZ on the primary DB instance

**Q27.** A mobile game stores player data in a DynamoDB table whose partition key is the player's identity ID. Players get temporary AWS credentials through Amazon Cognito. Each player must be able to read and write only their own items. What should the architect do?
A. Encrypt each player's items with a separate KMS key
B. Create a global secondary index for each player
C. Use an IAM policy with a dynamodb:LeadingKeys condition that matches the caller's Cognito identity ID
D. Create a separate table for each player

**Q28.** An Auto Scaling group terminates instances during scale-in before the application can upload its local log files to S3, so logs are lost. How should the architect fix this?
A. Add a lifecycle hook for the terminating transition; a script uploads the logs and then completes the lifecycle action
B. Increase the scaling cooldown period
C. Store logs on instance store volumes
D. Enable termination protection on all instances

**Q29.** An Auto Scaling group runs a web application behind an ALB. Users are logged out at random, and it correlates with scale-in events. Session data is stored in instance memory. What is the MOST resilient fix?
A. Disable scale-in for the Auto Scaling group
B. Move to larger instance types
C. Store session state in Amazon ElastiCache (Valkey or Redis OSS) or DynamoDB so the instances are stateless
D. Enable sticky sessions with a longer cookie duration

**Q30.** A static website in S3 is served worldwide through Amazon CloudFront. The bucket must not be publicly accessible, and users must not bypass CloudFront. What should the architect do?
A. Use the S3 website endpoint as the origin with a public-read bucket policy
B. Configure CloudFront Origin Access Control and a bucket policy allowing only that distribution; keep Block Public Access enabled
C. Generate a presigned URL for every object
D. Allow the bucket to be read from CloudFront edge IP ranges using aws:SourceIp

**Q31.** Financial records stored in S3 must be retained for 7 years. No user, including the account root user, may delete or overwrite them during that time. What should the architect configure?
A. S3 Versioning with MFA Delete
B. S3 Object Lock in compliance mode with a 7-year retention period
C. A lifecycle rule transitioning objects to S3 Glacier Deep Archive
D. S3 Object Lock in governance mode with a 7-year retention period

**Q32.** A shopping-cart service stores key-value data and serves customers in North America, Europe and Asia. Each Region must accept writes locally and continue operating if any one Region fails. Which database meets these requirements?
A. Amazon RDS Multi-AZ with cross-Region read replicas
B. Amazon Aurora Global Database
C. Amazon DynamoDB global tables
D. Amazon ElastiCache clusters in each Region

**Q33.** A company has 25 AWS accounts and uses Okta for employee identities. It wants centrally managed single sign-on to all accounts, with permissions based on job function and the LEAST operational overhead. What should it use?
A. An Amazon Cognito user pool federated with Okta
B. A SAML identity provider and roles configured separately in each account
C. IAM Identity Center with Okta as the external identity provider and permission sets assigned to groups
D. IAM users created in each account and synced from Okta

**Q34.** A web application runs on a single EC2 instance that also hosts its MySQL database. The company needs the application to survive the loss of an Availability Zone with the least management effort. Which solution meets this requirement?
A. Run two instances in a cluster placement group
B. Run the web tier in an Auto Scaling group across two AZs behind an ALB, and move the database to Amazon RDS for MySQL with Multi-AZ
C. Move to a larger instance and take nightly EBS snapshots
D. Run an Auto Scaling group in one AZ and add an RDS read replica

**Q35.** Traffic jumps every weekday at 08:30 when offices open. New instances take 10 minutes to be ready, so target tracking reacts too late. What is the simplest fix?
A. Increase the scaling cooldown period
B. Add a scheduled scaling action that raises the group's capacity at around 08:15 on weekdays
C. Switch to larger instance types
D. Use step scaling on CPU with lower thresholds

**Q36.** An Amazon RDS for MySQL database receives the same expensive read queries thousands of times per minute. Slightly stale results are acceptable. The company wants lower latency and less database load. What should the architect do?
A. Enable Multi-AZ
B. Increase the provisioned IOPS of the RDS storage
C. Add DynamoDB Accelerator (DAX) in front of RDS
D. Add Amazon ElastiCache (Valkey or Redis OSS) using a lazy-loading strategy with a TTL

**Q37.** Users on several continents upload multi-gigabyte video files to one S3 bucket in us-east-1. Uploads are slow and sometimes fail partway. What should the architect recommend?
A. Store the uploads in S3 One Zone-IA
B. Request a higher bucket throughput limit from AWS Support
C. Enable Cross-Region Replication to buckets near the users
D. Enable S3 Transfer Acceleration and use multipart uploads

**Q38.** A new application on DynamoDB has unknown, highly spiky traffic. The company wants to avoid paying for idle capacity and avoid capacity planning. Which capacity mode should it use?
A. Provisioned capacity with reserved capacity
B. Provisioned capacity sized for the peak
C. Provisioned capacity with a high minimum for auto scaling
D. On-demand capacity mode

**Q39.** A security engineer must identify any S3 buckets, KMS keys, SQS queues or IAM roles in the organization that grant access to principals outside the organization. Which service provides this with the least effort?
A. IAM Access Analyzer with the organization as the zone of trust
B. AWS Trusted Advisor
C. IAM Access Advisor
D. AWS CloudTrail Lake queries

**Q40.** In a three-tier application, the application EC2 instances must accept traffic only from the internet-facing ALB. Instances scale in and out frequently. Which configuration meets the requirement with the least maintenance?
A. Add a NACL rule allowing only the public subnets' CIDR ranges
B. In the application security group, allow the application port with the ALB's security group as the source
C. Allow the ALB's current private IP addresses in the application security group
D. Allow 0.0.0.0/0 on the application port because the instances are in private subnets

**Q41.** EC2 instances in private subnets have no route to the internet. Administrators need shell access without opening inbound ports or running a bastion host. Which TWO actions are required? (Choose two.)
A. Create interface VPC endpoints for ssm, ssmmessages and ec2messages
B. Attach an instance profile that includes the AmazonSSMManagedInstanceCore policy
C. Allow inbound TCP 22 from 0.0.0.0/0 in the security group
D. Assign Elastic IP addresses to the instances
E. Create a gateway VPC endpoint for Systems Manager

**Q42.** Nightly batch jobs checkpoint their progress and can be stopped and restarted at any time. The company wants the LOWEST compute cost. What should it use?
A. 3-year Standard Reserved Instances
B. Spot Instances diversified across several instance types and AZs with a price-capacity-optimized allocation strategy
C. Dedicated Hosts
D. On-Demand Instances

**Q43.** A company stores sensitive reports in an S3 bucket. Application servers in private subnets read the bucket through an S3 gateway VPC endpoint. The security team requires that objects can be read ONLY through that endpoint, even by IAM principals that have s3:GetObject permissions. What should a solutions architect do?
A. Add a bucket policy that denies s3:GetObject when aws:SourceVpce does not equal the endpoint ID
B. Add a bucket policy that allows s3:GetObject only when aws:SourceIp matches the private subnets' CIDR ranges
C. Attach an endpoint policy to the gateway endpoint that allows access only to the reports bucket
D. Enable S3 Block Public Access on the bucket

**Q44.** An EFS file system holds 10 TB, and about 80% of the files aren't accessed after 30 days. Those files must still be available immediately to NFS clients when needed. What is the MOST cost-effective solution?
A. Move the files to S3 Glacier Deep Archive with DataSync
B. Enable EFS lifecycle management to move files to EFS Infrequent Access after 30 days, and back to Standard on access
C. Migrate to Amazon FSx for Lustre
D. Move the data to EBS sc1 volumes

**Q45.** At 14:05 a faulty script deleted thousands of items from a DynamoDB table that has point-in-time recovery enabled. How should the team recover the data?
A. Enable TTL to undo the deletions
B. Restore the point-in-time backup in place over the existing table
C. Promote a global table replica in another Region
D. Restore the table to a point in time just before 14:05 into a new table, then repoint the application or copy the missing items back

**Q46.** A company runs several microservices behind a single domain. Requests to /api/orders must reach the orders service and /api/users the users service. What should the architect use?
A. A Network Load Balancer with one listener per service
B. A Gateway Load Balancer
C. A Classic Load Balancer for each service
D. An Application Load Balancer with path-based listener rules forwarding to separate target groups

**Q47.** Linux web servers in an Auto Scaling group spanning three AZs need a shared POSIX file system for user uploads that stays available if one AZ fails. Which solution is BEST?
A. An EBS gp3 volume with Multi-Attach enabled
B. Amazon EFS One Zone
C. Instance store volumes synchronised with rsync
D. Amazon EFS with Regional storage and a mount target in each AZ

**Q48.** A self-managed database on EC2 needs consistent sub-millisecond latency and more IOPS than a gp3 volume can provide, with the highest EBS durability. Which volume type should the architect choose?
A. gp3 with maximum provisioned IOPS
B. io2 Block Express
C. st1
D. sc1

**Q49.** A company runs a steady compute workload 24/7 and will keep it for three years. Over that period it plans to move parts of it from EC2 to AWS Fargate and AWS Lambda. Which purchasing option provides the MOST savings with that flexibility?
A. Standard Reserved Instances for 3 years
B. Compute Savings Plan
C. EC2 Instance Savings Plan
D. Spot Instances

**Q50.** A regulation requires the company to keep backups of an Amazon RDS database for one year. What should the architect do?
A. Enable Multi-AZ
B. Create an AWS Backup plan that takes RDS snapshots with a one-year retention rule (or schedule manual snapshots)
C. Create a read replica and keep it for a year
D. Set the automated backup retention period to 365 days

**Q51.** Analysts run complex SQL joins and aggregations over 50 TB of historical sales data to power BI dashboards. The current RDS database is overloaded. Which service is the BEST fit?
A. Additional RDS read replicas
B. Amazon ElastiCache
C. Amazon Redshift
D. Amazon DynamoDB

**Q52.** A company is setting up S3 Cross-Region Replication from a bucket in ap-south-1 to a new bucket in ap-southeast-1 for disaster recovery. The source bucket already holds 50 TB of objects that must also be in the destination. Which TWO actions are required? (Choose two.)
A. Enable S3 Versioning on both the source and destination buckets
B. Enable S3 Object Lock on the source bucket
C. Use S3 Batch Replication to replicate the existing objects
D. Enable S3 Transfer Acceleration on the destination bucket
E. Create a lifecycle rule that transitions objects to the other Region

**Q53.** An application on EC2 in ap-south-1 must be recoverable in ap-southeast-1 within a few hours if the primary Region fails. The company wants to minimise cost. What should the architect do?
A. Rely on the AMI being available automatically in every Region
B. Enable EBS Multi-Attach between the two Regions
C. Run a full-size duplicate fleet in ap-southeast-1 at all times
D. Copy AMIs and EBS snapshots to ap-southeast-1 on a schedule (for example with AWS Backup or Data Lifecycle Manager) and launch from them during a disaster

**Q54.** A tightly coupled HPC application uses MPI and needs the lowest possible inter-node latency and highest network throughput. What should the architect do?
A. Use a partition placement group across AZs
B. Launch the instances in a cluster placement group in one AZ using instance types that support Elastic Fabric Adapter
C. Launch the instances in different Regions close to the users
D. Spread the instances across three AZs in a spread placement group

**Q55.** A multiplayer game uses UDP, must handle millions of packets per second with very low latency, and partners need to allow-list one fixed IP address per AZ. Which load balancer should the architect choose?
A. A Network Load Balancer with an Elastic IP address in each AZ
B. A Classic Load Balancer
C. An Application Load Balancer
D. An Application Load Balancer with AWS WAF

**Q56.** A platform team wants application teams to create IAM roles for their own workloads, but no role they create may ever grant IAM administration or access to services outside an approved list. What should the platform team implement?
A. Require a permissions boundary on every role the application teams create, enforced with an iam:PermissionsBoundary condition in their policies
B. IAM Access Analyzer external access findings
C. An SCP that denies iam:CreateRole in the application accounts
D. Allow only inline policies on roles

**Q57.** NAT gateway data processing charges are high. Analysis shows most of the traffic is instances in private subnets downloading objects from S3 in the same Region. What is the MOST cost-effective change?
A. Create an S3 gateway VPC endpoint and associate it with the private subnets' route tables
B. Route the private subnets' 0.0.0.0/0 traffic straight to the internet gateway
C. Enable S3 Transfer Acceleration
D. Create an S3 interface VPC endpoint

**Q58.** Web servers in a public subnet are receiving malicious traffic from 198.51.100.0/24. The team must block this range at the subnet level immediately. What should it do?
A. Remove the subnet's route to the internet gateway
B. Add a deny rule for the range to the web servers' security group
C. Add an inbound DENY rule for 198.51.100.0/24 to the subnet's network ACL with a lower rule number than the allow rules
D. Enable VPC Flow Logs with a REJECT filter for the range

**Q59.** Regulatory archives must be kept for 10 years and are almost never accessed. Retrieval within 48 hours is acceptable. Which storage class is MOST cost-effective?
A. S3 Glacier Instant Retrieval
B. S3 Standard-IA
C. S3 Glacier Flexible Retrieval
D. S3 Glacier Deep Archive

**Q60.** A product catalogue on DynamoDB is read-heavy and must deliver microsecond read latency. Developers want minimal code changes. What should the architect recommend?
A. A global secondary index
B. Amazon ElastiCache with lazy-loading logic in the application
C. Strongly consistent reads
D. Amazon DynamoDB Accelerator (DAX)

**Q61.** A company must bring existing software licences that are billed per physical socket and core. It wants the lowest-cost compliant option on AWS. What should it use?
A. On-Demand Instances with default tenancy
B. EC2 Dedicated Hosts
C. EC2 Dedicated Instances
D. Spot Instances

**Q62.** Application logs written to S3 are queried often for 30 days, occasionally until day 90 with millisecond access, then kept until day 365 for audits where retrieval in hours is acceptable, then deleted. Which lifecycle configuration is MOST cost-effective?
A. S3 Standard → S3 One Zone-IA at 7 days → expire at 365 days
B. Keep everything in S3 Standard and delete at 365 days
C. S3 Standard → S3 Glacier Deep Archive at day 1 → expire at 365 days
D. S3 Standard → S3 Standard-IA at 30 days → S3 Glacier Flexible Retrieval at 90 days → expire at 365 days

**Q63.** A bank's hybrid workloads depend on AWS Direct Connect. The connection must survive the failure of an entire Direct Connect location as well as a single device. Which design meets this requirement?
A. Two connections terminating on the same device at one location
B. Direct Connect connections at two separate Direct Connect locations, with redundant connections at each
C. A Direct Connect gateway in front of a single connection
D. One higher-bandwidth connection at one location

**Q64.** Analysts run heavy reporting queries against a production Amazon RDS for PostgreSQL instance, slowing customer transactions. The company wants to isolate the reporting load without changing the application. What should the architect do?
A. Add a DAX cluster in front of the database
B. Enable Multi-AZ and point the reporting tool at the standby
C. Increase the backup retention period
D. Create a read replica and point the reporting tool at the replica's endpoint

**Q65.** A penetration test found that a server-side request forgery (SSRF) flaw in a web application running on EC2 could retrieve the instance role's credentials from instance metadata. Which change mitigates this with the LEAST application change?
A. Require IMDSv2 (session tokens) on the instances
B. Disable the instance metadata service and hard-code credentials
C. Block 169.254.169.254 in the instance's security group
D. Remove the instance role and store access keys in environment variables

<details>
<summary><strong>Answer key (open only after all 65 are answered)</strong></summary>

Each line gives the answer, the domain and source day, and why it wins.

1. **D** · *Cost-optimized · Day 2* — Compute Optimizer analyses utilisation metrics and recommends right-sized instance types (also EBS, Lambda and ASGs).
2. **A** · *High-performing · Day 5* — FSx for Lustre is the HPC/ML parallel file system and lazy-loads from S3. File Gateway is for on-premises access to S3.
3. **A** · *Secure · Day 6* — Secrets Manager has built-in rotation for RDS. Parameter Store has no native rotation; the other options are custom and keep the secret on disk.
4. **D** · *Resilient · Day 5* — FSx for Windows provides SMB with AD; Multi-AZ adds a standby file server in another AZ. EFS is NFS for Linux.
5. **C** · *Cost-optimized · Day 3* — On-Demand base (covered by a Savings Plan) protects the minimum; Spot is cheapest for interruptible burst. All-Spot risks dropping below 4.
6. **B** · *Secure · Day 1* — EC2 has no resource-based policies, so cross-account means assuming a role. Keys in environment variables are long-lived credentials; RAM doesn't share instances for API access.
7. **C** · *High-performing · Day 6* — RDS Proxy pools and shares connections. More Lambda memory doesn't change connection counts.
8. **A** · *Cost-optimized · Day 5* — Incomplete multipart parts are billed but invisible in listings; the lifecycle action cleans them up.
9. **A** · *High-performing · Day 3* — Warm pools keep instances already initialised (stopped or running) for fast scale-out. Scheduled scaling needs a predictable pattern.
10. **A** · *Secure · Day 6* — RDS encryption is fixed at creation; the snapshot-copy-restore path is the documented method. EBS default encryption has no effect on RDS.
11. **B** · *Resilient · Day 6* — Aurora Global Database replicates at the storage layer with typically sub-second lag and fast managed failover. Cross-Region replicas lag more and need manual promotion; DynamoDB isn't relational.
12. **B** · *High-performing · Day 4* — Transit Gateway is the hub for many VPCs and on-premises with routing domains. A DX gateway doesn't route VPC-to-VPC; PrivateLink exposes services, not networks.
13. **D** · *High-performing · Day 6* — GSIs can be added any time and use a different partition key. LSIs must be created with the table and keep the same partition key; Scan reads the whole table.
14. **B** · *Resilient · Day 4* — A NAT gateway is redundant only within its AZ. Per-AZ NAT gateways and route tables remove the cross-AZ dependency.
15. **A** · *Resilient · Day 5* — Versioning keeps prior versions and turns deletes into delete markers; the lifecycle rule caps cost. CRR alone replicates the mistakes; compliance-mode Object Lock blocks legitimate deletes.
16. **A** · *Cost-optimized · Day 6* — Aurora Serverless v2 scales capacity with demand and shrinks when idle (check current docs for minimum capacity and scale-to-zero). Redshift is a warehouse; self-managed adds ops.
17. **B** · *Cost-optimized · Day 5* — Intelligent-Tiering moves objects between tiers by observed access with no retrieval fees. IA classes charge per-GB retrieval and assume you know the pattern.
18. **A** · *Secure · Day 5* — Reading SSE-KMS objects requires kms:Decrypt. Bucket Keys reduce KMS request volume/cost; they don't grant access.
19. **D** · *Secure · Day 2* — Encryption by default enforces it at creation for every new volume in that Region. Config only detects; a Lambda can't encrypt a volume in place.
20. **C** · *Secure · Day 4* — Gateway endpoints are only usable by traffic originating inside the VPC. Interface endpoints have private IPs reachable over DX/VPN.
21. **C** · *Resilient · Day 3* — By default an ASG uses EC2 status checks only, which pass on a hung app. ELB health checks let the ASG replace instances the load balancer considers unhealthy.
22. **B** · *Secure · Day 2* — AWS managed keys can't be shared or edited. Cross-account encrypted AMIs require a customer managed key shared with the target account.
23. **A** · *Secure · Day 1* — SCPs bind every principal in member accounts, including root. IAM policies can't restrict a root user and local admins can edit them; Config is detective.
24. **B** · *Resilient · Day 2* — Spread places each instance on distinct racks. Cluster packs them together; one Dedicated Host is one piece of hardware.
25. **D** · *Secure · Day 4* — DX is private but not encrypted. IPsec VPN over DX (or MACsec where supported) adds encryption while keeping DX performance.
26. **D** · *Resilient · Day 6* — Multi-AZ uses synchronous replication and automatic failover. Read replicas are asynchronous and require promotion, so they can lose data.
27. **C** · *Secure · Day 6* — LeadingKeys gives row-level access control on the partition key. Per-player tables or indexes don't scale; encryption isn't authorization.
28. **A** · *Resilient · Day 3* — Terminating:Wait holds the instance until the hook completes. Termination protection doesn't stop ASG scale-in; cooldown doesn't delay termination of a chosen instance.
29. **C** · *Resilient · Day 3* — Externalising sessions lets any instance serve any user and survive termination. Stickiness only delays the loss.
30. **B** · *Secure · Day 5* — OAC lets CloudFront sign requests to a private bucket. Public-read lets users bypass CloudFront; edge IP allow-lists are brittle.
31. **B** · *Secure · Day 5* — Compliance mode can't be shortened or bypassed by anyone. Governance mode can be bypassed with s3:BypassGovernanceRetention; MFA Delete can be turned off by root; storage class doesn't prevent deletion.
32. **C** · *Resilient · Day 6* — Global tables are multi-active: every replica Region accepts writes. Aurora Global Database has a single primary writer Region.
33. **C** · *Secure · Day 1* — Identity Center centralises workforce SSO across accounts. Per-account SAML works but multiplies configuration; Cognito is for customer-facing apps.
34. **B** · *Resilient · Day 3/6* — HA = every tier in ≥ 2 AZs; managed Multi-AZ removes DB management. A cluster placement group is single-AZ by definition.
35. **B** · *High-performing · Day 3* — A known, recurring time → scheduled scaling (predictive scaling also fits a recurring pattern). Lower thresholds still react after the spike starts.
36. **D** · *High-performing · Day 6* — A cache in front of RDS absorbs repeated reads; TTL bounds staleness. DAX only fronts DynamoDB; Multi-AZ doesn't serve reads.
37. **D** · *High-performing · Day 5* — Transfer Acceleration uses the edge network for long-distance uploads; multipart lets failed parts retry and parallelises transfers.
38. **D** · *Cost-optimized · Day 6* — On-demand bills per request and absorbs spikes with no planning. Provisioned modes suit steady, predictable traffic.
39. **A** · *Secure · Day 1* — Access Analyzer's external access findings evaluate resource policies against your zone of trust. Access Advisor shows last-accessed services; CloudTrail shows activity, not policy exposure.
40. **B** · *Secure · Day 3/4* — Security group referencing follows the ALB nodes as they change. Hard-coded IPs drift; a CIDR rule admits anything in those subnets.
41. **A, B** · *Secure · Day 4* — Session Manager needs the role and a private path to the SSM APIs. Gateway endpoints exist only for S3 and DynamoDB.
42. **B** · *Cost-optimized · Day 2* — Interruptible, restartable work is the Spot use case. Diversification reduces interruptions.
43. **A** · *Secure · Day 4/5* — Only a bucket-side Deny keyed on the endpoint stops every other path. aws:SourceIp doesn't see private IPs through a gateway endpoint; an endpoint policy limits what the VPC can reach, not who can reach the bucket; Block Public Access doesn't affect authenticated principals.
44. **B** · *Cost-optimized · Day 5* — EFS IA keeps the same file system path and access with a lower storage price. Glacier breaks immediate access; sc1 isn't shared.
45. **D** · *Resilient · Day 6* — DynamoDB restores always create a new table. Global table replicas received the same deletes.
46. **D** · *High-performing · Day 3* — Path-based routing is a layer 7 ALB feature; NLBs route on ports only.
47. **D** · *Resilient · Day 5* — Regional EFS stores data across AZs and is mountable from each. Multi-Attach isn't supported on gp3 and is single-AZ; EFS One Zone dies with its AZ.
48. **B** · *High-performing · Day 2* — io2 Block Express is the top-performance, highest-durability EBS SSD. st1 and sc1 are HDD throughput types.
49. **B** · *Cost-optimized · Day 2* — Compute Savings Plans apply across EC2 families, Regions, Fargate and Lambda. EC2 Instance SPs and Standard RIs are tied to EC2 and an instance family.
50. **B** · *Resilient · Day 6* — Automated backups max out at 35 days. AWS Backup or manual snapshots persist as long as you keep them.
51. **C** · *High-performing · Day 6* — Columnar, MPP data warehousing at TB–PB scale is Redshift. Read replicas are still row-store OLTP engines.
52. **A, C** · *Resilient · Day 5* — Replication requires versioning on both buckets and only copies new objects; Batch Replication backfills existing ones. Lifecycle rules can't move objects between Regions.
53. **D** · *Resilient · Day 2* — Backup-and-restore matches an RTO of hours at the lowest cost. AMIs and snapshots are Regional and must be copied; Multi-Attach is single-AZ.
54. **B** · *High-performing · Day 2* — Cluster placement + EFA gives the lowest latency and highest throughput. Spreading increases latency.
55. **A** · *High-performing · Day 3* — NLB handles UDP at extreme scale and supports static Elastic IPs per AZ. ALB is HTTP/HTTPS only, with changing IPs.
56. **A** · *Secure · Day 1* — Permissions boundaries cap delegated role creation. Denying CreateRole blocks the teams entirely; Access Analyzer detects external sharing, it doesn't cap permissions.
57. **A** · *Cost-optimized · Day 4* — Gateway endpoints carry S3 traffic free of NAT processing charges. Interface endpoints have hourly and per-GB fees; routing private subnets to the IGW doesn't work without public IPs and exposes them.
58. **C** · *Secure · Day 4* — NACLs support deny and are evaluated lowest number first. Security groups can only allow; removing the IGW route takes the site down; Flow Logs only record.
59. **D** · *Cost-optimized · Day 5* — Deep Archive is the cheapest class, with retrieval within 12–48 hours. Flexible Retrieval is cheap but costs more to store.
60. **D** · *High-performing · Day 6* — DAX is an API-compatible in-memory cache for DynamoDB. ElastiCache would need caching code; strongly consistent reads are slower, not faster.
61. **B** · *Cost-optimized · Day 2* — Dedicated Hosts expose sockets and cores for licensing. Dedicated Instances isolate hardware but don't give socket/core visibility.
62. **D** · *Cost-optimized · Day 5* — Each phase matches the access need. Deep Archive at day 1 breaks the frequent-query phase; transitions to IA classes need 30 days in Standard.
63. **B** · *Resilient · Day 4* — Only multiple locations protect against a location failure; redundant connections per location protect against device failure. A DX gateway adds Region reach, not redundancy.
64. **D** · *High-performing · Day 6* — Read replicas offload reads. A Multi-AZ instance standby isn't readable; DAX works only with DynamoDB.
65. **A** · *Secure · Day 1/2* — IMDSv2 needs a PUT-obtained session token that simple SSRF can't fetch. Security groups don't filter link-local metadata traffic; static keys make things worse.

</details>

---

## 11. Scoring guide

### Step 1 — raw score

Count correct answers (a *Choose two* question counts only if both picks are right).

| Raw score | % | Read it as | Next move |
| --- | --- | --- | --- |
| 56–65 | 86%+ | Strong on Days 1–6 | Keep pace; spend review time on the few misses |
| 52–55 | 80–85% | On track for the go/no-go bar | Log misses; re-drill the weakest domain tomorrow morning |
| 46–51 | 70–79% | Borderline | Re-read the sections behind every miss before Day 8; add 30 bank questions on the weakest domain |
| < 46 | < 70% | Foundation gaps | Redo the decision tables of Days 2–6 from memory tonight; take Day 8 more slowly; re-check the Day 12 bar |

The real exam is **scaled** (720/1000 to pass), includes 15 unscored questions, and isn't a straight percentage — treat 80% on practice exams as the safety margin the README's go/no-go rule asks for, not as a conversion.

### Step 2 — score by domain

Tally your correct answers per domain using these question numbers:

| Domain (exam weight) | Questions | Count | Your score | 80% bar |
| --- | --- | --- | --- | --- |
| Secure (30%) | 3, 6, 10, 18, 19, 20, 22, 23, 25, 27, 30, 31, 33, 39, 40, 41, 43, 56, 58, 65 | 20 | __ / 20 | 16 |
| Resilient (26%) | 4, 11, 14, 15, 21, 24, 26, 28, 29, 32, 34, 45, 47, 50, 52, 53, 63 | 17 | __ / 17 | 14 |
| High-performing (24%) | 2, 7, 9, 12, 13, 35, 36, 37, 46, 48, 51, 54, 55, 60, 64 | 15 | __ / 15 | 12 |
| Cost-optimized (20%) | 1, 5, 8, 16, 17, 38, 42, 44, 49, 57, 59, 61, 62 | 13 | __ / 13 | 11 |

### Step 3 — map weak domains back to the lessons

| Weak domain | Re-read first | Then |
| --- | --- | --- |
| Secure | Day 1 §2 (policy evaluation), Day 4 §3–4 (SG/NACL, endpoints), Day 5 §5 (S3 security, encryption) | This lesson's §4 encryption table; Day 6 encryption and Secrets Manager rules |
| Resilient | Day 3 §3–4 (health checks, ASG), Day 6 §3–4 (Multi-AZ, replicas, Aurora) | This lesson's §2–3 scope and HA tables; Day 4 NAT/DX resilience |
| High-performing | Day 2 §2, §4–5 (instance families, placement, EBS types), Day 6 §5–6 (DynamoDB indexes, caching) | Day 3 ALB vs NLB; Day 5 FSx and S3 performance |
| Cost-optimized | Day 2 §3 (purchasing options), Day 5 §3 (storage classes, lifecycle) | Day 4 NAT vs endpoint cost; Day 6 capacity modes |

### Step 4 — log every miss

For each wrong (or lucky-guess) answer, add one line to `miss-log.md`:

```markdown
| 7 | RDS HA | Read replica | Multi-AZ | "Automatic failover, no data loss" = synchronous standby; replicas are async + manual promote |
```

Log lucky guesses too — they're misses you haven't paid for yet. Look for **repeats**: two misses with the same discriminator (e.g. both gateway-vs-interface endpoint) are a pattern, and patterns are what Day 13 targets.

**Don't retake this exam for at least a few days.** A second attempt now measures memory of the answers, not understanding.

---

## 12. Flashcards (cover the right column)

| Prompt | Answer |
| --- | --- |
| Scope of a NAT gateway / EBS volume / subnet | AZ |
| Scope of an ALB / ASG / EFS (Regional) / DynamoDB table | Region |
| Move an EBS volume to another AZ | Snapshot → new volume in the target AZ |
| Move an AMI to another Region | Copy the AMI |
| Encrypt an existing RDS instance | Snapshot → encrypted copy → restore |
| Two "flip a switch" encryption settings (future only) | EBS encryption by default; S3 default encryption |
| Share an encrypted AMI cross-account | Customer managed KMS key + share key and AMI |
| AccessDenied on SSE-KMS object despite `s3:GetObject` | Missing `kms:Decrypt` |
| RDS/Aurora automated backup max retention | 35 days |
| Backups for a year, many services, cross-account | AWS Backup |
| Which restores happen in place? | S3 object versions; Aurora Backtrack |
| Gateway endpoints exist for | S3 and DynamoDB only |
| S3 from on-prem over DX privately | Interface endpoint |
| Block an IP range | NACL deny (SGs can't deny) |
| Shell access without SSH or bastion | Session Manager (role + SSM endpoints if no internet) |
| ASG not replacing app-unhealthy instances | Health check type ELB |
| Sync, automatic failover, standby not readable | RDS Multi-AZ instance |
| Relational, cross-Region RPO ~1 s | Aurora Global Database |
| Key-value, multi-Region active-active writes | DynamoDB global tables |
| Microsecond DynamoDB reads, no code change | DAX |
| Query an existing DynamoDB table by a new attribute | GSI |
| Steady 3-yr compute moving to Fargate/Lambda | Compute Savings Plan |
| Unpredictable S3 access, no retrieval fees | Intelligent-Tiering |
| Cheapest S3 class, 12–48 h retrieval | Glacier Deep Archive |
| Licences per socket/core | Dedicated Hosts |
| Service shared to overlapping-CIDR VPCs | NLB + PrivateLink endpoint service |

## 13. Before you close today

- [ ] Sections 1–8 reviewed; redrew the §1 architecture from memory and labelled the HA, security and cost move at each layer
- [ ] Lab done and torn down (instance terminated, bucket policy and bucket deleted, gateway endpoint deleted, role and instance profile deleted)
- [ ] Warm-up: 10 integration questions
- [ ] Practice Exam 1 taken in one 130-minute sitting, **then** scored
- [ ] Raw score and per-domain scores written at the top of `miss-log.md` (e.g. `Exam 1 (Day 7): 51/65 — S 16/20 · R 13/17 · P 12/15 · C 10/13`)
- [ ] Every miss and lucky guess logged in `miss-log.md`
- [ ] Re-read the lesson sections behind your weakest domain (§11 step 3)

**Tomorrow (Day 8):** decoupling and serverless — SQS, SNS, EventBridge, Kinesis (including Amazon Data Firehose), Lambda, API Gateway and Step Functions.
