# 📊 Cluster 10a — CloudWatch, CloudTrail & Config (Monitoring + Security)

> The monitoring/observability foundation for the Cloud Security track: CloudWatch (metrics, alarms, logs, EventBridge), and the "three C's" distinction that the exam tests constantly — backed by a hands-on lab where a real security misconfiguration is caught by all three services.

---

## ☁️ CloudWatch — the four pillars

CloudWatch = the **monitoring & observability** service ("how is my system performing?"). The car-dashboard of AWS.

1. **Metrics** — numerical data over time (CPU, network, request counts). Organized by **namespace** (`AWS/EC2`, `AWS/Lambda`…). The needles on the dashboard.
2. **Alarms** — watch a metric, trigger an action when it crosses a threshold. The warning lights.
3. **Logs** — centralized text logs in **log groups** (per source) and **log streams** (per instance/execution).
4. **Dashboards** — visual panels combining metrics.

```
Services ─emit→ METRICS ─watched by→ ALARMS ─trigger→ SNS / Auto Scaling / Lambda
                                        ▲
LOGS ─(metric filter turns pattern into)┘
```

### ⭐ The EC2 memory gotcha (high-frequency exam item)
Default EC2 metrics: **CPU, network, disk I/O, status checks.** **NOT** included: **memory utilization** and **disk-space-used** — because those live *inside* the guest OS, and the hypervisor (which publishes default metrics from *outside*) can't see in. Fix: install the **Unified CloudWatch Agent** inside the instance → publishes memory/disk as **custom metrics** (and can ship logs too). The older "CloudWatch Logs Agent" only ships logs; the **Unified Agent** does logs + system metrics.

### Alarms — anatomy & actions
- **States:** OK · ALARM · INSUFFICIENT_DATA.
- **Definition:** metric + statistic + period + threshold + evaluation periods (e.g., "avg CPU > 70% for 3 consecutive 5-min periods").
- **Actions:** **notify** (SNS) · **scale** (Auto Scaling) · **remediate** (EC2 stop/terminate/reboot/**recover**).
- **Composite alarm** — combine alarms with AND/OR so you're only paged when *both* conditions are true (reduces alarm fatigue).
- **Recover action**: an EC2 status-check alarm can auto-recover an impaired instance.

### ⭐ Metric filters — the bridge from logs to security alerts
A **metric filter** scans incoming log events for a pattern (e.g., `UnauthorizedOperation`, root usage, `ERROR`) and increments a metric — which you can then **alarm** on.
```
Log events ─metric filter (matches pattern)→ metric ─→ alarm ─→ SNS
```
The CIS AWS Foundations Benchmark prescribes exactly this for root usage, unauthorized API calls, sign-in failures, security-group changes. **This is bread-and-butter Cloud Security work.**

### Other CloudWatch pieces
- **Logs Insights** — query language to interactively search/analyze logs.
- **Subscription filter** — stream logs in real time to Lambda/Kinesis/OpenSearch.
- **Synthetics (Canary)** — a script that **proactively** hits an endpoint on a schedule (like a simulated user) and checks it returns 200/loads correctly. Catches outages *before* users or metrics do. (Alarm = reactive watch; **canary = proactive test**.)
- **Live Tail** — real-time streaming view of log events (cloud `tail -f`).
- **Basic vs detailed monitoring** — EC2 default 5-min (basic, free); detailed = 1-min (small cost).

---

## 🔔 EventBridge — react to *events* (not metrics)

Formerly CloudWatch Events. A serverless **event bus**: AWS services (and apps) emit **events**; **rules** match them by **event pattern** or **schedule (cron)**; matched events go to **targets** (Lambda, SNS, SQS, Step Functions…).

- **Alarm** watches a **metric** (a number crossing a threshold).
- **EventBridge** reacts to an **event** (a discrete thing that happened — an API call, a GuardDuty finding, a state change, a Config compliance change).

Core security pattern: `GuardDuty finding → EventBridge rule → Lambda (auto-remediate) / SNS (alert)`. Also the AWS way to run **scheduled/cron jobs**: `EventBridge schedule → Lambda`.

---

## 🆚 The "Three C's" — one question each (exam gold)

| | **CloudWatch** | **CloudTrail** | **Config** |
|---|---|---|---|
| Question | How is it **performing**? | **WHO** did what, when? | What's the **config** / is it **compliant**? |
| Data | Metrics, logs, alarms | API call history | Resource config state + compliance |
| Car analogy | Dashboard | Black box / trip log | Inspection record |
| Example | CPU 80%, 500 err/min | "sunny opened port 22 at 3 PM from IP x" | "SG allows 0.0.0.0/0:22 → NON-COMPLIANT" |

**Memory hooks:** CloudWat**CH** = wat**CH** performance · CloudTr**AIL** = audit **trail** of who did what.

**⚠️ My weak spot (corrected):** anything asking **"WHO" / "which user" / "who made the change/deletion" → ALWAYS CloudTrail.** Config tracks *state* and *that it changed*, but never the *identity*. (I wrongly reached for Config twice — fixed now.)

**CloudTrail vs Config (the subtle pair):** CloudTrail = the **actor/action** (who made the API call); Config = the **resulting state + compliance** (what the config is now, how it drifted). They complement: Config detects the bad state, CloudTrail names the culprit, CloudWatch sounds the alarm.

**AWS Config does three things:** (1) records **configuration history** of resources over time; (2) **Config Rules** evaluate resources vs desired state and flag NON-COMPLIANT; (3) **auto-remediation** (SSM/Lambda) can fix violations.

---

## 🛠️ Hands-On Lab: "Catch a Misconfiguration with the Three C's"

Scenario: someone opens SSH (22) to `0.0.0.0/0`. Built a real detection-and-response stack and watched each service play its role.

**Part 1 — Config detects the bad state.** Enabled AWS Config + managed rule **`restricted-ssh`**, created `c-lab-sg` (compliant), then opened port 22 to the world → Config flipped it to **NON-COMPLIANT** 🚩. *Proved: Config = compliance/state, doesn't care who.*

**Part 2 — CloudTrail names the culprit.** In CloudTrail **Event history**, filtered Event name = **`AuthorizeSecurityGroupIngress`** → found the event showing **`sunny-admin`** (me), the **timestamp**, the **source IP**, and the request params (port 22, 0.0.0.0/0). *Proved: WHO = CloudTrail. Seeing my own name made the distinction permanent.*

**Part 3 — EventBridge alerts in real time.** Created an SNS topic + email sub, then an **EventBridge rule** on `aws.config` → `Config Rules Compliance Change` (NON_COMPLIANT) → SNS. Created a non-compliant `c-lab-sg-2` → **email arrived**. *Proved: a discrete event (became non-compliant) → EventBridge, not an alarm (no metric/threshold).*

**Part 4 — Metric filter + alarm.** Set up CloudTrail → CloudWatch Logs, a **metric filter** matching `{ ($.eventName = "AuthorizeSecurityGroupIngress") }` → metric `SecurityLab/SGIngressChanges`, and an **alarm** (Sum ≥ 1) → SNS. *Proved: log-pattern → count → alarm. The other alerting path.* *(Note: highest latency path, 5-15 min CloudTrail→Logs→alarm.)*

**The two alerting paths, contrasted:** Part 3 (EventBridge) matched an *event* directly; Part 4 (metric filter + alarm) scanned *log text* and alarmed on a *count crossing a threshold*. Exactly the Q3-vs-Q4 distinction.

---

## 🔴 Exam Triggers

```
Performance / CPU / logs / alarm / dashboard            → CloudWatch
"Who did it" / which user / audit actions / forensics   → CloudTrail  (ALWAYS)
Resource config changes / drift / compliance            → Config
Monitor EC2 MEMORY/disk-used                            → install Unified CloudWatch Agent
Alert on a string in logs                              → metric filter + alarm
Continuously check resource compliance                  → Config Rules (+ auto-remediation)
Alert only when BOTH conditions true                    → composite alarm
Proactively test a website like a user                  → Synthetics canary
React to an event/finding instantly                     → EventBridge rule
Scheduled/cron job                                      → EventBridge schedule → Lambda
Reduce CloudWatch Logs cost                             → set log group retention
```

---

## 🧹 Cleanup pending (tomorrow)
To tear down: `c-lab-sg` + `c-lab-sg-2`, AWS Config (stop recorder + delete rule, empty/delete config S3 bucket), EventBridge rule, SNS topic, CloudTrail `c-lab-trail` (+ its S3 bucket), CloudWatch log group + metric filter + alarm. All tiny cost; tear down to keep at ~$0.

---

*Part of my AWS SAA journey toward Cloud Security Engineering. The lab turned the three-C distinction — especially "who = CloudTrail," my weak spot — from a flashcard into something I built and watched fire.*

**Cluster 10a: CloudWatch / CloudTrail / Config — TAUGHT ✅ · QUIZZED ✅ · LAB BUILT ✅. Next: 10b detection suite, 10c KMS/Secrets.**
