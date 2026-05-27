# 🏥 Project 2: HIPAA-Compliant Healthcare Platform — Design Document

> Complete architectural design for HIPAA-compliant healthcare platform on AWS. This document is my reference for building, testing, and discussing in interviews.

---

## 📋 Project Overview

### Business Context

**Company:** HealthCare Solutions Inc.  
**Role:** Cloud Security Engineer  
**Mission:** Build HIPAA-compliant patient portal in 90 days  
**Stakes:** $50K-$1.5M fines per HIPAA violation  
**Goal:** Pass HIPAA audit + serve 10K+ patients securely

### What Is HIPAA?

```
HIPAA = Health Insurance Portability and Accountability Act

Protects PHI (Protected Health Information):
- Patient names, addresses, contact info
- Medical records, diagnoses
- Insurance information
- Treatment data, lab results
- ANY identifiable health information
```

### Architecture Goals

```
✅ Encryption at rest + in transit (everything!)
✅ Network isolation (no direct internet access to data)
✅ Audit ALL access (CloudTrail + Flow Logs)
✅ Least privilege access (IAM)
✅ High availability (Multi-AZ)
✅ Disaster recovery (Cross-region)
✅ 6+ year audit retention
✅ Documented + tested procedures
```

---

## 🎯 5 Architectural Decisions

### Decision 1: Encryption Strategy 🔐

**Choice:** SSE-KMS with Customer Managed Keys (CMK)

**Architecture:**

| Service | Encryption | Key Type |
|---------|-----------|----------|
| S3 | SSE-KMS | CMK |
| RDS | KMS encryption | CMK |
| EBS | KMS encryption | CMK |
| ElastiCache | At-rest + in-transit | CMK |
| In-transit | HTTPS/TLS | ACM cert |

**Why CMK over AWS Managed Keys:**
- ✅ Customer controls key access
- ✅ Audit trail via CloudTrail
- ✅ Can revoke access instantly
- ✅ HIPAA-compliant
- ✅ Cost: $1/month per key (worth it!)

**Multi-Region Strategy:**
- Use Multi-Region KMS keys
- Same key ID in primary + DR region
- Enables cross-region data portability

**Bucket Policy Enforcement:**

```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": [
    "arn:aws:s3:::medical-records/*"
  ],
  "Condition": {
    "Bool": {
      "aws:SecureTransport": "false"
    }
  }
}
```

---

### Decision 2: Network Architecture 🌐

**Choice:** 3-Tier VPC with Multi-AZ

**Design:**

```
VPC: 10.0.0.0/16

Public Subnets (2 AZs):
- us-east-1a: 10.0.1.0/24
- us-east-1b: 10.0.2.0/24
- Contains: ALB, NAT Gateway

Private App Subnets (2 AZs):
- us-east-1a: 10.0.10.0/24
- us-east-1b: 10.0.11.0/24
- Contains: EC2 web servers

Private Data Subnets (2 AZs):
- us-east-1a: 10.0.20.0/24
- us-east-1b: 10.0.21.0/24
- Contains: RDS, ElastiCache
- NO internet access
```

**Security Groups (4 total):**

```
1. alb-sg:
   - Inbound: 443 from 0.0.0.0/0
   - Outbound: 443 to app-sg

2. app-sg:
   - Inbound: 443 from alb-sg
   - Outbound: 3306 to rds-sg, 6379 to cache-sg, 443 to internet

3. rds-sg:
   - Inbound: 3306 from app-sg ONLY
   - Outbound: None

4. cache-sg:
   - Inbound: 6379 from app-sg ONLY
   - Outbound: None
```

**VPC Endpoints:**
- S3 Gateway Endpoint (FREE!)
- KMS Interface Endpoint
- Secrets Manager Endpoint
- CloudWatch Logs Endpoint
- SSM Endpoints (for Session Manager)

**No Bastion Host:**
- Use AWS Systems Manager Session Manager
- No SSH keys to manage
- All sessions logged
- HIPAA-friendly

**Web Layer Protection:**
- CloudFront (CDN + DDoS via Shield Standard)
- AWS WAF (SQL injection, XSS, rate limiting)
- ACM cert in us-east-1

---

### Decision 3: Audit Strategy 📝

**Choice:** Multi-layer comprehensive logging

**Log Sources:**

```
1. CloudTrail (API calls)
   - Management events: ENABLED
   - Data events: ENABLED for S3 + KMS
   - Log file validation: ENABLED
   - Multi-region trail
   
2. VPC Flow Logs (network traffic)
   - All accepts + rejects
   - Stored in S3 + CloudWatch
   
3. S3 Access Logs (request details)
   - Free, hourly
   - Backup for CloudTrail data events
   
4. RDS Audit Logs (database queries)
   - MariaDB Audit Plugin enabled
   - All queries captured
   
5. Application Logs (CloudWatch Logs)
   - PHI access events
   - Authentication events
   - Business logic events
   
6. ALB Access Logs (web requests)
   - Every HTTP request
   - Source IP, response time
```

**Log Storage:**

```
Multiple S3 buckets (separate):
- logs-cloudtrail-prod
- logs-vpc-flow-logs
- logs-alb-access
- logs-s3-access
- logs-application
- logs-glacier-archive

Security:
✅ SSE-KMS encrypted
✅ Versioning enabled
✅ MFA Delete enabled
✅ Object Lock (WORM)
✅ Bucket policies enforce HTTPS

Lifecycle:
- Hot (CloudWatch): 30 days
- Warm (S3 Standard): 90 days
- Cold (S3 IA): 1 year
- Archive (Glacier Deep Archive): 7+ years
```

**Real-Time Alerts:**

```
CloudWatch Logs → Metric Filter → Alarm → SNS

Alert scenarios:
- Failed login attempts (3 in 5 min)
- Unusual PHI access patterns
- Root account usage
- Encryption disabled
- Configuration changes
- Database export commands
```

---

### Decision 4: Identity & Access Management 🔐

**Choice:** AWS SSO + IAM Roles + Least Privilege

**Architecture:**

```
Corporate Identity (Active Directory)
        ↓
AWS IAM Identity Center (SSO)
        ↓
Permission Sets:
├── DoctorAccess (medical records, their patients)
├── NurseAccess (records, their department)
├── PatientAccess (own records only)
├── AdminAccess (admin tasks, no PHI)
├── DeveloperAccess (no production)
├── SecurityAccess (logs, audit)
└── BreakGlassAccess (emergency)
```

**For Humans:**
- ✅ AWS SSO (no IAM users!)
- ✅ MFA mandatory (virtual minimum)
- ✅ Hardware MFA for admins
- ✅ Temporary credentials only
- ✅ Permission sets per role

**For Services:**
- ✅ IAM Roles (never users!)
- ✅ No hard-coded access keys
- ✅ Temporary credentials (auto-rotate)
- ✅ EC2 → Role → S3/RDS/KMS

**Secrets Management:**
- ✅ AWS Secrets Manager (DB passwords)
- ✅ Auto-rotation every 30 days
- ✅ KMS encrypted
- ✅ Access controlled via IAM

**Emergency Access:**
- ✅ Break-glass user (hardware MFA)
- ✅ Sealed credentials
- ✅ Documented procedure
- ✅ Auto-alert when used
- ✅ Mandatory review within 24h

**Example Policy (Doctor):**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:PutObject"],
    "Resource": "arn:aws:s3:::patient-records/*",
    "Condition": {
      "StringEquals": {
        "s3:ExistingObjectTag/AssignedDoctor": "${aws:username}"
      }
    }
  }]
}
```

---

### Decision 5: Backup & Disaster Recovery 💾

**Choice:** Pilot Light DR Strategy

**RPO/RTO Targets:**
- RPO: < 1 hour (data loss)
- RTO: < 4 hours (downtime)

**Multi-Region Setup:**

```
Primary Region: us-east-1
- Full production active
- All resources running

Secondary Region: us-west-2 (DR)
- Pilot light (scaled-down)
- Continuously updated
- Ready to scale up
```

**Replication:**

```
RDS:
- Multi-AZ in primary (HA)
- Automated backups: 35 days (max)
- Cross-region snapshot daily
- Point-in-Time Recovery (PITR)

S3:
- Cross-Region Replication (CRR)
- Versioning enabled (required)
- Multi-Region KMS keys
- Standard replication (15 min SLA)

KMS:
- Multi-Region keys
- Replicated us-east-1 → us-west-2
- Same key ID both regions

Configuration:
- CloudFormation/Terraform in git
- Can recreate infrastructure
- Documented runbooks
```

**Route 53 Failover:**
- Primary: us-east-1
- Secondary: us-west-2
- Health checks every 30 seconds
- Auto-failover on failure

**Testing:**
- Quarterly walkthroughs
- Bi-annual partial failover
- Annual full failover
- Documented results

---

## 🧪 Test Cases (To Validate During Build)

### Encryption Tests

```
TEST 1: S3 Encryption Validation
GIVEN: Upload file to S3 without encryption header
WHEN: Bucket policy enforces encryption
THEN: Upload should FAIL
EXPECTED: HTTP 403 Forbidden

TEST 2: HTTPS Enforcement
GIVEN: Try to access S3 via HTTP
WHEN: Bucket policy requires HTTPS
THEN: Request should FAIL
EXPECTED: HTTP 403 Forbidden

TEST 3: KMS Key Revocation
GIVEN: Disable KMS CMK
WHEN: Try to read encrypted S3 object
THEN: Decryption should FAIL
EXPECTED: AccessDenied error

TEST 4: RDS SSL Enforcement
GIVEN: Connect to RDS without SSL
WHEN: Parameter group requires SSL
THEN: Connection should FAIL
EXPECTED: Connection refused
```

### Network Tests

```
TEST 5: RDS Internet Isolation
GIVEN: RDS in private data subnet
WHEN: Try to connect from internet
THEN: Connection should FAIL
EXPECTED: Timeout/Unreachable

TEST 6: VPC Endpoint Usage
GIVEN: S3 Gateway Endpoint configured
WHEN: EC2 accesses S3
THEN: Traffic should go through endpoint (private)
EXPECTED: VPC Flow Logs show internal IPs only

TEST 7: Security Group Rules
GIVEN: RDS SG only allows app SG
WHEN: Try to connect from public subnet
THEN: Connection should FAIL
EXPECTED: Connection refused

TEST 8: Session Manager Access
GIVEN: EC2 with SSM role, no SSH key
WHEN: Try to connect via Session Manager
THEN: Connection should SUCCEED
EXPECTED: Shell access via AWS Console
```

### Audit Tests

```
TEST 9: CloudTrail Capture
GIVEN: Make AWS API call
WHEN: CloudTrail is enabled
THEN: Event should appear in logs
EXPECTED: Event in CloudWatch within 5 minutes

TEST 10: Data Events for S3
GIVEN: Access S3 object
WHEN: Data events enabled
THEN: GetObject event logged
EXPECTED: User, action, resource visible

TEST 11: Tamper-Proof Logs
GIVEN: Try to delete CloudTrail log
WHEN: Object Lock enabled
THEN: Delete should FAIL
EXPECTED: AccessDenied

TEST 12: Real-Time Alert
GIVEN: 3 failed login attempts in 5 min
WHEN: Metric filter triggers
THEN: SNS notification sent
EXPECTED: Email + SMS received
```

### IAM Tests

```
TEST 13: Least Privilege
GIVEN: Doctor user logs in
WHEN: Tries to access other doctor's patient
THEN: Access should FAIL
EXPECTED: AccessDenied error

TEST 14: MFA Enforcement
GIVEN: User without MFA
WHEN: Tries to access sensitive resource
THEN: Access should FAIL
EXPECTED: MFA required prompt

TEST 15: Service Role Usage
GIVEN: EC2 with IAM role
WHEN: App calls S3 API
THEN: Should use role credentials (not user)
EXPECTED: CloudTrail shows role identity

TEST 16: User Offboarding
GIVEN: User disabled in AD
WHEN: User tries to login
THEN: Access should FAIL
EXPECTED: Authentication failure
```

### DR Tests

```
TEST 17: S3 Replication
GIVEN: Upload to primary bucket
WHEN: CRR is enabled
THEN: Object appears in DR bucket
EXPECTED: Replicated within 15 minutes

TEST 18: RDS Snapshot
GIVEN: Automated backups enabled
WHEN: Wait for backup window
THEN: Snapshot created
EXPECTED: Available for restore

TEST 19: PITR
GIVEN: Database with PITR
WHEN: Restore to 1 hour ago
THEN: New instance created with old data
EXPECTED: New instance available

TEST 20: Failover Routing
GIVEN: Primary endpoint unhealthy
WHEN: Health check fails
THEN: Route 53 routes to secondary
EXPECTED: DNS resolves to DR region
```

---

## 🎯 CTO/Interview Questions to Prepare For

### Encryption Questions

**Q1: "How do you ensure HIPAA-compliant encryption?"**

```
Answer:
"I use defense in depth with SSE-KMS and Customer 
Managed Keys (CMK). All data at rest is encrypted:
- S3 with SSE-KMS using CMK
- RDS with KMS encryption
- EBS volumes encrypted
- ElastiCache with at-rest encryption

All data in transit uses HTTPS/TLS:
- CloudFront with ACM cert
- ALB with ACM cert
- RDS forced SSL
- EC2 to S3 via HTTPS

I use CMK over AWS Managed Keys because:
- Full audit trail via CloudTrail
- Granular access control via IAM
- Can revoke access instantly
- HIPAA compliance requirements"
```

**Q2: "Why CMK over AWS Managed Keys?"**

```
Answer:
"CMK provides:
1. Customer control over the key
2. Granular IAM policies
3. Full audit trail
4. Key rotation control
5. Ability to disable/delete key

For HIPAA, the auditor needs to see:
- Who accessed encryption keys
- When they were used
- Ability to revoke access

AWS Managed Keys don't provide this level of 
control. The $1/month per key is negligible 
compared to compliance violations."
```

**Q3: "What's SSE-KMS with CMK?"**

```
Answer:
"SSE-KMS = Server-Side Encryption using KMS service
CMK = Customer Managed Key (type of key in KMS)

So SSE-KMS with CMK means:
- S3 server-side encrypts data
- Uses KMS service for encryption
- Specifically uses a CMK I control
- Full audit + control"
```

### Network Questions

**Q4: "Walk me through your network design."**

```
Answer:
"I use a 3-tier VPC architecture:

Public Tier:
- ALB receives HTTPS traffic
- NAT Gateway for outbound from private

App Tier (Private):
- EC2 web servers
- No direct internet access
- Accessible only from ALB

Data Tier (Private):
- RDS and ElastiCache
- No internet access at all
- Accessible only from App tier

I use VPC Endpoints for AWS service access:
- S3 Gateway Endpoint (free)
- KMS, Secrets Manager via Interface Endpoints
- Keeps traffic in AWS network

For admin access, Session Manager (no bastion).
For protection: WAF + Shield Standard."
```

**Q5: "Why VPC Endpoints?"**

```
Answer:
"VPC Endpoints provide:
1. Private connectivity to AWS services
2. No internet exposure
3. Lower NAT Gateway costs
4. Better security
5. HIPAA compliance

Without endpoints, traffic to S3 goes:
EC2 → NAT → Internet → S3 (exposed, costly)

With endpoint:
EC2 → VPC Endpoint → S3 (private, free for S3)

For HIPAA, this is critical because PHI 
never touches public internet."
```

**Q6: "Security Groups vs NACLs?"**

```
Answer:
"Both provide security but at different levels:

Security Groups:
- Instance-level firewall
- Stateful (return traffic auto-allowed)
- Allow rules only
- Reference other SGs

NACLs:
- Subnet-level firewall
- Stateless (must define both directions)
- Allow + Deny rules
- IP-based only

For HIPAA defense in depth, I use BOTH:
- SGs for instance-level control
- NACLs for subnet-level additional layer
- Together = layered security"
```

### Audit Questions

**Q7: "How do you audit access for HIPAA?"**

```
Answer:
"Comprehensive multi-layer audit:

1. CloudTrail captures all AWS API calls
   - Management events (config changes)
   - Data events (S3, KMS access)
   - Log file validation enabled

2. VPC Flow Logs capture network traffic
   - All accepts and rejects
   - Source, destination, ports

3. S3 Access Logs for object-level requests

4. RDS Audit Logs for database queries

5. Application logs to CloudWatch
   - PHI access events tagged
   - User actions tracked

All logs:
- Encrypted with KMS
- Stored in S3 with Object Lock
- Retained 7+ years
- Real-time alerts via SNS"
```

**Q8: "Show me who accessed Patient X-ray last Tuesday."**

```
Answer:
"I'd use CloudWatch Logs Insights:

fields @timestamp, userId, action, resource
| filter resource like /patient-12345/
| filter @timestamp > '2026-05-26' 
       and @timestamp < '2026-05-28'
| sort @timestamp desc

This queries multiple log sources:
- CloudTrail data events
- Application logs
- S3 access logs

Returns who, what, when, where in minutes.
Audit-ready response."
```

### IAM Questions

**Q9: "How do you enforce least privilege?"**

```
Answer:
"Multiple layers:

1. AWS SSO for centralized identity
   - Corporate AD federation
   - MFA mandatory

2. Permission Sets per role:
   - DoctorAccess: own patients only
   - NurseAccess: department patients
   - AdminAccess: no PHI

3. IAM Roles for services:
   - EC2 → Role → S3 (no keys)
   - Temporary credentials
   - Auto-rotation

4. Conditional policies:
   - Tag-based access control
   - IP restrictions
   - Time-based access

5. Secrets Manager for passwords

6. Hardware MFA for admins

7. Break-glass for emergencies"
```

**Q10: "What about service-to-service authentication?"**

```
Answer:
"I use IAM Roles, not access keys:

For EC2 accessing S3:
1. Create IAM Role with S3 permissions
2. Attach role to EC2 instance
3. App gets temporary credentials automatically
4. Credentials rotate every hour
5. No keys in code or config

For app passwords:
- Stored in AWS Secrets Manager
- Encrypted with KMS
- Auto-rotation enabled
- IAM controls access

Benefits:
- No long-lived credentials
- No keys to leak
- Automatic rotation
- Audit shows specific service"
```

### DR Questions

**Q11: "What's your DR strategy?"**

```
Answer:
"Pilot Light with multi-region:

RPO: 1 hour (max data loss acceptable)
RTO: 4 hours (max downtime acceptable)

Primary: us-east-1 (active)
Secondary: us-west-2 (pilot light)

Replication:
- RDS daily cross-region snapshots
- S3 Cross-Region Replication
- Multi-Region KMS keys
- IaC for infrastructure (CloudFormation)

Failover via Route 53:
- Health checks every 30s
- Automatic DNS failover

Testing:
- Quarterly walkthroughs
- Bi-annual partial failover
- Annual full DR test

Documented runbooks for everything."
```

**Q12: "How do you ensure backup integrity?"**

```
Answer:
"Multiple protections:

1. Encryption:
   - All backups encrypted with KMS
   - Same level as production data

2. Tamper-proof:
   - S3 Object Lock (WORM)
   - Versioning enabled
   - MFA Delete required

3. Geographic separation:
   - Cross-region backup
   - Different physical locations

4. Tested regularly:
   - Quarterly restore tests
   - Documented results
   - Updated procedures

5. Long-term retention:
   - 7+ years for HIPAA
   - Glacier Deep Archive
   - Cost-effective

6. Backup of backups:
   - AWS Backup service
   - Vault Lock for compliance"
```

### Compliance Questions

**Q13: "How do you handle a HIPAA audit?"**

```
Answer:
"Preparation done from day 1:

1. Documentation:
   - Architecture diagrams
   - Data flow diagrams
   - Security controls matrix
   - Risk assessment

2. Evidence ready:
   - CloudTrail logs
   - Access reviews
   - DR test results
   - Training records

3. Procedures documented:
   - Incident response
   - Breach notification
   - Access management
   - Change management

4. Technical controls verified:
   - Encryption everywhere
   - Audit logs working
   - Backups tested
   - Access controls enforced

5. BAA with AWS in place

Auditor can verify in hours, not days."
```

**Q14: "What if there's a breach?"**

```
Answer:
"Incident response plan:

1. Detection (immediate):
   - CloudWatch alarms
   - SNS notifications
   - Security team paged

2. Containment (within 1 hour):
   - Isolate affected systems
   - Revoke compromised credentials
   - Block source IPs

3. Investigation (within 24 hours):
   - Review CloudTrail logs
   - Analyze VPC Flow Logs
   - Identify scope of breach

4. Notification (within 60 days for HIPAA):
   - Affected individuals
   - HHS Office for Civil Rights
   - Media if >500 individuals

5. Remediation:
   - Fix vulnerability
   - Update controls
   - Document lessons learned

6. Post-mortem:
   - Update procedures
   - Train team
   - Strengthen architecture"
```

---

## 📚 Key Learnings (What to MEMORIZE)

### 🔴 MUST REMEMBER (Critical)

```
1. SSE-KMS with CMK = HIPAA encryption standard
   - SSE = Server-Side Encryption
   - KMS = Key Management Service
   - CMK = Customer Managed Key (YOU control)

2. CNAME cannot be at ROOT domain
   - Use Alias for AWS resources at root
   - Alias is FREE, CNAME costs queries

3. ACM cert for CloudFront MUST be in us-east-1
   - Regardless of CloudFront region
   - Common exam trap

4. IAM Roles for services, NOT access keys
   - EC2 → Role → AWS services
   - Temporary credentials, auto-rotate

5. 3-Tier VPC for HIPAA
   - Public (ALB), App (EC2), Data (RDS)
   - Network isolation
   - Defense in depth

6. CloudTrail Data Events = critical for HIPAA
   - Management events FREE (default)
   - Data events COST extra (must enable)
   - For S3 object access tracking

7. VPC Gateway Endpoint for S3 = FREE
   - For S3 and DynamoDB only
   - Stays in AWS network
   - HIPAA-friendly

8. Multi-AZ vs Multi-Region
   - Multi-AZ = HA within region
   - Multi-Region = DR across regions
   - Different problems, different solutions

9. RPO vs RTO
   - RPO = data loss (max acceptable)
   - RTO = downtime (max acceptable)
   - Drive DR strategy choice

10. Object Lock = tamper-proof storage
    - WORM (Write-Once-Read-Many)
    - For compliance logs
    - HIPAA requirement
```

### 🟡 GOOD TO REMEMBER (Important)

```
1. Multi-Region KMS keys
   - Same key ID across regions
   - For encrypted data portability
   - Simpler DR

2. WAF + Shield Standard
   - Shield Standard = FREE
   - Shield Advanced = $3,000/month
   - WAF = Layer 7 firewall

3. AWS SSO + Permission Sets
   - Federated identity
   - Centralized access management
   - Better than IAM users

4. Session Manager > Bastion
   - No SSH keys needed
   - All sessions logged
   - HIPAA-friendly

5. Secrets Manager > hard-coding
   - Auto-rotation
   - KMS encrypted
   - IAM controlled

6. Pilot Light DR strategy
   - Balance of cost and recovery
   - Good for most HIPAA apps
   - RPO/RTO in hours

7. Route 53 Health Checks for failover
   - Required for failover routing
   - 30s intervals (standard)
   - Trigger automatic DNS changes

8. CloudWatch Logs Insights
   - SQL-like queries on logs
   - Cross-source correlation
   - Fast audit responses

9. Encryption-related KMS Keys
   - One per data type (best practice)
   - Patient records vs logs vs backups
   - Easier audit, blast radius limited

10. Break-glass procedure
    - Emergency access
    - Hardware MFA
    - Full audit + review
```

### 🟢 NICE TO KNOW (Reference Only)

```
1. AWS Backup service for centralized backups
2. AWS Config for compliance monitoring
3. Security Hub for centralized findings
4. GuardDuty for threat detection
5. Macie for PII discovery
6. Athena for log analysis
7. Inspector for vulnerability scanning
8. WAF managed rule sets
9. Field-Level Encryption for sensitive form fields
10. Lambda@Edge for custom logic
```

---

## 💰 Cost Management (CRITICAL!)

### 🚨 RESOURCE DELETION CHECKLIST

```
After project complete, MUST DELETE:

1. EC2 Instances (Auto Scaling Group)
   - Delete ASG first
   - Then Launch Template
   
2. Load Balancer
   - Delete ALB
   - Delete Target Groups

3. NAT Gateway ($$$ - expensive!)
   - Delete NAT Gateway
   - Release Elastic IP
   
4. RDS Database
   - Take final snapshot (optional)
   - Delete database
   - Delete snapshots if not needed

5. ElastiCache
   - Delete cluster

6. CloudFront Distribution
   - Disable first (takes 15+ min)
   - Then delete

7. S3 Buckets
   - Empty buckets first
   - Then delete

8. VPC Endpoints
   - Delete (Interface endpoints cost $$)

9. VPC
   - Delete after subnets/SGs cleared

10. KMS Keys
    - Schedule deletion (7-30 days)
    - Keys cost $1/month each
    
11. CloudWatch Logs
    - Delete log groups
    - Or set retention to 1 day

12. ACM Certificates
    - Delete if not used
```

### Cost Tracking

```
Use AWS Cost Explorer:
- Set billing alarms
- Daily spend tracking
- Project tagging

For learning projects:
- Set budget: $20-30 max
- Daily alert at $5
- Auto-delete unused resources

Project 2 estimated cost:
- 1 day build: $5-10
- Resources deleted: $0 ongoing
- Total: Under $20
```

---

## 🛠️ Build Plan (Implementation Phases)

### Phase 1: Foundation (1 hour)
```
✅ Create VPC
✅ Create 6 subnets (2 AZs × 3 tiers)
✅ Create IGW + NAT Gateway
✅ Configure Route Tables
✅ Create 4 Security Groups
✅ Set up VPC Endpoints
```

### Phase 2: KMS + Storage (1 hour)
```
✅ Create KMS CMK
✅ Create S3 buckets (patient-records + logs)
✅ Configure bucket encryption (SSE-KMS)
✅ Set bucket policies (HTTPS enforcement)
✅ Enable versioning + Object Lock
```

### Phase 3: Data Layer (1 hour)
```
✅ Create RDS Multi-AZ (encrypted)
✅ Configure SSL enforcement
✅ Create ElastiCache (encrypted)
✅ Set up Secrets Manager
✅ Configure database backups
```

### Phase 4: Compute Layer (1 hour)
```
✅ Create IAM Role for EC2
✅ Create Launch Template
✅ Set up Auto Scaling Group
✅ Configure ALB (HTTPS)
✅ Set up Target Group
✅ Configure health checks
```

### Phase 5: Edge Layer (30 min)
```
✅ Request ACM cert (us-east-1!)
✅ Create CloudFront distribution
✅ Configure WAF
✅ Set up Route 53 (if using domain)
```

### Phase 6: Audit Layer (45 min)
```
✅ Enable CloudTrail (data events!)
✅ Enable VPC Flow Logs
✅ Enable S3 Access Logs
✅ Configure CloudWatch Logs
✅ Set up SNS for alerts
✅ Create metric filters + alarms
```

### Phase 7: Test Cases Validation (1 hour)
```
✅ Run all 20 test cases
✅ Document results
✅ Verify each control works
✅ Capture screenshots for portfolio
```

### Phase 8: Cleanup (15 min)
```
✅ Delete all resources (checklist above)
✅ Verify zero ongoing costs
✅ Document any leftover items
```

**Total Build Time:** 6-7 hours over 2 sessions

---

## 📊 Architecture Diagram (Final)

```
🌐 HIPAA-COMPLIANT HEALTHCARE PLATFORM

Internet
   ↓
Route 53 (Failover Routing)
   ↓
CloudFront + WAF + Shield Standard
   ├── ACM cert (us-east-1)
   ├── HTTPS only
   └── WAF rules
   ↓
═══════════════════════════════════════════════
║ PRIMARY VPC (us-east-1): 10.0.0.0/16        ║
║                                                ║
║ Public Subnets (2 AZs):                       ║
║ └── ALB (HTTPS)                               ║
║ └── NAT Gateway                               ║
║                                                ║
║ Private App Subnets (2 AZs):                  ║
║ └── EC2 Auto Scaling Group                    ║
║     ├── IAM Role (no keys)                    ║
║     └── EBS encrypted                         ║
║                                                ║
║ Private Data Subnets (2 AZs):                 ║
║ └── RDS Multi-AZ (KMS encrypted)              ║
║ └── ElastiCache (encrypted)                   ║
║                                                ║
║ VPC Endpoints:                                ║
║ └── S3 Gateway (FREE)                         ║
║ └── KMS, Secrets, CloudWatch Logs             ║
║ └── SSM endpoints                             ║
═══════════════════════════════════════════════
   ↓ Cross-Region Replication ↓
═══════════════════════════════════════════════
║ DR VPC (us-west-2)                            ║
║ - Pilot light (scaled-down)                   ║
║ - S3 CRR target                               ║
║ - RDS snapshot recovery ready                 ║
║ - Multi-Region KMS keys                       ║
═══════════════════════════════════════════════

Storage:
├── S3 (patient records, SSE-KMS)
├── S3 (logs buckets, separate)
└── Glacier (long-term archive)

Audit:
├── CloudTrail (all regions)
├── VPC Flow Logs
├── S3 Access Logs
├── RDS Audit Logs
└── CloudWatch Logs

Identity:
├── AWS SSO + Corporate AD
├── MFA mandatory
├── IAM Roles for services
└── Secrets Manager

Monitoring:
├── CloudWatch Alarms
├── SNS notifications
└── Real-time alerts
```

---

## 🎯 Interview Stories (Ready to Tell)

### Story 1: HIPAA Architecture Design

```
"Recently I designed a HIPAA-compliant healthcare 
platform on AWS for a 90-day compliance deadline.

The architecture used defense in depth:
- 3-tier VPC with network isolation
- SSE-KMS encryption with Customer Managed Keys
- AWS SSO with MFA for all users
- IAM Roles for services
- CloudTrail with data events for full audit
- Multi-region DR with Pilot Light strategy

Key decisions:
- Chose CMK over AWS Managed Keys for audit control
- Used VPC Endpoints to keep traffic in AWS network
- Implemented Session Manager instead of bastion
- Designed for 1-hour RPO, 4-hour RTO

Validated with 20 test cases covering encryption,
network isolation, audit capabilities, and DR.

Result: Audit-ready architecture documented
with runbooks and validation procedures."
```

### Story 2: Cost Optimization

```
"While building cloud projects, I focus on cost 
control:

For my HIPAA project:
- Used S3 Gateway Endpoint (free) instead of NAT
- Pilot Light DR instead of warm standby
- Right-sized instances (t3.micro for testing)
- Lifecycle policies for logs (hot → cold → archive)
- Deleted resources after testing

Result: Total project cost under $20 vs $200+
if naively built."
```

---

## 🌟 Personal Learning Log

```
What I learned this week:

Technical:
✅ SSE-KMS vs SSE-S3 (key control matters)
✅ Multi-Region KMS keys (DR enabler)
✅ VPC Endpoints (cost + security)
✅ Session Manager (modern access)
✅ CloudTrail Data Events (HIPAA critical)

Architectural:
✅ Defense in depth at every layer
✅ Trade-offs between cost and capability
✅ RPO/RTO drive DR strategy
✅ Documentation = compliance evidence
✅ Test cases = validation

Career:
✅ Real-world business context matters
✅ Justify every decision (interview gold)
✅ Senior engineers think holistically
✅ Documentation is part of engineering
✅ Cost awareness is professional

Soft Skills:
✅ Asking right questions about KMS terminology
✅ Pausing to absorb concepts
✅ Connecting theory to practice
✅ Thinking like an architect, not just user
```

---

## 🚀 Next Steps

```
✅ Document complete (this file!)
⏭️ BUILD PHASE: Start implementation
⏭️ Validate all 20 test cases
⏭️ Document build screenshots
⏭️ Tear down all resources
⏭️ Move to Project 3 (Black Friday scaling)
⏭️ Then Project 4 (Multi-region global)
⏭️ Continue SAA exam prep
```

---

*Part of my AWS Solutions Architect Associate journey toward Cloud Security Engineering. This document represents real architect-level design work — moving from theory consumption to engineering practice.*

**Project 2 Design: COMPLETE ✅
Ready to BUILD ✅
Interview-ready stories ✅
Test cases defined ✅
Cost-conscious approach ✅**
