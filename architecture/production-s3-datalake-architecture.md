# Production-Grade Multi-Region Data Lake on Amazon S3
### A real-world system design walkthrough (financial-services grade)

> Goal of this doc: not "pass the exam" — understand how a large, regulated company
> actually designs a data platform on S3, and *why* each decision is made.
> Every choice below maps to a real requirement (cost, resilience, security, compliance).

---

## 1. The business context (why this system exists)

A financial-services company ingests **transaction data** from many sources (apps,
partner feeds, internal systems). They need to:

- Store everything cheaply and at massive scale (petabytes, growing).
- Run analytics on it (fraud detection, reporting, ML).
- Survive a full AWS Region outage.
- Satisfy regulators: an **isolated second copy**, **immutable records for 7 years**,
  and a full **audit trail** of who touched what (including encryption keys).
- Keep costs sane as data ages.

S3 is the backbone because it gives near-infinite scale, 11 nines of durability,
built-in encryption, and integrates with the entire analytics/ML stack.

---

## 2. The layered design (the "zones" pattern)

Real data lakes are NOT one bucket. They use a **multi-zone (medallion) pattern** —
data flows through quality stages, each its own bucket with its own rules.

```
            INGEST                  PROCESS                 SERVE
  sources -> [ RAW / Bronze ] --> [ CLEAN / Silver ] --> [ CURATED / Gold ]
              (landing zone)       (validated, dedup)     (business-ready)
```

| Zone | Bucket purpose | Who writes | Who reads | Typical storage class |
|------|----------------|-----------|-----------|----------------------|
| **Raw (Bronze)** | Exact copy of source data, untouched | Ingestion pipelines | ETL jobs only | Standard -> tier fast (rarely re-read) |
| **Clean (Silver)** | Validated, de-duplicated, schema-applied | Glue ETL | ETL + some analysts | Standard / Intelligent-Tiering |
| **Curated (Gold)** | Business-ready tables, aggregates | Glue ETL | Analysts, BI, ML | Standard (hot, queried daily) |

**Why separate buckets/zones?**
- Different access controls per zone (raw data is sensitive; curated is broadly read).
- Different lifecycle rules per zone (raw can archive aggressively; gold stays hot).
- Blast-radius isolation: a bad job in ETL can't corrupt the raw landing data.
- Clear data lineage for auditors.

> Naming convention (real companies do this):
> `company-datalake-raw-us-east-1-prod`, `company-datalake-clean-...`, etc.
> Region + environment in the name prevents cross-env accidents.

---

## 3. Ingestion (how data gets in)

Multiple paths depending on source:

- **Streaming** (app events, clickstream): Kinesis Data Firehose -> writes batched
  files directly into the Raw bucket. Firehose handles buffering, compression, partitioning.
- **Batch / files** (partner feeds): uploaded to Raw via **S3 Transfer Acceleration**
  if sources are geographically distant (edge upload -> AWS backbone).
- **Database sources**: AWS DMS (change data capture) lands records into Raw.

**Event-driven processing kicks off automatically:**
```
Object lands in Raw bucket
   -> S3 Event Notification (ObjectCreated)
       -> triggers Lambda (light validation / routing)  OR
       -> drops message in SQS (decouple heavy ETL)      OR
       -> EventBridge rule (advanced routing to Glue/Step Functions)
```
This is the "push" pattern — no polling, no idle servers. You pay only when data arrives.

---

## 4. Partitioning + file format (the performance decision most people miss)

How you LAY OUT data in S3 dictates query cost and speed.

- **Partitioning**: organize objects by query pattern, e.g.
  `s3://.../clean/transactions/year=2026/month=06/day=20/region=latam/`
  Athena/Spark then scan only the relevant partitions instead of the whole dataset
  => massively less data scanned => cheaper + faster.
- **Columnar format**: store as **Parquet** (or ORC), not raw JSON/CSV.
  Columnar = read only the columns a query needs + heavy compression.
  A query that scans 1 TB of JSON might scan 50 GB of Parquet for the same answer.
- **Compaction**: many tiny files kill performance (overhead per object). A
  compaction job merges small files into larger ~128MB-1GB objects.

> Cost lens: Athena charges per TB scanned. Partitioning + Parquet can cut a query's
> cost by 10-100x. This is the single biggest lever in data-lake economics.

---

## 5. Cataloging + querying (making the lake usable)

A pile of files isn't useful until something knows the schema.

- **AWS Glue Data Catalog** = the central metadata store (databases, tables, schemas).
- **Glue Crawlers / ETL jobs** populate and update the catalog.
- **AWS Lake Formation** = fine-grained access control on top of the catalog
  (table-, column-, even row-level permissions) — critical for "analysts can see
  these columns but not the PII columns."
- **Query engines** read S3 directly:
  - **Athena** — serverless SQL over S3 (ad-hoc).
  - **Redshift Spectrum** — warehouse queries that reach into S3.
  - **EMR / Spark** — heavy distributed processing & ML feature pipelines.

---

## 6. Security architecture (defense in depth)

Layers, outermost to innermost:

1. **Block Public Access = ON** at the account level and every bucket. Non-negotiable.
   Overrides any misconfigured policy. This is the #1 leak defense.
2. **Bucket policies** — explicit allow only to specific roles/services.
   Deny any request that isn't over HTTPS (`aws:SecureTransport = false` -> Deny).
3. **IAM roles per pipeline** — ingestion role, ETL role, analyst role. Each least-privilege.
   No long-lived keys; services assume roles, get temporary credentials.
4. **VPC Gateway Endpoint for S3** — traffic between your VPC and S3 stays on the
   AWS private network, never traverses the internet (also saves NAT cost).
5. **Encryption at rest = SSE-KMS** with customer-managed keys (CMKs).
   - Why KMS over SSE-S3? Because KMS logs every key use to **CloudTrail** =>
     auditors can prove exactly who decrypted what, and when.
   - Separate CMKs per zone/data-classification => granular control + revocation.
6. **Encryption in transit** — SSL/TLS always (enforced by bucket policy).
7. **Lake Formation** — column/row-level governance for analysts.
8. **CloudTrail (data events) + GuardDuty (S3 protection) + Macie**
   (auto-discovers PII/PHI in buckets) => detection layer.

---

## 7. Resilience + DR (surviving a region loss)

- **Versioning = ON** on all buckets (required for replication; also undo-protection).
- **Cross-Region Replication (CRR)** Raw + Curated to a second Region
  (e.g., us-east-1 -> us-west-2). Asynchronous, automatic, preserves names/metadata/
  versions/ACLs, encrypted in transit.
- **S3 Replication Time Control (RTC)** when you need a predictable RPO — replicates
  the bulk of objects within 15 minutes, with an SLA + metrics.
- **Existing data** (before CRR was switched on) -> backfilled with **S3 Batch Replication.**
- **DR copy stays in Standard** for instant recovery, EVEN THOUGH the source tiers to
  Glacier — because lifecycle rules are per-bucket and DO NOT replicate. (This is the
  subtle, important interaction: cost-optimize the source without crippling the DR copy.)
- Catalog + Lake Formation permissions are ALSO replicated to the DR region — otherwise
  you'd have data but no way to query or govern it after failover.

> Real result (Bridgewater, AWS case study): S3 Replication took their standby region
> from ~2 weeks behind to ~2 hours behind — >99% resiliency improvement.

---

## 8. Compliance + immutability (the regulator requirements)

- **S3 Object Lock, Compliance mode** on the regulated transaction records:
  WORM (Write Once Read Many). For the 7-year retention, NO ONE — not even root —
  can delete or alter them until retention expires. Defends against insider threat
  AND ransomware (a stolen admin credential still can't destroy the records).
- **Legal Hold** for litigation: lock specific objects indefinitely until released.
- **The isolated second copy** requirement is met by CRR to a different region
  (some regs demand 500+ miles separation and *different* encryption keys — CRR
  supports re-encrypting with a destination-region CMK).
- **AWS Artifact** for compliance reports / BAA; **Audit Manager** to continuously
  assess against frameworks.

---

## 9. Cost optimization (the lens that runs through everything)

- **Lifecycle policies per zone**:
  - Raw: Standard -> Standard-IA (30d) -> Glacier Deep Archive (90d) -> expire (per retention).
  - Curated: stays hot (queried daily); maybe Intelligent-Tiering for unpredictable parts.
- **Intelligent-Tiering** where access patterns are unknown — AWS auto-moves objects,
  no retrieval fees between frequent/infrequent tiers.
- **Abort incomplete multipart uploads** (lifecycle rule, e.g. 7 days) — silent cost killer.
- **Expire old noncurrent versions** — versioning without cleanup bloats cost forever.
- **Partitioning + Parquet** — cut Athena $/query by 10-100x (biggest lever).
- **VPC S3 Gateway Endpoint** — avoid NAT Gateway data-processing charges on S3 traffic.
- **CRR cost trade-off**: you pay 2x storage + cross-region transfer. That's the price
  of regional resilience — a deliberate, justified spend, not waste.

---

## 10. The whole picture (text diagram)

```
  SOURCES (apps, partners, DBs)
      | (Firehose / Transfer Acceleration / DMS)
      v
  +------------------ us-east-1 (PRIMARY) ----------------------+
  |  RAW bucket  --ETL(Glue)-->  CLEAN  --ETL-->  CURATED       |
  |   |  ^ S3 Events -> Lambda/SQS/EventBridge                  |
  |   |  - Versioning ON, SSE-KMS, Block Public Access ON       |
  |   |  - Object Lock (Compliance) on regulated records        |
  |   |  - Lifecycle: tier to IA/Glacier                        |
  |   |                                                         |
  |  Glue Data Catalog + Lake Formation (governance)            |
  |  Athena / Redshift Spectrum / EMR  (query & ML)             |
  |  CloudTrail + GuardDuty + Macie    (detect)                 |
  +------------------------------------------------------------+
      |  S3 Cross-Region Replication (async, versioned, RTC)
      |  + catalog/permissions replication
      v
  +------------------ us-west-2 (DR) --------------------------+
  |  Replica buckets (kept in Standard for instant recovery)  |
  |  Replicated catalog + Lake Formation permissions          |
  |  Can be promoted to primary on regional failure           |
  +-----------------------------------------------------------+
```

---

## 11. Design principles to carry forward (the transferable lessons)

1. **Separate by concern** (zones/buckets) for security, lifecycle, and blast radius.
2. **Layout dictates economics** — partitioning + columnar format beat everything else.
3. **Event-driven, serverless ingestion** — pay per event, no idle fleet.
4. **Defense in depth** — Block Public Access + bucket policy + IAM + KMS + endpoints + detection.
5. **Resilience is deliberate spend** — CRR doubles storage; you do it for the regions
   you actually must protect.
6. **Per-bucket independence** (lifecycle, encryption) is a *feature* — exploit it
   (cheap source, fast DR copy).
7. **Immutability for compliance** — Object Lock Compliance mode is your WORM + ransomware shield.
8. **Replicate metadata too** — data without its catalog/permissions is useless after failover.

---

*Built as a study + portfolio artifact. Maps to AWS whitepapers: "Building Data Lakes
on AWS", "Storage Best Practices for Data and Analytics", and the AWS Disaster Recovery
whitepaper.*
