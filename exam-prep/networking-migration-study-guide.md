# 🌐 Networking, Migration & Connectivity — SAA Exam Study Guide

> **Use-case-first.** Everything below was reasoned out by building one system — a fictional game studio, **"Atlas Games,"** migrating to AWS and scaling globally. Each service earned its place by solving a real requirement, not by memorizing a list. Every section includes the **cost lens** (Domain 4), because cost is where these decisions actually get made.

---

## 🎮 The Atlas Games blueprint (the thread)

Atlas runs on-prem, blows up to millions of global players, and moves to AWS. Their needs, mapped to services:

| Need | Service |
|---|---|
| Move 200 TB+ of archives to S3, one-time, modest bandwidth | **Snowball** |
| Ongoing file sync on-prem → AWS | **DataSync** |
| Migrate a live SQL DB with no downtime | **DMS (CDC)** |
| Prod VPC ↔ Analytics VPC, private | **VPC Peering** |
| Many VPCs + on-prem, one hub | **Transit Gateway** |
| On-prem ↔ AWS: high bandwidth, consistent, private | **Direct Connect** |
| Encrypted link now + DX backup | **Site-to-Site VPN** |
| Global players → regional game servers, low latency + failover | **Global Accelerator** |

---

## Part 1 — Integration & Analytics (breadth)

| Service | What it does | Trigger |
|---|---|---|
| **API Gateway** | Managed front door for APIs (REST/HTTP/WS) | "expose/manage/throttle an API," serverless API |
| **Step Functions** | Orchestrate multi-step workflows (state machine) | "coordinate multiple Lambdas/steps with retries" |
| **Athena** | SQL queries directly on S3, serverless, pay per query | "query S3 data with SQL, no infrastructure" |
| **Redshift** | Data **warehouse** for big analytical/BI queries | "data warehouse," "OLAP on structured data" |
| **Glue** | Serverless **ETL** + data catalog | "ETL," "transform/prepare data" |
| **EMR** | Managed **Hadoop/Spark** big-data clusters | "Spark/Hadoop big-data processing" |
| **QuickSight** | BI dashboards / visualization | "dashboards," "visualize data" |
| **OpenSearch** | Search + log analytics | "full-text search," "search logs" |
| **MSK** | Managed Apache Kafka | "managed Kafka," "migrate Kafka" |

**Distinctions:** Athena = *query S3 ad-hoc, serverless* vs Redshift = *load into a warehouse for heavy repeated BI*. Glue = *serverless ETL* vs EMR = *full big-data clusters you control*.
**Pipeline pattern:** `Kinesis Firehose → S3 → Glue/Athena → QuickSight` (stream in → land → transform/query → visualize).
**Key reflex:** "data **warehouse**" → always **Redshift** (EMR is never a warehouse).

---

## Part 2 — Migration & Transfer

**Snow Family** — physical devices to move large data **offline** (Snowball = TB–PB; Snowmobile = EB). For 200 TB on a modest line, internet upload takes *months*; Snowball ships in days, encrypted at rest with KMS, integrity-verified, no internet exposure.
> *Huge data + weak bandwidth + one-time → Snowball.* Cost: flat per-job fee, no ongoing cost.

**DataSync** — automated **online** transfer + ongoing sync to S3/EFS/FSx. Built-in checksum verification, preserves file structure, TLS in transit, managed.
> *Decent bandwidth + automated/ongoing verified transfer → DataSync.* Cost: per-GB.

**DMS (Database Migration Service)** — migrate a **live** database with minimal downtime.
- **Full load** then **CDC (Change Data Capture)** — keeps the AWS target in lockstep with the live source while customers keep using the source. Cut over in seconds.
- **Homogeneous** (SQL Server → RDS SQL Server) = direct. **Heterogeneous** (SQL Server → Aurora/PostgreSQL) = run **SCT (Schema Conversion Tool)** first.
- Built-in **data validation** (confirms source = target). **Fallback** is free: source is untouched until cutover.
> *Migrate a live DB, minimal/zero downtime → DMS (+ SCT if engine changes).* Cost: replication instance hours while migrating, then off.

**Storage Gateway** — hybrid storage (File/Volume/Tape) so on-prem apps use AWS storage / replace tape backups.
**Transfer Family** — managed SFTP/FTPS/FTP into S3 or EFS.

---

## Part 3 — Connectivity (deep dives)

### VPC Peering — Prod ↔ Analytics
Private 1-to-1 link between two VPCs; traffic stays on the AWS backbone, uses private IPs.
**Setup:** request → accept (works cross-account), then **add routes on BOTH sides** (each route table points the other's CIDR to `pcx-…`), then **SGs must allow** (can reference the peer's SG in-region).
**Gotchas (exam gold):**
- **Not transitive** — A↔B and B↔C does NOT give A↔C. Each pair needs its own peering.
- **No overlapping CIDRs** — must be distinct ranges or it won't route.
- Works cross-region.
**Cost:** no hourly/processing fee — only data transfer. *Cheapest for a few VPCs.*
> *Two VPCs, private, cheapest → Peering.*

### Transit Gateway — the hub
Hub-and-spoke router connecting many VPCs + on-prem centrally. Peering mesh = **n(n-1)/2** connections (7 VPCs = 21, explodes); TGW = **n** (7 = 7, linear).
**Cost:** per **attachment** + per-**GB processed** — you pay a premium for operational simplicity at scale.
> *Many VPCs / "simplify topology" → Transit Gateway.*

### Direct Connect — the datacenter's private line
A **dedicated physical** circuit on-prem → AWS, bypassing the internet. Consistent latency/bandwidth (1/10/100 Gbps).
**Pieces:** **DX location** (colo meet-me) · **Dedicated vs Hosted** connection · **VIFs**: **Private** (→ VPC), **Public** (→ S3/public services over the line), **Transit** (→ Transit Gateway) · **Direct Connect Gateway** (reach multiple regions / the TGW).
**Gotchas (exam gold):**
- **Weeks–months to provision** → if "need it now," it's NOT DX.
- **NOT encrypted by default** → "private AND encrypted" = **DX + VPN over it** (or MACsec).
- **One DX = single point of failure** → needs dual DX or a VPN backup for HA.
**Cost:** port-hours + **cheaper egress rate** than internet + colo/cross-connect fees. Expensive upfront; *volume + consistency justify it*.
> *Hybrid + high bandwidth + consistent + lower egress at volume → Direct Connect.*

### Site-to-Site VPN — instant, encrypted, backup
**Encrypted IPsec tunnel** over the public internet.
**Pieces:** **Customer Gateway (CGW)** = your side (on-prem router + public IP) · **Virtual Private Gateway (VGW)** = AWS side (or terminate on Transit Gateway) · **two tunnels** (different AZs) for redundancy · **BGP** for dynamic routing + auto-failover.
**Three combos (exam gold):**
1. **Interim** while DX provisions (up in minutes vs weeks).
2. **Backup/failover** for DX (BGP reroutes if DX drops) — cheap HA insurance.
3. **VPN over DX** → private **and** encrypted.
**Weakness:** rides the internet → variable performance.
**Cost:** ~$0.05/hr + data transfer. Cheap.
> *Encrypted hybrid link, fast + cheap → VPN. The interim, the backup, and the encryption layer for DX.*

### Global Accelerator vs CloudFront — getting users in
- **CloudFront** = CDN, **caches content** at the edge (HTTP/S). Static/dynamic web, media, downloads. *Usually saves cost* (offloads origin, cheaper egress).
- **Global Accelerator** = no caching; **static Anycast IPs** routing over the AWS backbone to the **nearest healthy regional endpoint**; supports **TCP/UDP**; fast multi-region failover. For gaming/real-time/non-HTTP. *Adds cost* (you pay for performance + failover).
> *Cache web content → CloudFront. Non-HTTP (TCP/UDP), gaming, static IPs, regional failover → Global Accelerator.*
> One-liner: **CloudFront caches content; Global Accelerator accelerates connections to your endpoints.**
> **Trap:** Global Accelerator is for *users → your AWS app* — NEVER for on-prem↔AWS hybrid data transfer (that's DX/VPN).

---

## Part 4 — The cost lens (Domain 4)

- **Egress is the silent killer:** data **OUT** of AWS to the internet is charged; **IN** is free. Cross-AZ and cross-region traffic also cost. "AWS → on-prem" is egress.
- **NAT Gateway** = hourly + per-GB processed → the #1 surprise bill. **Gateway VPC Endpoint for S3 is free** and bypasses NAT for S3 traffic.
- **Snowball** = flat per-job (beats bandwidth upgrade / DataSync for huge one-time moves).
- **VPN** = cheap; **Direct Connect** = expensive but *lowers egress at volume* + consistent → justified by volume.
- **Peering** = no hourly fee (cheapest for few VPCs); **Transit Gateway** = per-attachment + per-GB (buys scale).
- **CloudFront** usually *saves* (caching offloads origin); **Global Accelerator** *adds* cost.

---

## Part 5 — Exam reflexes (accumulated this session)

```
Huge data + weak bandwidth + one-time          → Snowball
Ongoing automated verified transfer            → DataSync
Live DB, minimal downtime                       → DMS (+ SCT if engine changes)
Two VPCs, private, cheapest                      → VPC Peering
Many VPCs / simplify topology                   → Transit Gateway
Hybrid: consistent + high bandwidth + low egress → Direct Connect
"private AND encrypted"                          → Direct Connect + VPN
Need hybrid connectivity NOW / DX backup        → Site-to-Site VPN
Cache web/media content (HTTP/S)                 → CloudFront
TCP/UDP, gaming, static IPs, regional failover   → Global Accelerator
NAT bill too high (S3 traffic)                   → S3 Gateway Endpoint (free)
"data warehouse"                                 → Redshift
Spikes / don't lose work / decouple             → SQS
Fault-tolerant + interruptible + cheapest        → Spot
```

---

## Part 6 — Diagnostic baseline

Day-1 cross-domain diagnostic: **8/10**. Strong: Security (2/2), Performance (3/3). Two narrow gaps, both fixed:
- **SQS decoupling** — "loses work during spikes" → buffer with a queue, not vertical scaling.
- **Spot** — "fault-tolerant + interruptible + cheapest" → Spot, not On-Demand.

---

*Part of my AWS SAA journey toward Cloud Security Engineering. This is exam-prep breadth + connectivity depth, learned use-case-first with cost reasoning baked in. Remaining: Global Accelerator hands-on intuition, a clean encrypted RDS build, and timed practice exams.*
