# Day 4 — VPC Networking, Endpoints & Hybrid Connectivity

**Domain weight:** mostly *Design Secure Architectures* (30%) — network segmentation and private access are security questions — plus *Design Resilient Architectures* (26%) for multi-AZ NAT and redundant hybrid links, and *Cost-Optimized* (20%) for NAT vs endpoint data charges.
**Time plan:** Learn 90 min · Lab 30 min · Drill 45 min · Review 30 min.

By the end of today you should be able to:

- Build a public/private multi-AZ VPC on paper: CIDRs, subnets, route tables, IGW, NAT gateways, and say what makes a subnet "public".
- Explain security groups vs network ACLs well enough to debug a "traffic is blocked" question (stateful vs stateless, ephemeral ports, deny rules).
- Pick a gateway endpoint, interface endpoint (PrivateLink) or NAT gateway for "reach AWS services privately" — and know the cost difference.
- Choose between VPC peering, Transit Gateway and PrivateLink for connecting VPCs, including when CIDRs overlap.
- Choose between Site-to-Site VPN, Direct Connect, DX + VPN backup and Client VPN for hybrid connectivity, with the resilience and encryption caveats.

---

## 1. The VPC mental model

A **VPC** is your own logically isolated network inside one **Region**. It spans every AZ in that Region; **subnets** each live in exactly **one AZ**.

```
┌──────────────────────────── VPC 10.0.0.0/16 (ap-south-1) ────────────────────────────┐
│                                                                                      │
│  ┌──────── AZ a ─────────┐        ┌──────── AZ b ─────────┐                          │
│  │ Public  10.0.0.0/24   │        │ Public  10.0.1.0/24   │   rt-public:             │
│  │  ALB node, NAT GW a   │        │  ALB node, NAT GW b   │   10.0.0.0/16 → local    │
│  ├───────────────────────┤        ├───────────────────────┤   0.0.0.0/0  → igw       │
│  │ Private 10.0.10.0/24  │        │ Private 10.0.11.0/24  │   rt-private-a / -b:     │
│  │  app EC2 (ASG)        │        │  app EC2 (ASG)        │   10.0.0.0/16 → local    │
│  ├───────────────────────┤        ├───────────────────────┤   0.0.0.0/0  → nat-a/b   │
│  │ Data    10.0.20.0/24  │        │ Data    10.0.21.0/24  │   rt-data:               │
│  │  RDS primary          │        │  RDS standby          │   10.0.0.0/16 → local    │
│  └───────────────────────┘        └───────────────────────┘   (no internet route)    │
│                                                                                      │
│                    Internet gateway (one per VPC, HA by design)                      │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

### CIDR blocks

- IPv4 VPC CIDR: between **/16** (65,536 addresses) and **/28** (16). You can add **secondary CIDRs** later if you run out.
- Use RFC 1918 ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`) and **plan for no overlap** with on-prem and other VPCs — overlap blocks peering and complicates TGW/VPN routing later. This is the single most expensive mistake to fix after the fact.
- **AWS reserves 5 IPs in every subnet**: network address (`.0`), VPC router (`.1`), DNS (`.2`), future use (`.3`), and the last address. A `/24` gives **251** usable, a `/28` gives **11**.
- IPv6: VPC gets a `/56`, subnets get `/64`. IPv6 addresses are globally unique and public — "private" IPv6 egress uses an **egress-only internet gateway** (below).

> Trap: "How many instances fit in a /28 subnet?" → 16 − 5 = **11**, not 16 or 14.

### What makes a subnet public?

Nothing on the subnet itself. A subnet is **public** if its **route table** has `0.0.0.0/0 → igw-…`. An instance in it also needs a **public IPv4 address or Elastic IP** to be reachable. Remove either and it's not reachable from the internet.

Note: AWS charges for **all public IPv4 addresses** (in use or idle) — check current pricing. That's one more reason the exam pattern keeps instances in private subnets behind a load balancer.

### Route tables

- Every subnet is associated with exactly one route table (the **main** route table if you don't choose). One route table can serve many subnets.
- Every route table has a `local` route for the VPC CIDR — intra-VPC traffic always works at the routing layer (SGs/NACLs still apply).
- **Longest prefix match** wins: `10.0.5.0/24 → pcx-…` beats `10.0.0.0/8 → tgw-…`.
- Targets you'll see: `igw`, `nat`, `vgw` (virtual private gateway), `tgw`, `pcx` (peering), `vpce` (gateway endpoint, via prefix list), `eni` (appliance), `eigw`.

> Best practice: leave the **main** route table private and explicitly associate public subnets with a public route table. A new subnet then defaults to private — fail safe.

### The default VPC

Every Region has one: a `/16`, a public `/20` subnet per AZ, an IGW, and auto-assigned public IPs. Fine for labs; exam answers for production use a **custom VPC** with private subnets.

---

## 2. Getting out: IGW, NAT gateway, egress-only IGW

| Component | Direction | Scope | Notes |
| --- | --- | --- | --- |
| **Internet gateway** | In and out (IPv4 & IPv6) | VPC (one per VPC) | Horizontally scaled, redundant — no HA work for you. Does 1:1 NAT for instances with public IPs. |
| **NAT gateway (public)** | **Out only** (IPv4) | **One AZ** | Lives in a **public** subnet, needs an Elastic IP. Private instances route `0.0.0.0/0` to it. Managed, scales automatically. |
| **NAT gateway (private)** | Out to other private networks | One AZ | No EIP; translates to its private IP. Used to reach on-prem/other VPCs from overlapping CIDRs (via TGW/VGW). |
| **NAT instance** | Out only | One instance | Legacy. You manage HA, patching, sizing; must **disable source/destination check**. Only right if the question needs something NAT GW can't do (e.g. port forwarding, bastion on the same box) or extreme cost minimisation at tiny scale. |
| **Egress-only IGW** | **Out only, IPv6** | VPC | The IPv6 equivalent of NAT for "outbound only". IPv6 doesn't use NAT. |

### NAT gateway HA — the classic exam pattern

A NAT gateway is **zonal**. If you put one NAT GW in AZ a and route both AZs' private subnets to it:

1. Losing AZ a breaks internet egress for AZ b too.
2. AZ b traffic crosses AZs → cross-AZ data charges.

So the resilient answer is **one NAT gateway per AZ, and each private route table points to the NAT in its own AZ**. (AWS has also introduced a regional availability mode for NAT gateways that spans AZs — check current docs; the exam's classic answer is still "one per AZ".)

```hcl
# Each private subnet's route table → NAT in the same AZ
resource "aws_route" "private_default" {
  for_each               = aws_route_table.private      # keyed by AZ
  route_table_id         = each.value.id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = aws_nat_gateway.this[each.key].id
}
```

Other NAT GW facts:

- Charged **per hour + per GB processed** (check current pricing). High-volume S3/DynamoDB traffic through NAT is a classic cost trap — use a **gateway endpoint** instead (section 4).
- NAT GW has **no security group**; control traffic with SGs on the instances and NACLs on subnets.
- It only handles traffic *initiated from inside*. "Allow the internet to reach my private instance" is never solved by NAT — use a load balancer in public subnets.

> Trap: a NAT gateway placed in a **private** subnet doesn't work — it needs its own route to the IGW.

---

## 3. Security groups vs network ACLs

This is the most-tested comparison in networking.

| | **Security group** | **Network ACL** |
| --- | --- | --- |
| Attached to | ENI (instance, ALB, RDS, Lambda in VPC, endpoint…) | Subnet |
| State | **Stateful** — return traffic is automatically allowed | **Stateless** — return traffic must be explicitly allowed |
| Rules | **Allow only** | **Allow and deny** |
| Evaluation | All rules evaluated together (union) | In **rule-number order**, lowest first; first match wins |
| Default (new custom) | Inbound: none · Outbound: all | **Custom NACL: deny all** in and out |
| Default (VPC's default) | Allows inbound from members of the same SG | **Default NACL: allow all** |
| Can reference | Other SGs (by ID), prefix lists, CIDRs | CIDRs only |
| Typical use | Primary, fine-grained control | Coarse subnet guardrail; **block a specific IP/CIDR** |

```
Internet ──► NACL (public subnet, in) ──► SG (ALB, in) ──► ALB
             NACL (public subnet, out) ─► NACL (private subnet, in) ─► SG (app, in) ─► EC2
Return path: SGs auto-allow it; every NACL on the way must allow it explicitly (ephemeral ports).
```

### The ephemeral-port trap

Clients pick a random **source port** (Linux usually 32768–60999; AWS guidance uses **1024–65535** to cover all OSes). With a stateless NACL, the **response** goes back to that port, so:

- Public subnet NACL allowing inbound 443 must also allow **outbound 1024–65535** to `0.0.0.0/0`.
- A private subnet calling out through NAT must allow **inbound 1024–65535** for the replies.

If a question says "SG allows 443, NACL allows inbound 443, connection still times out" → the NACL is missing the **outbound ephemeral range**.

### SG referencing — the pattern you should default to

```
alb-sg:  inbound 443 from 0.0.0.0/0
app-sg:  inbound 8080 from alb-sg      ← source is a security group ID, not a CIDR
db-sg:   inbound 3306 from app-sg
```

Scaling adds instances to `app-sg` automatically; no IPs to maintain. SG references also work across **peered VPCs in the same Region** and in **shared VPCs**.

### Blocking a bad actor

| Need | Tool |
| --- | --- |
| Block one IP/CIDR at the subnet | **NACL deny** rule with a low rule number |
| Block at the HTTP layer, rate-limit, geo-block | **AWS WAF** on ALB/CloudFront/API Gateway (Day 10) |
| Stateful inspection, domain allow-lists, IDS/IPS for the whole VPC | **AWS Network Firewall** |
| Security groups? | **Can't deny** — wrong answer for "block this IP" |

---

## 4. Reaching AWS services privately: VPC endpoints

By default, calls from a private instance to S3/SQS/etc. go to public service endpoints — via NAT. VPC endpoints keep that traffic on the AWS network without an IGW or NAT.

| | **Gateway endpoint** | **Interface endpoint (PrivateLink)** |
| --- | --- | --- |
| Services | **S3 and DynamoDB only** | Most AWS services (SQS, SNS, KMS, Secrets Manager, SSM, ECR, CloudWatch, STS…), S3 and DynamoDB too, partner/SaaS and your own services |
| How it works | A **route-table entry** to a managed prefix list (`pl-…`) → `vpce-…` | **ENI(s) with private IPs** in your subnets; private DNS makes the normal service hostname resolve to them |
| Cost | **Free** | Per hour per AZ + per GB (check current pricing) |
| Security groups | No (use endpoint policy) | **Yes**, on the ENIs |
| Reachable from on-prem (VPN/DX) or peered VPCs | **No** | **Yes** |
| HA | Built in | Deploy in **multiple AZs** |

```
Private subnet EC2 ── s3 API call ──► route table: pl-xxxx (S3 prefix list) → vpce-gw ──► S3
                                      (no NAT, no IGW, no data-processing charge)

On-prem server ──DX/VPN──► Interface endpoint ENI 10.0.10.25 (com.amazonaws.ap-south-1.s3) ──► S3
                           (gateway endpoints can't be used from outside the VPC)
```

### Locking it down from both sides

**Endpoint policy** (on the endpoint) — what can be done *through* it:

```json
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": "*",
    "Action": ["s3:GetObject", "s3:PutObject"],
    "Resource": "arn:aws:s3:::acme-app-data/*"
  }]
}
```

**Bucket policy** (on the resource) — only accept requests that came *through* the endpoint:

```json
{
  "Statement": [{
    "Sid": "DenyUnlessFromMyEndpoint",
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:*",
    "Resource": ["arn:aws:s3:::acme-app-data", "arn:aws:s3:::acme-app-data/*"],
    "Condition": { "StringNotEquals": { "aws:SourceVpce": "vpce-0abc123def456" } }
  }]
}
```

> Traps:
> - `aws:SourceIp` doesn't work for traffic via a VPC endpoint (it carries private IPs) — use `aws:SourceVpce` or `aws:SourceVpc`.
> - Gateway endpoints are **same-Region** only.
> - Private DNS for interface endpoints requires the VPC attributes `enableDnsSupport` and `enableDnsHostnames` = true.
> - Running Session Manager on instances with **no internet path** needs interface endpoints for `ssm`, `ssmmessages` and `ec2messages` (check current docs for the exact list).

### Cost decision: NAT vs endpoints

"Private instances pull terabytes from S3 every day and the NAT gateway bill is high" → **S3 gateway endpoint** (free; removes the per-GB NAT processing charge). This is one of the most common cost-optimisation answers on the exam.

---

## 5. Connecting VPCs: peering, Transit Gateway, PrivateLink

### VPC peering

A private, one-to-one connection between two VPCs (same or different account, same or different Region). Uses the AWS backbone; no single point of failure, no bandwidth bottleneck.

Rules that create exam questions:

1. **Non-transitive.** A↔B and B↔C does *not* give A↔C. You need A↔C peered too → full mesh = n(n−1)/2 connections.
2. **No overlapping CIDRs.**
3. **No edge-to-edge routing.** A can't use B's IGW, NAT gateway, VPN or Direct Connect.
4. Both sides must **add routes** (and allow in SGs/NACLs). Creating the peering alone does nothing.

### AWS Transit Gateway (TGW)

A **regional, managed hub router**. Attach VPCs, Site-to-Site VPNs, Direct Connect (via a DX gateway), and other TGWs (inter-Region **peering**).

```
            ┌───────── Transit Gateway (ap-south-1) ─────────┐
 VPC-Prod ──┤  TGW route table "prod"   → prod, shared, vpn  ├── Site-to-Site VPN (on-prem)
 VPC-Dev  ──┤  TGW route table "dev"    → dev, shared        ├── DX gateway (Direct Connect)
 VPC-Shared─┤  TGW route table "shared" → all                ├── TGW peering → eu-west-1 TGW
            └────────────────────────────────────────────────┘
```

- **Transitive** by design: everything attached can reach everything else, subject to TGW route tables.
- **Multiple TGW route tables** = segmentation (prod can't reach dev, both reach shared services).
- **Share across accounts** with **AWS RAM** (Day 1).
- **ECMP** across multiple VPN connections to aggregate VPN bandwidth beyond a single tunnel.
- Supports **multicast** (rarely tested, but TGW is the answer if it is).
- Cost: per attachment-hour + per GB processed (check current pricing). Peering has no hourly or processing charge (only normal data transfer) — for **two** VPCs, peering is cheaper and simpler.

### AWS PrivateLink (endpoint services)

Expose **one service** from a provider VPC to many consumer VPCs/accounts, without exposing the whole network.

```
Consumer VPC (10.0.0.0/16)                    Provider VPC (10.0.0.0/16 — same CIDR is fine!)
  app ──► Interface endpoint ENI ══PrivateLink══► Endpoint service ──► NLB ──► service instances
```

- Provider fronts the service with an **NLB** (or GWLB for appliances) and creates an **endpoint service**; consumers create an interface endpoint. The provider can require **acceptance** of connection requests.
- **Unidirectional** (consumer → provider) and **works with overlapping CIDRs**.
- It's how SaaS vendors on AWS Marketplace deliver private connectivity.
- Related: **VPC Lattice** is a newer managed service-to-service networking layer (service network, auth policies, HTTP/gRPC routing) across VPCs and accounts — recognise the name if it appears for microservice connectivity.

### Choosing

| Requirement | Pick |
| --- | --- |
| Connect 2–3 VPCs, simplest/cheapest | **VPC peering** |
| Connect dozens/hundreds of VPCs + on-prem, central routing, segmentation | **Transit Gateway** |
| Expose one app/API to many VPCs or customers, CIDRs overlap, no full network access | **PrivateLink** |
| VPCs across Regions as a hub | TGW + **inter-Region TGW peering** (or inter-Region VPC peering for a pair) |
| Many accounts need subnets in one centrally managed VPC | **VPC sharing** via AWS RAM (participants launch into shared subnets; owner manages the network) |

---

## 6. Hybrid connectivity: VPN and Direct Connect

### AWS Site-to-Site VPN

IPsec tunnels over the **public internet** between your **customer gateway** (on-prem device) and either a **virtual private gateway** (VGW, attached to one VPC) or a **Transit Gateway**.

- Each VPN connection has **two tunnels** terminating in different AWS endpoints — configure both for HA.
- Throughput per tunnel is limited (commonly cited ~1.25 Gbps — check current docs). Scale with **ECMP on Transit Gateway** across multiple connections.
- **Set up in minutes/hours** — the answer for "connect quickly" and for **backup** to Direct Connect.
- Routing: static or **BGP** (dynamic, preferred for failover).
- **Accelerated Site-to-Site VPN** (TGW only) enters AWS at the nearest edge via Global Accelerator for more consistent performance.
- **VPN CloudHub**: multiple branch offices connect to one VGW and can reach each other (hub-and-spoke over VPN).

### AWS Client VPN

Managed OpenVPN-based **remote-access** VPN for individual users' laptops into a VPC (and onward to on-prem). "Remote employees need secure access to private resources" → Client VPN. Not the same as Site-to-Site.

### AWS Direct Connect (DX)

A **private, dedicated physical connection** from your network to AWS at a DX location.

| | Site-to-Site VPN | Direct Connect |
| --- | --- | --- |
| Path | Public internet | Private fibre to a DX location |
| Encryption | **IPsec, always** | **Not encrypted by default** |
| Bandwidth | Limited per tunnel | Dedicated: 1, 10, 100 Gbps (and 400 Gbps at some locations); hosted via partners: sub-1 Gbps up to multi-Gbps — check current docs |
| Latency/consistency | Variable | **Consistent, low** |
| Setup time | Minutes | **Weeks to months** |
| Cost model | Low hourly + data | Port-hours + lower data-transfer-out rates |

**Virtual interfaces (VIFs)** on a DX connection:

| VIF | Reaches |
| --- | --- |
| **Private VIF** | A VPC via its VGW, or many VPCs via a **Direct Connect gateway** |
| **Public VIF** | AWS **public** service endpoints (S3, DynamoDB public IPs…) over DX instead of the internet |
| **Transit VIF** | One or more **Transit Gateways** via a Direct Connect gateway |

**Direct Connect gateway** is a **global** object: one DX connection can reach VGWs/TGWs in **any Region**. It does **not** route between the VPCs attached to it (not a transit router for VPC↔VPC traffic).

**Encrypting DX** — the exam wants one of:

- **IPsec VPN over DX** (e.g. a private-IP VPN to a TGW over a transit VIF, or a VPN over a public VIF), or
- **MACsec** (layer-2 encryption) on supported dedicated connections (check docs for supported speeds/locations).

### DX resilience

```
Development / non-critical:   On-prem ── DX loc 1 ── 1 connection ──────────── AWS
High resiliency:              On-prem ─┬ DX loc 1 ── 1 connection ──┐
                                       └ DX loc 2 ── 1 connection ──┴───────── AWS
Maximum resiliency:           On-prem ─┬ DX loc 1 ── 2 connections (separate devices) ─┐
                                       └ DX loc 2 ── 2 connections (separate devices) ─┴─ AWS
Cost-effective backup:        DX (primary) + Site-to-Site VPN (backup, BGP prefers DX)
```

- "**Most** resilient / critical production" → **two DX locations, redundant connections** (the DX Resiliency Toolkit's *maximum resiliency* model).
- "**Cost-effective** failover for DX" → **Site-to-Site VPN as backup**.
- "Need connectivity **now**, DX in 2 months" → start with **VPN**, migrate to DX.
- **Link aggregation group (LAG)** bundles multiple connections at the *same* location into one logical link — more bandwidth, but not location resilience.

### Hybrid DNS — Route 53 Resolver endpoints

| Direction | Use |
| --- | --- |
| **Inbound endpoint** | On-prem DNS servers forward queries **into** the VPC to resolve private hosted zones / VPC names |
| **Outbound endpoint + forwarding rules** | VPC resources resolve **on-prem** domains (e.g. `corp.example.com`) by forwarding to on-prem DNS |

Share Resolver rules across accounts with RAM.

---

## 7. Visibility, access and other VPC features

| Feature | What it does / exam cue |
| --- | --- |
| **VPC Flow Logs** | Capture IP traffic **metadata** (src/dst, ports, bytes, ACCEPT/REJECT) at VPC, subnet or ENI level → CloudWatch Logs, S3 or Amazon Data Firehose. **No payloads.** Query in S3 with Athena. Doesn't log some traffic (e.g. to the Amazon DNS server, instance metadata, DHCP) — check docs. |
| **Traffic Mirroring** | Copies actual **packets** from ENIs to an inspection appliance/NLB — "inspect packet contents / IDS". |
| **Reachability Analyzer** | Static analysis of the path between two resources — "why can't A reach B?" without sending traffic. |
| **Network Access Analyzer** | Finds unintended network access paths against your rules (e.g. "nothing in the data subnet reachable from the internet"). |
| **AWS Network Firewall** | Managed stateful firewall/IPS for a VPC: domain filtering, Suricata rules. Deploy in a firewall subnet; route traffic through it (often centrally via TGW). |
| **Bastion host** | Public EC2 jump box for SSH into private instances — legacy pattern. |
| **Session Manager** | Shell access with **no inbound ports, no SSH keys, no bastion**, logged to CloudWatch/S3 — preferred answer. |
| **EC2 Instance Connect Endpoint** | SSH/RDP to private instances **without a public IP or bastion**, IAM-controlled. |
| **Elastic network interface (ENI)** | Virtual NIC; can be moved between instances in the same AZ (fail-over a fixed private IP/MAC). |
| **Elastic IP** | Static public IPv4; remap to another instance on failure. Charged like any public IPv4. |
| **Managed prefix lists** | Named sets of CIDRs reusable in SGs and route tables. |
| **DHCP option sets** | Custom DNS servers / domain name for the VPC. |

### Data transfer cost rules of thumb (check current pricing)

- Same AZ over **private IP**: generally free. Over **public/Elastic IP**: charged even within the AZ.
- **Cross-AZ**: charged in each direction — a reason to keep chatty tiers AZ-local where possible and to use per-AZ NAT gateways.
- Inbound from internet: free. Outbound to internet: charged (CloudFront often cheaper for egress-heavy content — Day 9).
- NAT gateway and TGW add **per-GB processing** on top.

---

## 8. Exam traps roundup

1. **Security groups can't deny.** "Block IP x.x.x.x" → NACL (or WAF for HTTP).
2. **NACLs are stateless** → missing outbound ephemeral ports is the usual bug.
3. **Peering is non-transitive and has no edge-to-edge routing** — a peered VPC can't use your NAT, IGW, VPN or DX.
4. **Overlapping CIDRs** rule out peering; use **PrivateLink** (or a private NAT gateway design).
5. **Gateway endpoints** = S3/DynamoDB, free, VPC-internal only. From **on-prem** you need an **interface endpoint**.
6. **One NAT gateway per AZ** for HA; NAT GW goes in a **public** subnet.
7. **Direct Connect isn't encrypted** — add IPsec VPN or MACsec.
8. **DX takes weeks**; VPN is the quick start and the cheap backup.
9. **DX gateway** spans Regions but does not route VPC↔VPC.
10. Flow Logs = metadata; packet contents = **Traffic Mirroring**.
11. A subnet is **one AZ**; a VPC is **one Region**.
12. 5 reserved IPs per subnet.

---

## 9. Exam decision table

| If the question says… | Pick |
| --- | --- |
| Private instances need to download patches from the internet | **NAT gateway** in a public subnet (one per AZ for HA) |
| IPv6 instances need outbound-only internet access | **Egress-only internet gateway** |
| Private instances read/write S3 or DynamoDB; reduce NAT cost / keep off internet | **Gateway VPC endpoint** |
| Private access to SQS / KMS / Secrets Manager / SSM / ECR from a VPC | **Interface endpoint** (PrivateLink) |
| On-prem servers must reach S3 privately over DX/VPN | **S3 interface endpoint** |
| Bucket must accept requests only from one VPC endpoint | Bucket policy with **`aws:SourceVpce`** |
| Block a specific malicious IP range from a subnet | **NACL deny** rule |
| Allow app tier only from the load balancer | App SG inbound **from the ALB's security group** |
| SG and NACL inbound look right but responses never arrive | NACL **outbound ephemeral ports** 1024–65535 |
| Connect two VPCs, low cost, low effort | **VPC peering** |
| Connect many VPCs and on-prem with central routing and segmentation | **Transit Gateway** (+ TGW route tables, share via RAM) |
| Offer a service to other VPCs/accounts with overlapping CIDRs | **PrivateLink** endpoint service behind an **NLB** |
| Many accounts launch into a centrally managed network | **VPC sharing** with AWS RAM |
| Connect on-prem to AWS quickly / encrypted over internet | **Site-to-Site VPN** |
| Consistent low latency, high bandwidth, private link to AWS | **Direct Connect** |
| DX must be encrypted | **IPsec VPN over DX** or **MACsec** |
| Cost-effective failover for a DX link | **Site-to-Site VPN backup** |
| Maximum resilience for hybrid connectivity | **2 DX locations × redundant connections** |
| One DX connection to VPCs in several Regions | **Direct Connect gateway** |
| Aggregate VPN bandwidth beyond one tunnel | **TGW with ECMP** across multiple VPN connections |
| Remote employees access private VPC resources | **AWS Client VPN** |
| On-prem must resolve private hosted zone names (or vice versa) | **Route 53 Resolver inbound** (or **outbound**) endpoints |
| Shell access to private instances without bastion or open ports | **Session Manager** (or **EC2 Instance Connect Endpoint** for SSH) |
| Troubleshoot accepted/rejected traffic | **VPC Flow Logs** (query with Athena) |
| Inspect packet payloads | **Traffic Mirroring** |
| Diagnose why two resources can't connect, without sending traffic | **Reachability Analyzer** |
| Centralised stateful inspection / domain filtering for VPC egress | **AWS Network Firewall** |

---

## 10. Lab (30 min, free tier)

Goal: build a two-AZ VPC by hand, prove that a private subnet has no internet but **can** reach S3 through a gateway endpoint, and watch a NACL deny show up in Flow Logs.

**Cost note:** VPCs, subnets, route tables, IGWs, gateway endpoints and EC2 Instance Connect Endpoints have no hourly charge (check current docs). **NAT gateways, interface endpoints and public IPv4 addresses are charged** — this lab avoids a NAT gateway on purpose. Use a free-tier-eligible instance type and tear down at the end.

### Console steps

1. **VPC** → *Create VPC* → **VPC only**: name `lab-vpc`, `10.20.0.0/16`. After creation, *Edit VPC settings* → enable **DNS hostnames** (DNS resolution is on by default).
2. **Subnets** (all in `lab-vpc`):
   - `lab-public-a` `10.20.0.0/24` in AZ a · `lab-public-b` `10.20.1.0/24` in AZ b
   - `lab-private-a` `10.20.10.0/24` in AZ a · `lab-private-b` `10.20.11.0/24` in AZ b
   Note the available IP count on a `/24`: **251**.
3. **Internet gateway** `lab-igw` → *Attach to VPC* `lab-vpc`.
4. **Route tables:**
   - `lab-rt-public`: add `0.0.0.0/0 → lab-igw`; associate both public subnets.
   - `lab-rt-private`: leave only the `local` route; associate both private subnets.
5. **Gateway endpoint:** *Endpoints* → *Create* → service `com.amazonaws.<region>.s3`, type **Gateway**, VPC `lab-vpc`, route table `lab-rt-private`. Open `lab-rt-private` and see the new `pl-… → vpce-…` route.
6. **Security groups:**
   - `lab-eice-sg`: no inbound; outbound TCP 22 to `10.20.0.0/16`.
   - `lab-private-sg`: inbound TCP 22 **from `lab-eice-sg`**; default outbound.
7. **EC2 Instance Connect Endpoint:** *Endpoints* → *Create* → type **EC2 Instance Connect Endpoint**, `lab-vpc`, subnet **`lab-private-b`**, SG `lab-eice-sg`. (Takes a few minutes.) It goes in a *different* subnet from the instance on purpose: NACLs only filter traffic crossing a subnet boundary, so step 11 wouldn't show anything if both sat in the same subnet.
8. **Instance:** launch Amazon Linux 2023, free-tier type, subnet `lab-private-a`, **auto-assign public IP disabled**, SG `lab-private-sg`, an instance profile with `AmazonS3ReadOnlyAccess` (recreate the Day 1 role if you deleted it), IMDSv2 required.
9. **Flow log:** select `lab-private-a` subnet → *Flow logs* → *Create* → filter **All**, destination **CloudWatch Logs** (let the console create the role), log group `/lab/vpc-flow`.
10. **Test:** EC2 → instance → *Connect* → **EC2 Instance Connect** → *Connect using EC2 Instance Connect Endpoint*. Then:
    ```bash
    curl -m 5 https://aws.amazon.com          # times out: no IGW/NAT route from this subnet
    aws s3 ls --region <your-region>          # works: gateway endpoint + instance role
    ```
    That contrast is the whole point of section 4.
11. **NACL deny:** create NACL `lab-nacl-private` on `lab-vpc` (new custom NACLs **deny everything**). Associate it with `lab-private-a` → your SSH session freezes. Add inbound rule 100 allow TCP 22 from `10.20.0.0/16`, outbound rule 100 allow TCP **1024–65535** to `10.20.0.0/16` → reconnect works. Remove the outbound rule and watch it break again: stateless in action. Then check `/lab/vpc-flow` in CloudWatch Logs for `REJECT` records.
12. **Optional:** VPC → *Reachability Analyzer* → path from the EICE network interface to the instance on port 22; read the explanation it gives when the NACL blocks it.
13. **Tear down (in this order):** terminate the instance → delete the EC2 Instance Connect Endpoint → delete the flow log and the `/lab/vpc-flow` log group → delete the S3 gateway endpoint → re-associate subnets to the default NACL and delete `lab-nacl-private` → detach and delete `lab-igw` → delete `lab-vpc` (removes subnets, route tables and SGs). Check *Elastic IPs* for leftovers.

### Terraform variant

```hcl
provider "aws" { region = "ap-south-1" }

data "aws_availability_zones" "az" { state = "available" }

data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

locals {
  azs = slice(data.aws_availability_zones.az.names, 0, 2)
}

resource "aws_vpc" "lab" {
  cidr_block           = "10.20.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags                 = { Name = "lab-vpc" }
}

resource "aws_subnet" "public" {
  count             = 2
  vpc_id            = aws_vpc.lab.id
  cidr_block        = cidrsubnet("10.20.0.0/16", 8, count.index)        # 10.20.0.0/24, 10.20.1.0/24
  availability_zone = local.azs[count.index]
  tags              = { Name = "lab-public-${count.index}" }
}

resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.lab.id
  cidr_block        = cidrsubnet("10.20.0.0/16", 8, 10 + count.index)   # 10.20.10.0/24, 10.20.11.0/24
  availability_zone = local.azs[count.index]
  tags              = { Name = "lab-private-${count.index}" }
}

resource "aws_internet_gateway" "igw" { vpc_id = aws_vpc.lab.id }

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.lab.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
}

resource "aws_route_table" "private" { vpc_id = aws_vpc.lab.id }   # local route only

resource "aws_route_table_association" "public" {
  count          = 2
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count          = 2
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private.id
}

resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.lab.id
  service_name      = "com.amazonaws.ap-south-1.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = [aws_route_table.private.id]
}

resource "aws_security_group" "eice" {
  name   = "lab-eice-sg"
  vpc_id = aws_vpc.lab.id
  egress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [aws_vpc.lab.cidr_block]
  }
}

resource "aws_security_group" "private" {
  name   = "lab-private-sg"
  vpc_id = aws_vpc.lab.id
  ingress {
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.eice.id]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_ec2_instance_connect_endpoint" "eice" {
  subnet_id          = aws_subnet.private[1].id   # different subnet from the instance (NACL test)
  security_group_ids = [aws_security_group.eice.id]
}

resource "aws_instance" "private" {
  ami                         = data.aws_ami.al2023.id
  instance_type               = "t3.micro"            # use your account's free-tier-eligible type
  subnet_id                   = aws_subnet.private[0].id
  vpc_security_group_ids      = [aws_security_group.private.id]
  associate_public_ip_address = false
  iam_instance_profile        = "lab-ec2-s3-read"     # an existing profile with AmazonS3ReadOnlyAccess
  metadata_options { http_tokens = "required" }
  tags = { Name = "lab-private" }
}

# Flow logs to S3 (no IAM role needed for this destination)
resource "aws_s3_bucket" "flow" {
  bucket_prefix = "lab-vpc-flow-"
  force_destroy = true
}

resource "aws_flow_log" "subnet" {
  subnet_id            = aws_subnet.private[0].id
  traffic_type         = "ALL"
  log_destination_type = "s3"
  log_destination      = aws_s3_bucket.flow.arn
}

output "connect" {
  value = "aws ec2-instance-connect ssh --instance-id ${aws_instance.private.id} --connection-type eice"
}
```

Run `terraform apply`, connect with the output command, repeat the `curl` vs `aws s3 ls` test, then add an `aws_network_acl` + `aws_network_acl_association` to repeat the stateless experiment.

**Tear down:** `terraform destroy`, then confirm in the console that no instances, endpoints or Elastic IPs remain.

---

## 11. Practice questions

Answer all ten before opening the answers.

**Q1.** A company runs an application on EC2 instances in private subnets across two Availability Zones. The instances must download software updates from the internet. The solution must remain available if one AZ fails. What should a solutions architect do?
A. Create one NAT gateway in a public subnet and route both private subnets to it
B. Create a NAT gateway in a public subnet in each AZ and route each private subnet to the NAT gateway in its own AZ
C. Create a NAT gateway in each private subnet
D. Attach an egress-only internet gateway and route the private subnets to it

**Q2.** EC2 instances in private subnets process several terabytes of data from Amazon S3 each day. The data flows through a NAT gateway, and the NAT gateway charges have become significant. Which change reduces cost the MOST?
A. Replace the NAT gateway with a NAT instance
B. Create an S3 gateway VPC endpoint and associate it with the private subnets' route tables
C. Create an S3 interface VPC endpoint in each AZ
D. Move the instances into public subnets with Elastic IPs

**Q3.** A web server in a public subnet has a security group allowing inbound TCP 443 from `0.0.0.0/0`. The subnet's custom network ACL allows inbound TCP 443 from `0.0.0.0/0` and has no other rules besides the default deny. Users cannot load the site. What is the cause?
A. The security group must also allow outbound TCP 443
B. The network ACL must allow outbound traffic on ephemeral ports 1024–65535
C. The network ACL must allow inbound ephemeral ports
D. The instance needs a NAT gateway to respond

**Q4.** A security team detects malicious traffic from a single IP address against an application in a VPC. The team must block that IP address at the subnet level immediately. What should a solutions architect do?
A. Add a deny rule for the IP address to the instances' security group
B. Add a deny rule with a low rule number for the IP address to the subnet's network ACL
C. Remove the internet gateway from the VPC
D. Create a VPC Flow Log filter for the IP address

**Q5.** A company has 60 VPCs across multiple accounts and an on-premises data center connected by Site-to-Site VPN. It wants all VPCs to communicate with each other and with on premises, while keeping development VPCs isolated from production VPCs. Which solution has the LEAST operational overhead?
A. Full-mesh VPC peering plus a VPN connection to each VPC
B. A Transit Gateway shared with AWS RAM, with the VPN attached to it and separate TGW route tables for production and development
C. VPC peering to a central VPC that holds the VPN connection
D. AWS PrivateLink endpoint services in every VPC

**Q6.** A SaaS provider wants to offer its API, running behind instances in its own VPC, privately to hundreds of customer VPCs. Many customers use the same CIDR ranges as the provider. Customers must not get network access to anything other than the API. What should the provider use?
A. VPC peering with each customer
B. A Transit Gateway shared with customers
C. An endpoint service (AWS PrivateLink) fronted by a Network Load Balancer
D. A public Application Load Balancer with security group allow-lists

**Q7.** A company uses a 10 Gbps AWS Direct Connect connection for production traffic. Compliance now requires all traffic between the data center and the VPC to be encrypted in transit. Which solution meets the requirement while continuing to use Direct Connect?
A. Enable encryption on the private virtual interface
B. Establish an IPsec Site-to-Site VPN over the Direct Connect connection
C. Use VPC Flow Logs with encryption enabled
D. Replace the private VIF with a public VIF

**Q8.** A company has a single Direct Connect connection to its VPC. Management wants a failover path in case the connection fails, at the LOWEST cost. What should a solutions architect recommend?
A. A second Direct Connect connection at a different DX location
B. A Site-to-Site VPN connection as a backup, with BGP preferring Direct Connect
C. A link aggregation group at the same DX location
D. AWS Client VPN

**Q9.** On-premises applications connected through Direct Connect must upload files to an Amazon S3 bucket privately, without traversing the public internet and without using public IP addresses. What should a solutions architect configure?
A. An S3 gateway endpoint in the VPC
B. An S3 interface endpoint in the VPC, accessed over the private VIF
C. A NAT gateway in the VPC
D. S3 Transfer Acceleration

**Q10.** VPC A is peered with VPC B, and VPC B is peered with VPC C. Instances in VPC A cannot reach instances in VPC C. VPC A also cannot use VPC B's NAT gateway for internet access. What explains this?
A. The peering connections need DNS resolution enabled
B. VPC peering is not transitive and does not support edge-to-edge routing
C. Security groups block cross-VPC traffic by default
D. The VPCs must be in the same Availability Zone

<details>
<summary><strong>Answers</strong></summary>

1. **B** — NAT gateways are zonal; one per AZ with AZ-local routes survives an AZ failure and avoids cross-AZ charges. C fails (NAT GW needs a route to the IGW); D is IPv6-only.
2. **B** — Gateway endpoints for S3 are free and remove NAT per-GB processing. C works but adds hourly and per-GB endpoint charges; A shifts cost to self-management.
3. **B** — NACLs are stateless; responses leave on the client's ephemeral port and need an outbound allow. SGs are stateful, so A is unnecessary.
4. **B** — Only NACLs (not SGs) support deny rules; a low rule number ensures it's evaluated before any allow.
5. **B** — TGW is transitive, scales to many VPCs, attaches the VPN once, segments with route tables, and is shared via RAM. A needs ~1,770 peerings; C fails because peering has no edge-to-edge routing.
6. **C** — PrivateLink exposes one service, works with overlapping CIDRs, and gives consumers no wider network access. Peering and TGW need non-overlapping CIDRs and open the network.
7. **B** — DX isn't encrypted by default; IPsec over DX (or MACsec on supported connections) adds encryption while keeping DX's path. There's no "encryption" switch on a VIF.
8. **B** — VPN backup is the low-cost DX failover pattern. A is the high-resilience (and higher-cost) answer.
9. **B** — Gateway endpoints can't be reached from on-premises; an interface endpoint has private IPs reachable over the private VIF.
10. **B** — Peering is non-transitive (A↔C needs its own peering) and a peer can't use another VPC's NAT, IGW, VPN or DX.

</details>

---

## 12. Flashcards (cover the right column)

| Prompt | Answer |
| --- | --- |
| VPC CIDR size range | /16 to /28 |
| Reserved IPs per subnet | 5 (first four + last) |
| Usable IPs in a /24 | 251 |
| A subnet spans how many AZs? | Exactly one |
| What makes a subnet public? | Route table has `0.0.0.0/0 → IGW` |
| Where does a public NAT gateway live? | In a public subnet, with an Elastic IP |
| NAT HA pattern | One NAT GW per AZ, AZ-local routes |
| Outbound-only internet for IPv6 | Egress-only internet gateway |
| Stateful: SG or NACL? | Security group |
| Which supports deny rules? | NACL |
| NACL rule evaluation | Lowest rule number first, first match wins |
| Default custom NACL behaviour | Deny all in and out |
| Ephemeral port range to allow | 1024–65535 |
| Services with gateway endpoints | S3 and DynamoDB |
| Gateway endpoint usable from on-prem? | No — use an interface endpoint |
| Condition key for "only via my endpoint" | `aws:SourceVpce` |
| Is VPC peering transitive? | No |
| Connect overlapping-CIDR consumers to one service | PrivateLink (NLB + endpoint service) |
| Hub router for many VPCs + VPN + DX | Transit Gateway |
| Share a TGW or subnets across accounts | AWS RAM |
| Tunnels per Site-to-Site VPN connection | 2 |
| Is Direct Connect encrypted by default? | No — IPsec VPN over DX or MACsec |
| VIF to reach AWS public services over DX | Public VIF |
| VIF to reach a Transit Gateway | Transit VIF (via DX gateway) |
| One DX to VPCs in many Regions | Direct Connect gateway |
| Cheap DX failover | Site-to-Site VPN backup |
| Maximum DX resilience | 2 locations × 2 connections |
| Remote-user VPN | AWS Client VPN |
| On-prem resolves private hosted zones | Route 53 Resolver inbound endpoint |
| Packet-level capture | Traffic Mirroring |
| Diagnose a blocked path without traffic | Reachability Analyzer |

## 13. Before you close today

- [ ] Read sections 1–9; redraw the section 1 diagram from memory, including route tables
- [ ] Lab done and torn down (no instances, endpoints or EIPs left)
- [ ] 10 questions above + 20 bank questions on VPC, endpoints and hybrid connectivity
- [ ] Misses logged in `miss-log.md` — especially any SG/NACL or peering/TGW/PrivateLink mix-ups

**Tomorrow (Day 5):** Storage — S3 storage classes, lifecycle, versioning, replication, encryption and access controls, plus EFS, FSx (Windows, Lustre, NetApp ONTAP, OpenZFS) and Storage Gateway.
