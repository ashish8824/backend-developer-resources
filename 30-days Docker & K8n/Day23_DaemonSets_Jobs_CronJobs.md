# Day 23: DaemonSets, Jobs & CronJobs — Workload Patterns for Every Use Case

**Goal for today:** Understand the three remaining Kubernetes controllers — DaemonSets (one Pod per node), Jobs (run to completion), and CronJobs (scheduled tasks). Learn when each applies, how they differ from Deployments and StatefulSets, and how to implement real-world use cases: log collection, monitoring agents, database migrations, and scheduled reports. By the end you should be able to choose the right controller for any workload pattern.

**Format:** WHAT → WHY → HOW for every concept.

---

## 0. Quick Recap (open your teaching session with this)

- Day 14: Deployment — stateless, interchangeable, long-running Pods
- Day 22: StatefulSet — stateful, unique identity, ordered Pods

**The full controller map:**
```
Controller       │ Pods           │ Lifecycle        │ Use case
─────────────────┼────────────────┼──────────────────┼──────────────────────────
Deployment       │ N identical    │ Run forever      │ Stateless APIs, frontends
StatefulSet      │ N with identity│ Run forever      │ Databases, message brokers
DaemonSet        │ 1 per node     │ Run forever      │ Node agents, log collectors
Job              │ 1 to N         │ Run to completion│ Migrations, batch tasks
CronJob          │ 1 to N         │ Scheduled        │ Reports, backups, cleanup
```

**Today's connecting thought:**
> Deployments and StatefulSets cover the "always-on" workloads. But real systems also need infrastructure agents on every single node (DaemonSets), one-off tasks that should run exactly once (Jobs), and recurring scheduled work (CronJobs). Today we complete the full controller picture.

---

## 1. DaemonSet — One Pod on Every Node

### WHAT
A **DaemonSet** ensures that exactly **one copy of a Pod runs on every node** in the cluster (or a selected subset of nodes). When a new node joins the cluster, the DaemonSet automatically adds a Pod to it. When a node is removed, its Pod is garbage collected.

### WHY
Some infrastructure tasks must run on every node — not because you need N replicas for availability, but because each instance serves that specific node:

```
Cluster with 5 nodes:
┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
│ Node 1  │ │ Node 2  │ │ Node 3  │ │ Node 4  │ │ Node 5  │
│         │ │         │ │         │ │         │ │         │
│[fluentd]│ │[fluentd]│ │[fluentd]│ │[fluentd]│ │[fluentd]│
└─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘

Each fluentd collects logs from ITS OWN NODE's containers.
You can't have one central fluentd — it can't read files on other nodes.
You need exactly ONE per node. That's a DaemonSet.
```

**Classic DaemonSet use cases:**
- **Log collection:** Fluentd, Filebeat, Logstash — reads container logs from node's filesystem
- **Monitoring agents:** Prometheus Node Exporter — collects node-level metrics (CPU, disk, network)
- **Security agents:** Falco, Datadog Agent — monitors processes on each node
- **Network plugins:** Calico, Flannel, Weave — implements CNI networking on each node
- **Storage plugins:** CSI node drivers — manages local storage on each node

### HOW — DaemonSet YAML

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system    # log collectors typically run in kube-system
  labels:
    app: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd

  # ── Update Strategy ─────────────────────────────────────────────────────────
  updateStrategy:
    type: RollingUpdate       # RollingUpdate | OnDelete
    rollingUpdate:
      maxUnavailable: 1       # update 1 node at a time

  template:
    metadata:
      labels:
        app: fluentd
    spec:
      # ── Run on every node including control plane ────────────────────────────
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
        - key: node-role.kubernetes.io/master
          operator: Exists
          effect: NoSchedule

      # ── Access node's filesystem (to read container logs) ────────────────────
      volumes:
        - name: varlog
          hostPath:
            path: /var/log           # node's actual /var/log directory
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
            type: DirectoryOrCreate

      containers:
        - name: fluentd
          image: fluent/fluentd-kubernetes-daemonset:v1.16-debian-elasticsearch8-1
          env:
            - name: FLUENT_ELASTICSEARCH_HOST
              value: "elasticsearch-svc.monitoring.svc.cluster.local"
            - name: FLUENT_ELASTICSEARCH_PORT
              value: "9200"
            - name: FLUENT_ELASTICSEARCH_SCHEME
              value: "http"

          resources:
            requests:
              cpu: "100m"
              memory: "200Mi"
            limits:
              cpu: "500m"
              memory: "500Mi"

          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true

          livenessProbe:
            httpGet:
              path: /fluentd.healthcheck?json=%7B%22ping%22%3A+%22pong%22%7D
              port: 9880
            initialDelaySeconds: 5
            periodSeconds: 30

      terminationGracePeriodSeconds: 30
```

### Running on a Subset of Nodes (nodeSelector)

Sometimes you only want a DaemonSet on specific nodes — e.g., GPU nodes, nodes with SSDs:

```yaml
spec:
  template:
    spec:
      # Only schedule on nodes with label: node-type=gpu
      nodeSelector:
        node-type: gpu

      # Or use nodeAffinity for more complex rules:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: node-type
                    operator: In
                    values: [gpu, high-memory]
```

### DaemonSet CLI Commands

```bash
kubectl get daemonsets -n kube-system
kubectl get ds -n kube-system              # shorthand
kubectl describe daemonset fluentd -n kube-system

# DaemonSet shows DESIRED=CURRENT=READY (one per node)
# NAME      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR
# fluentd   5         5         5       5            5           <none>

kubectl rollout status daemonset/fluentd -n kube-system
kubectl rollout restart daemonset/fluentd -n kube-system

# Check which node each pod runs on
kubectl get pods -l app=fluentd -o wide -n kube-system
# Shows NODE column — one pod per node
```

---

## 2. Tolerations and Taints — Why DaemonSets Need Them

### WHAT
**Taints** mark a node as "only accept Pods that explicitly tolerate me."
**Tolerations** on a Pod say "I can tolerate this taint — schedule me here."

### WHY for DaemonSets
Control plane nodes (master nodes) have a taint:
```
node-role.kubernetes.io/control-plane:NoSchedule
```
This prevents normal workloads from running on the control plane. But DaemonSets for logging and monitoring NEED to run on every node including control plane. Adding a toleration bypasses the taint.

### HOW

```yaml
# Add to DaemonSet pod spec to run on ALL nodes including control plane:
tolerations:
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule

# Custom taints on nodes:
# kubectl taint nodes node1 dedicated=monitoring:NoSchedule
tolerations:
  - key: dedicated
    value: monitoring
    operator: Equal
    effect: NoSchedule
```

---

## 3. Prometheus Node Exporter — Real DaemonSet Example

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
      annotations:
        prometheus.io/scrape: "true"    # tells Prometheus to scrape this pod
        prometheus.io/port: "9100"
    spec:
      hostNetwork: true                 # use node's network namespace directly
      hostPID: true                     # access node's process list

      tolerations:
        - operator: Exists              # tolerate ALL taints — run on every node

      containers:
        - name: node-exporter
          image: prom/node-exporter:v1.7.0
          args:
            - '--path.rootfs=/host'
            - '--collector.filesystem.mount-points-exclude=^/(dev|proc|sys|var/lib/docker/.+)($|/)'
          ports:
            - containerPort: 9100
              hostPort: 9100            # expose on node's port directly
          resources:
            requests:
              cpu: "100m"
              memory: "30Mi"
            limits:
              cpu: "250m"
              memory: "180Mi"
          volumeMounts:
            - name: root
              mountPath: /host
              readOnly: true

      volumes:
        - name: root
          hostPath:
            path: /

      terminationGracePeriodSeconds: 30
```

---

## 4. Job — Run a Task to Completion

### WHAT
A **Job** creates one or more Pods and ensures they run to **successful completion** (exit code 0). Unlike Deployments, Jobs are not long-running — they terminate when done.

### WHY
Some tasks should run exactly once and stop:
- Database schema migrations
- Data seeding or loading
- One-off data transformations
- Certificate generation
- Test suites

Without Jobs, you'd run these in a Pod — but a bare Pod won't be retried if it fails.
A Job handles failure automatically: if the Pod fails, Job creates a new one until success.

### HOW — Job YAML

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  namespace: production
spec:
  # ── Completion Settings ──────────────────────────────────────────────────────
  completions: 1          # number of successful completions needed (default: 1)
  parallelism: 1          # number of pods running in parallel (default: 1)

  # ── Failure Handling ─────────────────────────────────────────────────────────
  backoffLimit: 4         # retry up to 4 times before marking Job as failed
  activeDeadlineSeconds: 300   # fail the job if not completed within 5 minutes

  # ── Cleanup ──────────────────────────────────────────────────────────────────
  ttlSecondsAfterFinished: 3600   # delete job (and its pods) 1 hour after completion
                                   # prevents accumulation of completed jobs

  template:
    metadata:
      labels:
        app: db-migration
        job-name: db-migration
    spec:
      # ── CRITICAL: Jobs must use restartPolicy Never or OnFailure ────────────
      restartPolicy: OnFailure    # Never | OnFailure (NOT Always — Jobs would loop forever)

      containers:
        - name: migration
          image: taskapi:1.0
          command: ["node", "scripts/migrate.js"]

          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: taskapi-secrets
                  key: DATABASE_URL
            - name: NODE_ENV
              value: "production"

          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"

      # ── Wait for postgres before running ─────────────────────────────────────
      initContainers:
        - name: wait-for-postgres
          image: busybox:1.36
          command:
            - sh
            - -c
            - until nc -z postgres-svc 5432; do echo "waiting..."; sleep 2; done

      terminationGracePeriodSeconds: 30
```

```bash
kubectl apply -f migration-job.yaml

# Monitor job
kubectl get jobs -n production
# NAME           COMPLETIONS   DURATION   AGE
# db-migration   1/1           8s         1m    ← 1/1 = completed

kubectl describe job db-migration -n production

# View job pod logs
kubectl logs -l job-name=db-migration -n production

# Wait for job to complete (useful in scripts/CI-CD)
kubectl wait job/db-migration --for=condition=complete --timeout=120s -n production

# Check if job failed
kubectl get job db-migration -n production -o jsonpath='{.status.failed}'
```

### restartPolicy — Critical Distinction

```
restartPolicy: Never
  → If pod fails (exit code != 0): Job creates a NEW pod (never restarts same pod)
  → Useful for: tasks where you want fresh environment on retry
  → Risk: accumulates failed pods (set ttlSecondsAfterFinished)

restartPolicy: OnFailure
  → If pod fails: kubelet RESTARTS the same pod (up to backoffLimit)
  → Useful for: tasks where retry on same pod is fine
  → Simpler — fewer pods created on failure
```

---

## 5. Parallel Jobs — Batch Processing

Jobs can run multiple Pods in parallel for batch workloads:

### Pattern 1: Fixed completion count (process N items)

```yaml
spec:
  completions: 10     # need 10 successful completions total
  parallelism: 3      # run 3 pods at a time
  # Total pods over time: ceil(10/3) batches × 3 pods = ~12 pods
  # All 10 items get processed

  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: processor
          image: my-batch-processor:1.0
          env:
            - name: JOB_COMPLETION_INDEX
              valueFrom:
                fieldRef:
                  fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
              # Each pod gets a unique index: 0, 1, 2, ... 9
              # Your app can use this to process the right slice of data
```

### Pattern 2: Work queue (each pod pulls from a queue)

```yaml
spec:
  completions: null   # don't track completions — pods pull from queue
  parallelism: 5      # 5 workers pulling from queue simultaneously
  completionMode: NonIndexed

  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: worker
          image: my-worker:1.0
          # Worker pulls tasks from Redis/RabbitMQ/SQS and exits when queue empty
```

---

## 6. CronJob — Scheduled Recurring Tasks

### WHAT
A **CronJob** creates Jobs on a **schedule** — like a Kubernetes-native cron daemon. It uses standard cron syntax to define when to run.

### WHY
Production systems need regular scheduled work:
- Nightly database backups
- Hourly analytics reports
- Daily cleanup of old data
- Periodic cache warming
- Weekly email digests

### HOW — CronJob YAML

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
  namespace: production
spec:
  # ── Schedule (standard cron format) ─────────────────────────────────────────
  # ┌──────────── minute (0-59)
  # │  ┌─────────── hour (0-23)
  # │  │  ┌──────────── day of month (1-31)
  # │  │  │  ┌─────────── month (1-12)
  # │  │  │  │  ┌──────────── day of week (0-6, 0=Sunday)
  # │  │  │  │  │
  schedule: "0 2 * * *"    # every day at 2:00 AM UTC

  # ── Timezone (Kubernetes 1.27+) ──────────────────────────────────────────────
  timeZone: "Asia/Kolkata"   # IST — so "0 2 * * *" means 2am IST

  # ── Job execution policy ─────────────────────────────────────────────────────
  concurrencyPolicy: Forbid   # Allow | Forbid | Replace
                               # Forbid: skip new job if previous still running
                               # Replace: cancel previous, start new
                               # Allow: run multiple simultaneously

  # ── History limits ───────────────────────────────────────────────────────────
  successfulJobsHistoryLimit: 3   # keep last 3 successful job records
  failedJobsHistoryLimit: 3       # keep last 3 failed job records

  # ── Missed job handling ──────────────────────────────────────────────────────
  startingDeadlineSeconds: 3600   # if missed schedule, run within 3600s or skip

  # ── Suspend (pause without deleting) ─────────────────────────────────────────
  suspend: false

  # ── Job template ─────────────────────────────────────────────────────────────
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 1800    # 30 minute timeout for backup
      ttlSecondsAfterFinished: 86400 # clean up after 24 hours

      template:
        metadata:
          labels:
            app: db-backup
        spec:
          restartPolicy: OnFailure

          initContainers:
            - name: wait-for-postgres
              image: busybox:1.36
              command:
                - sh
                - -c
                - until nc -z postgres-svc 5432; do sleep 2; done

          containers:
            - name: backup
              image: postgres:15-alpine
              command:
                - sh
                - -c
                - |
                  TIMESTAMP=$(date +%Y%m%d_%H%M%S)
                  FILENAME="backup_${TIMESTAMP}.sql.gz"
                  echo "Starting backup: ${FILENAME}"

                  pg_dump \
                    -h postgres-svc \
                    -U $POSTGRES_USER \
                    $POSTGRES_DB | gzip > /backup/${FILENAME}

                  echo "Backup complete: ${FILENAME}"
                  ls -lh /backup/

              env:
                - name: POSTGRES_USER
                  valueFrom:
                    secretKeyRef:
                      name: taskapi-secrets
                      key: POSTGRES_USER
                - name: PGPASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: taskapi-secrets
                      key: POSTGRES_PASSWORD
                - name: POSTGRES_DB
                  valueFrom:
                    secretKeyRef:
                      name: taskapi-secrets
                      key: POSTGRES_DB

              volumeMounts:
                - name: backup-storage
                  mountPath: /backup

              resources:
                requests:
                  cpu: "200m"
                  memory: "256Mi"
                limits:
                  cpu: "500m"
                  memory: "512Mi"

          volumes:
            - name: backup-storage
              persistentVolumeClaim:
                claimName: backup-pvc   # or use S3/GCS via external tool
```

---

## 7. Cron Schedule Quick Reference

```
┌──────── minute (0-59)
│  ┌───── hour (0-23)
│  │  ┌── day of month (1-31)
│  │  │  ┌── month (1-12 or Jan-Dec)
│  │  │  │  ┌── day of week (0-6, 0=Sun, 7=Sun)
│  │  │  │  │
*  *  *  *  *

Common schedules:
"*/5 * * * *"    every 5 minutes
"0 * * * *"      every hour (at :00)
"0 0 * * *"      daily at midnight
"0 2 * * *"      daily at 2:00 AM
"0 2 * * 0"      weekly, Sunday at 2:00 AM
"0 2 1 * *"      monthly, 1st at 2:00 AM
"0 */6 * * *"    every 6 hours
"30 8 * * 1-5"   8:30 AM on weekdays (Mon-Fri)
"@hourly"        same as "0 * * * *"
"@daily"         same as "0 0 * * *"
"@weekly"        same as "0 0 * * 0"
"@monthly"       same as "0 0 1 * *"
```

---

## 8. ConcurrencyPolicy — Handling Overlapping Jobs

```
concurrencyPolicy: Forbid (recommended for most cases)
  If previous job is still running when schedule fires again:
  → Skip the new run
  → Use for: backups, migrations (never run two simultaneously)

concurrencyPolicy: Replace
  If previous job is still running when schedule fires again:
  → Cancel the old job, start new
  → Use for: cache warming (fresh run is always preferred)

concurrencyPolicy: Allow (default)
  → Run new job regardless of whether old is running
  → Use for: truly independent runs where overlap is fine
  → Risk: jobs pile up if each takes longer than the schedule interval
```

---

## 9. More Real-World CronJob Examples

### Database cleanup — delete old records

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup-old-tasks
  namespace: production
spec:
  schedule: "0 3 * * *"      # 3am daily
  timeZone: "Asia/Kolkata"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 2
      ttlSecondsAfterFinished: 3600
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: cleanup
              image: postgres:15-alpine
              command:
                - sh
                - -c
                - |
                  psql -h postgres-svc -U $POSTGRES_USER $POSTGRES_DB -c "
                    DELETE FROM tasks
                    WHERE done = true
                    AND created_at < NOW() - INTERVAL '30 days';

                    DELETE FROM audit_logs
                    WHERE created_at < NOW() - INTERVAL '90 days';

                    VACUUM ANALYZE tasks;
                  "
                  echo "Cleanup complete"
              env:
                - name: POSTGRES_USER
                  valueFrom:
                    secretKeyRef:
                      name: taskapi-secrets
                      key: POSTGRES_USER
                - name: PGPASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: taskapi-secrets
                      key: POSTGRES_PASSWORD
                - name: POSTGRES_DB
                  valueFrom:
                    secretKeyRef:
                      name: taskapi-secrets
                      key: POSTGRES_DB
              resources:
                requests:
                  cpu: "100m"
                  memory: "128Mi"
                limits:
                  cpu: "250m"
                  memory: "256Mi"
```

### Analytics report — every hour

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hourly-analytics
  namespace: production
spec:
  schedule: "0 * * * *"
  timeZone: "Asia/Kolkata"
  concurrencyPolicy: Replace    # if previous still running, replace it
  jobTemplate:
    spec:
      ttlSecondsAfterFinished: 3600
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: analytics
              image: taskapi:1.0
              command: ["node", "scripts/generate-hourly-report.js"]
              envFrom:
                - configMapRef:
                    name: taskapi-config
              env:
                - name: DATABASE_URL
                  valueFrom:
                    secretKeyRef:
                      name: taskapi-secrets
                      key: DATABASE_URL
              resources:
                requests:
                  cpu: "200m"
                  memory: "256Mi"
                limits:
                  cpu: "500m"
                  memory: "512Mi"
```

---

## 10. Manually Triggering a CronJob (one-off run)

```bash
# Create a Job from a CronJob template immediately (without waiting for schedule)
kubectl create job --from=cronjob/db-backup db-backup-manual-$(date +%s) -n production

# Useful for:
# - Testing your CronJob YAML before the schedule fires
# - Manual backups before a deployment
# - Re-running a failed job immediately
```

---

## 11. Job & CronJob CLI Commands

```bash
# ── Jobs ──────────────────────────────────────────────────────────────────────
kubectl get jobs -n production
kubectl get jobs -n production -w               # watch completion
kubectl describe job db-migration -n production
kubectl logs -l job-name=db-migration -n production
kubectl logs -l job-name=db-migration --previous -n production   # if failed

# Wait for completion (great for CI/CD pipelines)
kubectl wait job/db-migration \
  --for=condition=complete \
  --timeout=300s \
  -n production

# Delete a job (also deletes its pods)
kubectl delete job db-migration -n production

# ── CronJobs ───────────────────────────────────────────────────────────────────
kubectl get cronjobs -n production
kubectl get cj -n production                    # shorthand
kubectl describe cronjob db-backup -n production

# See job history for a CronJob
kubectl get jobs -l app=db-backup -n production

# Suspend a CronJob (pause scheduling without deleting)
kubectl patch cronjob db-backup -n production \
  -p '{"spec":{"suspend":true}}'

# Resume
kubectl patch cronjob db-backup -n production \
  -p '{"spec":{"suspend":false}}'

# Manually trigger
kubectl create job --from=cronjob/db-backup manual-backup-$(date +%Y%m%d) -n production

# Delete CronJob (does NOT delete running jobs)
kubectl delete cronjob db-backup -n production
```

---

## 12. Putting It All Together — The Complete Controller Decision Tree

```
What kind of workload is this?
        │
        ├── Long-running process (never exits on its own)?
        │         │
        │         ├── Stateless (pods are interchangeable)?
        │         │         └── DEPLOYMENT
        │         │
        │         ├── Stateful (unique identity, ordered, own storage)?
        │         │         └── STATEFULSET
        │         │
        │         └── Must run on EVERY node?
        │                   └── DAEMONSET
        │
        └── Task (runs and exits)?
                  │
                  ├── One-off (run once)?
                  │         └── JOB
                  │
                  └── Scheduled (runs on a cron)?
                            └── CRONJOB
```

---

## 13. Real-World Scenario — Database Migration in CI/CD

This is the production pattern for running migrations safely before a new API version:

```yaml
# migration-job.yaml — applied BEFORE deploying new API version
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration-v2-1-0    # versioned name prevents conflicts
  namespace: production
  annotations:
    deployed-at: "2025-01-15T10:00:00Z"
    deploy-version: "v2.1.0"
spec:
  backoffLimit: 3
  activeDeadlineSeconds: 300
  ttlSecondsAfterFinished: 86400

  template:
    spec:
      restartPolicy: OnFailure

      initContainers:
        - name: wait-for-postgres
          image: busybox:1.36
          command:
            - sh
            - -c
            - until nc -z postgres-svc 5432; do sleep 2; done

      containers:
        - name: migration
          image: taskapi:v2.1.0
          command: ["node", "scripts/migrate.js"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: taskapi-secrets
                  key: DATABASE_URL
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
```

```bash
# CI/CD deployment pipeline:

# Step 1: Run migration Job, wait for it
kubectl apply -f migration-job.yaml -n production
kubectl wait job/db-migration-v2-1-0 \
  --for=condition=complete \
  --timeout=300s \
  -n production

# Step 2: Only if migration succeeds, deploy new API version
if [ $? -eq 0 ]; then
  kubectl set image deployment/taskapi api=taskapi:v2.1.0 -n production
  kubectl rollout status deployment/taskapi -n production
else
  echo "Migration failed! Aborting deployment."
  exit 1
fi
```

---

## 14. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| `restartPolicy: Always` in Job | Job pod restarts forever, never "completes" | Use `OnFailure` or `Never` |
| No `backoffLimit` | Job retries indefinitely on failure | Set `backoffLimit: 3-5` |
| No `ttlSecondsAfterFinished` | Completed jobs accumulate, clutter cluster | Set `ttlSecondsAfterFinished: 3600` |
| No `activeDeadlineSeconds` | Hung job runs forever | Set a reasonable timeout |
| `concurrencyPolicy: Allow` on long CronJobs | Jobs pile up when each takes longer than interval | Use `Forbid` for non-idempotent jobs |
| No `timeZone` on CronJob | Schedule runs in UTC, confusing for region-specific tasks | Set `timeZone: "Asia/Kolkata"` |
| DaemonSet without tolerations | Doesn't run on control-plane nodes | Add toleration for control-plane taint |
| Updating DaemonSet image | All nodes update simultaneously | Use `maxUnavailable: 1` |

---

## 15. Hands-On Practice Tasks

```bash
# ── DaemonSet ──────────────────────────────────────────────────────────────────
# 1. Deploy a simple DaemonSet with nginx (one per node)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: demo-ds
  namespace: default
spec:
  selector:
    matchLabels:
      app: demo-ds
  template:
    metadata:
      labels:
        app: demo-ds
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          resources:
            limits:
              cpu: "100m"
              memory: "64Mi"
EOF

# 2. Verify one pod per node
kubectl get pods -l app=demo-ds -o wide
# NODE column shows each unique node

# 3. Clean up
kubectl delete daemonset demo-ds

# ── Job ────────────────────────────────────────────────────────────────────────
# 4. Run a simple batch job
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-calculator
spec:
  backoffLimit: 3
  ttlSecondsAfterFinished: 300
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: pi
          image: perl:5.34
          command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
          resources:
            limits:
              cpu: "500m"
              memory: "128Mi"
EOF

# 5. Watch job complete
kubectl get jobs -w
kubectl logs -l job-name=pi-calculator

# ── CronJob ────────────────────────────────────────────────────────────────────
# 6. Create a CronJob that prints time every minute
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: time-printer
spec:
  schedule: "*/1 * * * *"
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      ttlSecondsAfterFinished: 60
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: printer
              image: busybox
              command: ["sh", "-c", "date && echo 'CronJob ran successfully'"]
              resources:
                limits:
                  cpu: "50m"
                  memory: "32Mi"
EOF

# 7. Watch CronJob create jobs every minute
kubectl get cronjobs
kubectl get jobs -w

# 8. Manually trigger it
kubectl create job --from=cronjob/time-printer manual-$(date +%s)

# 9. Suspend the CronJob
kubectl patch cronjob time-printer -p '{"spec":{"suspend":true}}'

# 10. Clean up
kubectl delete cronjob time-printer
kubectl delete job pi-calculator
```

---

## 16. Quiz — Test Yourself / Test Others

1. What is a DaemonSet and what problem does it solve that a Deployment cannot?
2. Name four real-world use cases for DaemonSets.
3. Why do DaemonSets for logging/monitoring often need tolerations?
4. What is the difference between `restartPolicy: Never` and `restartPolicy: OnFailure` in a Job?
5. Why can't a Job use `restartPolicy: Always`?
6. What does `backoffLimit: 4` mean in a Job spec?
7. What does `ttlSecondsAfterFinished` do and why is it important?
8. What does `concurrencyPolicy: Forbid` do in a CronJob?
9. Write the cron schedule for: "every day at 11:30 PM IST" (assuming timeZone is set).
10. What is the right way to manually trigger a CronJob immediately without waiting for its schedule?

<details>
<summary>Answers</summary>

1. A DaemonSet ensures exactly one Pod runs on every node (or a selected subset of nodes) in the cluster. It automatically adds Pods to new nodes when they join and removes Pods when nodes leave. A Deployment cannot do this — it deploys N replicas without caring which nodes they land on, and multiple replicas might land on the same node while other nodes get none.

2. Any four of: log collection (Fluentd, Filebeat reading node's /var/log), node metrics collection (Prometheus Node Exporter), security monitoring (Falco, Datadog Agent), network plugins (Calico, Flannel — CNI), storage drivers (CSI node plugins), GPU drivers, service mesh proxies (Envoy per node).

3. Control-plane nodes have a taint (`node-role.kubernetes.io/control-plane:NoSchedule`) that prevents normal Pods from being scheduled on them. Since log collectors and monitoring agents need to run on ALL nodes including the control plane, they must add a toleration that explicitly allows them to be scheduled on tainted nodes.

4. `restartPolicy: Never` — if the Pod fails (non-zero exit code), it is NOT restarted; instead, the Job creates a brand new Pod for the retry. This preserves the failed Pod for debugging. `restartPolicy: OnFailure` — if the Pod fails, kubelet restarts the same container within the existing Pod. Fewer Pods created overall but the failed container state is overwritten.

5. `restartPolicy: Always` means the container is always restarted when it exits — including when it exits with code 0 (success). A Job is designed to run to completion, so it needs to stop when the task succeeds. With `Always`, a successful container would be restarted immediately, running forever and never completing the Job. The API Server rejects Job specs with `restartPolicy: Always`.

6. `backoffLimit: 4` means the Job will retry up to 4 times after failure before giving up and marking itself as failed. After 4 failed Pod attempts, the Job status becomes `Failed` and no more Pods are created. The exponential backoff delay between retries increases: 10s, 20s, 40s, 80s.

7. `ttlSecondsAfterFinished` automatically deletes a completed (or failed) Job and all its Pods after the specified number of seconds. Without it, completed Jobs accumulate indefinitely — cluttering the namespace and etcd. For example, `ttlSecondsAfterFinished: 3600` deletes the Job and its Pods 1 hour after completion.

8. `concurrencyPolicy: Forbid` prevents a new Job from being started if the previous Job created by this CronJob is still running. The new scheduled run is simply skipped. This prevents multiple instances of the same job running simultaneously — critical for non-idempotent operations like database migrations or backups.

9. `"30 23 * * *"` — minute 30, hour 23 (11 PM). Since `timeZone: "Asia/Kolkata"` is set, this runs at 11:30 PM IST every day.

10. `kubectl create job --from=cronjob/<cronjob-name> <unique-job-name> -n <namespace>`. This creates a Job using the CronJob's job template immediately. The job runs right away without waiting for the schedule. Example: `kubectl create job --from=cronjob/db-backup manual-backup-20250115 -n production`.

</details>

---

## 17. Summary

After today you can explain and implement all five Kubernetes controllers:

- **DaemonSet** — one Pod per node, runs forever; use for log collectors, monitoring agents, network/storage plugins; needs tolerations for control-plane nodes
- **Job** — runs to completion, handles retries; `restartPolicy: Never/OnFailure`; `backoffLimit`, `activeDeadlineSeconds`, `ttlSecondsAfterFinished`; parallel and work-queue patterns
- **CronJob** — scheduled Jobs using cron syntax; `concurrencyPolicy`, `timeZone`, `suspend`, `successfulJobsHistoryLimit`; manually trigger with `kubectl create job --from=cronjob`
- **Complete controller decision tree** — stateless long-running → Deployment, stateful → StatefulSet, every node → DaemonSet, one-off task → Job, scheduled → CronJob

**Coming up on Day 24:** Helm — Kubernetes' package manager. Instead of managing dozens of individual YAML files, Helm packages them into Charts with configurable values, versioned releases, and one-command upgrades and rollbacks. The tool every production K8s team uses.
