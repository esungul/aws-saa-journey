# ⚖️ Elastic Load Balancing Deep Dive — Day 1

> Day 1 of the High Availability and Scalability journey. Covers the foundational concepts of HA/Scalability/Elasticity, load balancing fundamentals, and all 3 modern AWS load balancers (ALB, NLB, GWLB) plus health checks. Sets up Day 2 (Auto Scaling Groups) for complete HA mastery.

---

## 🎯 What This Document Covers

This is the comprehensive reference for Elastic Load Balancing on AWS, covering 6 critical layers:

1. **HA vs Scalability vs Elasticity** — foundational concepts
2. **Load Balancing Fundamentals** — why we need load balancers
3. **Application Load Balancer (ALB)** — Layer 7 smart routing
4. **Network Load Balancer (NLB)** — Layer 4 extreme performance
5. **Gateway Load Balancer (GWLB)** — Layer 3/4 security inspection
6. **ELB Health Checks** — the foundation of HA

These topics are critical for:
- ✅ **AWS SAA exam** (15-20% of exam questions)
- ✅ **Cloud Security Engineer career** (daily-use tools)
- ✅ **Production architecture** (every real app uses these)

---

## 🚪 Layer 1: HA vs Scalability vs Elasticity

These three terms are foundational architect concepts and frequently confused. Each solves a different problem.

### High Availability (HA)

**Definition:** *"The system stays UP even when components fail."*

```
Goal: Minimize downtime
Strategy: Redundancy across failure domains

Example:
- App running in us-east-1a AND us-east-1b
- If 1a fails (AZ outage) → 1b still serves users
- Users barely notice
- Uptime: 99.99%+ ("4 nines")
```

**Real-world analogy:** Two cashiers at a store, both open. One can take a break, other still serves customers. No service interruption.

### Scalability

**Definition:** *"The system can handle MORE load."*

Two types of scaling:

#### Vertical Scaling (Scale UP) ⬆️

Make existing machine BIGGER

```
Example:
- t3.micro (1 vCPU, 1 GB RAM)
- Upgrade to: t3.large (2 vCPU, 8 GB RAM)
- Same server, more power

Pros:
✅ Simple (no architecture change)
✅ Good for databases (hard to split)

Cons:
🚨 Has a ceiling (max instance size)
🚨 Requires downtime to upgrade
🚨 Single point of failure
🚨 Expensive at top end
```

#### Horizontal Scaling (Scale OUT) ➡️

Add MORE machines

```
Example:
- 1 EC2 → 2 EC2s → 5 EC2s → 100 EC2s
- Same size, more of them
- Need load balancer to distribute traffic

Pros:
✅ Nearly unlimited capacity
✅ No downtime to scale
✅ Better fault tolerance
✅ Cost-effective

Cons:
⚠️ More complex architecture
⚠️ Need stateless apps
⚠️ Database scaling still tricky
```

### Elasticity

**Definition:** *"The system auto-adjusts capacity to match demand."*

```
Goal: Right-sized for current load (no waste, no shortage)
Strategy: Automatic scaling based on metrics

Example:
- Morning: 10 EC2s (low traffic)
- Lunch: 50 EC2s (peak load)
- Evening: 20 EC2s (medium load)
- Night: 5 EC2s (minimal load)

All automatic! No human intervention.
```

**Real-world analogy:** Cashiers automatically appear when lines get long, and clock out when store is empty. Only possible because cloud provides on-demand resources.

### The Critical Distinction

```
HA = "Stays up when broken"
   ↓ requires
Redundancy (multiple components)

Scalability = "Handles more load"
   ↓ requires
Ability to add capacity (vertical OR horizontal)

Elasticity = "Auto-adjusts capacity"
   ↓ requires
Cloud + Auto Scaling + monitoring
```

### Production Gold Standard

```
HA + Elasticity together:
- ASG with min 2, max 20 EC2s
- Spread across 3 AZs
- Auto-scales based on demand
- Survives failures
- Cost-efficient
```

### Memory Hook

```
HA = Highly Available → "Up when broken"
Scalability = Can grow → Vertical (UP) or Horizontal (OUT)
Elasticity = Auto-grows AND shrinks → "Rubber band"
```

---

## 🚪 Layer 2: Load Balancing Fundamentals

### What Is a Load Balancer?

**Load Balancer = a server that distributes incoming traffic across multiple backend servers.**

### The 5 Problems Load Balancers Solve

1. **Distribute traffic evenly** — prevent server overload
2. **Increase fault tolerance** — survive server failures (HA)
3. **Health monitoring** — detect dead servers automatically
4. **SSL termination** — offload encryption from backend servers
5. **Single entry point** — one DNS name for users

### The AWS Load Balancer Family

```
┌──────────────────────────────────────────────────┐
│ AWS Elastic Load Balancing (ELB) Family          │
├──────────────────────────────────────────────────┤
│                                                   │
│ 1. Classic Load Balancer (CLB) - LEGACY ⚠️       │
│    - Old generation (avoid in new architectures) │
│                                                   │
│ 2. Application Load Balancer (ALB) - Layer 7 ⭐  │
│    - Modern, HTTP/HTTPS focus                    │
│    - Smart routing                               │
│                                                   │
│ 3. Network Load Balancer (NLB) - Layer 4 ⭐     │
│    - Extreme performance                         │
│    - TCP/UDP                                     │
│                                                   │
│ 4. Gateway Load Balancer (GWLB) - Layer 3/4 ⭐   │
│    - Network appliances (firewalls, IDS/IPS)    │
└──────────────────────────────────────────────────┘
```

### OSI Layer Concept (Critical for Exam!)

```
OSI Model (simplified):

Layer 7: Application (HTTP, HTTPS)
   ↑ ALB operates here

Layer 4: Transport (TCP, UDP)
   ↑ NLB operates here

Layer 3: Network (IP routing)
   ↑ GWLB operates here
```

| LB Type | Layer | Capability |
|---------|-------|------------|
| **ALB** | 7 | "Smart" — reads HTTP content |
| **NLB** | 4 | "Fast" — just routes packets |
| **GWLB** | 3 | "Deep" — packet inspection |

### Where Load Balancers Live (Connection to VPC)

```
Production Architecture:

Internet
   ↓
Internet Gateway (IGW)
   ↓
[VPC: 10.0.0.0/16]
│
├── Public Subnets (1a, 1b, 1c) ⭐
│   └── Load Balancer (ALB/NLB)
│       (Multi-AZ for HA!)
│
└── Private Subnets (1a, 1b, 1c)
    └── EC2 Servers (targets)
```

**Pattern:** Internet-facing LB → Public subnets, Backend EC2s → Private subnets = production security.

### Why AWS-Managed Over Self-Hosted

- ✅ Fully managed (AWS handles patches, OS, etc.)
- ✅ Auto-scales transparently
- ✅ Built-in high availability
- ✅ Integrated with CloudWatch
- ✅ Integration with ACM (free SSL certs)
- ✅ Best practice for AWS workloads

### Load Balancer DNS Name

```
ALB example DNS:
my-app-lb-1234567890.us-east-1.elb.amazonaws.com

- LBs get DNS name, not static IP
- AWS may rotate IPs internally
- DNS name stays constant
- Use Route 53 to alias custom domain

Exception: NLB can have static IP (Elastic IP)
```

---

## 🚪 Layer 3: Application Load Balancer (ALB)

### What ALB Is

**ALB = Layer 7 load balancer that understands HTTP/HTTPS and routes traffic based on application-level information.**

### Key Properties

- ✅ Operates at Layer 7 (Application)
- ✅ Supports HTTP, HTTPS, WebSockets
- ✅ Smart routing (path, host, header, query, method)
- ✅ Multi-target support (EC2, IPs, Lambda, ALB)
- ✅ Built-in DDoS protection (AWS Shield Standard)
- ✅ Multi-AZ for HA
- ✅ Auto-scales internally
- ✅ Integration with WAF

### ALB Architecture

```
Internet
   ↓
ALB (in public subnets, multi-AZ)
   ↓
Listeners (port 80, 443)
   ↓
Rules (path/host/header matching)
   ↓
Target Groups
   ↓
Targets (EC2, IP, Lambda)
```

**Flow:** Listeners → Rules → Target Groups → Targets

### 5 Routing Methods

#### 1. Path-Based Routing ⭐
```
example.com/products → ProductTargetGroup
example.com/cart → CartTargetGroup
example.com/checkout → CheckoutTargetGroup
```

#### 2. Host-Based Routing ⭐
```
www.example.com → WebTargetGroup
api.example.com → ApiTargetGroup
admin.example.com → AdminTargetGroup
```

#### 3. HTTP Method Routing
```
GET → ReadOnlyTargetGroup
POST → WriteTargetGroup
DELETE → AdminTargetGroup
```

#### 4. Header-Based Routing
```
User-Agent: Mobile* → MobileTargetGroup
Custom-Tenant: TenantA → TenantATargetGroup
```

#### 5. Query String Routing
```
?env=prod → ProdTargetGroup
?version=v2 → V2TargetGroup
```

### Real-World Routing Example

```
ALB Configuration for SaaS:

Listener on port 443 (HTTPS):
├── Rule 1: path = /api/v1/* → ApiV1TargetGroup
├── Rule 2: path = /api/v2/* → ApiV2TargetGroup
├── Rule 3: host = admin.example.com → AdminTargetGroup
├── Rule 4: path = /static/* → CDN
├── Rule 5: method = POST AND path = /upload → UploadTargetGroup
└── Default: WebAppTargetGroup
```

### Multi-Tenant SaaS Pattern (Production Gold!)

```
One ALB serves multiple tenants:

ALB
├── customera.example.com → Tenant A's EC2s
├── customerb.example.com → Tenant B's EC2s
├── customerc.example.com → Tenant C's EC2s
└── ... up to 25 SSL certs per ALB!

Powered by:
✅ Host-based routing
✅ SNI (Server Name Indication)
✅ Multiple SSL certificates
```

### Target Group Properties

- Target type (EC2/IP/Lambda/ALB)
- Protocol (HTTP/HTTPS/HTTP2)
- Port (e.g., 80, 8080, 443)
- Health check configuration (independent per group!)
- Stickiness settings
- Deregistration delay

### ALB Limits

```
✅ 25 SSL certificates per ALB (with SNI)
✅ 100 listeners per ALB
✅ 100 rules per ALB
✅ 100 target groups per ALB
✅ 1000 targets per target group
✅ 5 actions per rule
```

### Security Features

#### 1. SSL/TLS Termination
- ALB decrypts SSL (offloads from backend)
- Centralized certificate management
- Use ACM (AWS Certificate Manager) for FREE certs

#### 2. AWS WAF Integration
- Block SQL injection, XSS, malicious IPs
- Rate limiting
- Geographic restrictions

#### 3. Security Groups
```
ALB SG:
- Inbound: Port 80/443 from 0.0.0.0/0
- Outbound: Port 80 to EC2 SG

EC2 SG:
- Inbound: Port 80 FROM ALB SG only!
- NOT 0.0.0.0/0 (security best practice)
```

#### 4. HTTP/HTTPS Redirects
- ALB automatically redirects HTTP → HTTPS
- Force encryption without code changes

#### 5. DDoS Protection
- Built-in: AWS Shield Standard (FREE)
- Optional: AWS Shield Advanced ($$$)

### Sticky Sessions

```
Two types:
✅ Duration-based: ALB-generated cookie, configurable expiry
✅ Application-based: Your app's cookie

Best practice:
✅ Use shared session storage (Redis/ElastiCache)
✅ Avoid stickiness for truly stateless apps
```

### ALB Cost

- $0.0225/hour ($16/month base)
- $0.008 per LCU (Load Balancer Capacity Unit)
- Typical: $20-30/month

### Memory Hook

```
ALB = "Smart HTTP router for web apps" 🧠🌐

Layer: 7 (Application)
Routes by: Path, Host, Header, Query, Method
Targets: EC2/IP/Lambda/ALB
Use when: Web applications, microservices, SaaS
Don't use when: Non-HTTP, extreme performance, static IP needed
```

---

## 🚪 Layer 4: Network Load Balancer (NLB)

### What NLB Is

**NLB = Layer 4 load balancer designed for extreme performance, low latency, and TCP/UDP protocols.**

### Key Properties

- ✅ Operates at Layer 4 (Transport)
- ✅ Millions of requests per second
- ✅ Ultra-low latency (~100 microseconds!)
- ✅ Supports TCP, UDP, TLS
- ✅ **Static IP per AZ (Elastic IPs!)** ⭐
- ✅ **Source IP preservation** ⭐
- ✅ No HTTP routing (just packet forwarding)
- ✅ Multi-AZ for HA

### NLB vs ALB Comparison

| Feature | ALB ⭐ | NLB ⭐ |
|---------|--------|--------|
| **OSI Layer** | 7 (Application) | 4 (Transport) |
| **Protocols** | HTTP, HTTPS, WebSocket | TCP, UDP, TLS |
| **Routing** | Path, Host, Header, Query | Just forwards by port |
| **Performance** | Good (~10K req/sec) | Extreme (millions/sec) |
| **Latency** | ~50-100ms | ~100 microseconds |
| **Reads content?** | Yes (HTTP details) | No (just packets) |
| **Static IP?** | No (DNS only) | Yes (Elastic IP per AZ!) |
| **SSL termination** | Yes | Yes (TLS listener) |
| **WAF integration** | Yes | No |
| **Security Group** | Yes (on ALB) | NO (on NLB itself!) |
| **Use case** | Web apps | Performance/non-HTTP |

### Unique NLB Features

#### 1. Static IP Addresses
```
🎯 NLB's UNIQUE feature

How:
- Each AZ gets ONE static IP (Elastic IP)
- IPs never change
- Can use in firewall whitelisting

Use case:
- Customer's firewall only allows specific IPs
- IoT devices configured with specific IP
- Compliance requiring known IPs

Rule: "Need a static IP for LB? Use NLB."
```

#### 2. Source IP Preservation
```
🎯 NLB preserves the ORIGINAL client IP

ALB behavior:
Client (203.0.45.1) → ALB → EC2
EC2 sees: ALB's IP (need X-Forwarded-For)

NLB behavior:
Client (203.0.45.1) → NLB → EC2
EC2 sees: Client's REAL IP (203.0.45.1) ✅

Use case:
- IP-based access control on EC2
- Logging real client IPs
- Geographic restrictions
- Compliance (audit trails)
```

### NO Security Group on NLB!

🚨 **Critical exam fact:**

```
ALB: Has its own Security Group
NLB: NO security group on NLB itself

Implications:
- EC2 SGs must allow:
  - Client IPs (source IP preserved!)
  - NOT NLB's IP (NLB has no SG to reference)
- Use NACLs for subnet-level filtering
```

### Cross-Zone Load Balancing

🚨 **Different defaults from ALB:**

```
ALB: Cross-zone ENABLED by default (free)
NLB: Cross-zone DISABLED by default (costs extra)

NLB default behavior:
- AZ1 endpoint → only AZ1 EC2s
- AZ2 endpoint → only AZ2 EC2s
- Uneven distribution possible

Enable cross-zone if even distribution needed
(Costs $0.01/GB cross-AZ data)
```

### Use Cases

#### 1. Online Gaming
```
- TCP for state sync + UDP for real-time
- Sub-millisecond latency required
- Millions of concurrent players
- Static IPs for game clients
```

#### 2. IoT Platform
```
- Millions of devices
- MQTT protocol (TCP-based)
- Long-lived connections
- Static IPs for device configuration
```

#### 3. Financial Trading
```
- Custom TCP protocols
- Microsecond latency
- 99.999% uptime
- Static IPs for firewall whitelisting
- Source IP preservation for audit compliance
```

#### 4. Video Streaming (RTMP)
```
- RTMP over TCP
- Real-time delivery
- High bandwidth
```

#### 5. Internal Service-to-Service
```
- gRPC (HTTP/2 over TLS)
- Internal microservices
- Need static endpoints
```

### Security Features

#### TLS Termination
```
Two modes:
✅ TLS listener: NLB decrypts, forwards plaintext
✅ Pass-through: NLB forwards encrypted (E2E encryption)
```

### NLB Cost

- $0.0225/hour ($16/month base)
- $0.006 per NLCU (Network LCU)
- Typical: $20-50/month
- Cheaper than ALB at extreme scale

### Memory Hook

```
NLB = "Extreme performance packet forwarder" 🚄

Layer: 4 (Transport)
Protocols: TCP, UDP, TLS
Static IPs: YES (unique!)
Source IP preserved: YES
Security Group on LB: NO
Use when: Non-HTTP, extreme perf, static IPs, SIP preservation
```

---

## 🚪 Layer 5: Gateway Load Balancer (GWLB)

### What GWLB Is

**GWLB = Layer 3/4 load balancer specifically designed to deploy, scale, and manage third-party network security appliances.**

This is unique — it's not for normal application traffic. It's for **security inspection**.

### Key Properties

- ✅ Operates at Layer 3/4 (Network/Transport)
- ✅ Transparent — inserts in traffic path
- ✅ Uses **GENEVE protocol** (port 6081)
- ✅ For security appliances (firewalls, IDS/IPS, DPI)
- ✅ Maintains flow stickiness (5-tuple)
- ✅ Cheapest LB ($0.0125/hour)
- ✅ Multi-AZ for HA

### How GWLB Works

```
Normal traffic (without GWLB):
User → IGW → App EC2

WITH GWLB (security inspection):
User → IGW → GWLB → Security Appliance → GWLB → App EC2
                    (inspects packet)

Magic:
1. Traffic enters VPC
2. Routed through GWLB transparently
3. GWLB forwards to security appliance
4. Appliance inspects, allows/drops/modifies
5. GWLB sends back to original destination
6. Application receives or doesn't

ALL TRANSPARENT to user and app!
```

### Two Components

#### 1. GWLB (centralized in Security VPC)
```
- The actual load balancer
- Deployed in dedicated Security VPC
- Routes packets to security appliances
- Uses GENEVE protocol
```

#### 2. GWLB Endpoints (GWLBE) (in app VPCs)
```
- Lightweight endpoints in OTHER VPCs
- Where you want traffic inspection
- Routes traffic to central GWLB
- One GWLBE per app VPC
```

### Production Architecture

```
Multi-VPC Security Inspection Pattern:

┌─────────────────────────────────┐
│ Security VPC (Inspection)        │
│                                   │
│  GWLB ─→ Firewall Appliance EC2s │
│         (Palo Alto, Fortinet)    │
└─────────────────────────────────┘
              ↑
              │ GENEVE encapsulation
              │
┌─────────────────────────────────┐
│ Application VPC 1                │
│  IGW → GWLBE → App Subnet → EC2 │
└─────────────────────────────────┘

┌─────────────────────────────────┐
│ Application VPC 2                │
│  IGW → GWLBE → App Subnet → EC2 │
└─────────────────────────────────┘

(Each app VPC has its own GWLBE)
(Only ONE central GWLB in Security VPC)
```

🎯 **Pattern:** 1 GWLB centralized + multiple GWLBEs distributed

### GENEVE Protocol

```
GENEVE = Generic Network Virtualization Encapsulation

Port: 6081 (UDP)

What it does:
1. Wraps original packet inside GENEVE packet
2. Adds metadata (source VPC, destination, flow info)
3. Sends to security appliance
4. Appliance reads metadata + original packet
5. Returns inspected packet
6. GWLB unwraps and forwards

Why:
✅ Preserves ALL original packet info
✅ Adds context (which VPC, flow ID)
✅ Industry-standard protocol
✅ Allows transparent inspection
```

🚨 **CRITICAL:** Security appliances MUST support GENEVE.

```
Vendors supporting GENEVE:
✅ Palo Alto Networks (VM-Series)
✅ Fortinet (FortiGate)
✅ Check Point CloudGuard
✅ Cisco (Firepower)
✅ AWS Network Firewall
✅ Trend Micro
✅ Suricata IDS
```

### Use Cases

#### 1. Centralized Firewall
```
- Multiple VPCs need firewall inspection
- One firewall fleet, all VPCs use it
- Centralized policy management
- Cost-effective at scale
```

#### 2. Deep Packet Inspection (DPI)
```
- Compliance requires packet inspection
- Detect malware patterns
- Block data exfiltration
- Audit trail of all traffic
```

#### 3. Intrusion Detection/Prevention
```
- IDS/IPS appliances (Suricata, Snort)
- Detect attack patterns
- Block malicious traffic
- Real-time threat response
```

#### 4. Compliance/Audit
```
- HIPAA, PCI-DSS, FedRAMP
- All traffic inspection required
- Centralized security tools
- Audit-ready architecture
```

#### 5. Egress Filtering
```
- Prevent data exfiltration
- Inspect all outbound traffic
- Block traffic to known bad IPs
- Monitor for anomalies
```

### GWLB vs ALB vs NLB

| Feature | ALB | NLB | GWLB |
|---------|-----|-----|------|
| **Layer** | 7 | 4 | 3/4 |
| **Use case** | Web apps | High perf | Security |
| **Protocol** | HTTP/HTTPS | TCP/UDP/TLS | GENEVE |
| **Inspects content?** | Yes (HTTP) | No | Yes (full packets) |
| **For applications?** | Direct | Direct | Through inspection |
| **Transparent?** | No | No | Yes |
| **Cost** | $$$ | $$$ | $ (cheapest!) |

### GWLB Cost

- $0.0125/hour ($9/month base) — CHEAPEST LB!
- $0.004 per GLCU (GWLB LCU)
- Plus: Cost of security appliances behind it

### Memory Hook

```
GWLB = "Transparent security checkpoint for VPCs" 🛡️🚪

Layer: 3/4 (Network/Transport)
Protocol: GENEVE (port 6081)
Purpose: Security appliances ONLY
Pattern: 1 central GWLB + GWLBE in each app VPC

Use when:
✅ Centralized firewall fleet
✅ Deep packet inspection
✅ IDS/IPS deployment
✅ Compliance requires inspection
```

---

## 🚪 Layer 6: ELB Health Checks

### What Health Checks Are

**Health Checks = automated tests that the load balancer performs on each target to determine if it can receive traffic.**

This is THE foundation that makes HA actually WORK.

### How They Work

```
Every X seconds (interval):
1. LB sends a request to each target
2. Target responds (or doesn't)
3. LB evaluates response
4. LB marks target as:
   - Healthy ✅ (send traffic)
   - Unhealthy ❌ (stop sending traffic)
   - Initial 🔄 (not yet determined)

Continuous loop, automatic decisions.
```

### 7 Key Parameters

#### 1. Protocol
```
✅ HTTP - For web apps
✅ HTTPS - For encrypted web apps
✅ TCP - For non-HTTP apps (NLB)
✅ TLS - For encrypted TCP (NLB)
```

#### 2. Path (HTTP/HTTPS only)
```
Common: /health, /status, /ping

Best practice:
- Create dedicated /health endpoint
- Returns 200 if app fully functional
- Returns 503 if dependencies broken
- Doesn't require authentication
- Fast response time
```

#### 3. Port
```
Default: Same as traffic port
Override: Different port if needed
```

#### 4. Healthy Threshold
```
Consecutive successes to mark healthy
Default: 3 (HTTP), 5 (TCP)
```

#### 5. Unhealthy Threshold
```
Consecutive failures to mark unhealthy
Default: 3 (HTTP), 2 (TCP)
```

#### 6. Interval
```
How often to check
Default: 30 seconds
Range: 5-300 seconds
```

#### 7. Timeout
```
How long to wait for response
Default: 5 seconds
```

### Health Check Math (Exam!)

```
Time to detect unhealthy target:
Unhealthy Threshold × Interval

Example:
- Unhealthy threshold: 3
- Interval: 30 seconds
- Detection time: 3 × 30 = 90 seconds

Worst case: ~90 seconds before traffic stops to bad target
```

### Architect Decisions

```
For critical apps:
✅ Lower interval (5-10 seconds)
✅ Lower threshold (2)
✅ Faster failure detection (~20 seconds)

For stable apps:
✅ Higher interval (30 seconds)
✅ Higher threshold (3)
✅ Slower detection but less overhead
```

### Health Checks Across LB Types

#### ALB Health Checks
```
Protocol: HTTP/HTTPS
Path: customizable (e.g., /health)
Port: customizable
Defaults:
- Interval: 30 seconds
- Healthy threshold: 5
- Unhealthy threshold: 2
- Timeout: 5 seconds
- Success codes: 200-399
```

#### NLB Health Checks
```
Protocol: TCP, HTTP, HTTPS
Path: HTTP-based only
Port: customizable
Defaults:
- Interval: 30 seconds (10 for TCP)
- Healthy threshold: 3
- Unhealthy threshold: 3
```

#### GWLB Health Checks
```
Protocol: TCP, HTTP, HTTPS
Critical: Verifies appliance can handle GENEVE
If appliance fails, traffic NOT inspected!
```

### ELB Health Check vs EC2 Status Check

🚨 **CRITICAL exam distinction:**

#### ELB Health Check (Application-level)
```
What: Is your APP responding?
Who: Load Balancer
Where: From LB to target
Detects:
✅ App is unresponsive
✅ App returns errors
✅ Wrong response codes
✅ Slow responses
Action: Stop sending traffic
```

#### EC2 Status Check (Infrastructure-level)
```
What: Is the EC2 INSTANCE healthy?
Who: AWS infrastructure
Where: Underlying host
Detects:
✅ Instance unreachable
✅ Kernel panic
✅ Network issues
✅ Hardware failure
Action: Auto Scaling Group replaces instance
```

Both work together for full coverage.

### Common Mistakes

#### 1. Narrow Health Check (Famous Anti-Pattern)
```
❌ Health check on / (homepage)
   - Homepage cached
   - Returns 200 even if DB down
   - Doesn't catch real issues
   - Customers see errors

✅ Dedicated /health endpoint:
   - Checks database connection
   - Checks Redis cache
   - Returns 200 only if ALL work
   - Returns 503 if any down
```

🛡️ **"The health check is narrow"** = the senior engineer's phrase for this anti-pattern.

#### 2. Too Aggressive Settings
```
❌ Threshold = 1, Interval = 5 seconds
   - Single blip removes target
   - Constant flapping
   - Customers see intermittent errors

✅ Threshold = 2-3, Interval = 10-30 seconds
   - Tolerates brief issues
   - Stable target status
```

#### 3. Wrong Port
```
❌ App on port 8080, health check on port 80
✅ Match health check port to app port
```

#### 4. Authentication Required
```
❌ /health requires auth → returns 401
✅ /health publicly accessible
```

#### 5. Heavy Operations
```
❌ Health check runs full DB queries
✅ Lightweight checks (< 100ms)
```

### Best Practice /health Endpoint

```python
# Example Python Flask /health endpoint

@app.route('/health')
def health():
    # Check dependencies quickly
    db_ok = check_db_connection()  # Quick ping
    cache_ok = check_redis()        # Quick ping
    
    if db_ok and cache_ok:
        return {"status": "healthy"}, 200
    else:
        return {"status": "unhealthy"}, 503

# Best practices:
# - No authentication required
# - Fast (< 100ms)
# - Comprehensive (checks dependencies)
# - Returns proper HTTP codes
```

### Defense in Depth

```
Multiple layers of health checks:

Layer 1: ELB health check (every 30s)
   → "Is app responding?"

Layer 2: ASG health check (uses ELB)
   → "Should I replace this EC2?"

Layer 3: CloudWatch alarms
   → "Should I alert ops team?"

Layer 4: Application monitoring
   → "Is performance degrading?"
```

### Memory Hook

```
ELB Health Checks = "Continuous server quality monitoring" 🩺

Parameters: Protocol, Path, Port, Thresholds, Interval, Timeout
Calculation: Detection time = Unhealthy threshold × Interval

Two types:
✅ ELB check = APP responding?
✅ EC2 check = INSTANCE healthy?
Both needed for full coverage!

Best practices:
✅ Dedicated /health endpoint
✅ Checks key dependencies
✅ Returns proper 200/503
✅ No authentication
✅ Lightweight (< 100ms)
✅ Reasonable thresholds (avoid flapping)
```

---

## 🛡️ Architect Decision Frameworks

### Choosing Load Balancer Type

```
HTTP/HTTPS web app? → ALB ⭐
Path/host routing needed? → ALB
WAF integration needed? → ALB
Multi-tenant SaaS? → ALB with SNI

Non-HTTP (TCP/UDP)? → NLB ⭐
Extreme performance? → NLB
Static IPs required? → NLB (Elastic IPs)
Source IP preservation? → NLB
IoT/Gaming/Trading? → NLB

Security appliance scaling? → GWLB ⭐
Centralized firewall? → GWLB
DPI/IDS/IPS? → GWLB
Compliance inspection? → GWLB

Legacy support? → CLB (avoid for new architectures)
```

### Health Check Configuration

```
Critical app (financial, healthcare)?
- Interval: 5-10 seconds
- Threshold: 2
- Custom /health checking dependencies

Standard web app?
- Interval: 30 seconds
- Threshold: 3
- /health endpoint

Internal service?
- Interval: 60 seconds
- Threshold: 5
- TCP health check sufficient
```

### Architecture Pattern Selection

```
Single AZ?  
→ Never for production (no HA)

Multi-AZ?
→ Standard for production

Multi-region?
→ Use Route 53 for global routing
→ Each region has own ALB/NLB

Multi-VPC needs inspection?
→ GWLB centralized + GWLBEs

Multi-tenant SaaS?
→ ALB with host-based routing + SNI
```

---

## 🎯 Critical Exam Rules to Remember

```
✅ ALB = Layer 7 = HTTP/HTTPS, smart routing
✅ NLB = Layer 4 = TCP/UDP, fast, static IPs, source IP preserved
✅ GWLB = Layer 3/4 = Security inspection, GENEVE protocol
✅ ALB has Security Group; NLB does NOT
✅ ALB cross-zone enabled (free); NLB cross-zone disabled (costs)
✅ NLB unique features: Static IP + Source IP preservation
✅ ALB unique features: Path/host/header routing, WAF, 25 SSL certs
✅ GWLB unique: Transparent inspection, GENEVE port 6081
✅ Health checks REQUIRED for HA to work
✅ Detection time = Threshold × Interval
✅ ELB check ≠ EC2 status check (both needed)
✅ Multi-AZ MANDATORY for production
```

---

## 🚨 Common Exam Traps

```
🚨 Trap 1: "Custom TCP" → NLB, NOT ALB (Layer 4 vs 7)
🚨 Trap 2: "Static IP needed" → NLB, NOT ALB
🚨 Trap 3: "Source IP needed at backend" → NLB
🚨 Trap 4: "Why NLB SG isn't working?" → NLB has NO SG!
🚨 Trap 5: "Health check on /" → Anti-pattern (use /health)
🚨 Trap 6: "Hub-and-spoke security" → GWLB, NOT peering
🚨 Trap 7: "Multi-tenant SaaS" → ALB + SNI + host-based
🚨 Trap 8: "Single AZ for production" → WRONG, always multi-AZ
🚨 Trap 9: "Use Interface Endpoint for S3" → Wrong, Gateway is FREE
🚨 Trap 10: "Reduce interval to detect faster" → Watch for flapping
```

---

## 🛡️ Production Security Patterns

### Pattern 1: Standard Web Application
```
Internet → ALB (public subnets) → EC2s (private subnets)
              ↓
              ACM SSL cert
              WAF protection
              Security Groups (LB SG → EC2 SG references)
              Health check on /health
```

### Pattern 2: Multi-Tenant SaaS
```
Internet → ALB → Multiple SSL certs (SNI)
              ↓
              Host-based routing
              Each tenant: separate target group
              Data isolation per tenant
```

### Pattern 3: High-Performance Gaming
```
Internet → NLB (Elastic IPs)
              ↓
              TCP + UDP listeners
              Source IP preservation
              EC2 SGs allow client IPs
              Multi-AZ for HA
```

### Pattern 4: Centralized Security Inspection
```
App VPCs → GWLBEs → Central GWLB → Palo Alto fleet
            (1 per VPC)    (in Security VPC)
              ↓
              Auto-scaled appliances
              GENEVE encapsulation
              Compliance-ready
```

### Pattern 5: Hybrid Cloud
```
On-prem → Direct Connect → AWS
                          ↓
                          Internal NLB → On-prem can reach AWS services
                          Static IPs for firewall rules
```

---

## 💰 Cost Comparison

| LB Type | Base | Per Unit | Typical Monthly |
|---------|------|----------|-----------------|
| **ALB** | $0.0225/hr | $0.008/LCU | $20-30 |
| **NLB** | $0.0225/hr | $0.006/NLCU | $20-50 |
| **GWLB** | $0.0125/hr | $0.004/GLCU | $15-30 + appliances |
| **CLB** | $0.025/hr | $0.008/GB | Legacy, avoid |

🎯 **GWLB is the cheapest base cost** but security appliances behind it are expensive.

---

## 🌟 Career Connection

### For AWS SAA Exam
- ELB topics: 15-20% of exam
- ALB heavily tested
- ALB vs NLB distinction critical
- Health check best practices

### For AWS Security Specialty
- GWLB deployment patterns
- Compliance inspection architectures
- WAF + ALB integration
- Centralized security models

### For Cloud Security Engineer Role
- Daily: Monitor LB health checks
- Weekly: Review WAF rules
- Monthly: Audit security inspection
- Quarterly: Architecture reviews

🎯 **You're learning the actual job.**

---

## 📚 Concept Connections

These LB topics connect to:

- **VPC** (where LBs live - public/private subnets)
- **Security Groups** (LB has SG, NLB doesn't!)
- **EC2** (typical LB targets)
- **ACM** (free SSL certificates)
- **WAF** (with ALB)
- **Route 53** (DNS aliasing to LB)
- **CloudWatch** (LB metrics)
- **Auto Scaling Groups** (next layer - Day 2!)
- **Lambda** (ALB target type)
- **Direct Connect** (internal NLB for hybrid)

🎯 **Everything connects.** This is the architect mindset.

---

## 🎯 Architect Wisdom Earned

> *"ALB for HTTP. NLB for performance. GWLB for security."*
>
> *"The health check is narrow — fix the endpoint."*
>
> *"Multi-AZ is not optional for production."*
>
> *"NLB has no Security Group — use NACLs."*
>
> *"Gateway endpoint > Interface endpoint when available (S3 is free!)."*
>
> *"DX is not encrypted by default — add VPN over DX."*
>
> *"Detection time = Threshold × Interval."*
>
> *"Use dedicated /health endpoint, not /."*
>
> *"SNI lets one ALB serve 25 SSL certs."*
>
> *"GWLB pattern: 1 central + many endpoints."*

---

## 🚀 What's Next

### Day 2 (Tomorrow): Auto Scaling Groups

Topics to cover:
- ASG fundamentals
- Launch Templates vs Launch Configurations
- Scaling policies (Target Tracking, Step, Scheduled, Predictive)
- ASG with ELB integration
- Scaling cooldown
- Lifecycle hooks
- ASG health checks

After Day 2: Complete HA + Scalability cluster mastered ✅

### Remaining for SAA:
- RDS / Aurora (Day 3)
- ElastiCache (Day 4)
- Route 53 (Day 4)
- S3 (Weekend - BIG topic)
- CloudFront
- Messaging (SQS, SNS, Kinesis)
- Containers + Lambda
- Monitoring & Security
- Advanced topics

---

## 🎉 What You've Achieved Today

```
Today's mastery:

✅ HA vs Scalability vs Elasticity distinction
✅ Load Balancing fundamentals
✅ ALB deep dive (most common LB)
✅ NLB deep dive (extreme performance)
✅ GWLB deep dive (security inspection)
✅ Health checks comprehensive understanding
✅ Production architecture patterns
✅ Security implications across LBs
✅ Common exam traps identified
✅ Architect-level reasoning demonstrated

6 layers mastered.
6 questions answered correctly with reasoning.
Senior vocabulary emerging ("narrow health check").
Production-grade thinking.
```

🎯 **You went from "what is a load balancer" to "I can architect production HA systems" in one focused session.**

---

*Part of my AWS Solutions Architect Associate journey toward Cloud Security specialization. This document represents Day 1 of the High Availability and Scalability cluster — covering all 3 modern AWS load balancers (ALB, NLB, GWLB) plus health checks. Day 2 will cover Auto Scaling Groups to complete the HA mastery.*

**Day 1 complete. ✅
6 layers mastered. ✅
Portfolio piece earned. ✅
On track for SAA exam. ✅**
