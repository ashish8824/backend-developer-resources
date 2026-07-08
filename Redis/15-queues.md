# Module 15: Queues

**Level:** Advanced

Queues let you do work **asynchronously in the background**, instead of forcing a user to wait for a slow task to finish. This module covers building a simple queue by hand, then the production-grade way using BullMQ.

---

## 1. Why Do We Need Queues?

Imagine a "Sign Up" endpoint that needs to:

1. Save the user to the database.
2. Send a welcome email.
3. Generate a PDF report.
4. Notify a Slack channel.

If you do all 4 steps synchronously, inside the request handler, the user has to **wait** for all of them to finish before they see a response — even though steps 2-4 don't need to block the response at all.

```
❌ Without a queue:
User signs up ──▶ Save to DB ──▶ Send email ──▶ Generate PDF ──▶ Notify Slack ──▶ Response
                                     (user is waiting through ALL of this)

✅ With a queue:
User signs up ──▶ Save to DB ──▶ Push job to queue ──▶ Response (instant!)
                                        │
                                        ▼ (happens in the background, separately)
                              Send email, generate PDF, notify Slack
```

This dramatically improves perceived performance, and if step 3 (PDF generation) temporarily fails, it doesn't affect the user's signup at all — it can simply be retried in the background.

---

## 2. Building a Simple Queue by Hand (Using Lists)

Recall from Module 4: Lists support `LPUSH`/`RPUSH` (add) and `LPOP`/`RPOP` (remove). This is exactly the primitive needed for a basic **FIFO (First In, First Out)** queue.

```js
// producer.js — adds jobs to the queue
const redis = require("./config/redisClient");

async function enqueueJob(jobData) {
  // Push the job (as a JSON string) onto the RIGHT end of the list
  await redis.rpush("email-queue", JSON.stringify(jobData));
  console.log("Job added to queue:", jobData);
}

enqueueJob({ to: "ashish@example.com", subject: "Welcome!" });
```

```js
// worker.js — continuously processes jobs from the queue
const redis = require("./config/redisClient");

async function processJob(job) {
  console.log(`Sending email to ${job.to} with subject "${job.subject}"`);
  // ... actual email-sending logic here ...
}

async function startWorker() {
  console.log("Worker started, waiting for jobs...");

  while (true) {
    // BLPOP = "Blocking Left Pop" — waits (blocks) until an item is available,
    // instead of constantly polling in a tight loop wasting CPU.
    // The "0" means "wait forever" (no timeout).
    const result = await redis.blpop("email-queue", 0);

    // result is an array: [listName, poppedValue]
    const job = JSON.parse(result[1]);

    try {
      await processJob(job);
    } catch (err) {
      console.error("Job failed:", err.message);
      // In a real system, you'd push this back to a "retry" or "failed" queue here
    }
  }
}

startWorker();
```

Run `worker.js` in one terminal, then run `producer.js` (or a script that calls `enqueueJob`) in another — the worker picks up and processes jobs as they arrive.

**Why `BLPOP` instead of a `setInterval` polling loop?**

```js
// ❌ Wasteful: constantly asking "is there a job yet? is there a job yet?" even when idle
setInterval(async () => {
  const job = await redis.lpop("email-queue");
  if (job) processJob(JSON.parse(job));
}, 1000);

// ✅ Efficient: the worker sleeps until Redis actually has something for it — zero wasted checks
const result = await redis.blpop("email-queue", 0);
```

---

## 3. Limitations of the Hand-Built Queue

This simple List-based queue works, but is missing features you'll want in real production systems:

- **No automatic retries** if a job fails.
- **No delayed jobs** ("run this in 10 minutes").
- **No job priorities.**
- **No visibility/dashboard** into pending, failed, or completed jobs.
- **If a worker crashes mid-job**, that job is simply lost (it was already popped off the list).

This is exactly why production Node.js apps use a dedicated library: **BullMQ**.

---

## 4. BullMQ — The Production-Grade Queue Library

BullMQ is a popular Node.js queue library built **on top of Redis**, adding all the missing production features.

```bash
npm install bullmq
```

**Adding jobs to a queue:**

```js
// producer.js
const { Queue } = require("bullmq");

// Connect the queue to Redis — behind the scenes, BullMQ uses Redis data structures
// like Sorted Sets and Lists, similar to what we built by hand, but far more robust.
const emailQueue = new Queue("email-queue", {
  connection: { host: "127.0.0.1", port: 6379 },
});

async function addWelcomeEmailJob(userEmail) {
  await emailQueue.add(
    "welcome-email",                 // job name (useful for filtering/logging)
    { to: userEmail, subject: "Welcome!" }, // job payload (data your worker will use)
    {
      attempts: 3,                    // retry up to 3 times if the job throws an error
      backoff: { type: "exponential", delay: 1000 }, // wait longer between each retry
      removeOnComplete: true,         // clean up successful jobs automatically
    }
  );
}

addWelcomeEmailJob("ashish@example.com");
```

**Processing jobs with a Worker:**

```js
// worker.js
const { Worker } = require("bullmq");

const worker = new Worker(
  "email-queue", // must match the queue name used by the producer
  async (job) => {
    // job.name = "welcome-email", job.data = { to, subject }
    console.log(`Processing job ${job.id}: sending email to ${job.data.to}`);

    await sendEmail(job.data.to, job.data.subject); // your actual email logic

    // If this throws an error, BullMQ automatically retries based on the "attempts" setting
  },
  { connection: { host: "127.0.0.1", port: 6379 }, concurrency: 5 } // process up to 5 jobs at once
);

worker.on("completed", (job) => {
  console.log(`✅ Job ${job.id} completed successfully`);
});

worker.on("failed", (job, err) => {
  console.error(`❌ Job ${job.id} failed after all retries:`, err.message);
});
```

**Delayed jobs (huge advantage over the hand-built version):**

```js
// Send a "did you forget something?" email 24 hours after signup
await emailQueue.add(
  "reminder-email",
  { to: userEmail },
  { delay: 1000 * 60 * 60 * 24 } // delay in milliseconds — 24 hours
);
```

**Job priorities:**

```js
// Lower number = higher priority. This job jumps ahead of normal jobs.
await emailQueue.add("urgent-email", data, { priority: 1 });
```

---

## 5. Where Does BullMQ Actually Store Data in Redis?

Under the hood, BullMQ uses a combination of Redis Lists (for waiting jobs), Sorted Sets (for delayed jobs, sorted by "when to run"), and Hashes (for job data/status) — essentially a much more sophisticated version of exactly what we built by hand in Section 2. This is a great example of why understanding raw Redis data structures (Module 4) makes you a much stronger user of higher-level tools like BullMQ.

---

## 6. Interview Question: "Why use Redis for a job queue instead of just calling `setTimeout` or running the task inline?"

**Answer:** Running background work inline blocks the response and ties the work's lifetime to a single request/process — if the server restarts, in-memory `setTimeout` tasks vanish entirely. A Redis-backed queue persists jobs independently of any single request or process, supports automatic retries, delays, and priorities, and lets you scale workers horizontally (run more worker processes to process jobs faster) completely independently of your web server's scaling needs.

---

## 7. Summary

- Queues let time-consuming or non-critical work happen in the background, keeping user-facing responses fast.
- A basic FIFO queue can be built with Redis Lists + `BLPOP` (blocking pop, avoids wasteful polling).
- Hand-built queues lack retries, delays, priorities, and crash-safety — which is exactly what **BullMQ** provides on top of Redis.
- BullMQ separates **Queue** (adding jobs) from **Worker** (processing jobs), and supports retries with backoff, delayed jobs, and priorities out of the box.

---

## 8. Homework

1. Build the hand-written producer/worker pair using `RPUSH`/`BLPOP` and confirm jobs are processed in order.
2. Install BullMQ and recreate the same welcome-email flow using `Queue` and `Worker`.
3. Add a job that intentionally throws an error, and observe BullMQ's automatic retry behavior in the console.

---

**Next up:** [Module 16 — Redis Cluster & Replication](./16-cluster-and-replication.md)
