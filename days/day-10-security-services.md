# Day 10 — Security Services: KMS, Secrets Manager, WAF/Shield, GuardDuty & co, Cognito

**Domain weight:** almost entirely *Design Secure Architectures* (30% — the largest domain: "encrypt at rest", "rotate credentials", "protect from SQL injection / DDoS", "detect threats", "authenticate app users"). *Resilient* (26%) shows up through multi-Region keys and replicated secrets; *Cost* (20%) through Parameter Store vs Secrets Manager, S3 Bucket Keys and Shield Standard vs Advanced.
**Time plan:** Learn 90 min · Lab 30 min · Drill 45 min · Review 30 min.

By the end of today you should be able to:

- Explain envelope encryption and KMS key types well enough to answer any "who can decrypt / why AccessDenied / how to rotate / cross-Region / cross-account" question.
- Choose between KMS, CloudHSM, a KMS custom key store, ACM and AWS Private CA from one requirement sentence.
- Pick Secrets Manager or Parameter Store on rotation, cost and size, and wire an app to read secrets with a role.
- Place WAF, Shield Standard/Advanced, Firewall Manager and Network Firewall at the right layer for a given attack.
- Tell GuardDuty, Inspector, Macie, Security Hub, Detective and Security Lake apart instantly.
- Decide between Cognito user pools, identity pools and IAM Identity Center, and design "app users upload to their own S3 prefix".

---

## 1. The mental model: three questions, many services

Almost every security question on SAA is one of three shapes. Classify first, then pick.

```
                        ┌──────────────────────────────────────────────────────┐
  1. PROTECT DATA       │ KMS · CloudHSM · ACM / Private CA · Secrets Manager  │
     (at rest, in       │ · Parameter Store · Macie (find it)                  │
     transit, secrets)  └──────────────────────────────────────────────────────┘
                        ┌──────────────────────────────────────────────────────┐
  2. PROTECT THE EDGE   │ Shield (L3/L4 DDoS) · WAF (L7) · Firewall Manager    │
     / NETWORK          │ (central policy) · Network Firewall (VPC traffic)    │
                        └──────────────────────────────────────────────────────┘
                        ┌──────────────────────────────────────────────────────┐
  3. DETECT &           │ GuardDuty (threats) · Inspector (vulns) · Macie (PII)│
     RESPOND            │ · Security Hub (aggregate + posture) · Detective     │
                        │ (investigate) · Security Lake (store)                │
                        └──────────────────────────────────────────────────────┘
  + WHO IS THE USER?    Cognito (app end users) · IAM Identity Center (workforce)
```

> Exam habit: if two answers both "work", the qualifier decides. *Least operational overhead* → the managed, native feature (Secrets Manager native rotation, WAF managed rule groups, GuardDuty) over custom Lambda, self-run IDS on EC2 or third-party appliances.

---

## 2. AWS KMS (Key Management Service)

KMS creates and controls cryptographic keys inside AWS-managed HSMs. The key material **never leaves KMS unencrypted**; you send data (or data keys) to KMS and get ciphertext back. Every use is an API call — so it's **authorised by policy** and **logged in CloudTrail**. That's the whole value proposition: central control + audit.

### Key types (who owns and manages the key)

| Type | Example | Who manages | Key policy editable? | Rotation | See it in your account? | Cost |
| --- | --- | --- | --- | --- | --- | --- |
| **Customer managed key** | `alias/app-prod` | You | **Yes** | Optional automatic (default 1 year, configurable period) + on-demand | Yes | Monthly fee per key + requests (check current docs) |
| **AWS managed key** | `aws/s3`, `aws/ebs`, `aws/rds` | AWS on your behalf | **No** | Automatic, every year, can't change | Yes (read-only) | No monthly fee; requests charged |
| **AWS owned key** | DynamoDB/SQS default encryption | AWS, shared across accounts | No | AWS decides | **No** | Free |

Why it matters on the exam:

- **"Control who can use the key / separate admins from users / disable the key / share cross-account"** → customer managed key. You can't change the key policy on an AWS managed key, so you **can't share `aws/s3` with another account** (Day 5 trap again).
- **"Audit every use of the key in CloudTrail"** → customer managed or AWS managed (both log); AWS owned keys don't appear in your CloudTrail.

### Key specs

| Spec | Use | Notes |
| --- | --- | --- |
| **Symmetric** (AES-256-GCM, `SYMMETRIC_DEFAULT`) | Default; what S3/EBS/RDS etc. integrate with | Never leaves KMS; you must call KMS to use it |
| **Asymmetric** (RSA, ECC) | Sign/verify, or encrypt by parties **outside AWS** using the public key | Public key is downloadable; no automatic rotation |
| **HMAC** | Generate/verify message authentication codes (tokens) | Niche on SAA |

### Envelope encryption (why KMS scales)

KMS `Encrypt` accepts at most **4 KB** of plaintext. Real data is encrypted with a **data key**, and only the data key is protected by the KMS key.

```
ENCRYPT
  App ── GenerateDataKey(KeyId) ──► KMS
  App ◄── { Plaintext data key, Encrypted data key } ──
  App: encrypt 2 GB file locally with plaintext data key (AES-256)
  App: discard plaintext key; store ciphertext + encrypted data key together

DECRYPT
  App ── Decrypt(encrypted data key) ──► KMS   (policy check + CloudTrail log)
  App ◄── plaintext data key ──
  App: decrypt file locally, discard key
```

- This is what S3 SSE-KMS, EBS, RDS and the AWS Encryption SDK do for you.
- Exam trigger: *"encrypt files larger than 4 KB with KMS"* / *"reduce KMS calls"* → **envelope encryption with `GenerateDataKey`** (and for S3, **Bucket Keys**).
- `GenerateDataKeyWithoutPlaintext` — give the encrypted data key to a component that will only decrypt later.

### Key policies, IAM and grants — the access model

Every KMS key has exactly one **key policy** (a resource-based policy). Unlike most AWS resources, **IAM policies alone are not enough** unless the key policy lets them be used.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnableIAMPoliciesInThisAccount",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::111122223333:root" },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "KeyAdminsCannotUseKey",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::111122223333:role/SecurityKeyAdmin" },
      "Action": ["kms:Create*", "kms:Describe*", "kms:Enable*", "kms:List*", "kms:Put*",
                 "kms:Update*", "kms:Revoke*", "kms:Disable*", "kms:Get*", "kms:Delete*",
                 "kms:TagResource", "kms:UntagResource", "kms:ScheduleKeyDeletion",
                 "kms:CancelKeyDeletion"],
      "Resource": "*"
    },
    {
      "Sid": "AppMayUseKeyOnlyViaS3",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::111122223333:role/AppRole" },
      "Action": ["kms:Encrypt", "kms:Decrypt", "kms:GenerateDataKey*", "kms:DescribeKey"],
      "Resource": "*",
      "Condition": { "StringEquals": { "kms:ViaService": "s3.ap-south-1.amazonaws.com" } }
    },
    {
      "Sid": "PartnerAccountMayDecrypt",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::444455556666:root" },
      "Action": ["kms:Decrypt", "kms:DescribeKey"],
      "Resource": "*"
    }
  ]
}
```

Read it the exam way:

1. **Statement 1** (`account:root` as principal) is the *default* — it means "IAM policies in this account may grant access to this key". It does **not** mean "only the root user". Remove it and IAM policies stop working for this key.
2. **Separation of duties:** admins manage the key but can't encrypt/decrypt; apps use it but can't delete it.
3. **`kms:ViaService`** — key usable only when the call comes through a specific service (e.g. only via S3 or only via EBS).
4. **Cross-account:** both sides must allow — the key policy names account 444455556666 **and** an IAM policy in 444455556666 grants `kms:Decrypt` on this key ARN. (Same "both sides" rule as Day 1.)

**Grants** — temporary, programmatic delegations of key use (e.g. EBS/RDS create grants so they can use your key on your behalf). Recognise the word; you rarely configure them by hand.

### Rotation, deletion and lifecycle

| Action | Behaviour |
| --- | --- |
| **Automatic rotation** (customer managed, symmetric, KMS-generated material) | New backing key material on a schedule; **key ID/ARN unchanged**; old material kept so old ciphertext still decrypts. Nothing to re-encrypt. |
| **On-demand rotation** | Rotate now, without waiting for the schedule. |
| **Manual rotation** | Create a new key and **repoint an alias** (`alias/app-prod`) to it. Required for asymmetric keys, HMAC keys and imported key material. Apps must use the alias. |
| **Disable** | Reversible; key unusable immediately. Use when unsure. |
| **Schedule deletion** | Waiting period **7–30 days** (default 30); cancellable during the window. After deletion, data encrypted under it is **unrecoverable**. |
| **Imported key material (BYOK)** | You generate material and import it; you can set an expiry and delete material on demand. No automatic rotation. |

> Trap: *"Rotate the key yearly with the least effort and no application changes"* → enable **automatic rotation** on a customer managed key. *"Rotate imported key material"* → manual rotation with a new key + alias.

### Regions: KMS keys are Regional

- A key lives in one Region. Copying an encrypted EBS snapshot, RDS snapshot or AMI to another Region **re-encrypts with a key in the destination Region** (Day 6 recap).
- **Multi-Region keys** (`mrk-…` IDs): a primary key plus replicas in other Regions sharing the **same key material and key ID**. Ciphertext from Region A decrypts in Region B **without a cross-Region call**. Each replica has its own key policy. Use for: client-side encryption with DynamoDB global tables, multi-Region apps and DR where data moves between Regions. Not needed for services that re-encrypt on copy anyway.

### Custom key stores and CloudHSM

| Need | Answer |
| --- | --- |
| Standard managed encryption, integrated with AWS services | **KMS** (multi-tenant AWS-managed HSMs) |
| Single-tenant HSMs, **you** control users and keys; AWS has no access to key material; PKCS#11 / JCE / CNG APIs; SSL offload; Oracle TDE | **AWS CloudHSM** |
| KMS integrations (S3, EBS, RDS…) **but** key material in HSMs you control | **KMS custom key store backed by CloudHSM** |
| Key material must stay **outside AWS** in your own HSM | **KMS external key store (XKS)** |

CloudHSM facts: runs in your VPC (ENIs in your subnets); deploy a **cluster across ≥ 2 AZs** for HA (one HSM is a single point of failure); AWS manages hardware, **you** manage users/keys — lose the credentials and AWS can't recover your keys. FIPS validation levels: KMS and CloudHSM HSMs are both FIPS 140 Level 3 validated (check current docs for the exact standard/version); **single tenancy and customer-exclusive control** is the differentiator the exam uses.

### Encryption quick reference across services (recap from Days 2–6)

| Service | At rest | Gotcha |
| --- | --- | --- |
| EBS | KMS; **encryption by default** per Region setting | Encrypt an unencrypted volume: snapshot → copy with encryption → new volume |
| RDS/Aurora | KMS, chosen **at creation** | Encrypt existing DB: snapshot → encrypted copy → restore |
| S3 | SSE-S3 default; SSE-KMS / DSSE-KMS / SSE-C | `kms:Decrypt` needed to read SSE-KMS objects; Bucket Keys cut cost |
| DynamoDB | Always on (owned / AWS managed / customer managed) | Choose customer managed for key-policy control |
| EFS | KMS at creation | In transit: mount helper TLS |
| SQS/SNS | SSE with SQS-owned key or KMS | Producers/consumers need KMS permissions with a CMK |
| Client-side | AWS Encryption SDK / DynamoDB Encryption Client | "Encrypt before it reaches AWS" |

---

## 3. Certificates: ACM and AWS Private CA

**AWS Certificate Manager (ACM)** provisions TLS certificates for **integrated services**: ELB (ALB/NLB), CloudFront, API Gateway, App Runner, and a few others.

- **Public certificates** from ACM are free (check current docs) and **renew automatically** when DNS validation stays in place (DNS validation with a CNAME — preferred over email validation, which needs human action).
- Traditionally ACM public certificates can't be exported, so you can't install them directly on an EC2 web server; AWS added a paid *exportable* public certificate option in 2025. For exam purposes: **"TLS on EC2 with ACM" → terminate TLS on the ALB/NLB** (or use ACM for Nitro Enclaves).
- **CloudFront certificates must be in `us-east-1`**. ALB certificates must be in the ALB's Region.
- **Imported** (third-party) certificates: ACM does **not** renew them. Watch expiry with ACM's expiry events in **EventBridge** or the AWS Config managed rule `acm-certificate-expiration-check`.
- **AWS Private CA**: your own private certificate authority for internal TLS, mTLS between services, IoT devices. ACM can issue private certs from it for integrated services.

```
Client ──TLS(ACM cert)──► ALB ──HTTP or TLS──► targets
                         │
           "end-to-end encryption required" → re-encrypt to targets (HTTPS target group),
           or NLB with TCP pass-through and TLS on the instances
```

---

## 4. Secrets: Secrets Manager vs SSM Parameter Store

Both store values encrypted with KMS and let an app fetch them at runtime with an IAM role — so passwords never sit in code, AMIs or environment files.

| | **AWS Secrets Manager** | **SSM Parameter Store** |
| --- | --- | --- |
| Built for | Secrets with a lifecycle: DB creds, API keys | Configuration + simple secrets |
| **Automatic rotation** | **Yes** — native for RDS, Aurora, Redshift, DocumentDB; Lambda rotation function for anything else; *managed rotation* for some services | **No** built-in rotation (you'd build EventBridge + Lambda yourself) |
| Encryption | Always KMS | `SecureString` uses KMS; `String`/`StringList` are plaintext |
| **Cross-Region replication** | **Yes** (replica secrets for multi-Region apps/DR) | No |
| Resource-based policy (cross-account) | **Yes** | No (Advanced tier can be shared via RAM — niche) |
| Size | Up to 64 KB (check current docs) | Standard 4 KB · Advanced 8 KB |
| Extras | Generate random passwords; versions/staging labels (`AWSCURRENT`, `AWSPREVIOUS`) | Hierarchies (`/prod/app/db/host`), versions, **parameter policies** (expiry notifications — Advanced tier) |
| Cost | Per secret per month + API calls (check current docs) | **Standard tier free**; Advanced tier charged |

Decision rule:

- **"Rotate automatically"** / **"RDS credentials every 30 days"** / **"cross-Region secret"** → **Secrets Manager**.
- **"Store configuration values and some encrypted strings at the lowest cost, no rotation"** → **Parameter Store (Standard, SecureString)**.
- Parameter Store can *reference* a Secrets Manager secret (`/aws/reference/secretsmanager/<name>`) — handy when an app already reads config from Parameter Store.
- RDS and Aurora can **manage the master user password in Secrets Manager** for you (create/rotate with no Lambda to write) — the least-overhead answer when offered.

How rotation works (single-user strategy, simplified):

```
EventBridge schedule ─► Secrets Manager ─► rotation Lambda (in VPC, reaches DB)
                                               1 createSecret   new password as AWSPENDING
                                               2 setSecret      ALTER USER in the DB
                                               3 testSecret     log in with AWSPENDING
                                               4 finishSecret   move AWSCURRENT label
Apps always read AWSCURRENT → no redeploy needed.
```

> Traps:
> - The rotation Lambda must reach both the DB (same VPC/SG rules) **and** the Secrets Manager endpoint — in private subnets without NAT, add an **interface VPC endpoint** for Secrets Manager.
> - Reading a secret encrypted with a customer managed key needs `secretsmanager:GetSecretValue` **and** `kms:Decrypt` on that key.
> - ECS: secrets injected into containers are fetched by the **task execution role**; the app's own AWS calls use the **task role** (Day 9).
> - Lambda: the **Parameters and Secrets Lambda Extension** caches values to reduce calls/cost.

```bash
# App code / user data: fetch at runtime with the instance role
aws secretsmanager get-secret-value --secret-id prod/app/db --query SecretString --output text
aws ssm get-parameter --name /prod/app/db/host --with-decryption --query Parameter.Value --output text
aws ssm get-parameters-by-path --path /prod/app/ --recursive --with-decryption
```

---

## 5. Edge and network protection: Shield, WAF, Firewall Manager, Network Firewall

### Layers

```
Internet
   │  L3/L4 floods (SYN, UDP reflection)  ──► AWS Shield (Standard: automatic, free)
   ▼
CloudFront / Route 53 / Global Accelerator   ◄── absorb at the edge (huge capacity)
   │  L7: SQLi, XSS, bots, HTTP floods, bad IPs/countries ──► AWS WAF web ACL
   ▼
ALB / API Gateway / AppSync / Cognito …  (WAF attaches here too)
   │
VPC: Network Firewall (stateful/IDS/IPS, domain allow-lists) · NACLs · Security groups
   ▼
Instances / containers
```

### AWS Shield

| | **Shield Standard** | **Shield Advanced** |
| --- | --- | --- |
| Cost | Free, automatic, every account | Paid subscription per organization + data transfer fees (check current docs — it's significant) |
| Protects | L3/L4 attacks on all AWS resources, strongest at CloudFront/Route 53 | CloudFront, Route 53, Global Accelerator, ELB, EC2 Elastic IPs |
| Extras | — | **24/7 Shield Response Team (SRT)**, **DDoS cost protection** (credits for scaling charges during an attack), advanced detection & L7 auto-mitigation with WAF, attack diagnostics/visibility, WAF fees for protected resources included, proactive engagement via Route 53 health checks |

> Exam cue: *"DDoS response team"*, *"protection against cost spikes from a DDoS"*, *"near real-time attack visibility"* → **Shield Advanced**. Plain *"protect against common DDoS"* with cost-sensitivity → Shield Standard (already on) + CloudFront + WAF rate-based rules.

DDoS-resilient architecture (the AWS best-practice answer): **Route 53 + CloudFront (+ WAF) in front, ALB/Auto Scaling behind, origins not directly reachable** (OAC, CloudFront managed prefix list or VPC origins — Day 9), Shield Advanced if the business needs SRT/cost protection.

### AWS WAF

A **web ACL** holds ordered **rules**; each rule matches requests and takes an action (**Allow, Block, Count, CAPTCHA, Challenge**). Requests not matched get the **default action**.

| Rule building block | Use |
| --- | --- |
| **AWS Managed Rules** rule groups | Core rule set (OWASP-style), **SQL database (SQLi)**, known bad inputs, Linux/Windows/PHP, IP reputation, anonymous IP list — fastest, least effort |
| **Bot Control**, **Account Takeover Prevention (ATP)**, **Account Creation Fraud Prevention (ACFP)** | Paid managed rule groups for bots, credential stuffing, fake sign-ups |
| **Rate-based rule** | Block an IP (or other key) exceeding N requests per window → HTTP floods, brute force |
| **IP set** | Allow/deny lists of CIDRs |
| **Geo match** | Block/allow by country (also feeds labels) |
| **String/regex match, size constraint** | Match headers, URI, query string, body |
| Marketplace rule groups | Third-party vendor rules |

```json
{
  "Name": "RateLimitLogin",
  "Priority": 1,
  "Statement": {
    "RateBasedStatement": {
      "Limit": 500,
      "AggregateKeyType": "IP",
      "ScopeDownStatement": {
        "ByteMatchStatement": {
          "FieldToMatch": { "UriPath": {} },
          "SearchString": "/login",
          "PositionalConstraint": "STARTS_WITH",
          "TextTransformations": [{ "Priority": 0, "Type": "NONE" }]
        }
      }
    }
  },
  "Action": { "Block": {} },
  "VisibilityConfig": { "SampledRequestsEnabled": true, "CloudWatchMetricsEnabled": true, "MetricName": "RateLimitLogin" }
}
```

**Where WAF attaches:** CloudFront, **ALB**, **API Gateway (REST)**, AppSync, Cognito user pools, App Runner, Verified Access, Amplify. **Not** NLB, not EC2 directly, not Global Accelerator.
**Scope:** web ACLs for CloudFront are *global* (managed via `us-east-1`); for everything else they're Regional, in the resource's Region.
**Logging:** to CloudWatch Logs, S3 or Amazon Data Firehose. Start new rules in **Count** mode to avoid blocking real users.

> Traps:
> - *"Block SQL injection / cross-site scripting"* → **WAF**, never security groups/NACLs (they don't see HTTP content).
> - *"Block a few malicious IPs at the ALB"* → WAF IP set. NACL deny also works at the subnet, but with CloudFront in front the ALB sees CloudFront IPs — block at **WAF on CloudFront**.
> - *"App behind an NLB needs L7 filtering"* → put WAF on CloudFront in front, or switch to ALB.
> - *"Restrict by country"* → CloudFront **geo restriction** (simple, whole distribution) or **WAF geo match** (granular, combine with other conditions, works on ALB too).

### AWS Firewall Manager

Central policy across an **AWS Organization**: WAF rules, Shield Advanced protections, **security groups** (audit/clean-up), Network Firewall, Route 53 Resolver DNS Firewall, third-party firewalls. New accounts and new resources (e.g. a new ALB) get the policy **automatically**. Prerequisites: Organizations, a Firewall Manager administrator account, AWS Config enabled.

> *"Apply the same WAF rules to every ALB in all 60 accounts, including future ones"* → **Firewall Manager**.

### AWS Network Firewall (recap from Day 4)

Managed, stateful VPC firewall with IDS/IPS (Suricata-compatible rules), **domain-name filtering** for egress, deployed in a firewall subnet with route-table steering (often centralised via Transit Gateway). Use when the requirement is **inspect/filter VPC traffic** (east–west or egress), not HTTP app-layer attacks on a web front end. **Route 53 Resolver DNS Firewall** blocks DNS lookups of malicious domains from the VPC — lighter-weight egress control.

---

## 6. Detect and respond: the "which security service?" family

These are the most confusable names on the exam. Learn the one-verb summary for each.

| Service | One-verb job | Looks at | Classic exam phrase |
| --- | --- | --- | --- |
| **Amazon GuardDuty** | **Detect threats** | CloudTrail management events, VPC Flow Logs, DNS logs (no need to enable these yourself) + optional protection plans: S3 data events, EKS audit logs, Runtime Monitoring (EC2/ECS/EKS), Malware Protection (EBS volumes, S3 uploads), RDS login activity, Lambda network activity | "Compromised instance", "crypto-mining", "unusual API calls", "communication with known malicious IPs", "credential exfiltration" |
| **Amazon Inspector** | **Find vulnerabilities** | EC2 (via SSM agent or agentless scanning), **ECR images**, **Lambda functions** — CVEs, network reachability, code vulns | "Continuously scan for software vulnerabilities / unpatched packages / CVEs in container images" |
| **Amazon Macie** | **Find sensitive data** | **S3 objects** — ML + patterns for PII, financial data, credentials; also flags public/unencrypted buckets | "Discover PII / credit card numbers in S3" |
| **AWS Security Hub** | **Aggregate + score posture** | Findings from GuardDuty, Inspector, Macie, Firewall Manager, partners; runs best-practice checks (AWS Foundational Security Best Practices, CIS, PCI DSS) via AWS Config | "Single pane of glass", "check accounts against CIS benchmark", "aggregate findings across accounts/Regions" |
| **Amazon Detective** | **Investigate root cause** | Builds a behaviour graph from CloudTrail, Flow Logs, GuardDuty findings, EKS logs | "Investigate / visualise the root cause of a GuardDuty finding" |
| **Amazon Security Lake** | **Store & normalise security data** | Central S3 data lake in OCSF format across accounts/Regions | "Centralise security logs in a standard schema for analytics/SIEM" |
| **AWS Config** (Day 11) | **Record config + compliance rules** | Resource configuration history | "Was this SG ever open to 0.0.0.0/0?", "detect non-compliant resources" |
| **AWS CloudTrail** (Day 11) | **Record API calls** | Who did what, when | "Who deleted the bucket?" |
| **AWS Audit Manager** | **Collect audit evidence** | Maps evidence to frameworks | "Prepare for an audit continuously" |
| **AWS Artifact** | **Download compliance reports** | AWS's SOC/PCI/ISO reports and agreements (e.g. BAA) | "Provide AWS's PCI compliance report to auditors" |
| **Trusted Advisor** | **Best-practice checks** | Cost, security, fault tolerance, performance, limits | "Open SGs, root MFA, quota checks" (full checks need Business+ Support) |

> Note on naming: in 2025 AWS introduced a new unified Security Hub and renamed the original posture-check service **AWS Security Hub CSPM**. Exam questions describe the job ("aggregate findings and run best-practice checks") — the answer is still Security Hub.

### Organization-wide operation

GuardDuty, Inspector, Macie, Security Hub and Detective all support a **delegated administrator** account in Organizations: enable once, auto-enable for new member accounts, see all findings centrally. That's the "multi-account, least overhead" answer — not enabling per account by hand.

### Automated response pattern (memorise)

```
GuardDuty / Inspector / Macie / Security Hub finding
          │  (all publish to EventBridge)
          ▼
   EventBridge rule (match finding type / severity)
          │
          ├─► SNS → email/Slack/pager
          ├─► Lambda → isolate instance (swap SG to "quarantine"), revoke keys, block IP in WAF
          └─► Step Functions / SSM Automation runbook
```

> *"Automatically remediate when GuardDuty detects X"* → **EventBridge rule → Lambda/SSM Automation**. GuardDuty itself only detects; it doesn't block.

---

## 7. Amazon Cognito: identity for your app's users

Cognito is for **customers / end users of your application** (web and mobile). Workforce users signing into AWS accounts → IAM Identity Center (Day 1).

### User pools vs identity pools

| | **User pool** | **Identity pool** (federated identities) |
| --- | --- | --- |
| Answers | **Who are you?** (authentication) | **What AWS resources may you touch?** (authorisation to AWS) |
| Gives you | **JWTs** (ID, access, refresh tokens) | **Temporary AWS credentials** (via STS) mapped to IAM roles |
| Features | User directory, sign-up/sign-in, MFA, password policies, email/phone verification, **social (Google, Apple, Facebook, Amazon) and SAML/OIDC federation**, managed login/hosted UI, Lambda triggers (pre sign-up, post confirmation, pre token generation…), adaptive/threat protection | Authenticated role vs **guest (unauthenticated) role**, rule-based role mapping, accepts tokens from user pools, social IdPs, SAML, OIDC, developer-authenticated identities |
| Typical integration | **ALB authenticate action**, **API Gateway Cognito authorizer**, AppSync | App calls **S3, DynamoDB** etc. directly with the temp creds |

```
          (1) sign in (email+pw, Google, SAML)
 Mobile ───────────────────────────────────► Cognito USER POOL ──► JWT tokens
  app  ◄────────────────────────────────────
   │
   ├─(2a) call your API with JWT ─► API Gateway (Cognito authorizer) / ALB ─► backend
   │
   └─(2b) exchange JWT ─► Cognito IDENTITY POOL ─► STS ─► temp AWS creds (IAM role)
                                                          │
                                         (3) PutObject directly to S3 prefix  s3://bucket/${sub}/
```

**Per-user isolation with IAM policy variables** — the role assumed via the identity pool:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::photo-app-uploads/${cognito-identity.amazonaws.com:sub}/*"
    },
    {
      "Effect": "Allow",
      "Action": ["dynamodb:GetItem", "dynamodb:PutItem", "dynamodb:Query"],
      "Resource": "arn:aws:dynamodb:ap-south-1:111122223333:table/UserData",
      "Condition": {
        "ForAllValues:StringEquals": {
          "dynamodb:LeadingKeys": ["${cognito-identity.amazonaws.com:sub}"]
        }
      }
    }
  ]
}
```

Each user can only touch their own S3 prefix and DynamoDB items whose partition key equals their identity ID — no backend code needed.

> Traps:
> - "Sign up / sign in / MFA / social login for app users" → **user pool**. "App needs AWS credentials to call S3/DynamoDB directly" → **identity pool**. Both together is common.
> - "Guest users can browse but not upload" → identity pool **unauthenticated role** with read-only permissions.
> - "Authenticate users at the load balancer without changing app code" → **ALB listener rule with `authenticate-cognito`** (or `authenticate-oidc`).
> - "Employees need SSO into AWS console/accounts" → **IAM Identity Center**, not Cognito.
> - "Users already exist in on-prem Microsoft AD and need to sign into AWS-hosted apps" → Directory Service (AD Connector / Managed Microsoft AD), or SAML federation into a Cognito user pool for a customer-facing app.

### Identity: one table to separate them

| Who is signing in? | To what? | Answer |
| --- | --- | --- |
| Employees | AWS accounts/console/CLI | IAM Identity Center |
| Employees | Corporate internal web apps, no VPN | AWS Verified Access (zero-trust, policy-based) |
| Customers | Your web/mobile app | Cognito user pool |
| Customers' app | AWS APIs directly | Cognito identity pool |
| Workload (EC2/Lambda/ECS) | AWS APIs | IAM role |
| External CI (GitHub Actions) | AWS APIs | IAM OIDC provider + role |
| Windows workloads | AD domain in AWS | AWS Managed Microsoft AD |

---

## 8. Exam traps roundup

1. **AccessDenied reading an SSE-KMS object / secret / encrypted SQS message** → add `kms:Decrypt` (key policy + IAM), not more S3/SQS permissions.
2. **Share encrypted data or snapshots cross-account** → must use a **customer managed key** and grant the other account in its key policy; AWS managed keys can't be shared.
3. **Copy encrypted snapshot/AMI to another Region** → choose a key **in the destination Region** (or use multi-Region keys for app-level ciphertext).
4. **Data > 4 KB with KMS** → envelope encryption (`GenerateDataKey`).
5. **Deleted KMS key** = data gone; prefer **disable**. Deletion waits 7–30 days.
6. **"Rotate" with no app change** → automatic key rotation (KMS) / Secrets Manager rotation (credentials). Parameter Store does not rotate.
7. **SQLi/XSS** → WAF; **SYN/UDP floods** → Shield; **24/7 DDoS team + cost protection** → Shield Advanced.
8. **WAF can't attach to NLB**; CloudFront WAF lives in the global scope (`us-east-1`), same as CloudFront's ACM cert.
9. **Many accounts, same firewall rules, future accounts too** → Firewall Manager.
10. **GuardDuty detects, doesn't block**; Inspector = CVEs; Macie = PII in S3; Detective = root-cause investigation; Security Hub = aggregate + posture checks.
11. **GuardDuty doesn't need you to turn on Flow Logs/DNS logs** — it reads them from independent streams.
12. **Cognito user pool = authentication (JWT); identity pool = AWS credentials.**
13. **CloudHSM single HSM = no HA** → cluster across AZs.
14. **ACM public certs auto-renew only with valid DNS (or completed email) validation**; imported certs never auto-renew.

---

## 9. Exam decision table

| If the question says… | Pick |
| --- | --- |
| Encrypt data at rest with full control of the key policy, audit, disable | **KMS customer managed key** |
| Encrypt, least effort, no key management | AWS managed key / service default encryption |
| Encrypt large files/objects with KMS | **Envelope encryption** (`GenerateDataKey`) |
| Rotate the encryption key yearly, no app change | **Automatic rotation** on a customer managed key |
| Ciphertext must decrypt in another Region without re-encrypting | **KMS multi-Region keys** |
| Dedicated single-tenant HSM, customer-exclusive key control, PKCS#11 | **CloudHSM** (cluster in ≥ 2 AZs) |
| AWS service integration but keys in HSMs you control | **KMS custom key store (CloudHSM)** |
| Key material must stay outside AWS | **KMS external key store** |
| Restrict key use to calls coming through one service | Key policy condition **`kms:ViaService`** |
| Free public TLS certificate with auto-renewal on ALB/CloudFront | **ACM** (CloudFront → cert in `us-east-1`) |
| Internal TLS / mTLS / private certificates | **AWS Private CA** |
| Alert before an imported certificate expires | ACM expiry events in **EventBridge** / Config rule |
| DB credentials rotated automatically | **Secrets Manager** (native RDS rotation / RDS-managed master password) |
| Secret replicated to a DR Region | **Secrets Manager replication** |
| Config values + encrypted strings, free, no rotation | **Parameter Store** (Standard, SecureString) |
| Protect web app from SQLi, XSS, bad bots | **AWS WAF** (managed rule groups / Bot Control) |
| Throttle IPs sending too many requests | **WAF rate-based rule** |
| Block requests from certain countries at the ALB | **WAF geo match** (CloudFront geo restriction if CloudFront) |
| Basic DDoS protection, no extra cost | **Shield Standard** (automatic) + CloudFront |
| DDoS response team, cost protection, advanced visibility | **Shield Advanced** |
| Same WAF/SG/Shield policy across all accounts incl. new ones | **Firewall Manager** |
| Stateful inspection / IDS-IPS / domain filtering of VPC traffic | **AWS Network Firewall** |
| Block DNS lookups of malicious domains from VPCs | **Route 53 Resolver DNS Firewall** |
| Detect compromised instances, crypto-mining, anomalous API calls | **GuardDuty** |
| Scan EC2, ECR images and Lambda for CVEs continuously | **Inspector** |
| Find PII/sensitive data in S3 | **Macie** |
| Aggregate findings + compliance checks (CIS, FSBP, PCI) across accounts | **Security Hub** |
| Investigate the root cause of a finding | **Detective** |
| Central security data lake in a standard schema | **Security Lake** |
| Auto-remediate a security finding | **EventBridge rule → Lambda / SSM Automation** |
| Download AWS compliance reports for auditors | **AWS Artifact** |
| App users sign up/sign in, MFA, social login | **Cognito user pool** |
| App users get temporary AWS credentials to S3/DynamoDB | **Cognito identity pool** |
| Each user may access only their own S3 prefix / DynamoDB items | Identity pool role with **`${cognito-identity.amazonaws.com:sub}`** / `dynamodb:LeadingKeys` |
| Authenticate users at the ALB without app changes | **ALB `authenticate-cognito` / `authenticate-oidc` action** |
| Employees SSO into AWS accounts | **IAM Identity Center** |
| Employees reach internal apps without a VPN | **AWS Verified Access** |

---

## 10. Lab (30 min, free tier)

Goal: feel envelope encryption, see a KMS permission failure, store secrets two ways, generate GuardDuty sample findings, and get a JWT from a Cognito user pool. Run CLI steps from **CloudShell** in `ap-south-1`.

Cost notes (check current pricing): a customer managed KMS key has a small prorated monthly fee — schedule deletion at teardown. Secrets Manager offers a free trial period per new secret. GuardDuty has a free trial for new accounts. Cognito user pools have a free monthly-active-user allowance. **WAF and Shield Advanced are not free** — WAF is a read-only walkthrough today.

### Console + CLI steps

1. **KMS → Customer managed keys → Create key**: Symmetric, Encrypt and decrypt. Alias `saa-day10`. Key administrators: your user/role. Key users: your user/role. Finish, then open **Key policy** and find the `account:root` statement and the admin/user statements from section 2. Open **Key rotation** → enable automatic rotation and note the rotation period field.
2. **Direct encrypt/decrypt (≤ 4 KB):**
   ```bash
   echo "day 10 secret" > small.txt
   aws kms encrypt --key-id alias/saa-day10 --plaintext fileb://small.txt \
     --query CiphertextBlob --output text | base64 -d > small.enc
   aws kms decrypt --ciphertext-blob fileb://small.enc \
     --query Plaintext --output text | base64 -d      # no key ID needed: it's in the ciphertext
   ```
3. **Envelope encryption (any size):**
   ```bash
   head -c 5000000 /dev/urandom > big.bin                         # 5 MB — too big for kms encrypt
   aws kms generate-data-key --key-id alias/saa-day10 --key-spec AES_256 > dk.json
   PLAIN=$(jq -r .Plaintext dk.json); jq -r .CiphertextBlob dk.json | base64 -d > dk.enc
   openssl enc -aes-256-cbc -pbkdf2 -in big.bin -out big.bin.enc -pass pass:"$PLAIN"
   unset PLAIN; rm dk.json                                        # keep only big.bin.enc + dk.enc
   # decrypt later:
   PLAIN=$(aws kms decrypt --ciphertext-blob fileb://dk.enc --query Plaintext --output text)
   openssl enc -d -aes-256-cbc -pbkdf2 -in big.bin.enc -out big.out -pass pass:"$PLAIN"
   cmp big.bin big.out && echo "envelope round-trip OK"
   ```
   (The `openssl` passphrase trick is a demo; the AWS Encryption SDK does this properly.) Open **CloudTrail → Event history**, filter *Event source = kms.amazonaws.com* → see `GenerateDataKey` and `Decrypt` events. That audit trail is why KMS exists.
4. **See KMS block access:**
   ```bash
   aws kms disable-key --key-id alias/saa-day10
   aws kms decrypt --ciphertext-blob fileb://small.enc   # DisabledException — data unreadable
   aws kms enable-key --key-id alias/saa-day10           # reversible, unlike deletion
   ```
   Every service using this key (S3, SSM, Secrets Manager) would fail the same way — and the error names KMS, not S3/SSM. That's the "AccessDenied on an encrypted object → check KMS" reflex.
5. **Parameter Store:**
   ```bash
   aws ssm put-parameter --name /saa/day10/db/host --type String --value db.internal
   aws ssm put-parameter --name /saa/day10/db/password --type SecureString \
     --key-id alias/saa-day10 --value 'N0t-a-real-pw!'
   aws ssm get-parameter --name /saa/day10/db/password                       # ciphertext
   aws ssm get-parameter --name /saa/day10/db/password --with-decryption     # plaintext
   aws ssm get-parameters-by-path --path /saa/day10 --recursive --with-decryption
   ```
6. **Secrets Manager:**
   ```bash
   aws secretsmanager create-secret --name saa/day10/db --kms-key-id alias/saa-day10 \
     --secret-string '{"username":"admin","password":"N0t-a-real-pw!"}'
   aws secretsmanager get-secret-value --secret-id saa/day10/db --query SecretString --output text
   ```
   In the console, open the secret → **Rotation** → look at the options (RDS native rotation vs Lambda) and **Replicate secret** → note the Region picker. Don't enable rotation (no DB today).
7. **GuardDuty:** GuardDuty → *Get started* → **Enable**. Then *Settings* → **Generate sample findings**. Browse findings: open one `CryptoCurrency:EC2/BitcoinTool.B!DNS` and one `UnauthorizedAccess:IAMUser/...`. Note severity, resource, and that each finding has an EventBridge-ready JSON. Look at the **Protection plans** list (S3, EKS, Runtime Monitoring, Malware Protection, RDS, Lambda).
8. **Cognito:** Cognito → *Create user pool* (the console may call it "Set up resources for your application") → app type *Single-page application* → sign-in with **email**. Then from CloudShell:
   ```bash
   POOL=<user-pool-id>; CLIENT=<app-client-id>
   aws cognito-idp update-user-pool-client --user-pool-id $POOL --client-id $CLIENT \
     --explicit-auth-flows ALLOW_USER_PASSWORD_AUTH ALLOW_REFRESH_TOKEN_AUTH
   aws cognito-idp admin-create-user --user-pool-id $POOL --username learner@example.com \
     --message-action SUPPRESS
   aws cognito-idp admin-set-user-password --user-pool-id $POOL \
     --username learner@example.com --password 'Day10-Passw0rd!' --permanent
   aws cognito-idp initiate-auth --client-id $CLIENT --auth-flow USER_PASSWORD_AUTH \
     --auth-parameters USERNAME=learner@example.com,PASSWORD='Day10-Passw0rd!' \
     --query AuthenticationResult.IdToken --output text | cut -d. -f2 | base64 -d 2>/dev/null; echo
   ```
   You'll see the JWT payload: `sub`, `email`, `iss` (your pool), `aud` (your client). That `sub` is what an API Gateway authorizer or ALB validates. (If `update-user-pool-client` resets other settings in your console-created client, that's fine for a lab.)
9. **WAF walkthrough (no create):** WAF & Shield → *Create web ACL* → pick *Regional* and note the resource types list (ALB, API Gateway, AppSync, Cognito…; **no NLB**). Click *Add managed rule groups* and read the AWS managed list (Core rule set, SQL database, Known bad inputs, IP reputation, Bot Control). Click *Add my own rules* → *Rate-based rule*. **Cancel** before creating.

### Terraform variant

```hcl
terraform {
  required_providers { aws = { source = "hashicorp/aws" } }
}
provider "aws" { region = "ap-south-1" }

data "aws_caller_identity" "me" {}

resource "aws_kms_key" "lab" {
  description             = "SAA Day 10 lab key"
  enable_key_rotation     = true
  deletion_window_in_days = 7
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "EnableIAMPolicies"
      Effect    = "Allow"
      Principal = { AWS = "arn:aws:iam::${data.aws_caller_identity.me.account_id}:root" }
      Action    = "kms:*"
      Resource  = "*"
    }]
  })
}

resource "aws_kms_alias" "lab" {
  name          = "alias/saa-day10-tf"
  target_key_id = aws_kms_key.lab.key_id
}

resource "aws_ssm_parameter" "db_password" {
  name   = "/saa/day10tf/db/password"
  type   = "SecureString"
  key_id = aws_kms_key.lab.arn
  value  = "N0t-a-real-pw!" # lab only — never commit real secrets to state/code
}

resource "aws_secretsmanager_secret" "db" {
  name                    = "saa/day10tf/db"
  kms_key_id              = aws_kms_key.lab.arn
  recovery_window_in_days = 0 # delete immediately on destroy (lab only)
}

resource "aws_secretsmanager_secret_version" "db" {
  secret_id     = aws_secretsmanager_secret.db.id
  secret_string = jsonencode({ username = "admin", password = "N0t-a-real-pw!" })
}

resource "aws_guardduty_detector" "this" {
  enable = true
}

resource "aws_cognito_user_pool" "app" {
  name                     = "saa-day10-tf"
  username_attributes      = ["email"]
  auto_verified_attributes = ["email"]
  password_policy {
    minimum_length    = 12
    require_numbers   = true
    require_symbols   = true
    require_uppercase = true
    require_lowercase = true
  }
}

resource "aws_cognito_user_pool_client" "spa" {
  name                = "spa"
  user_pool_id        = aws_cognito_user_pool.app.id
  generate_secret     = false
  explicit_auth_flows = ["ALLOW_USER_PASSWORD_AUTH", "ALLOW_REFRESH_TOKEN_AUTH"]
}

output "kms_key_arn"   { value = aws_kms_key.lab.arn }
output "user_pool_id"  { value = aws_cognito_user_pool.app.id }
output "client_id"     { value = aws_cognito_user_pool_client.spa.id }
output "detector_id"   { value = aws_guardduty_detector.this.id }
```

```bash
terraform init && terraform apply
aws guardduty create-sample-findings --detector-id "$(terraform output -raw detector_id)"
aws ssm get-parameter --name /saa/day10tf/db/password --with-decryption --query Parameter.Value --output text
```

> If GuardDuty is already enabled in the Region from the console steps, `aws_guardduty_detector` fails (one detector per Region) — skip that resource or import it.

### Teardown

- **Console/CLI:**
  ```bash
  aws secretsmanager delete-secret --secret-id saa/day10/db --force-delete-without-recovery
  aws ssm delete-parameters --names /saa/day10/db/host /saa/day10/db/password
  aws kms schedule-key-deletion --key-id alias/saa-day10 --pending-window-in-days 7
  aws guardduty delete-detector --detector-id "$(aws guardduty list-detectors --query 'DetectorIds[0]' --output text)"
  aws cognito-idp delete-user-pool --user-pool-id $POOL
  rm -f small.* big.* dk.enc
  ```
  (If the user pool has a domain configured, delete the domain first. If you'd rather keep GuardDuty during its free trial, **suspend** it instead — but set a reminder before the trial ends.)
- **Terraform:** `terraform destroy` (the KMS key goes into a 7-day pending-deletion window — that's expected).
- Check **Billing → Bills** tomorrow.

---

## 11. Practice questions

Answer all ten before opening the answers.

**Q1.** An application must encrypt 2 GB video files before storing them on an EBS-backed file system. The security team requires that keys be managed in AWS KMS and that every key use be logged. What should the application do?
A. Call `kms:Encrypt` on each file with a customer managed key
B. Call `kms:GenerateDataKey`, encrypt each file locally with the plaintext data key, and store the encrypted data key with the file
C. Export the KMS key material and use it with OpenSSL
D. Use an AWS owned key with `kms:Encrypt`

**Q2.** A company runs an application on EC2 that connects to an Amazon RDS for PostgreSQL database. Database credentials are currently stored in a configuration file. The company wants credentials rotated automatically every 30 days with the LEAST operational overhead. What should a solutions architect do?
A. Store credentials in SSM Parameter Store SecureString and write a Lambda function, triggered by EventBridge, to rotate them
B. Store credentials in AWS Secrets Manager with automatic rotation enabled, and have the application retrieve the secret at runtime using its instance role
C. Store credentials in an encrypted S3 object and rotate them with a cron job on the instance
D. Encrypt the configuration file with KMS and rotate the KMS key every 30 days

**Q3.** A company needs to store about 200 configuration values for its microservices, including a handful of API tokens that must be encrypted. The values change rarely and do not need rotation. The company wants the MOST cost-effective solution. What should it use?
A. AWS Secrets Manager for all values
B. SSM Parameter Store standard parameters, using SecureString for the tokens
C. Amazon DynamoDB with encryption using a customer managed key
D. AWS AppConfig with Secrets Manager for each value

**Q4.** A public web application runs on EC2 instances behind an Application Load Balancer. The company has seen SQL injection attempts and a flood of requests from a small set of IP addresses against the login page. What should a solutions architect do with the LEAST operational effort?
A. Add security group rules that deny the attacking IP addresses
B. Associate an AWS WAF web ACL with the ALB using the AWS managed SQL database rule group and a rate-based rule scoped to the login path
C. Deploy a third-party IDS on EC2 instances in front of the ALB
D. Enable AWS Shield Standard on the ALB

**Q5.** A financial services company's public applications run behind CloudFront and Route 53. It has suffered DDoS attacks and wants access to a 24/7 team of DDoS experts during attacks and protection against the scaling charges that an attack causes. What should it use?
A. AWS Shield Standard
B. AWS WAF with rate-based rules
C. AWS Shield Advanced
D. Amazon GuardDuty

**Q6.** A security team wants to be alerted when any EC2 instance in the account communicates with IP addresses associated with cryptocurrency mining, and wants the instance automatically isolated. Which solution meets these requirements with the LEAST effort?
A. Enable VPC Flow Logs and write Athena queries every hour
B. Enable Amazon GuardDuty and create an EventBridge rule on the finding type that invokes a Lambda function to replace the instance's security groups with a quarantine group
C. Enable Amazon Inspector and act on its findings manually
D. Deploy AWS Network Firewall with Suricata rules and review alerts daily

**Q7.** A company stores millions of customer documents in several S3 buckets. Auditors require the company to identify which objects contain personally identifiable information (PII) such as passport numbers and credit card numbers. What should a solutions architect recommend?
A. Amazon Inspector
B. Amazon Macie
C. Amazon Detective
D. AWS Security Hub

**Q8.** A mobile photo-sharing app lets users sign in with their Google accounts. After signing in, each user must upload photos directly to an S3 bucket, and must only be able to read and write objects under their own prefix. Which solution meets these requirements?
A. Create an IAM user for each app user and embed the access keys in the app
B. Use an Amazon Cognito user pool with Google as an identity provider, and a Cognito identity pool that issues temporary credentials for an IAM role whose policy restricts access to `${cognito-identity.amazonaws.com:sub}/*`
C. Use IAM Identity Center with Google Workspace as the identity source
D. Generate an S3 presigned URL for the whole bucket when the app starts and cache it on the device

**Q9.** A company must encrypt data in Amazon S3 and Amazon EBS using AWS KMS integrations, but regulators require that the key material be generated and stored in single-tenant HSMs that the company exclusively controls. What should a solutions architect do?
A. Use AWS managed keys with automatic rotation
B. Create a KMS custom key store backed by an AWS CloudHSM cluster and create KMS keys in it
C. Import key material into a standard KMS customer managed key
D. Use SSE-C with keys stored in Secrets Manager

**Q10.** A company uses DynamoDB global tables in two Regions. The application performs client-side encryption of sensitive attributes before writing. Items written in one Region must be decryptable by the application in the other Region, even if the first Region is unavailable. What should the company use?
A. A single-Region customer managed key, called cross-Region by the application
B. AWS managed key `aws/dynamodb` in each Region
C. A KMS multi-Region key with a replica in the second Region
D. Two independent customer managed keys, one per Region, with the same alias

<details>
<summary><strong>Answers</strong></summary>

1. **B** — `kms:Encrypt` is limited to 4 KB, so large files need envelope encryption; each `GenerateDataKey`/`Decrypt` is logged in CloudTrail. KMS key material can't be exported (C).
2. **B** — Secrets Manager has native RDS rotation and the app reads `AWSCURRENT` at runtime with its role. A works but means building and maintaining your own rotation. Rotating the KMS key (D) doesn't change the DB password.
3. **B** — Standard parameters are free and SecureString encrypts the tokens with KMS. Secrets Manager (A) charges per secret and its rotation isn't needed.
4. **B** — WAF on the ALB blocks SQLi with a managed rule group and throttles floods with a rate-based rule. Security groups (A) have no deny rules and can't see HTTP content; Shield Standard (D) is already on and only covers L3/L4.
5. **C** — The Shield Response Team and DDoS cost protection are Shield Advanced features.
6. **B** — GuardDuty detects crypto-mining traffic from Flow Logs/DNS logs it reads itself; EventBridge + Lambda automates isolation. Inspector (C) finds vulnerabilities, not live threats.
7. **B** — Macie discovers sensitive data in S3. Inspector looks for CVEs; Detective investigates findings; Security Hub aggregates.
8. **B** — User pool federates Google sign-in; identity pool swaps the token for temporary role credentials; the policy variable scopes each user to their own prefix. Identity Center (C) is for workforce access to AWS accounts.
9. **B** — A CloudHSM-backed custom key store keeps KMS integrations while key material lives in single-tenant HSMs you control. Imported material (C) still lives in KMS's multi-tenant HSMs.
10. **C** — Multi-Region keys share key material and ID across Regions, so ciphertext decrypts locally in either Region. A fails when the first Region is down; separate keys (D) have different material even with the same alias.

</details>

---

## 12. Flashcards (cover the right column)

| Prompt | Answer |
| --- | --- |
| Max plaintext for `kms:Encrypt` | 4 KB |
| API for envelope encryption | `GenerateDataKey` |
| Can you edit an AWS managed key's policy? | No |
| Share an encrypted snapshot cross-account — key type needed | Customer managed key |
| What does `arn:aws:iam::<acct>:root` in a key policy mean? | Let IAM policies in that account grant access to the key |
| Condition: only use the key via S3 | `kms:ViaService` |
| KMS key deletion waiting period | 7–30 days (default 30) |
| Automatic rotation changes the key ID? | No |
| Rotate imported key material | Manually: new key + move the alias |
| Same key material in two Regions | Multi-Region keys |
| Single-tenant HSM you control | CloudHSM (cluster in ≥ 2 AZs) |
| CloudFront certificate Region | us-east-1 |
| Does ACM renew imported certs? | No |
| Built-in rotation: Secrets Manager or Parameter Store? | Secrets Manager |
| Free tier of Parameter Store | Standard parameters |
| Secret readable in a DR Region | Secrets Manager replica secret |
| SQL injection protection | AWS WAF |
| Free automatic DDoS protection | Shield Standard |
| DDoS response team + cost protection | Shield Advanced |
| Can WAF attach to an NLB? | No |
| Throttle abusive IPs | WAF rate-based rule |
| Central WAF/SG policies across an org | Firewall Manager |
| Compromised instance / crypto-mining detection | GuardDuty |
| CVE scanning of EC2, ECR, Lambda | Inspector |
| PII in S3 | Macie |
| Aggregate findings + CIS/FSBP checks | Security Hub |
| Root-cause investigation graph | Detective |
| AWS compliance reports download | AWS Artifact |
| App sign-in, JWTs, social login | Cognito user pool |
| Temporary AWS creds for app users | Cognito identity pool |
| Policy variable for a Cognito user's own prefix | `${cognito-identity.amazonaws.com:sub}` |
| Authenticate at the load balancer | ALB `authenticate-cognito` / `authenticate-oidc` |

## 13. Before you close today

- [ ] Read sections 1–9 (focus: envelope encryption, key policy statements, Secrets Manager vs Parameter Store, WAF vs Shield, the detect-and-respond table, user pool vs identity pool)
- [ ] Lab done and torn down (secret, parameters, KMS key scheduled for deletion, GuardDuty detector, Cognito user pool)
- [ ] 10 questions above + 20 bank questions on KMS/secrets/WAF/Shield/GuardDuty/Cognito
- [ ] Misses logged in `miss-log.md` — it's still empty; Days 12–13 are built from it, so log any misses from Days 7–10 now

**Tomorrow (Day 11):** Disaster recovery strategies (backup & restore → pilot light → warm standby → multi-site), AWS Backup, migration (DMS, MGN, DataSync, Snow Family), and CloudWatch, CloudTrail and Config.
