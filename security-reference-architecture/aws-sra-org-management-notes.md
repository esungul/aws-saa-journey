# 🏛️ AWS Security Reference Architecture (SRA) — Org Management & Governance

> Study notes from working through the **AWS Security Reference Architecture** (AWS Prescriptive Guidance) section by section. The SRA is the real-world, AWS-published blueprint for security in a **multi-account** environment — built around a simple 3-tier app with security services layered on every account. Doubles as SAA governance prep **and** AWS Security Specialty / cloud-security career material.
>
> Source: `docs.aws.amazon.com/prescriptive-guidance/latest/security-reference-architecture/`

---

## 🗺️ The big picture

The SRA organizes everything under **AWS Organizations**:
- **Org Management account** (root of the org, payer) — locked down, minimal.
- **Security OU** → **Log Archive account** (central immutable logs) + **Security Tooling account** (delegated admin for security services).
- **Infrastructure OU** → **Network account**, **Shared Services account**.
- **Workloads OU** → application accounts (the actual workloads).

Core principles throughout: **least privilege · separation of duties · delegated administration · centralized logging · secure-by-default account onboarding.**

---

## 1. The Org Management account

**What it is:** the account used to *create* the Organization (your original AWS account). It's the **payer** (consolidated billing) and the place you manage **OUs** and attach **SCPs**.

**Why it's special / the gotcha:** it's the most powerful account in the org, **and SCPs do not restrict it.** So your org-wide guardrails can't protect it — which is exactly why it must be locked down.

**Best practices (keep these):**
- Use a **business-managed email**; keep accurate admin/security contacts (incl. a phone number).
- **MFA for all users**; regularly review who has access.
- **No workloads, few people** — use it only for org management.
- **Delegate** security functions to dedicated accounts rather than running them here.
- Daily work as an **IAM admin**, never the root user.

---

## 2. Services in the management account — triaged for SAA

### 🟢 Concentrate (high-yield)

**Service Control Policies (SCPs)** — created/applied *from* the management account; define which service APIs IAM principals in **member** accounts can use. **No effect on the management account.** Set a *ceiling* (restrict only, never grant); both SCP **and** IAM must allow an action. Control Tower deploys SCPs as preventive guardrails (mandatory / strongly recommended / elective).

**IAM Identity Center** — centralized workforce SSO to all accounts + SaaS apps. Runs in the management account by default, but **delegate its admin to a member account** (SRA: Shared Services account) for least privilege. Integrates with an **external SAML 2.0 IdP** or Microsoft AD (via Directory Service); use **SCIM** to auto-sync users/groups with HR (joiner/mover/leaver). Only one IdP/directory connected at a time.

**AWS Control Tower** — sets up & governs a secure multi-account **landing zone**. Orchestrates Organizations + Service Catalog + IAM Identity Center, using **CloudFormation (baseline) + SCPs (prevent) + Config rules (detect)**. **Account Factory** provisions new compliant accounts in a few steps. Aligns with Well-Architected security design principles.

**Distributed & centralized security guardrails** — Security Hub, GuardDuty, Config, IAM Access Analyzer, CloudTrail org trails, Macie deployed across all accounts via a **delegated administrator** with centralized monitoring. Part of account onboarding/baselining. (Detective + Audit Manager = optional, for dedicated forensics/audit teams.)

### 🟡 Know at a glance

- **AWS Artifact** — on-demand AWS **compliance reports** (SOC, PCI) + agreements (**HIPAA BAA**). In the management account because accepted agreements flow down to members. *Reflex: "where do I get AWS audit/compliance reports?" → Artifact.*
- **Resource Control Policies (RCPs)** — counterpart to SCPs: SCPs cap what **identities** can do; RCPs cap what can be done **to resources**. Member accounts only; subset of services.
- **Centralized root access** — newer IAM feature: remove long-term root creds from member accounts; short-term scoped root sessions; delegated to Security Tooling account.

### ⚪ Skim (lean Security-Specialty)

- **Declarative policies** — enforce EC2/VPC/EBS config at scale (e.g., IMDSv2, block public EBS snapshots), enforced in the control plane.
- **IAM access advisor** — service-last-accessed data; detective control for least-privilege right-sizing.
- **Systems Manager (Quick Setup / Explorer)** — org-wide patching/ops dashboards.

---

## 🔴 Governance best-practices keeper

```
Management account  → org management only; no workloads; few people; MFA; locked down
                      (most powerful + SCPs can't restrict it)
Root user           → lock away, MFA on, use only for tasks that require it; daily work = IAM admin
SCPs / RCPs         → ceiling only (restrict, never grant); member accounts only; SCP AND IAM must allow
OUs                 → logical folders; group accounts to apply guardrails at scale
IAM Identity Center → workforce SSO; DELEGATE admin out of mgmt account; external IdP + SCIM sync
Control Tower       → landing zone = CloudFormation + SCPs + Config; Account Factory for new accounts
Delegated admin     → run GuardDuty/Security Hub/Config/Macie from a Security Tooling account
AWS Artifact        → AWS compliance reports (SOC/PCI) + HIPAA BAA
```

---

## ✅ Consolidation check

SRA governance best-practices quiz: **8/8.** Solid command of management-account hardening, SCP behavior, delegation, Control Tower, and where compliance evidence lives.

---

*Part of my AWS SAA journey toward Cloud Security / AI Security. The SRA is the bridge from "pass the SAA" to "architect secure multi-account AWS" — directly on the SCS-C02 → CCSP path. Next SRA sections: Security Tooling account, Log Archive account, then Infrastructure & Workloads OUs.*
