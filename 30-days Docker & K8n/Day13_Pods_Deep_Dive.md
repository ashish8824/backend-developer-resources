# Day 13: Pods — The Smallest Deployable Unit in Kubernetes

**Goal for today:** Understand what a Pod truly is at a deep level, why it's the fundamental building block of Kubernetes, multi-container Pod patterns (sidecar, init containers, ambassador), the complete Pod lifecycle, and why you almost never create bare Pods in production. By the end, you should be able to write any Pod spec from scratch and explain every field in it.

**Format:** WHAT → WHY → HOW for every concept.

---

## 0. Quick Recap (open your teaching session with this)

- Day 11: K8s architecture — Control Plane + Worker Nodes, reconciliation loop
- Day 12: kubectl + Minikube — cluster setup, declarative YAML, debugging toolkit

**Today's connecting thought:**
> In Days 11–12 you created Pods indirectly via Deployments without fully understanding what a Pod is. Today we go all the way down to the atomic unit. Everything in Kubernetes — Deployments, StatefulSets, Jobs, DaemonSets — ultimately creates and manages Pods. If you understand Pods deeply, you understand all of Kubernetes.

---

## 1. What is a Pod?

### WHAT
A **Pod** is the smallest deployable unit in Kubernetes. It is a wrapper around one or more containers that:
- Always run **together on the same node**
- Share the **same network namespace** (same IP address, same localhost)
- Can share **storage volumes**
- Are **scheduled, started, and stopped as a single unit**

### WHY this abstraction exists
Why not just schedule containers directly, like Docker does?

The Pod abstraction exists because some workloads are genuinely composed of multiple tightly-coupled processes that *must* run together — for example, a web server and a log shipper that reads the server's log files. These aren't one container, but they're also not independent — they need to share the filesystem and communicate over localhost. The Pod is the right level of abstraction for this.

**Teaching line:**
> "A Pod is like a logical host — it has one IP address, and all containers inside it share that IP. From a networking perspective, it's as if all those containers are processes running on the same machine, able to communicate via `localhost`. The Pod is that machine."

### HOW — The fundamental properties

```
┌─────────────────────────────────────────────────────┐
│                        POD                           │
│  IP: 172.10.1.5                                     │
│                                                     │
│  ┌─────────────────┐    ┌─────────────────┐         │
│  │   Container 1    │    │   Container 2    │         │
│  │   (main app)     │    │   (log shipper)  │         │
│  │                  │    │                  │         │
│  │  port 3000       │    │  reads /var/log  │         │
│  └────────┬─────────┘    └────────┬─────────┘        │
│           │                       │                  │
│           └──── shared volume ────┘                  │
│                (/var/log/app)                        │
│                                                     │
│  Network: containers talk via localhost              │
│  Storage: containers share mounted volumes           │
└─────────────────────────────────────────────────────┘
```

---

## 2. Pod vs Container — The Critical Distinction

This is where most Docker developers get confused moving to Kubernetes. Clarify this explicitly.

| | Docker Container | Kubernetes Pod |
|---|---|---|
| Unit of | Execution | Scheduling |
| Contains | One process | One or more containers |
| Network | Own IP (or shared with host) | One shared IP per Pod |
| Scheduled by | Docker Daemon | Kubernetes Scheduler |
| Self-healing | `restart` policy | Managed by controller (Deployment etc.) |
| Scaling | `docker compose scale` | Controller manages replicas |

**Teaching line:**
> "Docker thinks in containers. Kubernetes thinks in Pods. When you move from Docker to Kubernetes, your unit of thinking shifts from 'container' to 'Pod'. A Pod is what gets scheduled, what gets an IP, what gets started and stopped together."

---

## 3. Writing a Pod Spec — Every Field Explained

### Minimal Pod

```yaml
apiVersion: v1          # API version for core objects
kind: Pod               # object type
metadata:
  name: my-api-pod      # unique name in the namespace
  namespace: default    # which namespace
spec:
  containers:
    - name: api         # container name (unique within pod)
      image: nginx:alpine  # image to run
```

```bash
kubectl apply -f pod.yaml
kubectl get pods
kubectl describe pod my-api-pod
kubectl delete pod my-api-pod
```

### Full Production Pod Spec — Every Field

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: taskapi-pod
  namespace: default
  labels:                       # key-value tags — used by Services to find Pods
    app: taskapi
    version: "1.0"
    tier: backend
  annotations:                  # non-identifying metadata (for tools/humans)
    description: "TaskAPI main application pod"
    team: "platform-engineering"

spec:
  # ── Scheduling ──────────────────────────────────────────────────────────────
  nodeSelector:                 # only schedule on nodes with this label
    disktype: ssd               # e.g., only on nodes with SSDs

  # ── Init Containers ─────────────────────────────────────────────────────────
  initContainers:               # run to completion BEFORE main containers start
    - name: wait-for-db
      image: busybox:1.36
      command: ['sh', '-c',
        'until nc -z db-service 5432; do echo waiting for db; sleep 2; done']

  # ── Main Containers ─────────────────────────────────────────────────────────
  containers:
    - name: api                 # main application container
      image: taskapi:1.0
      imagePullPolicy: Always   # Always | IfNotPresent | Never

      # ── Ports ──────────────────────────────────────────────────────────────
      ports:
        - name: http
          containerPort: 3000
          protocol: TCP

      # ── Environment Variables ───────────────────────────────────────────────
      env:
        - name: NODE_ENV
          value: "production"
        - name: PORT
          value: "3000"
        - name: DB_PASSWORD        # from a Secret (Day 16)
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        - name: APP_NAME           # from a ConfigMap (Day 16)
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: app-name

      # ── Resource Requests and Limits ────────────────────────────────────────
      # requests: minimum guaranteed — Scheduler uses this for placement
      # limits:   maximum allowed — container is killed if it exceeds this
      resources:
        requests:
          cpu: "250m"             # 0.25 CPU cores (250 millicores)
          memory: "256Mi"         # 256 mebibytes
        limits:
          cpu: "500m"             # 0.5 CPU cores
          memory: "512Mi"         # 512 mebibytes

      # ── Volume Mounts ───────────────────────────────────────────────────────
      volumeMounts:
        - name: app-logs
          mountPath: /app/logs    # where the volume appears in the container
        - name: config-vol
          mountPath: /app/config
          readOnly: true          # container can read but not write

      # ── Health Checks ───────────────────────────────────────────────────────
      # liveness: is the container still running correctly?
      livenessProbe:
        httpGet:
          path: /health
          port: 3000
        initialDelaySeconds: 15   # wait before first check
        periodSeconds: 20         # check every 20s
        failureThreshold: 3       # restart after 3 consecutive failures

      # readiness: is the container ready to receive traffic?
      readinessProbe:
        httpGet:
          path: /health
          port: 3000
        initialDelaySeconds: 5
        periodSeconds: 10
        failureThreshold: 3       # remove from Service endpoints after 3 failures

      # startup: is the container still starting? (for slow-starting apps)
      startupProbe:
        httpGet:
          path: /health
          port: 3000
        failureThreshold: 30      # allow up to 30 * 10s = 5 minutes to start
        periodSeconds: 10

  # ── Volumes ─────────────────────────────────────────────────────────────────
  volumes:
    - name: app-logs
      emptyDir: {}                # ephemeral — lives as long as the Pod
    - name: config-vol
      configMap:                  # mount a ConfigMap as files (Day 16)
        name: app-config

  # ── Pod Policies ────────────────────────────────────────────────────────────
  restartPolicy: Always           # Always | OnFailure | Never
  terminationGracePeriodSeconds: 30  # time for graceful shutdown before SIGKILL

  # ── Image Pull Secrets ──────────────────────────────────────────────────────
  imagePullSecrets:               # credentials for private registries
    - name: ecr-registry-secret
```

---

## 4. Labels and Selectors — The Glue of Kubernetes

### WHAT
**Labels** are key-value pairs attached to any Kubernetes object. **Selectors** are queries that filter objects by their labels. This is how Services find Pods, how Deployments manage Pods, and how you filter resources in kubectl.

### WHY
Labels are the fundamental wiring of Kubernetes. Without them, there's no way for a Service to know which Pods to send traffic to, or for a Deployment to know which Pods it's managing. They decouple resources from each other — a Service doesn't reference Pods by name (names change), it references them by label (stable).

### HOW

```yaml
# Pod with labels
metadata:
  labels:
    app: taskapi          # what application
    version: "2.1"        # which version
    tier: backend         # what tier
    env: production       # what environment
```

```bash
# Filter by label
kubectl get pods -l app=taskapi
kubectl get pods -l app=taskapi,env=production
kubectl get pods -l tier!=frontend                # NOT frontend
kubectl get pods -l 'env in (staging,production)' # IN set

# Add a label to a running pod
kubectl label pod my-pod new-label=value

# Remove a label
kubectl label pod my-pod new-label-

# Show labels in output
kubectl get pods --show-labels
```

**Teaching line:**
> "Labels in Kubernetes are like tags on GitHub issues. You can add as many as you want, and then filter by any combination. The power is in the querying — 'show me all pods that are app=api AND env=production AND version=2.0'."

---

## 5. Health Probes — Deep Dive

### WHAT
Kubernetes uses three types of probes to determine the health of a container:

| Probe | Question it answers | Action if it fails |
|---|---|---|
| **Liveness** | "Is this container still alive and functioning?" | **Restart** the container |
| **Readiness** | "Is this container ready to receive traffic?" | **Remove** from Service endpoints (stop sending traffic) |
| **Startup** | "Has this container finished starting up?" | Prevent liveness/readiness from running until startup passes |

### WHY each one exists

**Liveness** solves deadlock/hang scenarios:
```
Scenario: Your app starts fine but after 2 hours enters a deadlock
— it's technically running (process alive) but not doing anything useful.
→ Without liveness probe: K8s thinks it's fine. Users see hanging requests forever.
→ With liveness probe: K8s detects failed health checks, restarts the container.
```

**Readiness** solves the "not ready yet" scenario:
```
Scenario: Your app starts but takes 20 seconds to warm up (load caches, connect to DB).
→ Without readiness probe: K8s sends traffic immediately. Users see errors for 20s.
→ With readiness probe: K8s waits until the app passes readiness, THEN routes traffic.
```

**Startup** solves slow-starting legacy apps:
```
Scenario: Your app takes 3 minutes to start (e.g., a Java app loading classes).
→ Without startup probe: Liveness probe fires before app is ready → restarts it → never starts.
→ With startup probe: Liveness/readiness are paused until startup probe passes.
```

### HOW — Three probe types

**HTTP probe (most common):**
```yaml
livenessProbe:
  httpGet:
    path: /health         # must return 2xx status code to pass
    port: 3000
    httpHeaders:          # optional custom headers
      - name: Custom-Header
        value: Awesome
  initialDelaySeconds: 10
  periodSeconds: 15
  timeoutSeconds: 5       # fail if no response within 5s
  successThreshold: 1     # 1 success = healthy
  failureThreshold: 3     # 3 consecutive failures = restart
```

**TCP probe (for non-HTTP services like databases):**
```yaml
livenessProbe:
  tcpSocket:
    port: 5432            # passes if TCP connection can be established
  initialDelaySeconds: 15
  periodSeconds: 20
```

**Exec probe (run a command — passes if exit code is 0):**
```yaml
livenessProbe:
  exec:
    command:
      - sh
      - -c
      - "pg_isready -U postgres"    # passes if exit code = 0
  initialDelaySeconds: 30
  periodSeconds: 10
```

**Teaching line:**
> "Think of probes like a hospital monitoring system. Liveness is the heartbeat monitor — if it flatlines, restart the patient. Readiness is the 'fit for duty' check — if the patient isn't ready, don't assign them tasks yet. Startup is the post-surgery 'are you awake yet?' — don't run full checks until they've woken up from anaesthesia."

---

## 6. Resource Requests and Limits — Deep Dive

### WHAT
Every container should declare:
- **Requests:** The minimum resources it needs — used by the Scheduler for placement
- **Limits:** The maximum resources it's allowed — enforced at runtime

### WHY
Without resource declarations:
- Scheduler can't make good placement decisions (a node might be overloaded)
- One runaway container can starve all other containers on the same node
- OOM (Out of Memory) kills happen unpredictably across the whole node

With proper resource declarations:
- Scheduler places Pods on nodes that can actually run them
- Each container is guaranteed its requested amount
- Noisy neighbour problem is contained

### HOW

```yaml
resources:
  requests:
    cpu: "250m"       # 250 millicores = 0.25 of a CPU core
    memory: "256Mi"   # 256 mebibytes
  limits:
    cpu: "500m"       # hard ceiling: throttled if exceeded
    memory: "512Mi"   # hard ceiling: OOMKilled if exceeded
```

**CPU units:**
- `1` = 1 full CPU core
- `500m` = 0.5 CPU core (500 millicores)
- `250m` = 0.25 CPU core
- CPU is **throttled** if limit exceeded (slowed down, not killed)

**Memory units:**
- `Mi` = Mebibytes (1024 × 1024 bytes)
- `Gi` = Gibibytes
- `M` = Megabytes (1000 × 1000 bytes)
- Memory causes **OOMKill** (container killed) if limit exceeded — not gradual

```bash
# Check actual resource usage
kubectl top pods
kubectl top nodes

# Describe a node to see how much is allocated vs available
kubectl describe node minikube | grep -A 10 "Allocated resources"
```

**Teaching line:**
> "Requests are your reservation at a restaurant — the Scheduler won't seat you unless there's a table of that size. Limits are the restaurant's maximum occupancy — you can't fit more people than the limit, and if you try, someone gets removed."

---

## 7. Multi-Container Pod Patterns

This is one of the most important architectural concepts in Kubernetes. There are three standard patterns for multi-container Pods.

---

### 7.1 Sidecar Pattern

#### WHAT
A **sidecar** container runs alongside the main container and enhances or supports it — without being part of the main application logic.

#### WHY
Sidecars let you add cross-cutting concerns (logging, monitoring, security, proxying) to any app without modifying the app itself. The main container doesn't know or care about the sidecar.

#### HOW — Common examples

**Log shipper sidecar:**
```yaml
spec:
  containers:
    # Main application
    - name: api
      image: taskapi:1.0
      volumeMounts:
        - name: log-vol
          mountPath: /app/logs      # writes logs here

    # Sidecar: ships logs to Elasticsearch / CloudWatch
    - name: log-shipper
      image: fluentd:latest
      volumeMounts:
        - name: log-vol
          mountPath: /var/log/app   # reads logs from here

  volumes:
    - name: log-vol
      emptyDir: {}                  # shared between both containers
```

**Envoy proxy sidecar (service mesh pattern):**
```yaml
spec:
  containers:
    - name: api
      image: taskapi:1.0
      ports:
        - containerPort: 3000

    - name: envoy-proxy           # intercepts all traffic in/out of the pod
      image: envoyproxy/envoy:v1.28
      ports:
        - containerPort: 9901     # admin port
```

**Teaching line:**
> "The sidecar pattern is like attaching a motorcycle sidecar to the main bike. The sidecar shares the journey (same Pod, same node, same lifecycle) but has a different job. The motorcycle doesn't need to know the sidecar exists."

---

### 7.2 Init Container Pattern

#### WHAT
**Init containers** run to full completion **before** any main containers start. Only when ALL init containers have succeeded does Kubernetes start the main containers.

#### WHY
Init containers solve the startup dependency problem:
- Wait for a database to be ready before the API starts
- Run database migrations before the API starts
- Download config files from a secret store before the app reads them
- Set permissions or create directories the main container needs

Init containers run with different images and different permissions than the main container — giving you full flexibility for setup tasks.

#### HOW

```yaml
spec:
  # ── Init Containers (run in ORDER, each must complete before next starts) ───
  initContainers:

    # Init 1: Wait for PostgreSQL to be ready
    - name: wait-for-postgres
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          echo "Waiting for postgres..."
          until nc -z postgres-service 5432; do
            echo "postgres not ready, sleeping..."
            sleep 3
          done
          echo "postgres is ready!"

    # Init 2: Run database migrations
    - name: run-migrations
      image: taskapi:1.0
      command: ["node", "scripts/migrate.js"]
      env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url

  # ── Main Container (starts only after BOTH init containers complete) ────────
  containers:
    - name: api
      image: taskapi:1.0
      ports:
        - containerPort: 3000
```

**What happens if an init container fails?**
Kubernetes restarts it (based on `restartPolicy`) until it succeeds. The main containers never start until ALL init containers have exited with code 0.

**Teaching line:**
> "Init containers are like the prep kitchen in a restaurant. Before service starts, the prep kitchen washes vegetables, makes sauces, sets up stations. The main kitchen (main containers) only opens when all prep work is done. If prep fails, the restaurant doesn't open."

---

### 7.3 Ambassador Pattern

#### WHAT
An **ambassador** container proxies network connections between the main container and the outside world. The main container always connects to `localhost`, and the ambassador container handles the real routing.

#### WHY
Useful when:
- The main app only knows how to connect to `localhost` but needs to reach a remote service
- You want to add retries, circuit breaking, or TLS termination without changing the app
- You're migrating — the ambassador routes to old or new service depending on config

#### HOW

```yaml
spec:
  containers:
    # Main app — always connects to localhost:5432
    - name: api
      image: taskapi:1.0
      env:
        - name: DB_HOST
          value: "localhost"        # always localhost
        - name: DB_PORT
          value: "5432"

    # Ambassador — receives on localhost:5432, forwards to real DB
    - name: db-ambassador
      image: haproxy:2.8
      # configured to proxy localhost:5432 → real-postgres.rds.amazonaws.com:5432
      volumeMounts:
        - name: haproxy-config
          mountPath: /usr/local/etc/haproxy

  volumes:
    - name: haproxy-config
      configMap:
        name: haproxy-config
```

---

## 8. Pod Lifecycle — States and Transitions

### WHAT
A Pod moves through a series of **phases** and **conditions** from creation to termination.

### Pod Phases

```
            ┌─────────────────────────────────────────────────────┐
            │                  POD PHASE TRANSITIONS               │
            │                                                      │
  Created ──►  Pending  ──►  Running  ──►  Succeeded             │
            │      │              │                               │
            │      │              └──────────────► Failed         │
            │      │                                              │
            │      └──────────────────────────────► Unknown       │
            └─────────────────────────────────────────────────────┘
```

| Phase | Meaning |
|---|---|
| **Pending** | Pod accepted by K8s but not yet running — waiting for scheduling or image pull |
| **Running** | Pod scheduled to a node, at least one container is running |
| **Succeeded** | All containers have exited with code 0 (for Jobs/batch workloads) |
| **Failed** | All containers have stopped, at least one exited with non-zero code |
| **Unknown** | Pod status can't be determined (usually a node communication problem) |

### Container States (within a running Pod)

| State | Meaning |
|---|---|
| **Waiting** | Not yet started — pulling image, waiting for init containers |
| **Running** | Container is executing |
| **Terminated** | Container finished (successfully or with error) |

### Pod Conditions (more granular status)

```bash
kubectl describe pod my-pod
# Look for Conditions section:
# PodScheduled    True  → Scheduler assigned it to a node
# Initialized     True  → all init containers completed
# ContainersReady True  → all containers passed readiness probe
# Ready           True  → Pod can receive traffic
```

### Termination Lifecycle (graceful shutdown)

```
1. Pod marked for deletion (kubectl delete pod OR controller scales down)
2. Pod removed from Service endpoints immediately — stops receiving new traffic
3. preStop hook runs (if defined) — optional cleanup command
4. SIGTERM sent to all containers — app should begin graceful shutdown
5. terminationGracePeriodSeconds countdown begins (default: 30s)
6. App finishes in-flight requests and exits cleanly
7. If still running after grace period → SIGKILL (force kill)
```

**Implementing graceful shutdown in Node.js:**
```javascript
// Handle SIGTERM gracefully
process.on('SIGTERM', () => {
  console.log('SIGTERM received. Closing server gracefully...');
  server.close(() => {
    console.log('Server closed. Exiting.');
    process.exit(0);
  });
  // Force exit after 25s (before K8s sends SIGKILL at 30s)
  setTimeout(() => process.exit(1), 25000);
});
```

**PreStop hook — useful for apps that don't handle SIGTERM:**
```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5"]   # delay shutdown to drain connections
```

---

## 9. Why You Almost Never Create Bare Pods in Production

### The Problem with Bare Pods

```bash
# Create a bare pod (NOT recommended in production)
kubectl apply -f pod.yaml

# Now delete the pod
kubectl delete pod taskapi-pod

# What happens?
# → The pod is GONE. Forever. Nothing recreates it.
# → If the node it ran on dies, the pod is gone. Nothing recreates it.
# → You can't do a rolling update — no controller to manage the transition.
# → You can't scale — no replica count concept.
```

### The Solution — Always Use a Controller

| Instead of | Use | Why |
|---|---|---|
| Bare Pod | **Deployment** | Self-healing, rolling updates, scaling |
| Bare Pod (stateful) | **StatefulSet** | Stable identity, ordered rollout (Day 22) |
| One Pod per Node | **DaemonSet** | Runs exactly one Pod on every node (Day 23) |
| One-off task | **Job** | Runs to completion (Day 23) |
| Scheduled task | **CronJob** | Runs on a schedule (Day 23) |

**Teaching line:**
> "A bare Pod is like a sticky note — useful for quick notes, but if it falls off the desk, it's gone. A Deployment is like a filing system — if a file goes missing, it replaces it automatically. In production, always use the filing system."

---

## 10. Complete Working Example — TaskAPI Pod with All Best Practices

```yaml
# taskapi-pod-complete.yaml
apiVersion: v1
kind: Pod
metadata:
  name: taskapi
  namespace: default
  labels:
    app: taskapi
    version: "1.0"
    tier: backend
spec:
  # Init container: wait for db before API starts
  initContainers:
    - name: wait-for-db
      image: busybox:1.36
      command: ['sh', '-c',
        'until nc -z db-service 5432; do echo waiting; sleep 2; done']

  containers:
    - name: api
      image: taskapi:1.0
      imagePullPolicy: IfNotPresent    # use local image in dev

      ports:
        - containerPort: 3000

      env:
        - name: NODE_ENV
          value: "production"
        - name: PORT
          value: "3000"

      resources:
        requests:
          cpu: "250m"
          memory: "256Mi"
        limits:
          cpu: "500m"
          memory: "512Mi"

      livenessProbe:
        httpGet:
          path: /health
          port: 3000
        initialDelaySeconds: 15
        periodSeconds: 20
        failureThreshold: 3

      readinessProbe:
        httpGet:
          path: /health
          port: 3000
        initialDelaySeconds: 5
        periodSeconds: 10
        failureThreshold: 3

      volumeMounts:
        - name: app-logs
          mountPath: /app/logs

      # Graceful shutdown
      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 5"]

    # Sidecar: log aggregation
    - name: log-shipper
      image: busybox:1.36
      command: ['sh', '-c', 'tail -f /logs/app.log']
      volumeMounts:
        - name: app-logs
          mountPath: /logs

  volumes:
    - name: app-logs
      emptyDir: {}

  restartPolicy: Always
  terminationGracePeriodSeconds: 30
```

```bash
# Deploy and observe
kubectl apply -f taskapi-pod-complete.yaml

# Watch init container, then main containers
kubectl get pod taskapi -w

# Check all containers in the pod
kubectl describe pod taskapi

# Logs from specific container
kubectl logs taskapi -c api
kubectl logs taskapi -c log-shipper

# Exec into specific container
kubectl exec -it taskapi -c api -- sh
```

---

## 11. Hands-On Practice Tasks

1. Write a bare Pod spec for your TaskAPI with just the required fields — apply it, describe it, delete it
2. Add resource requests and limits to the Pod spec and observe scheduling
3. Add a liveness and readiness probe — deliberately break the health endpoint (return 500) and watch K8s restart the container
4. Write a multi-container Pod with a main container and a sidecar that shares a volume — `kubectl logs <pod> -c <sidecar>` to verify the sidecar runs
5. Write an init container that sleeps for 10 seconds before the main container starts — `kubectl get pod -w` to watch the `Init:0/1` → `Running` transition
6. Run `kubectl describe pod <pod>` on a running pod — find and read the Events section
7. Run `kubectl get pod <pod> -o yaml` on a running pod — observe the `status` section added by Kubernetes vs the `spec` you wrote

---

## 12. Quiz — Test Yourself / Test Others

1. What two things do all containers inside a Pod always share?
2. What is the difference between a liveness probe and a readiness probe? What does each one do when it fails?
3. When would you use a startup probe?
4. What happens if an init container fails?
5. What is the sidecar pattern? Give a real-world example.
6. What is the difference between resource requests and resource limits?
7. If a Pod's memory limit is 512Mi and the app uses 600Mi, what happens?
8. Why should you almost never create bare Pods in production?
9. What signals does Kubernetes send to a container when deleting a Pod, and in what order?
10. A Pod is in `CrashLoopBackOff` status. Walk through your exact debugging steps.

<details>
<summary>Answers</summary>

1. All containers in a Pod share the same network namespace (one IP address, communicate via localhost) and can share storage volumes (if explicitly mounted).

2. Liveness probe answers "is this container still functioning?" — if it fails repeatedly, Kubernetes restarts the container. Readiness probe answers "is this container ready to receive traffic?" — if it fails, the Pod is removed from the Service's endpoint list (traffic stops being sent to it) but the container is NOT restarted.

3. Startup probe is used for slow-starting applications (e.g., legacy Java apps) that take a long time to initialise. Without it, the liveness probe would fire before the app is ready and restart it in a loop. Startup probe disables liveness and readiness probes until the startup check passes.

4. Kubernetes restarts the init container until it succeeds (based on the Pod's restartPolicy). The main containers never start until ALL init containers have exited with code 0. If an init container keeps failing, the Pod stays in `Init:CrashLoopBackOff`.

5. A sidecar runs alongside the main container, sharing the Pod's lifecycle, and adds supporting functionality without changing the main app. Real example: a log-shipper container reads log files written by the main API container from a shared volume and ships them to Elasticsearch/CloudWatch.

6. Requests are the minimum resources the container is guaranteed — the Scheduler only places the Pod on a node with at least this much available. Limits are the hard maximum — the container is throttled (CPU) or OOMKilled (memory) if it tries to use more than the limit.

7. The container is OOMKilled (Out Of Memory killed) — terminated with reason OOMKilled. If the restartPolicy is Always, Kubernetes restarts it. If it keeps exceeding the limit, it enters CrashLoopBackOff.

8. Bare Pods have no controller watching over them. If a bare Pod is deleted or its node dies, nothing recreates it. You also can't do rolling updates, scaling, or self-healing. Production workloads should always use a Deployment (or StatefulSet/DaemonSet/Job for specific patterns) so a controller manages the Pod lifecycle.

9. First: the Pod is removed from Service endpoints (no new traffic). Second: the preStop lifecycle hook runs (if defined). Third: SIGTERM is sent — the app should start graceful shutdown. Fourth: after terminationGracePeriodSeconds (default 30s), if still running, SIGKILL is sent (forced kill).

10. (1) `kubectl logs <pod-name> --previous` — check logs of the crashed container instance for the error. (2) `kubectl describe pod <pod-name>` — check Events section for scheduling, image pull, or OOMKill errors. (3) Check exit code in `describe` output — code 1 = app error, code 137 = OOMKilled, code 143 = SIGTERM. (4) `kubectl exec -it <pod-name> -- sh` if the container stays up long enough. (5) Check resource limits — maybe the app needs more memory.

</details>

---

## 13. Summary

After today you can explain and implement:
- What a Pod truly is — a shared network + storage boundary for tightly coupled containers
- Every field of a full Pod spec: metadata, labels, initContainers, containers, resources, probes, volumes, lifecycle, restartPolicy
- Labels and selectors — the wiring that connects Pods to Services and controllers
- All three health probes: liveness (restart), readiness (traffic removal), startup (slow apps) — with the hospital monitoring analogy
- Resource requests (Scheduler placement) vs limits (runtime enforcement)
- All three multi-container patterns: sidecar (support), init container (setup), ambassador (proxy)
- The full Pod lifecycle: Pending → Running → Succeeded/Failed, graceful shutdown with SIGTERM
- Why bare Pods are wrong for production — always use a controller

**Coming up on Day 14:** ReplicaSets and Deployments — the controllers that manage Pods, implement rolling updates, enable scaling, and provide auto-healing. Everything you've learned about Pods today is the foundation for understanding how Deployments manage them.
