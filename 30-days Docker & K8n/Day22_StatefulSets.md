# Day 22: StatefulSets — Running Stateful Workloads in Kubernetes

**Goal for today:** Understand what StatefulSets are, why Deployments are wrong for databases and stateful apps, what "stable identity" means and why it matters, how ordered deployment and scaling works, how StatefulSets integrate with PVCs using volumeClaimTemplates, and real-world patterns for running PostgreSQL, Redis Sentinel, and Kafka in Kubernetes. By the end, you should be able to design and deploy any stateful workload in Kubernetes and explain every line of the YAML.

**Format:** WHAT → WHY → HOW for every concept.

---

## 0. Quick Recap (open your teaching session with this)

- Day 13: Pods — ephemeral, no stable identity
- Day 14: Deployments — stateless workloads, interchangeable pods
- Day 17: Storage — PVCs for persistence
- Day 21: Capstone — ran PostgreSQL as a Deployment (single replica, Recreate strategy — a workaround)

**Today's connecting thought:**
> In Day 21 you ran PostgreSQL as a Deployment with `strategy: Recreate` and a single replica. That works for a simple setup, but it's a workaround. The real problem: Deployments treat all Pods as identical and interchangeable — perfect for stateless APIs, wrong for databases. A PostgreSQL primary and its replicas are NOT interchangeable. Each has a unique role and identity. StatefulSets are the Kubernetes-native way to model exactly this.

---

## 1. The Problem — Why Deployments Break for Stateful Apps

### Problem 1: Pods are anonymous in Deployments

```
Deployment pods:
  taskapi-7f9c8d-abc1   ← random suffix, could be anything
  taskapi-7f9c8d-def2   ← interchangeable — you don't care which one serves a request
  taskapi-7f9c8d-ghi3   ← all identical, all stateless

Database pods — what you NEED:
  postgres-0            ← PRIMARY: accepts reads AND writes
  postgres-1            ← REPLICA: reads only, replicates from postgres-0
  postgres-2            ← REPLICA: reads only, replicates from postgres-0

postgres-0 is NOT interchangeable with postgres-1.
If Kubernetes kills postgres-0 and brings it back as postgres-2,
the entire replication setup breaks.
```

### Problem 2: Deployments share one PVC — or each pod gets a random PVC

```
Deployment with PVC:
  Option A: All pods share one PVC (impossible for databases — only RWO available per node)
  Option B: Each pod gets its own PVC — but if pod-A dies and is replaced by pod-B,
            pod-B gets a BRAND NEW empty PVC — all data from pod-A is lost

What you NEED for a database:
  postgres-0 always gets the SAME PVC (postgres-data-0)
  postgres-1 always gets the SAME PVC (postgres-data-1)
  If postgres-0 crashes and restarts, it MUST get postgres-data-0 back
  — its own data, not a new empty volume
```

### Problem 3: No guaranteed startup/shutdown ordering

```
Database cluster startup requires ORDER:
  1. postgres-0 starts first (becomes primary, initialises data)
  2. postgres-1 starts second (connects to postgres-0 as replica)
  3. postgres-2 starts third (connects to postgres-0 as replica)

Deployment starts all pods simultaneously — no order guarantee.
Replica tries to connect to primary that doesn't exist yet → crash.
```

**Teaching line:**
> "A Deployment says 'I need 3 workers — any 3 will do, they're all the same.' A StatefulSet says 'I need worker-0, worker-1, and worker-2 — in that exact order, each with their own desk, and worker-0 must always sit at desk-0.' Databases need the StatefulSet model."

---

## 2. What is a StatefulSet?

### WHAT
A **StatefulSet** is a Kubernetes controller that manages a set of Pods while guaranteeing:

1. **Stable, unique Pod names** — `<statefulset-name>-0`, `<statefulset-name>-1`, `<statefulset-name>-2` — never random suffixes
2. **Stable network identities** — each Pod gets a stable DNS hostname that never changes
3. **Stable persistent storage** — each Pod gets its own PVC that follows it across restarts
4. **Ordered deployment** — Pods start in order (0, 1, 2) and each must be Running+Ready before the next starts
5. **Ordered scaling and updates** — scale down in reverse order (2, 1, 0), rolling updates go in order

### WHY
Each guarantee exists to solve a specific database problem:
- Stable names → replica can always find primary by name, not IP
- Stable DNS → connection strings don't change when pods restart
- Stable storage → data doesn't disappear when pod is rescheduled
- Ordered startup → primary initialises before replicas try to connect

---

## 3. StatefulSet vs Deployment — The Comparison

| Feature | Deployment | StatefulSet |
|---|---|---|
| Pod names | Random suffix (`api-7f9c-abc1`) | Ordered index (`postgres-0`) |
| Pod identity | Anonymous, interchangeable | Unique, stable |
| DNS hostname | Random (`ip-10-0-0-1.local`) | Stable (`postgres-0.postgres-headless`) |
| PVC per pod | Shared or random | Dedicated, follows pod (`postgres-data-0`) |
| Startup order | Parallel (all at once) | Sequential (0 before 1 before 2) |
| Shutdown order | Parallel | Reverse sequential (2 before 1 before 0) |
| Rolling update | All at once within maxUnavailable | One at a time, in reverse order |
| Use case | Stateless apps | Databases, distributed systems |

---

## 4. StatefulSet YAML — Every Field Explained

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production

spec:
  # ── How many replicas ─────────────────────────────────────────────────────
  replicas: 3

  # ── Which pods this StatefulSet manages ──────────────────────────────────
  selector:
    matchLabels:
      app: postgres

  # ── The Headless Service that provides DNS for each pod ───────────────────
  serviceName: postgres-headless    # REQUIRED — must match a Headless Service name
                                    # enables stable DNS for each pod

  # ── Update strategy ───────────────────────────────────────────────────────
  updateStrategy:
    type: RollingUpdate             # RollingUpdate | OnDelete
    rollingUpdate:
      partition: 0                  # only update pods with index >= partition
                                    # partition: 2 = only update pod-2, leave 0 and 1
                                    # used for canary updates

  # ── Pod management policy ─────────────────────────────────────────────────
  podManagementPolicy: OrderedReady  # OrderedReady (default) | Parallel
                                     # OrderedReady: 0 must be Running+Ready before 1 starts
                                     # Parallel: all start simultaneously (loses ordering)

  # ── Retain PVCs when pods are deleted ─────────────────────────────────────
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain             # Retain | Delete
    whenScaled: Retain              # what to do with PVCs when scaling down

  # ── Pod template (same as Deployment) ─────────────────────────────────────
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15-alpine
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: postgres-data     # name matches volumeClaimTemplates below
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "2"
              memory: "2Gi"

  # ── PVC Template — creates a dedicated PVC for EACH pod ────────────────────
  volumeClaimTemplates:
    - metadata:
        name: postgres-data           # PVC name = postgres-data-<pod-index>
                                      # postgres-data-0, postgres-data-1, postgres-data-2
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: standard
        resources:
          requests:
            storage: 10Gi
```

---

## 5. The Headless Service — Stable DNS per Pod

### WHAT
StatefulSets require a **Headless Service** (`clusterIP: None`) to provide stable DNS hostnames for each Pod.

### WHY
With a Headless Service, each Pod gets its own DNS entry:
```
postgres-0.postgres-headless.production.svc.cluster.local → 172.17.0.4 (pod-0 IP)
postgres-1.postgres-headless.production.svc.cluster.local → 172.17.0.5 (pod-1 IP)
postgres-2.postgres-headless.production.svc.cluster.local → 172.17.0.6 (pod-2 IP)
```

These hostnames are **stable** — if postgres-0 crashes and restarts with IP 172.17.0.9, the DNS entry updates automatically. You always connect to postgres-0 by name, never by IP.

### HOW

```yaml
# Headless Service — required for StatefulSet DNS
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: production
  labels:
    app: postgres
spec:
  clusterIP: None          # ← headless: no virtual IP, returns pod IPs directly
  selector:
    app: postgres
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432

---
# Regular ClusterIP Service — for normal traffic to any replica
apiVersion: v1
kind: Service
metadata:
  name: postgres-svc
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

**DNS names for StatefulSet pods:**
```
Format:  <pod-name>.<headless-service-name>.<namespace>.svc.cluster.local

postgres-0.postgres-headless.production.svc.cluster.local  → always pod-0
postgres-1.postgres-headless.production.svc.cluster.local  → always pod-1

Short form (within same namespace):
postgres-0.postgres-headless   → pod-0
postgres-1.postgres-headless   → pod-1
```

---

## 6. volumeClaimTemplates — Automatic Per-Pod PVCs

### WHAT
`volumeClaimTemplates` is the most important StatefulSet feature. It acts as a template to automatically create a unique PVC for each Pod — and permanently associates each PVC with its Pod.

### HOW — What gets created

```
StatefulSet: postgres, replicas: 3
volumeClaimTemplates name: postgres-data, size: 10Gi

Kubernetes automatically creates:
  PVC: postgres-data-postgres-0  → 10Gi → always mounted by postgres-0
  PVC: postgres-data-postgres-1  → 10Gi → always mounted by postgres-1
  PVC: postgres-data-postgres-2  → 10Gi → always mounted by postgres-2

postgres-0 crashes → comes back → mounts postgres-data-postgres-0 (its own data)
postgres-1 crashes → comes back → mounts postgres-data-postgres-1 (its own data)
```

```bash
kubectl get pvc -n production
# NAME                       STATUS   VOLUME        CAPACITY   STORAGECLASS
# postgres-data-postgres-0   Bound    pvc-aaa       10Gi       standard
# postgres-data-postgres-1   Bound    pvc-bbb       10Gi       standard
# postgres-data-postgres-2   Bound    pvc-ccc       10Gi       standard
```

**Critical: PVCs are NOT deleted when you delete a StatefulSet or scale down.**
```bash
kubectl delete statefulset postgres     # pods deleted, PVCs REMAIN
kubectl scale statefulset postgres --replicas=1  # pod-1 and pod-2 deleted, their PVCs REMAIN

# To delete PVCs manually:
kubectl delete pvc postgres-data-postgres-1
kubectl delete pvc postgres-data-postgres-2
```

This is intentional — protects against accidental data loss.

---

## 7. Ordered Lifecycle — How StatefulSets Start and Stop

### Startup (OrderedReady — default)

```
Create StatefulSet with replicas: 3

Step 1: Create postgres-0
        → Wait until postgres-0 is Running AND Ready (passes readiness probe)
Step 2: Create postgres-1
        → Wait until postgres-1 is Running AND Ready
Step 3: Create postgres-2
        → All pods running

If postgres-0 fails to become Ready → postgres-1 and postgres-2 NEVER start
→ This forces you to fix postgres-0 first
```

### Scaling Down (reverse order)

```
Scale from 3 → 1:
  1. Delete postgres-2 → wait for termination
  2. Delete postgres-1 → wait for termination
  3. postgres-0 remains (the primary)

Never deletes the primary (pod-0) before replicas.
```

### Rolling Update (reverse order, one at a time)

```
Update image from postgres:15 to postgres:15.1:
  1. Update postgres-2 first → wait for Ready
  2. Update postgres-1 → wait for Ready
  3. Update postgres-0 last (the primary updates last)
```

**Why reverse order for updates?**
In a typical primary-replica setup, you update replicas first (less critical), then the primary. If an update breaks postgres-2, you can roll back before touching the primary — safest possible update strategy.

---

## 8. Canary Updates with partition

The `partition` field enables canary deployments for StatefulSets:

```yaml
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    partition: 2    # only update pods with index >= 2
                    # postgres-2 gets new image
                    # postgres-0 and postgres-1 keep old image
```

```bash
# Step 1: Set partition to update only the last replica
kubectl patch statefulset postgres -n production \
  -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":2}}}}'

# Update image
kubectl set image statefulset/postgres postgres=postgres:15.2 -n production

# Only postgres-2 gets updated → test it → if good:
# Step 2: Move partition to update all
kubectl patch statefulset postgres -n production \
  -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":0}}}}'
# Now postgres-1 and postgres-0 update in order
```

---

## 9. Complete Production Example — PostgreSQL Primary + Replicas

This is how you run a real PostgreSQL cluster in Kubernetes with one primary and two read replicas.

```yaml
# ── Secret ─────────────────────────────────────────────────────────────────
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: production
type: Opaque
stringData:
  POSTGRES_USER: postgres
  POSTGRES_PASSWORD: supersecretpassword
  POSTGRES_DB: taskdb
  REPLICATION_PASSWORD: replicapassword

---
# ── Headless Service (DNS for each pod) ────────────────────────────────────
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: production
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432

---
# ── Regular Service (load balance reads across all pods) ───────────────────
apiVersion: v1
kind: Service
metadata:
  name: postgres-svc
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432

---
# ── StatefulSet ─────────────────────────────────────────────────────────────
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  serviceName: postgres-headless
  podManagementPolicy: OrderedReady
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0

  template:
    metadata:
      labels:
        app: postgres
    spec:
      # Init container: configure primary vs replica based on pod index
      initContainers:
        - name: init-postgres
          image: postgres:15-alpine
          command:
            - bash
            - -c
            - |
              # Get pod index from hostname (postgres-0, postgres-1, postgres-2)
              ORDINAL=${HOSTNAME##*-}

              if [ "$ORDINAL" = "0" ]; then
                echo "Configuring as PRIMARY (postgres-0)"
                # Primary config already default — just log
                echo "PRIMARY mode: accepting all connections"
              else
                echo "Configuring as REPLICA (postgres-${ORDINAL})"
                echo "Will replicate from postgres-0.postgres-headless"
              fi
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_PASSWORD

      containers:
        - name: postgres
          image: postgres:15-alpine
          ports:
            - containerPort: 5432

          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_PASSWORD
            - name: POSTGRES_DB
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_DB
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name    # injected: postgres-0, postgres-1, etc.

          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "2"
              memory: "2Gi"

          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data

          startupProbe:
            exec:
              command:
                - sh
                - -c
                - pg_isready -U postgres
            failureThreshold: 30
            periodSeconds: 5
            timeoutSeconds: 3

          livenessProbe:
            exec:
              command:
                - sh
                - -c
                - pg_isready -U postgres
            periodSeconds: 20
            failureThreshold: 6
            timeoutSeconds: 5

          readinessProbe:
            exec:
              command:
                - sh
                - -c
                - pg_isready -U postgres
            periodSeconds: 10
            failureThreshold: 3
            timeoutSeconds: 5

      terminationGracePeriodSeconds: 60

  # ── PVC Template — one 10Gi PVC per pod ─────────────────────────────────
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: standard
        resources:
          requests:
            storage: 10Gi
```

---

## 10. Connecting to Specific Pods

With StatefulSets and DNS, your application can connect to specific pods:

```javascript
// Connect to PRIMARY only (writes)
const primaryPool = new Pool({
  host: 'postgres-0.postgres-headless.production.svc.cluster.local',
  port: 5432,
  database: 'taskdb',
  user: 'postgres',
  password: process.env.POSTGRES_PASSWORD
});

// Connect to any replica (reads — load balanced by the regular Service)
const replicaPool = new Pool({
  host: 'postgres-svc',    // goes to any available pod
  port: 5432,
  database: 'taskdb',
  user: 'postgres',
  password: process.env.POSTGRES_PASSWORD
});

// Write goes to primary
async function createTask(title) {
  return primaryPool.query(
    'INSERT INTO tasks (title) VALUES ($1) RETURNING *', [title]
  );
}

// Reads can go to any replica
async function getTasks() {
  return replicaPool.query('SELECT * FROM tasks ORDER BY created_at DESC');
}
```

---

## 11. StatefulSet for Redis Sentinel (HA Redis)

Redis Sentinel provides automatic failover for Redis — another classic StatefulSet use case.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: redis
  serviceName: redis-headless

  template:
    metadata:
      labels:
        app: redis
    spec:
      initContainers:
        - name: init-redis
          image: redis:7-alpine
          command:
            - bash
            - -c
            - |
              ORDINAL=${HOSTNAME##*-}
              mkdir -p /data
              if [ "$ORDINAL" = "0" ]; then
                # Primary config
                cat > /data/redis.conf << EOF
              bind 0.0.0.0
              port 6379
              appendonly yes
              EOF
              else
                # Replica config — replicate from redis-0
                cat > /data/redis.conf << EOF
              bind 0.0.0.0
              port 6379
              replicaof redis-0.redis-headless 6379
              appendonly yes
              EOF
              fi
          volumeMounts:
            - name: redis-data
              mountPath: /data

      containers:
        - name: redis
          image: redis:7-alpine
          command: ["redis-server", "/data/redis.conf"]
          ports:
            - containerPort: 6379
          volumeMounts:
            - name: redis-data
              mountPath: /data
          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "1Gi"
          livenessProbe:
            exec:
              command: ["redis-cli", "ping"]
            periodSeconds: 20
            failureThreshold: 3
          readinessProbe:
            exec:
              command: ["redis-cli", "ping"]
            periodSeconds: 10
            failureThreshold: 3

  volumeClaimTemplates:
    - metadata:
        name: redis-data
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: standard
        resources:
          requests:
            storage: 5Gi
```

---

## 12. StatefulSet CLI Commands

```bash
# ── Create and manage ──────────────────────────────────────────────────────────
kubectl apply -f statefulset.yaml
kubectl get statefulsets -n production
kubectl get sts -n production                    # shorthand
kubectl describe statefulset postgres -n production

# ── Check pods (notice ordered names) ─────────────────────────────────────────
kubectl get pods -l app=postgres -n production
# NAME           READY   STATUS    RESTARTS   AGE
# postgres-0     1/1     Running   0          5m    ← always index 0
# postgres-1     1/1     Running   0          4m    ← always index 1
# postgres-2     1/1     Running   0          3m    ← always index 2

# ── Check PVCs (one per pod) ───────────────────────────────────────────────────
kubectl get pvc -n production
# NAME                       STATUS   VOLUME   CAPACITY
# postgres-data-postgres-0   Bound    ...      10Gi
# postgres-data-postgres-1   Bound    ...      10Gi
# postgres-data-postgres-2   Bound    ...      10Gi

# ── Connect to a specific pod ─────────────────────────────────────────────────
kubectl exec -it postgres-0 -n production -- psql -U postgres

# ── Verify DNS (from inside another pod) ──────────────────────────────────────
kubectl run test --image=busybox --rm -it --restart=Never -n production -- sh
# Inside:
nslookup postgres-0.postgres-headless
nslookup postgres-1.postgres-headless
nslookup postgres-headless    # returns ALL pod IPs

# ── Scaling ───────────────────────────────────────────────────────────────────
kubectl scale statefulset postgres --replicas=5 -n production
# pods 3 and 4 are added in order AFTER 0,1,2 are Ready

kubectl scale statefulset postgres --replicas=1 -n production
# pods 2, then 1 are removed (reverse order)
# pod 0 (primary) remains

# ── Rolling update ─────────────────────────────────────────────────────────────
kubectl set image statefulset/postgres postgres=postgres:15.2 -n production
kubectl rollout status statefulset/postgres -n production
# Updates pod-2, then pod-1, then pod-0

# ── Rollback ──────────────────────────────────────────────────────────────────
kubectl rollout undo statefulset/postgres -n production

# ── Force restart a specific pod (re-run without deleting PVC) ────────────────
kubectl delete pod postgres-1 -n production
# StatefulSet creates a new postgres-1 immediately with the same PVC

# ── Pause updates (OnDelete strategy) ─────────────────────────────────────────
kubectl patch statefulset postgres -n production \
  -p '{"spec":{"updateStrategy":{"type":"OnDelete"}}}'
# With OnDelete: pods only update when you manually delete them
# Gives you full manual control over which pod updates when
```

---

## 13. StatefulSet vs Deployment — When to Use Which

```
Use DEPLOYMENT when:
  ✅ App is stateless (each pod is identical)
  ✅ Pods can be replaced with a fresh copy without data loss
  ✅ Order of startup/shutdown doesn't matter
  ✅ Pods don't need to know about each other
  Examples: API servers, web frontends, workers, proxies

Use STATEFULSET when:
  ✅ Each pod needs a stable, unique identity
  ✅ Each pod needs its own persistent storage
  ✅ Pods need to discover each other by stable hostname
  ✅ Startup/shutdown order matters
  ✅ You're running a distributed system with leader/follower
  Examples: PostgreSQL, MySQL, MongoDB, Redis, Kafka, Zookeeper,
            Elasticsearch, etcd, Cassandra
```

---

## 14. Common StatefulSet Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Forgetting the Headless Service | StatefulSet can't create pods (required field) | Always create Headless Service BEFORE StatefulSet |
| Using `podManagementPolicy: Parallel` | Replicas start before primary, connection failures | Use `OrderedReady` for databases |
| Deleting PVCs after scaling down | Data loss when scaling back up | Never delete PVCs without explicit intention |
| Using a regular Service for writes | Writes randomly hit replicas, data corruption | Always direct writes to pod-0 hostname explicitly |
| Forgetting `PGDATA` subdirectory | PostgreSQL init fails on existing empty volume | Always set `PGDATA` to a subdirectory |
| Updating all replicas at once | No safe rollback point | Use partition for canary updates |

---

## 15. Hands-On Practice Tasks

```bash
# 1. Create the Headless Service first
kubectl apply -f postgres-headless-svc.yaml -n production

# 2. Create the StatefulSet
kubectl apply -f statefulset.yaml -n production

# 3. Watch pods start in order
kubectl get pods -l app=postgres -n production -w
# Should see: postgres-0 Running → postgres-1 starts → postgres-1 Running → postgres-2 starts

# 4. Check PVCs were created automatically
kubectl get pvc -n production
# Should see postgres-data-postgres-0, postgres-data-postgres-1, postgres-data-postgres-2

# 5. Test DNS stability
kubectl exec -it postgres-0 -n production -- sh
# Inside:
# hostname   → should print "postgres-0"
# exit

# 6. Delete postgres-0 and watch it come back with SAME name
kubectl delete pod postgres-0 -n production
kubectl get pods -l app=postgres -n production -w
# Watch postgres-0 be recreated (NOT with a random name)

# 7. Verify PVC still attached after pod recreation
kubectl exec -it postgres-0 -n production -- psql -U postgres -c "\l"
# Data should still be there

# 8. Test ordered scale-down
kubectl scale statefulset postgres --replicas=1 -n production
kubectl get pods -w    # watch 2 deleted before 1

# 9. Test DNS from another pod
kubectl run dns-test --image=busybox --rm -it --restart=Never -n production -- \
  nslookup postgres-0.postgres-headless

# 10. Check rollout history
kubectl rollout history statefulset/postgres -n production
```

---

## 16. Quiz — Test Yourself / Test Others

1. What are the five guarantees a StatefulSet provides that a Deployment does not?
2. Why is a Headless Service required for a StatefulSet?
3. What DNS name does `postgres-1` get in a StatefulSet named `postgres` with headless service `postgres-headless` in namespace `production`?
4. What is `volumeClaimTemplates` and how does it differ from specifying a volume directly?
5. What happens to PVCs when you delete a StatefulSet?
6. What does `podManagementPolicy: OrderedReady` guarantee during startup?
7. What does the `partition` field in `updateStrategy.rollingUpdate` control?
8. Why do rolling updates in a StatefulSet go in REVERSE order (2 → 1 → 0)?
9. In what scenario would you use `updateStrategy: OnDelete`?
10. Your StatefulSet pods are stuck in Pending. The Headless Service exists. What is the next most likely cause?

<details>
<summary>Answers</summary>

1. (1) Stable, unique Pod names with ordinal index (`postgres-0`, `postgres-1`). (2) Stable network identity — each pod gets a stable DNS hostname via the Headless Service. (3) Stable persistent storage — each pod gets its own dedicated PVC via volumeClaimTemplates, permanently bound to that pod. (4) Ordered deployment — pods start sequentially (0 must be Ready before 1 starts). (5) Ordered scaling and updates — scale down in reverse order, rolling updates one at a time in reverse.

2. The Headless Service provides the DNS infrastructure for per-pod stable hostnames. When `serviceName` references a Headless Service, Kubernetes creates DNS A records for each pod in the format `<pod-name>.<service-name>.<namespace>.svc.cluster.local`. Without this, there's no way to address individual pods by stable name. The `serviceName` field is required — StatefulSet creation fails without it.

3. `postgres-1.postgres-headless.production.svc.cluster.local`. Within the same namespace: `postgres-1.postgres-headless`. Format: `<statefulset-name>-<ordinal>.<headless-service-name>.<namespace>.svc.cluster.local`.

4. `volumeClaimTemplates` is a PVC template that Kubernetes uses to automatically create one unique PVC per pod (`postgres-data-0`, `postgres-data-1`, `postgres-data-2`). The association between pod and PVC is permanent — if the pod is deleted and recreated, it always gets the same PVC back. Specifying a volume directly (e.g., a single PVC name) would mean all pods try to mount the same PVC, which is impossible with RWO access mode and wrong for databases where each node needs its own data.

5. PVCs are NOT deleted when you delete a StatefulSet (or scale it down). They remain in the cluster indefinitely. This is intentional data protection — accidental StatefulSet deletion should not cause data loss. You must manually delete PVCs with `kubectl delete pvc` if you want to remove the data.

6. `OrderedReady` guarantees that pod-0 must be in Running AND Ready state (passing readiness probe) before pod-1 is created, and pod-1 must be Running AND Ready before pod-2 is created. If any pod fails to become Ready, the sequence stops — subsequent pods are never created until the problem is resolved.

7. `partition` acts as a threshold: only pods with an ordinal index >= partition value receive the new image during a rolling update. Pods with index < partition keep the old image. Example: `partition: 2` — only pod-2 gets updated; pods 0 and 1 stay unchanged. Used for canary deployments: test the update on the last replica before committing to updating the primary.

8. Pods are updated in reverse order (highest ordinal first) because in a typical primary-replica setup, pod-0 is the primary (most critical). By updating replicas first (pod-2, pod-1), you can test the new version on non-critical replicas. If the update breaks something, you can rollback before it ever reaches the primary — protecting the most important node in the cluster.

9. `OnDelete` is used when you want complete manual control over when each pod is updated. With OnDelete, the update only happens when you manually delete a pod — the StatefulSet then recreates it with the new spec. Use it for very sensitive databases where you want to personally control and verify each pod's update, one at a time, with manual checks between each.

10. The most likely cause is that the PVCs from `volumeClaimTemplates` can't be provisioned — either the StorageClass doesn't exist, the cluster has no dynamic provisioner, or quota limits are exceeded. Check with `kubectl describe pvc <pvc-name>` and look for Events explaining why provisioning failed. Also check `kubectl get storageclass` to verify the StorageClass exists.

</details>

---

## 17. Summary

After today you can explain and implement:
- **Why Deployments break for stateful apps** — anonymous pods, shared/lost PVCs, no ordering
- **The 5 StatefulSet guarantees** — stable names, stable DNS, stable storage, ordered startup, ordered updates
- **StatefulSet vs Deployment** — complete comparison table
- **Headless Service** — why required, DNS format per pod, two-service pattern (headless + regular)
- **volumeClaimTemplates** — automatic per-pod PVC creation, permanent pod-PVC binding
- **Ordered lifecycle** — startup (0→1→2), scale-down (2→1→0), rolling updates (2→1→0)
- **partition** — canary updates for StatefulSets
- **Complete PostgreSQL StatefulSet** — 3 replicas with init containers, probes, volumeClaimTemplates
- **Redis StatefulSet** — primary/replica configuration via init container
- **Connecting to specific pods** — writes to pod-0 hostname, reads via regular Service
- **PVC retention** — PVCs survive StatefulSet deletion, must be manually cleaned up
- Common mistakes and debugging approach

**Coming up on Day 23:** DaemonSets, Jobs & CronJobs — three more controllers for specific workload patterns: running one Pod on every node (log collectors, monitoring agents), one-off batch tasks, and scheduled recurring tasks.
