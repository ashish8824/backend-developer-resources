# Day 21: CloudWatch Logs
### *(Reading Your Application's Story — Finding Problems Before Users Do)*

> **Roadmap reference:** Week 3, Day 21 — "CloudWatch Logs"

---

## Why This Matters

Yesterday's metrics tell you **what** is happening — CPU is high, error rate spiked, response time increased. But metrics don't tell you **why**. For that, you need logs.

```
Metrics say: "5xx errors jumped from 0 to 47 at 3:15 PM"
Logs say:    "TypeError: Cannot read property 'userId' of undefined
              at /home/ubuntu/app/routes/users.js:42:18"
```

**Logs are the story behind the numbers.** They're what you actually read when debugging a production issue. CloudWatch Logs is where all those stories live in AWS.

---

## 1. What is CloudWatch Logs?

**CloudWatch Logs is AWS's centralized log storage and analysis service — it collects, stores, searches, and alerts on log data from your applications and AWS services.**

```
Where logs come from:

  Your Express App  ──────────────────────► CloudWatch Logs
  EC2 System Logs   ──────────────────────►
  Lambda Functions  ──────────────────────► (automatically)
  RDS Logs          ──────────────────────►
  ALB Access Logs   ──────────────────────►
  CloudTrail Logs   ──────────────────────►
```

### Core Concepts

```
LOG GROUP
  → A container for logs from one source
  → e.g., "/aws/ec2/my-nodejs-app"
       or  "/aws/lambda/my-function"
  → You define the name, typically matching the service/app

LOG STREAM
  → A sequence of log events from ONE specific source instance
  → e.g., one log stream per EC2 instance, per Lambda invocation
  → Lives inside a Log Group

LOG EVENT
  → A single line/entry in a log stream
  → Has a timestamp and a message
  → e.g., "2026-06-22T10:00:00Z  GET /users 200 45ms"
```

```
LOG GROUP: /my-nodejs-app
    │
    ├── LOG STREAM: i-0abc1234 (EC2 instance A)
    │     ├── LOG EVENT: 10:00 GET /health 200 2ms
    │     ├── LOG EVENT: 10:00 GET /users  200 45ms
    │     └── LOG EVENT: 10:01 POST /upload 500 ERROR
    │
    └── LOG STREAM: i-0def5678 (EC2 instance B)
          ├── LOG EVENT: 10:00 GET /health 200 3ms
          └── LOG EVENT: 10:00 GET /users  200 41ms
```

---

## 2. Sending Your Express App Logs to CloudWatch

By default, your Node.js app's `console.log()` output goes nowhere useful in production — it's lost when the SSH session ends or the instance restarts. CloudWatch Logs fixes that.

### Step 1: IAM Role for CloudWatch Logs

Your EC2 instance needs permission to write logs to CloudWatch:

```
AWS Console → IAM → Roles → Create role
  Trusted entity: AWS service → EC2
  Attach policy: CloudWatchAgentServerPolicy
  Name: ec2-cloudwatch-role

EC2 → your instance → Actions → Security
  → Modify IAM role → attach: ec2-cloudwatch-role
```

### Step 2: Install and Configure CloudWatch Agent

```bash
# SSH into your EC2 instance
sudo apt-get install -y amazon-cloudwatch-agent

# Create the agent config file
sudo nano /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

Paste this configuration:

```json
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/home/ubuntu/.pm2/logs/my-api-out.log",
            "log_group_name": "/my-nodejs-app/out",
            "log_stream_name": "{instance_id}",
            "timezone": "UTC"
          },
          {
            "file_path": "/home/ubuntu/.pm2/logs/my-api-error.log",
            "log_group_name": "/my-nodejs-app/errors",
            "log_stream_name": "{instance_id}",
            "timezone": "UTC"
          }
        ]
      }
    }
  },
  "metrics": {
    "metrics_collected": {
      "mem": { "measurement": ["mem_used_percent"] },
      "disk": { "measurement": ["disk_used_percent"],
                "resources": ["/"] }
    }
  }
}
```

```bash
# Start the CloudWatch Agent with your config
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -s \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json

# Verify it's running
sudo systemctl status amazon-cloudwatch-agent
```

```
What this config does:
  - Watches PM2's stdout log file → sends to /my-nodejs-app/out
  - Watches PM2's stderr/error log file → sends to /my-nodejs-app/errors
  - Each EC2 instance gets its own Log Stream (by instance ID)
  - Also collects memory + disk metrics (from Day 20)
```

---

## 3. Structured Logging — Making Logs Actually Searchable

Plain `console.log("user created")` is hard to search and filter at scale. **Structured logging** (JSON format) makes every log entry machine-readable and queryable.

```javascript
// BAD — hard to parse and search
console.log("User 42 uploaded file image.jpg to S3 bucket in 234ms");

// GOOD — structured JSON, every field searchable
console.log(JSON.stringify({
  timestamp:  new Date().toISOString(),
  level:      "INFO",
  action:     "FILE_UPLOAD",
  userId:     42,
  filename:   "image.jpg",
  s3Key:      "uploads/1719500000000-image.jpg",
  durationMs: 234,
  requestId:  req.headers['x-request-id']
}));
```

```
Now in CloudWatch Logs Insights you can query:
  fields @timestamp, userId, filename, durationMs
  | filter action = "FILE_UPLOAD"
  | filter durationMs > 500
  | sort durationMs desc
  | limit 20

→ "Show me the 20 slowest file uploads, most recent first"
  This is impossible to answer efficiently with plain text logs.
```

### Winston — A Proper Logger for Node.js

```bash
npm install winston winston-cloudwatch
```

```javascript
// logger.js
const winston = require('winston');

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()         // outputs structured JSON
  ),
  transports: [
    new winston.transports.Console(),    // still logs to terminal
    new winston.transports.File({        // also writes to file
      filename: 'logs/error.log',        // (which CloudWatch Agent picks up)
      level: 'error'
    }),
    new winston.transports.File({
      filename: 'logs/combined.log'
    })
  ]
});

module.exports = logger;
```

```javascript
// In your routes:
const logger = require('./logger');

app.post('/users', async (req, res) => {
  const start = Date.now();
  try {
    const result = await pool.query(
      'INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *',
      [req.body.name, req.body.email]
    );
    logger.info({
      action: 'CREATE_USER',
      userId: result.rows[0].id,
      durationMs: Date.now() - start
    });
    res.status(201).json(result.rows[0]);
  } catch (err) {
    logger.error({
      action: 'CREATE_USER_FAILED',
      error: err.message,
      body: req.body,
      durationMs: Date.now() - start
    });
    res.status(500).json({ error: err.message });
  }
});
```

---

## 4. CloudWatch Logs Insights — Querying Your Logs

**Logs Insights is a query engine for CloudWatch Logs — like SQL for your log data.**

```
CloudWatch → Logs Insights
  → Select log group: /my-nodejs-app/errors
  → Time range: Last 1 hour
```

### Useful Queries for a Node.js Backend

```sql
-- Find all errors in the last hour
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 50

-- Count errors by type
fields @timestamp, action, error
| filter level = "error"
| stats count(*) as errorCount by action
| sort errorCount desc

-- Find slow API responses (over 500ms)
fields @timestamp, action, durationMs, userId
| filter durationMs > 500
| sort durationMs desc
| limit 20

-- Track a specific user's activity
fields @timestamp, action, userId
| filter userId = 42
| sort @timestamp desc

-- Error rate over time (per 5 minutes)
fields @timestamp, level
| filter level = "error"
| stats count(*) as errors by bin(5m)
| sort @timestamp asc
```

---

## 5. Metric Filters — Turning Log Data Into Metrics

**A Metric Filter watches incoming log data, looks for specific patterns, and converts matching log events into a CloudWatch metric.**

```
Use case: Count 5xx errors from your Express app's logs

  Log event arrives: {"level":"error","action":"ROUTE_ERROR","status":500}
          ↓
  Metric Filter: "if log contains level=error AND status=500, increment counter"
          ↓
  CloudWatch Metric: "AppErrors" ticks up by 1
          ↓
  CloudWatch Alarm: "If AppErrors > 10 in 5 minutes → send alert"
```

```
Creating a Metric Filter:
  CloudWatch → Log groups → /my-nodejs-app/errors
    → Metric filters → Create metric filter

    Filter pattern: { $.level = "error" }
    Filter name: ErrorCount
    Metric namespace: MyNodeApp
    Metric name: ErrorCount
    Metric value: 1

  → Create filter
```

> This bridges the gap between logs (raw text) and metrics (numbers you can alarm on). Once an error log becomes a metric, you can set an alarm: "if more than 10 errors in 5 minutes, send me an SMS." This is the professional way to monitor application health.

---

## 6. Log Retention — Don't Pay to Store Logs Forever

By default, CloudWatch Logs **keeps logs forever** — and you pay for storage. For most applications, old logs lose value quickly:

```
Set retention on your Log Groups:

CloudWatch → Log Groups → /my-nodejs-app/out
  → Actions → Edit retention setting
  → Retention period: 30 days (or 7, 14, 90 — your choice)

Recommended retention periods:
  Application debug logs: 7–14 days
  Application error logs:  30–90 days
  Security/audit logs:     1–7 years (compliance requirement)
  RDS/database logs:       30 days
```

> This is the same "don't let AWS bill you for forgotten resources" discipline from Day 2/3, applied to log storage.

---

## 7. AWS Services That Send Logs Automatically

These don't need CloudWatch Agent configuration — they send logs natively:

```
SERVICE             LOG GROUP                   WHAT'S LOGGED
──────────          ──────────────────          ──────────────
Lambda              /aws/lambda/<fn-name>       Function output + errors
RDS (enabled)       /aws/rds/instance/...       Slow queries, errors
ALB (S3-based)      your-s3-bucket/alb-logs/    Every HTTP request details
CloudTrail          /aws/cloudtrail/...          Every API call in your account
VPC Flow Logs       /aws/vpc/flowlogs/...        Network traffic records
```

### Enable RDS Logs

```
RDS → your instance → Modify
  → Log exports:
    ✓ Postgresql log (error log)
    ✓ Upgrade log
  → Apply immediately
```

---

## 8. The Debugging Workflow — Metrics to Logs to Root Cause

This is the exact workflow you'd use in a real production incident:

```
Step 1: CloudWatch Dashboard shows spike in 5xx errors
  (Day 20 — ALB HTTPCode_Target_5XX_Count metric spiked)
          ↓
Step 2: Check when it started
  (Timeline: errors began at 3:14 PM)
          ↓
Step 3: Open Logs Insights on /my-nodejs-app/errors
  Query: filter @timestamp > "2026-06-22T15:14:00"
         | filter level = "error"
         | sort @timestamp asc
  → You see: "Cannot connect to database: too many connections"
          ↓
Step 4: Switch to metrics — RDS DatabaseConnections
  (Day 20) — confirm connections maxed out at 3:12 PM
          ↓
Step 5: Root cause identified: a code deployment at 3:10 PM
  introduced a connection pool misconfiguration — connections
  weren't being released back to the pool after use
          ↓
Step 6: Fix deployed, connections normalized, errors stop
```

> This workflow — Metrics → Logs → Root Cause — is called **Observability**, and being able to describe it fluently is one of the highest-signal interview answers a backend developer can give.

---

## 9. Interview Questions (Practice Explaining These Out Loud)

**Q1: What is CloudWatch Logs, and how does it differ from CloudWatch Metrics?**
> CloudWatch Metrics tracks numerical measurements over time (CPU%, error count). CloudWatch Logs stores the actual text output from applications and services — the detailed "what happened" behind the numbers. Together, metrics tell you there's a problem, logs tell you why.

**Q2: What is a Log Group vs a Log Stream?**
> A Log Group is a container for logs from one application or source (e.g., `/my-nodejs-app/errors`). A Log Stream is a sequence of log events from one specific instance of that source (e.g., one EC2 instance's logs within that group).

**Q3: What is a Metric Filter in CloudWatch?**
> A rule that watches incoming log events for a specific pattern and converts matching events into a CloudWatch metric — bridging the gap between raw logs and the numbers you can alarm on.

**Q4: Why use structured (JSON) logging instead of plain text logs?**
> Structured logs make every field individually queryable — you can filter by user ID, action type, duration, error message, etc. using Logs Insights. Plain text logs require regex pattern matching, which is brittle, slow, and hard to maintain at scale.

**Q5: Why should you set a retention period on Log Groups?**
> CloudWatch Logs charges for storage. Old logs quickly lose debugging value but continue accumulating cost. Setting retention periods (7–90 days for most app logs) automatically deletes old data, keeping storage costs controlled.

**Q6: Walk me through how you'd debug a production error spike using CloudWatch.**
> Start with Metrics to identify when errors spiked and which metric (5xx count, latency). Switch to Logs Insights to query log events from that time window for error-level entries. Read the stack traces to find the root cause, cross-reference with other metrics (DB connections, CPU) to confirm the hypothesis.

---

## 10. Hands-On Assignment for Today

1. Install or confirm the **CloudWatch Agent** is running on your EC2 with the config from Section 2 — verify log groups appear in CloudWatch → Log Groups.

2. Add **structured JSON logging** to your Express app using the pattern in Section 3 — restart via PM2.

3. Make a few API calls (`GET /users`, `POST /users`, a deliberate bad request) — then go to **CloudWatch → Logs → Log Groups → /my-nodejs-app/out** and confirm your logs appear.

4. Open **Logs Insights** — run the "find all errors" query and the "slow responses > 500ms" query from Section 4.

5. Create a **Metric Filter** on `/my-nodejs-app/errors` that counts error-level log events → creates a metric → create an alarm if ErrorCount > 5 in 5 minutes.

6. **Set retention** on all your log groups — 14 days for output logs, 30 days for error logs.

7. Write a short note covering:
   - The difference between a Log Group and a Log Stream
   - Why structured JSON logging beats plain text for debugging
   - The Metrics → Logs → Root Cause debugging workflow in your own words

---

## 11. Day 21 Goal — Can You Teach This?

By the end of today, you should be able to explain, without notes:

> *"CloudWatch Logs is centralized log storage for my AWS resources and application. My Express app writes structured JSON logs, the CloudWatch Agent ships them to a Log Group, and I use Logs Insights to query across all instances at once. Metric Filters convert log patterns into CloudWatch metrics I can alarm on. When something goes wrong, I start with metrics to identify the time and type of problem, then use Logs Insights to find the specific error causing it — that combination is what makes production debugging manageable instead of chaotic."*

---

## Week 3 Complete! 🎉

You've covered Networking (Days 15–17), Load Balancing (Day 18), Auto Scaling (Day 19), and now full Monitoring with Metrics + Logs (Days 20–21). The Week 3 Mini Project — a highly available Node.js application — is fully complete.

---

**Next up — Week 4 begins: Day 22: Lambda Introduction (Serverless Computing)**
