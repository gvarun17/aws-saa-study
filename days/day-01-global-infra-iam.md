# Day 1 — Global Infrastructure, IAM & Organizations

**Domain weight:** mostly *Design Secure Architectures* (30%), plus the foundation for *Resilient* (26%).
**Time plan:** Learn 90 min · Lab 30 min · Drill 45 min · Review 30 min.

By the end of today you should be able to:

- Pick a Region for a given scenario and explain why.
- Explain why "highly available" on the exam almost always means "multi-AZ".
- Trace any request through IAM policy evaluation and say whether it's allowed.
- Choose between users, roles, Identity Center, resource policies, SCPs and permissions boundaries for a scenario.

---

## 1. Global infrastructure

```
┌──────────────────────── Region (e.g. ap-south-1) ────────────────────────┐
│  ┌──── AZ a ────┐   ┌──── AZ b ────┐   ┌──── AZ c ────┐                  │
│  │ 1+ data      │◄─►│ 1+ data      │◄─►│ 1+ data      │   low-latency    │
│  │ centers      │   │ centers      │   │ centers      │   private links  │
│  └──────────────┘   └──────────────┘   └──────────────┘                  │
└──────────────────────────────────────────────────────────────────────────┘
        Edge locations (hundreds, worldwide): CloudFront · Route 53 · Global Accelerator
```

### Regions

A Region is a physically separate geographic area (e.g. `us-east-1`, `ap-south-1`, `eu-west-1`). Regions are **isolated**: data you store in one Region stays there unless *you* replicate or move it.

**How to choose a Region (exam order of priority):**

1. **Compliance / data residency** — "customer data must remain in Germany" overrides everything else.
2. **Latency** — closest to users.
3. **Service availability** — newer services aren't in every Region.
4. **Price** — varies by Region (`us-east-1` is usually cheapest).

Global vs regional services matter for questions like "where is this configured?":

| Global (one control plane) | Regional |
| --- | --- |
| IAM, Organizations, Route 53, CloudFront, WAF (for CloudFront), Global Accelerator | EC2, VPC, S3 buckets (bucket lives in one Region, namespace is global), RDS, Lambda, DynamoDB, most others |

> Trap: S3 bucket **names** are globally unique, but each bucket lives in **one Region**.

### Availability Zones (AZs)

An AZ is one or more discrete data centers with independent power, cooling and networking. A Region has at least 3 AZs, connected by redundant, low-latency links (single-digit ms).

**The resilience rule:** assume an AZ can fail. Anything that must be highly available spans ≥ 2 AZs.

| Service | What "multi-AZ" looks like |
| --- | --- |
| EC2 | Auto Scaling group across subnets in multiple AZs, behind a load balancer |
| RDS | Multi-AZ deployment (synchronous standby in another AZ) |
| EFS | Regional by default (stored across AZs) |
| S3 Standard | Automatically stores across ≥ 3 AZs |
| S3 One Zone-IA / EBS / instance store | **Single AZ** — lost if that AZ is lost |

> AZ names like `us-east-1a` map to different physical AZs per account. AZ **IDs** (`use1-az1`) are consistent across accounts — relevant when sharing subnets or aligning with another account.

### Edge locations and other footprints

- **Edge locations / PoPs:** far more numerous than Regions. Host CloudFront caches, Route 53 DNS, and Global Accelerator entry points. They make things *faster for users*; you don't run your app there.
- **Local Zones:** extension of a Region into a metro area for single-digit-ms latency (e.g. media rendering, gaming).
- **Wavelength Zones:** AWS compute inside telecom 5G networks.
- **Outposts:** AWS racks in *your* data center — for low latency to on-prem systems or data that must stay on-site.

Exam cue → answer:

| Scenario phrase | Answer |
| --- | --- |
| "Users worldwide, static content slow" | CloudFront |
| "Data must stay in country X" | Deploy in a Region in country X |
| "Single-digit ms latency to users in a specific city" | Local Zone |
| "Run AWS services on premises" | Outposts |
| "Survive a data-center failure" | Multi-AZ |
| "Survive a Region failure" | Multi-Region (DR — covered Day 11) |

---

## 2. IAM fundamentals

IAM controls **who** (principal) can do **what** (actions) on **which** resources, under **which conditions**. It's global and free.

### Identities

| Identity | Credentials | Use it for |
| --- | --- | --- |
| **Root user** | Email + password (+ MFA) | Almost nothing. Account/billing tasks only. |
| **IAM user** | Long-term password and/or access keys | Legacy, break-glass, or a few service accounts that can't use roles |
| **IAM group** | None (container) | Attach policies to many users at once. **Cannot be nested**; groups can't be principals in policies. |
| **IAM role** | **Temporary** credentials via STS `AssumeRole` | EC2, Lambda, ECS tasks, cross-account access, federated humans — the default answer |

**Why roles win:** temporary credentials expire automatically (15 min – 12 h), nothing to leak or rotate. Any exam answer that puts access keys on an EC2 instance, in code, or in an AMI is wrong.

A role has two policies:

1. **Trust policy** — *who can assume* the role (a service like `ec2.amazonaws.com`, another account, a SAML/OIDC provider).
2. **Permissions policy** — *what the role can do* once assumed.

```json
// Trust policy: let EC2 assume this role
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ec2.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
```

For EC2, the role is delivered through an **instance profile**; the app reads temporary creds from the instance metadata service (use **IMDSv2**, which requires a session token and blocks SSRF-style credential theft).

### Policy anatomy

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "ReadReports",
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::acme-reports",
      "arn:aws:s3:::acme-reports/*"
    ],
    "Condition": {
      "Bool": { "aws:SecureTransport": "true" },
      "IpAddress": { "aws:SourceIp": "203.0.113.0/24" }
    }
  }]
}
```

- `ListBucket` targets the **bucket** ARN; `GetObject` targets **objects** (`/*`). Mixing these up is a classic misconfiguration.
- **Conditions** to recognise: `aws:SourceIp`, `aws:SourceVpc` / `aws:SourceVpce` (restrict to a VPC endpoint), `aws:SecureTransport` (force HTTPS), `aws:MultiFactorAuthPresent`, `aws:PrincipalOrgID` (anyone in my Organization), `aws:RequestedRegion`, tag conditions (`aws:ResourceTag/…`, `aws:PrincipalTag/…` — the basis of ABAC).

### Policy types

| Type | Attached to | Grants? | Typical exam use |
| --- | --- | --- | --- |
| **Identity-based** (AWS managed, customer managed, inline) | User, group, role | Yes | Normal permissions |
| **Resource-based** | Resource (S3 bucket, SQS queue, SNS topic, KMS key, Lambda) | Yes, and names a `Principal` | Cross-account access without assuming a role |
| **Permissions boundary** | User or role | **No** — caps max | Let devs create roles safely |
| **SCP** (Organizations) | Account / OU / root | **No** — caps max for the account | Org-wide guardrails |
| **Session policy** | An assumed-role session | No — caps the session | Rare on SAA |

Prefer **customer managed** policies over inline (reusable, versioned). Inline is for a strict 1:1 binding to one identity.

### Policy evaluation logic (memorise this)

```
Request arrives
   │
   ├─ Any explicit DENY in any applicable policy? ──► DENIED (final)
   │
   ├─ SCPs allow it? (if in an Organization) ──── no ─► DENIED
   │
   ├─ Resource-based policy allows it? ─────────── yes ► ALLOWED*
   │
   ├─ Permissions boundary allows it? (if set) ── no ─► DENIED
   │
   ├─ Identity-based policy allows it? ─────────── no ─► DENIED (implicit)
   │
   └─ ALLOWED
```

\* Same-account simplification. For **cross-account** access, *both* sides must allow: the caller's identity policy **and** the resource's policy (or the target role's trust policy).

Three rules carry 90% of exam questions:

1. **Default is deny.**
2. **Explicit deny always wins.**
3. **Guardrails (SCP, boundary) don't grant** — effective permissions are the *intersection* of guardrail and identity policy.

Worked example — the quick check from chat:
User policy allows `s3:*`. SCP on the OU denies `s3:DeleteBucket`. → **Cannot delete.** The SCP's explicit deny wins; and even if the SCP merely *didn't list* DeleteBucket under an allow-list SCP, it would be an implicit deny at the guardrail level.

### Cross-account access: two patterns

| Pattern | How | When |
| --- | --- | --- |
| **Assume a role** | Account B creates a role trusting Account A; A's users call `sts:AssumeRole` | Broad access; caller gives up its own permissions while in the role |
| **Resource-based policy** | Bucket/queue/key policy names Account A as `Principal` | Single resource; caller keeps its own permissions (useful for copying data between accounts) |

S3 note: with cross-account uploads, the bucket owner may not own the objects unless **Object Ownership = Bucket owner enforced** (ACLs disabled — the modern default).

### Federation and workforce identity

| Need | Answer |
| --- | --- |
| Employees sign into many AWS accounts with corporate credentials | **IAM Identity Center** (successor to AWS SSO) + permission sets, connected to Okta/Entra ID/AD |
| Single account, SAML 2.0 IdP | IAM SAML identity provider + roles |
| Mobile/web app end users | **Amazon Cognito** (Day 10) |
| GitHub Actions / external workload needs AWS access | **OIDC identity provider** + role (no long-lived keys) |
| On-prem servers need AWS creds | IAM Roles Anywhere (X.509 certificates) |
| Microsoft AD needed in AWS | AWS Directory Service (Managed Microsoft AD or AD Connector) |

### Credential hygiene and auditing

- Root user: MFA, no access keys, locked away.
- Password policy for any IAM users; MFA for humans.
- **IAM Access Analyzer:** finds resources shared outside your account/org; validates policies; can generate least-privilege policies from CloudTrail activity.
- **Credential report** (account-wide, CSV of users and key ages) vs **Access Advisor** (last-accessed services per identity — use it to trim permissions).
- **CloudTrail** records every API call (who did what) — auditing, Day 11.

---

## 3. AWS Organizations

Organizations groups accounts under one **management account** with consolidated billing.

```
Root
├── Management account (no SCPs apply here)
├── OU: Security      → log archive acct, audit acct
├── OU: Workloads
│   ├── OU: Prod      → SCP: deny leaving org, deny disabling CloudTrail, region allow-list
│   └── OU: Dev
└── OU: Sandbox       → SCP: deny expensive instance types
```

Why multiple accounts? Strongest isolation boundary (blast radius, billing, quotas, security).

### Service control policies

- Apply to OUs or accounts; inherited down the tree.
- Affect **every principal in member accounts, including the root user**.
- **Never** affect the management account. Never grant anything.
- Common guardrails: deny `organizations:LeaveOrganization`; deny `cloudtrail:StopLogging`/`DeleteTrail`; deny actions outside approved Regions (`aws:RequestedRegion`); deny untagged resource creation.

**Consolidated billing:** one bill; usage aggregated for volume discounts; **Reserved Instance and Savings Plans discounts shared** across accounts by default.

Related services to recognise:

| Service | What it does |
| --- | --- |
| **AWS Control Tower** | Sets up a governed multi-account landing zone (Organizations + Identity Center + guardrails + log archive) in a few clicks. Answer for "quickly set up a secure multi-account environment following best practices." |
| **AWS RAM** (Resource Access Manager) | Share resources across accounts — VPC subnets, Transit Gateways, Route 53 Resolver rules, License Manager configs |
| **Tag policies** | Enforce tag standards across the org |

---

## 4. Exam decision table

| If the question says… | Pick |
| --- | --- |
| App on EC2/Lambda/ECS needs AWS API access | IAM **role** (instance profile / execution role / task role) |
| Access keys in code / config files / AMI | Always wrong — replace with a role |
| Prevent *everyone*, incl. root, in member accounts from X | **SCP** |
| Let developers create roles but never exceed certain permissions | **Permissions boundary** |
| Grant another account access to one S3 bucket | **Bucket policy** with that account as principal (or cross-account role) |
| Restrict bucket access to requests from my VPC | Bucket policy with `aws:SourceVpce` condition + VPC endpoint |
| Allow anything from my Organization only | `aws:PrincipalOrgID` condition |
| Workforce SSO to many accounts | **IAM Identity Center** |
| Set up many accounts with best-practice guardrails fast | **Control Tower** |
| Find resources shared publicly or externally | **IAM Access Analyzer** |
| Which permissions has this role actually used? | **Access Advisor** (last accessed) |
| Enforce MFA for sensitive actions | Policy condition `aws:MultiFactorAuthPresent` |
| CI/CD (e.g. GitHub Actions) deploying to AWS without keys | **OIDC provider + role** |

---

## 5. Lab (30 min, free tier)

Goal: prove to yourself that roles beat keys and that explicit deny wins.

1. **Create a role for EC2** with the AWS managed policy `AmazonS3ReadOnlyAccess`.
2. Launch a `t3.micro` (or `t4g.micro`) Amazon Linux instance with that role attached and **IMDSv2 required**. Connect with **Session Manager** (no SSH key, no open port 22 — note this, it's another exam answer).
3. Run `aws s3 ls` — works with no credentials configured. Then run:
   ```bash
   TOKEN=$(curl -sX PUT http://169.254.169.254/latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 60")
   curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/
   ```
   See the temporary credentials and their `Expiration`.
4. Add an **inline policy** to the role: `Deny s3:ListAllMyBuckets` on `*`. Run `aws s3 ls` again → `AccessDenied`. Explicit deny beat the managed allow.
5. Open **IAM Access Analyzer** → create an account analyzer and look at findings.
6. **Terraform variant** (if you prefer): write `aws_iam_role`, `aws_iam_instance_profile`, `aws_iam_role_policy` and an `aws_instance` with `metadata_options { http_tokens = "required" }`.
7. **Tear down:** terminate the instance; delete the role.

---

## 6. Practice questions

Answer all ten before opening the answers.

**Q1.** A company's application runs on EC2 and needs to write to a DynamoDB table. The security team prohibits long-term credentials on instances. What should a solutions architect do?
A. Store an IAM user's access keys in AWS Secrets Manager and retrieve them at startup
B. Create an IAM role with DynamoDB write permissions and attach it to the instances via an instance profile
C. Embed access keys in the AMI used by the Auto Scaling group
D. Create an IAM group with DynamoDB permissions and add the instances to it

**Q2.** A company uses AWS Organizations. It must ensure resources in all member accounts are created only in `eu-west-1` and `eu-central-1`. What is the MOST effective solution?
A. IAM policies in each account denying other Regions
B. An SCP attached to the root that denies actions when `aws:RequestedRegion` is not one of the two Regions (excluding global services)
C. AWS Config rules that delete resources in other Regions
D. A permissions boundary on every IAM role

**Q3.** Developers must be able to create IAM roles for their Lambda functions, but must never be able to create a role with more permissions than a defined set. What should be used?
A. An SCP
B. A permissions boundary required on any role developers create
C. IAM Access Analyzer
D. A resource-based policy on Lambda

**Q4.** Account A needs to read objects in an S3 bucket in Account B. Account A's users should keep their own permissions while doing so. What is the simplest solution?
A. Create an IAM user in Account B and share its keys
B. Add a bucket policy in Account B granting `s3:GetObject` to Account A's principals, and grant the same in Account A's identity policies
C. Enable S3 Cross-Region Replication to Account A
D. Use AWS RAM to share the bucket

**Q5.** A company has 40 AWS accounts and uses Microsoft Entra ID for employee identities. Employees need single sign-on to the AWS accounts with permissions based on job function. Which solution has the LEAST operational overhead?
A. Create IAM users in each account matching Entra ID users
B. IAM Identity Center connected to Entra ID, with permission sets assigned per group
C. A SAML provider and roles configured individually in each of the 40 accounts
D. Amazon Cognito user pools

**Q6.** A security engineer needs to confirm that no S3 bucket, KMS key or IAM role in the organization is accessible from outside the organization. Which service provides this?
A. AWS Config
B. Amazon GuardDuty
C. IAM Access Analyzer with the organization as the zone of trust
D. AWS Trusted Advisor

**Q7.** An S3 bucket policy must deny any request not made over HTTPS. Which condition key should be used?
A. `aws:SourceIp`
B. `aws:SecureTransport`
C. `s3:x-amz-server-side-encryption`
D. `aws:MultiFactorAuthPresent`

**Q8.** A company is building a new multi-account environment and wants a pre-configured landing zone with centralized logging, SSO and guardrails, set up with minimal effort. What should it use?
A. AWS Organizations alone
B. AWS Control Tower
C. AWS CloudFormation StackSets
D. AWS Service Catalog

**Q9.** A user has an identity policy allowing `ec2:*`. Their account's SCP allows only `ec2:Describe*` and `s3:*`. What can the user do?
A. All EC2 actions
B. Only `ec2:Describe*` actions
C. `ec2:Describe*` and all S3 actions
D. Nothing, because the policies conflict

**Q10.** A GitHub Actions pipeline deploys infrastructure to AWS using an IAM user's access keys stored as repository secrets. The security team wants to eliminate long-lived credentials. What should the architect recommend?
A. Rotate the keys every 30 days with Lambda
B. Store the keys in Secrets Manager instead
C. Configure GitHub as an OIDC identity provider in IAM and have the pipeline assume a role
D. Use the root user's keys with MFA

<details>
<summary><strong>Answers</strong></summary>

1. **B** — Roles give temporary credentials. Secrets Manager still means a long-term key exists (A); groups only hold users, never EC2 instances (D).
2. **B** — An SCP with `aws:RequestedRegion` applies to everyone, including root, in every member account. Config (C) is detective and destructive; per-account IAM (A) doesn't scale and can be changed by account admins.
3. **B** — Permissions boundaries are exactly the "delegate role creation safely" tool. SCPs (A) cap whole accounts, not specific developer-created roles.
4. **B** — Cross-account needs both sides to allow: the resource policy in B and identity policies in A. Callers keep their own permissions (unlike assuming a role). RAM (D) doesn't share S3 buckets.
5. **B** — Identity Center centralises SSO across all accounts with permission sets. C works but means 40 configurations.
6. **C** — Access Analyzer with an organization zone of trust flags anything shared outside the org.
7. **B** — `aws:SecureTransport = false` → Deny.
8. **B** — Control Tower builds the landing zone. Organizations alone (A) provides no guardrails or logging setup.
9. **B** — Effective permissions = intersection. The SCP allows S3, but the user's identity policy doesn't, so no S3. EC2 is limited to Describe by the SCP.
10. **C** — OIDC federation issues short-lived role credentials per run, with no stored keys. A and B still keep long-lived keys around; D is never right.

</details>

---

## 7. Flashcards (cover the right column)

| Prompt | Answer |
| --- | --- |
| Minimum AZs in a Region | 3 |
| Does an SCP grant permissions? | No — it only caps them |
| Does an SCP affect the management account? | No |
| Does an SCP affect a member account's root user? | Yes |
| Can IAM groups be nested? | No |
| What wins: explicit allow or explicit deny? | Explicit deny |
| Two policies on every role | Trust policy + permissions policy |
| Credential source for EC2 roles | Instance metadata (IMDSv2) via instance profile |
| Service for workforce SSO across accounts | IAM Identity Center |
| Service for a best-practice multi-account landing zone | Control Tower |
| Share subnets/Transit Gateway across accounts | AWS RAM |
| Condition key: only principals in my org | `aws:PrincipalOrgID` |
| Condition key: force HTTPS | `aws:SecureTransport` |
| Tool for last-accessed services per role | Access Advisor |
| Tool for externally shared resources | IAM Access Analyzer |

## 8. Before you close today

- [ ] Read sections 1–4
- [ ] Lab done and torn down
- [ ] 10 questions above + 20 bank questions on IAM/Organizations
- [ ] Misses logged in `miss-log.md`

**Tomorrow (Day 2):** EC2 — instance families, purchasing options, placement groups, EBS volume types, instance store, AMIs and snapshots.
