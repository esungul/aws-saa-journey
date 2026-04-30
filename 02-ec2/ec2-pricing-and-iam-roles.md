# 💰 EC2 Pricing Models & IAM Roles — Architect Deep-Dive

> Complete coverage of EC2 pricing models, capacity options, and the security pattern of attaching IAM Roles to EC2 instances. With architect-level decision frameworks and security-first lens.

---

## 🎯 Why Multiple Pricing Models?

AWS offers different pricing options because **different applications have different needs**:

- **Predictable workloads** → cost commitment makes sense
- **Variable workloads** → flexibility matters more than discount
- **Fault-tolerant batch jobs** → can accept interruption for huge savings
- **Compliance requirements** → may need dedicated hardware

The architect's job: **match the pricing model to the workload pattern.**

---

## 📊 The 5 Standard Pricing Models

### 1. On-Demand
- 💰 Pay per second (Linux) / hour (Windows)
- 📅 No commitment, no discount
- ⚡ Most flexible, most expensive
- ✅ **Use for:** Unpredictable workloads, testing, short-term needs

### 2. Reserved Instances (RI)
- 📅 1 or 3-year commitment to a specific instance type
- 💰 Up to 72% discount
- 🔒 Locked-in instance type/region

**Three sub-types:**

| Sub-Type | Flexibility | Discount | Use When |
|----------|-------------|----------|----------|
| **Standard** | Locked | Highest (~72%) | Stable workload, no changes |
| **Convertible** | Can change instance type | Lower (~54%) | Need flexibility |
| **Scheduled** | Specific time windows | Variable | Predictable peak hours |

### 3. Savings Plans ⭐ (Modern Alternative)
- 💵 Commit to a **dollar amount per hour** for 1-3 years
- 💰 Up to 72% discount
- ✨ **Flexible** — can switch instance types, regions, even Lambda/Fargate

**Two types:**

| Type | Flexibility | Discount | Use Case |
|------|-------------|----------|----------|
| **Compute Savings Plans** | EC2 + Lambda + Fargate (any region) | Up to ~66% | Maximum flexibility ⭐ |
| **EC2 Instance Savings Plans** | Specific EC2 family in region | Up to ~72% | Higher discount, less flexible |

**Architect insight:** AWS recommends Savings Plans over Reserved Instances for most modern workloads — same potential discount, much more flexibility.

### 4. Spot Instances
- 💰 Up to 90% discount vs On-Demand
- ⚠️ AWS can terminate with **2-minute warning**
- 🎯 Uses AWS's spare capacity

**Trigger phrases (use Spot when you see):**
- "Fault-tolerant"
- "Can be interrupted"
- "Batch processing"
- "Stateless"
- "Minimize cost"
- "Big data / ML training"

**AVOID Spot when you see:**
- "Critical / Production"
- "99.99% uptime"
- "Stateful database"
- "Real-time API"
- "Customer-facing"
- "Payment processing"

### 5. Dedicated Hosts vs Dedicated Instances ⚠️ THE TRAP

Both prevent shared tenancy. The difference:

| Feature | Dedicated **Host** 🏢 | Dedicated **Instance** 🏠 |
|---------|----------------------|---------------------------|
| What you rent | Entire physical server | Just an instance with dedicated HW |
| Hardware visibility | ✅ See CPU sockets, host ID | ❌ No visibility |
| Multiple instances per server | ✅ Yes | ❌ No |
| BYOL support | ✅ Yes | ❌ No |
| Per-core licensing | ✅ Yes | ❌ No |
| Cost | $$$ Most expensive | $$ Less expensive |

**Decision framework:**

```
Need "dedicated hardware" (no shared tenancy)?
│
├── BYOL / per-core licensing mentioned?
│   ├── YES → Dedicated HOST 🏢 ($$$)
│   └── NO  → Dedicated INSTANCE 🏠 ($$)
│
└── No mention of dedicated hardware?
    └── Regular EC2 (shared tenancy) ✅
```

**The exam trap:** Most candidates pick "Dedicated Host" when "Dedicated Instance" is correct. **Watch for BYOL keywords** — without them, Dedicated Instance is the cheaper, sufficient choice.

---

## 🎯 Capacity Reservation (NOT a Discount Model!)

Often confused with Reserved Instances, but completely different purpose.

### What it is:
- 📍 Reserve EC2 capacity in a specific Availability Zone
- ✅ AWS **guarantees** the capacity is available when needed
- 💰 You pay full **On-Demand price** (no discount!)
- ⏰ No commitment, cancel anytime

### Capacity Reservation vs Reserved Instance:

| Feature | Capacity Reservation | Reserved Instance |
|---------|---------------------|-------------------|
| Discount? | ❌ No (full On-Demand price) | ✅ Up to 72% |
| Capacity guarantee? | ✅ YES | ❌ No (just discount) |
| Commitment? | None | 1 or 3 years |
| Use case | Availability assurance | Cost optimization |

### When to use:
- 🎯 **Disaster Recovery** scenarios (must be able to launch DR instances)
- 🎯 **Predictable peak loads** (Black Friday, major events)
- 🎯 **Compliance/SLA** requirements with availability guarantees
- 🎯 **Critical events** (live broadcasts, product launches)

**Architect power move:** Combine RI (discount) + Capacity Reservation (guarantee) = best of both worlds for mission-critical workloads.

---

## 🚀 Spot Fleet — Managed Spot at Scale

### The Problem:
Plain Spot Instances can all get terminated together if they're from the same pool. If you need 100 instances, putting all eggs in one basket = disaster.

### The Solution:
A **Spot Fleet** = managed group of Spot Instances spread across:
- 🌍 Multiple instance types
- 📍 Multiple Availability Zones
- 🎯 Multiple Spot pools

If one pool gets reclaimed, AWS automatically replaces from another.

### The 4 Allocation Strategies (Critical for Exam!)

| Strategy | Optimizes For | When to Use |
|----------|---------------|-------------|
| **lowestPrice** | 💰 Cheapest pools | Maximum savings, OK with interruptions |
| **diversified** | 🌐 Spread across many pools | Maximum resilience |
| **capacityOptimized** ⭐ | 📈 Pools with most capacity | **Lowest interruption rate** |
| **priceCapacityOptimized** ⭐⭐ | ⚖️ Balance price + capacity | **Modern recommendation** |

### Trigger Word → Strategy Map:

| Question Says | Strategy |
|---------------|----------|
| "Minimize cost above all else" | `lowestPrice` |
| "Maximum resilience / diversification" | `diversified` |
| "Lowest interruption rate" | `capacityOptimized` |
| "Balanced cost and stability" | `priceCapacityOptimized` ⭐ |

**Architect default:** When in doubt → `priceCapacityOptimized` (AWS's modern recommendation).

---

## 🛡️ IAM Role on EC2 — The Security Pattern

### The Problem It Solves:
Your EC2 web server needs to:
- Read files from S3
- Write logs to CloudWatch
- Read database credentials from Secrets Manager

How does it authenticate to AWS?

❌ **Wrong way:** Hardcoded access keys (in code, environment variables, config files)
✅ **Right way:** Attach an IAM Role to the EC2 instance

### How It Works:

```
1. Create IAM Role with permissions
   - Trust Policy: "EC2 service can assume me"
   - Permissions: "S3 read, CloudWatch write, etc."

2. Attach Role to EC2 (AWS wraps it in an Instance Profile)

3. EC2 exposes temp credentials at:
   http://169.254.169.254/latest/meta-data/iam/security-credentials/

4. AWS SDK on EC2 automatically:
   - Detects the Instance Profile
   - Pulls temp credentials
   - Auto-refreshes when they expire

5. NO HUMAN INVOLVEMENT — fully automatic ✨
```

### Key Concepts:

- **Instance Profile** = container that holds the IAM Role for EC2
- **Limit:** 1 EC2 = 1 Instance Profile = 1 IAM Role at a time
- **Magic URL:** `169.254.169.254` (Instance Metadata Service / IMDS)
- **Temp credentials** auto-rotate (typically every few hours)

---

## 🚨 IMDSv1 vs IMDSv2 — Critical Security Topic

### IMDSv1 (Legacy, Vulnerable):
- ⚠️ Anyone with network access can fetch credentials
- ⚠️ No authentication required
- ⚠️ Vulnerable to **SSRF (Server-Side Request Forgery)** attacks

### IMDSv2 (Modern, Secure):
- ✅ Requires session token (PUT request first)
- ✅ Token can't be relayed via SSRF
- ✅ Protects against the most common cloud attack pattern

### The Capital One Breach (2019) — Real-World Example:

1. Capital One had a misconfigured AWS WAF
2. Attacker exploited an SSRF vulnerability
3. Tricked the EC2 to query `169.254.169.254`
4. **IMDSv1** returned the IAM Role's credentials
5. Attacker accessed S3 buckets
6. **100M customer records stolen**
7. **$80M+ fine + reputational damage**

🛡️ **Root cause:** IMDSv1 was enabled (default at the time)
🛡️ **Architect rule:** Always enforce IMDSv2 in production

---

## 🛡️ Why Hardcoded Credentials Are Always Wrong

Even environment variables on EC2 are dangerous:

| Problem | Why It Matters |
|---------|----------------|
| **Long-term credentials** | Don't auto-rotate, leaked once = leaked forever |
| **Visible to multiple sources** | `env` command, logs, crash dumps, backups |
| **Static** | No way to revoke without manual action |
| **Bypass auditing** | CloudTrail can't link actions to specific instances |
| **Don't scale** | 100 EC2s = 100 places keys exist |

**With IAM Role on EC2:**
- ✅ Auto-rotation (handled by AWS)
- ✅ No keys stored anywhere
- ✅ Clean audit trail (role + session in CloudTrail)
- ✅ Scales infinitely (every new instance gets the role)
- ✅ Revocable (delete role or detach)

---

## 🎯 Architect Decision Framework

### Pricing Model Selection:

```
Workload runs 24/7 in production for 1+ years?
├── YES → Reserved Instance OR Savings Plan
│         └── Need flexibility (instance types, regions, services)?
│             ├── YES → Compute Savings Plan ⭐
│             └── NO  → Standard Reserved Instance
└── NO ↓

Can the workload survive 2-min interruption?
├── YES → Spot Instances 💰
│         └── Need many instances? → Spot Fleet
│             └── Default strategy: priceCapacityOptimized ⭐
└── NO ↓

Need physical isolation (compliance/BYOL)?
├── BYOL / per-core licensing? → Dedicated Host 🏢
├── No BYOL, just dedicated HW? → Dedicated Instance 🏠
└── Default → On-Demand
```

### Need availability guarantee for critical scenarios?
**Capacity Reservation** (regardless of pricing model — can be combined!)

---

## 🎯 Common Exam Patterns

| Scenario | Best Answer |
|----------|-------------|
| New app, unknown traffic | On-Demand |
| Bank's 24/7 system, 5+ years, fixed instance | Standard Reserved (3-year) |
| 24/7 production but expects to migrate to newer instances | Compute Savings Plan |
| ML training job, fault-tolerant, minimize cost | Spot Instances |
| Video rendering Spot Fleet, minimize interruptions | `capacityOptimized` strategy |
| Oracle Database with existing per-core license | Dedicated Host (BYOL) |
| Compliance requires dedicated hardware (no BYOL) | Dedicated Instance (the trap!) |
| DR scenario, must launch when needed | Capacity Reservation |
| EC2 needs to access S3, CloudWatch, Secrets Manager | IAM Role with least-privilege permissions + IMDSv2 |

---

## 🛡️ Security-First Takeaways

1. **Match pricing model to workload pattern** — not just lowest cost
2. **Spot ≠ sensitive data** — never put long-term sensitive data on Spot Instances
3. **Dedicated Host = BYOL trigger** — only use when licensing requires it
4. **Capacity Reservation for DR** — guaranteed launch is critical for incident response
5. **Always use IAM Roles on EC2** — never hardcode credentials
6. **Enforce IMDSv2** — protects against SSRF attacks (Capital One lesson)
7. **Apply least privilege** — only grant the specific permissions the EC2 needs
8. **Use Compute Savings Plans for flexibility** — locks you into spending, not architecture

---

*Part of my AWS Solutions Architect Associate journey toward Cloud Security specialization.*
