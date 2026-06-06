# ⚡ Lambda Event Sources — Hands-On Lab (Cluster 9)

> Built a Lambda function and triggered it two ways — S3 (push) and SQS (poll) — then added a dead-letter queue and watched a failing message get caught. Along the way, uncovered the core IAM concept that explains *why* push and poll need permissions in completely different places. All free-tier; torn down after.

---

## 🎯 What I Built

A single Lambda (`lambda-lab-processor`, Python 3.12) that logs whatever event it receives, wired to two different event sources to feel the difference:

```
S3 bucket  ──(push / async)──►  ┌──────────────────────┐
                                │  lambda-lab-processor  │
SQS queue  ◄──(poll)────────────│  (logs the event)      │
   │                            └──────────────────────┘
   └─ fails 2× ──►  lambda-lab-dlq  (dead-letter queue)
```

---

## ✅ What I Proved Live

| Concept | How I proved it |
|---|---|
| **Push (async) trigger** | Uploaded a file to S3 → S3 *handed* Lambda an event containing the file name (`eventSource: aws:s3`). Source initiated. |
| **Poll trigger** | Sent a message to SQS → Lambda *pulled* it on its own polling schedule (`eventSource: aws:sqs`), with a slight delay (polling, not instant push). |
| **DLQ + maxReceiveCount** | Broke the function on purpose, sent a "poison" message → it failed, retried, and after 2 receives SQS **moved it to the DLQ** instead of looping forever or losing it. |
| **memory → CPU** | Saw the memory setting (128 MB) and that CPU scales with it. |

The push-vs-poll contrast is the thing the exam tests constantly, and now it's not abstract — I watched both happen with different event shapes in CloudWatch Logs.

---

## ⭐ The Big Concept: Push vs Poll → WHO makes the call → WHICH policy

This was the standout insight. When I added the **S3** trigger, no permission error. When I added the **SQS** trigger, I got `The function execution role does not have permissions to call ReceiveMessage`. Why the asymmetry?

**Because AWS always checks the permissions of whoever *initiates* the API call.**

### S3 (PUSH) — no error
- **S3 is the caller** — it *invokes* Lambda.
- So the permission needed is "S3 may invoke this function" → lives in a **resource-based policy attached to the Lambda** (a "Lambda permission").
- The console **auto-added** it. My execution role was never involved in *receiving* the event (the role only governs what the code does *after* it runs — here, writing logs, already covered). → nothing missing → no error.

### SQS (POLL) — error until fixed
- **Lambda is the caller** — it reaches into the queue and calls `ReceiveMessage`.
- So the permission needed is "this Lambda may read this queue" → lives in an **identity-based policy on the execution role**.
- The console added it but pointed the `Resource` at the **function ARN** instead of the **queue ARN**, so the role couldn't poll → error.

### The two policy types (core IAM)
- **Identity-based policy** — attached to a user/role: *"this identity can do X to resource Y."* (My execution role's SQS read policy.)
- **Resource-based policy** — attached to a resource: *"this principal may do X to me."* (The Lambda permission letting S3 invoke it.)

### The rule that makes permission errors stop being mysterious
```
PUSH (S3, SNS → Lambda):  the SOURCE calls Lambda
   → permission on Lambda's RESOURCE policy (often auto-added)

POLL (Lambda → SQS, Kinesis):  LAMBDA calls the source
   → permission on Lambda's EXECUTION ROLE (identity-based)
```
> Push = resource policy on the callee. Poll = identity policy on the caller. Always ask: **whose call is it?**

This is the *same* lesson as my HIPAA debugging, from the other direction: there, AWS services (Auto Scaling, CloudTrail) had to be named in a **resource-based policy** (the KMS key policy) to use my key. Here, my Lambda needed an **identity-based policy** to call SQS.

---

## 🐞 Lab Debugging Notes

**1. Code didn't run — no event in the logs.** First S3 invocation showed `INIT/START/END/REPORT` but *no print output*. Cause: I pasted new code but didn't click **Deploy** — Lambda ran the old default code. *Lesson: in Lambda, edits don't take effect until you Deploy.*

**2. SQS trigger failed on permissions — but the actions were right.** The auto-generated inline policy had the correct actions (`ReceiveMessage`, `DeleteMessage`, `GetQueueAttributes`) but the wrong **Resource** — it listed the *function* ARN instead of the *queue* ARN. Fixed the Resource to `arn:aws:sqs:<region>:<acct>:lambda-lab-queue`. *Lesson: IAM permissions are always **action + resource** — the right verb on the wrong noun still gets denied. Also: SQS ARNs have **no `queue/` segment** (the console hint was wrong); grab the exact ARN from the queue's Details tab.*

---

## 🔴 Key Lambda Takeaways (exam)

```
1. Max runtime 15 min. Longer task → containers/EC2, NOT Lambda. (#1 trap.)
2. You set MEMORY; CPU scales with it. Slow function? Raise memory.
3. Cold start fix = PROVISIONED concurrency (pre-warmed).
   RESERVED concurrency = a cap/guarantee, NOT a warmer. (Don't confuse.)
4. Invocation patterns:
     Sync (caller waits)        → API Gateway, ALB
     Async / PUSH (auto-retry)  → SNS, S3, EventBridge
     Poll (Lambda pulls batches)→ SQS, Kinesis, DynamoDB Streams
   Mnemonic: queues/streams are POLLED; notifications/events are PUSHED.
   SNS PUSHES — it is not polled.
5. Async failure → auto-retries (2×) → DLQ / on-failure destination.
6. Fully serverless API = API Gateway + Lambda + DynamoDB.
7. PUSH needs a resource-policy on Lambda; POLL needs an identity-policy
   on the execution role. Ask "whose call is it?"
```

---

## 🧹 Cleanup
Deleted: Lambda function, S3 lab bucket, both SQS queues (main + DLQ), the execution role. All free-tier; $0 impact.

---

*Part of my AWS SAA journey toward Cloud Security Engineering. This lab turned the push/poll distinction and the identity-vs-resource policy model from memorized facts into things I've actually operated and debugged.*

**Cluster 9 Lambda: TAUGHT ✅ · LAB BUILT ✅ · IAM CONCEPT PROVEN ✅ · TORN DOWN ✅**
