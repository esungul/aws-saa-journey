# 🛡️ Cluster 10b — Detection Suite + Secrets (Monitoring + Security)

> AWS's managed security "detectives" — each watches a different domain — plus the secrets-storage distinction. This completes Cluster 10 alongside the 10a (CloudWatch/CloudTrail/Config) doc. Core of the Cloud Security Engineering track.

---

## 🗺️ The map: each service watches a different thing

Think of these as a security team with four specialists:

| Service | Watches… | One-liner |
|---|---|---|
| **GuardDuty** | Activity / behavior (logs) | Threat detection — "is something malicious happening?" |
| **Inspector** | Software (EC2, ECR, Lambda) | Vulnerability scanning — "what could be exploited?" |
| **Macie** | Data (S3) | Sensitive data discovery — "where is my PII/PHI?" |
| **Security Hub** | Everything (aggregates) | Single dashboard + compliance scoring |

> **GuardDuty = threats · Inspector = vulnerabilities · Macie = sensitive data · Security Hub = the dashboard over all of them.**

---

## 🔍 GuardDuty — intelligent threat detection

- **Agentless** — one-click enable, no software on instances. Independently reads **VPC Flow Logs + CloudTrail + DNS query logs** (and optionally EKS audit, RDS login, Lambda network, runtime agent).
- **How it detects:** machine learning / anomaly detection + built-in **threat intelligence** (known-bad IPs/domains) + behavioral patterns.
- **Findings (with severity):** compromised EC2 (crypto-mining, C2 beaconing, DDoS), compromised credentials (API calls from unusual IP/location), reconnaissance (port scans), comms with known-malicious IPs, S3 threats.
- **Detection, not prevention** — it tells you; you respond.
- **⭐ Auto-remediation pattern:** `GuardDuty finding → EventBridge rule → Lambda (quarantine instance) / SNS (alert)`. This is the exact pipeline I built by hand in the 10a lab.

**Triggers:** "ML threat detection," "compromised instances / malicious activity / crypto-mining," "analyze Flow Logs + CloudTrail + DNS, no agents."

---

## 🔧 Inspector — vulnerability scanning

- Scans three targets: **EC2 instances** (OS + software CVEs + network reachability), **ECR container images** (the engine behind "ECR image scanning"), and **Lambda functions** (code + dependencies).
- **Continuous & automated** — re-evaluates automatically when a **new CVE is published**, flagging every affected resource. Not a one-time scan. (EC2 uses the SSM agent; ECR/Lambda agentless.)
- **Finds:** software vulnerabilities (CVEs) + unintended network exposure. Each finding gets a **risk score** (CVSS-based) + remediation guidance.

### ⭐ Inspector vs GuardDuty (exam loves this pair)
| | **Inspector** | **GuardDuty** |
|---|---|---|
| Looks for | **Vulnerabilities** (weaknesses) | **Threats** (active behavior) |
| Question | "What *could* be exploited?" | "Is something bad *happening now*?" |
| Analogy | Inspector checking weak locks | Alarm catching the intruder |

> Inspector = the unlocked door *before* the break-in. GuardDuty = the break-in *as it happens*.

**Triggers:** "scan EC2/containers/Lambda for CVEs," "automated continuous vulnerability assessment," "unintended network exposure."

---

## 🔐 Macie — sensitive data discovery in S3

- All about **Amazon S3.** ML-powered **discovery + classification** of sensitive data: **PII** (SSNs, names, addresses), **PHI** (health data), **financial** (credit cards), **credentials** (keys in files).
- Uses **managed data identifiers** (built-in) + **custom data identifiers** (your own regex).
- Also evaluates **bucket posture** — flags public, unencrypted, or externally-shared buckets. Catches the danger combo: *a public bucket full of SSNs.*
- **HIPAA tie-in:** Macie would auto-discover the PHI in my `hipaa-patient-records` bucket and alert instantly if it ever went public/unencrypted. Core for HIPAA/GDPR/PCI ("know where sensitive data lives").

**Triggers:** "discover/classify PII/PHI in S3," "data privacy/compliance," "publicly exposed buckets with sensitive data." Shortcut: **Macie = S3 + sensitive data.**

---

## 🎛️ Security Hub — single pane of glass

- **Aggregates and normalizes** findings from GuardDuty, Inspector, Macie, Config, IAM Access Analyzer, Firewall Manager, and partner tools into **one dashboard** (standard AWS Security Finding Format). Doesn't detect on its own.
- **Runs compliance checks** against standards: **CIS AWS Foundations Benchmark, PCI-DSS, AWS Foundational Security Best Practices, NIST** — gives a compliance score + which controls fail.
- Multi-account via Organizations (delegated admin). Findings → EventBridge → auto-response.
- Analogy: the **SOC manager / mission control** where all specialists' reports land.

**Triggers:** "single/centralized dashboard of findings," "aggregate GuardDuty/Inspector/Macie," "automated compliance checks (CIS/PCI)," "overall security posture/score."

---

## 🔁 The unifying pattern

Every one of these feeds the same response pipeline (the one I built in the lab):
```
Any finding (GuardDuty / Inspector / Macie / Security Hub / Config)
        → EventBridge rule → Lambda (auto-remediate) / SNS (alert)
```
This is the heart of automated cloud security — and the SCS-C02 path.

---

## 🗝️ Secrets: Secrets Manager vs Parameter Store

| | **Secrets Manager** | **SSM Parameter Store** |
|---|---|---|
| Stores | Secrets (DB passwords, API keys) | Config values + secrets |
| **Rotation** | **Automatic built-in rotation** | No built-in rotation |
| Cost | Paid (per secret) | Free (standard tier) |
| Use when | You need rotation | General config / simple secrets, free |

> **"Automatic secret rotation" → Secrets Manager.** **"Free config/parameter storage" → Parameter Store.** Both encrypt with KMS.

---

## 🔴 Exam Trigger Quick-Reference

```
ML threat detection / compromised instances / crypto-mining      → GuardDuty
Scan EC2/ECR/Lambda for CVEs / continuous vuln assessment        → Inspector
Discover PII/PHI in S3 / data privacy / exposed sensitive data   → Macie
Centralized findings dashboard / compliance (CIS/PCI) scoring    → Security Hub
Automatic secret rotation                                        → Secrets Manager
Free parameter/config storage                                    → Parameter Store
Auto-respond to any security finding                             → EventBridge → Lambda
```

---

*Part of my AWS SAA journey toward Cloud Security Engineering. These detection services + the auto-remediation pattern are the foundation of the SCS-C02 / DevSecOps path ahead.*

**Cluster 10 COMPLETE ✅ — 10a (CloudWatch/CloudTrail/Config + lab) · 10b (detection suite) · KMS/Secrets. All exam content covered. Next: breadth fill-ins + practice exams.**
