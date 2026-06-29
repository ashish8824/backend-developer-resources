# Day 30 — Final Mock Interview: Coding, System Design, Behavioral

> **Why this day matters:** Knowledge and interview PERFORMANCE are different skills. You've spent 29 days building the knowledge. Today simulates the actual format — a live coding segment, a system design walkthrough, and behavioral questions — under realistic time pressure, so the FIRST time you experience this format isn't in the real interview. Treat today seriously: set actual timers, speak your answers out loud (even alone), and resist the urge to peek at the discussion notes before attempting each section.

---

## 1. How to Run Today

Block out roughly 90 minutes, structured like a real interview loop:
- **20 minutes** — Live coding exercise (Section 2)
- **25 minutes** — System design walkthrough (Section 3), using Day 29's framework
- **20 minutes** — Behavioral/experience questions (Section 4)
- **25 minutes** — Review weak spots from your Day 29 self-assessment

If possible, do this with another person reading the prompts to you (a friend, a study partner) rather than reading them yourself — having to respond to a prompt delivered by someone else, in real time, is a meaningfully different experience than reading it at your own pace.

---

## 2. Live Coding Round (20 minutes)

### Exercise: Implement a rate limiter middleware from scratch

**The prompt, exactly as it might be given to you:**
> "Write an Express middleware that limits each IP address to 5 requests per 10-second window, using Redis. If the limit is exceeded, return a 429 status with a clear error message and appropriate headers."

**Attempt this fully before reading the discussion below.** Use the patterns from Day 18/22, but write it from memory, not by copying.

<details>
<summary>Discussion notes (only open after attempting)</summary>

A strong solution demonstrates:
- Using `INCR` + `EXPIRE` for atomicity on first request in the window (Day 17's race-condition reasoning)
- Setting `X-RateLimit-*` and `Retry-After` headers (Day 22)
- Handling the Redis-unavailable case explicitly — and being ready to explain the fail-open/fail-closed tradeoff if asked (Day 22)
- Clean error handling that doesn't crash the app if Redis throws

A common follow-up in a real interview: *"How would this need to change if you wanted DIFFERENT limits for free vs. paid users?"* — be ready to extend your own solution live (identify by `req.user.tier` instead of just `req.ip`, look up the limit config based on tier).

Another common follow-up: *"What if this needs to run across 5 server instances?"* — confirm out loud that it already DOES work correctly across instances, specifically BECAUSE you used Redis as the shared store (Day 15/18's lesson) — this is a great moment to proactively explain WHY, not just confirm that it does.
</details>

### Exercise: Debug this broken code

**The prompt:**
```javascript
// This Express route is supposed to fetch a user and their orders,
// but it's behaving incorrectly. Find and explain ALL the bugs.
app.get('/users/:id/summary', (req, res) => {
  const user = User.findByPk(req.params.id);
  const orders = Order.findAll({ where: { userId: user.id } });

  let total = 0;
  orders.forEach(async (order) => {
    total += order.amount;
  });

  res.json({ user, orderCount: orders.length, total });
});
```

**Attempt to find every bug before reading the discussion.**

<details>
<summary>Discussion notes</summary>

1. **Missing `async`/`await` on the route handler entirely** — `User.findByPk` and `Order.findAll` return Promises (Sequelize, Day 11), but neither is awaited, so `user` and `orders` are Promise objects, not actual data.
2. **`user.id` accessed on a Promise, not the resolved user** — would throw or be `undefined`, compounding bug #1.
3. **`orders.forEach(async ...)` doesn't wait for anything** — directly from Day 3's lesson: `forEach` doesn't await async callbacks, so `res.json()` runs before `total` is actually fully calculated, AND before `orders` itself is even real data given bug #1.
4. **No error handling at all** — no try/catch, so if `findByPk`/`findAll` reject, this becomes an unhandled rejection in the route handler (Day 3/13).
5. **No check if `user` exists** — if the ID doesn't match any user, this would proceed with `user` as `null`, likely throwing when accessing `.id`, instead of returning a clean 404 (Day 9).

A corrected version should use `await` properly, a `for...of` loop (or just `.reduce()`, since this doesn't need to be async at all — summing numbers is synchronous) instead of `forEach` with async, a try/catch with `next(err)`, and an explicit not-found check. Being able to name FIVE distinct, real bugs in one short snippet — and explain WHY each one is a bug, citing the relevant underlying concept — is exactly the depth a 3 YOE interview expects.
</details>

---

## 3. System Design Walkthrough (25 minutes)

**The prompt:**
> "Design a system that lets users upload a video, which then gets processed (transcoded into multiple resolutions) and made available for streaming. Walk me through your design."

**Use Day 29's exact framework: Clarify → High-level architecture → Deep dive → Bottlenecks → Wrap-up. Set a timer for 15 minutes and talk through your full answer out loud before checking the discussion notes.**

<details>
<summary>Discussion notes — what a strong answer touches</summary>

**Clarify:** expected video sizes/lengths, how many resolutions needed, real-time processing requirement or is delay acceptable, expected upload volume.

**High-level architecture:**
```
Client ──> API server (accepts upload, returns IMMEDIATELY with a job ID)
              │
              ▼
        Object storage (e.g., S3 -- raw video file stored here, NOT in your database)
              │
              ▼
        Message Queue (Day 19) ──> Transcoding Workers (using ffmpeg via
                                     child_process/spawn -- Day 16!)
              │
              ▼
        Object storage (transcoded outputs) ──> CDN (Day 24) for actual streaming delivery
```

**Deep dive — the parts that matter most here:**
- Why NOT process the video synchronously in the upload request: transcoding takes minutes, would time out an HTTP request and block the API server's event loop/resources (Day 2, Day 19's entire motivation) — accept the upload, respond immediately with a job ID/status, queue the actual work.
- Using `spawn` (Day 16) specifically (not `exec`) for invoking ffmpeg, given large/streaming output and avoiding shell injection risk if any filename/parameter is user-influenced.
- Idempotency (Day 19) — if a transcoding job is retried after a worker crash mid-job, it shouldn't produce duplicate output files or corrupt partial state.
- Client needs a way to know when processing is done — either polling a status endpoint, a WebSocket push notification (Day 20) once complete, or a webhook.
- Serving the FINISHED video through a CDN (Day 24), not directly from your application servers, since video files are large, static once produced, and benefit enormously from edge caching.

**Bottlenecks:** transcoding is CPU-intensive — worker capacity needs to scale independently from the API layer (a microservices-style separation, Day 25, even if the rest of the system is otherwise a monolith), and you'd likely run MULTIPLE transcoding workers consuming from the same queue, scaled based on queue depth.

**Wrap-up:** mention monitoring queue depth/processing time as a key operational metric, and note you'd want to explore storage cost optimization (e.g., not keeping every resolution forever) with more time.

**If you covered most of these points, even imperfectly, you demonstrated working knowledge of FIVE separate days of this course (2, 16, 19, 20, 24, 25) applied cohesively to one problem — exactly the kind of synthesis a 3 YOE system design round is checking for.**
</details>

---

## 4. Behavioral / Experience-Based Questions (20 minutes)

These test communication and reflection, not just technical recall — but they're frequently UNDERPREPARED for, even by strong technical candidates. For each, prepare a real example from your own experience using a simple structure: **Situation → Action → Result → What you'd do differently/learned.** If you don't have 3 years of REAL experience yet, adapt these using your most substantial project work, and be honest about that framing rather than fabricating experience.

1. **"Tell me about a time you debugged a difficult production issue."**
   *A strong answer cites SPECIFIC technical detail — not "the server was slow," but something like "memory usage was climbing steadily over 3 days, and I used heap snapshots to trace it to forgotten EventEmitter listeners being added on every request" (Day 28). Specificity is what separates a real story from a generic one.*

2. **"Tell me about a technical decision you made that you'd reconsider now."**
   *A strong answer shows genuine reflection, not false humility — e.g., "I chose MongoDB for a project that turned out to have heavily relational data with complex queries; a relational database with proper joins would have been a better fit, and I learned to weigh data shape more carefully upfront" (Day 10's SQL vs NoSQL framing).*

3. **"How do you approach learning a new technology or concept you're unfamiliar with on the job?"**
   *Connect this to how you're using THIS course right now — building working examples, understanding the "why" behind a pattern, not just copying syntax, and validating understanding by explaining concepts out loud or teaching them to someone else.*

4. **"Describe a disagreement you had with a teammate about a technical approach, and how it was resolved."**
   *Strong answers focus on the REASONING process (how you evaluated tradeoffs together) rather than who "won," and acknowledge that reasonable disagreements often come down to genuine tradeoffs (e.g., a Day 25-style "monolith vs. microservices" disagreement) rather than one side being simply wrong.*

5. **"What's a piece of feedback you received that changed how you work?"**
   *Be specific and genuine — vague answers here are easy to spot and don't build trust with an interviewer.*

6. **"Why are you looking for a new role / why this company?"**
   *Not technical, but always asked — have a genuine, specific answer ready, not a generic one that could apply to any company.*

---

## 5. Final Review — Closing the Gaps

Go back to your Day 29 self-assessment checklist. For anything rated below a 3:

1. Re-open that specific day's markdown file.
2. Read ONLY the "Interview Q&A Recap" section first — if you can answer all of those confidently now, you may already be in better shape than you thought.
3. If any individual Q&A answer still feels shaky, re-read that day's relevant section in full, then re-attempt the Q&A out loud.
4. Re-attempt that day's "Hands-on practice" exercise if you skipped it the first time — concepts that survive being typed out and run are retained far better than concepts only read.

---

## 6. You've Completed the 30-Day Roadmap

Here's what you've actually built, end to end, across these 30 days:

- **Days 1-7:** Node.js runtime internals — modules, the event loop, async patterns, core modules, streams, and the raw HTTP module underneath every framework.
- **Days 8-14:** A complete, production-patterned REST API — Express, REST design conventions, MongoDB and SQL with real ORMs, JWT authentication with RBAC, and centralized error handling/security — culminating in one full working CRUD API with proper authorization checks.
- **Days 15-21:** The advanced topics that distinguish 3 YOE engineers — `cluster` and Worker Threads, `child_process`, Redis across caching/pub-sub/sessions/rate-limiting, message queues with idempotency, WebSocket scaling, and a real testing strategy with mocking.
- **Days 22-28:** System design and DevOps — rate limiting algorithms as a system design topic, load balancing, multi-layer caching, honest microservices tradeoffs, Docker, CI/CD with graceful shutdown, and production performance/monitoring.
- **Days 29-30:** Interview execution — a repeatable system design framework, a cross-course rapid-recall question bank, and today's full mock interview simulating the real format.

**A genuinely important closing point:** the goal was never to memorize 30 files of notes — it was to build a mental model where Redis, the event loop, clustering, and caching all connect to EACH OTHER, the way they actually do in real systems. If you found yourself, by Day 25 or so, naturally predicting "oh, this is probably another multi-instance state-sharing problem, Redis is probably the answer" BEFORE reading the day's content — that's the actual signal you're ready. That instinct, more than any single fact, is what a 3 YOE backend interview is truly trying to assess.

Good luck.
