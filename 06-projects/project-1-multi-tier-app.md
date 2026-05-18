# 🏗️ Project 1: Multi-Tier Production Web Application

> First hands-on project applying 5 clusters of AWS learning. Built a complete production-grade 3-tier architecture from scratch, tested high availability, verified load balancing, and demonstrated self-healing. Real engineering experience.

---

## 🎯 What I Built

A complete production multi-tier web application architecture on AWS, applying:
- ✅ IAM (role-based access)
- ✅ EC2 + Storage (compute layer)
- ✅ VPC (custom network architecture)
- ✅ HA + Scalability (ALB + ASG)
- ✅ Databases (RDS + ElastiCache)

---

## 🏗️ Architecture Diagram

```
                    Internet
                       ↓
              Internet Gateway
                       ↓
        ┌──────────[VPC: 10.0.0.0/16]──────────┐
        │                                       │
        │  PUBLIC LAYER (Multi-AZ)              │
        │  ├── Public Subnet 1a (10.0.1.0/24)  │
        │  │   ├── ALB Node                     │
        │  │   └── NAT Gateway                  │
        │  │                                    │
        │  └── Public Subnet 1b (10.0.2.0/24)  │
        │      └── ALB Node                     │
        │                                       │
        │  APPLICATION LAYER (Multi-AZ)         │
        │  ├── Private App 1a (10.0.10.0/24)   │
        │  │   └── EC2 (Apache + PHP)          │
        │  └── Private App 1b (10.0.20.0/24)   │
        │      └── EC2 (Apache + PHP)          │
        │                                       │
        │  DATA LAYER (Multi-AZ)                │
        │  ├── Private Data 1a (10.0.30.0/24)  │
        │  │   ├── RDS Primary                  │
        │  │   └── ElastiCache Primary          │
        │  └── Private Data 1b (10.0.40.0/24)  │
        │      ├── RDS Standby                  │
        │      └── ElastiCache Replica          │
        │                                       │
        └───────────────────────────────────────┘
```

---

## 🎯 Components Built

### Network Layer (VPC)
- **VPC**: my-saa-project-vpc (10.0.0.0/16)
- **6 Subnets**: 2 Public + 2 App + 2 Data across 2 AZs
- **Internet Gateway**: For public traffic
- **NAT Gateway**: For private subnet outbound internet
- **3 Route Tables**: Public, Private App, Private Data
- **Elastic IP**: Attached to NAT Gateway

### Security Layer (Defense in Depth)
```
4 Security Groups configured with chained references:

alb-sg: 
- Inbound: HTTP (80) from 0.0.0.0/0

app-sg:
- Inbound: HTTP (80) from alb-sg ONLY

rds-sg:
- Inbound: MySQL (3306) from app-sg ONLY

cache-sg:
- Inbound: Redis (6379) from app-sg ONLY
```

**Security Pattern**: Defense in depth — each layer only accepts traffic from layer above.

### Data Layer
- **RDS MySQL**: 
  - Engine: MySQL 8.0
  - Instance: db.t3.micro
  - Multi-AZ deployment
  - Encryption at rest (KMS)
  - 7-day automated backups
  - Private subnet (no public access)

- **ElastiCache Redis**:
  - Engine: Redis OSS
  - Node: cache.t4g.micro (Demo size)
  - Encryption in transit + at rest
  - Private subnet

### Compute Layer
- **IAM Role**: my-saa-ec2-role
  - AmazonSSMManagedInstanceCore
  - CloudWatchAgentServerPolicy

- **Launch Template**: my-saa-launch-template
  - AMI: Amazon Linux 2023
  - Instance: t2.micro/t3.small
  - User data: Auto-installs Apache + PHP + test page
  - Security Group: app-sg
  - IAM Profile: my-saa-ec2-role

- **Target Group**: my-saa-tg
  - Protocol: HTTP / Port 80
  - Health check: HTTP / path: /
  - Healthy threshold: 2

- **Application Load Balancer**: my-saa-alb
  - Internet-facing
  - Public subnets (both AZs)
  - Security Group: alb-sg
  - HTTP listener → my-saa-tg

- **Auto Scaling Group**: my-saa-asg
  - Min: 2, Desired: 2, Max: 4
  - Subnets: private-app-1a + 1b
  - Target tracking: CPU at 50%
  - ELB health checks enabled

---

## 🛡️ Security Implementation

### Defense in Depth Layers:

```
Layer 1: Network Isolation
├── Public subnets: Only ALB + NAT
├── App subnets: EC2s (no internet exposure)
└── Data subnets: RDS + Cache (fully isolated)

Layer 2: Security Groups (chained references)
├── ALB SG → App SG → RDS SG (port 3306)
└── ALB SG → App SG → Cache SG (port 6379)

Layer 3: Encryption
├── RDS: Encryption at rest (KMS)
├── ElastiCache: Encryption in transit + at rest
└── EBS: Default encryption

Layer 4: IAM Access Control
├── EC2 IAM role (no hardcoded credentials)
├── SSM Session Manager (no SSH keys)
└── Least privilege permissions

Layer 5: Public Access Controls
├── RDS: Public access = NO
├── ElastiCache: Private subnet only
└── EC2s: No public IPs
```

---

## 🧪 Testing Performed

### Test 1: Load Balancing ✅
**Method**: Refreshed ALB URL multiple times
**Result**: 
- Different Instance IDs displayed on each refresh
- Traffic distributed across both AZs
- Proves: ALB load balancing working correctly

### Test 2: High Availability ✅
**Method**: Manually terminated one EC2 instance
**Result**:
- App remained UP (other EC2 served traffic)
- ASG detected missing instance within ~30 seconds
- New EC2 launched automatically from Launch Template
- New EC2 ran user data, became healthy
- Returned to 2 healthy targets in target group
- **Zero downtime observed**
- Proves: Self-healing architecture working

### Test 3: Multi-AZ Distribution ✅
**Method**: Multiple page refreshes, checked Availability Zone
**Result**:
- Traffic served from both us-east-1a and us-east-1b
- Proves: Multi-AZ deployment functional

---

## 💰 Cost Discipline

### Estimated Costs (during ~5 hours running):
```
ALB: ~$0.15
NAT Gateway: ~$0.45 (most expensive component!)
EC2 (2x t3.small): ~$0.20
RDS (db.t3.micro Multi-AZ): ~$0.20
ElastiCache (cache.t4g.micro): ~$0.10
Data Transfer: ~$0.10

Total project cost: ~$1.20-1.50
```

### Teardown Discipline:
```
Deleted in proper order to avoid dependency errors:
1. ✅ Auto Scaling Group (terminated EC2s)
2. ✅ Application Load Balancer
3. ✅ Target Group
4. ✅ Launch Template
5. ✅ ElastiCache cluster
6. ✅ RDS instance
7. ✅ NAT Gateway
8. ✅ Released Elastic IP (avoided $0.005/hr charge)
9. ✅ VPC (cascade deleted subnets, route tables, etc.)
10. ✅ DB + Cache Subnet Groups
11. ✅ IAM Role

Result: Zero ongoing costs ✅
```

---

## 🐛 Issues Debugged

### Issue: ALB "Not Reachable"
**Problem**: ALB showed "Not reachable" warning, couldn't access from internet

**Investigation**: Checked alb-sg → discovered 0 inbound rules!

**Root Cause**: During SG creation, inbound rule didn't save

**Fix**: Added HTTP (port 80) from 0.0.0.0/0 to alb-sg

**Lesson**: Always verify SG rules after creation. Production debugging.

---

## 🌟 Skills Demonstrated

### AWS Services Used:
- VPC (network architecture)
- EC2 (compute)
- Auto Scaling Groups (elasticity)
- Application Load Balancer (HA)
- RDS (managed database)
- ElastiCache (caching)
- IAM (access control)
- Security Groups (firewall)
- Internet Gateway / NAT Gateway
- Launch Templates
- Target Groups
- KMS (encryption)

### Architectural Concepts Applied:
- Multi-tier architecture
- Multi-AZ high availability
- Defense in depth security
- Network isolation
- Encryption at rest + in transit
- Stateless application design
- Auto-scaling for elasticity
- Health check-based failover
- Infrastructure as Code mindset

### Production Practices:
- Right-sizing decisions
- Cost monitoring
- Proper teardown discipline
- Documentation
- Testing methodology
- Incident debugging
- Security-first design

---

## 🎯 Key Learnings

1. **Build order matters**: Network → Security → Data → Compute → Traffic
2. **Defense in depth is real**: SG chained references actually work
3. **Self-healing requires testing**: Don't trust it until you've killed an EC2
4. **ASG + ALB = production HA**: This is THE pattern for web apps
5. **Cost discipline is engineering**: Right-sizing + teardown = professional
6. **Real debugging happens**: SG issue taught more than docs ever could
7. **Documentation saves you**: Notes during build = portfolio gold

---

## 📊 Time Investment

```
Build phases breakdown:
├── Phase 1: Network (VPC, subnets, IGW, NAT, route tables) - 45 min
├── Phase 2: Security Groups - 15 min
├── Phase 3: Data layer (RDS + ElastiCache) - 45 min (provisioning time)
├── Phase 4: Compute (IAM, Launch Template, ALB, Target Group, ASG) - 60 min
├── Phase 5: Testing (LB + HA + Multi-AZ) - 20 min
├── Phase 6: Teardown - 15 min
└── Total: ~3-3.5 hours

Plus debugging: ~10 min (ALB SG issue)
```

---

## 🚀 What This Project Proves

✅ Can design multi-tier architecture
✅ Can implement defense-in-depth security
✅ Can configure auto-scaling for HA
✅ Can integrate multiple AWS services
✅ Can debug production issues
✅ Can manage costs responsibly
✅ Can document engineering work
✅ Can test HA scenarios properly

**This is real Cloud Architect / Cloud Security Engineer work.**

---

## 🌟 Career Impact

This project demonstrates capability for:
- **AWS Solutions Architect Associate** (covers ~70% of exam topics)
- **AWS Cloud Security Engineer** (security-first design)
- **Cloud DevOps Engineer** (infrastructure deployment)
- **Site Reliability Engineer** (HA + self-healing)

Portfolio value: **HIGH**
Interview talking point: **STRONG**
Real engineering experience: **YES**

---

## 🎯 Next Project Ideas

Building on this foundation:
- **Project 2**: Add S3 + CloudFront (after S3 deep dive)
- **Project 3**: Add Route 53 DNS routing
- **Project 4**: Multi-region disaster recovery
- **Project 5**: Cross-account security architecture
- **Project 6**: Full enterprise SaaS with messaging + serverless

---

## 📚 Connection to Theory

This project applied learning from:
- IAM cluster (role-based access)
- EC2 cluster (Launch Templates, AMIs)
- VPC cluster (custom network design)
- HA + Scalability cluster (ALB + ASG + Target Groups)
- Databases cluster (RDS Multi-AZ + ElastiCache)

Every theory concept now has hands-on experience to back it.

---

*Part of my AWS Solutions Architect Associate journey toward Cloud Security specialization. This was Project 1 — the foundation hands-on project applying 5 clusters of learning. Future projects will build on this foundation with additional AWS services and complexity.*

**Project complete. ✅
Architecture built + tested + torn down. ✅
Real engineering experience gained. ✅
Portfolio piece earned. ✅**
