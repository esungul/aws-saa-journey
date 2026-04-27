# 🛡️ AWS SAA Journey — Building Toward Cloud Security

> My public learning journey from **15+ years of Telecom IT Operations** to **AWS Cloud Security Engineer / DevSecOps**.

---

## 🎯 The Goal

I'm transitioning into cloud security with a deliberate certification and portfolio path:

| Stage | Certification | Status |
|---|---|---|
| 1 | **AWS Solutions Architect Associate (SAA-C03)** | 🟡 In Progress |
| 2 | AWS Security Specialty (SCS-C02) | ⏭️ Planned |
| 3 | (Optional) AWS Solutions Architect Professional | ⏭️ Future |

**Timeline:** 12-18 months of consistent execution.

---

## 🧭 Learning Principles

1. **"Behave like an architect, not sound like one"** — Real understanding through reasoning, not memorization
2. **Layered learning:** Theory (Udemy) → Depth (research + AI-assisted exploration) → Mastery (hands-on labs)
3. **Security-first lens** on every topic — *"What's the misconfiguration risk? Attack vector? Detection method?"*
4. **Stay focused** — depth over breadth, no rabbit holes
5. **Quality over speed** — consistency beats cramming

---

## 📚 What's In This Repo

This isn't a tutorial collection — it's my **public learning journal**, including:

- 📝 **Notes** with architect-level reasoning, not just facts
- 🛠️ **Hands-on labs** I've actually built (with bugs I encountered + how I fixed them)
- 🛡️ **Security-first analysis** of each topic
- 🧠 **Mental models** I've developed to make concepts stick

---

## 🗂️ Topics Covered

### ✅ Completed

#### 🔐 [IAM (Identity & Access Management)](./01-iam/)
- IAM fundamentals (users, groups, roles, policies)
- Cross-account role assumption (with real debug story)
- STS temporary credentials
- Multi-account strategy
- Least privilege & blast radius concepts

#### 🖥️ [EC2 (Elastic Compute Cloud)](./02-ec2/)
- Instance types and families
- Pricing models (On-Demand, Reserved, Spot, Dedicated)
- AMI security considerations

### 🟡 In Progress
- EC2 Security Groups
- EBS Storage

### ⏭️ Coming Soon
- VPC & Networking
- S3 & Storage Services
- Database Services (RDS, DynamoDB)
- Federation & SSO (SAML, IAM Identity Center, Cognito)
- Policy Evaluation Logic (SCPs, Permission Boundaries)

---

## 🌟 Featured Lab: Cross-Account IAM Role Debug

In my [Cross-Account Roles Lab](./01-iam/cross-account-roles-lab.md), I document a real production-grade debugging experience — including:
- The trust policy bug I introduced
- How I read AWS error messages systematically
- The "two-door" model for cross-account access
- Why AWS uses deliberately vague error messages (security)

This kind of hands-on debugging is what cloud security engineers do every day. 🎯

---

## 📅 Progress Log

| Week | Focus | Highlights |
|---|---|---|
| Week 1 | IAM + EC2 fundamentals | Cross-account role lab, first EC2 instance |

---

## 🤝 Connect

- **LinkedIn:** [Your LinkedIn URL]
- **Location:** Panama City, Panama 🇵🇦
- **Background:** 15+ years in Telecom IT Operations

---

*Public learning is the fastest way to learn. If you find anything useful, feel free to star ⭐ this repo or reach out. Always open to feedback from cloud and security professionals.*