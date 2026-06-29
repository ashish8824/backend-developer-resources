# Day 20: Health Checks — Liveness, Readiness & Startup Probes Mastered

**Goal for today:** Go beyond the basic probe definitions from Day 13 and truly master health check design — understanding the exact mechanics of each probe type, how probes integrate with rolling updates and Service endpoints, the precise timing parameters that prevent false positives and outages, real-world probe patterns for Node.js, Java, PostgreSQL, and Redis workloads, and the most common production mistakes. By the end, you should be able to design the health check strategy for any workload from scratch.

**Format:** WHAT → WHY → HOW for every concept.

---

## 0. Quick Recap (open your teaching session with this)

- Day 13: Pods — introduced probes conceptually (liveness restarts, readiness removes from endpoints)
- Day 14: Deployments — probes gating rolling updates
- Day 15: Services — probes controlling which Pods receive traffic via Endpoints

**Today's connecting thought:**
> Probes appeared in Day 13 as a "nice to have." Today you'll see they are load-bearing production infrastructure. A misconfigured probe can restart healthy pods, route traffic to broken pods, or cause a rolling update to stall forever. Getting probes right is one of the single biggest factors between a flaky cluster and a stable one.

---

## 1. The Three Probes — Precise Definitions Revisited

```
┌──────────────────────────────────────────────────────────────────────┐
│                     CONTAINER LIFECYCLE                               │
│                                                                       │
│  Container starts                                                     │
│       │                                                               │
│       ▼                                                               │
│  [STARTUP PROBE]  ← "Has the app finished starting up?"              │
│       │ passes                                                        │
│       ▼                                                               │
│  [READINESS PROBE] ← "Is the app ready to receive traffic?"          │
│  [LIVENESS PROBE]  ← "Is the app still alive and functioning?"       │
│       │                                                               │
│       │  Liveness fails   → kubelet restarts container               │
│       │  Readiness fails  → removed from Service endpoints           │
│       │  Startup fails    → kubelet restarts container (during start)│
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

### The One-Sentence Rule for Each

- **Liveness:** "If this fails, the container is broken beyond self-recovery — kill it and start fresh."
- **Readiness:** "If this fails, the container is alive but not ready for user traffic — remove it from the load balancer."
- **Startup:** "If this fails during startup, the container didn't start successfully — kill it and retry."

---

## 2. All Probe Parameters — Complete Reference

Every probe accepts these parameters:

```yaml
livenessProbe:
  httpGet:              # probe mechanism (httpGet | tcpSocket | exec | grpc)
    path: /health
    port: 3000

  # ── Timing Parameters ──────────────────────────────────────────────────────
  initialDelaySeconds: 15   # wait N seconds after container starts before first probe
                             # gives the app time to initialise before being checked
                             # too low: false failure before app ready
                             # too high: delays detection of actual startup failure

  periodSeconds: 20         # run probe every N seconds
                             # too low: unnecessary overhead, noisy
                             # too high: slow detection of failures

  timeoutSeconds: 5         # probe must respond within N seconds or counts as failure
                             # set > typical response time but < periodSeconds

  successThreshold: 1       # consecutive successes needed to mark as healthy
                             # liveness/startup: always 1 (can't be changed)
                             # readiness: can be > 1 (require multiple successes before accepting traffic)

  failureThreshold: 3       # consecutive failures before taking action
                             # liveness: restart after N failures
                             # readiness: remove from endpoints after N failures
                             # startup: restart after N failures
```

**Visualising the timing:**

```
Container starts
      │
      ├── wait initialDelaySeconds (15s)
      │
      ├── Probe 1 ──► response within timeoutSeconds? success/fail
      │
      ├── wait periodSeconds (20s)
      │
      ├── Probe 2 ──► success/fail
      │
      ├── wait periodSeconds (20s)
      │
      ├── Probe 3 ──► fail
      │
      └── fail count = failureThreshold (3) → ACTION (restart or remove from endpoints)
```

**Total time to detect failure:**
`initialDelaySeconds + (failureThreshold × periodSeconds) + (failureThreshold × timeoutSeconds)`

Example: `15 + (3 × 20) + (3 × 5)` = 15 + 60 + 15 = **90 seconds** from container start to first restart.

---

## 3. Liveness Probe — Design Philosophy

### WHAT it does
Liveness probe answers: "Is this container in a state it can never recover from on its own?"

When it fails `failureThreshold` times consecutively → **kubelet kills the container and restarts it**.

### WHEN to use it
✅ Use liveness for:
- Deadlock detection (app is running but completely stuck)
- Memory leak that causes OOM but doesn't kill the process
- Infinite loop or hung goroutine/thread pool
- Corrupted internal state that only a restart can fix

❌ Do NOT use liveness for:
- Temporary dependency outages (DB down → liveness restarts app → still no DB → restart loop)
- Slow responses under load (liveness times out → restarts healthy pod → makes load worse)
- Anything that causes a restart cascade

### The most dangerous liveness anti-pattern

```yaml
# ❌ NEVER DO THIS: liveness checks an external dependency
livenessProbe:
  httpGet:
    path: /health/deep    # this endpoint checks DB, Redis, external APIs
    port: 3000

# What happens when DB goes down:
# 1. DB down
# 2. /health/deep returns 503 (DB check fails)
# 3. liveness probe fails 3 times
# 4. Kubernetes RESTARTS your perfectly healthy API pod
# 5. Pod restarts, DB still down, same failure
# 6. CrashLoopBackOff — app is down even though it's fine, DB is the problem
```

**Teaching line:**
> "Liveness should only check the application itself — not its dependencies. A deadlocked app is a liveness problem. A database being down is not — your app is fine, the database isn't. Don't restart the app because its database is sick."

### Correct liveness probe patterns

```yaml
# ✅ CORRECT: liveness checks only the app process itself
livenessProbe:
  httpGet:
    path: /healthz        # returns 200 if process is alive and not deadlocked
    port: 3000            # does NOT check DB, Redis, or external services

# ✅ CORRECT: lightweight liveness check
livenessProbe:
  httpGet:
    path: /livez           # convention: /livez for liveness, /readyz for readiness
    port: 3000
  periodSeconds: 30        # check every 30s — not too frequent
  failureThreshold: 3
  timeoutSeconds: 5
  initialDelaySeconds: 30
```

### What /healthz should return

```javascript
// Node.js — liveness endpoint
// Only checks if the process itself is healthy
app.get('/healthz', (req, res) => {
  // Check: is the event loop responding?
  // Check: is memory within bounds?
  const memUsage = process.memoryUsage();
  const heapUsedMB = memUsage.heapUsed / 1024 / 1024;

  if (heapUsedMB > 500) {  // memory leak detection
    return res.status(503).json({
      status: 'unhealthy',
      reason: 'memory_pressure',
      heapUsedMB
    });
  }

  res.json({
    status: 'alive',
    uptime: process.uptime(),
    heapUsedMB: heapUsedMB.toFixed(2)
  });
});
```

---

## 4. Readiness Probe — Design Philosophy

### WHAT it does
Readiness probe answers: "Is this container currently able to handle user traffic?"

When it fails → **Pod is removed from Service Endpoints** (no new traffic routed to it).
When it passes again → **Pod is re-added to Endpoints** (traffic resumes automatically).

**The container is NOT restarted.** It just goes in/out of the load balancing pool.

### WHEN to use it
✅ Use readiness for:
- App startup warm-up (loading caches, connecting to DB, JVM class loading)
- Temporary overload (app is healthy but currently overwhelmed — stop sending more traffic)
- Dependency check (DB is down — don't send requests that will fail anyway)
- Graceful shutdown (stop receiving new requests before shutting down)

### Readiness is the RIGHT place to check dependencies

```javascript
// ✅ CORRECT: readiness checks dependencies
app.get('/readyz', async (req, res) => {
  const checks = {};

  // Check database connection
  try {
    await pool.query('SELECT 1');
    checks.database = 'connected';
  } catch (err) {
    checks.database = 'disconnected';
  }

  // Check Redis connection
  try {
    await redisClient.ping();
    checks.redis = 'connected';
  } catch (err) {
    checks.redis = 'disconnected';
  }

  // Check if we're still warming up
  const uptimeSeconds = process.uptime();
  if (uptimeSeconds < 5) {
    return res.status(503).json({
      status: 'warming_up',
      uptimeSeconds,
      checks
    });
  }

  // Only ready if all dependencies are available
  const allReady = checks.database === 'connected' && checks.redis === 'connected';
  const status = allReady ? 200 : 503;

  res.status(status).json({
    status: allReady ? 'ready' : 'not_ready',
    checks
  });
});
```

```yaml
# Readiness probe: checks dependencies — safe to do here
readinessProbe:
  httpGet:
    path: /readyz
    port: 3000
  initialDelaySeconds: 5    # start checking quickly
  periodSeconds: 10
  failureThreshold: 3       # 3 failures = remove from endpoints
  successThreshold: 1       # 1 success = add back to endpoints
```

### Readiness during rolling updates — why it's essential

```
Rolling update without readiness probe:
  New pod starts → K8s marks it ready immediately (container running)
  → Traffic sent to new pod before app connected to DB
  → Users see errors for 10+ seconds

Rolling update WITH readiness probe:
  New pod starts → readiness probe fails (connecting to DB...)
  → Pod NOT added to endpoints yet → no user traffic
  → DB connection established → /readyz returns 200
  → Pod added to endpoints → users see no errors
  → Old pod removed → complete zero-downtime update
```

---

## 5. Startup Probe — Design Philosophy

### WHAT it does
Startup probe answers: "Has this container finished its initialisation sequence?"

While startup probe is running → **liveness and readiness probes are PAUSED**.
When startup passes → liveness and readiness probes begin normally.
When startup fails `failureThreshold` times → container is **restarted**.

### WHEN to use it
Use startup probe for any application that takes a long time to start:
- Java/Spring Boot applications (JVM startup, dependency injection)
- Applications loading large ML models
- Apps with heavy initialisation sequences
- Legacy apps with unpredictable startup times

### The problem startup probes solve

```
Without startup probe — Java app that takes 3 minutes to start:

Option A: Set initialDelaySeconds: 180
  → Fast-failing apps wait 3 full minutes before liveness checks start
  → Dead pods not detected for 3 minutes
  → Unacceptable for fast apps

Option B: Low initialDelaySeconds (e.g., 15)
  → Java app still loading at 15s
  → Liveness fires → fails → restarts Java app
  → Java app restarts → liveness fires at 15s again → restart loop
  → App NEVER starts

With startup probe:
  → startupProbe runs during initialisation (up to 5 minutes allowed)
  → Liveness/readiness completely paused
  → Startup passes after 2.5 minutes
  → Liveness/readiness begin with their own separate timers
  → Normal operations
```

```yaml
# Startup probe for a slow-starting Java application
startupProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  failureThreshold: 30    # allow up to 30 × 10s = 5 minutes for startup
  periodSeconds: 10
  timeoutSeconds: 5
  # NO initialDelaySeconds needed — startup probe runs from the very beginning

# Liveness only starts AFTER startup probe passes
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  periodSeconds: 30
  failureThreshold: 3
  timeoutSeconds: 5

# Readiness also only starts AFTER startup probe passes
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  periodSeconds: 10
  failureThreshold: 3
```

---

## 6. Probe Mechanisms — Deep Dive

### 6.1 HTTP GET Probe

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 3000
    scheme: HTTPS         # HTTP (default) | HTTPS
    httpHeaders:          # custom headers (e.g., Host header for virtual hosting)
      - name: X-Health-Check
        value: kubernetes
      - name: Accept
        value: application/json
```

**Returns success if:** HTTP status code ≥ 200 and < 400.

---

### 6.2 TCP Socket Probe

```yaml
# Passes if a TCP connection can be established on the port
livenessProbe:
  tcpSocket:
    port: 5432      # postgres port
  initialDelaySeconds: 30
  periodSeconds: 10
```

**Use for:** non-HTTP services (PostgreSQL, MySQL, Redis, Kafka) where you just want to verify the port is accepting connections.

---

### 6.3 Exec Probe

```yaml
# Passes if the command exits with code 0
livenessProbe:
  exec:
    command:
      - /bin/sh
      - -c
      - "pg_isready -U postgres -h localhost"

# OR for Redis:
livenessProbe:
  exec:
    command:
      - redis-cli
      - ping
      # exits 0 if Redis responds with PONG
```

**Use for:** databases and services with their own health-check CLIs.

---

### 6.4 gRPC Probe (Kubernetes 1.24+)

```yaml
# For gRPC services implementing the standard health protocol
livenessProbe:
  grpc:
    port: 50051
    service: "myservice.MyService"
```

---

## 7. Real-World Probe Configurations by Workload

### 7.1 Node.js / Express API

```yaml
# Implement these endpoints in your app:
# GET /healthz  → liveness (process alive, not deadlocked)
# GET /readyz   → readiness (DB connected, dependencies ready)

startupProbe:
  httpGet:
    path: /healthz
    port: 3000
  failureThreshold: 10     # 10 × 5s = 50s max startup time
  periodSeconds: 5

livenessProbe:
  httpGet:
    path: /healthz
    port: 3000
  initialDelaySeconds: 0   # startup probe handles the delay
  periodSeconds: 30        # check every 30s — not too noisy
  failureThreshold: 3      # 3 × 30s = 90s to detect failure
  timeoutSeconds: 5

readinessProbe:
  httpGet:
    path: /readyz
    port: 3000
  initialDelaySeconds: 0
  periodSeconds: 10        # check frequently — faster traffic routing
  failureThreshold: 3
  successThreshold: 1
  timeoutSeconds: 5
```

```javascript
// Express health endpoints
const db = require('./db');
const redis = require('./redis');
const startTime = Date.now();

// Liveness: is the process alive and not stuck?
app.get('/healthz', (req, res) => {
  res.json({
    status: 'alive',
    uptime: ((Date.now() - startTime) / 1000).toFixed(0) + 's'
  });
});

// Readiness: are all dependencies available?
app.get('/readyz', async (req, res) => {
  try {
    await Promise.all([
      db.query('SELECT 1'),
      redis.ping()
    ]);
    res.json({ status: 'ready', db: 'up', redis: 'up' });
  } catch (err) {
    res.status(503).json({ status: 'not_ready', error: err.message });
  }
});
```

---

### 7.2 Java / Spring Boot

Spring Boot Actuator provides built-in liveness and readiness endpoints out of the box.

```yaml
# application.yml
management:
  endpoint:
    health:
      probes:
        enabled: true
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true
```

```yaml
startupProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  failureThreshold: 30     # 30 × 10s = 5 minutes for JVM startup
  periodSeconds: 10
  timeoutSeconds: 5

livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  periodSeconds: 30
  failureThreshold: 3
  timeoutSeconds: 5

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  periodSeconds: 10
  failureThreshold: 3
  timeoutSeconds: 5
```

---

### 7.3 PostgreSQL

```yaml
livenessProbe:
  exec:
    command:
      - sh
      - -c
      - pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}
  initialDelaySeconds: 30    # postgres takes time to initialise on first run
  periodSeconds: 10
  failureThreshold: 6        # be generous — postgres is stateful and critical
  timeoutSeconds: 5

readinessProbe:
  exec:
    command:
      - sh
      - -c
      - pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 3
  timeoutSeconds: 5
```

---

### 7.4 Redis

```yaml
livenessProbe:
  exec:
    command:
      - redis-cli
      - ping
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 3
  timeoutSeconds: 3

readinessProbe:
  exec:
    command:
      - redis-cli
      - ping
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
  timeoutSeconds: 3
```

---

### 7.5 Nginx / Static Frontend

```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 30
  failureThreshold: 3
  timeoutSeconds: 5

readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 3
  periodSeconds: 10
  failureThreshold: 3
```

---

## 8. Probes and Rolling Updates — The Exact Integration

This is the section that ties Days 14, 15, and 20 together. The rolling update algorithm uses probes at every step.

```
Rolling update triggered (maxSurge: 1, maxUnavailable: 0, minReadySeconds: 10)

Step 1: Kubernetes creates new Pod (v2)
Step 2: startupProbe runs until it passes (or fails → restart)
Step 3: readinessProbe runs
Step 4: Pod passes readiness → added to Service Endpoints
Step 5: minReadySeconds timer starts (10 seconds)
Step 6: After 10 seconds still healthy → Kubernetes considers this Pod "available"
Step 7: NOW it removes an old Pod (v1)
Step 8: Repeat until all Pods are replaced

If readiness never passes:
  → New Pod stays out of Endpoints (no user traffic)
  → Rolling update pauses — it won't remove old Pods
  → Old Pods keep serving traffic
  → kubectl rollout status shows: "Waiting for deployment to finish..."
  → You need to either fix the new image OR run kubectl rollout undo
```

**`minReadySeconds` — the extra safety buffer:**
```yaml
spec:
  minReadySeconds: 30   # Pod must stay healthy for 30s after readiness passes
                         # before Kubernetes considers it truly "available"
                         # prevents "flash of readiness" from broken pods
                         # that pass readiness briefly then fail

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0    # combined with minReadySeconds: safest possible rollout
      maxSurge: 1
```

---

## 9. Graceful Shutdown — Probes at Termination Time

When a Pod is being terminated, there's a subtle race condition that probes help solve:

```
Pod deletion requested
      │
      ├── 1. Pod removed from Service Endpoints (immediately)
      │      (kube-proxy updates iptables rules)
      │      ← BUT: iptables update takes ~1-2 seconds to propagate
      │             Requests might STILL be routed here during propagation
      │
      ├── 2. preStop hook runs (if defined)
      │      ← Use this to add a sleep: gives time for iptables to update
      │
      ├── 3. SIGTERM sent to container
      │      ← App begins graceful shutdown
      │
      ├── 4. terminationGracePeriodSeconds countdown (default: 30s)
      │
      └── 5. SIGKILL if still running after grace period

Best practice to handle the propagation race:
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5"]
    # This 5-second sleep gives iptables propagation time to complete
    # before SIGTERM is sent and the app starts refusing connections
```

```javascript
// Node.js graceful shutdown — handle SIGTERM properly
const server = app.listen(3000);

process.on('SIGTERM', () => {
  console.log('SIGTERM received — beginning graceful shutdown');

  // Stop accepting new connections
  server.close(() => {
    console.log('HTTP server closed');

    // Close DB connections
    pool.end(() => {
      console.log('DB pool closed');
      process.exit(0);
    });
  });

  // Force exit after 25s (before K8s sends SIGKILL at 30s)
  setTimeout(() => {
    console.error('Forced shutdown after timeout');
    process.exit(1);
  }, 25000);
});
```

---

## 10. Common Probe Mistakes and How to Fix Them

| Mistake | Consequence | Fix |
|---|---|---|
| Liveness checks external dependencies | CrashLoopBackOff when DB is down | Liveness: check only the app process. Readiness: check dependencies |
| `initialDelaySeconds` too low | Pod restarts before app is ready | Use startupProbe instead, or increase initialDelaySeconds |
| `periodSeconds` too low | Probe overhead, noisy logs | Liveness: 30s. Readiness: 10-15s |
| `timeoutSeconds` > `periodSeconds` | Probes overlap — unpredictable | Always `timeoutSeconds < periodSeconds` |
| No `failureThreshold` (uses default 3) | Acceptable for most cases | Increase for stateful/critical services |
| No probes at all | Traffic sent to broken pods, broken pods never restarted | Always configure both liveness and readiness |
| Single `/health` for both probes | Liveness restarts pod when DB is down | Separate `/healthz` (liveness) and `/readyz` (readiness) |
| `successThreshold > 1` on liveness | Configuration error — rejected by K8s | liveness `successThreshold` must be 1 |
| Slow health endpoint | Probe times out causing false failure | Keep health endpoints fast (<100ms), no slow DB queries |
| No startup probe for slow apps | Liveness restarts app during startup | Add startupProbe with generous `failureThreshold` |

---

## 11. Diagnosing Probe Failures

```bash
# ── See probe failure events ───────────────────────────────────────────────────
kubectl describe pod <pod-name>
# Look for Events like:
# Warning  Unhealthy  2m  kubelet  Liveness probe failed: HTTP probe failed with statuscode: 503
# Warning  Unhealthy  1m  kubelet  Readiness probe failed: Get "http://10.0.0.5:3000/readyz": EOF

# ── Check why probe is failing from outside ────────────────────────────────────
# Forward the pod port and test the health endpoint manually
kubectl port-forward pod/<pod-name> 8080:3000
curl http://localhost:8080/healthz
curl http://localhost:8080/readyz

# ── Test the probe from inside the container ───────────────────────────────────
kubectl exec -it <pod-name> -- wget -qO- http://localhost:3000/healthz
kubectl exec -it <pod-name> -- curl http://localhost:3000/readyz

# ── See real-time probe events ──────────────────────────────────────────────────
kubectl get events -n production --field-selector reason=Unhealthy --watch

# ── Check if pod is in/out of endpoints ───────────────────────────────────────
kubectl get endpoints <service-name>
# If pod IP is missing: readiness probe is failing

# ── Check CrashLoopBackOff details ────────────────────────────────────────────
kubectl describe pod <pod-name> | grep -A 5 "Last State"
# Shows exit code and reason for previous container death

# ── Check container restart count ─────────────────────────────────────────────
kubectl get pods
# RESTARTS column — high number = liveness probe is triggering too often
```

---

## 12. Complete Probe Configuration — TaskAPI Production

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: taskapi
  namespace: production
spec:
  replicas: 3
  minReadySeconds: 15        # pod must stay healthy 15s before replacing old pod
  selector:
    matchLabels:
      app: taskapi
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    metadata:
      labels:
        app: taskapi
    spec:
      containers:
        - name: api
          image: taskapi:2.0
          ports:
            - containerPort: 3000

          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"

          # ── Startup Probe ────────────────────────────────────────────────────
          # Node.js starts fast — 30s max is plenty
          startupProbe:
            httpGet:
              path: /healthz
              port: 3000
            failureThreshold: 6      # 6 × 5s = 30s max startup
            periodSeconds: 5
            timeoutSeconds: 3

          # ── Liveness Probe ───────────────────────────────────────────────────
          # Check only process health — NOT DB/Redis
          livenessProbe:
            httpGet:
              path: /healthz         # lightweight: just process alive check
              port: 3000
            periodSeconds: 30        # check every 30s
            failureThreshold: 3      # restart after 3 consecutive failures
            timeoutSeconds: 5        # must respond within 5s
            successThreshold: 1      # must be 1 for liveness

          # ── Readiness Probe ──────────────────────────────────────────────────
          # Check process + dependencies — gates traffic routing
          readinessProbe:
            httpGet:
              path: /readyz          # checks: DB connection, Redis connection
              port: 3000
            periodSeconds: 10        # check every 10s
            failureThreshold: 3      # remove from endpoints after 3 failures
            successThreshold: 1      # re-add after 1 success
            timeoutSeconds: 5

          # ── Graceful Shutdown ────────────────────────────────────────────────
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]

      terminationGracePeriodSeconds: 35   # 5s preStop + 25s drain + 5s buffer
```

---

## 13. Hands-On Practice Tasks

```bash
# 1. Implement /healthz and /readyz endpoints in your TaskAPI

# 2. Deploy with probes and watch startup sequence
kubectl apply -f deployment.yaml
kubectl get pods -w   # watch: Init → Running, notice startup probe
kubectl describe pod <pod-name> | grep -A 20 "Conditions"

# 3. Verify readiness gate is working
kubectl get endpoints taskapi-service
# Pod IP should appear only after readiness passes

# 4. Simulate liveness failure
kubectl exec -it <pod-name> -- sh
# Inside pod: kill the app or make /healthz return 503
# Watch K8s restart the container
kubectl get pods -w   # watch RESTARTS increment

# 5. Simulate readiness failure (DB down)
kubectl scale deployment postgres --replicas=0
# Watch pod disappear from endpoints
kubectl get endpoints taskapi-service -w
# Bring DB back
kubectl scale deployment postgres --replicas=1
# Watch pod reappear in endpoints

# 6. Watch a rolling update with probes
kubectl set image deployment/taskapi api=taskapi:3.0
kubectl rollout status deployment/taskapi
kubectl get pods -w   # observe startup → readiness → old pod removed

# 7. Simulate broken new image (failing startup)
kubectl set image deployment/taskapi api=taskapi:broken
kubectl rollout status deployment/taskapi   # should hang
kubectl get pods   # new pods in CrashLoopBackOff
kubectl rollout undo deployment/taskapi     # rescue
```

---

## 14. Quiz — Test Yourself / Test Others

1. What is the fundamental difference in action taken when liveness fails vs readiness fails?
2. Why should liveness probe NOT check external dependencies like databases?
3. When does a startup probe disable liveness and readiness probes?
4. Calculate the maximum time from container start to first restart given: `initialDelaySeconds: 20`, `periodSeconds: 15`, `failureThreshold: 3`, `timeoutSeconds: 5`.
5. A pod is in `CrashLoopBackOff`. What probe is most likely causing it?
6. A pod is Running but not in the Service Endpoints. What probe is failing?
7. Your Java app takes 4 minutes to start. What probe prevents it from being killed during startup and how do you configure it?
8. What does `minReadySeconds: 30` do in a Deployment spec?
9. What is the `successThreshold` constraint on a liveness probe?
10. Describe the race condition during Pod termination and how `preStop` sleep solves it.

<details>
<summary>Answers</summary>

1. Liveness failure: after `failureThreshold` consecutive failures, kubelet **kills and restarts the container**. The container gets a fresh start. Readiness failure: after `failureThreshold` consecutive failures, the Pod's IP is **removed from the Service's Endpoints list** — no new traffic is routed to it. The container is NOT restarted. If readiness recovers, the Pod is re-added automatically.

2. If liveness checks an external dependency like a database, when the database goes down the liveness probe fails → Kubernetes restarts the API pod → API pod restarts but database is still down → liveness fails again → CrashLoopBackOff. The API pod is perfectly healthy — restarting it doesn't help. The correct approach: liveness checks only the application process itself; readiness checks dependencies and removes the pod from traffic when they're unavailable.

3. The startup probe disables liveness and readiness probes from the moment the container starts until the startup probe itself passes (or fails and triggers a restart). While startup is running, neither liveness nor readiness runs at all. Only after startup passes do liveness and readiness begin their own independent check cycles.

4. Total = initialDelaySeconds + (failureThreshold × periodSeconds) + (failureThreshold × timeoutSeconds) = 20 + (3 × 15) + (3 × 5) = 20 + 45 + 15 = **80 seconds** from container start to first restart.

5. The **liveness** probe is most likely causing CrashLoopBackOff — it is detecting a failure and restarting the container repeatedly. If the startup probe is failing, it also causes restarts early in the lifecycle. Check `kubectl describe pod` Events for "Liveness probe failed" messages.

6. The **readiness** probe is failing. A Pod is Running (liveness is passing, container is alive) but removed from Endpoints when readiness fails. Run `kubectl describe pod` and look for "Readiness probe failed" events. Check the /readyz endpoint manually with `kubectl port-forward`.

7. A **startup probe** prevents the liveness probe from running until the app finishes starting. Configure it with a generous `failureThreshold × periodSeconds` that covers the maximum startup time: `failureThreshold: 24, periodSeconds: 10` = up to 240 seconds (4 minutes). The liveness probe is completely paused until startup passes.

8. `minReadySeconds: 30` requires a newly created Pod to remain continuously healthy (all containers passing readiness) for 30 seconds before Kubernetes counts it as "available." This prevents a Pod that passes readiness briefly then immediately fails from causing an old Pod to be removed prematurely. It adds a stability verification window to every rolling update step.

9. The `successThreshold` for a liveness probe **must always be 1**. Kubernetes enforces this — any other value will be rejected by the API Server with a validation error. The reasoning: once a container fails liveness, it gets restarted; there's no concept of "needs 2 consecutive successes" for liveness since the container is fresh after restart. Only readiness allows `successThreshold > 1`.

10. When a Pod is deleted, it's immediately removed from Service Endpoints — but iptables rules on all nodes take 1-2 seconds to propagate. During this window, in-flight requests may still be routed to the terminating Pod. At roughly the same time, SIGTERM is sent. If the app immediately closes connections on SIGTERM, those still-arriving requests fail. The `preStop` sleep (e.g., `sleep 5`) delays SIGTERM by 5 seconds, giving iptables propagation time to complete everywhere before the app starts shutting down — eliminating dropped requests during graceful termination.

</details>

---

## 15. Summary

After today you can fully design and explain:
- The precise **action** each probe type takes on failure — restart (liveness/startup) vs endpoint removal (readiness)
- **All probe parameters**: `initialDelaySeconds`, `periodSeconds`, `timeoutSeconds`, `successThreshold`, `failureThreshold` with timing calculations
- **Liveness design**: check only the app process — never external dependencies
- **Readiness design**: check dependencies — the RIGHT place for DB/Redis checks
- **Startup probe**: pauses liveness/readiness during initialisation — essential for Java/slow apps
- **All probe mechanisms**: httpGet, tcpSocket, exec, gRPC with real implementations
- **Real workload configs**: Node.js, Spring Boot, PostgreSQL, Redis, Nginx
- **Probes + rolling updates**: how readiness gates the update, `minReadySeconds` safety buffer
- **Graceful shutdown**: the iptables race condition, `preStop` sleep, SIGTERM handling in Node.js
- All probe anti-patterns and their consequences
- Full debugging toolkit for probe failures

**Coming up on Day 21:** Week 3 Capstone — deploy a complete full-stack app (API + PostgreSQL + Redis + Nginx) to your local Kubernetes cluster with everything from Days 11–20: Deployments, Services, Ingress, ConfigMaps, Secrets, PVCs, ResourceQuotas, LimitRanges, and production-grade health probes — all wired together.
