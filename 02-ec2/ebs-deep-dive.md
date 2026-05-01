# 💾 EBS (Elastic Block Store) — Deep Dive

> Layer-by-layer architect-level coverage of EBS: volume types, snapshots, Fast Snapshot Restore, and Multi-Attach. With security-first patterns and disaster recovery strategies.

---

## 🎯 What is EBS?

**EBS = Elastic Block Store** — virtual hard drives you attach to EC2 instances.

### The Real-World Analogy:

```
Your laptop = EC2 instance
Your laptop's SSD/HDD = EBS volume
```

Just like your laptop needs storage to run the OS, store files, and boot up, your EC2 needs **persistent storage** — that's what EBS provides.

---

## 🌟 Key Properties of EBS

### 1. Persistent Storage 💾
- Data **survives** instance stop/start
- Unlike RAM (lost on stop) or instance store (lost on stop/terminate)

### 2. Network-Attached 🌐
- EBS volumes are NOT physically inside the EC2 server
- They live on AWS storage servers, connected via network
- Think of it like an external hard drive over a super-fast network

### 3. AZ-Locked 📍 (Critical!)
- An EBS volume exists in **ONE specific Availability Zone**
- You **CANNOT** attach an EBS volume to an EC2 in a different AZ
- Volume in `us-east-1a` ↔ EC2 must also be in `us-east-1a`

### 4. Detachable & Re-attachable 🔄
- Detach EBS from one EC2
- Attach to another EC2 (in same AZ!)
- Useful for failover, instance replacement

### 5. Replicated within AZ ✅
- Within a single AZ, EBS data is automatically replicated for durability
- AWS handles this — protects against hardware failure

---

## 🛡️ The AZ-Locking Rule (Heavily Tested!)

This is one of the most-tested concepts on the SAA exam:

```
✅ Allowed:
EC2 in us-east-1a + EBS volume in us-east-1a → can attach

❌ NOT allowed:
EC2 in us-east-1a + EBS volume in us-east-1b → CANNOT attach
```

### Why?

EBS volumes are physically located in specific AZ data centers. Cross-AZ network latency would degrade EBS performance significantly. AWS enforces AZ-locking for performance reasons.

🎯 **Architect rule:** *"EBS volumes and EC2 instances must be in the SAME Availability Zone."*

### How to Move EBS Across AZs?

You can't directly. But you CAN:
1. 📸 **Snapshot** the EBS volume
2. 🌍 Copy snapshot to another AZ (or even another region!)
3. ✨ **Create a new volume** from the snapshot in the new AZ

This is the **Snapshot pattern** — covered in Layer 3.

---

## 📊 EBS Volume Types

EBS isn't one-size-fits-all. AWS offers different volume types optimized for different workloads.

### The 4 Main Types:

#### 1️⃣ General Purpose SSD (gp2 / gp3) ⚡

**The default choice for most workloads.**

- ✅ Balanced price/performance
- ✅ Good for boot volumes, dev/test, small databases
- ✅ **gp3** is newer (better) — more IOPS at lower cost than gp2
- 🎯 **Use for:** Most general workloads, web servers, app servers

#### 2️⃣ Provisioned IOPS SSD (io1 / io2) 🚀

**Highest performance, for I/O-intensive workloads.**

- ✅ Up to 64,000 IOPS per volume (io2)
- ✅ Most expensive
- ✅ Designed for **mission-critical databases**
- 🎯 **Use for:** Production databases (Oracle, SQL Server, MySQL with heavy load)

#### 3️⃣ Throughput Optimized HDD (st1) 📊

**For big data, sequential reads (like streaming through files).**

- ✅ Cheap, large capacity
- ✅ Great for **sequential** workloads (read big files top to bottom)
- ❌ **CANNOT be a boot volume**
- 🎯 **Use for:** Big data analytics, log processing, data warehouses

#### 4️⃣ Cold HDD (sc1) ❄️

**Cheapest, for rarely accessed data.**

- ✅ Lowest cost
- ✅ Infrequently accessed data
- ❌ **CANNOT be a boot volume**
- 🎯 **Use for:** Archive data, infrequent backups, cold storage

---

## 🎯 Quick Comparison Table:

| Type | Performance | Cost | Use Case | Boot Volume? |
|------|-------------|------|----------|--------------|
| **gp3 / gp2** ⚡ | Good (3,000-16,000 IOPS) | Medium | Most workloads, default | ✅ Yes |
| **io1 / io2** 🚀 | Excellent (up to 64,000 IOPS) | High | Mission-critical DBs | ✅ Yes |
| **st1** 📊 | Sequential throughput | Low | Big data, logs | ❌ No |
| **sc1** ❄️ | Lowest | Lowest | Archive, cold storage | ❌ No |

---

## 🎯 IOPS vs Throughput (Architect Distinction):

| Term | What It Measures | Best For |
|------|------------------|----------|
| **IOPS** | Number of read/write operations per second | Databases (small random reads/writes) |
| **Throughput** | Total data transferred per second (MB/s) | Big data (large sequential reads) |

🎯 **For databases:** IOPS matters more (lots of small operations) → use **io2**
🎯 **For big data:** Throughput matters more (large sequential reads) → use **st1**

---

## 🎯 The Architect Decision Pattern:

```
Boot volume needed?
├── YES → gp3 (default) or io2 (if performance critical)
└── NO ↓

Database with high I/O requirements?
├── YES → io1 / io2 (Provisioned IOPS)
└── NO ↓

Big data / sequential reads?
├── YES → st1 (Throughput Optimized HDD)
└── NO ↓

Cold/archive data, accessed rarely?
├── YES → sc1 (Cold HDD)
└── NO  → gp3 (general purpose default)
```

---

## 🎯 Memory Hook for Volume Types:

```
gp = General Purpose (the default for most things)
io = Input/Output (high performance for databases)
st = Streaming Throughput (big data, sequential reads)
sc = Saving Costs (cold storage, cheapest)
```

🎯 **Critical exam fact:** Only **gp** and **io** can be **boot volumes**. **st1** and **sc1** are HDD-based and too slow for booting an OS.

---

## 📸 EBS Snapshots (The Backup Mechanism)

**A snapshot is a point-in-time backup of an EBS volume.** Think of it like a photograph of your hard drive at a specific moment.

### Real-World Analogy:

```
Your laptop's SSD = EBS volume
Time Machine backup = EBS snapshot
```

If your SSD fails, you restore from Time Machine. Same with EBS snapshots.

---

## 🌟 Key Properties of Snapshots:

### 1. Stored in S3 (Behind the Scenes) 📦
- Snapshots are stored in **AWS S3** infrastructure
- You don't see them as S3 buckets — AWS manages this
- Highly durable (99.999999999% — 11 nines!)

### 2. Region-Wide (Not AZ-locked!) 🌍
- Unlike EBS volumes (AZ-locked), snapshots are **region-wide**
- This is HUGE — enables data movement across AZs!

### 3. Incremental 📊
- First snapshot = full backup of the volume
- Subsequent snapshots = only the **changed blocks** (much smaller!)
- You only pay for the actual storage used

### 4. Cross-Region Copy 🌐
- Can copy snapshots to other regions
- Enables **disaster recovery** scenarios
- Great for compliance (data must exist in multiple regions)

---

## 🎯 The "Move EBS Across AZs" Pattern:

```
EBS volume in us-east-1a
   ↓ (1) Create snapshot
Snapshot stored in S3 (region-wide)
   ↓ (2) Restore in different AZ
New EBS volume in us-east-1b ✅
```

🎯 **Use case:** Disaster recovery, cross-AZ migrations, instance replacement.

---

## 🛡️ The Cross-Region Disaster Recovery Pattern:

```
Production Region (us-east-1)         DR Region (us-west-2)
┌──────────────────────┐              ┌──────────────────────┐
│ EBS Volume           │              │                      │
│        ↓             │              │                      │
│ Snapshot             │ →copy→        │ Copied Snapshot      │
└──────────────────────┘              │        ↓             │
                                       │ EBS Volume (new)     │
                                       └──────────────────────┘
```

If the entire `us-east-1` region goes down, you can restore from the snapshot in `us-west-2`. ✅

---

## 🛡️ Snapshot Best Practices:

### 1. Automate with Data Lifecycle Manager (DLM) ⚙️
- Schedule automatic snapshots (daily, weekly)
- Auto-delete old snapshots (retention policies)
- No manual work — set it and forget it

### 2. Tag Snapshots 🏷️
- Tag with: workload, retention, owner, environment
- Easier to manage hundreds of snapshots
- Example: `Environment=Prod, Owner=DBA-team, Retention=90days`

### 3. Encrypt Snapshots 🔐
- If volume is encrypted → snapshot is automatically encrypted
- Encryption is **inherited** across snapshot/restore operations
- Critical for sensitive data (PCI, HIPAA, GDPR)

### 4. Don't Forget to Delete Old Ones! 💰
- Snapshots cost money (S3 storage)
- Old snapshots accumulate fast
- Use lifecycle policies to auto-delete after retention period

---

## 🚨 Common Snapshot Pitfalls:

### Pitfall 1: Taking Snapshots of Mounted Volumes
- Can cause **inconsistent state** if volume is being written to
- **Best practice:** Stop the EC2 OR detach volume before snapshot for "clean" state
- For databases: Use database-level backups (RDS snapshots) instead

### Pitfall 2: Not Testing Restore!
- Backups are useless if restore doesn't work
- Periodically test: snapshot → restore → verify data
- This is a **disaster recovery best practice**

### Pitfall 3: Forgetting About Cost
- Old snapshots accumulate silently
- A 100GB snapshot taken weekly for 1 year = 5,200GB of storage costs
- Use **DLM** to auto-delete

---

## ⚡ Fast Snapshot Restore (FSR)

### The Problem FSR Solves:

When you create a new EBS volume from a snapshot, AWS does something subtle:

```
1. Restore EBS volume from snapshot
2. Volume appears available IMMEDIATELY ✅
3. You attach it to EC2 ✅
4. EC2 reads first block → ⚠️ SLOW! (block being lazy-loaded from S3)
5. EC2 reads second block → ⚠️ SLOW! (lazy-loading)
6. Eventually all blocks loaded → ✅ Performance normal
```

The restored volume has **degraded performance for the first reads** until all data is "warmed up" from S3.

### When This Causes Problems:
- 🚀 You need **immediate full performance** on a restored volume
- ⏰ You can't wait for blocks to be lazy-loaded
- 🎯 Critical scenarios: **disaster recovery, time-sensitive workloads, dev environment refreshes**

---

## 🎯 The FSR Solution:

**Fast Snapshot Restore = pre-warm the volume so it has full performance immediately.**

```
With FSR enabled on a snapshot:
1. Restore EBS volume from snapshot
2. Volume appears available IMMEDIATELY ✅
3. Volume is ALREADY pre-warmed (no lazy-loading) ✅
4. You attach it to EC2 ✅
5. EC2 reads any block → ⚡ FAST IMMEDIATELY (full performance)
```

🌟 **Result:** No performance degradation on first reads.

---

## 🌟 Key Properties of FSR:

### 1. Per-AZ Configuration 📍
- FSR is enabled on a **per-snapshot, per-AZ** basis
- If you might restore in `us-east-1a` AND `us-east-1b` → enable FSR for both AZs
- Each AZ enablement adds cost

### 2. Costs Money 💰
- FSR has a **per-hour cost per AZ** (~$0.75/hour per AZ)
- Only enable for snapshots where instant performance matters

### 3. Maximum 50 Snapshots Per Region 📊
- Soft limit, but designed to prevent abuse
- Used selectively, not on every snapshot

### 4. Activation Time ⏰
- After enabling FSR, takes ~60 minutes to fully prepare
- Plan ahead for DR scenarios

---

## 🛡️ When to Use FSR:

✅ **Use FSR for:**
- 🚨 **Critical DR scenarios** (need instant performance for restored volumes)
- ⚡ **Time-sensitive recoveries** (database failover, compliance audits)
- 🎯 **Frequently-restored snapshots** (e.g., dev environment templates)
- 🏭 **Production restoration** with strict RTO (Recovery Time Objective)

❌ **DON'T use FSR for:**
- 💰 Cost-sensitive workloads (FSR costs money continuously)
- 📊 Rarely-restored snapshots
- 🛠️ Dev/test environments where lazy-loading is acceptable

---

## 🛡️ Real-World FSR Use Case:

A bank has strict regulatory requirements:
- 🚨 If primary database fails, must be operational within **15 minutes** (RTO = 15 min)
- 💾 Database is 500GB
- 🐌 Without FSR: lazy-loading 500GB = could take hours to reach full performance
- ✅ **With FSR enabled:** Restore EBS from snapshot → instant full performance → meets 15-min RTO

🎯 **Architect rule:** *"FSR is a premium feature. Use it surgically, not blanketly."*

---

## 🔗 EBS Multi-Attach (Advanced)

### The Standard EBS Rule:

Normally, an EBS volume has a **strict 1:1 relationship** with EC2:

```
EBS Volume → ONE EC2 instance at a time

❌ You CANNOT attach the same EBS volume to multiple EC2s simultaneously (default)
```

This prevents **data corruption** — if two EC2s wrote to the same volume simultaneously, files would clash.

---

## 🌟 But Some Workloads Need Shared Storage...

For specific cluster scenarios, AWS offers **EBS Multi-Attach.**

### What is EBS Multi-Attach?

**Multi-Attach allows ONE EBS volume to be attached to MULTIPLE EC2 instances simultaneously.**

```
With Multi-Attach enabled:

EBS Volume (1 volume)
    │
    ├── Attached to EC2-A
    ├── Attached to EC2-B
    └── Attached to EC2-C (up to 16 instances)
```

All instances can **read AND write** to the same volume concurrently.

---

## ⚠️ Strict Limitations:

This is NOT a free-for-all. AWS imposes strict rules:

### 1. **Only io1 / io2 Volumes** ✅
- Must be Provisioned IOPS SSD
- Other volume types (gp3, st1, sc1) **don't support** Multi-Attach

### 2. **Same AZ Only** 📍
- All EC2 instances and the EBS volume must be in **same Availability Zone**
- Cross-AZ Multi-Attach = NOT supported

### 3. **Maximum 16 Instances** 🔢
- Can attach to up to 16 EC2s simultaneously
- More than 16? Need different solution (EFS, FSx)

### 4. **File System Must Support Concurrent Writes** 🛡️
- Standard ext4 / NTFS = ❌ Will corrupt data!
- Need **cluster-aware** file systems:
  - GFS2 (Global File System)
  - OCFS2 (Oracle Cluster File System)
  - These coordinate writes between instances

### 5. **Specific Use Cases Only** 🎯
- AWS designed Multi-Attach for **high-availability cluster scenarios**
- NOT for general "shared storage" needs (use EFS for that)

---

## 🚨 The Critical Warning:

🚨 **Multi-Attach with a non-cluster file system = DATA CORRUPTION!**

If two EC2s write to the same block at the same time without coordination, your data is **destroyed**. This is why cluster-aware file systems are mandatory.

🎯 **Most companies should NOT use Multi-Attach** — it's an edge case for specific cluster databases.

---

## 🎯 When to Use Multi-Attach vs Alternatives:

| Need | Best Solution |
|------|---------------|
| **Shared file system** (general purpose) | **Amazon EFS** (NFS-based, easy) |
| **Windows shared storage** | **Amazon FSx for Windows** |
| **High-performance cluster database** (Oracle RAC) | **EBS Multi-Attach** |
| **Distributed storage** (Hadoop) | **HDFS over EBS** (per node) |

🎯 **Architect rule:** *"Multi-Attach is for clustered databases that REQUIRE shared block storage. For everything else, use EFS or FSx."*

---

## 🎯 The AWS Storage Hierarchy:

```
File-level access needed?
├── Standard NFS (Linux file shares) → Amazon EFS
├── Windows SMB shares → Amazon FSx for Windows
├── High-performance compute → FSx for Lustre
└── Block-level (raw disk) → EBS

Block-level shared between cluster nodes?
├── Standard cluster DB → EBS Multi-Attach
└── Otherwise → Use EFS instead

Object storage (HTTP API)?
└── S3 (not a file system, different paradigm)
```

---

## 🎯 Common Exam Patterns

| Scenario | Best Solution |
|----------|--------------|
| EC2 needs persistent storage | **EBS volume** (gp3 default) |
| Production MySQL database | **io2** (high IOPS) |
| Big data sequential reads | **st1** (throughput optimized) |
| Old archive data | **sc1** (cheapest) |
| Cross-AZ EBS migration | **Snapshot → restore in target AZ** |
| Cross-region disaster recovery | **Snapshot + cross-region copy** |
| Database fails, must restore in 5 min | **Enable FSR** on the DR snapshot |
| Web servers need shared file system | **Amazon EFS** (NOT Multi-Attach!) |
| Oracle RAC cluster database | **EBS Multi-Attach** with io2 |
| Boot volume for any EC2 | **gp3** (default) |

---

## 🛡️ Security-First Takeaways

1. **Always encrypt EBS volumes** containing sensitive data
2. **Encryption is inherited** through snapshots (encrypted volume = encrypted snapshots)
3. **Snapshot encryption is automatic** when volume is encrypted
4. **Cross-account snapshot sharing** requires KMS key sharing for encrypted snapshots
5. **Backup hygiene** — automated snapshots + cross-region copy = disaster recovery foundation
6. **Test restoration** quarterly — backups are worthless if restore doesn't work
7. **Old snapshots cost money** — implement lifecycle policies
8. **EBS is AZ-locked** — design for redundancy across AZs from the start
9. **FSR for tight RTO** — use selectively to control cost
10. **Multi-Attach is dangerous** without cluster file systems — prefer EFS for shared file access

---

## 🎯 Architect Decision Framework

### When designing EBS for an application:

```
Need persistent storage attached to EC2?
├── YES → EBS volume in same AZ as EC2
└── NO  → Consider instance store (ephemeral, no charge)

What's the workload pattern?
├── General purpose → gp3
├── High IOPS database → io2
├── Sequential big data → st1
├── Cold archive → sc1
└── Boot volume → gp3 or io2 only

Backup strategy?
├── Manual snapshots → Use Data Lifecycle Manager (DLM) for automation
├── Region disaster recovery → Cross-region snapshot copy
├── Tight RTO requirement → Enable FSR on critical snapshots
└── Compliance → Encrypted volumes + retained snapshots

Need shared storage?
├── File-level (NFS) → Amazon EFS
├── Windows file share → FSx for Windows
├── Cluster database (Oracle RAC) → EBS Multi-Attach with io2
└── Otherwise → Don't use Multi-Attach (use EFS)

Cross-AZ requirement?
└── Use snapshots to migrate (volumes are AZ-locked)
```

---

## 📚 Concept Connections

This EBS foundation connects to many other AWS topics:

- **EC2** → Requires EBS for persistent storage
- **Hibernate** → Requires encrypted EBS root volume
- **Snapshots** → Stored in S3 (managed by AWS)
- **DLM (Data Lifecycle Manager)** → Automates snapshot management
- **KMS** → Encrypts EBS volumes and snapshots
- **EFS** → Alternative for shared file systems
- **AMI** → Built from EBS snapshots
- **Auto Scaling** → Each instance needs its own EBS volume

---

*Part of my AWS Solutions Architect Associate journey toward Cloud Security specialization. Complete EBS thread covering 5 layers from fundamentals through advanced cluster patterns.*
