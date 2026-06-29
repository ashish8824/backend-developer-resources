# Day 18: Namespaces & Resource Management — Quotas, Limits & Cluster Structure

**Goal for today:** Understand what namespaces are and why they exist, how to structure a production cluster for multiple teams and environments, how ResourceQuotas prevent one team from consuming all cluster resources, how LimitRanges set default resource constraints on every container, and how all of these combine into a production-grade multi-tenant cluster design. By the end, you should be able to design and configure the resource management layer for any real Kubernetes cluster.

**Format:** WHAT → WHY → HOW for every concept.

---

## 0. Quick Recap (open your teaching session with this)

- Day 15: Services — stable networking, ClusterIP, NodePort, LoadBalancer
- Day 16: ConfigMaps & Secrets — externalising configuration
- Day 17: Storage — PV, PVC, StorageClass, dynamic provisioning

**Today's connecting thought:**
> So far we've been running everything in the `default` namespace. In a real company, multiple teams (backend, frontend, data, platform), multiple environments (dev, staging, prod), and multiple projects share the same Kubernetes cluster. Without proper organisation and resource controls, one team's runaway process can starve every other team. Today we build the isolation and governance layer that makes shared clusters safe and fair.

---

## 1. What Are Namespaces? (Revisited and Deepened)

### WHAT
A **namespace** is a virtual cluster within a cluster — a logical partition that isolates Kubernetes objects (Pods, Services, ConfigMaps, Secrets, PVCs) from objects in other namespaces.

### WHY namespaces exist — the three use cases

**Use case 1: Environment separation**
```
cluster
├── namespace: development    → dev workloads, fewer resources, relaxed policies
├── namespace: staging        → staging workloads, mirrors production config
└── namespace: production     → production workloads, strict policies, more resources
```

**Use case 2: Team separation**
```
cluster
├── namespace: team-backend   → backend team's services
├── namespace: team-frontend  → frontend team's services
├── namespace: team-data      → data engineering pipelines
└── namespace: team-platform  → shared infrastructure (monitoring, ingress, etc.)
```

**Use case 3: Project/application separation**
```
cluster
├── namespace: queuecare      → all QueueCare services
├── namespace: pulsebloom     → all PulseBloom services
└── namespace: shared-infra   → shared monitoring, logging
```

### WHAT namespaces isolate vs what they don't

```
✅ ISOLATED per namespace:
  Pods, Deployments, Services, ConfigMaps, Secrets
  PersistentVolumeClaims, ServiceAccounts
  ResourceQuotas, LimitRanges
  RBAC (Roles, RoleBindings)
  NetworkPolicies

❌ NOT namespace-scoped (cluster-wide):
  Nodes
  PersistentVolumes
  StorageClasses
  ClusterRoles, ClusterRoleBindings
  Namespaces themselves
```

**Teaching line:**
> "Namespaces are like floors in an office building. Each floor (namespace) has its own rooms (Pods), phone system (Services), filing cabinets (ConfigMaps/Secrets), and security badges (RBAC). The building's electricity, elevators, and plumbing (nodes, PVs, StorageClasses) are shared. You can't take another floor's filing cabinet, but you share the same building infrastructure."

---

## 2. Default Namespaces — What Comes Pre-installed

```bash
kubectl get namespaces

# NAME              STATUS   AGE
# default           Active   10d   ← where resources go if you don't specify
# kube-system       Active   10d   ← Kubernetes internal components (API Server, etcd, etc.)
# kube-public       Active   10d   ← publicly readable, rarely used
# kube-node-lease   Active   10d   ← node heartbeat data (lease objects)
```

**Critical rule:** Never deploy your application workloads into `kube-system`. Never delete `kube-system`. It runs the control plane components that keep the cluster alive.

---

## 3. Creating and Working with Namespaces

### Creating namespaces

```yaml
# declarative (preferred)
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    env: production
    team: platform
  annotations:
    contact: "platform-team@company.com"
    purpose: "Production workloads"
```

```bash
# imperative (quick)
kubectl create namespace staging
kubectl create namespace development
kubectl create namespace production

# shorthand
kubectl create ns monitoring
```

### Working across namespaces

```bash
# All kubectl commands are namespace-aware
kubectl get pods                          # default namespace
kubectl get pods -n production            # production namespace
kubectl get pods -n kube-system           # system namespace
kubectl get pods --all-namespaces         # all namespaces
kubectl get pods -A                       # shorthand for --all-namespaces

# Apply resources to a specific namespace
kubectl apply -f deployment.yaml -n production

# Or specify in the YAML itself (preferred)
metadata:
  namespace: production

# Set default namespace for current context (so you don't type -n every time)
kubectl config set-context --current --namespace=production
kubectl config view --minify | grep namespace     # verify

# Switch namespace quickly with kubens
kubens production
```

---

## 4. ResourceQuota — Limiting Total Namespace Consumption

### WHAT
A **ResourceQuota** sets hard limits on the total amount of resources (CPU, memory, storage, object count) that ALL objects in a namespace can consume combined. It's a guardrail for the entire namespace.

### WHY
Without ResourceQuotas:
```
Scenario: One developer in the team-backend namespace accidentally deploys
a Deployment with 1000 replicas each requesting 1 CPU.
→ Consumes 1000 CPUs from the cluster
→ team-frontend Pods can't get scheduled (no CPU left)
→ Cluster-wide outage caused by one mistake
```

With ResourceQuotas:
```
team-backend namespace has quota: cpu: 20, memory: 40Gi
→ The 1000-replica deployment can only use 20 CPUs max
→ Other namespaces are unaffected
→ One team can't starve another
```

### HOW — ResourceQuota YAML

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    # ── Compute Resources ─────────────────────────────────────────────────
    requests.cpu: "20"            # total CPU requests across all pods: max 20 cores
    requests.memory: 40Gi         # total memory requests: max 40Gi
    limits.cpu: "40"              # total CPU limits: max 40 cores
    limits.memory: 80Gi           # total memory limits: max 80Gi

    # ── Storage Resources ─────────────────────────────────────────────────
    requests.storage: 500Gi       # total PVC storage requests: max 500Gi
    persistentvolumeclaims: "10"  # max number of PVCs
    # StorageClass-specific quota:
    fast-ssd.storageclass.storage.k8s.io/requests.storage: 200Gi

    # ── Object Count Limits ───────────────────────────────────────────────
    pods: "50"                    # max number of pods
    services: "20"                # max number of services
    secrets: "30"                 # max number of secrets
    configmaps: "30"              # max number of configmaps
    deployments.apps: "20"        # max number of deployments
    services.loadbalancers: "2"   # max number of LoadBalancer services (expensive!)
    services.nodeports: "0"       # forbid NodePort services entirely
```

```bash
kubectl apply -f quota.yaml

# Check quota usage
kubectl describe resourcequota production-quota -n production

# Output:
# Name:            production-quota
# Namespace:       production
# Resource         Used    Hard
# --------         ----    ----
# limits.cpu       2500m   40
# limits.memory    5Gi     80Gi
# pods             8       50
# requests.cpu     1250m   20
# requests.memory  2560Mi  40Gi
```

### Important: ResourceQuota FORCES resource declarations

When a ResourceQuota with compute resources exists in a namespace, **every Pod must declare resource requests and limits** — or it will be rejected.

```bash
# Without resource requests/limits in a namespace with quota:
kubectl run my-pod --image=nginx -n production

# Error from server (Forbidden): pods "my-pod" is forbidden:
# failed quota: production-quota: must specify limits.cpu for: nginx;
# must specify limits.memory for: nginx;
# must specify requests.cpu for: nginx;
# must specify requests.memory for: nginx
```

This is actually a **useful enforcement mechanism** — it forces developers to think about resource requirements. But it creates a chicken-and-egg problem if developers forget. **LimitRange** (Section 5) solves this by providing defaults.

---

## 5. LimitRange — Default and Max Resources per Container

### WHAT
A **LimitRange** sets default resource requests/limits for containers, and enforces minimum/maximum boundaries per container (or PVC) in a namespace.

### WHY
LimitRange solves two problems:

**Problem 1: ResourceQuota requires declarations but developers forget**
→ LimitRange provides **default** values so containers without explicit resources get sane defaults automatically

**Problem 2: A single container could claim all namespace quota**
→ LimitRange sets **per-container** max limits so no single container can hog everything

### HOW — LimitRange YAML

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: production-limitrange
  namespace: production
spec:
  limits:
    # ── Container-level limits ─────────────────────────────────────────────
    - type: Container
      default:                      # DEFAULT limits (applied if not specified)
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:               # DEFAULT requests (applied if not specified)
        cpu: "250m"
        memory: "256Mi"
      max:                          # MAXIMUM allowed per container
        cpu: "4"
        memory: "8Gi"
      min:                          # MINIMUM required per container
        cpu: "50m"
        memory: "64Mi"
      maxLimitRequestRatio:         # limits / requests ratio max
        cpu: "4"                    # limits.cpu cannot exceed 4x requests.cpu
        memory: "4"

    # ── Pod-level limits ───────────────────────────────────────────────────
    - type: Pod
      max:
        cpu: "8"                    # total CPU across all containers in one pod: max 8
        memory: "16Gi"

    # ── PVC limits ─────────────────────────────────────────────────────────
    - type: PersistentVolumeClaim
      max:
        storage: 50Gi               # max PVC size: 50Gi
      min:
        storage: 1Gi                # min PVC size: 1Gi
```

```bash
kubectl apply -f limitrange.yaml

# Check LimitRange
kubectl describe limitrange production-limitrange -n production

# Now deploy a Pod WITHOUT resource declarations:
kubectl run auto-defaults --image=nginx -n production

# The Pod is created! LimitRange injected the defaults:
kubectl describe pod auto-defaults -n production | grep -A 10 "Limits"
# Limits:
#   cpu:     500m    ← from LimitRange default
#   memory:  512Mi   ← from LimitRange default
# Requests:
#   cpu:     250m    ← from LimitRange defaultRequest
#   memory:  256Mi   ← from LimitRange defaultRequest
```

---

## 6. ResourceQuota + LimitRange Together — The Complete Pattern

These two objects work together as a team:

```
LimitRange: "Every container gets defaults and has boundaries"
ResourceQuota: "The whole namespace has a total budget"

LimitRange ensures every Pod has resource declarations
  → ResourceQuota can then accurately track and enforce the total
```

```
namespace: production
│
├── LimitRange
│   └── default: 250m CPU, 256Mi RAM per container
│   └── max: 4 CPU, 8Gi RAM per container
│
└── ResourceQuota
    └── total: 20 CPU, 40Gi RAM for entire namespace
    └── When all 50 pods run at default: 50 × 250m = 12.5 CPU (fits in 20 CPU quota)
```

**Teaching line:**
> "LimitRange is the speed limit per car on a road. ResourceQuota is the total road capacity. LimitRange prevents any single car from going too fast. ResourceQuota prevents traffic jams by limiting how many cars can be on the road total."

---

## 7. Production Cluster Structure — Real-World Namespace Design

### Pattern 1: Environment-per-namespace (small-medium teams)

```yaml
# Namespaces
development:    # dev team workloads — relaxed quotas
staging:        # pre-prod — mirrors prod config
production:     # prod — strict quotas, RBAC, policies
monitoring:     # Prometheus, Grafana, alerting (shared)
ingress-nginx:  # ingress controller (shared)
cert-manager:   # TLS certificate management (shared)
```

### Pattern 2: Team-per-namespace (larger organisations)

```yaml
# Application team namespaces
team-backend-dev:
team-backend-prod:
team-frontend-dev:
team-frontend-prod:
team-data-dev:
team-data-prod:

# Shared platform namespaces
monitoring:
logging:
ingress-nginx:
cert-manager:
vault:
```

### Complete namespace setup script

```bash
#!/bin/bash
# setup-namespaces.sh — run once to configure a new cluster

# Create namespaces
for ns in development staging production monitoring; do
  kubectl create namespace $ns --dry-run=client -o yaml | kubectl apply -f -
done

# Label namespaces for network policies and monitoring
kubectl label namespace production  env=production  team=platform
kubectl label namespace staging     env=staging     team=platform
kubectl label namespace development env=development team=platform
kubectl label namespace monitoring  team=platform   purpose=monitoring
```

---

## 8. Complete Multi-Namespace Setup — YAML

```yaml
# ── Namespaces ────────────────────────────────────────────────────────────────
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    env: production
---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    env: staging
---
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    env: development

---
# ── Production ResourceQuota ──────────────────────────────────────────────────
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"
    services: "30"
    persistentvolumeclaims: "20"
    secrets: "50"
    configmaps: "50"
    services.loadbalancers: "3"

---
# ── Staging ResourceQuota (smaller) ──────────────────────────────────────────
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota
  namespace: staging
spec:
  hard:
    requests.cpu: "8"
    requests.memory: 16Gi
    limits.cpu: "16"
    limits.memory: 32Gi
    pods: "50"

---
# ── Development ResourceQuota (smallest) ─────────────────────────────────────
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota
  namespace: development
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "30"

---
# ── Production LimitRange ─────────────────────────────────────────────────────
apiVersion: v1
kind: LimitRange
metadata:
  name: limitrange
  namespace: production
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "250m"
        memory: "256Mi"
      max:
        cpu: "4"
        memory: "8Gi"
      min:
        cpu: "50m"
        memory: "64Mi"
    - type: PersistentVolumeClaim
      max:
        storage: 100Gi
      min:
        storage: 1Gi

---
# ── Development LimitRange (more generous per container, lower total) ─────────
apiVersion: v1
kind: LimitRange
metadata:
  name: limitrange
  namespace: development
spec:
  limits:
    - type: Container
      default:
        cpu: "200m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "2"
        memory: "4Gi"
```

---

## 9. Namespace Scoping for All Resources

When deploying to a specific namespace, every resource must reference it:

```yaml
# Option 1: Specify in metadata (most explicit — use in production)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: taskapi
  namespace: production      # ← always specify in production YAML

---
# Option 2: Use -n flag with kubectl apply
kubectl apply -f deployment.yaml -n production

---
# Option 3: Set default namespace in context (dev convenience)
kubectl config set-context --current --namespace=production
# Now: kubectl apply -f deployment.yaml  → goes to production
```

**Cross-namespace Service DNS:**
```javascript
// Same namespace: short name works
const client = await connect({ host: 'postgres-svc' });

// Different namespace: must use full DNS name
// postgres-svc.production.svc.cluster.local
const client = await connect({
  host: 'postgres-svc.production.svc.cluster.local'
});
```

---

## 10. Quality of Service (QoS) Classes

### WHAT
Kubernetes automatically assigns every Pod a **QoS class** based on its resource declarations. This determines which Pods get evicted first when a node runs out of memory.

### Three QoS Classes

**Guaranteed — highest priority, evicted last**
```yaml
# requests == limits for ALL containers
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "500m"      # ← same as request
    memory: "512Mi"  # ← same as request
```

**Burstable — medium priority**
```yaml
# requests < limits (or only limits set)
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "500m"      # ← higher than request
    memory: "512Mi"
```

**BestEffort — lowest priority, evicted first**
```yaml
# NO resource declarations at all
# container has no requests or limits
```

```bash
# Check QoS class of a pod
kubectl describe pod my-pod | grep "QoS Class"
# QoS Class: Burstable
```

**Teaching line:**
> "QoS classes are like airline seating. Guaranteed class (Business/First) passengers stay on the plane when there's overbooking. BestEffort (standby) passengers get bumped first. Burstable (Economy) are somewhere in between. When a node is under memory pressure, Kubernetes bumps BestEffort Pods first to free up memory."

**Production recommendation:**
- Critical services (DB, API): **Guaranteed** — set requests = limits
- Normal services: **Burstable** — set requests < limits for burst capacity
- Non-critical batch jobs: **BestEffort** — only if you accept eviction risk

---

## 11. Node Pressure — What Triggers Eviction

### WHAT
When a node runs low on resources, the **kubelet** evicts Pods to reclaim resources. The order of eviction follows QoS classes (BestEffort first, then Burstable, then Guaranteed).

```bash
# Check node resource pressure
kubectl describe node worker-node-1

# Conditions section:
# MemoryPressure   False   # True when node is low on memory → evicts BestEffort pods
# DiskPressure     False   # True when node is low on disk → evicts pods with most disk usage
# PIDPressure      False   # True when node is low on process IDs
# Ready            True    # False when node itself is unhealthy
```

---

## 12. Namespace Monitoring and Reporting

```bash
# ── See quota usage across all namespaces ─────────────────────────────────────
kubectl get resourcequota -A

# ── Detailed quota for one namespace ─────────────────────────────────────────
kubectl describe resourcequota -n production

# ── See LimitRange in a namespace ────────────────────────────────────────────
kubectl describe limitrange -n production

# ── See resource usage per namespace (requires metrics-server) ────────────────
kubectl top pods -n production
kubectl top pods -A --sort-by=cpu        # sort by CPU across all namespaces
kubectl top pods -A --sort-by=memory     # sort by memory

# ── See all resources in a namespace ─────────────────────────────────────────
kubectl get all -n production

# ── Count objects per namespace ───────────────────────────────────────────────
kubectl get pods -A --no-headers | awk '{print $1}' | sort | uniq -c | sort -rn

# ── Get everything in a namespace (for auditing) ──────────────────────────────
kubectl api-resources --verbs=list --namespaced -o name | \
  xargs -I{} kubectl get {} -n production --ignore-not-found 2>/dev/null
```

---

## 13. Namespace Deletion — The Nuclear Option

```bash
# Delete a namespace — DELETES EVERYTHING inside it!
kubectl delete namespace staging

# This deletes: all Pods, Deployments, Services, ConfigMaps,
# Secrets, PVCs (and their data if reclaimPolicy: Delete!), etc.

# Namespace stuck in "Terminating"?
# (common when finalizers block deletion)
kubectl get namespace staging -o json | \
  jq '.spec.finalizers = []' | \
  kubectl replace --raw "/api/v1/namespaces/staging/finalize" -f -
```

---

## 14. Best Practices — Namespace Design Rules

```
✅ DO:
  - Always specify namespace in production YAML files
  - Apply ResourceQuota to every non-system namespace
  - Apply LimitRange to every non-system namespace
  - Use labels on namespaces (env, team, purpose)
  - Keep kube-system clean — no application workloads
  - Use separate namespaces for monitoring/logging tools
  - Document namespace purpose in annotations

❌ DON'T:
  - Don't deploy applications in the 'default' namespace
  - Don't modify kube-system resources
  - Don't delete kube-system or kube-public
  - Don't create one namespace per microservice (too granular)
  - Don't use namespaces as a substitute for proper RBAC
  - Don't forget to set ResourceQuota — dev mistakes can starve prod
```

---

## 15. Complete Namespace CLI Reference

```bash
# ── Namespace management ──────────────────────────────────────────────────────
kubectl get namespaces
kubectl get ns
kubectl describe namespace production
kubectl create namespace my-ns
kubectl delete namespace my-ns
kubectl label namespace my-ns env=production
kubectl annotate namespace my-ns contact="team@example.com"

# ── Context and namespace switching ──────────────────────────────────────────
kubectl config current-context
kubectl config set-context --current --namespace=production
kubectl config view --minify | grep namespace

# ── ResourceQuota ─────────────────────────────────────────────────────────────
kubectl get resourcequota -n production
kubectl describe resourcequota -n production
kubectl apply -f quota.yaml

# ── LimitRange ────────────────────────────────────────────────────────────────
kubectl get limitrange -n production
kubectl describe limitrange -n production
kubectl apply -f limitrange.yaml

# ── Cross-namespace operations ────────────────────────────────────────────────
kubectl get pods -A                        # all namespaces
kubectl get pods --all-namespaces
kubectl get pods -n kube-system
kubectl top pods -A --sort-by=memory
kubectl delete all --all -n development    # delete everything (not namespace itself)
```

---

## 16. Hands-On Practice Tasks

```bash
# 1. Create three namespaces: development, staging, production
kubectl create ns development
kubectl create ns staging
kubectl create ns production

# 2. Label them
kubectl label ns production env=production
kubectl label ns staging env=staging
kubectl label ns development env=development

# 3. Apply ResourceQuota to each (start with development)
kubectl apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    pods: "10"
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
EOF

# 4. Apply LimitRange to development
kubectl apply -f - <<EOF
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: development
spec:
  limits:
    - type: Container
      default:
        cpu: "200m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "1"
        memory: "2Gi"
EOF

# 5. Try deploying without resource declarations
kubectl run test --image=nginx -n development
kubectl describe pod test -n development | grep -A 6 "Limits"
# Should show auto-injected defaults from LimitRange

# 6. Check quota usage
kubectl describe resourcequota dev-quota -n development

# 7. Try to exceed the quota (create 11 pods when limit is 10)
kubectl run excess --image=nginx --replicas=11 -n development
# Should fail with quota exceeded error

# 8. Set your default namespace to development
kubectl config set-context --current --namespace=development
kubectl get pods   # no -n flag needed now

# 9. Check QoS class of a pod
kubectl describe pod test -n development | grep "QoS Class"

# 10. Reset to default namespace
kubectl config set-context --current --namespace=default
```

---

## 17. Quiz — Test Yourself / Test Others

1. What do namespaces isolate and what do they NOT isolate?
2. What is the difference between ResourceQuota and LimitRange?
3. Why does a ResourceQuota on compute resources force every container to declare resource requests/limits?
4. How does LimitRange solve the chicken-and-egg problem created by ResourceQuota?
5. What are the three QoS classes? Which gets evicted first when a node runs low on memory?
6. How do you achieve **Guaranteed** QoS class for a container?
7. A developer deploys a Pod with no resource declarations to a namespace with a ResourceQuota. What happens?
8. What is the DNS name format for a Service in a different namespace?
9. Why should you avoid deploying application workloads to the `default` namespace in production?
10. Name three things that are cluster-scoped (not namespace-scoped) in Kubernetes.

<details>
<summary>Answers</summary>

1. Namespaces isolate: Pods, Deployments, Services, ConfigMaps, Secrets, PVCs, ServiceAccounts, ResourceQuotas, LimitRanges, RBAC Roles/RoleBindings, NetworkPolicies. NOT isolated (cluster-wide): Nodes, PersistentVolumes, StorageClasses, ClusterRoles/ClusterRoleBindings, and Namespaces themselves.

2. ResourceQuota sets hard limits on the **total** resources consumed by all objects in a namespace combined (e.g., total CPU = 20 cores, total pods = 50). LimitRange sets **per-container** defaults and boundaries (e.g., each container gets default 250m CPU, max 4 CPU). ResourceQuota is the namespace budget; LimitRange is the per-container policy.

3. When a ResourceQuota specifying compute resources (requests.cpu, limits.memory, etc.) exists in a namespace, Kubernetes requires every Pod to have explicit resource requests and limits — otherwise it can't accurately track quota usage. Without declarations, Kubernetes would have no idea how much of the quota a Pod is consuming.

4. LimitRange injects default resource requests and limits onto any container that doesn't declare them. So when ResourceQuota requires declarations, LimitRange provides them automatically — developers can deploy without explicitly writing resource declarations and still satisfy the quota requirement.

5. Guaranteed (requests == limits for all containers) — evicted last. Burstable (requests < limits, or partial declarations) — evicted second. BestEffort (no resource declarations) — evicted first when the node is under memory pressure.

6. Set `requests` exactly equal to `limits` for **every container** in the Pod (including init containers). Both CPU and memory must match. If any container has requests != limits, the Pod is Burstable, not Guaranteed.

7. If the namespace has a ResourceQuota with compute resources, the Pod is **rejected** — the API Server returns a Forbidden error stating the Pod must specify resource limits and requests. If a LimitRange also exists, it would inject defaults and the Pod would be accepted.

8. `<service-name>.<namespace>.svc.cluster.local` — e.g., `postgres-svc.production.svc.cluster.local`. The short name `postgres-svc` only works within the same namespace due to DNS search domain resolution.

9. Using the `default` namespace in production creates several problems: no resource isolation from other workloads or experiments, no quota enforcement unless you add quotas to `default` (affecting all objects), it becomes a garbage dump of mixed resources, harder to apply RBAC, and if you accidentally delete all resources in `default` you affect everything. Production workloads should be in dedicated, well-managed namespaces.

10. Any three of: Nodes, PersistentVolumes, StorageClasses, ClusterRoles, ClusterRoleBindings, Namespaces themselves, CustomResourceDefinitions (CRDs), ValidatingWebhookConfigurations, MutatingWebhookConfigurations, PriorityClasses.

</details>

---

## 18. Summary

After today you can explain and implement:
- **Namespaces** — virtual clusters, what they isolate vs what they don't, three use cases (env, team, project separation)
- The 4 default namespaces and why `kube-system` is sacred
- **ResourceQuota** — total namespace budget (CPU, memory, object counts), how it forces resource declarations
- **LimitRange** — per-container defaults and boundaries, how it solves the ResourceQuota chicken-and-egg problem
- **ResourceQuota + LimitRange together** — the speed limit per car + total road capacity analogy
- **QoS Classes** — Guaranteed (evicted last), Burstable, BestEffort (evicted first)
- Production cluster structure patterns — environment-per-namespace vs team-per-namespace
- Complete multi-namespace setup with graduated quotas (production > staging > development)
- Cross-namespace Service DNS — full name format requirement
- Node pressure and eviction — when and why Pods get evicted
- Namespace best practices and anti-patterns

**Coming up on Day 19:** Ingress — HTTP/S routing, path-based and host-based routing, TLS termination, and the Ingress Controller pattern. This is how you expose multiple services through a single cloud load balancer with proper URL routing — the production alternative to creating one LoadBalancer Service per application.
