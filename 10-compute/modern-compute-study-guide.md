# ✈️ Modern Compute Study Guide (Cluster 9)
### Lambda · Containers (ECS / Fargate / ECR) · When to use what

> Offline reading. Part 1 consolidates Lambda (including the things to nail from your quiz). Part 2 teaches containers from scratch. Part 3 is the decision framework the exam loves. Part 4 is a self-quiz — answers at the very bottom, no peeking.

---

# PART 1 — LAMBDA (consolidated)

## The one-line essence
**Lambda runs your code in response to an event, scales itself, and charges only while it runs.** No servers to manage; idle costs nothing.

Mental model: **EC2 is renting an apartment** (always yours, always paying, you maintain it). **Lambda is a vending machine** (sits idle for free, springs to life when triggered, does one job, goes quiet).

## Three properties that define it
1. **Event-driven** — a function doesn't run on its own; something *triggers* it (S3 upload, API call, queue message, schedule).
2. **Auto-scaling** — 1 request → 1 copy; 1,000 requests → 1,000 parallel copies. No Auto Scaling Group to configure.
3. **Pay per use** — billed per invocation + duration (in ms) × memory. Idle = $0.

## ⭐ Invocation patterns — PUSH vs POLL (your weak spot)
This is the single most testable Lambda detail. Memorize the split:

| Pattern | How it works | Sources |
|---|---|---|
| **Synchronous** | Caller waits for the response (request-reply) | API Gateway, ALB |
| **Asynchronous (PUSH)** | Source hands Lambda the event, doesn't wait; **Lambda auto-retries on failure** | **SNS, S3, EventBridge** |
| **Poll-based (POLL)** | Lambda *polls* the source and pulls **batches** | **SQS, Kinesis Data Streams, DynamoDB Streams** |

**The mnemonic that fixes it forever:**
> **Queues and streams are POLLED** (SQS, Kinesis, DynamoDB Streams) — they're buckets of messages sitting there, so Lambda reaches in and *pulls*.
> **Announcements and events are PUSHED** (SNS notification, S3 upload, EventBridge) — they *hand* Lambda the event.

⚠️ **SNS is PUSH, not poll.** SNS *pushes* notifications out (that's its whole job — pub/sub). Don't group it with SQS. SQS is a queue → polled. SNS is a notifier → pushes.

## ⭐ Async failures → retries → DLQ (your weak spot)
For **asynchronous** invokes (SNS, S3):
- Lambda **automatically retries** — 2 additional attempts.
- Events that *still* fail go to a **Dead-Letter Queue (DLQ)** or an **on-failure destination**, so nothing is lost.
- (Same DLQ safety-net concept from SQS — a poison event doesn't vanish, it's parked for inspection.)

## ⭐ Concurrency — reserved vs provisioned (your weak spot)
Three things, don't mix them up:
- **Account default** — ~1,000 concurrent executions per account (soft limit, raisable).
- **Reserved concurrency** — sets a **cap / guarantee** on how many copies of *one specific function* can run. A limit. (Nothing to do with warming or CPU.)
- **Provisioned concurrency** — keeps a set number of environments **pre-warmed** → **eliminates cold starts** for latency-sensitive functions.

> Burn this in: **cold start problem → PROVISIONED concurrency.** Reserved is just a cap.

## Cold starts
When a function hasn't run recently, AWS must spin up a fresh environment (load code, start runtime) *before* running — that startup delay is the **cold start** (~100ms to a couple seconds). Once warm, the next invocations reuse the environment and are fast. Analogy: a food truck firing up the grill for the first order of the day; after that, orders fly out. Fix for latency-sensitive work: **provisioned concurrency**.

## Limits to memorize
- **Max runtime: 15 minutes.** ⭐ The #1 exam trap — any task longer than 15 min means **Lambda is the WRONG answer** (use containers or EC2).
- **Memory: 128 MB → 10 GB**, and **CPU scales with memory.** So to make a function *faster*, raise its memory. (You only set memory; you never set CPU directly.)
- **Ephemeral `/tmp` storage:** 512 MB default, up to 10 GB.
- **Deployment package** size limits (zipped/unzipped; container image up to 10 GB).

## Common patterns
- **S3 upload → Lambda** processes the file (resize image, scan document).
- **API Gateway → Lambda → DynamoDB** = a fully serverless API (no servers anywhere — DynamoDB is serverless too).
- **EventBridge schedule (cron) → Lambda** = scheduled jobs.
- **Security automation:** detection (GuardDuty/Config/CloudWatch alarm) → **SNS/EventBridge** → **Lambda** auto-remediates (no human in the loop).

## Exam triggers
- Short, event-driven, spiky/unpredictable work → **Lambda**
- Task > 15 min, or needs full OS/control → **containers or EC2**
- "Can't tolerate cold-start latency" → **provisioned concurrency**
- "Cap how much a function can scale" → **reserved concurrency**
- Fully serverless REST API → **API Gateway + Lambda + DynamoDB**

---

# PART 2 — CONTAINERS (ECS / Fargate / ECR)

This is the "**what if it doesn't fit Lambda?**" answer — longer-running, more control, but still no full-server management hassle.

## What's a container? (easy mode)
A **container** packages your app *plus everything it needs to run* (libraries, runtime, config) into one portable bundle. It runs the same on your laptop, in test, and in production — no "works on my machine" problems.

Analogy: a **shipping container**. Standardized box; doesn't matter what's inside or which ship/truck/crane handles it — it just fits everywhere. Docker is the tech that makes these boxes.

**Container vs VM (EC2):** a VM virtualizes a whole machine (its own OS — heavy, slow to boot). A container shares the host OS kernel and only packages the app (lightweight, starts in seconds, you pack many per host). Containers = faster, denser, more portable.

**Container vs Lambda:** Lambda is even more hands-off but limited (15 min, specific runtimes). Containers run *anything*, for *as long as you want*, with full control of the environment — at the cost of a bit more setup.

## The three AWS pieces

### 1. ECR — Elastic Container Registry
Where your container **images** live. Think "private Docker Hub inside AWS." You build an image, push it to ECR, and your orchestrator pulls it from there to run. (Registry = storage for images.)

### 2. ECS — Elastic Container Service
AWS's **orchestrator** — it runs and manages your containers: how many copies, where they run, restarting failed ones, connecting them to load balancers. Key terms:
- **Task definition** — the blueprint (which image, how much CPU/memory, ports, env vars). Like a launch template, but for containers.
- **Task** — a running instance of a task definition (one or more containers).
- **Service** — keeps a desired number of tasks running (like an ASG for containers) and ties them to a load balancer.
- **Cluster** — the logical group where tasks run.

### 3. The two ways ECS runs your containers (THIS is the exam point)

| | **EC2 launch type** | **Fargate launch type** |
|---|---|---|
| Who manages the servers? | **You do** — you run a fleet of EC2 instances, patch them, scale them | **AWS does** — no servers to see or manage |
| Billing | Pay for the EC2 instances (running 24/7) | Pay only for the vCPU/memory your tasks use |
| Control | Full control of the host | No host access; AWS abstracts it |
| Best when | You need host-level control, special instance types (GPU), or have steady high utilization | You want **serverless containers** — just run the container, forget the infrastructure |

> ⭐ **Fargate = serverless containers.** Same idea as Lambda (no servers to manage, pay for use) but for full containerized apps with no 15-minute limit. This is the most-tested container concept: *"run containers without managing servers"* → **Fargate.**

## EKS — one line
**EKS = Elastic Kubernetes Service** — managed Kubernetes, for teams already standardized on Kubernetes or needing its ecosystem/multi-cloud portability. ECS is AWS-native and simpler; EKS is Kubernetes and more portable but more complex. Exam: *"we use Kubernetes"* → **EKS**.

## Exam triggers
- "Run containers **without managing servers/infrastructure**" → **Fargate**
- "Run containers but we need host control / special instances / steady load" → **ECS on EC2**
- "We already use **Kubernetes**" → **EKS**
- "Store our container images in AWS" → **ECR**
- "Long-running app (> 15 min) but don't want to manage servers" → **Fargate** (not Lambda!)

---

# PART 3 — DECISION FRAMEWORK: EC2 vs Lambda vs Containers

Walk it top-down:

```
Is the work event-driven, short (< 15 min), and spiky?
        │
        ├── YES → LAMBDA (serverless functions, pay-per-ms)
        │
        └── NO → Is it a containerized app?
                    │
                    ├── YES → ECS / Fargate
                    │          ├── No servers to manage?     → FARGATE
                    │          ├── Need host control/GPU?    → ECS on EC2
                    │          └── Already use Kubernetes?    → EKS
                    │
                    └── NO → Need a full OS / legacy app / full control?
                                  → EC2
```

**The "no server management" ladder (least → most hands-off):**
EC2 (you manage the server) → ECS on EC2 (you manage the hosts) → **Fargate** (no servers, containers) → **Lambda** (no servers, just functions).

**One-liners:**
- **EC2** = full control, you manage everything, always-on billing.
- **ECS on EC2** = containers, but you still run the host fleet.
- **Fargate** = containers, *no* servers — serverless containers.
- **Lambda** = functions, *no* servers, event-driven, 15-min cap, pay-per-ms.

---

# PART 4 — SELF-QUIZ (answers at the bottom)

1. A job runs for 40 minutes and you don't want to manage servers. Lambda or Fargate? Why?
2. Which three sources does Lambda **poll**? Why isn't SNS one of them (one word)?
3. Cold-start latency is hurting a user-facing function. Which concurrency type fixes it?
4. What does ECR do?
5. In ECS, what's the difference between the **EC2 launch type** and the **Fargate launch type**?
6. An async (SNS-triggered) Lambda keeps failing. What happens automatically, and where do dead events go?
7. You only set a Lambda's memory. How does it get CPU?
8. "We've standardized on Kubernetes across clouds." Which AWS service?
9. Build a fully serverless REST API. Name the three services.
10. Reserved concurrency does what — and is it the same as provisioned? (No.)

---
---

## ✅ ANSWERS (no peeking until you've tried)

1. **Fargate.** 40 min exceeds Lambda's 15-min hard limit, so Lambda is out. Fargate runs containers for any duration with no server management — the "serverless but long-running" answer.
2. **SQS, Kinesis Data Streams, DynamoDB Streams.** SNS isn't polled because it **pushes** (it's a notifier/pub-sub — it hands events out, Lambda doesn't reach in for them).
3. **Provisioned** concurrency (keeps environments pre-warmed). Not reserved — reserved is just a cap.
4. **ECR (Elastic Container Registry)** stores your container **images** — a private Docker registry in AWS.
5. **EC2 launch type:** you run and manage the underlying EC2 host fleet (patch, scale, pay for instances). **Fargate launch type:** AWS manages the infrastructure — serverless containers, pay only for the vCPU/memory your tasks use, no hosts to see.
6. Lambda **auto-retries** (2 more attempts). Events that still fail go to a **dead-letter queue (DLQ)** or on-failure destination.
7. **CPU scales proportionally with memory.** Raise memory to get more CPU (and a faster function). You never set CPU directly.
8. **EKS (Elastic Kubernetes Service)** — managed Kubernetes.
9. **API Gateway + Lambda + DynamoDB** (all serverless — no servers anywhere).
10. **Reserved concurrency = a cap/guarantee on how many concurrent executions one function can use.** It is **not** the same as provisioned. Provisioned = pre-warmed environments to kill cold starts; reserved = a scaling limit. Different jobs entirely.

---

*Your scorecard target: 8+/10. The ones to be airtight on are #1 (15-min ceiling), #2 (poll vs push), #3 & #10 (provisioned vs reserved), and the Fargate = serverless-containers idea. Those four show up the most.*

**Safe travels — when you're back, we'll finish the ECS hands-on bit and move to Cluster 10 (Monitoring + Security), which is the heart of your Cloud Security track.**
