# 🔐 IAM Fundamentals

> Notes from my AWS SAA learning journey — covering core IAM concepts with a security-first lens.

---

## 🎯 What is IAM?

**IAM (Identity and Access Management)** is AWS's authentication and authorization service. It's:

- 🌍 **Global** (not region-specific)
- 💰 **Free** to use
- 🛡️ **Critical** — almost every AWS security incident involves IAM misconfiguration

Two questions IAM always answers:

- **Authentication:** Who are you? (login)
- **Authorization:** What can you do? (permissions)

---

## 🧱 Core Building Blocks

| Entity | Purpose | Credentials |
|--------|---------|-------------|
| **Root User** | Account owner with unlimited power | Email + password (never use daily!) |
| **IAM User** | Person or application | Long-term access keys / password |
| **Group** | Collection of users (for permissions) | None — inherits from policies |
| **Role** | Identity assumed temporarily | None — gets temp credentials via STS |
| **Policy** | JSON document defining permissions | N/A |

---

## 🛡️ Root User Best Practices

The root user has unlimited power and **cannot have permissions limited**. Treat it like nuclear codes:

- ✅ **Enable MFA immediately** (virtual MFA app like Google Authenticator)
- ✅ **Delete root access keys** if any exist
- ✅ **Use only for ~10 specific tasks** (close account, change support plan, etc.)
- ✅ **Create an IAM admin user for daily work** instead

**Why this matters:** If root credentials leak, the entire account is compromised. There's no recovery — only damage control.

---

## 👥 The User → Group → Policy Pattern

**Bad practice:** Attaching policies directly to users (doesn't scale)

**Best practice:**
1. Create groups by role/function (e.g., `Developers`, `Admins`, `ReadOnly`)
2. Attach policies to groups
3. Add users to groups

**Why?**
- ✅ **Scalable** — onboard 50 devs by adding to one group
- ✅ **Auditable** — see "who has what" at a glance
- ✅ **Maintainable** — change a policy once, applies to everyone

---

## 📜 Policy Structure

Every IAM policy is a JSON document with these key elements:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": "192.168.1.0/24"
        }
      }
    }
  ]
}
```

| Field | Purpose |
|-------|---------|
| **Effect** | Allow or Deny |
| **Action** | What API call (e.g., `s3:GetObject`) |
| **Resource** | What it applies to (ARN) |
| **Condition** | Optional extra rules (IP, MFA, time, etc.) |

---

## 🎯 Policy Evaluation Logic

When AWS evaluates a request, it follows strict rules:

1. **Default = Implicit Deny** (if no policy explicitly allows it)
2. **Explicit Allow** in any policy → Access granted
3. **Explicit Deny** in any policy → **Always wins**, overrides everything

Order of evaluation:
1. SCPs (at AWS Organizations level)
2. Permission Boundaries
3. Identity-based policies
4. Resource-based policies
5. Session policies (for assumed roles)

---

## 🎭 Roles — The Most Important IAM Concept

**A role is an identity that anyone authorized can temporarily "wear"** to perform actions, without needing their own permanent credentials.

### Why Roles Exist

❌ **Bad:** Hardcoding access keys in EC2 applications
- Keys leak when servers are compromised
- Keys never rotate
- Hard to audit

✅ **Good:** Attaching a Role to EC2
- AWS automatically gives temp credentials
- Auto-rotation
- No secrets stored on disk

**Rule of thumb:** *AWS service needs to talk to another AWS service → use a Role. Always.*

### Anatomy of a Role

Every role has **two policies**:

1. **Trust Policy** — *Who* can assume this role?
2. **Permissions Policy** — *What* can the assumed role do?

### Common Role Types

| Scenario | Role Type |
|----------|-----------|
| EC2 needs AWS access | Service role (Instance Profile) |
| Lambda function executing | Lambda execution role |
| Account A user accessing Account B | Cross-account role |
| Corporate AD users → AWS | SAML federation role |
| AWS service acts on your behalf | Service-linked role |

---

## 🎟️ STS (Security Token Service)

STS issues **temporary credentials** when a role is assumed:

- **Access Key ID** (starts with `ASIA...`)
- **Secret Access Key**
- **Session Token** ← unique to STS, proves credentials are temporary
- **Expiration** (default 1 hour, max 12 hours)

### ARN Prefix Identifiers

| Prefix | Meaning |
|--------|---------|
| AKIA   | Long-term IAM user access keys |
| ASIA   | Temporary STS credentials |
| AROA   | IAM Role identifier |
| AIDA   | IAM User identifier |

These prefixes make it trivial to identify credential types in CloudTrail logs — a key skill for security investigations.

---

## 🏢 Multi-Account Strategy

**Best practice:** Separate environments (Dev, Staging, Prod) into different AWS accounts.

### Benefits

- 🛡️ **Hard isolation** — mistakes in Dev can't affect Prod
- 💰 **Billing separation** — see costs per environment
- 📋 **Compliance** — auditors love account-level boundaries
- 🎯 **Reduced "blast radius"** — limit damage if something goes wrong

### Tools for Multi-Account

- **AWS Organizations** — manage multiple accounts centrally
- **Service Control Policies (SCPs)** — restrict what entire accounts can do
- **IAM Identity Center** (formerly AWS SSO) — single sign-on across accounts

### Exam Triggers

When you see these phrases, think multi-account:
- "Reduce blast radius"
- "Strong isolation boundary"
- "Centralized governance"
- "Single sign-on across accounts"

---

## 🤝 Cross-Account Access

When you need to grant access between two AWS accounts you own (or with a third party):

✅ **Use cross-account roles** — never share IAM users or access keys

The "Two-Door" Model:
1. Account A's trust policy must allow Account B
2. Account B's IAM policy must allow the user to call AssumeRole

**For third-party access:** Always require **External ID** to prevent the "confused deputy" attack.

📁 See [Cross-Account Roles Lab](./cross-account-roles-lab.md) for a hands-on walkthrough including a real debugging story.

---

## 🛡️ Permission Boundaries vs SCPs

These are commonly confused but serve different purposes:

| Concept | Scope | Purpose |
|---------|-------|---------|
| **Permission Boundary** | Single user/role | Maximum permissions a user can have |
| **SCP (Service Control Policy)** | Entire account/OU | Maximum permissions for whole account |

**Key insight:** Neither *grants* permissions — they only *limit* them. You still need an Allow in an identity or resource policy.

---

## 🎯 Federation Quick Reference

When users come from outside AWS:

| User Type | Solution |
|-----------|----------|
| Corporate employees (existing AD) | SAML 2.0 federation OR IAM Identity Center |
| Multiple AWS accounts + SSO | **IAM Identity Center** (modern, recommended) |
| Mobile/web app end-users | Amazon Cognito (User Pools + Identity Pools) |

**The key insight:** Federation never creates IAM users — external identities assume roles to get temporary credentials.

---

## 🛡️ Security-First Takeaways

1. **Never use the root user** for daily work — enable MFA and lock it away
2. **Use roles, not access keys**, for any AWS service-to-service communication
3. **Apply least privilege** — grant only what's needed, nothing more
4. **Use groups for permissions** — never attach policies directly to users
5. **Multi-account strategy** for production — limits blast radius
6. **Always use External ID** when granting access to third parties
7. **Audit with CloudTrail** — every IAM action should be logged and monitored

---

## 📚 Quick Reference: Common Exam Patterns

| Scenario | Best Solution |
|----------|--------------|
| EC2 needs S3 access | IAM Role with Instance Profile |
| Grant temporary access to 3rd party | Cross-account role + External ID |
| Limit what a junior dev can grant | Permission Boundary |
| Restrict whole AWS account | SCP via AWS Organizations |
| On-prem AD users → AWS Console | IAM Identity Center / SAML |
| Mobile app users login | Cognito |
| Force complex passwords for all IAM users | Account password policy |
| Detect public S3 buckets / over-permissive roles | IAM Access Analyzer |

---

*Part of my AWS Solutions Architect Associate journey toward Cloud Security specialization.*
