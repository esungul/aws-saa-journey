# 🛠️ VPC Hands-On Lab — Complete Build Journey (Day 2)

> Building a production-grade VPC architecture in AWS console from scratch. This document captures the full journey across two iterations — including real debugging, successful completion, and end-to-end verification via two-hop SSH from laptop through bastion to private app server.

---

## 🎯 Project Goal

Build a production-grade VPC architecture in AWS console demonstrating:
- VPC fundamentals (CIDR, subnets, IGW, route tables)
- NAT Gateway pattern (outbound-only internet for private resources)
- Bastion Host security pattern (single hardened entry point)
- Multi-tier Security Groups with SG references
- Defense-in-depth networking
- Cost-conscious cloud engineering (build, verify, tear down)

This was Day 2 of a strategic 3-day VPC plan: theory (Day 1) → hands-on build (Day 2) → advanced topics (Day 3).

**Result: Successfully built, verified end-to-end via two-hop SSH, and tore down with $0 ongoing cost.** ✅

---

## 🏗️ Final Architecture

```
                       Internet
                          ↓
                  Internet Gateway
                          ↓
       ┌────────[VPC: 10.0.0.0/16]──────────────────┐
       │                                              │
       │  AZ: us-east-1a                              │
       │  ┌── Public Subnet (10.0.1.0/24) ─────────┐ │
       │  │   ├── Bastion Host (with EIP)          │ │
       │  │   │   - Public IP: 3.228.2.240          │ │
       │  │   │   - Private IP: 10.0.1.75           │ │
       │  │   │   - SG: bastion-sg                  │ │
       │  │   │     (SSH from my laptop only)       │ │
       │  │   └── NAT Gateway (with EIP)            │ │
       │  │       - EIP: 23.20.12.109               │ │
       │  └────────────────────────────────────────┘ │
       │                          ↑                   │
       │  ┌── Private Subnet (10.0.10.0/24) ──────┐  │
       │  │   └── App Server EC2                  │  │
       │  │       - Private IP: 10.0.10.209       │  │
       │  │       - Public IP: NONE ✅            │  │
       │  │       - SG: app-sg                    │  │
       │  │         (SSH from bastion-sg only)    │  │
       │  └───────────────────────────────────────┘  │
       │                                              │
       └──────────────────────────────────────────────┘
```

**Region:** us-east-1 (N. Virginia)
**Approach:** Single-AZ for first hands-on lab (proven concept; multi-AZ in next iteration)

---

## ✅ Build Summary (5 Phases Complete)

### Phase 1: Foundation ✅
- VPC `vpc-lab` (10.0.0.0/16) created
- 2 subnets: `public-subnet-1a` (10.0.1.0/24), `private-subnet-1a` (10.0.10.0/24)
- Internet Gateway `igw-lab` created and attached
- All 251 IPs available per subnet (AWS reserved 5, exactly as theory predicted)

### Phase 2: Public Routing ✅
- Custom route table `public-rt` created (separate from main RT)
- Route added: `0.0.0.0/0 → IGW`
- Associated with `public-subnet-1a`
- Public subnet now genuinely public (has IGW route)

### Phase 3: NAT Gateway + Private Routing ✅
- Elastic IP allocated (`eip-nat-lab`)
- NAT Gateway `nat-gw-lab` created in **public subnet** (correct placement!)
- Custom route table `private-rt` created
- Route added: `0.0.0.0/0 → NAT Gateway`
- Associated with `private-subnet-1a`
- Private subnet has outbound-only internet via NAT

### Phase 4: EC2 + SSH ✅ (THE CLIMAX)
- **Security Groups:**
  - `bastion-sg`: SSH (port 22) from my laptop's IP/32 only
  - `app-sg`: SSH from `bastion-sg` (Security Group reference, not IP-based!)
- **EC2 Instances:**
  - `bastion-host`: t3.micro in public subnet, EIP attached (3.228.2.240)
  - `app-server`: t3.micro in private subnet, NO public IP (10.0.10.209)
- **IMDSv2 enforced** on both instances (Capital One protection)
- **Two-hop SSH verified:** Laptop → Bastion → App Server ✅

### Phase 5: Documentation + Teardown ✅
- All resources deleted in correct dependency order
- $0 ongoing cost verified
- All service pages confirm clean state

---

## 🎯 Two-Hop SSH Verification (The Proof!)

This is the moment the architecture came alive — proving every component works together.

### The Command Chain:

```bash
# Step 1: Add key to SSH agent (on Mac)
ssh-add EC2login.pem
# Output: Identity added: EC2login.pem

# Step 2: SSH to bastion WITH agent forwarding (-A flag is the magic)
ssh -A -i EC2login.pem ec2-user@3.228.2.240
# Now inside bastion at [ec2-user@ip-10-0-1-75 ~]$

# Step 3: From bastion, SSH to app server (NO -i flag needed!)
ssh ec2-user@10.0.10.209
# Now inside app server at [ec2-user@ip-10-0-10-209 ~]$
```

### Tests Run on App Server (Architecture Proven):

```bash
# Test 1: Confirm hostname (proves private subnet)
hostname
# Output: ip-10-0-10-209.ec2.internal ✅

# Test 2: Verify NO public IP (proves private subnet)
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/public-ipv4
# Output: 404 Not Found ✅ (No public IP exists - exactly as designed!)

# Test 3: Internet via NAT Gateway
ping -c 4 google.com
# Output: 3 packets transmitted, 3 received, 0% packet loss ✅

# Test 4: Verify NAT translation (the magic!)
curl ifconfig.me
# Output: 23.20.12.109 ✅
# This is the NAT Gateway's EIP - NOT the app server's private IP!
# Proves: NAT translating outbound traffic, hiding private IP from internet
```

---

## 🛡️ What This Architecture Proves

The successful two-hop SSH + tests demonstrated **8 architectural concepts working together:**

| # | Concept | How It Was Proven |
|---|---------|-------------------|
| 1 | VPC isolation | All resources in vpc-lab, separate from default VPC |
| 2 | Public/private subnet separation | Different route tables, different behaviors |
| 3 | Internet Gateway routing | Mac → Bastion path worked |
| 4 | NAT Gateway translation | App server reached internet without public IP |
| 5 | Security Groups filter correctly | Only authorized traffic reached EC2s |
| 6 | SG references work | app-sg trusts bastion-sg without IP changes |
| 7 | Defense-in-depth | App server unreachable from internet directly |
| 8 | IMDSv2 enforcement | Capital One-style attacks prevented |

🎯 **One 30-second SSH test = 8 concepts proven in production-grade infrastructure.**

---

## 🐛 Real Debugging Stories (The Valuable Part)

### Bug #1: Misconfigured Duplicate Route Table

**What happened:**
While creating the private route table, accidentally created a duplicate (`rtb-0500cdb800e1c13eb`) with `0.0.0.0/0 → Internet Gateway` (instead of NAT Gateway).

**How caught:**
Took screenshot of route table list before associating subnets. Noticed unusual configuration.

**Why it matters:**
If associated with private-subnet-1a, this would have given the private subnet direct internet access (no NAT proxy). Database servers would have been directly internet-reachable — exactly the misconfiguration that causes real-world data breaches.

**The fix:**
Created fresh route table with correct configuration. Better to start clean than debug a misconfigured resource.

**Lesson:**
Always **verify configurations before associating subnets**. Routes determine subnet behavior — one wrong target type = security boundary breached.

---

### Bug #2: Internet Gateway Detach Error During Teardown

**What happened:**
```
Error: "The Internet Gateway igw-0518075abb9166666 cannot be detached 
       as there are some mapped public address(es). Please delete the 
       following public NAT Gateways, then try again."
```

**Root cause:**
NAT Gateway depends on IGW. AWS prevents detaching IGW while NAT exists (would break NAT).

**Why this is a feature:**
This is a **safety mechanism**. AWS protects you from breaking your NAT Gateway by removing its dependency. In production, this prevents accidental outages.

**The fix:**
Delete NAT Gateway FIRST → wait for "Deleted" state → then detach IGW.

**Lesson:**
**Teardown order matters.** Resources have dependencies. Delete in reverse-creation order:
1. NAT Gateway (uses EIP and IGW)
2. EIP (was used by NAT)
3. Route Tables (referenced gateways)
4. IGW detach + delete
5. Subnets
6. VPC

---

### Bug #3: Bastion Missing Public IP

**What happened:**
After launching bastion EC2, found it had no public IP. SSH from laptop would fail.

**Root cause:**
"Auto-assign public IP" wasn't explicitly enabled during launch (subnet default = disabled).

**The fix:**
Allocated separate Elastic IP and associated it with bastion-host.

**Why this is actually better:**
- ✅ Stable IP (won't change on stop/start)
- ✅ Production-grade pattern
- ✅ Better for SG whitelisting (predictable IP)

**Lesson:**
When launching EC2 in public subnet:
- Enable "Auto-assign public IP" explicitly OR
- Allocate Elastic IP after launch (production approach)

---

### Bug #4: SSH Agent Forwarding Setup

**What happened:**
First attempt to SSH from bastion to app server tried to use the .pem file directly, which doesn't exist on bastion (intentionally — security risk).

**Root cause:**
Tried to SSH from bastion as if .pem file was there. Got error: "EC2login.pem not accessible".

**The fix:**
Used SSH agent forwarding pattern:
1. `ssh-add EC2login.pem` (add to local agent on Mac)
2. `ssh -A -i EC2login.pem ec2-user@bastion` (forward agent with -A)
3. `ssh ec2-user@app-server` (no -i needed - agent forwards key)

**Lesson:**
SSH agent forwarding is the **production standard** for bastion patterns:
- ✅ Private key NEVER leaves your local machine
- ✅ Bastion just relays SSH requests
- ✅ Even if bastion compromised, attacker can't steal key
- ❌ Don't copy .pem to bastion (would be security risk)

---

## 💰 Cost Analysis

**Resources that incurred costs:**

| Resource | Duration | Estimated Cost |
|----------|----------|----------------|
| NAT Gateway | ~6-8 hours total (across 2 days) | ~$0.30 - $0.40 |
| Elastic IP (NAT) | While attached | $0 (free) |
| Elastic IP (Bastion) | While attached | $0 (free) |
| EC2 t3.micro instances | ~3 hours | $0 (free tier) |
| Data Transfer | Minimal | < $0.10 |
| **Total Estimated** | | **~$0.50 - $1.00** |

**Resources that were FREE (within Free Tier):**
- VPC creation: $0
- Subnets: $0
- Internet Gateway: $0
- Route Tables: $0
- Security Groups: $0
- Route configurations: $0

**Real cost lesson:**
NAT Gateway is the dominant cost driver in VPC labs. ~$0.045/hour × time running. Always tear down NAT Gateway after lab work.

**Verification of $0 ongoing cost:**
- ✅ No NAT Gateways in "Available" state
- ✅ No Elastic IPs allocated
- ✅ No EC2 instances running
- ✅ No vpc-lab in VPC list
- ✅ No EBS volumes from lab

---

## 🎯 Architect Decisions Made

### Decision 1: Single-AZ vs Multi-AZ
**Chose:** Single-AZ for first hands-on lab
**Why:** Don't try to learn everything at once. Get the architecture working end-to-end first.

### Decision 2: Custom VPC vs Default VPC
**Chose:** Custom VPC with `10.0.0.0/16`
**Why:** Production approach. Default VPC lacks security customization.

### Decision 3: NAT Gateway vs NAT Instance
**Chose:** NAT Gateway
**Why:** AWS-managed, no maintenance, highly available, auto-scaling. NAT Instance is legacy.

### Decision 4: EIP for Bastion vs Auto-assigned Public IP
**Chose:** Elastic IP
**Why:** Stable IP for SG whitelisting, won't change on stop/start, production-grade pattern.

### Decision 5: Security Group References vs IP-Based Rules
**Chose:** SG references (`app-sg` allows from `bastion-sg`)
**Why:** Self-maintaining as instances change, scalable for multiple bastions, production standard.

### Decision 6: SSH Agent Forwarding vs Copying .pem
**Chose:** Agent forwarding with `-A` flag
**Why:** Private key never leaves local machine. If bastion compromised, key is safe.

### Decision 7: IMDSv2 Required
**Chose:** Required (not optional)
**Why:** Prevents Capital One-style SSRF attacks. Security baseline for all EC2s.

---

## 🛡️ Security-First Insights from This Build

### Insight 1: Routes Are Security Boundaries
Route tables aren't just navigation — they're security boundaries. One wrong target (IGW vs NAT) = exposure level changes drastically.

### Insight 2: AWS Dependency Checks Save You
The "cannot detach IGW" error felt frustrating but is actually AWS protecting against misconfigurations that would break NAT functionality.

### Insight 3: Verification Habit
Taking screenshots after each step caught the misconfigured route table BEFORE it caused damage. **Verify, verify, verify.**

### Insight 4: Defense in Depth Is Architectural, Not Optional
The two-hop SSH pattern + private subnet isolation + SG references all work TOGETHER. Removing any one layer weakens the whole.

### Insight 5: NAT Translation Is Production Magic
App server with **no public IP** can still reach the internet (proven via curl ifconfig.me showing NAT's EIP). One-way internet access = production gold standard.

### Insight 6: Cost Discipline Is Non-Negotiable
NAT Gateway's hourly billing forces conscious engineering. Every running resource = active cost. Tear down or pay.

---

## 📊 Iteration Story (Why Two Attempts Mattered)

### Attempt 1 (Day 2 - First Try):
- ✅ Built 75% of architecture
- 🐛 Caught misconfigured route table (good debugging!)
- 🐛 Hit IGW detach dependency error (learned teardown order)
- ⏸️ Stopped before completing SSH test
- 🛡️ Made smart decision to tear down rather than rush

### Attempt 2 (Day 2 Retry - This Document):
- ✅ Applied all lessons from first attempt
- ✅ Cleaner execution, fewer mistakes
- 🐛 Hit new issue: bastion missing public IP (different bug!)
- ✅ Solved with EIP allocation pattern
- 🐛 Hit SSH agent forwarding learning curve
- ✅ Mastered production SSH bastion pattern
- ✅ Successfully completed two-hop SSH
- ✅ Verified entire architecture
- ✅ Tore down with $0 cost

**Why both iterations matter:**
The two-iteration journey is MORE valuable than a one-shot success. Real engineering involves:
- Persistence through obstacles
- Different bugs in different attempts
- Cumulative learning across iterations
- Pattern recognition over time

🎯 **Hiring managers value engineers who persist through failure more than those who get lucky on the first try.**

---

## 📚 Key Concepts Reinforced Through Building

| Theory | Hands-On Experience |
|--------|---------------------|
| "VPC is a private network in AWS" | Created `vpc-lab`, saw it isolated from default |
| "Subnets are AZ-locked" | Saw `us-east-1a` lock when creating subnets |
| "AWS reserves 5 IPs per subnet" | `/24` showed 251 available (256 - 5) |
| "Route tables determine public/private" | Created public-rt with IGW, private-rt with NAT |
| "NAT Gateway must be in public subnet" | Selected public-subnet-1a during NAT creation |
| "EIP needed for NAT Gateway" | Allocated EIP first, then created NAT |
| "Resources have dependencies" | IGW couldn't detach while NAT existed |
| "Tear down in reverse order" | Followed dependency order successfully |
| "Public/Private = route table config" | Same EC2 type, different subnets, different reachability |
| "SG references > IP rules for tiers" | app-sg trusting bastion-sg worked perfectly |
| "Bastion = single hardened entry" | One bastion accessing everything else |
| "Two-hop SSH is THE pattern" | Successfully demonstrated end-to-end |
| "NAT translates outbound IPs" | Proven: app server's outbound = NAT's EIP |

🎯 **Theory + practice = architect-level understanding.** Each concept now has a real memory anchor.

---

## 🌟 What This Project Demonstrates

### Technical Skills:
- ✅ AWS Console proficiency (navigation, configuration, debugging)
- ✅ VPC architecture design
- ✅ Network security principles (public/private separation)
- ✅ NAT Gateway pattern implementation
- ✅ Route table configuration and management
- ✅ Bastion host hardened deployment
- ✅ Security Group references for tier-based access
- ✅ SSH key management and agent forwarding

### Engineering Skills:
- ✅ Real-world debugging (4 distinct issues caught and resolved)
- ✅ Verification habits (screenshots, audits, testing)
- ✅ Cost-conscious thinking (teardown discipline)
- ✅ Decision-making under pressure (when to fix vs recreate)
- ✅ Persistence through obstacles (two iterations)
- ✅ Documentation as you go

### Architectural Thinking:
- ✅ Defense-in-depth design
- ✅ Production-grade patterns
- ✅ Awareness of trade-offs
- ✅ Future-proof naming conventions
- ✅ Security-first mindset
- ✅ Cost as a design constraint

---

## 🎯 Production-Grade Patterns Implemented

### 1. Multi-Tier VPC Architecture
```
Public Subnet  → Internet-facing tier (load balancers, bastion, NAT)
Private Subnet → Application/Database tier (no internet exposure)
```

### 2. Bastion Host Pattern
```
Single hardened entry point for all SSH access
- Reduces attack surface (1 EC2 exposed vs N)
- Centralized audit trail
- IP whitelisting manageable
- Easy to revoke access (kill bastion)
```

### 3. SG Reference Pattern
```
app-sg: "Allow SSH from bastion-sg"
Not: "Allow SSH from 10.0.1.50" (IP-based)

Benefits:
- Self-maintaining as bastion IPs change
- Scalable for multiple bastions
- Production standard
```

### 4. NAT Gateway Pattern
```
Private resources → NAT Gateway → Internet
- Outbound only (no inbound)
- Hides private IPs from internet
- Allows updates, CloudWatch, package downloads
```

### 5. Defense in Depth
```
Layer 1: VPC isolation
Layer 2: Subnet separation (public/private)
Layer 3: Route tables (different routes)
Layer 4: NAT Gateway (one-way internet)
Layer 5: Bastion Host (single SSH entry)
Layer 6: Security Groups (instance firewall)
Layer 7: SG references (tier-based access)
Layer 8: IMDSv2 (Capital One protection)

A breach must defeat ALL 8 layers.
```

---

## 🛡️ Architect Wisdom Earned

> *"Build, verify, document, tear down, verify zero cost, reflect."*
>
> *"Routes route. Security Groups filter. Both required, neither sufficient alone."*
>
> *"Teardown order is the reverse of creation order — AWS enforces it."*
>
> *"AWS dependency errors are features protecting your infrastructure."*
>
> *"Cost discipline is more important than feature completion."*
>
> *"Private keys never leave your local machine — agent forwarding for everything else."*
>
> *"Public IP missing = subnet default not enabled = use Elastic IP."*
>
> *"Two-hop SSH = defense in depth in action."*
>
> *"Persistence through iteration > getting lucky on first try."*
>
> *"Every running resource is a question: am I still using this?"*

---

## 🎯 Architect Decision Framework Validated

When designing a VPC for production, this build validated:

```
What's the use case?
├── Production → Custom VPC, /16 CIDR
├── Multi-AZ for HA (next iteration)
└── Private subnets for sensitive resources

How many subnets?
├── Public: Load balancers, NAT, bastion
├── Private app: Application servers
└── Private DB: Databases (most isolated)

How to give private internet?
├── Outbound only → NAT Gateway ✅ (proven works)
├── No internet → No NAT (most secure)
└── Per-AZ NAT for HA

How to allow admin access?
├── Modern → Systems Manager Session Manager
├── Traditional → Bastion Host with SG references ✅ (proven works)
└── Avoid → Public IPs on private resources

Cost optimization?
├── Tear down when done ✅ (practiced today)
├── One NAT per AZ (don't over-provision)
├── Tag everything ✅ (Project=vpc-lab used)
└── Monitor billing weekly
```

🎯 **Each ✅ is something proven through this build.**

---

## 📊 Concept Connections to Other AWS Topics

This VPC build foundation connects to many other AWS topics:

- **IAM** → Could attach IAM Role to EC2s for credential-less AWS API access
- **Security Groups** → Used SG references for tier-based access ✅
- **EC2 Networking** → Public/Private IPs, ENIs, IMDSv2 (all applied!) ✅
- **EBS** → 8 GiB gp3 volumes attached to EC2s
- **AMI** → Used Amazon Linux 2023 (golden image pattern)
- **CloudWatch** → Could ship metrics from app server via NAT
- **VPC Flow Logs** → Day 3 topic, monitoring this exact traffic
- **VPC Endpoints** → Day 3 topic, bypass NAT for AWS services
- **Route 53** → Could point DNS to bastion's EIP
- **Load Balancer** → Could distribute to multiple bastions
- **Auto Scaling** → Could replace single EC2s with ASGs

🌟 **VPC is the foundation everything else builds on.** This single build touched 5+ AWS services.

---

## 🚀 What's Next

### Immediate:
- ✅ Update GitHub portfolio (this document!)
- ⏭️ Set up Budget Alert ($10) for future labs
- ⏭️ Enable Cost Anomaly Detection

### Day 3 (Advanced VPC Topics):
- VPC Peering (connect VPCs)
- VPC Endpoints (private AWS service access without NAT)
- VPC Flow Logs (security monitoring) 🛡️⭐
- Site-to-Site VPN (overview)
- Direct Connect (overview)
- Transit Gateway (multi-VPC at scale)
- Networking costs deep-dive

### Future Iterations:
- Multi-AZ deployment of this same architecture
- Auto Scaling Groups instead of single EC2s
- Application Load Balancer in front of app tier
- RDS in private subnet
- VPC Flow Logs analysis

---

## 💡 Personal Reflection

The most valuable lesson from this build wasn't any single technical concept. It was learning that **debugging is a CORE architect skill, not a failure mode.**

When things didn't work:
- I didn't panic
- I read error messages carefully
- I verified state before acting
- I made cost-conscious decisions
- I documented for future reference
- I returned to finish what I started

**That's how senior cloud engineers operate every day.**

The path from "watching tutorials" to "building real infrastructure" runs through experiences exactly like this one — messy, imperfect, but genuinely educational.

🎯 **Persistence + verification + iteration = architect-level mastery.**

The two-iteration journey (75% on first attempt, 100% with full SSH test on second) demonstrates the real path to skill: **try, fail, learn, retry, succeed, document.**

---

## 🎉 What I Can Now Confidently Say in Interviews

> *"I've designed and built a multi-tier VPC architecture in AWS, including bastion host pattern, NAT Gateway translation, and Security Group references for tier-based access. I implemented defense-in-depth with 8 distinct security layers, verified end-to-end via two-hop SSH from my laptop through the bastion to a private app server, and proved the NAT Gateway's outbound translation by checking the apparent IP from the private subnet. I debugged real issues including misconfigured route tables, missing public IP assignments, and SSH agent forwarding setup. The entire architecture cost approximately $1 to build and test, and I tore down all resources to verify $0 ongoing cost — demonstrating both technical skill and operational discipline."*

🎯 **That's not memorization.** **That's experience.** 💎

---

*Part of my AWS Solutions Architect Associate journey toward Cloud Security specialization. This document represents real learning experience — including failures, debugging, and ultimate success across two iterations. Real engineering happens in the messy middle, then becomes clean through persistence.*

**Build complete: ✅
End-to-end verified: ✅
Tear down complete: ✅
$0 ongoing cost: ✅
Portfolio piece earned: ✅**
