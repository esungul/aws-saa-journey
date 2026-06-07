# 🐳 ECS & Fargate — Hands-On (Cluster 9, Compute Part 2)

> Learned containers from zero (container vs VM, image vs container, ECR, ECS, launch types), then ran a real nginx container on **Fargate** — serving a live page with zero servers managed — and tore it down. Companion to `modern-compute-study-guide.md`; this is the hands-on + the concepts internalized.

---

## 🧠 The Concepts (built up step by step)

**1. Why containers exist.** An app quietly depends on its surroundings (library versions, runtime, config) — move it and it breaks ("works on my machine"). A **container packages the app + everything it needs** into one portable bundle that runs identically everywhere. Analogy: a standardized **shipping container** — standardize the outside so the inside can be anything.

**2. Container vs VM.** A **VM (EC2)** carries its own **full OS** — heavy (GBs), boots in minutes, few per host. A **container shares the host's OS**, packaging only the app — light (MBs), starts in seconds, many per host. Analogy: VM = a house (own foundation/plumbing); container = an apartment (shares the building's infrastructure, private inside). VMs give stronger isolation; containers win on speed, density, portability.

**3. Image vs container.** Same pattern as **AMI → EC2 instance**:
- **Image** = the read-only blueprint (app + dependencies + config).
- **Container** = a running instance of that image.
- One image → many identical containers (like one AMI → many instances). The cookie cutter vs the cookies.

**4. ECR (Elastic Container Registry).** A **private Docker Hub inside AWS** — managed storage for container images. Flow: build → **push to ECR** → compute **pulls from ECR** → runs. IAM-controlled, encrypted, and supports **image scanning** for vulnerabilities (a real security feature — catch a bad library in the image before it runs).

**5. ECS (Elastic Container Service).** AWS's container **orchestrator** — runs/manages containers, restarts failures, scales, load-balances. Vocabulary maps onto EC2:

| EC2 world | ECS world | Role |
|---|---|---|
| Launch template | **Task definition** | The blueprint (image, CPU/mem, ports, role) |
| EC2 instance | **Task** | A running instance of the blueprint |
| Auto Scaling Group | **Service** | Keeps N tasks running, replaces failures, ties to LB |
| (capacity pool) | **Cluster** | The environment tasks run in |

**6. ⭐ EC2 launch type vs Fargate (most-tested).** Where do tasks actually run?

| | **EC2 launch type** | **Fargate launch type** |
|---|---|---|
| Who manages the servers? | **You** (patch, scale, pay 24/7) | **AWS** (invisible) |
| Billing | Pay for instances always | Pay per vCPU/memory used |
| Best for | Host control, GPUs, steady load | Simplicity, variable load, "just run it" |

> **Fargate = serverless containers.** Same philosophy as Lambda (no servers, pay-per-use) but for full containers with **no 15-minute limit**.

**The "how much do I manage?" ladder:**
`EC2 (whole server) → ECS on EC2 (host fleet) → Fargate (no servers, containers) → Lambda (no servers, functions)`

---

## ✅ Hands-On: Ran a Live Container on Fargate

Built, proved, and tore down:

1. **Cluster** — `fargate-lab-cluster` (AWS Fargate, serverless — no EC2 fleet registered).
2. **Task definition** — `fargate-lab-task`: image `public.ecr.aws/nginx/nginx:latest`, 0.25 vCPU / 0.5 GB, port 80, execution role to pull the image + log.
3. **Security group** — `fargate-lab-sg`: inbound HTTP 80 from 0.0.0.0/0 (so a browser can reach it).
4. **Ran the task** — Fargate, default VPC public subnet, public IP enabled. Status went `Provisioning → Pending → Running`.
5. **Proof** — opened `http://<task-public-IP>` → **"Welcome to nginx!"** served by a container, on **zero servers I launched, patched, or saw.**
6. **Teardown** — stopped the task (billing stops), deleted the cluster, SG. Fargate bills only while running → pennies.

> The "aha": I asked AWS to *run a container*, and it found compute, pulled the image, and served traffic — with no instance to manage anywhere. That's serverless containers in practice.

---

## 🐞 Debugging Lesson: ECS Service-Linked Role

**Symptom:** creating the cluster failed —
```
Unable to assume the service linked role. Please verify that the ECS service linked role exists.
```

**Root cause:** ECS needs the service-linked role **`AWSServiceRoleForECS`** to act on my behalf (for Fargate, it manages the task ENIs). It's normally auto-created on first ECS use, but didn't exist yet on this account.

**Fix (CloudShell):**
```bash
aws iam create-service-linked-role --aws-service-name ecs.amazonaws.com
```
Then retried the cluster — success.

**Lesson:** A **service-linked role** is a predefined, AWS-managed IAM role that grants a service exactly the permissions it needs to operate for you. I've now seen this pattern three times: **Auto Scaling** (`AWSServiceRoleForAutoScaling`, needed KMS access for encrypted EBS), **CloudTrail** (needed a grant on the CMK), and **ECS** (`AWSServiceRoleForECS`). Once you run managed services, these are everywhere — and the fix is "make sure the role exists," not "write a policy," because AWS defines the scope.

---

## 🔴 Exam Triggers (containers)

```
"Run containers without managing servers/infrastructure"   → FARGATE
"Need host control / GPUs / specialized instances / steady" → ECS on EC2
"Long-running app (>15 min), no server management"          → FARGATE (not Lambda)
"We use Kubernetes"                                          → EKS
"Store container images in AWS"                              → ECR
"Scan container images for vulnerabilities"                  → ECR image scanning
Fargate = serverless containers (memorize this phrase)
```

---

*Part of my AWS SAA journey toward Cloud Security Engineering. Containers went from flight-reading to fully understood and operated — including running one live on Fargate and debugging the service-linked-role gap.*

**Cluster 9 Compute: COMPLETE ✅ — Lambda (taught + lab) + Containers (taught + live Fargate run).**
