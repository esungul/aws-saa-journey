# 🏥 Project 2: HIPAA Platform — Build Results & Debugging Log

> The actual build of the HIPAA-compliant healthcare platform: what was deployed, the CTO questions answered with live proof, two real debugging war stories, and the lessons. Built, validated, and torn down in one session for ~$2-3. Companion to `hipaa-project-design.md` (the design) and `hipaa-implementation-guide.md` (the step-by-step).

---

## 📋 What Was Built

A working 3-tier, multi-AZ, HIPAA-aligned architecture in `us-east-1`, then fully decommissioned.

```
Internet
   │
   ▼
Application Load Balancer (public subnets, both AZs)   ← hipaa-alb
   │  forwards inward
   ▼
EC2 Auto Scaling Group (private app subnets)           ← hipaa-web-asg
   ├─ 2 instances, one per AZ
   ├─ IAM role (no keys), encrypted EBS
   └─ Session Manager access (no SSH)
   │
   ▼  (designed; isolation proven live)
RDS / data tier (private data subnets, no internet route)

Storage:  S3 patient-records (SSE-KMS) + logs bucket
Audit:    CloudTrail (mgmt + S3 data events) + VPC Flow Logs
Keys:     KMS Customer Managed Key (hipaa-data-key)
```

**VPC:** `10.0.0.0/16`, 6 subnets across us-east-1a/1b (2 public, 2 private-app, 2 private-data), IGW, NAT gateway, 3 route tables, 4 security groups.

---

## ✅ CTO Questions Answered — With Live Proof

Each was validated during the build and captured as a screenshot.

| # | CTO Question | Proof | Status |
|---|---|---|---|
| 1 | How is patient data encrypted at rest? | S3 default encryption = SSE-KMS with CMK; Block Public Access on | ✅ |
| 2 | Encrypted in transit too? | Bucket policy denies any non-HTTPS request (`aws:SecureTransport=false`) | ✅ |
| 5 | Can anyone reach the database from the internet? | `private-data-rt` has only the local route — no internet path | ✅ |
| 7 | Who can reach the database? | `rds-sg` allows only 3306 from `app-sg`, no outbound | ✅ |
| — | Servers access data without stored credentials? | `hipaa-ec2-role` (SSM + CloudWatch + scoped S3/KMS) — temp creds only | ✅ |
| — | Admins log in securely? | Session Manager (keyless, no bastion, fully logged) | ✅ |
| — | Compute self-healing + HA? | ASG desired=2, one per AZ, ELB health checks, auto-replace | ✅ |
| — | Public entry point resilient? | ALB internet-facing across both AZs | ✅ |
| — | App live + architecture sound? | Public ALB served page from private-IP instance (10.0.11.140) | ✅ |
| 9 | Who accessed what, when — tamper-proof? | CloudTrail: mgmt + S3 **data events**, log-file validation on | ✅ |
| — | Network-level visibility? | VPC Flow Logs (All: accepts + rejects) → CloudWatch | ✅ |

**The whole HIPAA story, proven:** network isolation · encryption at rest + in transit · least-privilege access · keyless admin · high availability · full audit logging.

---

## 🐞 Debugging War Stories (the real learning)

These two issues are the most interview-valuable part of the project. Both were diagnosed by tracing symptoms to root cause, not guessing.

### War Story 1 — Auto Scaling launches failing on KMS

**Symptom:** ASG desired=2 but zero instances; Activity log showed each launch terminating with:
```
Client.InvalidKMSKey.InvalidState: The KMS key provided is in an incorrect state
```

**Misleading part:** the error says "incorrect state," implying the key was disabled. It wasn't — the key was `Enabled`.

**Root cause:** the launch template encrypts the EBS volume with the **customer-managed key (CMK)**. To create that encrypted volume, the **EC2 Auto Scaling service-linked role** (`AWSServiceRoleForAutoScaling`) must be allowed to use the key. The key's policy only listed `sunny-admin` as a user — the ASG role had no access, so KMS refused, the volume couldn't be created, and the instance was terminated.

**Fix:** added `AWSServiceRoleForAutoScaling` as a key user on `hipaa-data-key` (the critical permission being `kms:CreateGrant`), then bounced desired capacity 0→2.

**Lesson:** The default `aws/ebs` key works automatically; a **CMK does not** — AWS service principals must be explicitly granted access in the key policy. And `InvalidKMSKey.InvalidState` usually means a **permission** problem, not a literal key-state problem.

### War Story 2 — Everything healthy, but 502 + SSM offline (the sneaky one)

**Symptom:** instances launched, but the site returned **502 Bad Gateway** and **Session Manager showed Offline** with:
```
RequestError: send request failed
```

**Why it was hard:** every network control checked out perfectly —
- app subnets → `0.0.0.0/0 → NAT` ✓
- NAT's public subnet → `0.0.0.0/0 → IGW` ✓
- `app-sg` outbound = all traffic ✓

So routing, NAT, and security groups were all correct, yet nothing could reach the internet by name.

**Root cause:** the VPC had **DNS resolution DISABLED**. (DNS *hostnames* was enabled — a different setting.) With resolution off, instances couldn't resolve *any* hostname via the Amazon DNS resolver. So the SSM agent couldn't resolve `ssmmessages.us-east-1.amazonaws.com`, and `yum` couldn't resolve the package repos → the web server never installed → 502.

**Fix:** enabled **DNS resolution** on the VPC, then replaced the instances (user-data only runs once at boot, so the broken instances couldn't self-heal — they had to be terminated and relaunched).

**Lesson:** `DNS resolution` and `DNS hostnames` are **two separate VPC settings**. Resolution off silently breaks SSM, yum, and all endpoint-by-name access *even when every route and security group is perfect*. A "send request failed" error points at the network layer, but the real culprit can be **name resolution**. Always check both DNS settings.

### Bonus — CloudTrail on an existing bucket + CMK

**Symptom:** `InsufficientEncryptionPolicyException` when pointing CloudTrail at the existing logs bucket encrypted with the CMK.

**Root cause:** same pattern as War Story 1 — the **CloudTrail service principal** wasn't granted access to the bucket or the CMK. CloudTrail only auto-configures permissions on a bucket it creates itself.

**Resolution (this build):** let CloudTrail create its own bucket and use SSE-S3 default encryption (logs still encrypted at rest). **Production note:** use the CMK and add `cloudtrail.amazonaws.com` to the key policy + bucket policy — the same service-principal-on-CMK pattern.

---

## 🧭 Architectural / Cost Decisions (documented, not accidental)

- **Single-AZ RDS instead of Multi-AZ:** Free-tier and cost-control choice. Production HIPAA uses Multi-AZ for automatic failover (99.95% uptime). The toggle is one click; the architecture is identical. Demonstrates cost-awareness.
- **RDS documented as design (not run live):** A console glitch blocked the instance-class dropdown. Since the database's *security posture* is proven by the network layer (route table isolation + `rds-sg`) regardless of whether MySQL is running, RDS was documented rather than fought. Encryption mechanism already proven live on S3 (same CMK, SSE-KMS).
- **CloudTrail logs with SSE-S3 (not CMK) for this build:** avoided extra key-policy work; production would use the CMK with `cloudtrail.amazonaws.com` granted.
- **No bastion host:** Session Manager for all admin access — keyless, logged, HIPAA-friendly.
- **HTTP (not HTTPS) listener on the ALB:** test build has no domain/ACM cert; production adds an HTTPS:443 listener with ACM. Architecture is otherwise identical.

---

## 🔴 Key Learnings to Remember

```
1. CMK access for service principals: EBS-via-ASG, CloudTrail, etc. must be
   explicitly granted in the KMS key policy. The aws/ managed keys are automatic;
   CMKs are not. kms:CreateGrant is the key permission for EBS.

2. InvalidKMSKey.InvalidState usually = a PERMISSION problem, not key state.

3. DNS resolution ≠ DNS hostnames. Two separate VPC toggles.
   Resolution OFF = nothing resolves by name (SSM, yum, endpoints all fail)
   even with perfect routing. "send request failed" can mean DNS, not network.

4. Health check passing ≠ app working. SG is stateful, so an inbound health
   check can pass even when outbound is broken. ASG "InService" ≠ TG "healthy".

5. User-data runs ONCE at boot. If the network was broken at launch, fixing it
   later doesn't fix those instances — terminate and relaunch.

6. 502 Bad Gateway = ALB reached the target but got a bad/no response
   (vs 503 = no healthy targets). Different signals, different debugging.

7. CloudTrail data events (not just management events) are the HIPAA-critical
   piece — they log object-level access to PHI ("who read which record").

8. CloudTrail only auto-configures permissions on a bucket IT creates. Point it
   at an existing bucket/CMK and you must add the policies yourself.

9. The database is protected two independent ways: no internet route (routing)
   AND security-group restriction (instance firewall) = defense in depth.

10. Cleanup discipline: delete in dependency order (ASG → ALB → NAT → EIP → ...).
    Verify $0 in Cost Explorer. KMS keys take a 7-day minimum to delete.
```

---

## 💰 Cost Summary

```
Meter-runners during build: NAT (~$0.045/hr), ALB (~$0.023/hr), 2× EC2 (~$0.02/hr)
Total session cost:         ~$2-3
Ongoing after cleanup:      $0 (KMS key ~$1/mo until 7-day deletion finalizes)
```

Cleanup verified in Cost Explorer — no ongoing hourly charges.

---

## 🎯 Interview Story (ready to tell)

> "I built a HIPAA-compliant healthcare platform on AWS — 3-tier VPC, SSE-KMS encryption with customer-managed keys, least-privilege IAM roles, Session Manager for keyless admin access, and full audit logging via CloudTrail data events and VPC Flow Logs. I validated each control with live test cases.
>
> The valuable part was the debugging. Auto Scaling launches were failing with `InvalidKMSKey.InvalidState` — which sounds like a key-state issue but was actually the Auto Scaling service-linked role lacking permission to use the CMK for EBS encryption. Then the app returned 502s with the SSM agent offline, even though every route, NAT, and security group was correct — the root cause was VPC DNS resolution being disabled, so nothing resolved by name. Both taught me that AWS error messages often point at the wrong layer, and that systematic root-cause tracing beats guessing.
>
> I also kept it cost-controlled — built, validated, and torn down for a few dollars, verifying zero ongoing charges in Cost Explorer."

---

*Part of my AWS Solutions Architect Associate journey toward Cloud Security Engineering. This document captures real build + debugging experience — the difference between knowing AWS concepts and operating AWS under pressure.*

**Project 2 HIPAA: BUILT ✅ · DEBUGGED ✅ · PROVEN ✅ · TORN DOWN ✅ · $0 ongoing ✅**
