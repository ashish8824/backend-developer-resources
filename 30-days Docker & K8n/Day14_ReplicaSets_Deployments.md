# Day 14: ReplicaSets & Deployments — Scaling, Rolling Updates & Auto-Healing

**Goal for today:** Understand what ReplicaSets and Deployments are, how they layer on top of Pods, how rolling updates and rollbacks work under the hood, how scaling operates, and why Deployments are the standard way to run any stateless workload in Kubernetes. By the end, you should be able to write a production Deployment from scratch, perform a rolling update, roll it back, and explain every step of what Kubernetes is doing internally.

**Format:** WHAT → WHY → HOW for every concept.

---

## 0. Quick Recap (open your teaching session with this)

- Day 13: Pods — shared network + storage unit, health probes, resource limits, multi-container patterns, lifecycle
- Key takeaway: Never create bare Pods in production — always use a controller

**Today's connecting thought:**
> Yesterday you learned that bare Pods are like sticky notes — if they fall off the desk, they're gone forever. Today we build the filing system that manages those sticky notes: first **ReplicaSets** (which ensure the right number of Pods exist), then **Deployments** (which manage ReplicaSets and add rolling updates, rollbacks, and revision history). These two controllers are what power every stateless production workload in Kubernetes.

---

## 1. ReplicaSet — Ensuring the Right Number of Pods

### WHAT
A **ReplicaSet** is a controller that ensures a specified number of identical Pod replicas are running at all times. If a Pod dies, the ReplicaSet creates a new one. If there are too many, it deletes extras.

### WHY
This directly solves the Day 10 problems:
- **Self-healing** → Pod crashes → ReplicaSet detects actual < desired → creates replacement
- **Scaling** → you want 5 instances → ReplicaSet maintains exactly 5 across the cluster
- **Availability** → node dies taking a Pod → ReplicaSet reschedules on a healthy node

### HOW — ReplicaSet YAML

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: taskapi-rs
  labels:
    app: taskapi
spec:
  replicas: 3                    # desired number of Pods

  selector:                      # how this RS finds its Pods
    matchLabels:
      app: taskapi               # manages any Pod with label app=taskapi

  template:                      # Pod blueprint (same as a bare Pod spec)
    metadata:
      labels:
        app: taskapi             # MUST match selector above
    spec:
      containers:
        - name: api
          image: taskapi:1.0
          ports:
            - containerPort: 3000
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
```

```bash
kubectl apply -f replicaset.yaml
kubectl get replicasets
kubectl get pods -l app=taskapi   # shows 3 pods

# Watch self-healing in action
kubectl delete pod <any-pod-name>   # delete one
kubectl get pods -w                 # watch it come back

# Scale manually
kubectl scale replicaset taskapi-rs --replicas=5
kubectl get pods

# Describe to see events
kubectl describe replicaset taskapi-rs
```

### ReplicaSet Selector — Critical Concept

The `selector.matchLabels` is how the ReplicaSet knows which Pods it "owns". This is important:

```
ReplicaSet          Pod
selector:           labels:
  app: taskapi  ──► app: taskapi    ✅ owned by this RS
  
                    labels:
                      app: other    ✅ NOT owned — RS ignores it
```

**Gotcha to teach:** If you manually create a Pod with labels matching a ReplicaSet's selector, the RS will **adopt** it and count it towards the desired replicas — potentially deleting other Pods to bring the count back to `replicas`.

---

## 2. Why You Rarely Use ReplicaSets Directly

### The Problem ReplicaSets Can't Solve
ReplicaSets are great at maintaining replica count — but they have **no concept of versions or updates**.

```bash
# Scenario: you want to update from taskapi:1.0 to taskapi:2.0

# With a bare ReplicaSet, you have two terrible options:
# Option A: Update the image in the RS spec → RS replaces ALL pods simultaneously
#           → ALL pods go down at once → full downtime

# Option B: Delete the RS and create a new one
#           → Again, full downtime

# Neither option gives you a rolling update or rollback.
```

This is exactly why **Deployments** exist — they wrap ReplicaSets and add the upgrade orchestration layer.

**Teaching line:**
> "A ReplicaSet is a great worker that keeps your pods running, but it has no memory — it doesn't know what came before. A Deployment is a manager with a notebook — it remembers every version, orchestrates smooth transitions between them, and can undo the last change if something goes wrong."

---

## 3. Deployment — The Standard Way to Run Stateless Workloads

### WHAT
A **Deployment** is a higher-level controller that manages ReplicaSets. It:
- Creates and manages ReplicaSets (you never create ReplicaSets directly)
- Adds **rolling update** capability — transition from one version to another smoothly
- Keeps a **revision history** — enables rollback to any previous version
- Provides **declarative updates** — you specify desired state, it handles the transition

### WHY
Deployments solve every problem from Day 10 that ReplicaSets couldn't:
- ✅ Self-healing (inherited from RS)
- ✅ Scaling (inherited from RS)
- ✅ Rolling updates — zero downtime deploys
- ✅ Rollback — one command to undo a bad deploy
- ✅ Revision history — full audit of what deployed when

### HOW — The Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: taskapi
  namespace: default
  labels:
    app: taskapi

spec:
  # ── Replica Count ────────────────────────────────────────────────────────────
  replicas: 3

  # ── Selector ─────────────────────────────────────────────────────────────────
  selector:
    matchLabels:
      app: taskapi           # Deployment manages Pods with this label

  # ── Update Strategy ──────────────────────────────────────────────────────────
  strategy:
    type: RollingUpdate      # RollingUpdate (default) | Recreate
    rollingUpdate:
      maxUnavailable: 1      # at most 1 Pod unavailable during update
      maxSurge: 1            # at most 1 extra Pod above desired during update

  # ── Revision History ──────────────────────────────────────────────────────────
  revisionHistoryLimit: 5    # keep last 5 ReplicaSets for rollback (default: 10)

  # ── Min Ready Seconds ─────────────────────────────────────────────────────────
  minReadySeconds: 10        # wait 10s after Pod is ready before considering update successful

  # ── Pod Template (same as a bare Pod spec) ────────────────────────────────────
  template:
    metadata:
      labels:
        app: taskapi         # MUST match selector.matchLabels
    spec:
      containers:
        - name: api
          image: taskapi:1.0
          ports:
            - containerPort: 3000

          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"

          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
            failureThreshold: 3

          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 20
            failureThreshold: 3

          env:
            - name: NODE_ENV
              value: "production"
            - name: PORT
              value: "3000"

      terminationGracePeriodSeconds: 30
```

---

## 4. The Deployment → ReplicaSet → Pod Hierarchy

This is the most important relationship to visualise and teach clearly:

```
Deployment (taskapi)
│  manages revision history
│  orchestrates rolling updates
│
├── ReplicaSet v1 (taskapi-6d8b7c9f4b)   ← old RS (image: taskapi:1.0)
│   replicas: 0 (scaled down after update)
│
└── ReplicaSet v2 (taskapi-7f9c8d5e2a)   ← current RS (image: taskapi:2.0)
    replicas: 3
    │
    ├── Pod: taskapi-7f9c8d5e2a-xk2p9
    ├── Pod: taskapi-7f9c8d5e2a-m8rt4
    └── Pod: taskapi-7f9c8d5e2a-q3wn7
```

```bash
# See all three layers at once
kubectl get deployment taskapi
kubectl get replicasets -l app=taskapi   # notice the hash in the RS name
kubectl get pods -l app=taskapi          # notice the RS hash + pod hash in names

# Pod name format: <deployment-name>-<rs-hash>-<pod-hash>
# taskapi-7f9c8d5e2a-xk2p9
#          ^^^^^^^^^^     ← RS hash (changes on update)
#                    ^^^^^ ← Pod hash (unique per pod)
```

---

## 5. Update Strategies — RollingUpdate vs Recreate

### Strategy 1: RollingUpdate (default — use this)

```
Before update: 3 pods running taskapi:1.0
After update:  3 pods running taskapi:2.0

Rolling update process (maxUnavailable=1, maxSurge=1):

Step 1:  [v1] [v1] [v1]          → start: 3 old pods
Step 2:  [v1] [v1] [v1] [v2]     → create 1 new (surge) — now 4 pods
Step 3:  [v1] [v1] [v2]          → old pod passes readiness → delete 1 old
Step 4:  [v1] [v1] [v2] [v2]     → create another new — 4 pods again
Step 5:  [v1] [v2] [v2]          → second new passes readiness → delete old
Step 6:  [v1] [v2] [v2] [v2]     → create last new
Step 7:  [v2] [v2] [v2]          → last old deleted — update complete!

At NO point were all pods unavailable → zero downtime
```

**maxUnavailable:** how many Pods can be unavailable (below desired) during update
**maxSurge:** how many extra Pods can exist (above desired) during update

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0    # never go below desired capacity (safest)
    maxSurge: 1          # allow 1 extra pod during transition

# OR as percentages:
    maxUnavailable: 25%  # 25% of replicas can be unavailable
    maxSurge: 25%        # can temporarily exceed desired by 25%
```

### Strategy 2: Recreate (for when you CANNOT run two versions simultaneously)

```yaml
strategy:
  type: Recreate         # kills ALL old pods, THEN starts new ones
```

```
Step 1: [v1] [v1] [v1]   → all 3 old pods terminated
Step 2: (nothing running for a few seconds — DOWNTIME)
Step 3: [v2] [v2] [v2]   → all 3 new pods started
```

**When to use Recreate:**
- Database schema migrations where v1 and v2 app can't coexist with same DB schema
- Apps that hold exclusive locks on resources
- Any situation where running two versions simultaneously would cause data corruption

**Teaching line:**
> "RollingUpdate is like renovating a hotel room by room while guests keep checking in — some rooms are being renovated, but the hotel stays open. Recreate is like closing the entire hotel for renovation — everyone leaves, you renovate, then guests come back. Use Recreate only when you absolutely must."

---

## 6. Performing a Rolling Update — The Full Workflow

### Method 1: Update the YAML and apply (recommended)

```yaml
# Change image from taskapi:1.0 to taskapi:2.0 in deployment.yaml
containers:
  - name: api
    image: taskapi:2.0    # ← changed
```

```bash
kubectl apply -f deployment.yaml

# Watch the rolling update
kubectl rollout status deployment/taskapi
# Output: Waiting for deployment "taskapi" rollout to finish: 1 out of 3 new replicas have been updated...
# Output: Waiting for deployment "taskapi" rollout to finish: 2 out of 3 new replicas have been updated...
# Output: deployment "taskapi" successfully rolled out

# Watch pods being replaced in real time
kubectl get pods -w
```

### Method 2: `kubectl set image` (quick, imperative)

```bash
kubectl set image deployment/taskapi api=taskapi:2.0
kubectl rollout status deployment/taskapi
```

### Method 3: `kubectl edit` (live edit — careful in prod)

```bash
kubectl edit deployment/taskapi    # opens in $EDITOR
# Change the image, save → triggers rolling update immediately
```

---

## 7. Rollback — Undoing a Bad Deployment

One of the most powerful features of Deployments — and one of the most important for production operations.

### How rollback works internally

When you deploy a new version, Kubernetes:
1. Creates a **new ReplicaSet** for the new version
2. Scales it up while scaling the old RS down
3. Keeps the old RS around (with 0 replicas) for rollback

Rollback = reverse the process: scale old RS back up, scale new RS down.

```bash
# Check rollout history
kubectl rollout history deployment/taskapi

# Output:
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>
# 3         <none>

# Add change cause (annotation) for better history — best practice
kubectl apply -f deployment.yaml
kubectl annotate deployment/taskapi kubernetes.io/change-cause="Deploy taskapi:2.0 - new auth feature"

# Now history shows causes:
# REVISION  CHANGE-CAUSE
# 1         Initial deployment - taskapi:1.0
# 2         Deploy taskapi:2.0 - new auth feature

# See what's in a specific revision
kubectl rollout history deployment/taskapi --revision=1

# Roll back to previous version
kubectl rollout undo deployment/taskapi

# Roll back to specific revision
kubectl rollout undo deployment/taskapi --to-revision=1

# Watch the rollback
kubectl rollout status deployment/taskapi

# Verify which image is now running
kubectl get deployment taskapi -o jsonpath='{.spec.template.spec.containers[0].image}'
```

---

## 8. Scaling Deployments

### Manual Scaling

```bash
# Scale up
kubectl scale deployment taskapi --replicas=10

# Scale down
kubectl scale deployment taskapi --replicas=2

# Scale to 0 — effectively pause the app (no downtime for other services)
kubectl scale deployment taskapi --replicas=0

# Or update replicas in YAML and apply
```

### Why readinessProbe is ESSENTIAL for rolling updates

```
Without readinessProbe:
  New Pod starts → image pulled → container starts → K8s marks it Ready immediately
  → traffic routed to it → but app takes 10s to connect to DB → users get errors

With readinessProbe:
  New Pod starts → image pulled → container starts → readiness probe starts checking
  → probe fails (app not ready yet) → Pod stays out of Service endpoints
  → probe passes (app connected to DB) → Pod added to Service endpoints
  → traffic routed → users see no errors

The rolling update ALSO waits for readiness before proceeding.
Without readiness: K8s replaces all pods immediately, possibly sending traffic to broken new pods.
With readiness: each new pod must prove it's ready before the next old pod is removed.
```

---

## 9. Deployment Status — Reading the Full Picture

```bash
kubectl describe deployment taskapi
```

Key fields to understand:

```
Name:               taskapi
Namespace:          default
Replicas:           3 desired | 3 updated | 3 total | 3 available | 0 unavailable
                    ^desired    ^updated    ^total    ^available    ^unavailable

StrategyType:       RollingUpdate
RollingUpdateStrategy:  25% max unavailable, 25% max surge

Conditions:
  Available       True    MinimumReplicasAvailable  ← at least 1 replica available
  Progressing     True    NewReplicaSetAvailable    ← rollout succeeded / in progress

OldReplicaSets:   taskapi-6d8b7c9f4b (0/0 replicas)   ← previous RS, 0 replicas
NewReplicaSet:    taskapi-7f9c8d5e2a (3/3 replicas)   ← current RS, all 3 running

Events:
  Scaled up replica set taskapi-7f9c8d5e2a to 1
  Scaled down replica set taskapi-6d8b7c9f4b to 2
  Scaled up replica set taskapi-7f9c8d5e2a to 2
  ...
```

---

## 10. Pausing and Resuming Deployments

### WHAT
You can **pause** a Deployment mid-rollout to inspect the state — then **resume** or **undo** it.

### WHY
Useful for canary-style testing: deploy to 1 pod, pause, check metrics, then resume or rollback.

```bash
# Pause rollout after triggering it
kubectl rollout pause deployment/taskapi

# While paused — check the intermediate state
kubectl get pods
kubectl get replicasets

# Resume if looking good
kubectl rollout resume deployment/taskapi

# Or undo if looking bad
kubectl rollout undo deployment/taskapi
```

---

## 11. Deployment Complete Reference YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: taskapi
  namespace: production
  labels:
    app: taskapi
    tier: backend
  annotations:
    kubernetes.io/change-cause: "Deploy taskapi:2.0 - new auth endpoint"

spec:
  replicas: 3
  revisionHistoryLimit: 5       # keep 5 old RS for rollback

  selector:
    matchLabels:
      app: taskapi

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0         # never reduce below 3 running pods
      maxSurge: 1               # allow up to 4 pods during update

  minReadySeconds: 10           # pod must be ready for 10s before counted as available

  template:
    metadata:
      labels:
        app: taskapi
      annotations:
        prometheus.io/scrape: "true"    # annotations for monitoring tools
        prometheus.io/port: "3000"

    spec:
      # Spread pods across nodes for resilience
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: taskapi

      containers:
        - name: api
          image: taskapi:2.0
          imagePullPolicy: Always

          ports:
            - name: http
              containerPort: 3000

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

          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
            failureThreshold: 3

          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 20
            failureThreshold: 3

          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]

      terminationGracePeriodSeconds: 30
      imagePullSecrets:
        - name: ecr-secret
```

---

## 12. Common Deployment Patterns and Anti-Patterns

### Patterns to follow

```bash
# ✅ Always annotate change-cause for history clarity
kubectl annotate deployment/taskapi \
  kubernetes.io/change-cause="v2.1.0: fix auth bug CVE-2024-1234"

# ✅ Pin image to specific SHA for true immutability
image: 123456789.dkr.ecr.ap-south-1.amazonaws.com/taskapi@sha256:abc123...

# ✅ Set maxUnavailable: 0 for zero-downtime deploys
# ✅ Always have readinessProbe — rollout waits for it
# ✅ Set reasonable revisionHistoryLimit (5-10)
# ✅ Use topologySpreadConstraints for resilience
```

### Anti-patterns to avoid

```yaml
# ❌ No readinessProbe → traffic sent to unready pods during rollout
# ❌ image: taskapi:latest → non-deterministic, can't roll back reliably
# ❌ maxUnavailable: 100% → all pods killed at once → downtime
# ❌ No resource limits → one pod can OOM-kill the node
# ❌ revisionHistoryLimit: 0 → no rollback possible
# ❌ replicas: 1 → single point of failure, no rolling update possible
```

---

## 13. Full Workflow — Deploy, Update, Roll Back

```bash
# ── 1. Initial Deployment ─────────────────────────────────────────────────────
kubectl apply -f deployment.yaml
kubectl rollout status deployment/taskapi
kubectl get pods -l app=taskapi

# ── 2. Verify app is working ──────────────────────────────────────────────────
kubectl port-forward deployment/taskapi 8080:3000 &
curl http://localhost:8080/health

# ── 3. Update the image ───────────────────────────────────────────────────────
# Edit deployment.yaml: change image to taskapi:2.0
kubectl apply -f deployment.yaml
kubectl annotate deployment/taskapi \
  kubernetes.io/change-cause="Deploy v2.0 - new features"

# Watch the rolling update
kubectl rollout status deployment/taskapi
kubectl get pods -w    # watch pods being replaced

# ── 4. Check history ──────────────────────────────────────────────────────────
kubectl rollout history deployment/taskapi

# ── 5. Simulate a bad deploy ──────────────────────────────────────────────────
kubectl set image deployment/taskapi api=taskapi:broken-version
kubectl rollout status deployment/taskapi
# → hangs because new pods fail readiness probe

# ── 6. Roll back ──────────────────────────────────────────────────────────────
kubectl rollout undo deployment/taskapi
kubectl rollout status deployment/taskapi
# → back to working version

# ── 7. Scale up ───────────────────────────────────────────────────────────────
kubectl scale deployment taskapi --replicas=5
kubectl get pods -w

# ── 8. Scale down ─────────────────────────────────────────────────────────────
kubectl scale deployment taskapi --replicas=3

# ── 9. Clean up ───────────────────────────────────────────────────────────────
kubectl delete deployment taskapi
```

---

## 14. Deployment vs ReplicaSet vs Pod — The Summary Table

| Feature | Bare Pod | ReplicaSet | Deployment |
|---|---|---|---|
| Self-healing | ❌ | ✅ | ✅ |
| Scaling | ❌ | ✅ | ✅ |
| Rolling update | ❌ | ❌ | ✅ |
| Rollback | ❌ | ❌ | ✅ |
| Revision history | ❌ | ❌ | ✅ |
| Use in production | ❌ Never | Rarely direct | ✅ Always |

---

## 15. Hands-On Practice Tasks

1. Write a Deployment YAML from scratch for your TaskAPI (no copy-paste — muscle memory matters)
2. Deploy it: `kubectl apply -f deployment.yaml`
3. Watch pods come up: `kubectl get pods -w`
4. Delete a pod manually — confirm it's recreated
5. Update the image tag in the YAML, apply, and watch the rolling update with `kubectl get pods -w`
6. Run `kubectl rollout history deployment/taskapi` — add a change-cause annotation
7. Simulate a bad deploy: set image to `taskapi:does-not-exist` → watch it fail → rollback
8. Scale to 5 replicas, then back to 3
9. Run `kubectl describe deployment taskapi` and read every section
10. Run `kubectl get replicasets` and observe the old RS (0 replicas) kept for rollback

---

## 16. Quiz — Test Yourself / Test Others

1. What is the relationship between a Deployment, a ReplicaSet, and a Pod?
2. What does a ReplicaSet do? What can't it do that a Deployment can?
3. What is a rolling update? Walk through the steps using `maxUnavailable=1, maxSurge=1` with 3 replicas.
4. What is the difference between `RollingUpdate` and `Recreate` strategies? When would you use each?
5. Why is a `readinessProbe` essential for a rolling update to work correctly?
6. What happens internally when you run `kubectl rollout undo deployment/taskapi`?
7. What does `revisionHistoryLimit: 5` do?
8. What does `maxUnavailable: 0` mean, and why would you set it?
9. If a rolling update is stuck (new pods keep failing readiness), what do you do?
10. Name three anti-patterns to avoid when writing a Deployment.

<details>
<summary>Answers</summary>

1. A Deployment manages one or more ReplicaSets. Each ReplicaSet manages a group of identical Pods. When you update a Deployment, it creates a new ReplicaSet (for the new version) and gradually scales it up while scaling the old one down. The Deployment maintains revision history by keeping old ReplicaSets (at 0 replicas) for rollback.

2. A ReplicaSet ensures the desired number of Pods are always running (self-healing + scaling). It cannot perform rolling updates, maintain revision history, or enable rollbacks — it has no concept of versions. A Deployment wraps ReplicaSets to add all of these capabilities.

3. Start: 3 v1 pods running. Step 1: Create 1 new v2 pod (surge to 4 total). Step 2: New pod passes readiness probe. Step 3: Delete 1 old v1 pod (back to 3). Step 4: Create another v2 pod (surge to 4). Step 5: Passes readiness, delete another v1. Step 6: Create last v2 pod (4 total). Step 7: Passes readiness, delete last v1 (3 v2 pods). At no point were all pods unavailable.

4. RollingUpdate gradually replaces old pods with new ones while keeping the app available — zero downtime. Recreate kills all old pods then starts new ones — causes downtime but ensures no two versions run simultaneously. Use Recreate when v1 and v2 cannot coexist (e.g., incompatible database schema changes).

5. Without a readinessProbe, Kubernetes marks a new Pod ready immediately when the container starts — before the app may actually be ready. Rolling updates proceed based on readiness: each new pod must pass the readiness probe before the next old pod is removed. Without it, traffic is sent to half-started pods and old pods are removed prematurely.

6. Kubernetes identifies the previous ReplicaSet (the one before the current RS), scales it back up to the desired replicas count, and scales the current (broken) RS back down to 0 — using the same rolling update mechanism in reverse. The revision numbering adjusts accordingly.

7. `revisionHistoryLimit: 5` tells Kubernetes to keep only the 5 most recent old ReplicaSets for rollback purposes. Older ones are deleted. Default is 10. Setting it too high wastes etcd storage; setting it to 0 means no rollback is possible.

8. `maxUnavailable: 0` means zero Pods can be below the desired count during an update — the rolling update must always maintain full capacity. Combined with `maxSurge: 1`, it means you always run at full capacity plus one extra pod during the transition. Use when you cannot afford any reduction in serving capacity.

9. Run `kubectl rollout undo deployment/taskapi` to immediately roll back to the previous working version. Then investigate why the new pods failed: `kubectl describe pod <new-pod-name>` and `kubectl logs <new-pod-name>`.

10. Any three of: using `latest` tag (can't roll back reliably), `maxUnavailable: 100%` (full downtime), no readinessProbe (traffic to unready pods), no resource limits (node instability), `revisionHistoryLimit: 0` (no rollback possible), `replicas: 1` (SPOF, no real rolling update), no change-cause annotations (unauditable history).

</details>

---

## 17. Summary

After today you can explain and implement:
- **ReplicaSet** — ensures desired Pod count, self-healing, scaling, selector-based Pod ownership
- **Deployment → ReplicaSet → Pod** hierarchy — the three layers and how they relate
- Why ReplicaSets alone are insufficient — no rolling update, no version management
- **RollingUpdate** strategy — step-by-step mechanics with maxUnavailable and maxSurge
- **Recreate** strategy — when to use it and why it causes downtime
- **Rolling update workflow** — update image, watch rollout, verify, annotate history
- **Rollback** — how it works internally (scaling old RS up, new RS down)
- Why `readinessProbe` is essential for zero-downtime rolling updates
- Deployment anti-patterns and production best practices
- Full `kubectl rollout` command set

**Coming up on Day 15:** Services — the stable network endpoints that load-balance traffic across Pods. Pods come and go with changing IPs, but Services give you a permanent address. You'll learn ClusterIP, NodePort, LoadBalancer, and how kube-proxy implements them — completing the networking picture for your applications.
