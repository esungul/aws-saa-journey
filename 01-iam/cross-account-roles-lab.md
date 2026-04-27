# 🔐 Lab: Cross-Account IAM Role Assumption

> **A real debugging story** — including the bug I introduced and how I fixed it. This is the kind of misconfiguration that happens in production.

---

## 🎯 Goal

Set up cross-account IAM access so a user in Account B can perform actions in Account A — **without sharing access keys** between accounts.

---

## 🏗️ Architecture

```
+-------------------------+         +-------------------------+
|   ACCOUNT A             |         |   ACCOUNT B             |
|   "Resource Owner"      |         |   "Consumer"            |
|                         |         |                         |
|   IAM Role:             | <-------|   IAM User: bob-dev     |
|   CrossAccountTestRole  |         |                         |
|                         |         |   Policy:               |
|   Trust Policy:         |         |   "Allow                |
|   "Account B can        |         |    sts:AssumeRole on    |
|    assume me"           |         |    Account A's role"    |
|                         |         |                         |
|   Permissions:          |         |   (No other AWS         |
|   IAMReadOnlyAccess     |         |    permissions)         |
+-------------------------+         +-------------------------+

Flow:
1. Bob calls sts:AssumeRole on Account A's role
2. STS validates trust policy + Bob's permissions
3. STS returns temporary credentials (ASIA... prefix)
4. Bob uses temp creds to access Account A
```

---

## 🔑 Key Concepts (Architect Mental Model)

### The "Two-Door" Model

Cross-account access requires **TWO doors to be unlocked**:

1. **Door 1 — Account A's Trust Policy:** *"Account B is allowed to assume me"*
2. **Door 2 — Account B's IAM Policy:** *"Bob is allowed to call AssumeRole"*

If **either** door is locked → access denied.

### Authentication vs Authorization

- **Authentication:** Who are you? (proven by valid credentials)
- **Authorization:** What can you do? (defined by IAM policies)

These are **separate gates**. Just because someone is authenticated doesn't mean they're authorized.

### Asymmetric Privilege

In this design, `bob-dev`:
- Cannot list users in his **home account** (Account B)
- Can list users in **Account A** (foreign account, via assumed role)

This is **least privilege done right** — Bob has zero direct permissions in his home account, gaining access only by *temporarily wearing a costume*.

---

## 🔧 Setup Steps

### Account A (Resource Owner)

**1. Create IAM Role:** `CrossAccountTestRole`

**Trust Policy** (replace ACCOUNT_B_ID with actual Account B ID):

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::ACCOUNT_B_ID:root"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

**Permissions Policy:** `IAMReadOnlyAccess` (AWS-managed) — used for testing only. In production, I would use a custom least-privilege policy.

### Account B (Consumer)

**2. Create IAM User:** `bob-dev` (no console access, CLI only)

**3. Attach inline policy** allowing only `sts:AssumeRole` on the specific role ARN:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::ACCOUNT_A_ID:role/CrossAccountTestRole"
        }
    ]
}
```

**4. Configure AWS CLI profile** for `bob-dev`:

```bash
aws configure --profile bob-dev
```

---

## 🐛 The Bug I Introduced

After setting everything up, running this failed:

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::ACCOUNT_A_ID:role/CrossAccountTestRole \
  --role-session-name bob-cross-account-test \
  --profile bob-dev
```

**Error:**

```
An error occurred (AccessDenied) when calling the AssumeRole operation:
User: arn:aws:iam::ACCOUNT_B_ID:user/bob-dev is not authorized to perform:
sts:AssumeRole on resource: arn:aws:iam::ACCOUNT_A_ID:role/CrossAccountTestRole
```

### Initial Diagnosis

The error implied Bob's IAM policy was missing or wrong. I verified Bob's policy was correctly attached and pointed to the right resource ARN. So the bug had to be on the other side.

### Root Cause

**Account A's trust policy was pointing to Account A itself**, not Account B:

```json
"Principal": {
    "AWS": "arn:aws:iam::ACCOUNT_A_ID:root"
}
```

Account A was effectively saying *"I trust myself"* — useless for cross-account access. This commonly happens because the AWS Console pre-fills the current account ID in the trust policy template.

### Fix

Updated the trust policy to point to Account B's ID:

```json
"Principal": {
    "AWS": "arn:aws:iam::ACCOUNT_B_ID:root"
}
```

---

## ✅ Verification (After Fix)

### Step 1: Confirm Bob has no direct IAM access in Account B

```bash
aws iam list-users --profile bob-dev
```

Result: `AccessDenied` — proves least privilege works correctly.

### Step 2: Bob assumes the role

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::ACCOUNT_A_ID:role/CrossAccountTestRole \
  --role-session-name bob-cross-account-test \
  --profile bob-dev
```

Result: Temporary credentials returned with:
- `AccessKeyId` starting with `ASIA` (signaling temporary STS credentials)
- `SessionToken` (proof of STS-issued temp credentials)
- `Expiration` timestamp (default 1 hour)

### Step 3: Use temp credentials to access Account A

```bash
export AWS_ACCESS_KEY_ID="ASIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_SESSION_TOKEN="..."

aws sts get-caller-identity
```

Output:

```json
{
    "UserId": "AROA...:bob-cross-account-test",
    "Account": "ACCOUNT_A_ID",
    "Arn": "arn:aws:sts::ACCOUNT_A_ID:assumed-role/CrossAccountTestRole/bob-cross-account-test"
}
```

I'm now operating in Account A, using Bob's identity from Account B.

```bash
aws iam list-users
```

Returns Account A's users — proving cross-account access works.

---

## 💡 Architect Insights From This Lab

### 1. AWS Error Messages Are Deliberately Vague

The error said *"is not authorized"* without specifying which side blocked the request. This is intentional — to prevent **information disclosure attacks**. If errors revealed which side was blocking, attackers could probe accounts to map trust relationships and identify valid user IDs.

### 2. Identity Type Indicators Matter

AWS uses different ARN prefixes to instantly signal credential type:

| Prefix | Meaning |
|--------|---------|
| AKIA   | Long-term IAM user access keys |
| ASIA   | Temporary STS session credentials |
| AROA   | IAM Role identifier |
| AIDA   | IAM User identifier |

Security tools and analysts use these prefixes to immediately identify credential type in CloudTrail logs.

### 3. Session Names Become Audit Trails

The `--role-session-name bob-cross-account-test` value appears in every CloudTrail event:

```
arn:aws:sts::ACCOUNT_A_ID:assumed-role/CrossAccountTestRole/bob-cross-account-test
```

In production, set meaningful session names (e.g., user identity + ticket number) for forensic clarity.

### 4. Why This Beats Sharing Access Keys

| Concern | Sharing Access Keys | Cross-Account Roles |
|---------|---------------------|---------------------|
| Credential lifetime | Permanent | Temporary (1hr default) |
| Blast radius if leaked | Full access until detected | Time-bounded |
| Revocation | Rotate keys + notify all | Delete role / detach trust |
| Audit trail | Murky | Clean (assumed-role ARN) |
| Governance | Shared secret | Each account governs independently |

---

## 🛡️ Security-First Takeaways

- Always use roles, never share access keys between accounts
- Enforce least privilege — Bob's only permission was `sts:AssumeRole` on one specific resource
- Temporary credentials limit blast radius — even if compromised, they auto-expire
- Set meaningful session names for audit trails
- Methodically check both "doors" when debugging cross-account access

---

## 🎯 What I Would Do Differently in Production

- Use a custom least-privilege policy instead of `IAMReadOnlyAccess`
- Add **External ID** condition for any third-party access scenarios (prevents confused deputy attack)
- Configure **CloudTrail** in both accounts to log the role assumption flow
- Set up **CloudWatch alerts** for unusual cross-account activity
- Use **IAM Identity Center** instead of static IAM users for individual employee access
- Implement **session duration limits** based on use case (default 1hr is reasonable for most cases)

---

*Built as part of my AWS Solutions Architect Associate journey, with a Cloud Security specialization focus.*
