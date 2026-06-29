# Day 15: Services — Stable Networking, Load Balancing & Service Discovery

**Goal for today:** Understand why Pods need Services, every Service type (ClusterIP, NodePort, LoadBalancer, ExternalName, Headless), how kube-proxy implements them, how Kubernetes DNS enables service discovery, and how to wire a complete multi-tier application together using Services. By the end, you should be able to design and implement the full networking layer for any Kubernetes application.

**Format:** WHAT → WHY → HOW for every concept.

---

## 0. Quick Recap (open your teaching session with this)

- Day 13: Pods — shared network unit, health probes, multi-container patterns
- Day 14: Deployments — rolling updates, self-healing, scaling, rollback

**Today's connecting thought:**
> Deployments give you a reliable way to run Pods. But Pods have two fatal networking problems: their IPs change every time they restart, and you have 3 identical Pods with 3 different IPs — how does anything know which one to talk to? Services solve both problems. A Service is the stable, permanent networking address in front of a set of Pods. This is the last piece needed to run a real application in Kubernetes.

---

## 1. The Problem — Why Pods Need Services

### Problem 1: Pod IPs are ephemeral

```
Day 1:
  api-pod-1: 172.17.0.4   ← your frontend connects to this IP
  api-pod-2: 172.17.0.5
  api-pod-3: 172.17.0.6

Day 2: (pod-1 crashed and was replaced)
  api-pod-1: 172.17.0.9   ← IP CHANGED — frontend connection is broken
  api-pod-2: 172.17.0.5
  api-pod-3: 172.17.0.6
```

Every time a Pod is replaced (crash, rolling update, scaling), it gets a NEW IP. Any hardcoded IP in your app breaks immediately.

### Problem 2: Which of the 3 Pods do you talk to?

```
You have 3 replicas of your API:
  api-pod-1: 172.17.0.4
  api-pod-2: 172.17.0.5
  api-pod-3: 172.17.0.6

Which one should your frontend call?
All three? Pick one? Random?
How do you load balance across them?
```

### The Solution — Kubernetes Service

```
Before Services:                    After Services:
                                    
frontend → 172.17.0.4 (breaks)     frontend → api-service:3000 (permanent)
                                                     │
                                              Load Balancer
                                           ┌──────┼──────┐
                                           ▼      ▼      ▼
                                        pod-1  pod-2  pod-3
                                     (IPs don't matter anymore)
```

**Teaching line:**
> "A Service is like a phone number for a department at a company. You call the Sales department — you don't care which salesperson answers. The phone system (Service) routes your call to any available person (Pod). If someone leaves (Pod deleted), the number still works. If more people join (scale up), calls are distributed to them too."

---

## 2. What is a Kubernetes Service?

### WHAT
A **Service** is a Kubernetes object that:
1. Provides a **stable virtual IP** (ClusterIP) and DNS name that never changes
2. **Selects** a set of Pods using label selectors
3. **Load balances** traffic across all selected healthy Pods
4. Tracks which Pods are healthy via **Endpoints** objects

### HOW Services connect to Pods

Services use the same label selector mechanism as Deployments:

```yaml
# Deployment creates pods with label: app=taskapi
template:
  metadata:
    labels:
      app: taskapi      # ← Pods get this label

---
# Service selects pods with label: app=taskapi
spec:
  selector:
    app: taskapi        # ← Service routes to these Pods
```

```bash
# See which pod IPs a service is routing to
kubectl get endpoints taskapi-service

# Output:
# NAME              ENDPOINTS                                      AGE
# taskapi-service   172.17.0.4:3000,172.17.0.5:3000,172.17.0.6:3000   5m
```

The **Endpoints** object is automatically maintained by Kubernetes — when a Pod starts and passes its readiness probe, its IP is added. When a Pod is deleted or fails readiness, its IP is removed.

---

## 3. Service Types — The Four Types You Must Know

```
ClusterIP      → internal only (within cluster)
NodePort       → external via node IP + port
LoadBalancer   → external via cloud load balancer
ExternalName   → alias for external DNS name
Headless       → no ClusterIP, returns pod IPs directly
```

---

### 3.1 ClusterIP — Internal Service (Default)

#### WHAT
A **ClusterIP** Service assigns a stable virtual IP that is only reachable from **within the cluster**. This is the default Service type.

#### WHY
For internal communication between services — your API talking to your database, your frontend talking to your API. Internal traffic should never leave the cluster.

#### HOW

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: default
spec:
  type: ClusterIP          # default — can omit this line
  selector:
    app: postgres          # routes to pods with this label
  ports:
    - name: postgres
      port: 5432           # port the Service listens on (what others call)
      targetPort: 5432     # port the Pod listens on (where traffic is forwarded)
      protocol: TCP
```

```bash
kubectl apply -f service.yaml
kubectl get services

# Output:
# NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
# postgres-service   ClusterIP   10.96.100.45    <none>        5432/TCP   1m

# The CLUSTER-IP (10.96.100.45) is stable — it never changes
# EXTERNAL-IP is <none> — not reachable from outside the cluster
```

**How another Pod in the cluster connects:**
```javascript
// In your Node.js API, connect using the Service NAME (DNS resolves it)
const pool = new Pool({
  host: 'postgres-service',    // ← Service name, not an IP
  port: 5432,
  database: 'taskdb'
});

// Kubernetes DNS: postgres-service → 10.96.100.45
// kube-proxy: 10.96.100.45:5432 → one of the postgres pod IPs
```

**Port vs TargetPort vs NodePort:**
```
port:       what the Service listens on (what callers use)
targetPort: what the container inside the Pod listens on
nodePort:   (NodePort/LoadBalancer only) port on the host node

Caller → Service:port → Pod:targetPort

Example:
  port: 80          # call the service on port 80
  targetPort: 3000  # forward to container's port 3000
```

---

### 3.2 NodePort — External via Node IP

#### WHAT
A **NodePort** Service opens a specific port on **every node** in the cluster. Traffic hitting that port on any node is forwarded to the Service and then to the Pods.

#### WHY
Exposes a service externally without a cloud load balancer — useful for:
- Local development (Minikube)
- On-premises clusters without cloud LB
- Quick testing/demos

#### HOW

```yaml
apiVersion: v1
kind: Service
metadata:
  name: taskapi-nodeport
spec:
  type: NodePort
  selector:
    app: taskapi
  ports:
    - port: 80             # ClusterIP port (internal)
      targetPort: 3000     # Pod port
      nodePort: 30080      # port opened on EVERY node (30000-32767 range)
                           # omit to let K8s auto-assign in that range
```

```bash
kubectl apply -f service.yaml
kubectl get services

# NAME                TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
# taskapi-nodeport    NodePort   10.96.200.15   <none>        80:30080/TCP   1m

# Access from outside the cluster:
# http://<any-node-IP>:30080

# In Minikube:
minikube service taskapi-nodeport --url
# → http://192.168.49.2:30080
```

**How NodePort works:**
```
External request → Node IP:30080
                        │
                   kube-proxy
                        │
                   Service ClusterIP:80
                        │
                 ┌──────┴──────┐
                 ▼             ▼
              Pod:3000      Pod:3000
```

**Limitations of NodePort:**
- Must use high ports (30000-32767) — can't use port 80/443 directly
- Exposes a port on ALL nodes even if only some run the service
- No built-in SSL termination
- Not suitable for production at scale — use LoadBalancer instead

---

### 3.3 LoadBalancer — External via Cloud Load Balancer

#### WHAT
A **LoadBalancer** Service provisions an **external cloud load balancer** (AWS ELB/ALB, GCP Load Balancer, Azure LB) automatically and points it at your Service. This is the standard way to expose services externally in cloud environments.

#### WHY
- Gets a proper external IP/DNS name (not a high-numbered port)
- Traffic enters the cloud LB → NodePort → ClusterIP → Pod (all automatic)
- Integrates with cloud health checks
- Works with AWS EKS, GKE, AKS out of the box

#### HOW

```yaml
apiVersion: v1
kind: Service
metadata:
  name: taskapi-lb
  annotations:
    # AWS-specific: creates an NLB instead of Classic ELB
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
spec:
  type: LoadBalancer
  selector:
    app: taskapi
  ports:
    - port: 80           # external port on the load balancer
      targetPort: 3000   # pod port
      protocol: TCP
```

```bash
kubectl apply -f service.yaml

# Watch for EXTERNAL-IP (takes ~1-2 min for AWS to provision ELB)
kubectl get services -w

# NAME           TYPE           CLUSTER-IP      EXTERNAL-IP                          PORT(S)
# taskapi-lb     LoadBalancer   10.96.50.100    a1b2.ap-south-1.elb.amazonaws.com   80:31234/TCP

# Access at: http://a1b2.ap-south-1.elb.amazonaws.com
```

**LoadBalancer in Minikube (for local testing):**
```bash
# Minikube doesn't have a real cloud — use this to simulate
minikube tunnel    # run in a separate terminal — creates a route to LoadBalancer IPs
kubectl get service taskapi-lb    # EXTERNAL-IP will show a real IP
```

**Traffic path on AWS EKS:**
```
Internet → AWS ELB (DNS name)
               │
               ▼
         NodePort on any EC2 node
               │
               ▼
         kube-proxy → ClusterIP
               │
               ▼
         ┌─────┴─────┐
         ▼           ▼
      Pod:3000    Pod:3000
```

**Important:** Every LoadBalancer Service creates a SEPARATE cloud load balancer. 10 LoadBalancer services = 10 AWS ELBs = 10x the cost. In practice, you use ONE LoadBalancer + Ingress controller for routing (Day 19) instead of multiple LoadBalancer services.

---

### 3.4 ExternalName — DNS Alias for External Services

#### WHAT
An **ExternalName** Service maps a Kubernetes Service name to an external DNS name — no proxying, just a DNS CNAME alias.

#### WHY
Useful for referencing external services (RDS databases, SaaS APIs) by a Kubernetes-style name inside the cluster — without coupling your app code to external hostnames.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: database              # how your app refers to it inside K8s
  namespace: production
spec:
  type: ExternalName
  externalName: mydb.abc123.ap-south-1.rds.amazonaws.com  # real external DNS
```

```javascript
// Your app code connects to 'database' — clean, not coupled to RDS hostname
const pool = new Pool({ host: 'database', port: 5432 });
// K8s DNS resolves 'database' → CNAME → mydb.abc123.ap-south-1.rds.amazonaws.com
```

This lets you swap the real backend (change the `externalName`) without changing app code.

---

### 3.5 Headless Service — Direct Pod IP Access

#### WHAT
A **Headless** Service has no ClusterIP (`clusterIP: None`). Instead of returning a single virtual IP, DNS returns the **IP of each individual Pod** directly.

#### WHY
Used for stateful applications where each Pod has a unique identity and needs to be addressed individually:
- Databases (PostgreSQL primary vs replicas)
- StatefulSets where Pods have stable hostnames (Day 22)
- Service meshes that need direct Pod-to-Pod routing

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None          # ← this makes it headless
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

```bash
# Normal service DNS: returns one ClusterIP
# postgres-service → 10.96.100.45 (virtual IP, kube-proxy load balances)

# Headless service DNS: returns all pod IPs
# postgres-headless → 172.17.0.4, 172.17.0.5, 172.17.0.6
# Client does its own load balancing or picks a specific IP

# Inside a pod, check the DNS response:
kubectl exec -it my-pod -- nslookup postgres-headless
# Returns: 172.17.0.4, 172.17.0.5, 172.17.0.6
```

---

## 4. Kubernetes DNS — How Service Discovery Works

### WHAT
Every Kubernetes cluster runs **CoreDNS** — a DNS server in the `kube-system` namespace. Every Service gets a DNS name automatically, and all Pods in the cluster can resolve these names.

### WHY
This is what makes `host: 'postgres-service'` work in your app — no hardcoded IPs, no service registry, no extra configuration. Kubernetes DNS is the service discovery mechanism.

### HOW — DNS Name Format

```
Full DNS name:
<service-name>.<namespace>.svc.cluster.local

Examples:
postgres-service.default.svc.cluster.local     → postgres in default namespace
postgres-service.production.svc.cluster.local  → postgres in production namespace
redis.monitoring.svc.cluster.local             → redis in monitoring namespace
```

**Short names work within the same namespace:**
```
# From a Pod in the 'default' namespace:
postgres-service                               → resolves (same namespace)
postgres-service.default                       → resolves
postgres-service.default.svc.cluster.local    → resolves (full name)

# From a Pod in the 'production' namespace:
postgres-service                               → fails (different namespace)
postgres-service.default                       → resolves (explicit namespace)
```

```bash
# Verify DNS works from inside a pod
kubectl exec -it my-pod -- nslookup postgres-service
kubectl exec -it my-pod -- nslookup postgres-service.default.svc.cluster.local

# Check /etc/resolv.conf inside a pod — shows search domains
kubectl exec -it my-pod -- cat /etc/resolv.conf
# nameserver 10.96.0.10         ← CoreDNS ClusterIP
# search default.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5
```

**The search domains** explain why short names work — the OS tries appending each search domain until one resolves.

---

## 5. How kube-proxy Implements Services

### WHAT
kube-proxy runs on every node and translates Service ClusterIPs into actual Pod IPs using Linux **iptables** (or IPVS) rules.

### HOW — The iptables chain

```
Packet arrives at Service ClusterIP:port
        │
        ▼
iptables PREROUTING chain
        │
        ▼
KUBE-SERVICES chain
        │
        ▼ (matches this Service's ClusterIP)
KUBE-SVC-XXXX chain
        │
        ├── 33% probability → KUBE-SEP-POD1 → DNAT to 172.17.0.4:3000
        ├── 33% probability → KUBE-SEP-POD2 → DNAT to 172.17.0.5:3000
        └── 33% probability → KUBE-SEP-POD3 → DNAT to 172.17.0.6:3000
```

DNAT (Destination NAT) rewrites the destination IP from the Service ClusterIP to a real Pod IP. The Pod never knows it was called via a Service.

**When a Pod is added/removed:** kube-proxy updates the iptables rules on ALL nodes within seconds — the load balancing is always current.

---

## 6. Complete Multi-Tier Application Wiring

This is how you wire together a real application: Nginx → API → PostgreSQL → Redis. All Services + Deployments together.

### Architecture

```
External Traffic
      │
      ▼ (port 80)
┌─────────────────────────────────────────────────────────┐
│  Service: nginx-lb (LoadBalancer)                        │
│  → Deployment: nginx (reverse proxy)                    │
└─────────────────────────────┬───────────────────────────┘
                              │ (proxies to api-service:3000)
                              ▼
┌─────────────────────────────────────────────────────────┐
│  Service: api-service (ClusterIP, port 3000)             │
│  → Deployment: taskapi (3 replicas)                     │
└────────────────┬────────────────────┬───────────────────┘
                 │ (port 5432)         │ (port 6379)
                 ▼                     ▼
┌──────────────────────┐   ┌──────────────────────┐
│ Service: postgres-svc │   │ Service: redis-svc    │
│ (ClusterIP, 5432)     │   │ (ClusterIP, 6379)     │
│ → Deployment: pg      │   │ → Deployment: redis   │
└──────────────────────┘   └──────────────────────┘
```

### Full YAML — All Services

```yaml
# ── PostgreSQL Service (ClusterIP — internal only) ─────────────────────────────
apiVersion: v1
kind: Service
metadata:
  name: postgres-svc
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432

---
# ── Redis Service (ClusterIP — internal only) ───────────────────────────────────
apiVersion: v1
kind: Service
metadata:
  name: redis-svc
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: redis
  ports:
    - port: 6379
      targetPort: 6379

---
# ── TaskAPI Service (ClusterIP — internal, nginx talks to this) ─────────────────
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: taskapi
  ports:
    - name: http
      port: 3000
      targetPort: 3000

---
# ── Nginx Service (LoadBalancer — external entry point) ─────────────────────────
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
  namespace: default
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - name: http
      port: 80
      targetPort: 80
```

### How the API connects to other services (DNS)

```javascript
// In taskapi server.js — use Service NAMES, not IPs
const pool = new Pool({
  host: 'postgres-svc',     // ← DNS: postgres-svc.default.svc.cluster.local
  port: 5432,
  database: 'taskdb',
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
});

const redis = createClient({
  url: 'redis://redis-svc:6379'  // ← DNS: redis-svc.default.svc.cluster.local
});
```

---

## 7. Service Discovery Patterns — Environment Variables

### WHAT
Besides DNS, Kubernetes also injects environment variables into every Pod for each Service in the same namespace.

```bash
# Inside any pod, check injected env vars
kubectl exec -it my-pod -- env | grep SERVICE

# Variables injected for postgres-svc:
# POSTGRES_SVC_SERVICE_HOST=10.96.100.45
# POSTGRES_SVC_SERVICE_PORT=5432
# POSTGRES_SVC_PORT=tcp://10.96.100.45:5432
# POSTGRES_SVC_PORT_5432_TCP=tcp://10.96.100.45:5432
# POSTGRES_SVC_PORT_5432_TCP_ADDR=10.96.100.45
# POSTGRES_SVC_PORT_5432_TCP_PORT=5432
```

**Limitation:** Environment variables are injected at Pod startup — if a Service is created AFTER a Pod, the Pod won't have the env vars. DNS doesn't have this limitation — always prefer DNS over env vars for service discovery.

---

## 8. Session Affinity — Sticky Sessions

### WHAT
By default, a Service load-balances requests across ALL healthy Pods randomly (or round-robin). With **session affinity**, requests from the same client are always routed to the same Pod.

### WHY
Some applications store session state in memory — a user must always hit the same Pod to get their session. (Better design avoids this by using Redis for sessions, but sometimes you inherit stateful apps.)

```yaml
spec:
  sessionAffinity: ClientIP       # None (default) | ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800       # 3 hours — how long to maintain sticky session
```

---

## 9. Debugging Services — Full Toolkit

```bash
# ── Check Service is created ──────────────────────────────────────────────────
kubectl get service taskapi-service

# ── Check Endpoints (are pods being selected?) ────────────────────────────────
kubectl get endpoints taskapi-service

# If ENDPOINTS shows <none> — the selector doesn't match any pods
# Fix: check selector in Service matches labels on Pods
kubectl get pods --show-labels    # see pod labels
kubectl describe service taskapi-service  # see service selector

# ── Test DNS resolution from inside a pod ────────────────────────────────────
kubectl run test-pod --image=busybox --rm -it --restart=Never -- sh
# Inside the pod:
nslookup taskapi-service
nslookup taskapi-service.default.svc.cluster.local
wget -qO- http://taskapi-service:3000/health

# ── Test direct pod access (bypass service) ──────────────────────────────────
kubectl port-forward pod/taskapi-xxx-yyy 8080:3000
curl http://localhost:8080/health

# ── Test service access ───────────────────────────────────────────────────────
kubectl port-forward service/taskapi-service 8080:3000
curl http://localhost:8080/health

# ── Common problems and fixes ─────────────────────────────────────────────────
```

| Problem | Symptom | Fix |
|---|---|---|
| Selector mismatch | Endpoints shows `<none>` | Match Service selector to Pod labels exactly |
| Wrong port | Connection refused | Check `port` vs `targetPort` vs container's actual port |
| Pod not ready | Pod in Endpoints but traffic fails | Pod failing readiness probe — check `kubectl describe pod` |
| Wrong namespace | DNS not resolving | Use full DNS name: `service.namespace.svc.cluster.local` |
| No pod running | Endpoints empty | Check Deployment — maybe 0 replicas running |

---

## 10. Service vs Ingress — When to Use Which

This is a common confusion. Preview it here (full Ingress coverage on Day 19).

| | Service | Ingress |
|---|---|---|
| **Layer** | L4 (TCP/UDP) | L7 (HTTP/HTTPS) |
| **Routing** | By port only | By hostname + URL path |
| **TLS** | Basic (LoadBalancer) | Full cert management |
| **Cost** | 1 LB per service | 1 LB for all services |
| **Use for** | Internal comms, simple external exposure | Production HTTP/S routing |

```
Without Ingress (expensive):
  app1-service (LoadBalancer) → ELB 1 → pods
  app2-service (LoadBalancer) → ELB 2 → pods
  app3-service (LoadBalancer) → ELB 3 → pods
  (3 load balancers = 3x cost)

With Ingress (efficient):
  Single LoadBalancer → Ingress Controller
    /app1 → app1-service → pods
    /app2 → app2-service → pods
    /app3 → app3-service → pods
  (1 load balancer = 1x cost, unlimited routing)
```

---

## 11. Service Types Summary — Quick Reference Card

```
ClusterIP (default)
  → accessible: within cluster only
  → use for: DB, Redis, internal APIs
  → selector: yes
  → external IP: no

NodePort
  → accessible: via <node-ip>:<nodePort>
  → use for: dev/testing, on-prem clusters
  → selector: yes
  → port range: 30000-32767

LoadBalancer
  → accessible: via cloud LB external IP/DNS
  → use for: production external exposure (without Ingress)
  → selector: yes
  → requires: cloud provider

ExternalName
  → accessible: returns CNAME to external DNS
  → use for: aliasing external services (RDS, etc.)
  → selector: no (no pods)
  → no proxying, pure DNS

Headless (clusterIP: None)
  → accessible: returns pod IPs directly
  → use for: StatefulSets, direct pod addressing
  → selector: yes (returns pod IPs)
  → no virtual IP
```

---

## 12. Hands-On Practice Tasks

```bash
# 1. Deploy PostgreSQL with a ClusterIP Service
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15-alpine
          env:
            - name: POSTGRES_PASSWORD
              value: "secret"
            - name: POSTGRES_DB
              value: "taskdb"
          ports:
            - containerPort: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-svc
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
EOF

# 2. Deploy your TaskAPI with a ClusterIP Service
# 3. Deploy Nginx with a NodePort Service (for local access via Minikube)
# 4. Check endpoints: kubectl get endpoints postgres-svc
# 5. Test DNS from inside a pod:
kubectl run test --image=busybox --rm -it --restart=Never -- \
  nslookup postgres-svc

# 6. Break the selector (change it to app=wronglabel) — watch endpoints go empty
# 7. Fix it — watch endpoints come back
# 8. Port-forward to the service and test it
kubectl port-forward service/postgres-svc 5432:5432 &
psql -h localhost -U postgres -d taskdb

# 9. Scale the API to 5 replicas and watch endpoints update:
kubectl scale deployment taskapi --replicas=5
kubectl get endpoints api-service -w

# 10. In Minikube: expose with NodePort and access via browser
minikube service taskapi-nodeport --url
```

---

## 13. Quiz — Test Yourself / Test Others

1. Why do Pods need Services? Name both problems Services solve.
2. What are the four main Service types and when do you use each?
3. What is the difference between `port`, `targetPort`, and `nodePort` in a Service spec?
4. How does Kubernetes DNS work? What DNS name does a Service called `api-service` in the `production` namespace get?
5. What is an Endpoints object and how is it maintained?
6. What happens to traffic during a rolling update — specifically, how does the Service know to stop sending traffic to terminating pods?
7. When would you use a Headless Service?
8. What is the main downside of using multiple LoadBalancer Services in production?
9. A Service's `kubectl get endpoints` shows `<none>`. What is the most likely cause and how do you fix it?
10. What is the difference between a Service and an Ingress?

<details>
<summary>Answers</summary>

1. Problem 1: Pod IPs are ephemeral — they change every time a Pod restarts. Services provide a stable ClusterIP that never changes. Problem 2: Multiple Pods running the same app need load balancing — clients can't know all Pod IPs. Services provide a single address that automatically load-balances across all healthy Pods.

2. ClusterIP: stable internal IP, only reachable within the cluster — use for databases, Redis, internal APIs. NodePort: opens a port on every node — use for development, on-premises clusters without cloud LB. LoadBalancer: provisions a cloud load balancer with external IP — use for production external exposure. ExternalName: DNS CNAME alias for external hostnames — use to reference external services (RDS etc.) by a K8s-native name.

3. `port` is what the Service listens on — what callers use to connect to the Service. `targetPort` is the port on the Pod container that traffic is forwarded to. `nodePort` (NodePort/LoadBalancer only) is the port opened on every node's external interface for external access (must be 30000-32767).

4. Kubernetes runs CoreDNS which automatically assigns a DNS name to every Service. The full format is `<service-name>.<namespace>.svc.cluster.local`. So `api-service` in `production` gets: `api-service.production.svc.cluster.local`. Within the same namespace, the short name `api-service` also resolves.

5. An Endpoints object is automatically created and maintained by Kubernetes for each Service that has a selector. It contains the list of Pod IPs and ports currently backing that Service. When a Pod passes its readiness probe, its IP is added; when it fails readiness or is deleted, its IP is removed. kube-proxy watches Endpoints to keep iptables rules current.

6. When a Pod enters termination, it is immediately removed from the Service's Endpoints object — kube-proxy updates iptables rules and new connections no longer route to it. The Pod then receives SIGTERM and has `terminationGracePeriodSeconds` to finish in-flight requests. The readiness probe failing (or the Pod being marked for deletion) is the trigger for removal from Endpoints.

7. Headless Services are used when you need to address individual Pods directly rather than load-balancing across them — primarily for StatefulSets where each Pod has a stable identity (e.g., postgres-0, postgres-1) and clients need to connect to a specific one (e.g., connecting to the primary vs replica).

8. Each LoadBalancer Service provisions a separate cloud load balancer instance (e.g., AWS ELB). Multiple services means multiple load balancers, each with its own cost and DNS name. The solution is to use a single LoadBalancer + an Ingress controller, which routes all HTTP/S traffic using a single load balancer with path and hostname-based routing rules.

9. The most likely cause is a label selector mismatch — the Service's `selector` labels don't match the labels on any running Pods. Fix: run `kubectl get pods --show-labels` to see actual pod labels, compare with `kubectl describe service <name>` to see the Service selector, then correct whichever is wrong.

10. A Service operates at L4 (TCP/UDP) and routes based on port only — each Service needs its own load balancer for external access. An Ingress operates at L7 (HTTP/HTTPS) and routes based on hostname and URL path, allowing many services to share a single load balancer. Services handle internal communication; Ingress handles external HTTP routing in production.

</details>

---

## 14. Summary

After today you can explain and implement:
- Why Pods need Services — ephemeral IPs + load balancing across replicas
- All 5 Service types: **ClusterIP** (internal), **NodePort** (external via node), **LoadBalancer** (cloud LB), **ExternalName** (DNS alias), **Headless** (direct pod IPs)
- The `port` / `targetPort` / `nodePort` distinction
- **Kubernetes DNS** — CoreDNS, full DNS name format, short names within namespace, search domains
- How **kube-proxy** implements Services via iptables DNAT rules
- **Endpoints** objects — how Services track healthy Pods
- Full multi-tier application wiring — Nginx → API → PostgreSQL → Redis
- Session affinity for stateful apps
- Full debugging toolkit for Service networking problems
- Service vs Ingress — when to use which (preview for Day 19)

**Coming up on Day 16:** ConfigMaps and Secrets — externalising configuration and managing sensitive data. How to inject environment variables, config files, and secrets into Pods without baking them into images — the production pattern for 12-factor apps in Kubernetes.
