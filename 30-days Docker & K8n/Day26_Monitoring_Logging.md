# Day 26: Monitoring & Logging — Prometheus, Grafana & Observability

**Goal for today:** Understand the three pillars of observability (metrics, logs, traces), how Prometheus scrapes and stores metrics, how Grafana visualises them, how to set up alerting, and how to implement structured logging in Kubernetes. By the end, you should be able to set up a complete observability stack and know how to diagnose any production issue from metrics and logs alone.

**Format:** WHAT → WHY → HOW for every concept.

---

## 0. Quick Recap (open your teaching session with this)

- Day 20: Health probes — liveness, readiness, startup (reactive health checking)
- Day 23: DaemonSets — running agents on every node (e.g., log collectors)

**Today's connecting thought:**
> Health probes tell Kubernetes whether a Pod is alive and ready — but they don't tell you WHY it crashed, HOW MUCH memory it's using over time, or WHETHER request latency is increasing. Monitoring and logging answer all of these questions. The difference between probes and monitoring: probes are binary (alive/dead), monitoring is dimensional (how alive, trending toward death, what's causing slowdown).

---

## 1. The Three Pillars of Observability

Every production system needs three types of telemetry:

```
┌─────────────────────────────────────────────────────────────────┐
│                    OBSERVABILITY PILLARS                          │
├───────────────┬─────────────────────────────────────────────────┤
│  METRICS      │ Numerical measurements over time                  │
│               │ "API has 2500 req/min, p99 latency 340ms"        │
│               │ Tool: Prometheus + Grafana                        │
├───────────────┼─────────────────────────────────────────────────┤
│  LOGS         │ Discrete timestamped events from the application  │
│               │ "2025-01-15 10:43:22 ERROR: DB connection failed" │
│               │ Tool: kubectl logs, Fluentd, Loki, ELK            │
├───────────────┼─────────────────────────────────────────────────┤
│  TRACES       │ End-to-end request path across services           │
│               │ "Request took 340ms: API 20ms, DB 300ms, Redis 5ms│
│               │ Tool: Jaeger, Zipkin, OpenTelemetry (Day 30)      │
└───────────────┴─────────────────────────────────────────────────┘
```

Today focuses on **Metrics** (Prometheus + Grafana) and **Logs** (kubectl logs + Loki). Traces are covered on Day 30.

---

## 2. What is Prometheus?

### WHAT
**Prometheus** is an open-source monitoring system that:
- **Scrapes** (pulls) metrics from your applications and infrastructure at regular intervals
- **Stores** time-series data (metric name + labels + timestamp + value)
- **Queries** data with PromQL (Prometheus Query Language)
- **Alerts** when metrics cross defined thresholds

### WHY Prometheus over alternatives (Datadog, CloudWatch, etc.)
- **Free and open source** — no per-host cost at scale
- **Pull model** — Prometheus pulls metrics from services (not push) — easier to control
- **Native Kubernetes** — auto-discovers services and Pods via K8s API
- **Ecosystem** — thousands of exporters for every technology
- **Industry standard** — most K8s tools expose Prometheus metrics by default

### The Prometheus Data Model

```
metric_name{label1="value1", label2="value2"} numeric_value timestamp

Examples:
http_requests_total{method="GET", path="/health", status="200"} 45231 1705123456
http_request_duration_seconds{quantile="0.99"} 0.341
container_memory_usage_bytes{pod="api-xyz", namespace="production"} 268435456
node_cpu_seconds_total{cpu="0", mode="user"} 12345.67
```

**Four metric types:**
| Type | What it tracks | Example |
|---|---|---|
| Counter | Always increasing value | Total requests, errors, bytes sent |
| Gauge | Value that goes up and down | Current memory usage, active connections |
| Histogram | Distribution of values (buckets) | Request duration, response size |
| Summary | Similar to Histogram, pre-calculated quantiles | p50, p90, p99 latency |

---

## 3. How Prometheus Scrapes Metrics

### The Pull Model

```
Every 15-30 seconds:

Prometheus → GET http://api-pod:3000/metrics
           → GET http://node-exporter:9100/metrics
           → GET http://postgres-exporter:9187/metrics

Each /metrics endpoint returns:
  # HELP http_requests_total Total HTTP requests
  # TYPE http_requests_total counter
  http_requests_total{method="GET",status="200"} 1247
  http_requests_total{method="POST",status="201"} 89
  http_requests_total{method="POST",status="400"} 12
  # HELP process_resident_memory_bytes Resident memory
  # TYPE process_resident_memory_bytes gauge
  process_resident_memory_bytes 52428800
```

### Service Discovery in Kubernetes

Prometheus auto-discovers Pods to scrape using Kubernetes API:

```yaml
# Prometheus scrapes any Pod with these annotations:
annotations:
  prometheus.io/scrape: "true"     # enable scraping
  prometheus.io/port: "3000"       # which port to scrape
  prometheus.io/path: "/metrics"   # metrics path (default: /metrics)
```

---

## 4. Exposing Metrics from Your Node.js App

### Install prom-client

```bash
npm install prom-client
```

### Add metrics to your Express app

```javascript
// metrics.js
const client = require('prom-client');

// Enable default metrics (process CPU, memory, event loop, GC)
const register = new client.Registry();
client.collectDefaultMetrics({ register });

// ── Custom Metrics ────────────────────────────────────────────────────────────

// Counter: total HTTP requests
const httpRequestsTotal = new client.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'path', 'status'],
  registers: [register]
});

// Histogram: request duration
const httpRequestDurationSeconds = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'path', 'status'],
  buckets: [0.01, 0.05, 0.1, 0.3, 0.5, 1, 2, 5],  // seconds
  registers: [register]
});

// Gauge: active database connections
const dbConnectionsActive = new client.Gauge({
  name: 'db_connections_active',
  help: 'Number of active database connections',
  registers: [register]
});

// Counter: database errors
const dbErrorsTotal = new client.Counter({
  name: 'db_errors_total',
  help: 'Total number of database errors',
  labelNames: ['operation'],
  registers: [register]
});

module.exports = {
  register,
  httpRequestsTotal,
  httpRequestDurationSeconds,
  dbConnectionsActive,
  dbErrorsTotal
};
```

```javascript
// server.js — wire metrics into Express
const express = require('express');
const {
  register,
  httpRequestsTotal,
  httpRequestDurationSeconds,
  dbConnectionsActive,
  dbErrorsTotal
} = require('./metrics');

const app = express();

// ── Metrics middleware (runs on every request) ─────────────────────────────────
app.use((req, res, next) => {
  const start = Date.now();

  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    const labels = {
      method: req.method,
      path: req.route?.path || req.path,
      status: res.statusCode.toString()
    };

    httpRequestsTotal.inc(labels);
    httpRequestDurationSeconds.observe(labels, duration);
  });

  next();
});

// ── Metrics endpoint (scraped by Prometheus) ───────────────────────────────────
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

// ── Health endpoints ──────────────────────────────────────────────────────────
app.get('/health', (req, res) => {
  res.json({ status: 'healthy' });
});

// ── API routes ────────────────────────────────────────────────────────────────
app.get('/api/tasks', async (req, res) => {
  try {
    const result = await pool.query('SELECT * FROM tasks');
    res.json(result.rows);
  } catch (err) {
    dbErrorsTotal.inc({ operation: 'select' });
    res.status(500).json({ error: err.message });
  }
});

// ── Update DB connections gauge ────────────────────────────────────────────────
setInterval(() => {
  dbConnectionsActive.set(pool.totalCount - pool.idleCount);
}, 5000);
```

### Update Deployment annotations to enable scraping

```yaml
template:
  metadata:
    labels:
      app: taskapi
    annotations:
      prometheus.io/scrape: "true"
      prometheus.io/port: "3000"
      prometheus.io/path: "/metrics"
```

---

## 5. Installing the Monitoring Stack (Helm)

The easiest way to get the full stack — Prometheus + Grafana + AlertManager + Node Exporter + kube-state-metrics.

```bash
# Add Prometheus community Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install the full kube-prometheus-stack
# (Prometheus + Grafana + AlertManager + Node Exporter + kube-state-metrics)
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=admin123 \
  --set prometheus.prometheusSpec.scrapeInterval=15s

# Wait for pods to be ready
kubectl get pods -n monitoring -w

# Access Grafana (port-forward)
kubectl port-forward service/prometheus-grafana 3000:80 -n monitoring
# Open: http://localhost:3000
# Login: admin / admin123

# Access Prometheus directly
kubectl port-forward service/prometheus-kube-prometheus-prometheus 9090:9090 -n monitoring
# Open: http://localhost:9090
```

---

## 6. PromQL — The Query Language

PromQL is how you ask Prometheus questions about your metrics.

### Basic Queries

```promql
# ── Simple metric value ───────────────────────────────────────────────────────
http_requests_total                              # all time series for this metric
http_requests_total{status="200"}                # filter by label
http_requests_total{status=~"2.."}               # regex: any 2xx status
http_requests_total{status!="200"}               # not equal
http_requests_total{namespace="production"}      # filter by namespace

# ── Rates (for counters — always use rate() or irate()) ──────────────────────
rate(http_requests_total[5m])                    # per-second rate over last 5 minutes
irate(http_requests_total[5m])                   # instant rate (last two data points)
increase(http_requests_total[1h])                # total increase over 1 hour

# ── Aggregation ───────────────────────────────────────────────────────────────
sum(rate(http_requests_total[5m]))               # total req/s across all pods
sum(rate(http_requests_total[5m])) by (status)   # req/s grouped by status
avg(container_memory_usage_bytes) by (pod)       # avg memory per pod
max(container_cpu_usage_seconds_total)           # highest CPU value

# ── Percentiles from Histograms ───────────────────────────────────────────────
histogram_quantile(0.99,
  rate(http_request_duration_seconds_bucket[5m]))   # p99 latency
histogram_quantile(0.50,
  rate(http_request_duration_seconds_bucket[5m]))   # p50 (median) latency

# ── Arithmetic ────────────────────────────────────────────────────────────────
container_memory_usage_bytes / 1024 / 1024       # bytes → megabytes
(1 - idle_cpu_ratio) * 100                       # CPU utilisation percentage

# ── Comparison (for alerts) ───────────────────────────────────────────────────
http_requests_total{status=~"5.."} > 100         # returns metric if > 100
container_memory_usage_bytes > 500 * 1024 * 1024 # memory > 500MB
```

### Practical Queries for TaskAPI

```promql
# Request rate (req/s)
sum(rate(http_requests_total{namespace="production"}[5m]))

# Error rate (%)
100 * sum(rate(http_requests_total{status=~"5..",namespace="production"}[5m]))
  / sum(rate(http_requests_total{namespace="production"}[5m]))

# p99 API latency
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket{namespace="production"}[5m]))
  by (le))

# Memory usage per pod (MB)
container_memory_usage_bytes{namespace="production"}
  / 1024 / 1024

# CPU usage per pod (cores)
rate(container_cpu_usage_seconds_total{namespace="production"}[5m])

# Pod restarts (liveness probe failures)
increase(kube_pod_container_status_restarts_total{namespace="production"}[1h])

# Active DB connections
db_connections_active{namespace="production"}
```

---

## 7. Grafana — Visualising Metrics

### Key Grafana Concepts

```
Dashboard  → a collection of panels
Panel      → one visualisation (graph, gauge, table, stat)
Data Source → where data comes from (Prometheus, Loki, etc.)
Query      → PromQL query that feeds a panel
Alert      → notification rule triggered when query exceeds threshold
Variable   → dropdown filter applied to all panels ($namespace, $pod)
```

### Creating a Dashboard — Step by Step

```
1. Login to Grafana (admin / your-password)
2. Left sidebar → + → Dashboard
3. Add panel → click "Add visualization"
4. Select datasource: "Prometheus"
5. Enter PromQL query
6. Choose visualisation type (Time series, Gauge, Stat, etc.)
7. Add title, unit, thresholds
8. Save panel, repeat for each metric
9. Save dashboard with a name
```

### Essential Dashboard for TaskAPI

```yaml
# Panels to create (paste these PromQL queries):

Panel 1: Request Rate (req/s) — Time series
  Query: sum(rate(http_requests_total{namespace="$namespace"}[5m])) by (pod)
  Unit: requests/sec

Panel 2: Error Rate (%) — Gauge
  Query: 100 * sum(rate(http_requests_total{status=~"5..",namespace="$namespace"}[5m]))
           / sum(rate(http_requests_total{namespace="$namespace"}[5m]))
  Threshold: green<1, yellow<5, red>=5

Panel 3: p99 Latency (ms) — Time series
  Query: histogram_quantile(0.99,
           sum(rate(http_request_duration_seconds_bucket{namespace="$namespace"}[5m])) by (le))
           * 1000
  Unit: milliseconds

Panel 4: Pod Memory Usage (MB) — Time series
  Query: container_memory_usage_bytes{namespace="$namespace", container!=""}
           / 1024 / 1024
  Unit: megabytes

Panel 5: Pod CPU Usage (cores) — Time series
  Query: rate(container_cpu_usage_seconds_total{namespace="$namespace"}[5m])
  Unit: cores

Panel 6: Pod Restarts — Stat
  Query: increase(kube_pod_container_status_restarts_total{namespace="$namespace"}[1h])
  Unit: restarts/hour, threshold: 0=green, 1=yellow, 3=red

Panel 7: Running Pods — Stat
  Query: count(kube_pod_status_ready{namespace="$namespace", condition="true"})
  Unit: pods
```

---

## 8. AlertManager — Automated Alerting

### PrometheusRule — Define Alerts in Kubernetes

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: taskapi-alerts
  namespace: production
  labels:
    # Must match Prometheus operator's ruleSelector
    release: prometheus    # matches kube-prometheus-stack release name
spec:
  groups:
    - name: taskapi.rules
      rules:

        # ── High Error Rate ──────────────────────────────────────────────────
        - alert: HighErrorRate
          expr: |
            100 * sum(rate(http_requests_total{status=~"5..",namespace="production"}[5m]))
                / sum(rate(http_requests_total{namespace="production"}[5m])) > 5
          for: 2m              # must be true for 2 minutes before alerting
          labels:
            severity: critical
            team: backend
          annotations:
            summary: "High error rate on TaskAPI"
            description: "Error rate is {{ $value | humanizePercentage }} (threshold: 5%)"
            runbook: "https://wiki.company.com/runbooks/high-error-rate"

        # ── High Latency ─────────────────────────────────────────────────────
        - alert: HighP99Latency
          expr: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_seconds_bucket{namespace="production"}[5m]))
              by (le)) > 2
          for: 5m
          labels:
            severity: warning
            team: backend
          annotations:
            summary: "p99 latency above 2 seconds"
            description: "p99 latency is {{ $value | humanizeDuration }}"

        # ── Pod Crash Looping ────────────────────────────────────────────────
        - alert: PodCrashLooping
          expr: |
            increase(kube_pod_container_status_restarts_total{namespace="production"}[1h]) > 5
          for: 0m              # alert immediately (no `for` grace period)
          labels:
            severity: critical
          annotations:
            summary: "Pod {{ $labels.pod }} is crash looping"
            description: "{{ $labels.pod }} restarted {{ $value }} times in the last hour"

        # ── High Memory Usage ────────────────────────────────────────────────
        - alert: HighMemoryUsage
          expr: |
            container_memory_usage_bytes{namespace="production"}
              / container_spec_memory_limit_bytes{namespace="production"} > 0.85
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.pod }} memory above 85%"
            description: "Memory usage is {{ $value | humanizePercentage }} of limit"

        # ── No Pods Running ───────────────────────────────────────────────────
        - alert: NoRunningPods
          expr: |
            count(kube_pod_status_ready{namespace="production",condition="true"}) < 1
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "No running pods in production"
            description: "All pods are down!"
```

### AlertManager Configuration (routing alerts)

```yaml
# alertmanager-config.yaml
global:
  resolve_timeout: 5m
  slack_api_url: "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK"

route:
  group_by: ['alertname', 'namespace']
  group_wait: 10s         # wait before sending first alert (group similar alerts)
  group_interval: 5m      # min time between sending groups
  repeat_interval: 1h     # resend if alert still firing after this time
  receiver: 'team-slack'

  routes:
    # Critical alerts → PagerDuty (wake someone up)
    - match:
        severity: critical
      receiver: 'pagerduty'
      continue: true       # also send to slack

    # Warning alerts → Slack only
    - match:
        severity: warning
      receiver: 'team-slack'

receivers:
  - name: 'team-slack'
    slack_configs:
      - channel: '#alerts-production'
        title: '{{ .GroupLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          *Severity:* {{ .Labels.severity }}
          {{ end }}
        send_resolved: true

  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: $PAGERDUTY_SERVICE_KEY
        description: '{{ .GroupLabels.alertname }}: {{ .CommonAnnotations.summary }}'

inhibit_rules:
  # If "NoRunningPods" fires, suppress "HighErrorRate" (expected)
  - source_match:
      alertname: NoRunningPods
    target_match:
      alertname: HighErrorRate
    equal: ['namespace']
```

---

## 9. Logging in Kubernetes

### The Twelve-Factor App Logging Rule

> "A twelve-factor app never concerns itself with routing or storage of its output stream. It should not attempt to write to or manage logfiles. Instead, each running process writes its event stream, unbuffered, to stdout."

**In Kubernetes terms:** your app writes to stdout/stderr, kubectl and the logging infrastructure handle the rest.

### Structured Logging — JSON Format

```javascript
// Instead of:
console.log('User login failed for email: ' + email);

// Use structured JSON logging:
const log = {
  timestamp: new Date().toISOString(),
  level: 'error',
  message: 'User login failed',
  service: 'taskapi',
  pod: process.env.POD_NAME,
  namespace: process.env.POD_NAMESPACE,
  email: email,           // searchable field
  ip: req.ip,
  requestId: req.id
};
console.log(JSON.stringify(log));

// Output:
// {"timestamp":"2025-01-15T10:43:22Z","level":"error","message":"User login failed",
//  "service":"taskapi","pod":"taskapi-abc-123","namespace":"production",
//  "email":"user@example.com","ip":"10.0.1.5","requestId":"req-456"}
```

### Winston Logger (production-grade)

```javascript
// logger.js
const winston = require('winston');

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()   // JSON to stdout
  ),
  defaultMeta: {
    service: 'taskapi',
    pod: process.env.POD_NAME,
    namespace: process.env.POD_NAMESPACE,
    version: process.env.APP_VERSION
  },
  transports: [
    new winston.transports.Console()  // ONLY stdout — let K8s handle the rest
  ]
});

module.exports = logger;

// Usage:
logger.info('Task created', { taskId: task.id, userId: req.user.id });
logger.error('Database error', { error: err.message, stack: err.stack, query: 'SELECT *' });
logger.warn('High memory usage', { heapUsedMB: 450 });
```

### Inject Pod Metadata via Downward API

```yaml
# In deployment.yaml — expose pod info as env vars
env:
  - name: POD_NAME
    valueFrom:
      fieldRef:
        fieldPath: metadata.name
  - name: POD_NAMESPACE
    valueFrom:
      fieldRef:
        fieldPath: metadata.namespace
  - name: NODE_NAME
    valueFrom:
      fieldRef:
        fieldPath: spec.nodeName
  - name: APP_VERSION
    value: "2.1.0"
```

---

## 10. kubectl logs — Day-to-Day Log Access

```bash
# ── Basic log access ──────────────────────────────────────────────────────────
kubectl logs <pod-name>                         # all logs from pod
kubectl logs <pod-name> -n production           # specific namespace
kubectl logs <pod-name> -f                      # follow (live tail)
kubectl logs <pod-name> --tail=100              # last 100 lines
kubectl logs <pod-name> --since=1h              # last 1 hour
kubectl logs <pod-name> --since-time="2025-01-15T10:00:00Z"  # since time
kubectl logs <pod-name> --timestamps            # show timestamps
kubectl logs <pod-name> -c <container>          # specific container in pod
kubectl logs <pod-name> --previous              # previous (crashed) container

# ── Multi-pod log aggregation ─────────────────────────────────────────────────
kubectl logs -l app=taskapi -n production                          # all pods with label
kubectl logs -l app=taskapi -n production -f                       # follow all
kubectl logs -l app=taskapi -n production --all-containers=true    # all containers

# ── Searching logs ────────────────────────────────────────────────────────────
kubectl logs <pod-name> | grep ERROR
kubectl logs <pod-name> | grep -E "ERROR|WARN"
kubectl logs <pod-name> | jq 'select(.level == "error")'           # JSON logs
kubectl logs <pod-name> | jq 'select(.message | contains("DB"))'   # search field
kubectl logs -l app=taskapi | grep "requestId.*req-456"            # trace request

# ── Stern — better multi-pod log tailing (install separately) ─────────────────
# brew install stern
stern taskapi -n production                     # all pods matching "taskapi"
stern taskapi -n production --since 10m         # last 10 minutes
stern taskapi -n production --container api     # specific container
stern . -n production                           # ALL pods in namespace
```

---

## 11. Loki — Log Aggregation at Scale

For production you need centralised log storage — `kubectl logs` only works for recent logs and is limited.

```
kubectl logs → reads from node's local file (/var/log/containers/)
              → only available while pod exists
              → no cross-pod search
              → no historical data

Loki → aggregates logs from all pods centrally
     → persistent storage (days/weeks/months)
     → fast full-text search with LogQL
     → integrates with Grafana (same dashboards as metrics)
     → cheaper than Elasticsearch (indexes only labels, not full text)
```

### Install Loki Stack with Helm

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install Loki + Promtail (log collector agent)
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set grafana.enabled=false \    # use existing Grafana
  --set prometheus.enabled=false   # use existing Prometheus
```

### Promtail — The Log Collector DaemonSet

Promtail runs as a DaemonSet (one per node), reads container logs from the node's filesystem, and ships them to Loki.

```yaml
# Promtail is configured to:
# 1. Watch /var/log/pods/* on each node
# 2. Parse container log files
# 3. Add Kubernetes labels (pod, namespace, container, node)
# 4. Ship to Loki
```

### LogQL — Querying Logs in Grafana

```logql
# ── Basic filtering ───────────────────────────────────────────────────────────
{app="taskapi", namespace="production"}                # all logs from taskapi
{app="taskapi"} |= "ERROR"                             # lines containing ERROR
{app="taskapi"} |= "ERROR" |! "healthcheck"            # ERROR but not healthcheck
{app="taskapi"} |~ "DB.*error"                         # regex match

# ── JSON parsing (for structured logs) ───────────────────────────────────────
{app="taskapi"} | json                                 # parse JSON fields
{app="taskapi"} | json | level="error"                 # filter by parsed field
{app="taskapi"} | json | line_format "{{.message}}"    # reformat output

# ── Rate / aggregation ────────────────────────────────────────────────────────
rate({app="taskapi"} |= "ERROR" [5m])                  # error log rate per second
count_over_time({app="taskapi"} |= "ERROR" [1h])       # errors in last hour
sum(rate({namespace="production"}[5m])) by (app)       # log rate per service
```

---

## 12. Key Metrics to Monitor for Any K8s App

```
GOLDEN SIGNALS (the four most important metrics for any service):
1. Latency    — how long requests take (p50, p95, p99)
2. Traffic    — how much load (req/s)
3. Errors     — failure rate (5xx %)
4. Saturation — how close to limit (CPU %, memory %)

KUBERNETES INFRASTRUCTURE:
- Node CPU/memory utilisation (alert at 80%)
- Pod restart count (alert at > 3 in 1 hour)
- PVC usage (alert at 80% full)
- Pod pending count (stuck scheduling)

DATABASE (PostgreSQL):
- Active connections vs max_connections
- Query latency (p99)
- Transaction rate
- Replication lag (for replicas)

REDIS:
- Memory usage vs maxmemory
- Connected clients
- Commands/second
- Cache hit rate (hits / (hits + misses))
```

---

## 13. Complete Observability Setup — kubectl Commands

```bash
# ── Check what metrics are available ─────────────────────────────────────────
kubectl port-forward pod/<taskapi-pod> 8080:3000 -n production &
curl http://localhost:8080/metrics

# ── Check Prometheus targets (is your app being scraped?) ─────────────────────
kubectl port-forward service/prometheus-kube-prometheus-prometheus 9090:9090 -n monitoring &
# Open http://localhost:9090/targets — should see your pods listed

# ── Access Grafana ─────────────────────────────────────────────────────────────
kubectl port-forward service/prometheus-grafana 3000:80 -n monitoring &
# Open http://localhost:3000

# ── Check AlertManager (are alerts firing?) ───────────────────────────────────
kubectl port-forward service/prometheus-kube-prometheus-alertmanager 9093:9093 -n monitoring &
# Open http://localhost:9093

# ── View cluster-level metrics ────────────────────────────────────────────────
kubectl top nodes                        # node CPU and memory
kubectl top pods -n production           # pod CPU and memory
kubectl top pods -n production --sort-by=memory  # sort by memory

# ── Check events (important for debugging) ────────────────────────────────────
kubectl get events -n production --sort-by='.lastTimestamp'
kubectl get events -n production --field-selector reason=OOMKilling
kubectl get events -n production --field-selector reason=BackOff
```

---

## 14. Hands-On Practice Tasks

```bash
# 1. Install kube-prometheus-stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set grafana.adminPassword=admin123

# 2. Wait for all pods
kubectl get pods -n monitoring -w

# 3. Access Prometheus and explore
kubectl port-forward svc/prometheus-kube-prometheus-prometheus 9090:9090 -n monitoring &
# Go to http://localhost:9090
# Try: up (which targets are up)
# Try: container_memory_usage_bytes{namespace="monitoring"}
# Try: rate(container_cpu_usage_seconds_total[5m])

# 4. Access Grafana
kubectl port-forward svc/prometheus-grafana 3000:80 -n monitoring &
# Go to http://localhost:3000 (admin/admin123)
# Explore pre-built dashboards:
# → "Kubernetes / Compute Resources / Namespace (Pods)"
# → "Kubernetes / Nodes"

# 5. Add metrics to your Node.js app (Section 4 code)
# Rebuild and push image, update deployment

# 6. Add annotations to Deployment
# prometheus.io/scrape: "true"
# prometheus.io/port: "3000"

# 7. Verify your app appears in Prometheus targets
# http://localhost:9090/targets

# 8. Create a simple PrometheusRule alert
kubectl apply -f prometheus-rule.yaml -n production

# 9. Check the alert appears in Prometheus
# http://localhost:9090/alerts

# 10. View logs from all production pods
kubectl logs -l app=taskapi -n production --tail=50 --timestamps
```

---

## 15. Quiz — Test Yourself / Test Others

1. What are the three pillars of observability and what does each measure?
2. What is the difference between a Counter and a Gauge metric type?
3. Why does Prometheus use a pull model instead of having apps push metrics to it?
4. What annotations must a Pod have for Prometheus to scrape it?
5. Why should you always use `rate()` instead of raw counter values in PromQL?
6. What is the difference between `kubectl logs` and a centralised logging system like Loki?
7. Why should your app log to stdout only, not to a file?
8. What are the four Golden Signals for monitoring any service?
9. What does the `for: 2m` field in a PrometheusRule alert mean?
10. A pod is OOMKilled repeatedly. Which PromQL query would help you understand the memory trend before the kill?

<details>
<summary>Answers</summary>

1. Metrics: numerical measurements over time — CPU usage, request rates, latency percentiles, error counts. Used for dashboards, trending, and alerting (Prometheus + Grafana). Logs: discrete timestamped events from the application — error messages, access logs, debug traces. Used for debugging specific incidents (kubectl logs, Loki). Traces: end-to-end request path across multiple services — shows where time was spent in a distributed request (Jaeger, OpenTelemetry).

2. A Counter is a monotonically increasing value that only goes up — it counts events like total requests, total errors, bytes sent. It resets to zero when the process restarts. A Gauge is a value that can go up or down — it represents current state like memory usage, active connections, queue depth. Use rate() with counters, direct value with gauges.

3. The pull model gives Prometheus control — it decides when and how often to scrape, making it easier to detect if a target goes down (no scrape = target is unavailable), easier to rate-limit scraping, and simpler for targets (they just expose an HTTP endpoint, no outbound firewall rules needed). Push would require apps to know Prometheus's address, handle retries, and Prometheus would have no way to detect "silence = down" vs "silence = not pushing."

4. Three annotations: `prometheus.io/scrape: "true"` (enables scraping), `prometheus.io/port: "3000"` (which port to scrape on), and optionally `prometheus.io/path: "/metrics"` (the path — defaults to /metrics). These annotations are detected by the Prometheus ServiceMonitor or PodMonitor via Kubernetes service discovery.

5. Counters always increase — a raw counter value (e.g., 45231 requests) is not useful for dashboards. `rate()` calculates the per-second rate of increase over a time window, giving you meaningful values like "12 req/s." Without rate(), you'd see a line that only ever goes up and you can't see actual request patterns. Also, rate() correctly handles counter resets on pod restarts.

6. `kubectl logs` reads from the node's local log files for currently running pods — limited to recent logs, unavailable after pod deletion, can't search across pods, no historical data, no persistence. Centralised logging (Loki, Elasticsearch) collects logs from all pods via a DaemonSet agent, stores them persistently for weeks/months, provides full-text search across all services and namespaces, and retains logs after pods are deleted — essential for post-incident analysis.

7. Writing to files inside a container creates stale logs that are hard to access (need kubectl exec), lost when the container is rebuilt, not rotated automatically (disk fills up), and not accessible to log aggregation agents. stdout/stderr is automatically captured by the container runtime and written to node-level log files that DaemonSet log collectors (Promtail, Fluentd) can read, rotate, and ship to centralised storage.

8. The Four Golden Signals: Latency (how long requests take — especially p99), Traffic (how much load the system handles — requests/second), Errors (rate of failed requests — 5xx percentage), Saturation (how "full" the service is — CPU%, memory%, queue depth). These four together tell you almost everything you need to know about whether a service is healthy.

9. `for: 2m` means the alerting condition must be continuously true for 2 minutes before the alert transitions from "Pending" to "Firing" and a notification is sent. This prevents flapping — brief spikes that resolve quickly don't generate alerts. Without `for`, every brief spike triggers an alert. Use short or no `for` for critical alerts (pod down), longer `for` for trend-based alerts (high latency).

10. `container_memory_usage_bytes{namespace="production", pod=~"<pod-pattern>"}` to see the raw memory time series and watch it climb toward the limit. Also useful: `container_memory_usage_bytes / container_spec_memory_limit_bytes` to see percentage of limit, and `increase(kube_pod_container_status_restarts_total{namespace="production"}[1h])` to confirm restart count. The trend graph shows when memory usage accelerated — which correlates with the code path or traffic pattern that caused the OOM.

</details>

---

## 16. Summary

After today you can explain and implement:
- **Three observability pillars** — metrics, logs, traces and what each answers
- **Prometheus data model** — metric types (counter, gauge, histogram, summary), labels, time-series
- **Pull model scraping** — how Prometheus discovers and scrapes Pods via annotations
- **prom-client** — exposing custom metrics from Node.js (counters, histograms, gauges)
- **PromQL** — rate(), histogram_quantile(), aggregations, filtering, practical queries
- **kube-prometheus-stack** — one Helm install for Prometheus + Grafana + AlertManager
- **Grafana dashboards** — Golden Signals panels, variables, thresholds
- **PrometheusRule** — defining alerts with expr, for, severity, annotations
- **AlertManager** — routing alerts to Slack/PagerDuty, grouping, inhibit rules
- **Structured JSON logging** — stdout only, Winston logger, Downward API for pod metadata
- **kubectl logs** — all flags, multi-pod tailing, JSON parsing with jq, stern
- **Loki + LogQL** — centralised log aggregation, filtering, JSON parsing, rate queries

**Coming up on Day 27:** CI/CD with GitHub Actions — deploying to Kubernetes automatically on every push, including image build, scan, push to ECR, run migrations, and Helm upgrade with automatic rollback on failure.
