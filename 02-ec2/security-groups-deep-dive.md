# 🛡️ EC2 Security Groups — Deep Dive

> Complete architect-level coverage of Security Groups, with security-first patterns and exam-relevant insights.

---

## 🎯 What Is a Security Group?

**A Security Group is a virtual firewall attached to each EC2 instance.** It controls who can talk to your instance — like a bouncer standing at the door checking a guest list.

Key facts:
- 🛡️ Default = **deny** (anything not explicitly allowed is blocked)
- 🚪 Attached at the **instance level** (each EC2 has its own)
- 🔄 **Stateful** (response traffic automatically allowed)
- ✅ **Allow-only** (no deny rules — use NACLs for that)

---

## 🧱 Anatomy of a Security Group Rule

Every rule has 4 fields:

| Field | Purpose | Examples |
|-------|---------|----------|
| **Type** | Pre-defined service shortcut | HTTP, HTTPS, SSH, MySQL |
| **Protocol** | Network protocol | TCP, UDP, ICMP |
| **Port** | Service port number | 80, 443, 22, 3306 |
| **Source / Destination** | Who is allowed | IP, CIDR, or SG reference |

Example rule:

```
Type:     SSH
Protocol: TCP
Port:     22
Source:   203.0.113.45/32
```

Translation: *"Allow SSH (port 22) only from this one specific IP."*

---

## 📚 Common Service Ports (Memorize for Exam)

| Service | Port | Protocol |
|---------|------|----------|
| HTTP | 80 | TCP |
| HTTPS | 443 | TCP |
| SSH | 22 | TCP |
| RDP (Windows) | 3389 | TCP |
| MySQL/Aurora | 3306 | TCP |
| PostgreSQL | 5432 | TCP |
| MongoDB | 27017 | TCP |
| Redis | 6379 | TCP |
| DNS | 53 | UDP/TCP |
| SMTP | 25 / 587 | TCP |

---

## 🌐 Understanding CIDR Notation

CIDR notation defines IP ranges:

```
0.0.0.0/0      →  Entire internet (every IP)
10.0.0.0/16    →  ~65,000 IPs (a VPC)
10.0.1.0/24    →  ~256 IPs (a subnet)
203.0.113.45/32 →  Exactly ONE IP
```

Memory hook:
- `/0` = wide open (everyone)
- `/32` = locked to one specific machine
- Higher number after `/` = more secure (smaller range)

---

## 🔄 Stateful Behavior (THE Most Important Concept)

Security Groups are **stateful** — meaning when traffic comes IN, the response is **automatically allowed** to go OUT, even without an outbound rule.

### Bouncer Analogy

- 🚪 You enter the club (inbound allowed)
- 🤝 Bouncer remembers you came in
- 🚶 When you leave, bouncer says: *"Go ahead — I remember you"*

### Why It Matters

| Connection Step | What SG Does |
|-----------------|--------------|
| User browser → port 80 → server | Inbound rule must allow |
| Server → response → user browser | **Auto-allowed (stateful)** |

You only configure **inbound** — outbound responses are handled automatically. ⭐

### Compare With Stateless (NACL):

- **Stateful (SG):** Inbound allow → response auto-allowed
- **Stateless (NACL):** Must configure BOTH directions explicitly

---

## ⚙️ Default Behavior of a New Security Group

| Direction | Default | Why |
|-----------|---------|-----|
| 🔽 **Inbound** | ❌ DENY all | Safer — prevent accidental exposure |
| 🔼 **Outbound** | ✅ ALLOW all | Convenient — most apps need outbound |

### Architect Note

The default outbound = ALLOW is **convenient but risky** for sensitive workloads. If an attacker compromises the instance:
- 📤 They can exfiltrate data to their server
- 📡 They can reach command-and-control servers
- 🔄 They can pivot to attack other systems

**For high-security workloads** (databases, payment systems, healthcare), lock down outbound to specific destinations only. This is called **"egress lockdown"** and it's a top-tier security pattern.

---

## 🚫 Allow-Only Rule Set

Security Groups can **only allow traffic**. There is no "deny" rule.

### How to "block" something

You don't add deny rules — you simply don't add an allow rule. Default = denied.

### When You Need Explicit Deny

If you need to **block specific IPs** (e.g., known attackers), use **NACLs** at the subnet level. NACLs support both allow and deny rules.

| Use Case | Tool |
|----------|------|
| Allow trusted sources | Security Group |
| Deny known bad actors | **NACL** (subnet level) |
| Block specific IP from entire VPC | **NACL** |

---

## ⭐ SG Referencing — The Architect Superpower

Instead of allowing IP addresses, you can allow **another Security Group** as the source.

### Example

```
db-sg: Allow port 3306 from app-sg
```

Translation: *"Allow MySQL traffic from any instance that has the `app-sg` Security Group attached."*

### Why It's Genius

| With IPs (Manual) | With SG References (Dynamic) |
|-------------------|------------------------------|
| Must update SG when adding new app servers | Works automatically with auto-scaling |
| IP changes = broken access | SG reference is dynamic |
| 50 app servers = 50 rule updates | One rule, scales infinitely |
| High maintenance | Zero maintenance |

### The 3-Tier Pattern

```
Internet
   ↓
[Web Tier]    web-sg: Allow port 80/443 from 0.0.0.0/0
   ↓
[App Tier]    app-sg: Allow port 8080 from web-sg
   ↓
[Database]    db-sg:  Allow port 3306 from app-sg
```

**Result:** Even if a hacker compromises the web server, they cannot reach the database without first compromising the app tier. **Defense in depth.** 🛡️

---

## 🏷️ Multiple SGs Per Instance

You can attach **up to 5 Security Groups** to one EC2 instance. Their rules **combine with OR logic** — if any SG allows the traffic, it's allowed.

### Modular SG Pattern

Instead of one giant SG with 20 rules, create focused SGs:

```
Web Server EC2
├── web-sg          (Allow HTTP/HTTPS from internet)
├── ssh-sg          (Allow SSH from office IP)
└── monitoring-sg   (Allow Prometheus from monitoring server)
```

Benefits:
- ✅ **Reusable** across instances (same `monitoring-sg` for all servers)
- ✅ **Auditable** (focused rules, easier to review)
- ✅ **Modular** (change one without affecting others)
- ✅ **Team-friendly** (different teams own different SGs)

---

## 🎯 Source Specification Hierarchy

From **best** to **worst** for production:

| Rank | Source Type | Example | Use When |
|------|-------------|---------|----------|
| 🥇 | **SG Reference** | `app-sg` | Multi-tier architectures (always preferred) |
| 🥈 | **Tight CIDR** | `10.0.5.0/24` | Fixed network segments |
| 🥉 | **Single IP /32** | `203.0.113.45/32` | Specific external machines (e.g., your office) |
| ❌ | **Open Internet** | `0.0.0.0/0` | ONLY for public services (web/HTTPS) |

**Architect rule:** *"Identity > Location. SG references > IP addresses (whenever possible)."*

---

## 🚨 Top 3 Real-World Disasters

### 🔥 Disaster #1: SSH Open to the Internet

```
Type: SSH | Port: 22 | Source: 0.0.0.0/0  ❌
```

**Result:** Bots scan for this constantly. Brute-force attempts begin within hours.

**Fix:** Restrict to specific office IPs OR use **AWS Systems Manager Session Manager** (no SSH port needed at all!).

### 🔥 Disaster #2: Database Open to the Internet

```
Type: MySQL | Port: 3306 | Source: 0.0.0.0/0  ❌
```

**Result:** Hackers find your database via Shodan within minutes. Brute force, data dumps, ransom demands.

**Fix:** Database SG should ONLY allow access from app-tier SG (using SG reference). **Never internet.**

### 🔥 Disaster #3: Wide-Open Test Rule Forgotten

A developer adds *"Allow all from my IP"* to debug, then leaves the company. The IP gets reassigned to someone else. The new owner now has access to production.

**Fix:** Audit Security Groups quarterly. Use **AWS Config rules** to auto-detect overly permissive SGs.

---

## 🛡️ Security Group vs NACL — Quick Comparison

| Feature | Security Group | NACL |
|---------|---------------|------|
| **Layer** | Instance level | Subnet level |
| **Stateful?** | ✅ Stateful | ❌ Stateless |
| **Allow / Deny** | Allow only | Allow AND Deny |
| **Rule Processing** | All rules evaluated | Numbered order (lowest first) |
| **Default Inbound** | Deny all | Allow all |
| **Default Outbound** | Allow all | Allow all |
| **Applied To** | EC2 instances | Subnets (all resources in subnet) |

### Memory Hook

- **SG = "Stateful Gatekeeper"** at the instance door
- **NACL = "Numbered ACL"** at the subnet boundary

Both can be used together for **defense in depth**.

---

## 🎯 The Modern Architect's SSH Strategy

Traditional approach:
- Open port 22 from a bastion host or office IP
- Manage SSH keys carefully
- Audit constantly

**Modern AWS approach (recommended):**
- Use **AWS Systems Manager Session Manager**
- ❌ No SSH port needed (port 22 stays closed)
- ❌ No keys to manage
- ✅ Access controlled by IAM policies
- ✅ Full session logging via CloudTrail

This is the **Cloud Security architect-level answer** to "How do developers access production EC2?" 💎

---

## 🎯 Common Exam Patterns

| Scenario | Answer |
|----------|--------|
| "Why does the response work without outbound rule?" | Security Groups are **stateful** |
| "How to block specific malicious IPs?" | NACL (with deny rule) |
| "Database should only be accessible by app servers" | SG reference (`app-sg` as source) |
| "Auto-scaling app tier needs to talk to database" | SG references (handles dynamic IPs) |
| "Multi-tier secure architecture" | Web→App→DB with chained SG references |
| "Modern SSH access without exposing port 22" | SSM Session Manager |
| "Default behavior of a new SG" | Inbound deny, outbound allow |

---

## 🛡️ Security-First Takeaways

1. **Default deny, explicit allow** — never use `0.0.0.0/0` for admin/database ports
2. **Use SG references over IPs** — identity-based access scales better
3. **Multi-tier with SG chaining** — defense in depth, contain breaches
4. **Modular SG design** — focused SGs > monolithic ones
5. **Lock down outbound** for sensitive workloads — prevents data exfiltration
6. **Audit quarterly** — old test rules are common breach vectors
7. **Modern SSH = Session Manager** — eliminate port 22 entirely

---

*Part of my AWS Solutions Architect Associate journey toward Cloud Security specialization.*
