# Day 27 — CI/CD, PM2 in Production, Environment Strategy, Graceful Shutdown

> **Why this day matters:** This is the day that bridges "I can build an app" and "I can be trusted to ship and operate it in production." Interviewers ask about CI/CD and graceful shutdown specifically because they reveal whether you've actually been ON-CALL or responsible for a real deployed service, versus only ever running `node app.js` locally and pushing code over a wall to someone else.

---

## 1. Background: what does CI/CD actually automate, and why?

**CI (Continuous Integration):** automatically building and testing every code change (typically on every pull request or every push) — the goal is to catch bugs and integration problems EARLY, before they reach production, and to make sure the codebase is ALWAYS in a working, mergeable state rather than discovering breakage only after merging several people's changes together.

**CD (Continuous Delivery/Deployment):** automatically deploying code that passes CI to an environment (staging, then production) — removing manual, error-prone deployment steps (a person SSHing into a server, manually pulling code, manually restarting the app) in favor of a consistent, repeatable, auditable process.

**Why this matters beyond "it's more convenient":** manual deployment processes are a significant source of real production incidents — a forgotten step, a typo in a deploy command, deploying the wrong branch, skipping tests "just this once" under time pressure. CI/CD removes human error from a process that benefits enormously from being mechanical and repeatable EVERY single time.

```
Developer pushes code
        │
        ▼
┌─────────────────┐
│   CI Pipeline     │  -- Lint code, run tests (Day 21!), build the app/Docker image (Day 26!)
└────────┬─────────┘
         │ (only if ALL of the above succeed)
         ▼
┌─────────────────┐
│   CD Pipeline      │  -- Deploy to staging, run further checks, then deploy to production
└─────────────────┘
```

---

## 2. A Basic GitHub Actions Workflow for a Node.js App

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

# WHEN this workflow runs -- triggers
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    # Spinning up a Postgres service CONTAINER specifically for the
    # duration of this CI job -- directly applying Day 26's container
    # knowledge: this is a temporary, isolated database just for tests,
    # not your actual production database
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: test_db
        ports:
          - 5432:5432

    steps:
      # 'uses' references a REUSABLE, pre-built action -- checkout pulls
      # your repository's code into this CI runner's environment
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm' # caches node_modules between runs for faster CI -- same
                        # underlying motivation as Day 26's Docker layer caching

      - name: Install dependencies
        run: npm ci  # 'npm ci' (Day 1 callback!) instead of 'npm install' --
                       # stricter, faster, reproducible installs exactly
                       # matching package-lock.json -- ideal for CI specifically

      - name: Run linter
        run: npm run lint

      # Day 21's testing knowledge, now running automatically on every push
      - name: Run tests
        run: npm test
        env:
          DATABASE_URL: postgres://postgres:testpass@localhost:5432/test_db

      - name: Build Docker image
        run: docker build -t my-app:${{ github.sha }} .
        # ${{ github.sha }} tags the image with the EXACT commit hash --
        # a real production practice, so you can always trace exactly
        # which code is running in any given deployed image

  deploy:
    needs: test  # this job only runs if the 'test' job above succeeded ENTIRELY
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' # only deploy from the main branch, never from PR branches

    steps:
      - name: Deploy to production
        run: |
          echo "Deploying commit ${{ github.sha }} to production..."
          # Real deployment commands would go here -- pushing the Docker
          # image to a registry, triggering a deployment platform's API,
          # SSHing into servers to pull the new image, etc. -- the exact
          # mechanism varies a lot by hosting platform/company
```

**Interview tip:** if asked "walk me through what happens when you push code, in your ideal setup," the answer should mention: automated linting and testing BEFORE anything is allowed to merge or deploy, building an immutable, versioned artifact (a Docker image tagged with the commit hash, in this example) rather than deploying "whatever the server happens to have," and a deployment step that's GATED on all previous steps succeeding — this whole flow demonstrates you understand CI/CD as a SAFETY mechanism, not just an automation convenience.

---

## 3. PM2 in Production — completing Day 15's clustering discussion

### Background: what's PM2's actual job in a real deployed environment?

Recall Day 15: `cluster` lets Node use multiple cores, but you still need SOMETHING managing those worker processes — restarting crashed workers, aggregating logs, and (critically, for today) handling **zero-downtime deploys**. PM2 is the most common tool filling this role in real Node.js production deployments (alongside, or sometimes instead of, container orchestration like Kubernetes).

```bash
# Starting your app in cluster mode, one worker per CPU core (Day 15!)
pm2 start app.js -i max --name my-api

# Viewing running processes -- shows CPU/memory usage per worker, uptime,
# restart count (a HIGH restart count is itself a useful debugging signal --
# it usually means something is crashing repeatedly)
pm2 list

pm2 logs my-api          # aggregated logs across ALL workers
pm2 monit                # a live dashboard of CPU/memory usage

# ZERO-DOWNTIME RELOAD -- this is the genuinely important production
# feature PM2 provides beyond just "keep my app running"
pm2 reload my-api
```

### Zero-Downtime Reloads — how this actually works, and why it matters

**Background: what would happen WITHOUT this?** If you simply `pm2 restart` (or naively kill and restart) your app to deploy new code, there's a window of time where NO worker is available to handle incoming requests — every request arriving during that gap fails or hangs. For a production API serving real traffic, even a few seconds of total downtime on every single deploy is a real, user-visible problem, especially for a team deploying multiple times a day.

**`pm2 reload` solves this by restarting workers ONE AT A TIME**, rather than all at once: it starts a NEW worker with the updated code, waits for it to be ready, THEN stops one of the OLD workers — repeating this process across all workers, one by one. At every single moment during this process, there's always at least one worker available to handle traffic — true zero-downtime deployment.

```javascript
// For 'pm2 reload' to work correctly, your app needs to handle a
// specific signal gracefully -- this is EXACTLY why graceful shutdown
// (section 4 below) isn't just a "nice to have," it's a REQUIREMENT
// for zero-downtime deploys to actually work as intended.
```

```javascript
// ecosystem.config.js -- a more structured way to configure PM2,
// instead of passing everything as CLI flags
module.exports = {
  apps: [{
    name: 'my-api',
    script: 'app.js',
    instances: 'max',         // one worker per CPU core (Day 15)
    exec_mode: 'cluster',
    env_production: {
      NODE_ENV: 'production',
      PORT: 3000,
    },
    max_memory_restart: '500M', // automatically restart a worker if it
                                  // exceeds this memory usage -- a real
                                  // safety net against memory leaks
                                  // (Day 28's topic) causing a full outage
  }],
};
```

```bash
pm2 start ecosystem.config.js --env production
```

---

## 4. Graceful Shutdown — the detail that makes zero-downtime deploys actually safe

### Background: what goes wrong WITHOUT this, precisely?

When a process manager (PM2, or an orchestrator like Kubernetes) wants to stop a worker (for a deploy, a scale-down, or a restart), it sends a termination signal — typically `SIGTERM`. **If your app doesn't explicitly handle this signal, Node's default behavior is to terminate the process IMMEDIATELY** — abruptly cutting off ANY requests currently being processed by that worker, mid-flight, with no response ever sent to those clients. For a worker handling, say, 20 concurrent requests at the exact moment it's killed, all 20 of those users get a failed/hung request, even though the OVERALL deploy was supposed to be "zero-downtime."

```javascript
// THE FIX: explicitly listen for SIGTERM, and on receiving it, stop
// accepting NEW connections, but let IN-FLIGHT requests finish naturally
// before actually exiting the process.
const express = require('express');
const app = express();

// ... your routes ...

const server = app.listen(3000, () => {
  console.log('Server running on port 3000');
});

// Track whether we're in the process of shutting down, to avoid
// accepting confusing NEW work once a shutdown has been initiated
let isShuttingDown = false;

process.on('SIGTERM', () => {
  console.log('SIGTERM received, starting graceful shutdown...');
  isShuttingDown = true;

  // server.close() stops the server from accepting NEW connections,
  // but importantly does NOT immediately kill connections that are
  // ALREADY in progress -- it waits for them to finish naturally first.
  server.close(() => {
    console.log('All in-flight requests completed, exiting cleanly');

    // ALSO close other resources cleanly here -- database connections,
    // Redis connections, any open file handles -- an abrupt process
    // exit can leave these in an inconsistent state otherwise
    // (e.g., sequelize.close(), redisClient.quit())

    process.exit(0); // 0 = clean, successful exit (Day 1 callback!)
  });

  // A SAFETY NET: if for some reason in-flight requests never finish
  // (a hung request, a bug), don't let the process hang FOREVER --
  // force-exit after a reasonable timeout as a last resort.
  setTimeout(() => {
    console.error('Forcing shutdown after timeout -- some requests may not have completed');
    process.exit(1); // 1 = exit due to an error/forced condition
  }, 10000); // 10 second grace period -- tune based on your typical request duration
});

// A genuinely useful middleware addition: reject NEW requests
// immediately during the shutdown window, with a clear signal to load
// balancers/clients to retry elsewhere, rather than accepting work
// you're about to stop being able to serve.
app.use((req, res, next) => {
  if (isShuttingDown) {
    res.setHeader('Connection', 'close');
    return res.status(503).json({ error: 'Server is shutting down, please retry' });
  }
  next();
});
```

**Interview tip — this is a genuinely high-value answer if asked "how do you handle zero-downtime deploys":** mentioning `SIGTERM` handling, `server.close()`'s behavior of finishing in-flight requests while rejecting new ones, AND the safety-net timeout, demonstrates real production experience — this exact pattern is something many engineers learn only AFTER experiencing a bad deploy that dropped real user requests.

### Database/Redis connections during shutdown — a detail often missed

```javascript
process.on('SIGTERM', async () => {
  server.close(async () => {
    await sequelize.close();   // Day 11 — close DB connections cleanly
    await redisClient.quit();  // Day 17 — close Redis connections cleanly
    console.log('Cleanup complete, exiting');
    process.exit(0);
  });
});
```

**Why this matters beyond "it's tidy":** abruptly killing a process while it holds open database connections can leave those connections in a weird state on the database server's side (sometimes requiring the DB's own timeout to clean up), and can occasionally interfere with in-flight transactions (Day 11's ACID) that hadn't fully committed or rolled back.

---

## 5. Environment Strategy: Dev, Staging, Production

### Background: why three (or more) environments, specifically?

**Development** — where you write and test code locally, often with relaxed settings (verbose logging, no rate limiting, a local/seeded database) for fast iteration.
**Staging** — a near-exact replica of production (same infrastructure shape, similar data volume/scale where feasible) used to catch issues that ONLY show up in a production-like environment, BEFORE actual users are affected.
**Production** — the real, live environment serving real users — the place where mistakes have real consequences (real user data, real money, real reputation).

```javascript
// A common, real pattern: environment-specific configuration, loaded
// based on NODE_ENV (Day 13's env config knowledge, applied across
// MULTIPLE environments rather than just one local .env file)
const config = {
  development: {
    logLevel: 'debug',
    rateLimit: false, // disabled locally for easier manual testing
  },
  staging: {
    logLevel: 'info',
    rateLimit: true,
  },
  production: {
    logLevel: 'warn', // less noise, only real concerns logged
    rateLimit: true,
  },
};

const currentConfig = config[process.env.NODE_ENV] || config.development;
```

**Why staging specifically matters, beyond "more testing is always good":** some bugs genuinely ONLY appear under production-like conditions — a race condition that only shows up under real concurrent load (Day 17/22's atomicity concerns), a configuration difference (a missing environment variable, a different database version), or a scaling issue (Day 15/23's multi-instance problems) that simply doesn't manifest when running one local instance. Staging exists specifically to catch this category of bug BEFORE it reaches real users.

---

## 6. How this connects to real backend work (3 YOE framing)

- **"Walk me through what happens when you push a commit, all the way to it running in production."** → CI runs lint/tests/build (gated, must all pass), CD deploys to staging first, further validation, then deploys to production — ideally with zero downtime via rolling/graceful worker replacement.
- **"You deployed a new version and noticed a spike in failed requests during the deploy itself — what would you check?"** → Whether the app properly handles `SIGTERM` for graceful shutdown — a missing handler means in-flight requests get abruptly dropped during every single deploy, not just this one.
- **"Why use `npm ci` instead of `npm install` in your CI pipeline?"** → `npm ci` performs a clean, strict install exactly matching `package-lock.json`, is faster (skips certain resolution steps `npm install` does), and fails outright if `package.json` and `package-lock.json` are out of sync — better suited for reproducible automated environments than `npm install`'s more lenient behavior.
- **"How would you decide what level of testing/validation happens in staging versus relying on production monitoring after deploy?"** → Discuss the tradeoff: staging catches issues before any real user is affected but can't perfectly replicate production scale/data; production monitoring (Day 28's topic) catches what staging inevitably misses — most mature teams rely on BOTH, not one instead of the other.

---

## 7. Hands-on practice for today

```javascript
// practice-day27.js -- a runnable demonstration of graceful shutdown
const express = require('express');
const app = express();

app.get('/slow', async (req, res) => {
  console.log('Slow request started...');
  await new Promise((resolve) => setTimeout(resolve, 5000)); // simulate a slow operation
  console.log('Slow request finished');
  res.json({ message: 'Done after 5 seconds' });
});

const server = app.listen(3000, () => console.log('Listening on port 3000'));

let isShuttingDown = false;

app.use((req, res, next) => {
  if (isShuttingDown) return res.status(503).json({ error: 'Shutting down' });
  next();
});

process.on('SIGTERM', () => {
  console.log('SIGTERM received');
  isShuttingDown = true;
  server.close(() => {
    console.log('Server closed gracefully');
    process.exit(0);
  });
});

process.on('SIGINT', () => process.emit('SIGTERM')); // lets Ctrl+C trigger the same graceful path locally
```

Run it (`node practice-day27.js`), start a slow request (`curl http://localhost:3000/slow`), and WHILE that request is still running, send a termination signal from another terminal (`kill -TERM <pid>`, finding the PID via `ps aux | grep node`, or just press Ctrl+C). Watch the logs: the slow request should COMPLETE successfully despite the shutdown signal having been sent, proving the graceful shutdown logic is working exactly as intended.

---

## 8. Interview Q&A Recap (say these out loud)

1. **Q: What's the difference between Continuous Integration and Continuous Deployment?**
   A: CI automatically builds and tests every code change to catch problems early and keep the codebase always in a working state. CD automatically deploys code that passes those checks to an environment, removing manual, error-prone deployment steps.

2. **Q: Why does `pm2 reload` achieve zero-downtime deploys while a naive restart doesn't?**
   A: It restarts cluster workers one at a time — starting a new worker with updated code, confirming it's ready, then stopping an old one, repeating across all workers — so at least one worker is always available to handle traffic throughout the entire process.

3. **Q: What happens to in-flight requests if your app doesn't handle `SIGTERM`, and how do you fix it?**
   A: Node's default behavior terminates the process immediately, abruptly dropping any requests currently being processed. The fix is explicitly listening for `SIGTERM`, calling `server.close()` to stop accepting new connections while letting in-flight ones finish, then exiting cleanly — ideally with a timeout safety net in case something hangs.

4. **Q: Why use `npm ci` instead of `npm install` in a CI pipeline?**
   A: It performs a strict, reproducible install exactly matching `package-lock.json`, is generally faster, and fails outright if the lockfile and `package.json` are out of sync — more appropriate for automated, reproducible environments than `npm install`'s more lenient behavior.

5. **Q: Why have a staging environment at all if you already have thorough automated tests?**
   A: Some issues only manifest under production-like conditions — race conditions under real concurrent load, configuration differences, or scaling issues that don't appear when running a single local instance — staging catches this category of bug before real users are affected.

6. **Q: Besides closing the HTTP server, what else should a graceful shutdown handler clean up?**
   A: Database connections, Redis connections, and any other open resources or in-flight transactions — abruptly killing a process while these are open can leave connections in an inconsistent state on the other end, or interfere with transactions that hadn't fully committed or rolled back.

---

## Tomorrow (Day 28) Preview
We cover **performance and monitoring**: identifying and fixing memory leaks, profiling tools (`--inspect`, clinic.js), understanding garbage collection at a practical level, and what to actually monitor in a production Node.js application — closing out the System Design & DevOps week before the final two days of dedicated interview preparation.
