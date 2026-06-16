# ⚙️ Resilience + Performance (Domains 2 & 3, ~50%) — SAA Exam Study Guide

> **Use-case-first**, continuing **"Atlas Games."** Two goals underneath everything: **Resilience = stay UP** (Domain 2, 26%), **Performance = stay FAST** (Domain 3, 24%). Together ~half the exam.

---

## 🛟 RESILIENCE (Domain 2)

### Disaster Recovery strategies — the spectrum

Driven by two metrics:
- **RTO (Recovery Time Objective)** = how long you can be **down** before recovery. (Downtime tolerance.)
- **RPO (Recovery Point Objective)** = how much **data** you can afford to lose. (Distance to last good copy.)

Smaller RTO/RPO = faster recovery + less data loss = **more cost.** Four strategies, cheapest/slowest → priciest/instant:

| Strategy | What's pre-running in DR | RTO | Cost |
|---|---|---|---|
| **Backup & Restore** | Only backups | Hours | Cheapest |
| **Pilot Light** | DB replicating live, app off | ~10s of min | Low |
| **Warm Standby** | Small but full copy always live | Minutes | Higher |
| **Multi-Site Active/Active** | Full prod live in multiple regions | ≈ 0 | Priciest |

**Architect move (cost lens):** pick *per workload*, not one for the whole company. Atlas's live multiplayer game → Warm Standby/Multi-Site (downtime = lost players/revenue). Back-office analytics → Backup & Restore (cheap, hours of downtime fine).

> **Reflex:** cheapest + hours OK → **Backup & Restore.** Near-zero downtime, pay for it → **Multi-Site Active/Active.**

### Multi-AZ is NOT disaster recovery
- **Multi-AZ** = **high availability within one region** — survives an *AZ* failure, automatic failover. (RDS Multi-AZ, ASG across AZs, ELB.)
- **DR strategies** = surviving an entire *region* going down (multi-region).

> **"AZ fails, auto-failover, same region" → Multi-AZ (HA).** **"Whole region down / cross-region" → a DR strategy.**

### Multi-AZ vs Read Replica (same engine, different jobs)
- **Multi-AZ** = failover / HA (standby copy, not for reads).
- **Read Replica** = scale **read** throughput (and can be cross-region for DR).

### Decoupling = resilience
When work **spikes and gets dropped**, the resilient fix is to **decouple with a queue (SQS)** so the queue absorbs the surge — workers pull at their own pace. **Buffer, don't bulk up** (vertical scaling just delays the same failure).

---

## 🚀 PERFORMANCE (Domain 3)

### The caching trio — distinguished by *what you cache in front of*

| Cache | In front of… | Use |
|---|---|---|
| **ElastiCache** (Redis/Memcached) | A **database** (relational) / sessions | Offload repeated reads, sub-ms; session store; leaderboards (Redis sorted sets) |
| **DAX** (DynamoDB Accelerator) | **DynamoDB** | Microsecond reads, drop-in |
| **CloudFront** | **Web/media content** at the edge | Global content delivery, offloads origin |

> **Reflex:** relational DB / sessions → **ElastiCache.** DynamoDB → **DAX.** Web content at the edge → **CloudFront.**

### Read scaling
- **Read replicas** — read-only copies serve read traffic, offloading the primary (scales read throughput).
- **ElastiCache** — cache hot results in memory so repeated reads skip the DB entirely.
- Often used **together**: cache the hottest data, spread the rest across replicas.

### Storage performance (EBS types)
| Type | Use |
|---|---|
| **gp3** (general SSD) | Sensible default; balanced, cost-effective |
| **io2 / io2 Block Express** (Provisioned IOPS SSD) | **High IOPS + low latency** databases |
| **st1 / sc1** (HDD) | Big sequential throughput / cold, cheap |
| **Instance store** | Highest performance but **ephemeral** (lost on stop) — scratch only |

> **Reflex:** high-IOPS DB → **io2.** Default → **gp3.** Cheap throughput → **HDD.** Max-speed temp scratch → **instance store.**

### Compute under load
- **Auto Scaling** scales **out** (more instances, horizontal) to meet demand — serves both performance *and* resilience.

---

## 🔴 Resilience + Performance reflex sheet

```
Cheapest DR, hours of downtime OK            → Backup & Restore
Near-zero downtime DR, will pay              → Multi-Site Active/Active
Minimal-cost warm DR, minutes RTO            → Pilot Light / Warm Standby
AZ failure, auto-failover, same region       → Multi-AZ (HA, not DR!)
Whole region down / cross-region recovery    → a DR strategy
Scale READ capacity of RDS                    → Read Replicas
Spikes / dropping work / decouple             → SQS queue (buffer, don't bulk up)
Cache relational DB reads / sessions          → ElastiCache (Redis/Memcached)
Cache DynamoDB reads (microsecond)            → DAX
Cache web/media content globally              → CloudFront
High IOPS + low latency storage               → io2 (Provisioned IOPS)
Default block storage                         → gp3
Temp/scratch max performance                  → instance store
Meet variable demand                          → Auto Scaling (scale out)
```

---

## ✅ Consolidation check

Day-4 combined question set (qualifier style): **8/8.** Notable wins: both DR-strategy questions (brand-new this session) correct, and the **SQS decoupling** question — a *Day-1 diagnostic miss* — now answered correctly. Reflex converted from gap to strength.

---

*Part of my AWS SAA journey toward Cloud Security / AI Security. All four domains now covered (Security 7/7, Resilience/Performance 8/8). Remaining: Day 5 Cost domain + first full timed mock, then Tutorials Dojo practice exams to a consistent 80%+ before booking.*
