# 🌐 EC2 Networking — Deep Dive

> Layer-by-layer architect-level coverage of EC2 networking fundamentals: IP addresses, NAT, Elastic IPs, and Elastic Network Interfaces. With security-first patterns and real-world failover scenarios.

---

## 🎯 What is an IP Address?

**An IP address is a unique identifier for a device on a network** — like a postal address for your computer.

Without an IP, the network has no way to deliver traffic to a device.

### Format (IPv4):
```
192.168.1.100
 │   │   │  │
 4 numbers separated by dots, each 0-255
```

Total: 32 bits = ~4 billion possible addresses (limited resource!)

---

## 🌍 Public vs Private IPs

The **fundamental distinction** in networking:

### 🌍 Public IPs
- Visible to the entire **public internet**
- Routable globally
- Like your **home street address** — anyone can reach you

### 🏠 Private IPs
- Only visible **within a private network**
- NOT routable on the public internet
- Like your **apartment number** — meaningless without the building's street address

---

## 📋 Private IP Ranges (Memorize for Exam!)

There are **3 specific ranges** reserved globally for private use:

```
10.0.0.0   – 10.255.255.255       (10.0.0.0/8)     — ~16 million IPs
172.16.0.0 – 172.31.255.255       (172.16.0.0/12)  — ~1 million IPs
192.168.0.0 – 192.168.255.255     (192.168.0.0/16) — ~65,000 IPs
```

**Memory hook:**
- `10.x.x.x` = Big private network (most enterprises, AWS VPCs)
- `172.16-31.x.x` = Medium private network (note: ONLY 16-31, not all of 172!)
- `192.168.x.x` = Small private network (home routers!)

If an IP starts with these → it's **PRIVATE**, not on the public internet.

### Quick Identification Test:

| IP | Type | Why |
|----|------|-----|
| `10.0.5.23` | Private | Starts with `10.` |
| `8.8.8.8` | Public | Outside private ranges |
| `192.168.1.1` | Private | Starts with `192.168.` |
| `54.190.122.5` | Public | Outside private ranges |
| `172.20.10.50` | Private | In `172.16-31.x.x` range |
| `172.50.1.1` | Public | NOT in 16-31 range |

---

## 🔄 NAT (Network Address Translation)

How does a device with a private IP talk to the public internet?

### The Translation Process:

```
Your Laptop (192.168.1.50)
   ↓ sends "Search for cats" to Google
Your Router
   ↓ NAT replaces source IP: 192.168.1.50 → 73.45.21.99 (router's public IP)
Internet → Google (8.8.8.8)
   ↓ Google sees only 73.45.21.99
   ↓ Google sends response back to 73.45.21.99
Your Router
   ↓ NAT translates back: 73.45.21.99 → 192.168.1.50
Your Laptop receives response ✅
```

### Key Insight:

**External services NEVER see private IPs.** They only know about public IPs. The router/NAT does the translation magic behind the scenes.

### In AWS:

The same pattern applies:

```
Internet (sees only public IPs)
       ↓
🌐 NAT Gateway / Internet Gateway
       ↓
Private Subnet
   ├── EC2 #1: 10.0.5.10 (private only)
   ├── EC2 #2: 10.0.5.11 (private only)
   └── EC2 #3: 10.0.5.12 (private only)
```

🛡️ **Security benefit:** External attackers cannot directly reach private IPs. Only the gateway is exposed (and it's hardened).

---

## 🔑 EC2 Public IP Behavior (Critical Detail!)

When you launch an EC2 instance in AWS, it gets:

### 1️⃣ Private IP (Always Assigned)
- 🏠 Used for VPC-internal communication
- 🔒 Not reachable from internet
- 📅 **Stays the SAME** for the life of the instance

### 2️⃣ Public IP (Optional)
- 🌍 Used for internet-facing communication
- ⚡ **Changes every time you stop/start the instance!**
- ⚠️ Comes from a shared AWS pool

### Why Public IPs Change:

AWS uses a **shared pool** of public IPs (limited resource):
- ⏸️ When you stop EC2: AWS reclaims the public IP
- 🔄 Returns IP to the pool for other customers
- 🆕 When you start: AWS assigns a NEW public IP

**Real-world consequences:**
- 🔴 DNS records pointing to old IP → broken
- 🔴 Whitelisted IPs at partners → broken
- 🔴 External integrations → broken

---

## 🎯 Elastic IP (EIP) — The Solution

**Elastic IP = a static, persistent Public IP that YOU own and control.**

### Key Properties:

1. ✅ **Persistent** — survives stop/start cycles
2. ✅ **Yours** — assigned to your AWS account, not the EC2
3. ✅ **Movable** — can detach from one EC2 and attach to another (failover!)
4. ⚠️ **Limited** — only 5 EIPs per region per account by default
5. 💰 **Cost** — FREE while attached to running EC2. **CHARGED** if unattached or attached to stopped EC2

---

## 💰 The "Charged When Idle" Trap (Heavily Tested!)

| Scenario | Charged? |
|----------|----------|
| EIP attached to **running** EC2 | ✅ FREE |
| EIP attached to **stopped** EC2 | ❌ CHARGED (~$3.65/month) |
| EIP allocated but **NOT attached** | ❌ CHARGED (~$3.65/month) |
| Multiple EIPs on one instance (only 1 free) | ❌ Extra ones charged |

🎯 **Why does AWS charge for unused EIPs?**
To prevent IPv4 hoarding. Public IPs are limited globally — AWS charges to encourage releasing unused ones back to the pool.

🛡️ **Architect rule:** *"If you don't need it, RELEASE it. Don't just keep it allocated."*

---

## 🎯 When to Use Elastic IPs

### ✅ Use EIP when:
- Need a stable public IP for DNS records
- Need to whitelist a specific IP at external partners
- Need fast failover capability (move IP to standby EC2)
- External systems are hardcoded to your IP

### ❌ DON'T use EIP when:
- You can use **DNS** instead (Route 53) — better practice!
- You're using a **Load Balancer** (has its own DNS)
- You can use a **NAT Gateway** (managed, scalable)

### Modern Architect Pattern:

```
Old Way (EIP-based):           Modern Way:
EIP → Single EC2              Route 53 → Load Balancer → Multiple EC2s

Problem with EIPs:             Benefits of modern way:
- Tied to one EC2              - Hides IP changes behind DNS
- Limited to 5/region          - Auto-scaling friendly
- Manual failover              - Automatic failover via ALB
- Operational burden           - Cleaner architecture
```

🎯 **Architect insight:** *EIPs are a legacy pattern. Use them only when truly necessary (e.g., partner whitelist requirements). Otherwise, prefer Load Balancers + DNS.*

---

## 🔌 ENI (Elastic Network Interface)

**ENI = a virtual network card attached to your EC2 instance.**

### What an ENI Holds:

```
EC2 Instance
    │
    └── ENI (eth0) — primary network interface
         ├── Private IP: 10.0.5.23 (1 primary + multiple secondary)
         ├── Public IP: 54.123.45.67 (if assigned)
         ├── Elastic IP: (if attached)
         ├── MAC address (hardware identifier)
         ├── Security Group(s) ← attached to ENI, not EC2!
         └── Source/destination check
```

### Key Insights:

#### 1. ENIs Can Be Detached and Re-Attached

Unlike a physical network card, an ENI can be:
- 🔓 Detached from one EC2
- 🔄 Attached to a different EC2

**Failover use case:**
```
EC2-A is down ❌
   ↓
Detach ENI from EC2-A
   ↓
Attach ENI to EC2-B
   ↓
EC2-B inherits SAME private/public IPs and security groups ✅
DNS records still work — no propagation delay needed!
```

#### 2. Multiple ENIs Per Instance

Larger EC2 instances support multiple ENIs. Use cases:

| Scenario | Why Multi-ENI Helps |
|----------|---------------------|
| Web server with admin interface | Public ENI + private admin ENI |
| Database with backup network | App traffic + backup traffic separated |
| Network appliances (firewalls) | Multiple ENIs for traffic inspection |
| Failover scenarios | Move ENI between instances |

#### 3. Security Groups Attach to ENIs (Not EC2 directly!)

```
Common misconception:  EC2 → Security Group
Reality:               EC2 → ENI → Security Group
```

This means an EC2 with multiple ENIs can have **different security policies** on each interface.

---

## 🛡️ Security-First Patterns

### Pattern 1: Network Segmentation via Multiple ENIs

```
Web Server EC2
├── ENI-1 (Public Subnet) — Web traffic, web-sg
├── ENI-2 (Private Subnet) — Admin/SSH, admin-sg
└── ENI-3 (Management Subnet) — Monitoring, monitor-sg
```

**Benefits:**
- ✅ Defense in depth at network layer
- ✅ Compromise of one interface doesn't expose others
- ✅ Different SGs per traffic type
- ✅ Cleaner audit and monitoring

### Pattern 2: ENI-Based Failover

```
Normal:    DNS → EIP → ENI on EC2-PRIMARY ✅
Failure:   EC2-PRIMARY fails ❌
Failover:  Detach ENI → Attach to EC2-STANDBY ✅
Result:    DNS → EIP → ENI on EC2-STANDBY ✅
           (No DNS change needed!)
```

🛡️ **Why this is brilliant:**
- Fast failover (seconds, not minutes)
- No DNS propagation delay
- All security groups follow the ENI
- Same security posture maintained

### Pattern 3: Private-First Architecture

```
Internet
   ↓
[Load Balancer] (public-facing)
   ↓
[Private Subnet]
├── App EC2 (private IP only)
└── Database (private IP only)
```

🛡️ **Architect rule:** Put as much as possible in private subnets. Only what NEEDS to be public should have public IPs.

---

## 🎯 Common Exam Patterns

| Scenario | Best Solution |
|----------|--------------|
| EC2 needs stable public IP | **Elastic IP** |
| Application requires partner IP whitelist | EIP + ensure attached to running EC2 |
| Multi-AZ web app, scalable | **Route 53 + Load Balancer** (NOT EIP) |
| Fast failover without DNS change | **Detach ENI from failed EC2, attach to standby** |
| Database in same VPC needs to reach app | Use **private IPs** (free, no NAT) |
| EC2 in private subnet needs internet | **NAT Gateway** with EIP |
| Want to identify if IP is private | Check if it starts with 10.x, 172.16-31.x, or 192.168.x |
| Surprise AWS bill from EIP | Likely attached to stopped EC2 or unattached |

---

## 🛡️ Security-First Takeaways

1. **Private IPs are inherently safer** — not reachable from internet
2. **Use private subnets** for databases, app servers, internal services
3. **NAT Gateway** for outbound-only internet from private resources
4. **Elastic IPs cost money when idle** — release when not needed
5. **ENIs enable network segmentation** — defense in depth at network layer
6. **DNS + Load Balancer > EIP** for modern scalable architectures
7. **EIP detachment for failover** is faster than DNS propagation

---

## 🎯 Architect Decision Framework

### When designing EC2 networking:

```
Does the EC2 need to receive traffic from the internet?
├── YES → Public subnet + Public IP / EIP / Load Balancer
└── NO  → Private subnet (preferred for security)

Does the EC2 in private subnet need outbound internet?
├── YES → NAT Gateway with EIP (managed, scalable)
└── NO  → No internet access (most secure)

Need stable IP for external integrations?
├── YES → Elastic IP (but consider DNS + LB instead)
└── NO  → Regular Public IP is fine

Need fast failover capability?
├── YES → Use ENI detach/attach pattern OR Load Balancer
└── NO  → Standard deployment

Need network segmentation?
├── YES → Multiple ENIs with different Security Groups
└── NO  → Single ENI is fine
```

---

## 📚 Concept Connections

This networking foundation connects to many other AWS topics:

- **Security Groups** → Attached to ENIs (not directly to EC2!)
- **VPC** → Where private IPs live (next deep-dive topic)
- **Route 53** → DNS pointing to EIPs or Load Balancers
- **Load Balancers** → Distribute traffic across multiple ENIs
- **NAT Gateway** → Translates private IPs for internet outbound
- **VPC Endpoints** → Private connectivity to AWS services without NAT

---

*Part of my AWS Solutions Architect Associate journey toward Cloud Security specialization. Next layers in this thread: EC2 Hibernate and Placement Groups.*
