# Day 22: Lambda Introduction — Serverless Computing
### *(Running Code Without Managing a Single Server — The Biggest Shift in Cloud Development)*

> **Roadmap reference:** Week 4, Day 22 — "Lambda Introduction"

---

## Why This Matters

Every service you've built so far runs on EC2 — a server that exists 24/7, whether it's handling requests or sitting idle at 3 AM. **Lambda is a fundamentally different model:** your code runs only when triggered, for exactly as long as it takes to complete, and you're billed for only those milliseconds.

This isn't just a cost optimization — it's a completely different way of architecting backends. Understanding Lambda deeply separates junior developers who "know AWS" from engineers who can design cloud-native systems.

---

## 1. What is AWS Lambda?

**AWS Lambda is a serverless compute service — you upload your code (a function), define what triggers it, and AWS handles everything else: provisioning servers, scaling, patching, availability, and billing.**

```
TRADITIONAL SERVER (EC2):
  Server exists → Server waits → Request arrives → Code runs → Response sent
  │               │                                                        │
  └── You pay ────┘────────────── paying the whole time ──────────────────┘

LAMBDA (Serverless):
  Request arrives → Lambda wakes up → Code runs → Response sent → Lambda sleeps
                    │                              │
                    └── You pay ONLY for this ─────┘
                        (measured in milliseconds)
```

> **Analogy:** EC2 is like hiring a full-time employee who sits at their desk all day whether or not there's work to do. Lambda is like hiring a contractor who appears instantly when needed, does the job, and leaves — you only pay for the hours actually worked.

---

## 2. Serverless vs Server-Based — The Real Differences

```
┌─────────────────────┬──────────────────┬──────────────────────┐
│ Aspect               │ EC2 (Server)      │ Lambda (Serverless)   │
├─────────────────────┼──────────────────┼──────────────────────┤
│ Server management    │ You manage        │ AWS manages           │
│ OS patching          │ Your job          │ AWS's job             │
│ Scaling              │ ASG (minutes)     │ Instant, automatic    │
│ Availability         │ You configure     │ Built-in, 99.99%      │
│ Billing unit         │ Per hour          │ Per 100ms + requests  │
│ Max runtime          │ Unlimited         │ 15 minutes            │
│ Always running cost  │ Yes               │ No — zero at zero use │
│ Cold starts          │ No                │ Yes (first invocation)│
│ State/disk           │ Persistent        │ Ephemeral (temp only) │
└─────────────────────┴──────────────────┴──────────────────────┘
```

---

## 3. The Lambda Execution Model

```
TRIGGER (what wakes Lambda up)
        │
        ▼
LAMBDA FUNCTION (your code runs here)
        │
        ├── Reads input from the event object
        ├── Does work (calls DB, S3, external API, etc.)
        └── Returns a response (or void for async triggers)
        │
        ▼
RESULT (response to caller, or side effect)
```

### The Handler Function

Every Lambda function has a **handler** — the entry point AWS calls when the function is triggered:

```javascript
// Node.js Lambda handler structure
exports.handler = async (event, context) => {

  // event   → the input data from whatever triggered this function
  //           (HTTP request body, S3 event, SQS message, etc.)

  // context → metadata about the invocation itself
  //           (function name, remaining time, request ID)

  // Your logic goes here
  const name = event.name || 'World';

  return {
    statusCode: 200,
    body: JSON.stringify({
      message: `Hello, ${name}!`,
      timestamp: new Date().toISOString()
    })
  };
};
```

---

## 4. What Can Trigger a Lambda Function?

This is one of Lambda's most powerful aspects — it integrates with almost every AWS service as a trigger:

```
┌─────────────────────────────────────────────────────────────┐
│                    LAMBDA TRIGGERS                            │
├─────────────────┬───────────────────────────────────────────┤
│ API Gateway      │ HTTP request hits your API endpoint        │
│                  │ → Most common: REST API backend             │
├─────────────────┼───────────────────────────────────────────┤
│ S3               │ File uploaded/deleted in a bucket          │
│                  │ → Resize image, process CSV, scan file     │
├─────────────────┼───────────────────────────────────────────┤
│ DynamoDB Streams │ Record inserted/updated/deleted            │
│                  │ → React to database changes in real time   │
├─────────────────┼───────────────────────────────────────────┤
│ SQS              │ Message arrives in a queue                 │
│                  │ → Process background jobs reliably         │
├─────────────────┼───────────────────────────────────────────┤
│ EventBridge      │ Scheduled cron expression                  │
│ (CloudWatch)     │ → Run reports, cleanup jobs, send emails   │
├─────────────────┼───────────────────────────────────────────┤
│ SNS              │ Notification published to a topic          │
│                  │ → Fan-out event processing                 │
├─────────────────┼───────────────────────────────────────────┤
│ Cognito          │ User sign-up/sign-in event                 │
│                  │ → Custom auth validation logic             │
└─────────────────┴───────────────────────────────────────────┘
```

---

## 5. Lambda Pricing — Why It's Often Dramatically Cheaper

```
Lambda pricing has two components:

1. REQUESTS:
   First 1 million requests per month → FREE (always, forever)
   After that → $0.20 per million requests

2. DURATION (compute time):
   First 400,000 GB-seconds per month → FREE
   After that → $0.0000166667 per GB-second

Real example:
  A function using 128MB RAM runs for 100ms per invocation
  128MB = 0.125 GB
  0.125 GB × 0.1 seconds = 0.0125 GB-seconds per invocation

  At 1 million invocations/month:
  1,000,000 × 0.0125 = 12,500 GB-seconds
  12,500 × $0.0000166667 = $0.21/month

  Compare to: EC2 t3.micro running 24/7 = ~$8–10/month
```

```
When Lambda is MUCH cheaper than EC2:
  → Irregular, spiky, or infrequent workloads
  → Background jobs triggered by events
  → APIs with highly variable traffic

When EC2 might be cheaper than Lambda:
  → Constant, high-volume traffic (Lambda running 24/7 adds up)
  → Long-running processes (Lambda has 15-minute max)
  → Workloads requiring persistent in-memory state
```

---

## 6. Cold Starts — Lambda's Main Drawback

**A cold start happens when Lambda needs to initialize a new execution environment for your function — because no warm instance is currently available.**

```
WARM invocation (fast):
  Request arrives → existing Lambda container ready → code runs
  Latency: 1–50ms typically

COLD START (slow):
  Request arrives → Lambda needs to spin up a new container →
  download your code → initialize Node.js runtime →
  run your handler
  Latency: 200ms–2000ms for Node.js (worse for Java)

When cold starts happen:
  → First ever invocation of a function
  → After a period of inactivity (no traffic)
  → When Lambda scales out rapidly to handle many concurrent requests
```

### Mittigating Cold Starts

```
Strategy 1: Keep functions WARM (Provisioned Concurrency)
  → AWS keeps a pool of pre-initialized containers running
  → Eliminates cold starts entirely
  → Costs extra (you pay for the idle warm capacity)
  → Worth it for user-facing APIs where latency matters

Strategy 2: Reduce package size
  → Smaller deployment package = faster cold start
  → Only include dependencies you actually use
  → Avoid large libraries when smaller alternatives exist

Strategy 3: Choose runtime wisely
  → Node.js and Python have much faster cold starts than Java/JVM
  → For latency-sensitive functions, avoid Java

Strategy 4: Scheduled warm-up pings
  → A cron EventBridge rule pings your function every 5 minutes
  → Keeps it warm cheaply without Provisioned Concurrency
  → Hacky but effective for low-traffic functions
```

---

## 7. Lambda Limits — Know These Before Designing

```
┌─────────────────────────────┬──────────────────────────────┐
│ Limit                        │ Value                         │
├─────────────────────────────┼──────────────────────────────┤
│ Max execution timeout        │ 15 minutes                    │
│ Max memory                   │ 10,240 MB (10 GB)             │
│ Max deployment package size  │ 50 MB (zip) / 250 MB unpacked │
│ Max /tmp storage             │ 10 GB (ephemeral)             │
│ Max concurrent executions    │ 1,000 per region (default)    │
│ Max response payload (sync)  │ 6 MB                          │
└─────────────────────────────┴──────────────────────────────┘
```

```
What these limits rule out:
  ✗ Long-running video encoding (> 15 min)
  ✗ Large file downloads/uploads through Lambda directly
    (use S3 pre-signed URLs instead — Day 9)
  ✗ Apps that need persistent in-memory caches
    (use ElastiCache instead)
  ✗ Heavy ML model inference (may exceed memory/time limits)
```

---

## 8. Lambda Execution Role — IAM for Functions

Every Lambda function needs an **Execution Role** — an IAM Role that defines what AWS resources the function is allowed to access.

```
Lambda function needs to:
  - Read a file from S3
  - Write a record to DynamoDB
  - Send a message via SNS

Without execution role: "Access Denied" on every AWS SDK call
With execution role:    Correct permissions granted securely,
                        no hardcoded credentials needed

This is the IAM Role pattern from Day 3, now applied to Lambda —
the function itself assumes the role at runtime, AWS provides
temporary credentials automatically.
```

```
Creating an Execution Role:
  IAM → Roles → Create role
    Trusted entity: AWS service → Lambda
    Attach policies:
      AWSLambdaBasicExecutionRole  ← always needed (writes logs to CloudWatch)
      AmazonS3ReadOnlyAccess        ← if your function reads from S3
      AmazonDynamoDBFullAccess      ← if your function uses DynamoDB
    Name: my-lambda-execution-role
```

---

## 9. When to Use Lambda vs EC2 — Decision Framework

```
USE LAMBDA when:
  ✓ Task is triggered by an event (upload, request, message)
  ✓ Task is short (< 15 minutes per invocation)
  ✓ Traffic is unpredictable or spiky
  ✓ You want zero operational overhead
  ✓ Cost at low/zero traffic should be near zero

USE EC2 when:
  ✓ Long-running processes (websocket server, data pipelines)
  ✓ Constant, predictable high traffic (Lambda always running)
  ✓ App needs persistent state or large in-memory cache
  ✓ You need full OS access or custom system configuration

Real examples:
  Lambda: Image resize on S3 upload ← perfect fit
  Lambda: Nightly report generation ← perfect fit
  Lambda: Webhook handler            ← great fit
  Lambda: Main REST API              ← depends on traffic & latency needs
  EC2:    WebSocket chat server       ← persistent connections, use EC2
  EC2:    ML inference server         ← long warm-up, high memory, use EC2
```

---

## 10. Common Lambda Architecture Patterns

### Pattern 1: S3 Event Processing

```
User uploads image to S3
        ↓
S3 triggers Lambda automatically
        ↓
Lambda resizes image, saves thumbnail back to S3
        ↓
Lambda updates record in DynamoDB/RDS with thumbnail URL
```

### Pattern 2: Scheduled Job (Cron)

```
EventBridge rule: "Every day at 3 AM"
        ↓
Lambda function runs
        ↓
Queries RDS for inactive users
        ↓
Sends notification emails via SES/SNS
        ↓
Function exits, Lambda sleeps until tomorrow 3 AM
```

### Pattern 3: API Backend (with API Gateway — Day 24)

```
HTTP request: POST /api/users
        ↓
API Gateway receives it
        ↓
Invokes Lambda function with request data in `event`
        ↓
Lambda queries RDS, returns response object
        ↓
API Gateway sends HTTP response to client
```

---

## 11. Interview Questions (Practice Explaining These Out Loud)

**Q1: What is AWS Lambda, and how is it different from EC2?**
> Lambda is a serverless compute service where you upload functions that run only when triggered — with no server to provision or manage. Unlike EC2, which runs 24/7 and you pay per hour, Lambda runs only when invoked and you pay per millisecond of execution — scaling instantly and automatically.

**Q2: What is a cold start in Lambda, and how do you mitigate it?**
> A cold start is the latency when Lambda needs to initialize a new execution environment — downloading code, starting the runtime, and initializing the handler. It adds 200ms–2s of latency. Mitigation: Provisioned Concurrency for user-facing APIs, keeping functions small, using Node.js/Python over Java, or scheduled warm-up pings.

**Q3: What triggers a Lambda function?**
> Many AWS services — API Gateway (HTTP requests), S3 (file events), SQS (queue messages), DynamoDB Streams (record changes), EventBridge (scheduled cron jobs), SNS (notifications), Cognito (auth events), and more.

**Q4: What is the maximum execution time for a Lambda function?**
> 15 minutes. Functions requiring longer execution must be redesigned — split into smaller functions, use Step Functions for orchestration, or switch to EC2/ECS for truly long-running processes.

**Q5: Why doesn't Lambda need hardcoded AWS credentials?**
> Lambda functions have an Execution Role — an IAM Role that Lambda assumes at runtime. AWS automatically provides temporary credentials through that role, so the function can call other AWS services without any credentials in the code.

**Q6: When would you NOT use Lambda?**
> For long-running processes (> 15 min), persistent WebSocket connections, applications needing large in-memory state, heavy ML inference, or consistently high-traffic APIs where Lambda's constant invocations would cost more than a dedicated EC2 instance.

---

## 12. Hands-On Assignment for Today

1. Go to **AWS Console → Lambda → Create function**:
   - Author from scratch
   - Name: `hello-world-function`
   - Runtime: Node.js 20.x
   - Create a new execution role with basic Lambda permissions

2. Replace the default code with this:
   ```javascript
   exports.handler = async (event) => {
     console.log('Event received:', JSON.stringify(event, null, 2));
     return {
       statusCode: 200,
       body: JSON.stringify({
         message: `Hello, ${event.name || 'World'}!`,
         timestamp: new Date().toISOString()
       })
     };
   };
   ```

3. Click **Deploy** → then **Test** with this event:
   ```json
   { "name": "Ashish" }
   ```

4. Check the **Execution result** — confirm response and logs.

5. Go to **CloudWatch → Log Groups → /aws/lambda/hello-world-function** — find your function's log stream and read the output.

6. Go to **Configuration → General configuration** — note the Memory (128MB) and Timeout (3s defaults) — think about what you'd change for a real-world function.

7. Write a short note covering:
   - The billing difference between EC2 and Lambda in your own words
   - One real use case from your own projects where Lambda would be perfect
   - Why cold starts matter for a user-facing API but not for a nightly batch job

---

## 13. Day 22 Goal — Can You Teach This?

By the end of today, you should be able to explain, without notes:

> *"Lambda is serverless compute — I upload a function, define a trigger, and AWS runs it on demand with no servers to manage, scaling instantly to zero or thousands of concurrent executions. I pay only for the milliseconds my code actually runs, making it extremely cost-effective for event-driven or irregular workloads. Cold starts are the main tradeoff — the first invocation has extra latency while Lambda initializes. Lambda is perfect for event-driven tasks like S3 processing or scheduled jobs; EC2 is better for persistent, long-running services."*

---

**Next up — Day 23: Create Lambda Function (hands-on deeper dive)**
