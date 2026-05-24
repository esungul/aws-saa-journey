# 🌍 Cluster 7: Route 53 + CloudFront — Day 1

> Day 1 of the Global Delivery cluster. Covers DNS (Route 53) and CDN (CloudFront) — the services that make AWS apps fast and globally available. Critical knowledge for SAA exam and Cloud Security Engineering career.

---

## 🎯 What This Document Covers

8 layers of global delivery mastery:

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

Day 2 will cover CloudFront security + S3 integration patterns.

These topics are critical for:
- ✅ **AWS SAA exam** (~10% of exam)
- ✅ **AWS Security Specialty** (CloudFront security)
- ✅ **Cloud Security Engineer role** (global delivery security)

---

## 🚪 Layer 1: DNS + Route 53 Overview

### What is DNS?

**DNS = Domain Name System = translates human-readable names to IP addresses.**

```
Without DNS: Users type 142.250.80.46
With DNS: Users type www.google.com → DNS → IP

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

### DNS Resolution Process

```
User types: www.example.com

1. Browser cache check
2. OS cache check
3. Local DNS Resolver (ISP/Google 8.8.8.8)
4. Root server query
5. TLD server query (.com)
6. Authoritative nameserver
7. Resolver caches result (TTL)
8. Returns IP to browser

All in milliseconds!
```

### TTL (Time To Live)

```
TTL = how long to cache DNS response

Short TTL (60-300 sec):
✅ Changes propagate fast
🚨 More queries

Long TTL (3600+ sec):
✅ Fewer queries
🚨 Slow to propagate

Best practice: 300 sec normally, 60 sec before changes
```

### What is Route 53?

**Route 53 = AWS's DNS service.**

```
✅ Highly available (100% SLA - only AWS service!)
✅ Scalable (billions of queries)
✅ Global service
✅ Smart routing (7 policies)
✅ Health checks built-in
✅ Domain registration
✅ Public + Private zones
✅ AWS integration
```

🎯 **Named after port 53 (DNS port).**

### Use Cases

```
1. Simple Website Hosting
2. Multi-Region Applications
3. Disaster Recovery
4. Blue/Green Deployments
5. Geo-Restriction (compliance)
6. Load Distribution
7. Hybrid Cloud (private zones)
```

### Memory Hook

```
Route 53 🌐
✅ AWS DNS service
✅ 100% SLA (only AWS service!)
✅ Smart routing (7 policies)
✅ Health checks + failover
✅ Domain registration
✅ Global + integrated
```

---

## 🚪 Layer 2: Hosted Zones + Record Types

### Hosted Zones

**Container for DNS records for a domain.**

```
Two types:

Public Hosted Zone:
✅ Internet-facing DNS
✅ For public websites
✅ Anyone can query

Private Hosted Zone:
✅ Internal to VPCs
✅ For internal services
✅ Only associated VPCs can query

Cost: $0.50/month per zone
```

### Record Types

```
1. A Record - IPv4 address ⭐
2. AAAA Record - IPv6 address
3. CNAME Record - name → name ⭐
4. Alias Record - AWS-specific magic ⭐
5. NS Record - nameservers (auto)
6. SOA Record - start of authority (auto)
7. MX Record - mail servers
8. TXT Record - text (verification, SPF, DKIM)
```

### A Record

```
Maps hostname → IPv4 address

Example:
Name: www.example.com
Type: A
Value: 192.0.2.1
TTL: 300
```

### CNAME Record

```
Points one name to ANOTHER name (not IP!)

Example:
Name: www.example.com
Type: CNAME
Value: example.com

🚨 CRITICAL RULE:
CANNOT use CNAME at ROOT domain!
✅ www.example.com → CNAME → other.com (OK)
❌ example.com → CNAME → other.com (NOT ALLOWED!)
```

### Alias Record (AWS Magic!) ⭐

🚨 **THE most important Route 53 concept!**

```
Alias = Route 53 special record

Benefits:
✅ Like CNAME but BETTER for AWS
✅ Works at ROOT domain! (CNAME can't!)
✅ FREE queries (no cost)
✅ AWS auto-resolves
✅ No extra DNS lookup

Supported targets:
✅ ALB, NLB
✅ CloudFront
✅ API Gateway
✅ S3 (static website)
✅ Elastic Beanstalk
✅ VPC endpoints
✅ Global Accelerator

🚨 NOT supported (use CNAME):
❌ EC2 DNS name
❌ External domains
❌ RDS endpoints
```

### Alias vs CNAME (CRITICAL!)

| Feature | CNAME | Alias |
|---------|-------|-------|
| **Works at root** | ❌ No | ✅ Yes |
| **AWS resources** | Works | Optimal |
| **External** | Works | Doesn't work |
| **Query cost** | $$$ | FREE |
| **Speed** | Slower | Faster |

🎯 **For AWS resources: ALWAYS use Alias!**

### Memory Hook

```
Hosted Zones + Records 📋
✅ Public Zone = internet
✅ Private Zone = VPC-only
✅ A = IPv4 mapping
✅ CNAME = name → name (NOT at root!)
✅ Alias = AWS magic (works at root!) ⭐
✅ Use Alias for AWS resources always
```

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
✅ One value (or multiple - returns all)
❌ No health checks
❌ No intelligence

Use: Basic websites, single endpoint
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
✅ Load distribution
```

### Policy 3: Latency-Based ⚡

```
Routes to LOWEST LATENCY endpoint

User in London → Ireland server (10ms)
User in Tokyo → Singapore server (30ms)

Use cases:
✅ Global apps
✅ Performance-critical apps
✅ Multi-region deployments
```

### Policy 4: Failover Routing 🔄

```
Primary/Secondary setup

Normal: Traffic → Primary
Primary fails → Traffic → Secondary
Primary recovers → Back to Primary

🚨 REQUIRES health checks!

Use cases:
✅ Disaster Recovery (active-passive)
✅ High availability
```

### Policy 5: Geolocation Routing 🌍

```
Routes by USER'S LOCATION (country/region)

✅ US users → us-server.com
✅ EU users → eu-server.com
✅ Asia users → asia-server.com
✅ Default → catch-all (CRITICAL!)

Use cases:
✅ Compliance (GDPR for EU)
✅ Localized content
✅ Geographic licensing
✅ Geo-restrictions
```

### Policy 6: Geoproximity Routing 📍

```
Based on GEOGRAPHIC DISTANCE + bias

User location + Resource location = distance
Bias adjusts coverage areas

Requires Route 53 Traffic Flow

Use cases:
✅ Custom geographic optimization
✅ Traffic shifting with bias
```

### Policy 7: Multi-Value Answer 🎲

```
Returns UP TO 8 healthy values
✅ Health checks included
✅ Client picks randomly

🚨 NOT a replacement for ALB!

Use cases:
✅ Basic DNS-level load balancing
✅ Health-aware routing
✅ When ALB is overkill
```

### Geolocation vs Geoproximity vs Latency

🚨 **DIFFERENT THINGS!**

```
Latency-based:
- "Where will user have FASTEST experience?"
- Performance-focused

Geolocation:
- "WHERE IS the user?" (country)
- Compliance-focused

Geoproximity:
- "Geographic distance + bias"
- Custom optimization
```

### Decision Tree

```
What's your goal?
├── Single endpoint? → Simple
├── Split by %? → Weighted
├── Fastest experience? → Latency
├── DR primary/backup? → Failover
├── Compliance/location? → Geolocation
├── Custom distance/bias? → Geoproximity
└── Health-aware multi-IP? → Multi-Value
```

### Common Combined Patterns

```
Global app with DR:
- Latency-based for performance
- Failover within each region

Compliance + performance:
- Geolocation for compliance
- Each region: ALB with multi-AZ

Blue/Green deployment:
- Weighted (95% old, 5% new)
- Gradually shift weights
```

### Memory Hook

```
7 Routing Policies 🎯
1. SIMPLE (default, no health)
2. WEIGHTED (split by %) - blue/green, A/B
3. LATENCY (fastest) - performance
4. FAILOVER (primary/secondary) - DR
5. GEOLOCATION (by country) - compliance
6. GEOPROXIMITY (distance + bias) - custom
7. MULTI-VALUE (up to 8 healthy) - basic LB

All except Simple support health checks!
Failover REQUIRES health checks!
Geolocation needs default!
```

---

## 🚪 Layer 4: Health Checks 💊

### What Are Health Checks?

**Automated monitoring of endpoints for routing decisions.**

```
How it works:
1. Route 53 sends request every 30 sec (or 10s fast)
2. Endpoint responds (or doesn't)
3. After 3 successful → Healthy
4. After 3 failures → Unhealthy
5. Routing decisions based on health
```

### 3 Types of Health Checks

```
1. Endpoint Health Checks ⭐
   ✅ Monitor specific URL/IP
   ✅ HTTP, HTTPS, TCP
   ✅ Most common

2. Calculated Health Checks
   ✅ Combine multiple checks
   ✅ AND, OR, NOT logic
   ✅ Up to 256 child checks

3. CloudWatch Alarm Health Checks
   ✅ Based on AWS metrics
   ✅ CPU, errors, latency
   ✅ AWS integration
```

### Settings

```
Interval:
✅ Standard: 30 seconds
✅ Fast: 10 seconds (more expensive)

Thresholds:
✅ Healthy threshold: 1-10 (default 3)
✅ Failure threshold: 1-10 (default 3)

String Matching:
✅ Check for specific text in response
✅ Beyond just HTTP 200
```

### Pricing

```
Basic endpoint: $0.50/check/month (AWS) or $0.75 (non-AWS)
Advanced features: +$1/check/month
Calculated: $1/check/month
CloudWatch: $1/check/month + CloudWatch costs

Affordable for HA!
```

### Use Cases

```
✅ Web server availability (endpoint check)
✅ Multi-tier app health (calculated)
✅ AWS metric-based (CloudWatch alarm)
✅ Multi-region DR (combined)
```

### Critical Rules

```
🚨 REQUIRED for failover routing
🚨 Test failover regularly
🚨 Use dedicated /health endpoint
🚨 Combine with CloudWatch alarms
🚨 Multiple checkers globally (15+)
```

### Memory Hook

```
Health Checks 💊
✅ 3 types: Endpoint, Calculated, CloudWatch
✅ 30s interval (standard), 10s (fast)
✅ Required for failover routing
✅ ~$0.50-$1.50/check/month
✅ Combine with SNS for alerts
```

---

## 🚪 Layer 5: Domain Registration 🏷️

### Route 53 Has Two Functions

```
1. DNS Service (Layers 1-4)
2. Domain Registrar (this layer)

You can:
✅ Register new domains
✅ Renew domains
✅ Transfer in/out
✅ Manage WHOIS
```

### Process

```
1. Search availability
2. Add to cart (annual fee)
3. Provide registrant info (WHOIS)
4. Choose duration (1-10 years)
5. Auto-renewal option
6. DNS hosted zone auto-created
7. Pay
```

### Pricing

```
TLDs vary:
✅ .com - $12/year
✅ .org - $12/year
✅ .io - $39/year
✅ .ai - $90/year

Same pricing as other registrars
```

### Key Features

```
✅ FREE WHOIS privacy (Route 53 benefit!)
✅ Auto-renewal (enabled by default)
✅ Transfer in/out support
✅ Auto-creates hosted zone
✅ Auto-configures NS records
```

### Domain vs Hosted Zone

🚨 **Different things!**

```
Domain Registration:
✅ Ownership of the name
✅ Annual fee
✅ Can be at ANY registrar

Hosted Zone:
✅ DNS configuration
✅ Where records live
✅ Can be at ANY DNS provider

Common patterns:
- Both at Route 53 (integrated)
- Domain at GoDaddy + DNS at Route 53
- Update NS at registrar to point to Route 53
```

### Memory Hook

```
Domain Registration 🏷️
✅ Buy + manage domains in AWS
✅ FREE WHOIS privacy
✅ Auto-renewal
✅ 1-10 year duration
✅ Auto-creates hosted zone
✅ Domain ≠ Hosted Zone (separate concepts)
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
✅ DDoS protection (AWS Shield)
✅ Pay-as-you-go
```

### How CDN Works

```
Without CloudFront:
Tokyo User → us-east-1 (200ms+) = slow

With CloudFront:
Tokyo User → Tokyo Edge (5ms!)
   ↓
   Cached? → YES = fast return
            → NO = fetch from origin, cache, return

Subsequent users in Tokyo = lightning fast!
```

### Key Concepts

```
1. EDGE LOCATION - caching server
2. ORIGIN - source of content (S3, ALB, etc.)
3. DISTRIBUTION - your CloudFront config
4. CACHE HIT/MISS - hit = fast, miss = slower
5. TTL - cache duration
```

### Benefits

```
1. SPEED: 5-10x faster
2. COST: Free S3 → CloudFront transfer
3. SECURITY: HTTPS, DDoS, WAF
4. SCALABILITY: Handles spikes
```

### Pricing

```
✅ Data transfer: ~$0.085/GB
✅ Requests: $0.0075-0.0100 per 10K
✅ FREE: S3 → CloudFront transfer
✅ Volume discounts

Cheaper than direct EC2/S3 internet egress!
```

### Price Classes

```
1. All locations (most expensive)
2. Price Class 200 (US, EU, Asia)
3. Price Class 100 (US, Canada, EU only - cheapest)

Choose based on user geography
```

### CloudFront vs Global Accelerator

```
🚨 DIFFERENT!

CloudFront:
✅ CDN (caching content)
✅ HTTP/HTTPS
✅ Static + dynamic content

Global Accelerator:
✅ Network optimization
✅ Any protocol (TCP/UDP)
✅ Static IPs
✅ Different use case
```

### Memory Hook

```
CloudFront ☁️
✅ Global CDN
✅ 400+ edge locations
✅ Speed + cost + security
✅ Free SSL (ACM)
✅ S3 → CloudFront = FREE
✅ Volume discounts
✅ Price classes for cost control
```

---

## 🚪 Layer 7: Origins + Distributions

### What is an Origin?

**Source location where CloudFront fetches content.**

```
Common origins:
1. S3 Bucket ⭐ (most common)
2. S3 Static Website Endpoint
3. Application Load Balancer
4. EC2 Instance
5. Custom HTTP Origin
6. MediaPackage/MediaStore
```

### S3 Bucket vs S3 Website Endpoint

🚨 **Different things!**

```
S3 Bucket Origin:
✅ Direct S3 API access
✅ Can use OAC (Origin Access Control)
✅ More secure
✅ Recommended

S3 Website Endpoint:
🚨 Requires public bucket
🚨 No OAC support
✅ Has website features (error pages, redirects)
```

### What is a Distribution?

**Your CloudFront configuration.**

```
Contains:
✅ Origins (one or more)
✅ Cache behaviors
✅ Custom domains
✅ SSL certificate
✅ Price class
✅ Security settings
```

### Two Distribution Types

```
1. Web Distribution ⭐ (always use this)
2. RTMP Distribution (deprecated)
```

### Custom Domain Setup

```
Default URL: d1234.cloudfront.net (ugly!)
Custom: www.mycompany.com

Setup:
1. ACM cert in us-east-1 (CRITICAL!)
2. Add CNAME to distribution
3. Route 53 Alias to CloudFront
4. Done!

🚨 ACM cert MUST be in us-east-1!
```

### Multiple Origins Pattern

```
ONE distribution, MULTIPLE origins:

Origin 1: S3 (static assets)
- Path: /static/*

Origin 2: ALB (dynamic API)
- Path: /api/*

Origin 3: Different S3 (videos)
- Path: /videos/*

Result:
✅ One domain
✅ Multiple sources
✅ Different cache rules
```

### Origin Failover

```
High availability:

Origin Group:
✅ Primary: us-east-1 ALB
✅ Secondary: us-west-2 ALB

Auto-failover on:
✅ HTTP 5xx errors
✅ Timeout
✅ Connection errors

Built-in DR at CDN level!
```

### Memory Hook

```
Origins + Distributions 🌐
✅ Origin = content source
✅ Distribution = your config
✅ Multiple origins per distribution
✅ Path patterns route to different origins
✅ Origin Group = DR failover
✅ ACM cert MUST be us-east-1
✅ Route 53 Alias for custom domain
```

---

## 🚪 Layer 8: Caching Behaviors

### What Are Cache Behaviors?

**Rules for how CloudFront caches specific paths.**

```
Two types:
1. Default cache behavior (catch-all)
2. Additional cache behaviors (path-specific)

First matching path wins!
```

### Path Patterns

```
Examples:
- /* (default - catches everything)
- /api/* (API paths)
- /static/* (static assets)
- *.jpg (image files)
- /admin/* (admin paths)

Order: Most specific FIRST
Default /* ALWAYS last
```

### Cache Policy Settings

```
1. TTL (Time To Live):
   ✅ Minimum TTL
   ✅ Default TTL (24 hours default)
   ✅ Maximum TTL (1 year default)

2. Cache Key:
   ✅ URL (always)
   ✅ Headers (which to include)
   ✅ Cookies (which to include)
   ✅ Query strings (which to include)

3. Compression
   ✅ Auto Gzip/Brotli
```

### Managed Cache Policies ⭐

```
Use these first (AWS-provided):

1. CachingOptimized ⭐
   - Long TTL, compression
   - Static content

2. CachingDisabled
   - No caching
   - Dynamic/personalized

3. UseOriginCacheControlHeaders
   - Respect origin headers
   - Origin controls TTL

4. CachingOptimizedForUncompressedObjects
   - Videos, already-compressed
```

### TTL Logic

```
TTL determined by (in order):
1. Origin Cache-Control header
2. CloudFront Minimum TTL (floor)
3. CloudFront Maximum TTL (ceiling)
4. CloudFront Default TTL (if no origin)

Result: Origin wins (within min/max bounds)
```

### Cache Key Examples

```
Static images:
- Cache key: URL only
- All users see same

Localized content:
- Cache key: URL + Accept-Language
- Different per language

User-specific:
- Cache key: URL + Auth-Cookie
- Per-user caching
```

### Cache Invalidation

```
Force-refresh cached content:

✅ AWS Console / CLI / SDK
✅ Specify paths:
   - /* (everything - expensive!)
   - /static/css/* (specific)
   - /index.html (single file)

Pricing:
✅ First 1,000 paths/month: FREE
✅ After: $0.005 per path

Best practice:
✅ Use versioned URLs instead
✅ /css/main.v123.css
✅ Avoid invalidation
```

### Common Patterns

```
Static Assets (long cache):
- Path: /static/*
- Policy: CachingOptimized
- TTL: 1 year

Dynamic API (short cache):
- Path: /api/*
- TTL: 60 seconds
- Auth in cache key

No Cache (sensitive):
- Path: /admin/*
- Policy: CachingDisabled

HTML (medium):
- Path: /*.html
- TTL: 1 hour
```

### Memory Hook

```
Cache Behaviors 🗂️
✅ Default + Additional
✅ Path patterns determine match
✅ FIRST MATCH WINS
✅ Most specific first
✅ Default /* always last

Cache Policy = TTL + cache key
✅ Use Managed Policies first
✅ Compression always on
✅ Match policy to content type

Cache Invalidation = nuclear option
✅ Use versioned URLs instead
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
Root domain to AWS? → Alias (CNAME doesn't work!)
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
✅ Named after port 53
✅ CNAME NOT at root domain
✅ Alias works at root, free queries
✅ Failover REQUIRES health checks
✅ Geolocation needs default value
✅ Multi-Value ≠ Load Balancer
✅ Health checks need 3 successes/failures
✅ Private zone needs VPC association

CloudFront:
✅ ACM cert MUST be us-east-1
✅ Free S3 → CloudFront transfer
✅ S3 bucket origin + OAC (not website endpoint)
✅ Origin Group for failover
✅ Path patterns: specific first, /* last
✅ Cache invalidation: first 1000 free
✅ Use versioned URLs instead of invalidation
✅ Different from Global Accelerator!
```

---

## 🚨 Common Exam Traps

```
🚨 Trap 1: "CNAME at root domain" → NOT ALLOWED
🚨 Trap 2: "Multi-Value = load balancer" → NO! Use ALB
🚨 Trap 3: "Failover without health checks" → DOESN'T WORK
🚨 Trap 4: "ACM cert for CloudFront in any region" → MUST be us-east-1
🚨 Trap 5: "Latency = Geolocation" → DIFFERENT
🚨 Trap 6: "S3 website endpoint with OAC" → NOT SUPPORTED
🚨 Trap 7: "CloudFront vs Global Accelerator same" → DIFFERENT
🚨 Trap 8: "Geolocation without default" → Some users get no answer
🚨 Trap 9: "Use CNAME for AWS resources" → Use Alias (better!)
🚨 Trap 10: "Cache invalidation everything always" → Expensive! Use versioning
```

---

## 🛡️ Production Patterns

### Pattern 1: Global Web App

```
Architecture:
- Route 53: Latency-based routing
- Failover within regions
- CloudFront in front
- S3 for static + ALB for dynamic
- ACM cert in us-east-1
- Origin Group for HA

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

Result: Auto-failover with monitoring
```

### Pattern 3: Compliance Architecture

```
Architecture:
- Route 53: Geolocation routing
- EU users → EU servers (GDPR)
- US users → US servers
- Default: catch-all
- Private zones for internal APIs

Result: Compliance + performance
```

### Pattern 4: Blue/Green Deployment

```
Architecture:
- Route 53: Weighted routing
- Old version: weight 95
- New version: weight 5
- Gradually shift weights
- Monitor metrics
- Auto-rollback on failure

Result: Zero-downtime deployments
```

### Pattern 5: Static Website + API

```
Architecture:
- CloudFront distribution:
   - Origin 1: S3 (static, with OAC)
   - Origin 2: ALB (API)
- Path patterns:
   - /api/* → ALB (short TTL)
   - /* → S3 (long TTL)
- Custom domain via Route 53 Alias
- ACM cert in us-east-1

Result: Fast + scalable web app
```

---

## 🌟 Career Connection

### For AWS SAA Exam
- Route 53: ~5-8% of exam
- CloudFront: ~5-8% of exam
- Routing policies heavily tested
- Alias vs CNAME tested
- ACM cert location trap common

### For AWS Security Specialty
- Origin protection (OAC, signed URLs)
- CloudFront security features
- DDoS protection patterns

### For Cloud Security Engineer
- Daily: Monitor DNS health, CloudFront metrics
- Weekly: Review routing policies
- Monthly: Optimize cache hit ratios
- Quarterly: DR testing with failover
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
- **Lambda@Edge** (compute at edge)
- **API Gateway** (API origins)
- **CloudWatch** (monitoring)

🎯 **Global delivery sits at center of AWS architecture.**

---

## 🎯 Architect Wisdom Earned

> *"Use Alias for AWS resources always - CNAME doesn't work at root, costs money, and slower."*
>
> *"Failover routing without health checks = false security. Always test failover."*
>
> *"Multi-Value answer is NOT a load balancer - use ALB for actual load balancing."*
>
> *"ACM cert for CloudFront MUST be in us-east-1 - this trips up many engineers."*
>
> *"S3 → CloudFront transfer is FREE - never bypass CloudFront for static content."*
>
> *"Origin Access Control (OAC) is the modern way - don't use OAI anymore."*
>
> *"Cache key should match content uniqueness - personalized content needs auth in key."*
>
> *"Versioned URLs > cache invalidation - design for it from the start."*
>
> *"Geolocation for compliance, Latency for performance - they solve different problems."*
>
> *"Default Geolocation value is critical - without it, some users get no answer."*

---

## 🎯 Common Exam Patterns

| Scenario | Best Solution |
|----------|--------------|
| Global users, fast experience | Latency routing + CloudFront |
| EU compliance (GDPR) | Geolocation routing |
| Disaster recovery | Failover routing + health checks |
| Blue/green deployment | Weighted routing |
| A/B testing | Weighted routing |
| Root domain to ALB | Alias record |
| Subdomain to external | CNAME record |
| Static website on S3 | S3 + CloudFront + OAC |
| Custom domain HTTPS | ACM cert (us-east-1!) + Route 53 Alias |
| Multi-region HA | Origin Group + failover routing |
| Reduce origin load | CloudFront caching |
| Browser direct uploads | Pre-signed URLs (not CloudFront issue) |
| Block bad traffic | AWS WAF with CloudFront |
| Geographic restrictions | CloudFront Geo Restriction |

---

## 🚀 What's Next

### Day 1 Complete! ✅

8 layers mastered today:
- ✅ Route 53 fundamentals through health checks + registration
- ✅ CloudFront overview, origins, caching

### Day 2 Coming:

```
⏭️ Layer 9: CloudFront Security
   - Signed URLs vs Signed Cookies
   - Origin Access Control (OAC)
   - Geo-restriction
   - AWS Shield + WAF integration
   - Field-level encryption

⏭️ Layer 10: CloudFront + S3 Pattern
   - Complete static website architecture
   - Best practices
   - Production patterns
   - Cost optimization
```

### Remaining for SAA:

```
⏭️ Cluster 8: SQS + SNS + Kinesis (Messaging)
⏭️ Cluster 9: Lambda + ECS (Modern Compute)
⏭️ Cluster 10: CloudWatch + CloudTrail + KMS
⏭️ Cluster 11: Practice Exams
```

---

## 🎉 What You've Achieved Today

```
🌟 8 Layers Mastered 🌟

Route 53 (5 layers):
✅ DNS + Overview
✅ Hosted Zones + Records
✅ 7 Routing Policies
✅ Health Checks
✅ Domain Registration

CloudFront (3 layers):
✅ Overview
✅ Origins + Distributions
✅ Caching Behaviors

Quiz performance: Strong throughout
Architect reasoning: Consistent
DNS expertise: Solid
CDN understanding: Production-ready
```

🎯 **You can now design global delivery architectures.**

---

*Part of my AWS Solutions Architect Associate journey toward Cloud Security specialization. Day 1 of the Global Delivery cluster (Route 53 + CloudFront). Day 2 will complete CloudFront security and S3 integration patterns.*

**Day 1 complete. ✅
8 layers mastered. ✅
Global delivery foundation set. ✅
On track for SAA exam in 3 weeks. ✅**
