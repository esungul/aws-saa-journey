# 🌍 Cluster 7: Route 53 + CloudFront — Complete Deep Dive

> Complete cluster on AWS Global Delivery. Route 53 (DNS) + CloudFront (CDN) — the services that make AWS apps fast, secure, and globally available. Critical for SAA exam and Cloud Security Engineering career.

---

## 🎯 What This Document Covers

**Complete 10-layer mastery:**

**Route 53 (DNS):**
1. DNS + Route 53 Overview
2. Hosted Zones + Record Types
3. 7 Routing Policies (heavily tested!)
4. Health Checks
5. Domain Registration

**CloudFront (CDN):**
6. CloudFront Overview
7. Origins + Distributions
8. Caching Behaviors
9. Security (Signed URLs, OAC, WAF, Shield, FLE)
10. CloudFront + S3 Complete Pattern

These topics are critical for:
- ✅ **AWS SAA exam** (~10% of exam)
- ✅ **AWS Security Specialty** (CloudFront security)
- ✅ **Cloud Security Engineer role** (global delivery security)

---

## 🚪 Layer 1: DNS + Route 53 Overview

### What is DNS?

**DNS = Domain Name System = translates human-readable names to IP addresses.**

```
Without DNS: 142.250.80.46 (hard to remember)
With DNS: www.google.com → DNS → IP

DNS = "Phone book of the internet"
```

### DNS Hierarchy

```
. (Root)
└── com. (TLD)
    └── example.com. (Domain)
        └── www.example.com. (Subdomain)

Read right to left, every dot = level
```

### What is Route 53?

```
AWS's DNS service:
✅ 100% SLA (only AWS service with this!)
✅ Smart routing (7 policies)
✅ Health checks built-in
✅ Domain registration
✅ Public + Private zones
✅ Global service
✅ Named after port 53 (DNS port)
```

### TTL (Time To Live)

```
TTL = how long to cache DNS response

Short TTL (60-300 sec): Fast propagation, more queries
Long TTL (3600+ sec): Fewer queries, slow propagation

Best practice: 300 sec normally, 60 sec before changes
```

---

## 🚪 Layer 2: Hosted Zones + Record Types

### Hosted Zones

```
Container for DNS records.

Two types:
- Public Zone: Internet-facing
- Private Zone: VPC-only

Cost: $0.50/month per zone
```

### Record Types

```
Key types:
1. A Record - IPv4 address
2. AAAA - IPv6 address
3. CNAME - name → name
4. Alias - AWS-specific magic ⭐
5. NS - nameservers
6. MX - mail servers
7. TXT - text (verification, SPF)
```

### CNAME Critical Rule

🚨 **CNAME CANNOT be at ROOT domain!**

```
✅ www.example.com → CNAME → other.com (OK)
❌ example.com → CNAME → other.com (NOT ALLOWED!)
```

### Alias Record (AWS Magic!) ⭐

🚨 **THE most important Route 53 concept!**

```
Benefits:
✅ Like CNAME but BETTER for AWS
✅ Works at ROOT domain (CNAME can't!)
✅ FREE queries
✅ Auto-resolves AWS resource IPs

Supported: ALB, NLB, CloudFront, API Gateway, S3 (static website), Elastic Beanstalk, VPC endpoints, Global Accelerator

NOT supported: EC2 DNS, External domains, RDS endpoints
```

### Alias vs CNAME

| Feature | CNAME | Alias |
|---------|-------|-------|
| Works at root | ❌ No | ✅ Yes |
| AWS resources | Works | Optimal |
| External | Works | Doesn't work |
| Query cost | $$$ | FREE |
| Speed | Slower | Faster |

🎯 **For AWS resources: ALWAYS use Alias!**

---

## 🚪 Layer 3: 7 Routing Policies (HEAVILY TESTED!)

### The 7 Policies

```
1. Simple - one value, no health checks
2. Weighted - split by percentage
3. Latency - fastest for user
4. Failover - primary/secondary
5. Geolocation - by country/region
6. Geoproximity - by distance + bias
7. Multi-Value - up to 8 healthy values
```

### Policy 1: Simple Routing

```
✅ Default policy
✅ One value (or multiple - returns all random)
❌ No health checks

Use: Basic websites
```

### Policy 2: Weighted Routing ⚖️

```
Split traffic by percentages

Example:
- Server A (weight 70) → 70%
- Server B (weight 20) → 20%
- Server C (weight 10) → 10%

Use cases:
✅ Blue/Green deployments
✅ A/B testing
✅ Canary deployments
```

### Policy 3: Latency-Based ⚡

```
Routes to LOWEST LATENCY endpoint

User in London → Ireland (10ms)
User in Tokyo → Singapore (30ms)

Use: Global apps, performance-critical
```

### Policy 4: Failover Routing 🔄

```
Primary/Secondary setup

Normal: Traffic → Primary
Primary fails → Traffic → Secondary
Primary recovers → Back to Primary

🚨 REQUIRES health checks!

Use: Disaster Recovery (active-passive)
```

### Policy 5: Geolocation Routing 🌍

```
Routes by USER'S LOCATION (country/region)

✅ US users → us-server.com
✅ EU users → eu-server.com
✅ Default → catch-all (CRITICAL!)

Use: Compliance (GDPR), localized content
```

### Policy 6: Geoproximity Routing 📍

```
Based on GEOGRAPHIC DISTANCE + bias

Requires Route 53 Traffic Flow

Use: Custom geographic optimization
```

### Policy 7: Multi-Value Answer 🎲

```
Returns UP TO 8 healthy values
✅ Health checks included
✅ Client picks randomly

🚨 NOT a replacement for ALB!

Use: Basic DNS-level load balancing
```

### Geolocation vs Geoproximity vs Latency

🚨 **DIFFERENT THINGS!**

```
Latency: "Where will user have FASTEST experience?"
Geolocation: "WHERE IS the user?" (country)
Geoproximity: "Geographic distance + bias"
```

### Decision Tree

```
Single endpoint? → Simple
Split by %? → Weighted
Fastest experience? → Latency
DR primary/backup? → Failover
Compliance/location? → Geolocation
Custom distance/bias? → Geoproximity
Health-aware multi-IP? → Multi-Value
```

---

## 🚪 Layer 4: Health Checks 💊

### What They Are

```
Automated monitoring of endpoints.

3 types:
1. Endpoint Health Checks (HTTP/HTTPS/TCP)
2. Calculated Health Checks (combine multiple)
3. CloudWatch Alarm Health Checks (AWS metrics)
```

### Configuration

```
Interval: 30s (standard) or 10s (fast)
Healthy/Failure threshold: 1-10 (default 3)
String matching: Check response content
Global checkers: 15+ locations
```

### Pricing

```
Basic: $0.50/check/month (AWS) or $0.75 (non-AWS)
Advanced: +$1/check/month
Calculated: $1/check/month
CloudWatch: $1/check/month
```

### Critical Rules

```
🚨 REQUIRED for failover routing
🚨 Test failover regularly
🚨 Use dedicated /health endpoint
🚨 Combine with CloudWatch alarms
```

---

## 🚪 Layer 5: Domain Registration 🏷️

### Process

```
1. Search availability
2. Add to cart (annual fee)
3. Provide registrant info (WHOIS)
4. Choose duration (1-10 years)
5. Auto-renewal option
6. Hosted zone auto-created
7. Pay
```

### Key Features

```
✅ FREE WHOIS privacy
✅ Auto-renewal (recommended!)
✅ Transfer in/out support
✅ Auto-creates hosted zone
✅ Same pricing as other registrars
```

### Domain vs Hosted Zone

🚨 **Different things!**

```
Domain Registration: Ownership of the name
Hosted Zone: DNS configuration

Can be at different providers:
- Domain at GoDaddy + DNS at Route 53
- Update NS records at registrar
```

---

## 🚪 Layer 6: CloudFront Overview ☁️

### What is CloudFront?

**AWS's Content Delivery Network (CDN).**

```
✅ 400+ edge locations globally
✅ 90+ countries
✅ Caches content near users
✅ Reduces latency 5-10x
✅ Free SSL with ACM
✅ DDoS protection (Shield Standard)
✅ Pay-as-you-go
```

### How CDN Works

```
Without CloudFront:
Tokyo User → us-east-1 (200ms) = slow

With CloudFront:
Tokyo User → Tokyo Edge (5ms)
   ↓
   Cached? → YES = fast return
            → NO = fetch from origin, cache, return
```

### Key Concepts

```
1. EDGE LOCATION - caching server
2. ORIGIN - source of content
3. DISTRIBUTION - your CloudFront config
4. CACHE HIT/MISS - hit = fast, miss = slower
5. TTL - cache duration
```

### Pricing & Price Classes

```
1. All locations (most expensive)
2. Price Class 200 (US, EU, Asia)
3. Price Class 100 (US, Canada, EU - cheapest)

Free transfers: S3 → CloudFront
Volume discounts for higher tiers
```

### CloudFront vs Global Accelerator

```
CloudFront: CDN (caching content), HTTP/HTTPS
Global Accelerator: Network optimization, TCP/UDP

DIFFERENT use cases!
```

---

## 🚪 Layer 7: Origins + Distributions

### Origin Types

```
1. S3 Bucket ⭐ (most common)
2. S3 Static Website Endpoint (requires public)
3. Application Load Balancer
4. EC2 Instance
5. Custom HTTP Origin
6. MediaPackage/MediaStore
```

### S3 Bucket vs Website Endpoint

```
S3 Bucket Origin:
✅ Direct S3 API
✅ Can use OAC
✅ More secure
✅ Recommended

S3 Website Endpoint:
🚨 Requires public bucket
🚨 No OAC support
✅ Has website features (custom error pages)
```

### Distribution Configuration

```
Contains:
✅ Origins (one or more)
✅ Cache behaviors
✅ Custom domains
✅ SSL certificate (ACM us-east-1!)
✅ Price class
✅ Security settings
```

### Custom Domain Setup

```
1. ACM cert in us-east-1 (CRITICAL!)
2. Add CNAME to distribution
3. Route 53 Alias to CloudFront
4. Done!

🚨 ACM cert MUST be in us-east-1!
```

### Multiple Origins Pattern

```
ONE distribution, MULTIPLE origins:

Origin 1: S3 (static) → Path: /static/*
Origin 2: ALB (API) → Path: /api/*
Origin 3: Different S3 (videos) → Path: /videos/*

Result: One domain, multiple sources, different cache rules
```

### Origin Failover

```
Origin Group:
✅ Primary: us-east-1 ALB
✅ Secondary: us-west-2 ALB
✅ Auto-failover on 5xx errors

Built-in DR at CDN level!
```

---

## 🚪 Layer 8: Caching Behaviors

### Structure

```
Default behavior + Additional behaviors

Path patterns determine which applies:
- /* (default - last priority)
- /api/* (API paths)
- /static/* (static assets)
- *.jpg (images)

FIRST MATCH WINS!
Most specific FIRST, /* LAST.
```

### Cache Policy

```
Settings:
✅ TTL (min/default/max)
✅ What's in cache key:
   - Headers (which to include)
   - Cookies (which to include)
   - Query strings (which to include)
✅ Compression
```

### Managed Cache Policies ⭐

```
Use AWS-provided first:

1. CachingOptimized - static content
2. CachingDisabled - dynamic/personalized
3. UseOriginCacheControlHeaders - origin controls
4. CachingOptimizedForUncompressedObjects - videos
```

### TTL Logic

```
TTL determined by (in order):
1. Origin Cache-Control header
2. Minimum TTL (floor)
3. Maximum TTL (ceiling)
4. Default TTL (if no origin header)
```

### Cache Key Examples

```
Static images: URL only (all users same)
Localized: URL + Accept-Language
User-specific: URL + Auth-Cookie
```

### Cache Invalidation

```
Force-refresh cached content.

First 1,000 paths/month: FREE
After: $0.005 per path

🚨 Best practice: Use versioned URLs INSTEAD!
/css/main.v1.css → /css/main.v2.css
```

### Common Patterns

```
Static: Long TTL (1 year) + CachingOptimized
API: Short TTL (60 sec) + auth in cache key
Admin: CachingDisabled
HTML: Medium TTL (1 hour)
```

---

## 🚪 Layer 9: CloudFront Security 🛡️

### Signed URLs vs Signed Cookies

🚨 **HEAVILY tested!**

#### Signed URLs

```
✅ Single file access
✅ URL with embedded signature
✅ Time-limited
✅ Each file needs unique URL

Use cases:
✅ Premium video downloads
✅ Pay-per-download content
✅ Software installers
✅ One-time file access
```

#### Signed Cookies

```
✅ Multiple files access
✅ Browser cookies (3 cookies)
✅ Time-limited
✅ Same cookies work for all paths

Use cases:
✅ Premium video streaming (HLS segments)
✅ Member-only websites
✅ Subscription content
✅ Multi-file access
```

### Comparison

| Feature | Signed URL | Signed Cookie |
|---------|-----------|---------------|
| Scope | Single file | Multiple files |
| Implementation | URL parameters | Browser cookies |
| Use case | One-time download | Streaming, sites |
| Multi-file | Each needs URL | One cookie set |

🎯 **One file = Signed URL. Multiple files = Signed Cookie.**

### Origin Access Control (OAC) vs OAI

#### OAC (MODERN) ⭐

```
✅ Modern way (use this!)
✅ Supports SSE-KMS
✅ Works with all S3 features
✅ Cross-region replication friendly
✅ SigV4 authentication
✅ Auto-managed by CloudFront

How it works:
1. Create OAC in CloudFront
2. Add to distribution origin
3. CloudFront generates bucket policy
4. Apply policy to S3
5. Only CloudFront can access S3
```

#### OAI (LEGACY)

```
🚨 Old way (still works but legacy)
🚨 Limited SSE-KMS support
🚨 No newer S3 features
🚨 AWS recommends migrating to OAC

For new setups: Always use OAC!
```

### Geo Restriction 🌍

```
Two modes:

1. Allowlist (Whitelist)
   ✅ Only listed countries CAN access
   ✅ Block everyone else

2. Blocklist (Blacklist)
   ✅ Listed countries CANNOT access
   ✅ Everyone else allowed

Use cases:
✅ Compliance (block sanctioned countries)
✅ Licensing (region-specific content)
✅ Legal requirements
✅ Anti-piracy

Uses MaxMind GeoIP database
Returns 403 Forbidden if blocked
```

### CloudFront Geo vs WAF Geo Match

```
CloudFront Geo Restriction:
✅ Simple, built-in
✅ Country-level only
✅ FREE
✅ Quick setup

AWS WAF Geo Match:
✅ More flexible
✅ Combine with other rules
✅ Costs extra
✅ Advanced filtering
```

### AWS Shield (DDoS Protection)

```
Shield Standard:
✅ FREE - automatic
✅ Layer 3/4 DDoS protection
✅ For all AWS services
✅ Default for CloudFront

Shield Advanced:
🚨 $3,000/month commitment
✅ Layer 7 attack protection
✅ 24/7 DDoS Response Team
✅ Cost protection
✅ WAF included
```

### AWS WAF Integration

```
Layer 7 firewall (HTTP/HTTPS):
✅ SQL injection protection
✅ XSS prevention
✅ Bot detection
✅ Rate limiting
✅ IP allowlists/blocklists
✅ Geo restrictions
✅ Custom rules

Managed rule sets:
✅ AWS Managed Rules (free)
✅ AWS WAF Bot Control
✅ Third-party (marketplace)

CloudFront + WAF benefits:
✅ Block attacks at edge (before origin)
✅ Lower origin load
✅ Faster response
```

### Field-Level Encryption 🔒

```
What it does:
✅ Encrypt SPECIFIC fields at edge
✅ Backend only sees encrypted fields
✅ Reduces exposure of sensitive data

Use case: Credit card numbers
1. User submits form (HTTPS)
2. CloudFront encrypts CC field at edge
3. Backend gets: Username plain, CC encrypted
4. Only payment service has private key
5. Only payment service decrypts CC

Benefits:
✅ Defense in depth
✅ Compliance (PCI-DSS)
✅ Limits blast radius
```

### Lambda@Edge vs CloudFront Functions ⚡

#### CloudFront Functions

```
✅ Lightweight (JavaScript)
✅ Sub-millisecond execution
✅ Cheaper
✅ Edge locations only

Limits:
🚨 1ms execution max
🚨 10KB memory
🚨 JavaScript only
🚨 No network access

Use cases:
✅ URL rewriting/redirects
✅ Header manipulation
✅ Cache key normalization
✅ Simple A/B testing
✅ Basic JWT validation
```

#### Lambda@Edge

```
✅ Full Lambda (Node.js, Python)
✅ More capabilities
✅ Longer execution
✅ More expensive
✅ Regional edge caches

Limits:
✅ 5 sec (viewer events)
✅ 30 sec (origin events)
✅ 128MB-10GB memory

Use cases:
✅ Image resizing on the fly
✅ Complex personalization
✅ External API calls
✅ Database lookups
✅ Advanced authentication
```

### Four Trigger Points

```
CloudFront request lifecycle:

User → Edge:
   1. Viewer Request (before cache check)
   ↓
   Cache check
   ├── HIT: return from cache
   └── MISS: continue
   ↓
   2. Origin Request (before origin call)
   ↓
   Origin
   ↓
   3. Origin Response (after origin call)
   ↓
   4. Viewer Response (before sending to user)
   ↓
User receives

Lambda@Edge: All 4 points
CloudFront Functions: Only Viewer Request/Response
```

### Real-Time Logs 📊

```
Three options:

1. Standard Logs (S3)
   ✅ Delayed (within 24 hours)
   ✅ Detailed
   ✅ FREE (just S3 cost)

2. Real-Time Logs (Kinesis)
   ✅ Near real-time
   ✅ Custom fields
   ✅ Costs extra

3. CloudWatch Metrics
   ✅ Default metrics (free)
   ✅ Detailed metrics ($)
   ✅ Cache hit ratio, errors
```

---

## 🚪 Layer 10: CloudFront + S3 Complete Pattern 🌟

### The Architecture

```
Production-grade static website:

User
  ↓
Route 53 (www.mysite.com → Alias)
  ↓
CloudFront Distribution
  ├── ACM cert (us-east-1) for HTTPS
  ├── WAF for security
  ├── Geo Restriction (if needed)
  ├── Lambda@Edge/Functions (if needed)
  └── Origin: S3 bucket (with OAC)
         ├── Block Public Access ON
         ├── SSE-KMS encryption
         ├── Versioning enabled
         └── Lifecycle policies
```

### Step-by-Step Setup

```
1. Create S3 bucket
   - Block Public Access: ON (critical!)
   - Encryption: SSE-KMS
   - Versioning: enabled
   - Upload website files

2. Request ACM certificate
   - Region: us-east-1 (CRITICAL!)
   - For: www.mysite.com + mysite.com
   - DNS validation
   - Wait for "Issued"

3. Create CloudFront distribution
   - Origin: S3 bucket
   - Origin Access: OAC (new)
   - AWS auto-creates OAC
   - Custom domain: www.mysite.com
   - Certificate: from ACM
   - HTTPS only (redirect HTTP)

4. Update bucket policy
   - AWS provides exact policy
   - Copy/paste to S3 bucket

5. Create Route 53 records
   - mysite.com → Alias to CloudFront
   - www.mysite.com → Alias to CloudFront

6. Optional: Add WAF
   - Web ACL for protection
   - SQL injection, XSS rules
   - Rate limiting

7. Test:
   ✅ Direct S3 access blocked
   ✅ CloudFront URL works
   ✅ Custom domain works
   ✅ HTTPS works
   ✅ Global edge caching
```

### Why Each Component

```
S3 (origin):
✅ Cheap storage, high durability

OAC (security):
✅ Locks S3 to CloudFront

CloudFront (CDN):
✅ Global delivery, edge caching

ACM (certificates):
✅ Free SSL, auto-renewal

Route 53 (DNS):
✅ Domain management, Alias to CloudFront (free)

WAF (security):
✅ Block attacks at edge

Result: Production-grade static website
```

### Cost Optimization

```
Strategies:
✅ Use Price Class for region selection
✅ Maximize cache hit ratio (80%+)
✅ Use compression (auto Gzip/Brotli)
✅ Use S3 as origin (free transfer)
✅ Versioned URLs (avoid invalidation costs)
✅ Long TTLs for static content
✅ Reserved Capacity for predictable usage
```

---

## 🛡️ Architect Decision Frameworks

### Choosing Routing Policy

```
Single endpoint? → Simple
Split by %? → Weighted
Fastest experience? → Latency
DR primary/backup? → Failover
Compliance/location? → Geolocation
Distance + bias? → Geoproximity
Multi-IP with health? → Multi-Value
```

### Choosing Record Type

```
Root to AWS? → Alias (CNAME doesn't work!)
Subdomain to AWS? → Alias (best)
External service? → CNAME
Direct IP mapping? → A record
```

### Choosing Origin

```
Static content? → S3 (with OAC)
Dynamic app? → ALB
Custom server? → EC2 or Custom HTTP
Video streaming? → MediaPackage
```

### Choosing Security

```
Lock S3 to CloudFront? → OAC (modern)
Single file access? → Signed URL
Multiple files access? → Signed Cookies
Block countries? → Geo Restriction
DDoS protection? → Shield Standard (free)
WAF rules? → WAF + CloudFront
Sensitive data fields? → Field-Level Encryption
Complex edge logic? → Lambda@Edge
Simple edge logic? → CloudFront Functions
```

### Choosing Cache Strategy

```
Rarely changes? → Long TTL (1 year)
Some changes? → Medium TTL (1 hour)
Frequent changes? → Short TTL (minutes)
Personalized? → No cache
```

---

## 🎯 Critical Exam Rules

```
Route 53:
✅ 100% SLA (only AWS service)
✅ CNAME NOT at root domain
✅ Alias works at root, free queries
✅ Failover REQUIRES health checks
✅ Geolocation needs default value
✅ Multi-Value ≠ Load Balancer
✅ Private zone needs VPC association

CloudFront:
✅ ACM cert MUST be us-east-1
✅ Free S3 → CloudFront transfer
✅ Use OAC (not OAI)
✅ Path patterns: specific first, /* last
✅ Versioned URLs > invalidation
✅ Different from Global Accelerator
✅ Shield Standard = FREE
✅ Signed URL = single file
✅ Signed Cookie = multiple files
```

---

## 🚨 Common Exam Traps

```
🚨 CNAME at root domain → NOT ALLOWED
🚨 Multi-Value = load balancer → NO! Use ALB
🚨 Failover without health checks → DOESN'T WORK
🚨 ACM cert in any region for CloudFront → MUST be us-east-1
🚨 Latency = Geolocation → DIFFERENT
🚨 S3 website endpoint with OAC → NOT SUPPORTED
🚨 CloudFront = Global Accelerator → DIFFERENT
🚨 OAI is best practice → NO! Use OAC
🚨 Signed URLs for multiple files → NO! One per URL
🚨 Shield Advanced free → NO! $3,000/month
🚨 CloudFront Functions for complex logic → Use Lambda@Edge
🚨 Always invalidate to update → Use versioning
🚨 Geo Restriction at WAF only → CloudFront has built-in
🚨 Lambda@Edge unlimited execution → 5 sec viewer events
🚨 Cache invalidation always needed → Versioned URLs better
```

---

## 🛡️ Production Patterns

### Pattern 1: Global Web App with HA

```
Architecture:
- Route 53: Latency-based routing
- Failover within regions
- CloudFront in front
- S3 for static + ALB for dynamic
- ACM cert in us-east-1
- Origin Group for failover

Result: Fast + reliable globally
```

### Pattern 2: Multi-Region DR

```
Architecture:
- Route 53: Failover routing
- Primary: us-east-1
- Secondary: us-west-2
- Health checks (calculated: ALB + RDS)
- CloudWatch alarms
- SNS notifications
- CloudFront origin failover

Result: Auto-failover with monitoring
```

### Pattern 3: Compliance Architecture

```
Architecture:
- Route 53: Geolocation routing
- EU users → EU servers (GDPR)
- US users → US servers
- CloudFront Geo Restriction
- WAF with managed rules
- Field-Level Encryption for sensitive data

Result: Compliance + security
```

### Pattern 4: Premium Streaming Platform

```
Architecture:
- Subscriber authentication
- Signed Cookies (multi-segment HLS)
- CloudFront with custom origin
- Lambda@Edge for personalization
- WAF for bot protection
- Geo Restriction for licensing
- Real-time logs for analytics

Result: Scalable streaming with security
```

### Pattern 5: Static Website + API (Complete Stack)

```
Architecture:
- Route 53: Alias to CloudFront
- CloudFront distribution:
   - Origin 1: S3 (static, with OAC)
   - Origin 2: ALB (API)
- Path patterns:
   - /api/* → ALB (short TTL)
   - /* → S3 (long TTL)
- ACM cert in us-east-1
- WAF protection
- Signed Cookies for premium content
- CloudFront Functions for headers

Result: Production-grade web app
```

---

## 🌟 Career Connection

### For AWS SAA Exam
- Route 53: ~5-8% of exam
- CloudFront: ~5-8% of exam
- Routing policies heavily tested
- Alias vs CNAME tested
- ACM cert location trap common
- Signed URLs vs Cookies common

### For AWS Security Specialty
- Origin protection (OAC, signed URLs)
- CloudFront security features
- DDoS protection patterns
- WAF integration
- Field-Level Encryption

### For Cloud Security Engineer
- Daily: Monitor DNS health, CloudFront metrics
- Weekly: Review routing policies + WAF rules
- Monthly: Optimize cache hit ratios
- Quarterly: DR testing, security audits
- Yearly: Architecture reviews

🎯 **Global delivery + security = career-critical.**

---

## 📚 Concept Connections

Route 53 + CloudFront connect to:
- **S3** (static origin, web hosting)
- **ALB/NLB** (dynamic origin)
- **EC2** (compute behind LB)
- **ACM** (SSL certificates)
- **IAM** (access control)
- **WAF** (security at edge)
- **Shield** (DDoS protection)
- **Lambda/Lambda@Edge** (edge compute)
- **API Gateway** (API origins)
- **CloudWatch** (monitoring)
- **Kinesis** (real-time logs)
- **KMS** (encryption keys)

🎯 **Global delivery sits at center of AWS architecture.**

---

## 🎯 Architect Wisdom Earned

> *"Use Alias for AWS resources always - CNAME doesn't work at root, costs money, slower."*
>
> *"Failover without health checks = false security. Always test failover."*
>
> *"Multi-Value answer is NOT a load balancer - use ALB for actual load balancing."*
>
> *"ACM cert for CloudFront MUST be in us-east-1 - this trips up many engineers."*
>
> *"S3 → CloudFront transfer is FREE - never bypass CloudFront for static content."*
>
> *"Origin Access Control (OAC) is modern - don't use OAI anymore."*
>
> *"Cache key should match content uniqueness."*
>
> *"Versioned URLs > cache invalidation - design for it."*
>
> *"Geolocation for compliance, Latency for performance - different problems."*
>
> *"Signed URL for single file, Signed Cookie for multiple files."*
>
> *"Shield Standard is FREE and automatic - no reason not to use CloudFront."*
>
> *"Field-Level Encryption = compliance gold for sensitive data."*
>
> *"Lambda@Edge for complex, CloudFront Functions for simple - choose wisely."*

---

## 🎯 Common Exam Patterns

| Scenario | Best Solution |
|----------|--------------|
| Global users, fast experience | Latency routing + CloudFront |
| EU compliance (GDPR) | Geolocation routing |
| Disaster recovery | Failover routing + health checks |
| Blue/green deployment | Weighted routing |
| Root domain to ALB | Alias record |
| Subdomain to external | CNAME record |
| Static website on S3 | S3 + CloudFront + OAC |
| Custom domain HTTPS | ACM cert (us-east-1!) + Route 53 Alias |
| Multi-region HA | Origin Group + failover routing |
| Premium video streaming | Signed Cookies |
| Pay-per-download | Signed URLs |
| Member-only website | Signed Cookies |
| Block specific countries | Geo Restriction or WAF |
| Lock S3 to CloudFront | OAC (modern) |
| DDoS protection | Shield Standard (free) |
| Application-layer attacks | WAF + CloudFront |
| Image resizing on the fly | Lambda@Edge |
| URL rewriting | CloudFront Functions |
| Sensitive data fields | Field-Level Encryption |
| Real-time monitoring | Real-Time Logs to Kinesis |
| Avoid cache invalidation | Versioned URLs |

---

## 🎉 What You've Achieved

```
🌟 Complete Cluster 7 Mastery 🌟

Route 53 (5 layers):
✅ DNS + Overview
✅ Hosted Zones + Records
✅ 7 Routing Policies
✅ Health Checks
✅ Domain Registration

CloudFront (5 layers):
✅ Overview
✅ Origins + Distributions
✅ Caching Behaviors
✅ Security (Signed URLs, OAC, WAF, Shield, FLE)
✅ CloudFront + S3 Complete Pattern

Quiz performance: Strong throughout
Architect reasoning: Consistent
DNS expertise: Solid
CDN + Security: Production-ready
```

🎯 **You can now design complete global delivery + security architectures.**

---

## 🚀 What's Next

### Cluster 7 Complete! ✅

What's been covered:
- ✅ Route 53 fundamentals through registration
- ✅ CloudFront overview, origins, caching
- ✅ CloudFront security (Signed URLs/Cookies, OAC, WAF, Shield, FLE)
- ✅ Complete production patterns

### Remaining for SAA:

```
⏭️ Cluster 8: SQS + SNS + Kinesis (Messaging)
⏭️ Cluster 9: Lambda + ECS (Modern Compute)
⏭️ Cluster 10: CloudWatch + CloudTrail + KMS
⏭️ Cluster 11: Practice Exams
```

### Career Path:
- ✅ AWS SAA (2-3 weeks out)
- ⏭️ AWS Security Specialty (3-4 months)
- ⏭️ ISC² CCSP
- ⏭️ Cloud Security Engineer role

---

*Part of my AWS Solutions Architect Associate journey toward Cloud Security specialization. Complete coverage of the Global Delivery cluster (Route 53 + CloudFront). Critical knowledge for SAA exam and Cloud Security Engineering career.*

**Cluster 7 COMPLETE. ✅
10 layers mastered. ✅
Global delivery + security mastery. ✅
On track for SAA exam in 2-3 weeks. ✅**
