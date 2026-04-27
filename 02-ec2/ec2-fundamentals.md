# 🖥️ EC2 Fundamentals

> Notes from my AWS SAA learning journey — covering EC2 with a security-first lens.

---

## 🎯 What is EC2?

**EC2 = Elastic Compute Cloud** — virtual servers in AWS that you can rent by the second.

### The Mental Model

Think of EC2 like **renting an apartment instead of buying a house**:

| Traditional Server | EC2 |
|-------------------|-----|
| Capital expense (buy hardware) | Operational expense (rent capacity) |
| 2-3 month procurement | 60-second launch |
| Fixed size | Resize anytime |
| Your data center | AWS handles physical infrastructure |
| Pay even when idle | Pay only while running |

### Why It Matters

EC2 is the foundation of AWS — almost every other AWS service either runs on EC2 internally or interacts with EC2 instances. Understanding EC2 well makes the rest of AWS easier to learn.

---

## 🧱 The 7 Things You Configure When Launching EC2

Every EC2 instance is defined by these 7 choices:

1. **AMI** — operating system + software image
2. **Instance Type** — CPU/memory/network specs
3. **Network** — which VPC and subnet
4. **Storage** — EBS volumes (disk for the instance)
5. **Security Group** — firewall rules (inbound/outbound)
6. **Key Pair** — SSH login credentials
7. **IAM Role** — what AWS APIs this instance can call

🛡️ **Security insight:** 3 of these 7 choices are security-related (Security Group, Key Pair, IAM Role). EC2 is fundamentally about secure access + secure compute.

---

## 📋 Instance Type Naming

Instance types follow this pattern:

```
m5.large
│  │ │
│  │ └── Size (nano, micro, small, medium, large, xlarge, 2xlarge...)
│  └──── Generation (5 = 5th gen — newer = better price/performance)
└─────── Family (purpose: m=general, c=compute, r=memory, t=burstable)
```

### The 5 Instance Families

| Family | Letter | Use Case | Example |
|--------|--------|----------|---------|
| **General Purpose** | M, T | Balanced workloads | Web servers, small databases |
| **Compute Optimized** | C | High CPU performance | Batch processing, video encoding |
| **Memory Optimized** | R, X | Lots of RAM | In-memory databases (Redis, SAP HANA) |
| **Storage Optimized** | I, D | Fast disk I/O | Big data, NoSQL, data warehouses |
| **Accelerated Computing** | P, G, Inf | GPUs / ML accelerators | AI/ML training, graphics |

**Memory aid:** *"My Cat Refuses Insulting Promotions"*
- **M** = General (My)
- **C** = Compute (Cat)
- **R** = Memory (Refuses)
- **I** = Storage (Insulting)
- **P** = GPU/Accelerated (Promotions)

---

## ⚡ Burstable vs Fixed Performance

### T-Family (Burstable)

- ✅ Cheap baseline performance
- ✅ Earns "CPU credits" when idle
- ✅ Spends credits during CPU spikes
- ⚠️ If credits run out → throttled to baseline (slow!)
- 🎯 **Best for:** Dev/test environments, low-traffic apps, occasional workloads

### M-Family (Fixed Performance)

- ✅ Consistent CPU at all times
- ✅ No credits, no throttling
- 💰 More expensive than T baseline
- 🎯 **Best for:** Production workloads needing predictable performance

### Common Exam Trap

> *"Application on `t3.micro` is slow despite low CPU usage. Why?"*

→ CPU credits exhausted → instance throttled. Either upgrade T-family with more baseline, enable `unlimited` mode, or move to M-family.

---

## 💰 The 5 Pricing Models

| Model | Discount | Commitment | Best For |
|-------|----------|------------|----------|
| **On-Demand** | 0% | None | Unpredictable workloads, short-term |
| **Reserved** (1-3 yr) | Up to 72% | 1 or 3 years | Steady-state production |
| **Savings Plans** | Up to 72% | $/hour commitment | Flexible across instance types |
| **Spot Instances** | Up to 90% | Can be terminated anytime | Fault-tolerant batch jobs |
| **Dedicated Hosts** | Varies | Physical server | Compliance / BYOL |

### Pricing Decision Tree

```
Workload runs 24/7 in production for 1+ year?
├── YES → Reserved Instances or Savings Plans
└── NO ↓

Can the workload survive 2-min interruption?
├── YES → Spot Instances 💰
└── NO ↓

Need physical isolation (compliance/BYOL)?
├── YES → Dedicated Hosts
└── NO ↓

Default → On-Demand
```

### Exam Trigger Phrases

| Phrase | Pricing Model |
|--------|--------------|
| "Fault-tolerant" / "Can be interrupted" | **Spot** |
| "Steady-state" / "Always running" | **Reserved** / Savings Plans |
| "Unpredictable workload" | **On-Demand** |
| "Compliance / BYOL / dedicated hardware" | **Dedicated Hosts** |
| "Save up to 90%" / "Bid for capacity" | **Spot** |
| "1 or 3-year commitment" | **Reserved** |

---

## 🛡️ AMI Security (Critical for Production)

An AMI is a snapshot of: OS + pre-installed software + configuration.

### AMI Types and Trust Levels

| AMI Type | Trust | Use in Production? |
|----------|-------|-------------------|
| **AWS-provided** (Amazon Linux, Ubuntu official) | High | ✅ Always safe |
| **Marketplace AMIs** (verified vendors) | High | ✅ Generally safe |
| **Community AMIs** | Unknown | ⚠️ Risk! |
| **Custom AMIs** (your own golden image) | High | ✅ Best for production |

### Why Community AMIs Are Dangerous

Community AMIs can contain:
- 🚨 **Malware** (cryptominers, backdoors)
- 🚨 **Pre-installed SSH keys** belonging to attackers
- 🚨 **Outdated packages** with known CVEs
- 🚨 **Hidden services** that exfiltrate data

### Architect Rule

**Never use Community AMIs in production.** Use only:
1. AWS-official AMIs
2. AWS Marketplace verified AMIs
3. Your own custom-built AMIs ("golden images")

This is heavily tested in the **Security Specialty exam**.

---

## 🌍 Where Does an EC2 Instance Live?

The hierarchy matters for availability and security:

```
🌍 AWS Cloud
└── 🌎 Region (e.g., us-east-1 — geographic area)
    └── 📍 Availability Zone (e.g., us-east-1a — physical data center)
        └── 🌐 VPC (your virtual network)
            └── 🌐 Subnet (a slice of VPC, lives in 1 AZ)
                └── 🖥️ EC2 Instance
```

### Architect Considerations

- **Region choice** → affects latency, cost, compliance (e.g., GDPR)
- **AZ choice** → affects availability (1 AZ failure = single instance dies)
- **VPC/Subnet** → affects network security and routing

### High Availability Pattern

For production:
- ✅ Deploy across **multiple Availability Zones** (if AZ-1a fails, AZ-1b is up)
- ✅ Use **public subnets** for load balancers
- ✅ Use **private subnets** for app servers and databases
- ✅ Use **VPC endpoints** to access AWS services without internet

---

## 🛡️ Security-First Lens on EC2

### Architect Questions to Ask Before Launch

1. **Workload pattern?** → determines instance family + pricing model
2. **Availability requirement?** → single AZ or multi-AZ?
3. **Security & compliance needs?** → public/private subnet, encryption, IAM role
4. **Budget constraints?** → On-Demand vs Reserved vs Spot
5. **Data sensitivity?** → custom AMI? encrypted EBS? security groups?

### Production Best Practices

- ✅ **Never use Community AMIs** for production
- ✅ **Use IAM Roles** (Instance Profiles) — never hardcode access keys
- ✅ **Multi-AZ deployment** for high availability
- ✅ **Custom AMIs** ("golden images") with security patches pre-applied
- ✅ **Encrypt EBS volumes** at rest
- ✅ **Tight Security Groups** — least privilege for inbound rules
- ✅ **CloudWatch monitoring** + budget alerts
- ✅ **Don't use Spot for stateful workloads** — they can vanish in 2 minutes

---

## 🎯 The Architect Mental Model

A regular engineer thinks: *"Launch a server."*

An architect thinks:
1. Which Region? (latency, compliance, cost)
2. Which AZs? (high availability)
3. Which VPC/Subnet? (network design)
4. Which AMI? (security, software)
5. Which instance type? (workload pattern)
6. Which pricing model? (cost optimization)
7. Which Security Group? (firewall rules)
8. Which IAM Role? (AWS API access)
9. Backup strategy? (EBS snapshots, AMI rotation)
10. Monitoring? (CloudWatch alarms)

**That's the difference between launching a server and designing a deployment.**

---

## 📚 Common Exam Scenarios

| Scenario | Answer |
|----------|--------|
| Dev environment with occasional spikes | T3 instance (burstable) |
| ML training job, fault-tolerant, minimize cost | Spot Instances |
| Production payment system, 99.99% uptime | Reserved Instances + Multi-AZ + M5 family |
| Compliance requires physical isolation | Dedicated Hosts |
| Workload runs 24/7 for 3 years | Reserved Instances (3-year, all upfront) |
| Web app needs to access S3 | IAM Role attached via Instance Profile |
| Slow performance on `t3.micro` despite low CPU usage | CPU credits exhausted (burstable throttled) |

---

*Part of my AWS Solutions Architect Associate journey toward Cloud Security specialization.*
