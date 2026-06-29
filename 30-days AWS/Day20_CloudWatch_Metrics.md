# Day 20: CloudWatch Metrics
### *(Seeing Inside Your Infrastructure — You Can't Fix What You Can't See)*

> **Roadmap reference:** Week 3, Day 20 — "CloudWatch Metrics"

---

## Why This Matters

You now have a full production-grade setup — VPC, EC2, RDS, S3, ALB, Auto Scaling. But right now you're flying blind. If your app starts slowing down, your database gets overloaded, or your instances are burning CPU for no reason — **you wouldn't know until users start complaining.**

CloudWatch is how AWS gives you eyes on your infrastructure. It's also the engine behind the Auto Scaling policies you set up yesterday — when you said "scale when CPU exceeds 50%," CloudWatch was the service measuring that CPU metric and triggering the action.

---

## 1. What is CloudWatch?

**Amazon CloudWatch is AWS's monitoring and observability service — it collects metrics, logs, and events from AWS resources and your applications, then lets you visualize, search, and alert on them.**

```
AWS Services (EC2, RDS, ALB, Lambda...)
        │
        │  automatically send metrics every 1-5 minutes
        ▼
   CLOUDWATCH
        │
        ├── Metrics    → numbers over time (CPU %, request count, etc.)
        ├── Logs       → text output (app logs, system logs) — Day 21
        ├── Alarms     → alert when a metric crosses a threshold
        ├── Dashboards → visual graphs of multiple metrics together
        └── Events     → trigger actions based on state changes
```

> **Key insight:** Many AWS services send metrics to CloudWatch automatically for free — you don't configure anything. EC2, RDS, ALB, Lambda all do this out of the box. You just need to know where to find and interpret those metrics.

---

## 2. What is a Metric?

**A metric is a time-series data point — a specific measurable value recorded at regular intervals.**

```
Metric example: EC2 CPUUtilization

  Time          CPU %
  10:00 AM  →   23%
  10:05 AM  →   45%
  10:10 AM  →   78%   ← something happened here
  10:15 AM  →   82%
  10:20 AM  →   41%
  10:25 AM  →   24%

Each data point = one observation at one moment in time.
Together they form a time-series graph that tells a story.
```

### Metric Anatomy

Every CloudWatch metric has three identifying properties:

```
Namespace  → which service owns this metric
             e.g., "AWS/EC2", "AWS/RDS", "AWS/ApplicationELB"

Metric Name → what is being measured
             e.g., "CPUUtilization", "DatabaseConnections"

Dimensions  → filters that narrow down WHICH resource
             e.g., InstanceId = "i-0abc1234"
             Without dimensions, a metric covers all resources
             of that type in your account.
```

---

## 3. The Most Important Metrics — By Service

### EC2 Metrics (Namespace: AWS/EC2)

```
┌─────────────────────────┬──────────────────────────────────────┐
│ Metric                   │ What it tells you                     │
├─────────────────────────┼──────────────────────────────────────┤
│ CPUUtilization           │ % of CPU being used                   │
│                          │ High = app overloaded or runaway proc  │
├─────────────────────────┼──────────────────────────────────────┤
│ NetworkIn / NetworkOut   │ Bytes received/sent by the instance   │
│                          │ Spike = traffic surge or data leak     │
├─────────────────────────┼──────────────────────────────────────┤
│ DiskReadOps / WriteOps   │ Disk read/write operations per second │
│                          │ High = disk-heavy workload             │
├─────────────────────────┼──────────────────────────────────────┤
│ StatusCheckFailed        │ Whether the instance itself is healthy │
│                          │ Value > 0 = something is wrong         │
└─────────────────────────┴──────────────────────────────────────┘

⚠️ IMPORTANT: By default, EC2 does NOT send memory or disk usage
   metrics to CloudWatch. CPU, network, and disk I/O are free.
   For RAM usage, you need the CloudWatch Agent (covered below).
```

### RDS Metrics (Namespace: AWS/RDS)

```
┌─────────────────────────┬──────────────────────────────────────┐
│ Metric                   │ What it tells you                     │
├─────────────────────────┼──────────────────────────────────────┤
│ CPUUtilization           │ Database server CPU usage             │
├─────────────────────────┼──────────────────────────────────────┤
│ DatabaseConnections      │ Number of active connections to DB    │
│                          │ Too high = connection pool exhaustion  │
├─────────────────────────┼──────────────────────────────────────┤
│ FreeStorageSpace         │ Remaining disk space on RDS           │
│                          │ CRITICAL — hitting 0 = DB goes down   │
├─────────────────────────┼──────────────────────────────────────┤
│ ReadLatency/WriteLatency │ Time per DB operation (seconds)       │
│                          │ High = slow queries or overloaded DB   │
├─────────────────────────┼──────────────────────────────────────┤
│ FreeableMemory           │ RAM available on DB instance          │
└─────────────────────────┴──────────────────────────────────────┘
```

### ALB Metrics (Namespace: AWS/ApplicationELB)

```
┌──────────────────────────────┬──────────────────────────────────┐
│ Metric                        │ What it tells you                 │
├──────────────────────────────┼──────────────────────────────────┤
│ RequestCount                  │ Total HTTP requests per period    │
│                               │ Traffic volume — baseline to watch│
├──────────────────────────────┼──────────────────────────────────┤
│ HTTPCode_Target_4XX_Count     │ 4xx errors from your app          │
│                               │ Bad requests, auth failures       │
├──────────────────────────────┼──────────────────────────────────┤
│ HTTPCode_Target_5XX_Count     │ 5xx errors from your app          │
│                               │ App crashes, unhandled exceptions │
├──────────────────────────────┼──────────────────────────────────┤
│ TargetResponseTime            │ How long your app takes to respond│
│                               │ High = slow app or overloaded DB  │
├──────────────────────────────┼──────────────────────────────────┤
│ HealthyHostCount              │ How many instances are healthy    │
│                               │ Drops = instances failing         │
└──────────────────────────────┴──────────────────────────────────┘
```

---

## 4. CloudWatch Alarms — Reacting to Metrics Automatically

**A CloudWatch Alarm watches a single metric and performs an action when that metric crosses a defined threshold.**

```
Alarm States:
  OK       → metric is within normal range
  ALARM    → metric has crossed the threshold
  INSUFFICIENT_DATA → not enough data points yet to evaluate

Alarm Actions (what happens when state changes):
  → Send an SNS notification (email/SMS to your phone)
  → Trigger Auto Scaling action (scale out/in)
  → Stop/Start/Reboot an EC2 instance
  → Invoke a Lambda function
```

### Creating a Critical Alarm — RDS Low Storage

This is one of the most important alarms for any production app:

```
CloudWatch → Alarms → Create alarm

  Select metric:
    RDS → Per-Database Metrics → FreeStorageSpace
    → Select your database instance

  Metric name: FreeStorageSpace
  Statistic: Minimum
  Period: 5 minutes

  Conditions:
    Threshold type: Static
    Whenever FreeStorageSpace is: Lower than
    Threshold value: 2000000000  (2 GB — in bytes)
    → Alert when less than 2 GB free storage remains

  Notification:
    → Create new SNS topic
    → Name: rds-storage-alert
    → Email: your@email.com
    → Create topic (you'll get a confirmation email to verify)

  Alarm name: RDS-Low-Storage-Warning
  → Create alarm
```

### Creating a CPU Alarm for EC2

```
CloudWatch → Alarms → Create alarm

  Select metric:
    EC2 → Per-Instance Metrics → CPUUtilization
    → Select your instance

  Conditions:
    Whenever CPUUtilization is: Greater than
    Threshold: 80
    Period: 5 minutes
    Datapoints to alarm: 3 out of 3
    (CPU must be > 80% for 15 consecutive minutes before alarming
     — avoids false alerts from momentary spikes)

  Alarm name: EC2-High-CPU
```

---

## 5. CloudWatch Dashboards — Your Visual Control Panel

**A Dashboard is a customizable collection of CloudWatch graphs showing multiple metrics at once — your single pane of glass for the health of your application.**

```
Creating a production dashboard for your Node.js app:

CloudWatch → Dashboards → Create dashboard
  Name: NodeJS-App-Dashboard

Add widgets:
  1. Line graph: EC2 CPUUtilization (all instances in ASG)
  2. Number: ALB HealthyHostCount (how many EC2s are alive)
  3. Line graph: ALB TargetResponseTime (app speed over time)
  4. Line graph: ALB HTTPCode_Target_5XX_Count (error rate)
  5. Line graph: RDS DatabaseConnections (DB connection count)
  6. Number: RDS FreeStorageSpace (remaining disk — in GB)
```

```
Your dashboard now shows at a glance:
  ✓ Are my EC2 instances healthy and not overloaded?
  ✓ Is my app responding quickly?
  ✓ Are there errors happening right now?
  ✓ Is my database running out of connections or disk?
```

---

## 6. Standard vs Detailed Monitoring

```
STANDARD MONITORING (default, free):
  Metrics collected every 5 minutes
  → Fine for most workloads and learning

DETAILED MONITORING (optional, small cost):
  Metrics collected every 1 minute
  → Needed when:
      - Auto Scaling needs to react faster
      - Debugging a performance issue in real time
      - SLA requires faster detection of problems

Enable per instance:
  EC2 → your instance → Actions → Monitor and troubleshoot
    → Enable detailed monitoring
```

---

## 7. CloudWatch Agent — Getting Memory and Disk Metrics

As mentioned earlier, **EC2 does NOT send RAM or disk usage metrics by default.** To get these, you install the CloudWatch Agent on your instance:

```bash
# SSH into your EC2 instance
ssh -i my-key.pem ubuntu@<your-instance-ip>

# Install CloudWatch Agent
sudo apt-get install -y amazon-cloudwatch-agent

# Run the configuration wizard
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
# → Follow the prompts, select: memory, disk usage, etc.

# Start the agent
sudo systemctl start amazon-cloudwatch-agent
sudo systemctl enable amazon-cloudwatch-agent
```

```
After installation, new metrics appear in CloudWatch:
  Namespace: CWAgent (your custom agent metrics)
    mem_used_percent    → RAM usage %  ← the one everyone wants
    disk_used_percent   → Disk usage %
    swap_used_percent   → Swap usage %
```

> **Why AWS doesn't include this by default:** The EC2 hypervisor can see CPU, network, and disk I/O from outside the instance — but RAM usage requires software running *inside* the OS. The CloudWatch Agent is that software.

---

## 8. Connecting CloudWatch to Everything You've Built

```
Day 18 (ALB):
  "The ALB runs health checks every 30 seconds"
        ↓
  CloudWatch receives HealthyHostCount metric updates
  You can now graph and alarm on this

Day 19 (Auto Scaling):
  "Scale when CPU > 50%"
        ↓
  Auto Scaling was actually creating a CloudWatch Alarm
  targeting CPUUtilization — that's the mechanism underneath

Day 14 (Backups):
  "Take a backup before risky operations"
        ↓
  FreeStorageSpace alarm from today warns you BEFORE
  disk fills up, giving time to clean up or resize

Day 3 (IAM):
  CloudWatch Agent needs an IAM Role attached to your EC2
  with the CloudWatchAgentServerPolicy — another real-world
  IAM Role use case
```

---

## 9. Interview Questions (Practice Explaining These Out Loud)

**Q1: What is Amazon CloudWatch?**
> AWS's monitoring and observability service that collects metrics, logs, and events from AWS resources, enables visualization via dashboards, and triggers automated actions via alarms when thresholds are crossed.

**Q2: What's the difference between a metric and an alarm in CloudWatch?**
> A metric is a raw time-series data point (e.g., CPU% over time). An alarm watches a metric and changes state (OK/ALARM/INSUFFICIENT_DATA) when it crosses a defined threshold, optionally triggering actions like SNS notifications or Auto Scaling events.

**Q3: Why doesn't EC2 send memory usage to CloudWatch by default?**
> The EC2 hypervisor can observe CPU, network, and disk I/O from outside the VM — but RAM usage requires software running inside the OS. The CloudWatch Agent must be installed inside the instance to collect and send memory metrics.

**Q4: What are the three alarm states in CloudWatch?**
> OK (metric within normal range), ALARM (threshold crossed), and INSUFFICIENT_DATA (not enough data points to evaluate yet).

**Q5: How does CloudWatch relate to Auto Scaling?**
> Auto Scaling policies use CloudWatch alarms as their trigger mechanism — when you set "scale out when CPU > 50%," Auto Scaling creates a CloudWatch alarm that watches CPUUtilization and fires a scale-out action when the threshold is breached.

**Q6: What is a CloudWatch Dashboard?**
> A customizable collection of graphs and metrics displayed together — your visual overview of application health, showing multiple services and metrics on one screen simultaneously.

---

## 10. Hands-On Assignment for Today

1. Go to **CloudWatch → Metrics → EC2** — find your instance's `CPUUtilization` and view it as a graph. Change the time range to "1h" to see recent activity.

2. Go to **CloudWatch → Metrics → RDS** — view `DatabaseConnections` and `FreeStorageSpace` for your RDS instance.

3. **Create an RDS Low Storage alarm** (Section 4) — set threshold at 2 GB, email yourself.

4. **Create an EC2 High CPU alarm** — 80% CPU for 3 consecutive 5-minute periods.

5. **Create a Dashboard** named `MyApp-Dashboard` with at least 4 widgets covering EC2 CPU, ALB response time, ALB 5xx errors, and RDS connections.

6. **Install the CloudWatch Agent** on your EC2 instance (Section 7) — confirm `mem_used_percent` appears in CloudWatch → Metrics → CWAgent.

7. Write a short note covering:
   - Why you'd alarm on RDS FreeStorageSpace specifically
   - What the 3-out-of-3 datapoints setting on the CPU alarm prevents
   - Which metric you'd check first if users reported your app was slow

---

## 11. Day 20 Goal — Can You Teach This?

By the end of today, you should be able to explain, without notes:

> *"CloudWatch collects metrics from all my AWS services automatically — CPU from EC2, connections from RDS, request counts and error rates from ALB. I set up alarms that trigger notifications or Auto Scaling actions when metrics cross thresholds, and dashboards that give me a visual overview of my entire app's health at a glance. The CloudWatch Agent adds memory and disk metrics that EC2 can't send by default. CloudWatch is the monitoring engine that makes Auto Scaling work — scaling policies are really just CloudWatch alarms under the hood."*

---

**Next up — Day 21: CloudWatch Logs**
