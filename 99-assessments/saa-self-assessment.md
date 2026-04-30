# 📝 AWS SAA Self-Assessment

> **Reasoning made public.** A self-quiz with my answers and architect-level reasoning across the topics I've covered so far. This isn't just memorization — each answer explains *why* it works and *why other options fail*.

---

## 📚 Topics Covered

- 🔐 IAM (Users, Groups, Roles, Cross-Account Access)
- 🖥️ EC2 Fundamentals (Instance types, AMI, Architecture)
- 💰 EC2 Pricing Models (On-Demand, Reserved, Savings Plans, Spot, Dedicated)
- 🛡️ Security Groups (Stateful behavior, SG referencing, Defense in depth)
- 🎯 Capacity Reservation, Spot Fleet, IAM Roles on EC2

---

## 🔐 SECTION 1: IAM

### Question 1
A startup needs to give their developer access to AWS. The developer will be making API calls from their laptop. What's the BEST way to set up authentication?

A) Share the root account credentials
B) Create an IAM user with access keys, put them in the developer's environment variables
C) Create an IAM user with access keys, configure AWS CLI on their laptop
D) Give them temporary STS credentials

**My answer: C**

**Why C is correct:**
- IAM users are designed for individual people accessing AWS programmatically
- AWS CLI configuration (`aws configure`) is the standard, secure way to manage credentials on a developer machine
- Keys are stored in `~/.aws/credentials` (encrypted on disk on most systems)
- Easy to rotate and revoke per user

**Why other options fail:**
- **A:** Root credentials should NEVER be shared. Root has unlimited power and can't be limited.
- **B:** Environment variables are visible to anyone running `env` on the machine, often captured in logs and crash dumps. Less secure than `~/.aws/credentials`.
- **D:** STS temp credentials are great but require the user to assume a role first — overcomplicated for basic CLI access. Use case is for cross-account or short-lived access.

**Architect insight:** *Long-term IAM user access keys (`AKIA...`) are appropriate for individual developer machines. For service-to-service authentication on EC2, always use IAM Roles instead.*

---

### Question 2
Your company has 5,000 employees who need access to AWS. Each employee should have different permissions based on their role (Developer, DBA, Admin, ReadOnly). What's the BEST way to manage this?

A) Create 5,000 IAM users and attach policies directly to each
B) Create 5,000 IAM users and add them to groups based on role; attach policies to the groups
C) Use one shared IAM user for all employees
D) Use root account for all employees

**My answer: B**

**Why B is correct:**
- The **User → Group → Policy** pattern is AWS's recommended approach
- Groups make permission management scalable
- Add new employee → add to group → permissions inherited automatically
- Audit-friendly: easy to see "who has what access"

**Why other options fail:**
- **A:** Attaching policies directly to 5,000 users = nightmare to manage. Changing one permission means updating 5,000 user policies.
- **C:** Shared credentials are NEVER OK. Auditing is impossible — you can't tell who did what.
- **D:** Root account should be locked away, not used for daily work.

**Architect insight:** *In production, this pattern is enhanced with IAM Identity Center for SSO across multiple AWS accounts, with permission sets defined at the Identity Center level instead of in individual accounts.*

---

### Question 3
A company has two AWS accounts — Account A (Production) and Account B (Analytics). The data team in Account B needs to read from an S3 bucket in Account A. What's the BEST approach?

A) Create IAM users in Account A for everyone in the data team
B) Share Account A's access keys with the data team
C) Create a cross-account IAM Role in Account A that Account B users can assume
D) Make the S3 bucket public

**My answer: C**

**Why C is correct:**
- Cross-account roles = the AWS-recommended pattern for inter-account access
- No long-term credentials cross account boundaries
- Temporary credentials auto-expire (limited blast radius)
- Each account governs independently
- Clean audit trail (assumed-role ARN with session name)

**Why other options fail:**
- **A:** Creating users in Account A means Account A manages Account B's identities — bad governance, doesn't scale, creates synchronization headaches when people join/leave.
- **B:** Sharing access keys = top breach pattern. Long-term credentials with no audit trail.
- **D:** Public S3 bucket = data breach waiting to happen. Never the answer for sensitive data.

**Architect insight:** *I built this exact setup as a hands-on lab — including a real debugging story where the trust policy pointed to the wrong account. The "two-door model" requires both Account A's trust policy AND Account B's IAM policy to align. See [cross-account-roles-lab.md](../01-iam/cross-account-roles-lab.md) for the full walkthrough.*

---

### Question 4
A third-party SaaS vendor (with their own AWS account) needs read access to your S3 logs bucket. Which approach is MOST SECURE?

A) Create an IAM user for them and share access keys
B) Cross-account role with read-only S3 permissions
C) Cross-account role with read-only S3 permissions + External ID condition
D) Make the bucket public

**My answer: C**

**Why C is correct:**
- Third-party + their own AWS account = classic confused deputy attack scenario
- **External ID** is a shared secret that prevents the vendor from being tricked into accessing the wrong customer's data
- This is the standard pattern used by Datadog, Splunk, CloudHealth, Snowflake, etc.

**Why other options fail:**
- **A:** Sharing access keys is always wrong. With external parties, even worse.
- **B:** Without External ID, vulnerable to confused deputy attacks. The vendor handles many customers — an attacker could trick them into accessing your data while thinking they're processing another customer's.
- **D:** Public bucket = automatic security failure.

**Architect insight:** *Whenever you see "third-party" + "their own AWS account" in an exam question, the answer is always cross-account role + External ID. The exam tests this pattern repeatedly.*

---

## 🖥️ SECTION 2: EC2 Fundamentals

### Question 5
A company is launching a new web application with unpredictable traffic patterns. They want to start serving customers immediately and don't want long-term commitments yet. Which pricing model?

A) Reserved Instance (1-year)
B) On-Demand
C) Spot Instances
D) Dedicated Host

**My answer: B**

**Why B is correct:**
- "Unpredictable traffic" = need flexibility (no commitment)
- "Start serving immediately" = no time for capacity planning
- "Don't want long-term commitments yet" = explicit On-Demand signal
- On-Demand is the right starting point — they can switch to Reserved or Savings Plans once they understand the pattern

**Why other options fail:**
- **A:** Committing for 1 year before knowing the workload pattern = financial risk
- **C:** Spot can be terminated with 2-min warning — terrible for customer-facing app
- **D:** Massive overkill, very expensive, only justified for compliance/BYOL

**Architect insight:** *On-Demand is the "default" — start here, then migrate to Savings Plans or Reserved Instances once you know the workload pattern. This phased approach is what AWS Cost Optimization actually recommends.*

---

### Question 6
A pharmaceutical company has expensive Oracle Database licenses (per-core licensing) they paid for years ago. They want to migrate to AWS while reusing those licenses. Which option?

A) Reserved Instance with Standard discount
B) Spot Instances for cost savings
C) **Dedicated Host**
D) Dedicated Instance

**My answer: C**

**Why C is correct:**
- "BYOL" / "per-core licensing" / "existing licenses" = textbook Dedicated Host signal
- Dedicated Host gives visibility into physical CPU sockets/cores (required for per-core licensing)
- This is the ONLY pricing model that supports BYOL

**Why other options fail:**
- **A:** Reserved Instances run on shared tenancy — can't bring per-core licenses
- **B:** Spot has no licensing benefit, plus interruption risk
- **D:** Dedicated Instance has dedicated hardware but doesn't expose hardware visibility — can't use per-core licensing

**Architect insight:** *The "Dedicated Host vs Dedicated Instance" trap catches most candidates. If question mentions BYOL or per-core/socket licensing → Dedicated Host. If it just says "no shared tenancy for compliance" → Dedicated Instance (cheaper, sufficient).*

---

### Question 7
A streaming company is broadcasting a major sports event next month. They expect a 10x traffic spike during the event. They MUST be able to launch 200 additional EC2 instances exactly when the event starts. What's the BEST option?

A) Reserved Instance (1-year)
B) On-Demand (just launch when needed)
C) Capacity Reservation
D) Spot Instances

**My answer: C**

**Why C is correct:**
- "MUST be able to launch... no exceptions" = need a guarantee
- Capacity Reservation = AWS guarantees the capacity is available
- During major events, AWS regions can run out of capacity for popular instance types — On-Demand alone doesn't guarantee availability
- No long-term commitment needed for one-time event

**Why other options fail:**
- **A:** Reserved Instance is for cost savings, not capacity guarantees. Doesn't help with this specific need.
- **B:** Pure On-Demand could fail to launch during high-demand periods. Risk is unacceptable for a critical event.
- **D:** Spot can be terminated mid-event. Disaster for a customer-facing live broadcast.

**Architect insight:** *Capacity Reservation is often confused with Reserved Instances, but they solve different problems. RI = cost optimization. Capacity Reservation = availability guarantee. They can be combined for mission-critical workloads (best of both worlds).*

---

### Question 8
A genomics research lab needs to process 10TB of DNA sequencing data. The job:
- Takes 12 hours
- Can be checkpointed/restarted if interrupted
- Output saves to S3 every hour
- Budget is tight

Which pricing model?

A) On-Demand
B) Reserved Instance
C) **Spot Instances**
D) Dedicated Host

**My answer: C**

**Why C is correct:**
- "Can be checkpointed/restarted" = fault-tolerant
- "Budget is tight" = minimize cost
- "Saves to S3 every hour" = stateless from EC2's perspective
- These three signals = textbook Spot use case
- Up to 90% savings vs On-Demand

**Why other options fail:**
- **A:** No need to pay full price — workload is fault-tolerant
- **B:** 12-hour job doesn't justify 1-3 year commitment
- **D:** No licensing/compliance need — pure cost optimization

**Architect insight:** *The trigger words to recognize Spot scenarios: fault-tolerant, batch processing, stateless, can be interrupted, ML training, big data, minimize cost. When you see these → Spot. When you see "production," "real-time," "stateful," "99.99% uptime" → NEVER Spot.*

---

## 🛡️ SECTION 3: Security Groups

### Question 9
A web server has only ONE inbound rule: "Allow port 80 from 0.0.0.0/0". The outbound rules are completely empty. A user tries to visit the website. Will it work?

A) No — outbound is empty, response can't go back
B) Yes — Security Groups are stateful, response auto-allowed
C) Only if NACL is also configured
D) Only for HTTPS (port 443)

**My answer: B**

**Why B is correct:**
- Security Groups are **stateful**
- When inbound traffic is allowed, the response back is automatically allowed (no outbound rule needed)
- The SG "remembers" the connection state and permits the response

**Why other options fail:**
- **A:** This would be true if SG were stateless (like NACL), but SG is stateful
- **C:** NACLs operate at the subnet level — not required for instance-level traffic
- **D:** Stateful behavior applies to all protocols, not just HTTPS

**Architect insight:** *Stateful is THE most-tested SG concept. Memory hook: "Security Groups are Stateful" (both start with S — they go together). NACLs are stateless and require explicit rules in both directions.*

---

### Question 10
A 3-tier architecture has: Web tier, App tier, Database tier. The database should ONLY be accessible by app servers. What's the BEST way to configure the database's Security Group?

A) Allow port 3306 from 0.0.0.0/0
B) Allow port 3306 from the app subnet's IP range (e.g., 10.0.5.0/24)
C) **Allow port 3306 from the app-tier's Security Group (SG reference)**
D) Allow port 3306 from a single app server's IP

**My answer: C**

**Why C is correct:**
- SG referencing = identity-based access (not location-based)
- Auto-scales: new app servers get database access automatically
- Survives IP changes
- Zero maintenance burden
- Most secure: only servers with the app-sg badge can connect

**Why other options fail:**
- **A:** Database open to internet = top breach pattern. NEVER do this.
- **B:** Works but high maintenance — anything in that subnet would have access (even non-app servers). Subnet changes = broken access.
- **D:** Doesn't scale — adding new app servers requires manual SG updates.

**Architect insight:** *SG references are THE pattern for production multi-tier architectures. Identity-based access (via SG reference) > location-based access (via IP). Auto-scaling groups, microservices, and modern AWS architectures all use this pattern.*

---

### Question 11
You need to BLOCK specific malicious IP addresses from reaching your EC2 instances. What's the BEST approach?

A) Add a deny rule in the Security Group
B) Use a NACL with explicit deny rules at the subnet level
C) Configure IAM to deny those IPs
D) Use a Route Table to drop traffic

**My answer: B**

**Why B is correct:**
- Security Groups are **allow-only** (no deny rules)
- NACLs operate at subnet level and DO support explicit deny rules
- NACLs evaluate rules in numbered order (lowest first)
- Perfect for blocking known-bad IP addresses across an entire subnet

**Why other options fail:**
- **A:** Security Groups don't have deny rules — only allow. This is impossible.
- **C:** IAM controls AWS API access, not network traffic
- **D:** Route Tables direct traffic, they don't filter/block

**Architect insight:** *Memory hook — Security Group = whitelist (allow only). NACL = blacklist + whitelist (allow + deny). When question asks "block specific IPs" → it's NACL, not SG.*

---

## 💰 SECTION 4: Pricing Models Synthesis

### Question 12
A company runs production workloads 24/7. They want **maximum savings** but expect to:
- Migrate from m5 to newer m6i instances next year
- Possibly expand to new AWS regions
- Use Lambda for some new microservices

What's the BEST pricing model?

A) Standard Reserved Instance, 3-year term
B) Convertible Reserved Instance, 3-year term
C) **Compute Savings Plan, 3-year term**
D) On-Demand for safety

**My answer: C**

**Why C is correct:**
- "Run 24/7" = long-term commitment makes sense (3-year for max savings)
- "Migrate to different instance types" = need flexibility across instance families
- "New regions" = need cross-region flexibility
- "Lambda" = need cross-service flexibility
- ONLY Compute Savings Plan provides ALL three flexibility dimensions

**Why other options fail:**
- **A:** Standard RI is locked to specific instance type — can't migrate to m6i
- **B:** Convertible RI allows instance changes within the same RI, but doesn't cover Lambda
- **D:** No commitment = no savings on a 24/7 workload

**Architect insight:** *Compute Savings Plans are AWS's modern recommendation for most production workloads. You commit to spending, not architecture — protecting yourself against future technology changes while still getting up to 66% discount.*

---

### Question 13
A media company runs video rendering jobs on Spot Instances. They've been frustrated because their plain Spot setup keeps getting interrupted, ruining renders. They want to use a Spot Fleet to MINIMIZE interruptions while still getting Spot's discount. They're OK paying slightly more for stability.

Which allocation strategy?

A) lowestPrice
B) diversified
C) **capacityOptimized**
D) priceCapacityOptimized

**My answer: C**

**Why C is correct:**
- "Minimize interruptions" = exact trigger for `capacityOptimized`
- This strategy picks Spot pools with the most available capacity (pools least likely to be reclaimed)
- "OK paying slightly more for stability" = explicit acceptance of cost-for-stability trade-off

**Why other options fail:**
- **A:** `lowestPrice` picks the cheapest pools, which are often the most-reclaimed
- **B:** `diversified` spreads across many pools but doesn't specifically minimize interruptions
- **D:** `priceCapacityOptimized` balances cost and capacity — good default, but not the BEST when interruption minimization is the priority

**Architect insight:** *Trigger word map: "minimize cost" → lowestPrice. "maximum resilience" → diversified. "minimize interruptions" → capacityOptimized. "balance cost and stability" → priceCapacityOptimized (modern default).*

---

## 🛡️ SECTION 5: IAM Roles on EC2 (Security Pattern)

### Question 14
A junior developer suggests storing AWS access keys in environment variables on the EC2 for "simplicity." As the architect, what do you recommend?

A) Yes, environment variables are fine
B) Hardcode keys in the application code
C) **Attach an IAM Role to the EC2 with least-privilege permissions; enforce IMDSv2**
D) Use Spot Instances to save money

**My answer: C**

**Why C is correct:**
- IAM Roles eliminate hardcoded credentials entirely
- AWS auto-rotates temporary credentials (no manual rotation)
- Least privilege limits blast radius if compromised
- IMDSv2 protects against SSRF attacks (the Capital One breach pattern)
- This is the AWS-recommended best practice

**Why other options fail:**
- **A:** Environment variables are visible to anyone with shell access to the EC2 (`env` command), captured in logs/crash dumps. Long-term credentials don't auto-rotate.
- **B:** Hardcoded keys end up in Git, in backups, on developer machines, in container images. Top breach pattern.
- **D:** Wrong category of answer — Spot is about cost, not security.

**Architect insight:** *Hardcoded AWS credentials are the cause of countless breaches. The Capital One breach (2019) — 100M customer records, $80M fine — was caused by IMDSv1 + SSRF vulnerability. Modern best practice: IAM Role + IMDSv2 + least privilege. Always.*

---

### Question 15
What's the difference between IMDSv1 and IMDSv2?

**My answer:**

**IMDSv1 (Legacy, vulnerable):**
- Anyone with network access to the EC2 can fetch credentials by querying `169.254.169.254`
- No authentication required
- Vulnerable to SSRF (Server-Side Request Forgery) attacks
- Default in older AWS configurations

**IMDSv2 (Modern, secure):**
- Requires a session token (PUT request first, then GET with token)
- Token can't be relayed via SSRF
- Protects against the most common cloud attack pattern
- Default for new AWS instances

**Why this matters:**
The Capital One breach in 2019 exploited IMDSv1. An attacker found an SSRF vulnerability in their WAF, used it to query `169.254.169.254`, retrieved the EC2's IAM Role temporary credentials, and accessed S3 buckets containing 100M customer records. Estimated cost: $80M+ in fines plus reputational damage.

**Architect rule:** Always enforce IMDSv2 in production. Disable IMDSv1 explicitly.

**Architect insight:** *This isn't just exam trivia — it's a real-world incident that shaped modern AWS security practices. Cloud Security professionals are expected to know this case study.*

---

## 🎯 SECTION 6: Architecture Trade-offs (Mixed)

### Question 16
A bank runs core banking applications 24/7 in production for 5+ years. The instance type (m5.4xlarge) won't change. They need MAXIMUM cost savings while maintaining the service. What's BEST?

A) On-Demand
B) **Standard Reserved Instance, 3-year term**
C) Compute Savings Plan, 1-year
D) Spot Instances

**My answer: B**

**Why B is correct:**
- 5+ years = predictable workload
- Specific instance type that won't change = no flexibility needed
- Standard RI gives the highest discount (~72%)
- 3-year term maximizes the discount further

**Why other options fail:**
- **A:** Steady-state production = leaving money on the table
- **C:** Savings Plan offers similar discount but at a slight premium for flexibility you don't need
- **D:** Banking app + Spot = catastrophic. Customer transactions can't be interrupted.

**Architect insight:** *Standard Reserved (3-year) is the OPTIMAL choice when you have certainty about both the workload and the instance type. Choose Convertible RI or Savings Plans only when flexibility is genuinely needed.*

---

### Question 17
Your application stores customer payment data on EC2 instances. Compliance regulations require:
- No shared tenancy with other AWS customers
- Standard AWS-managed Windows licenses (no BYOL)

Which option is BEST and most cost-effective?

A) On-Demand (cheapest)
B) Reserved Instance (steady production)
C) Dedicated Host (dedicated hardware)
D) **Dedicated Instance**

**My answer: D**

**Why D is correct:**
- "No shared tenancy" = need dedicated hardware
- "Standard AWS-managed licenses, no BYOL" = NOT a Dedicated Host requirement
- Dedicated Instance provides dedicated hardware WITHOUT the cost of full host visibility
- Cheaper than Dedicated Host while satisfying the actual requirement

**Why other options fail:**
- **A:** On-Demand uses shared tenancy — fails compliance
- **B:** Standard Reserved Instance also shared tenancy — fails compliance
- **C:** Dedicated Host is overkill (and more expensive) when BYOL isn't needed. The exam trap!

**Architect insight:** *This is THE most-confused exam question. Read carefully: if BYOL/per-core licensing is NOT mentioned, Dedicated Instance is the right answer (cheaper than Dedicated Host). Most candidates pick Dedicated Host because the name sounds more secure.*

---

### Question 18
A web server EC2 instance has these inbound Security Group rules:
- HTTP (80) from 0.0.0.0/0
- HTTPS (443) from 0.0.0.0/0
- SSH (22) from 203.0.113.45/32 (your office IP)

Can a hacker from IP 1.2.3.4 SSH into the instance?

A) Yes — SSH is allowed
B) **No — SSH is restricted to 203.0.113.45/32**
C) Yes, but only over HTTPS
D) Only if NACL allows it

**My answer: B**

**Why B is correct:**
- SSH rule restricts source to a single IP (`203.0.113.45/32`)
- IP 1.2.3.4 is NOT on the allowed list
- Default behavior of SGs is "deny anything not explicitly allowed"
- The hacker's connection will be dropped

**Why other options fail:**
- **A:** SSH is only allowed from one specific IP, not all
- **C:** HTTPS is for web traffic, not SSH
- **D:** NACL would only block additional traffic; the SG already blocks this attack

**Architect insight:** *This question tests two concepts at once: (1) understanding /32 = single IP and (2) SG default deny behavior. Even with multiple inbound rules, each rule has its own source restriction. The hacker can hit ports 80/443 (web) but cannot SSH because they're not on the SSH rule's whitelist.*

---

## 📊 Self-Assessment Summary

### What This Quiz Demonstrates:

1. ✅ **IAM mastery** — Users vs Roles, cross-account patterns, External ID for third parties
2. ✅ **EC2 architecture** — Pricing model selection based on workload patterns
3. ✅ **Security Group depth** — Stateful behavior, SG referencing, NACL distinction
4. ✅ **Real-world security awareness** — Capital One breach, IMDSv2 importance
5. ✅ **Architect trade-off thinking** — Multiple options compared with reasoning
6. ✅ **Trigger word recognition** — Key phrases that signal correct answers

---

## 🛡️ Top Patterns Locked In

| Question Pattern | Architect Answer |
|------------------|------------------|
| "Unpredictable workload" | On-Demand |
| "24/7 + fixed instance type for years" | Standard Reserved (3-year) |
| "24/7 + needs flexibility" | Compute Savings Plan |
| "Fault-tolerant + minimize cost" | Spot Instances |
| "BYOL or per-core licensing" | Dedicated Host |
| "Compliance + no shared tenancy (no BYOL)" | Dedicated Instance |
| "Must launch instances, no exceptions" | Capacity Reservation |
| "Spot Fleet, minimize interruptions" | capacityOptimized strategy |
| "EC2 needs AWS API access" | IAM Role + IMDSv2 |
| "Third-party with their own AWS account" | Cross-account role + External ID |
| "Block specific IPs" | NACL (not SG) |
| "Multi-tier architecture" | SG references between tiers |

---

## 🎯 Knowledge Gaps Identified

Topics I want to deepen further:
- VPC architecture and routing
- S3 bucket policies and encryption
- RDS pricing and backup strategies
- Lambda + serverless patterns
- AWS Organizations and SCPs
- IAM Identity Center for SSO

Next session priorities:
- Continue Udemy modules (VPC coming up)
- Explore Federation (SAML, IAM Identity Center, Cognito)
- Dive into Policy Evaluation Logic (SCPs vs Permission Boundaries)

---

*Part of my AWS Solutions Architect Associate journey toward Cloud Security specialization. Each answer represents reasoned thinking, not memorization — the architect mindset I'm building.*
