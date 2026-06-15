# 🔐 Security Domain (Domain 1, 30%) — SAA Exam Study Guide

> **Use-case-first**, continuing the **"Atlas Games"** build — now securing a big company: millions of players, ~200 employees, 8 AWS accounts, payment data under a regulator, and a website under daily attack. This is the largest exam domain *and* the Cloud Security career core. Each service below was reasoned out from a real requirement.

---

## 🗺️ The four security problems & their answers

| Problem | Answer |
|---|---|
| Authenticate millions of **app customers** | **Amazon Cognito** |
| One login + role-based access for the **workforce** across accounts | **IAM Identity Center** |
| Group & govern **many accounts**; enforce rules no one can override | **Organizations + OUs + SCPs + Control Tower** |
| Block **web-layer attacks** + **DDoS**, org-wide | **WAF + Shield + Firewall Manager** |
| **Dedicated, single-tenant** keys the regulator demands | **CloudHSM** (vs KMS) |

---

## 1. Identity — two different "logins"

The deciding question: **what is the person logging *into*?**

**Amazon Cognito** — authentication for **your application's customers/end-users**. Sign-up, sign-in over HTTPS, social/federated login (Google/Apple), password reset, MFA. Issues **JWT tokens** the app validates. Users live *outside* AWS's trust boundary and never get console/AWS access.
- **User Pools** = the user directory + authentication (does 99% of the work).
- **Identity Pools** = optional; exchange the identity for **temporary** AWS credentials *only* if a customer needs limited direct AWS access (e.g., upload to their own S3 folder).

**IAM Identity Center** (formerly AWS SSO) — SSO for your **workforce** into the AWS environment itself. One login → **permission sets** give role-based access per account (admin in Dev, read-only in Prod). Identities can live in its own directory, connected **Active Directory** (via AWS Directory Service), or an external IdP (Okta/Entra). Disable a user once → disabled everywhere.

> **Reflex:** "users of my app/website" → **Cognito.** "employees/workforce into AWS" → **IAM Identity Center.**

---

## 2. Multi-account governance

**AWS Organizations** — the umbrella holding all accounts under one management. Accounts group into **Organizational Units (OUs)** (e.g., Production OU, Development OU, Security OU). *(It's ONE Organization with MANY accounts — not many orgs.)*

**Service Control Policies (SCPs)** — attach at the **Root** (applies to all accounts) or an **OU** (applies to accounts in it) and set the **maximum** permissions. **Key nuance: an SCP never *grants* — it only sets a ceiling.** Even a full account admin cannot exceed it. This enforces "no one can disable CloudTrail," "only approved regions," "can't leave the Org." *(Both an SCP **and** an IAM/permission-set grant must allow an action for it to work.)*

**AWS Control Tower** — sits on top of Organizations and **automates** a secure multi-account **landing zone**: best-practice **guardrails** (preventive = SCPs; detective = Config), centralized logging into a **Log Archive** account, an **Audit** account, and an **Account Factory** that provisions *new* accounts secure-by-default. Also wires in IAM Identity Center.

**The hierarchy:**
```
Organization Root  ── SCP (org-wide guardrail)
   ├── Production OU   ── SCP (stricter)
   │     ├── Prod acct
   │     └── Analytics acct
   ├── Development OU
   │     ├── Dev acct
   │     └── Staging acct
   └── Security OU
         ├── Log Archive acct
         └── Audit acct
SCP attaches at Root or any OU → inherited downward → accounts can't exceed it.
```

> **Reflexes:** Manage many accounts → **Organizations (OUs)**. Hard guardrails no one can override → **SCPs**. Automate secure-by-default accounts → **Control Tower**.

---

## 3. Edge protection — match the tool to the attack's LAYER

The big lesson: **Security Groups work at Layer 3/4 (IP/port) — they CANNOT inspect request *content*.** A SQL injection is a malicious "letter" inside a valid "envelope" to port 443; the SG waves it through. You need a tool at the right layer.

| Layer | Threat | Tool |
|---|---|---|
| Network (L3/4) | Who can reach me, by IP/port | **Security Groups / NACLs** |
| Application (L7) | SQL injection, XSS, bad bots, rate-limit | **AWS WAF** |
| Volumetric | DDoS floods | **AWS Shield** |
| Org-wide enforcement | Apply all the above across accounts | **AWS Firewall Manager** |

- **WAF** inspects HTTP(S) requests; attaches to **ALB / CloudFront / API Gateway**; **managed rule groups** (SQLi/XSS), **rate-based rules** (bots/credential-stuffing), geo-blocking.
- **Shield Standard** = free, automatic, L3/4. **Shield Advanced** = paid; L7 DDoS, 24/7 response team, **cost protection** (refunds attack-driven scaling).
- **Firewall Manager** = central WAF/Shield/SG/Network Firewall policy across the whole Organization, including new accounts.

> **Reflexes:** SQLi/XSS/bad bots/rate-limit → **WAF** (NOT a security group). DDoS → **Shield**. Centralize across accounts → **Firewall Manager**.

---

## 4. Key management — KMS vs CloudHSM

The deciding question: **who owns and controls the physical hardware?**

- **KMS** = a safe-deposit box at a **shared** bank. AWS runs the hardware (**multi-tenant**, keys logically isolated); you manage keys + access policies. Cheap, fully integrated, FIPS 140-2/3. Right for ~99% of cases, including most compliance.
- **CloudHSM** = your **own private vault**, single-tenant dedicated hardware in your VPC. **You** have exclusive control of the key material; **AWS cannot access it.** FIPS 140-2 **Level 3**. Trade-off: more operational burden (you run the cluster, HA, backups), higher cost, less seamless integration, and **if you lose the keys, AWS can't recover them.**
- Bridge: a **KMS custom key store** can back KMS with your CloudHSM (integration + dedicated hardware).

> **Reflex:** "dedicated / single-tenant hardware," "exclusive/full key control," "FIPS 140-2 Level 3," "AWS no access" → **CloudHSM.** Everything else → **KMS.**

---

## 🔴 Security domain reflex sheet

```
App customers sign-in (social, MFA, no AWS access)   → Cognito (User Pools)
Customer needs temp AWS creds                         → Cognito Identity Pools
Workforce SSO + role-based across many accounts       → IAM Identity Center
Connect on-prem Active Directory                      → AWS Directory Service
Group/manage many accounts                            → Organizations (OUs)
Hard guardrail no admin can override                  → Service Control Policy (SCP)
Secure-by-default new accounts + landing zone         → Control Tower
SQL injection / XSS / bad bots / rate-limit           → WAF (not a security group!)
DDoS (advanced + 24/7 + cost protection)              → Shield Advanced
Enforce WAF/Shield/SG org-wide                         → Firewall Manager
Find resources shared externally                      → IAM Access Analyzer
Dedicated single-tenant keys, AWS no access            → CloudHSM
Managed, integrated, default encryption                → KMS
Auto secret rotation                                   → Secrets Manager
```

---

## ✅ Consolidation check

Day-3 Domain-1 question set (qualifier style): **7/7.** Notably correct on brand-new material — Cognito-vs-Identity-Center, SCPs, Control Tower, the WAF-not-SG layer lesson, Shield Advanced, and CloudHSM (which started the session as an unknown). Use-case learning landed.

---

*Part of my AWS SAA journey toward Cloud Security / AI Security. Domain 1 (Security, 30%) is now my stronghold — directly aligned with the SCS-C02 and CCSP path ahead. Remaining: Day 4 Resilience/Performance, Day 5 Cost + first timed mock, then practice exams to a consistent 80%+ before booking.*
