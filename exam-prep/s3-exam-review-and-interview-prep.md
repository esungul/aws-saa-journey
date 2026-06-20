# Amazon S3 — Exam Review & Interview Prep
### Active-recall study guide (SAA-C03 + interview questions)

> Built from an active-recall session: reasoning first, then confirming. Covers every
> exam-weighted S3 topic plus senior-level interview questions. Pair with the
> `production-s3-datalake-architecture.md` doc for the system-design view.

---

## 1. Encryption (who encrypts, who holds the key, does AWS see plaintext)

The deciding fork: **where does encryption happen, and does AWS ever see your key/plaintext?**

| Option | Who encrypts | Who holds key | AWS sees plaintext? | Pick when |
|--------|-------------|---------------|--------------------|-----------|
| **SSE-S3** | AWS | AWS | Yes | Simplest; no key management wanted |
| **SSE-KMS** | AWS | AWS KMS (you control + **audit via CloudTrail**) | Yes | Must audit *who used the key, when* |
| **SSE-C** | AWS | You send key per request | Yes | You manage keys but want AWS to encrypt (rare) |
| **Client-side (KMS key)** | You | KMS KeyId involved | No | Encrypt before upload, KMS-managed key |
| **Client-side (master key)** | You | You only | No | AWS must see **neither data nor key** |

**Reflexes**
- "Audit who used the encryption key" -> **SSE-KMS** (KMS + CloudTrail).
- "Unencrypted data must never reach AWS" -> **Client-side encryption**.
- "...and keys never sent to AWS either" -> **Client-side with client-side master key**.
- **SSE-C trap**: data still goes to AWS + you send the key -> it's *server-side*, fails "nothing unencrypted reaches AWS."

---

## 2. Storage classes (trade access frequency vs retrieval speed/cost)

| Class | Access pattern | Retrieval | Note |
|-------|---------------|-----------|------|
| **Standard** | Frequent | ms | Default, hot |
| **Standard-IA** | Infrequent, need fast | ms | Cheaper storage, retrieval fee |
| **One Zone-IA** | Infrequent + **re-creatable** | ms | Single AZ, ~20% cheaper than Std-IA |
| **Intelligent-Tiering** | **Unknown/changing** | ms | AWS auto-moves tiers, no retrieval fee |
| **Glacier Instant Retrieval** | Archive, need instant | ms | Rarely touched, instant when needed |
| **Glacier Flexible Retrieval** | Archive | minutes-hours | Cheaper |
| **Glacier Deep Archive** | Coldest, almost never | hours | Cheapest |

**Reflexes**
- "Re-creatable / can tolerate loss / backup exists elsewhere" -> **One Zone-IA**.
- "Unknown access pattern, auto-optimize" -> **Intelligent-Tiering**.
- "Archive but must return instantly (e.g. medical images)" -> **Glacier Instant Retrieval**.
- "7-yr compliance, hours-retrieval fine, cheapest" -> **Glacier Deep Archive**.
- Standard retrieval = **milliseconds** (not minutes). Minutes/hours = Glacier family.
- All classes = 11 nines durability EXCEPT One Zone-IA (single AZ).

---

## 3. Lifecycle policies (automate cost as data ages)

Two action types: **Transition** (move to cheaper class) and **Expiration** (delete).

Example log policy: Standard -> Standard-IA (30d) -> Glacier Deep Archive (90d) -> expire (365d).
- Nuance: 30-day minimum in Standard before Standard-IA transition.

**Advanced (the silent cost killers — high-yield):**
- **Noncurrent version** rules: transition/expire old versions (since the day they became noncurrent).
- **Expired delete markers**: clean up delete markers left with no object behind them.
- **Abort incomplete multipart uploads**: orphaned upload parts cost money invisibly -> abort after N days.
- AWS does NOT auto-clean these — a lifecycle rule must.

**One-line insight**: versioning protects you, but without lifecycle cleanup, old versions + delete markers + failed multipart parts silently bloat cost.

---

## 4. Versioning

- Three states: **Unversioned** (start) -> **Enabled** -> **Suspended** (stop new versions, keep existing). Cannot return to fully unversioned.
- **Current = newest version always.** Rollback = copy an old version to become a new current version (never resurrect in place).
- Protects against accidental delete/overwrite (old versions recoverable until a lifecycle rule expires them).

---

## 5. Replication (CRR / SRR)

- **CRR (Cross-Region)**: different region -> DR, compliance/data-residency, lower latency.
- **SRR (Same-Region)**: same region, different bucket -> log aggregation, prod->test, account separation.
- **Requirement**: versioning ON for BOTH source and destination (replication is version-aware).
- **Not retroactive**: only replicates objects created *after* enabling. Existing data -> **S3 Batch Replication**.
- **Not transitive**: A->B and B->C does not give A->C.
- **Deletes don't propagate** by default (safety); delete markers optionally replicated.
- **S3 Replication Time Control (RTC)**: predictable RPO, replicate within 15 min + SLA.
- **Lifecycle does NOT replicate**: each bucket runs its own lifecycle independently -> source can tier to Glacier while DR replica stays in Standard for instant recovery.

---

## 6. Security

- **Default**: buckets are private (deny by default).
- **Block Public Access**: master switch, ON by default, **overrides** any policy/ACL that tries to make a bucket public. #1 leak defense.
- **Presigned URL**: time-limited link to ONE private object; no AWS account needed; bucket stays private; expires (time-limited, not strictly one-time).
- **Bucket policy** (attached to bucket) vs **IAM policy** (attached to identity).
- Enforce HTTPS via bucket policy (`aws:SecureTransport=false` -> Deny).
- **MFA Delete**: require MFA to delete versions (extra protection).
- **Object Lock (WORM)**: write-once-read-many.
  - **Compliance mode**: nobody (not even root) can delete/alter until retention expires.
  - **Governance mode**: privileged users can override.
  - **Legal Hold**: indefinite lock until explicitly removed.
  - Requires versioning. Ransomware + insider-threat defense.

---

## 7. Other features

- **Transfer Acceleration**: faster long-distance uploads via CloudFront edge -> AWS private backbone. (CloudFront = faster downloads OUT; Transfer Acceleration = faster uploads IN.)
- **Event Notifications**: bucket events (ObjectCreated/Removed) -> Lambda (run code), SQS (queue/decouple), SNS (fan-out). Newer: EventBridge for advanced routing.
- **Static website hosting**: serve HTML/CSS/JS directly, often fronted by CloudFront.
- **Data consistency**: strong read-after-write consistency for all operations.

---

## 8. Design gotchas (architect lens)

- Replication = double storage + cross-region transfer cost (deliberate DR spend).
- Glacier has retrieval cost + minimum storage durations (30/90/180d) -> over-aggressive tiering can cost more.
- Durability (won't lose data) != availability (can reach it now). One Zone-IA trades AZ redundancy.
- Replicate metadata too (Glue Catalog, Lake Formation perms) — data without catalog is useless after failover.

---

## INTERVIEW QUESTIONS (practice cold, then check against sections above)

**Fundamentals**
1. Durability vs availability in S3 — definitions and numbers?
2. Walk through the storage classes and when to pick each.
3. A bucket policy allows public read but objects aren't public. Why?

**Security**
4. SSE-S3 vs SSE-KMS vs client-side — when does an audit requirement force KMS?
5. Give temporary access to one private object without making the bucket public — how?
6. Protect a bucket against ransomware / a compromised admin deleting data?
7. Walk through defense-in-depth on an S3 data lake (the 7 layers).

**Lifecycle & cost**
8. How does versioning create hidden costs, and how do you control them?
9. The three "silent" things a lifecycle policy should clean up?

**Replication & resilience**
10. CRR vs SRR use cases? What must be enabled first?
11. Enabled CRR but old objects didn't replicate — why and the fix?
12. If the source tiers to Glacier, does the DR replica follow? Explain.

**Architecture (senior)**
13. Design an event-driven pipeline for a file uploaded to S3. Guarantee it isn't processed twice (idempotency: message ID -> file hash -> transaction key -> DB unique constraint).
14. Why partition data and use Parquet in a data lake? Cost impact (Athena scans less -> 10-100x cheaper)?

---

## The 7-layer defense-in-depth (memorize this order)

1. **Network isolation** — Block Public Access + VPC Gateway Endpoint (traffic stays private).
2. **Authentication** — who are you (IAM credentials).
3. **Authorization** — least-privilege IAM roles per pipeline/zone.
4. **Encryption** — SSE-KMS at rest (audit via CloudTrail) + TLS in transit.
5. **Immutability** — Object Lock Compliance mode (ransomware/insider defense).
6. **Audit trail** — CloudTrail (API + key usage), stored immutably.
7. **Detection** — GuardDuty (threats) + Macie (PII discovery) + Security Hub (aggregate) + CloudWatch alarms.

*No single layer stops everything; an attacker must defeat all seven simultaneously.*
