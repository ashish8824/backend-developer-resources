# Day 25: Kubernetes Networking Deep Dive — CNI, NetworkPolicies & Zero-Trust

**Goal for today:** Understand how Kubernetes networking actually works under the hood (CNI plugins, Pod-to-Pod communication), how NetworkPolicies act as firewall rules between Pods, and how to implement a zero-trust networking model where services only communicate with what they're explicitly allowed to reach. By the end, you should be able to design and implement the complete network security layer for any production Kubernetes cluster.

**Format:** WHAT → WHY → HOW for every concept.

---

## 0. Quick Recap (open your teaching session with this)

- Day 4: Docker networking — bridge networks, DNS, port mapping
- Day 15: Kubernetes Services — ClusterIP, NodePort, LoadBalancer, kube-proxy

**Today's connecting thought:**
> So far we've used Services to route traffic between Pods, but we've never locked down WHO can talk to WHOM. By default, every Pod in a Kubernetes cluster can reach every other Pod on any port — regardless of namespace. That's a security disaster in production. Today we learn how the network actually works and how to enforce strict communication rules.

---

## 1. Kubernetes Networking Fundamentals — The Four Requirements

Kubernetes networking is built on four core requirements:

```
1. Every Pod gets its own unique IP address
   → No NAT between Pods on the same cluster
   → A Pod on Node 1 can directly reach a Pod on Node 2 using its IP

2. Every Pod can communicate with every other Pod WITHOUT NAT
   → Direct IP routing across nodes
   → 172.16.1.5 (Node 1 pod) → 172.16.2.8 (Node 2 pod) works directly

3. Agents on a node (kubelet, system daemons) can communicate with
   all Pods on that node

4. Every Pod sees its own IP the same way others see it
   → No IP masquerading from the Pod's perspective
```

**Why no NAT between Pods?**
NAT complicates distributed systems. If Pod A makes a request and the receiving Pod B sees a different source IP, then logging, rate limiting, and access control based on IP become unreliable. Kubernetes guarantees pods see real IPs.

---

## 2. CNI — Container Network Interface

### WHAT
**CNI (Container Network Interface)** is the standard API that Kubernetes uses to configure networking for Pods. When a Pod is created, Kubernetes calls the CNI plugin to:
1. Create a network interface for the Pod
2. Assign it an IP address
3. Set up routing so other Pods can reach it

### WHY
Without CNI, Kubernetes would have to implement its own networking for every cloud and on-premises environment — impossible. CNI defines a standard interface, and vendors implement it for their specific environments.

### Common CNI Plugins

| Plugin | Notes | Use case |
|---|---|---|
| **Calico** | Most popular, supports NetworkPolicies, BGP routing | Production, any cloud |
| **Flannel** | Simple, lightweight, overlay network | Simple setups, learning |
| **Weave Net** | Easy setup, mesh networking | Small clusters |
| **Cilium** | eBPF-based, high performance, L7 policies | High performance, service mesh |
| **AWS VPC CNI** | Native AWS VPC IPs for pods | AWS EKS |
| **Azure CNI** | Native Azure VNet IPs | AKS |

### How CNI Works (simplified)

```
New Pod scheduled on Node 2
        │
        ▼
kubelet calls CNI plugin
        │
        ├── CNI creates veth pair (virtual ethernet cable)
        │   One end inside Pod's network namespace
        │   Other end on the host's bridge network
        │
        ├── CNI assigns IP from Pod CIDR (e.g., 172.16.2.15)
        │
        ├── CNI sets up routes:
        │   "Traffic to 172.16.1.0/24 → send to Node 1"
        │   "Traffic to 172.16.3.0/24 → send to Node 3"
        │
        └── Pod can now send/receive traffic

Pod-to-Pod on SAME node:
  Pod-A (172.16.2.5) → bridge → Pod-B (172.16.2.8)
  Direct via host bridge — no hops

Pod-to-Pod on DIFFERENT nodes:
  Pod-A (172.16.2.5) on Node 2 → host routing → Node 3 → Pod-C (172.16.3.12)
  Via overlay network (VXLAN) or BGP routing depending on CNI
```

---

## 3. The Default — Everything Is Open

### The Problem

```
Default Kubernetes networking:

Namespace: frontend          Namespace: backend         Namespace: database
┌────────────────┐          ┌────────────────┐         ┌────────────────┐
│  frontend-pod  │ ◄──────► │   api-pod      │ ◄─────► │  postgres-pod  │
│  (172.16.1.5)  │ ◄──────► │  (172.16.2.8)  │ ◄─────► │  (172.16.3.4)  │
└────────────────┘          └────────────────┘         └─────────────────┘
         ↕                           ↕                          ↕
ALL traffic allowed between ALL pods on ALL ports — even across namespaces!

Security implications:
- Compromised frontend pod can directly query postgres on port 5432
- Any pod can reach the Kubernetes API server
- Lateral movement after a breach is unrestricted
- Namespaces provide NO network isolation by default
```

**Teaching line:**
> "Default Kubernetes networking is like an office where every desk has a phone that can call every other desk directly, including the CEO's personal line, the server room, and the vault. NetworkPolicies are the switchboard rules that say 'only reception can call the CEO, only IT can call the server room.'"

---

## 4. NetworkPolicy — Kubernetes Firewall Rules

### WHAT
A **NetworkPolicy** is a Kubernetes object that defines rules controlling which Pods can send traffic to which other Pods (and on which ports). It operates at the IP/port level (L3/L4).

### WHY
NetworkPolicies implement the **principle of least privilege** for networking:
- Your API can only reach PostgreSQL and Redis — nothing else
- PostgreSQL can only receive connections from the API — not from the frontend
- Frontend can only reach the API — not the database directly
- A compromised Pod can't reach other services it has no business talking to

### Critical Prerequisite

**NetworkPolicies are only enforced if your CNI plugin supports them.**
- Flannel: ❌ No NetworkPolicy support
- Calico: ✅ Full support
- Cilium: ✅ Full support + L7 policies
- AWS VPC CNI: ✅ (with Calico or Cilium on top)

In Minikube: `minikube start --cni=calico` to enable NetworkPolicy support.

---

## 5. NetworkPolicy Anatomy — Every Field

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-network-policy
  namespace: production      # policies are namespace-scoped

spec:
  # ── Step 1: Which pods does this policy apply TO? ───────────────────────────
  podSelector:
    matchLabels:
      app: taskapi           # this policy applies to pods with label app=taskapi
                             # empty podSelector {} = applies to ALL pods in namespace

  # ── Step 2: Which traffic directions does this policy govern? ───────────────
  policyTypes:
    - Ingress                # control incoming traffic to selected pods
    - Egress                 # control outgoing traffic from selected pods
    # If you specify Ingress but not Egress: only ingress is restricted
    # If you specify neither: nothing changes (policy has no effect)
    # If you specify Ingress with NO ingress rules: ALL ingress blocked
    # If you specify Egress with NO egress rules: ALL egress blocked

  # ── Step 3: Ingress rules (who can send TO selected pods) ───────────────────
  ingress:
    - from:                  # allow from these sources
        - podSelector:       # allow from pods with this label (same namespace)
            matchLabels:
              app: nginx
        - namespaceSelector: # allow from pods in namespaces with this label
            matchLabels:
              env: production
        - ipBlock:           # allow from this IP range
            cidr: 10.0.0.0/8
            except:          # except these sub-ranges
              - 10.0.1.0/24
      ports:                 # on these ports
        - protocol: TCP
          port: 3000

  # ── Step 4: Egress rules (who selected pods can send TO) ─────────────────────
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379
    - to:                    # allow DNS (always needed!)
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

---

## 6. The podSelector + namespaceSelector Combination Rule

This is the most confusing part of NetworkPolicy syntax. Learn it precisely.

```yaml
# Case 1: podSelector AND namespaceSelector (must satisfy BOTH)
from:
  - podSelector:
      matchLabels:
        app: nginx
    namespaceSelector:       # ← same list item (AND)
      matchLabels:
        env: production
# Allows: pods labelled app=nginx AND in namespace labelled env=production

# Case 2: podSelector OR namespaceSelector (either satisfies)
from:
  - podSelector:             # ← separate list items (OR)
      matchLabels:
        app: nginx
  - namespaceSelector:       # ← separate list item
      matchLabels:
        env: production
# Allows: ANY pod labelled app=nginx (in any namespace)
#         OR any pod in a namespace labelled env=production
```

**Teaching line:**
> "Same list item = AND (both must match). Separate list items = OR (either can match). The dash matters — it's the difference between 'nginx pods IN production namespace' and 'nginx pods OR anything in production namespace'."

---

## 7. Default Deny Policies — The Foundation of Zero-Trust

Start with "deny everything" and then explicitly allow only what's needed.

```yaml
# ── Default deny ALL ingress for entire namespace ──────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}            # applies to ALL pods in namespace
  policyTypes:
    - Ingress                # restrict ingress
  # No ingress rules = ALL ingress blocked

---
# ── Default deny ALL egress for entire namespace ───────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  # No egress rules = ALL egress blocked
  # ⚠️ This blocks DNS too — add DNS exception immediately

---
# ── Allow DNS egress (always needed after default-deny-egress) ─────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: production
spec:
  podSelector: {}            # all pods need DNS
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

**The zero-trust recipe:**
```
1. Apply default-deny-ingress (block all incoming)
2. Apply default-deny-egress (block all outgoing)
3. Apply allow-dns-egress (restore DNS)
4. Add specific allow policies for each service
5. Test each connection explicitly
```

---

## 8. Complete Zero-Trust Setup — TaskAPI Stack

A full NetworkPolicy implementation for the TaskAPI stack from Day 21.

```yaml
# ── Label namespaces (required for namespaceSelector) ─────────────────────────
# kubectl label namespace production kubernetes.io/metadata.name=production
# kubectl label namespace kube-system kubernetes.io/metadata.name=kube-system

---
# ── 1. Default deny all in production namespace ────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress

---
# ── 2. Allow DNS for all pods ──────────────────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53

---
# ── 3. Nginx: allow ingress from internet, egress to API ──────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: nginx-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: nginx
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - {}                     # allow ALL ingress (internet traffic — open to world)
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: taskapi
      ports:
        - protocol: TCP
          port: 3000
    # DNS already allowed by allow-dns policy

---
# ── 4. API: allow ingress from nginx + ingress-controller, egress to db + redis ─
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: taskapi
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Allow from nginx in same namespace
    - from:
        - podSelector:
            matchLabels:
              app: nginx
      ports:
        - protocol: TCP
          port: 3000
    # Allow from ingress-nginx controller (different namespace)
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
          podSelector:
            matchLabels:
              app.kubernetes.io/name: ingress-nginx
      ports:
        - protocol: TCP
          port: 3000
  egress:
    # Allow to postgres
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    # Allow to redis
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379
    # Allow to external APIs (HTTPS) — if your API calls external services
    - ports:
        - protocol: TCP
          port: 443
    # DNS handled by allow-dns policy

---
# ── 5. PostgreSQL: allow ingress ONLY from API ────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # ONLY the API can connect to postgres
    - from:
        - podSelector:
            matchLabels:
              app: taskapi
      ports:
        - protocol: TCP
          port: 5432
  egress: []   # postgres doesn't need to make outgoing connections

---
# ── 6. Redis: allow ingress ONLY from API ─────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: redis-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: redis
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: taskapi
      ports:
        - protocol: TCP
          port: 6379
  egress: []   # redis doesn't initiate outgoing connections
```

**Resulting communication matrix:**
```
                nginx  api  postgres  redis  internet
nginx     →      -     ✅     ❌        ❌       ←✅
api       →      ❌    -      ✅        ✅        →✅(443)
postgres  →      ❌    ❌     -         ❌        ❌
redis     →      ❌    ❌     ❌         -         ❌
```

---

## 9. NetworkPolicy Rules — Key Behaviours to Memorise

```
BEHAVIOUR 1: Additive (OR logic across policies)
  Multiple NetworkPolicies selecting the same pod = UNION of all rules
  Policy A allows port 80. Policy B allows port 443.
  Pod allows both 80 AND 443 — not just the last policy.

BEHAVIOUR 2: Implicit deny after first policy
  The MOMENT a NetworkPolicy selects a pod, all non-matching traffic is denied.
  Before any policy: all traffic allowed.
  After first ingress policy: only explicitly allowed ingress works.

BEHAVIOUR 3: Ingress and Egress are independent
  An ingress policy doesn't affect egress.
  To fully lock down a pod, you need BOTH ingress AND egress policies.

BEHAVIOUR 4: policyTypes matters
  policyTypes: [Ingress]          → only restricts ingress, egress still open
  policyTypes: [Ingress, Egress]  → restricts both
  No policyTypes field             → defaults based on which rules are present

BEHAVIOUR 5: Empty selector = all pods
  podSelector: {}  → applies to ALL pods in namespace
  from: [{}]       → allow from ALL pods (including other namespaces!)

BEHAVIOUR 6: Namespace labels required
  namespaceSelector only works if the namespace has the label you're selecting
  kubectl label namespace production env=production
```

---

## 10. Debugging NetworkPolicies

```bash
# ── Check what policies exist ─────────────────────────────────────────────────
kubectl get networkpolicies -n production
kubectl get netpol -n production          # shorthand
kubectl describe networkpolicy api-policy -n production

# ── Test connectivity between pods ────────────────────────────────────────────
# Create a test pod to run network tests
kubectl run nettest --image=nicolaka/netshoot \
  --rm -it --restart=Never -n production -- bash

# Inside the pod:
# Test TCP connection
nc -zv postgres-svc 5432     # should succeed if allowed
nc -zv redis-svc 6379        # should succeed if allowed
nc -zv some-blocked-svc 8080 # should fail if blocked

# Test HTTP
curl -m 5 http://api-svc:3000/health
wget -qO- --timeout=5 http://api-svc:3000/health

# Test DNS
nslookup postgres-svc
dig postgres-svc.production.svc.cluster.local

# ── Check if labels match your selector ───────────────────────────────────────
kubectl get pods -n production --show-labels
kubectl get namespaces --show-labels

# ── Check CNI supports NetworkPolicies ────────────────────────────────────────
kubectl get pods -n kube-system | grep -i calico
kubectl get pods -n kube-system | grep -i cilium
# If neither exists — NetworkPolicies won't be enforced!

# ── Common debugging mistakes ─────────────────────────────────────────────────
# 1. Namespace not labelled → namespaceSelector doesn't match
kubectl label namespace production kubernetes.io/metadata.name=production

# 2. Pod not labelled → podSelector doesn't match
kubectl get pod api-pod --show-labels

# 3. DNS blocked → all external connections fail silently
# Always add allow-dns policy alongside default-deny-egress

# 4. Ingress controller namespace not labelled
kubectl get namespace ingress-nginx --show-labels
kubectl label namespace ingress-nginx kubernetes.io/metadata.name=ingress-nginx
```

---

## 11. Advanced NetworkPolicy Patterns

### Pattern 1: Allow monitoring namespace to scrape all pods

```yaml
# Allow Prometheus (in monitoring namespace) to scrape all production pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: production
spec:
  podSelector: {}              # all pods
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
          podSelector:
            matchLabels:
              app: prometheus
      ports:
        - protocol: TCP
          port: 9090           # or whatever metrics port your app uses
```

### Pattern 2: Allow only within namespace (multi-tenant isolation)

```yaml
# No pod can receive traffic from OUTSIDE the namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: namespace-isolation
  namespace: team-a
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector: {}      # allow from any pod in SAME namespace
```

### Pattern 3: Allow egress to specific external IP (external API)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: taskapi
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 34.107.0.0/16    # Google APIs IP range
      ports:
        - protocol: TCP
          port: 443
```

### Pattern 4: Restrict database access by environment

```yaml
# Database only accepts connections from same-environment namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-env-isolation
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              env: production   # only production namespaced apps reach prod DB
```

---

## 12. Cilium — Next-Generation Network Policies (L7)

Standard NetworkPolicies only work at L3/L4 (IP + port). **Cilium** extends this to L7 (HTTP methods, paths, headers).

```yaml
# Cilium NetworkPolicy — L7 HTTP rules
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: api-l7-policy
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: taskapi
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: nginx
      toPorts:
        - ports:
            - port: "3000"
              protocol: TCP
          rules:
            http:
              - method: "GET"
                path: "/health"
              - method: "GET"
                path: "/api/tasks"
              - method: "POST"
                path: "/api/tasks"
              # Block DELETE, PUT to /admin from nginx
              # Only allow specific methods to specific paths
```

This is far more granular than standard NetworkPolicies — only specific HTTP methods to specific paths are allowed. Used in high-security environments.

---

## 13. NetworkPolicy Quick Reference Card

```
ALLOW ALL INGRESS (open):
  ingress:
    - {}

DENY ALL INGRESS (closed):
  policyTypes: [Ingress]
  # no ingress rules

ALLOW FROM SPECIFIC POD:
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: nginx
      ports:
        - port: 3000

ALLOW FROM NAMESPACE:
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              env: production

ALLOW FROM POD IN SPECIFIC NAMESPACE (AND):
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: nginx
          namespaceSelector:
            matchLabels:
              env: production

ALLOW EGRESS TO IP RANGE:
  egress:
    - to:
        - ipBlock:
            cidr: 10.0.0.0/8

ALLOW DNS EGRESS:
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
```

---

## 14. Hands-On Practice Tasks

```bash
# Setup: Minikube with Calico CNI for NetworkPolicy support
minikube start --cni=calico
# Wait for calico pods:
kubectl get pods -n kube-system | grep calico

# 1. Label namespaces
kubectl label namespace default kubernetes.io/metadata.name=default
kubectl label namespace kube-system kubernetes.io/metadata.name=kube-system

# 2. Deploy two test pods
kubectl run frontend --image=nginx --labels="app=frontend"
kubectl run backend --image=nginx --labels="app=backend"
kubectl expose pod backend --port=80 --name=backend-svc

# 3. Test connectivity BEFORE any policy (should work)
kubectl exec -it frontend -- curl -m 3 http://backend-svc
# ✅ Should succeed

# 4. Apply default-deny
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: default
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
EOF

# 5. Test AFTER default-deny (should fail)
kubectl exec -it frontend -- curl -m 3 http://backend-svc
# ❌ Should time out

# 6. Allow DNS + frontend→backend
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: default
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-to-backend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 80
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-ingress
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 80
EOF

# 7. Test AFTER explicit allow (should work)
kubectl exec -it frontend -- curl -m 3 http://backend-svc
# ✅ Should succeed again

# 8. Verify backend can't reach frontend (no egress allowed for backend)
kubectl exec -it backend -- curl -m 3 http://frontend
# ❌ Should fail (backend has no egress policy)

# 9. Clean up
kubectl delete networkpolicy --all
kubectl delete pod frontend backend
kubectl delete svc backend-svc
```

---

## 15. Quiz — Test Yourself / Test Others

1. What are the four core requirements of Kubernetes networking?
2. What is a CNI plugin and why does Kubernetes need one?
3. Why does Flannel NOT support NetworkPolicies but Calico does?
4. What is the default networking behaviour in Kubernetes before any NetworkPolicy is applied?
5. What is the difference between `podSelector` at the spec level vs inside `from`/`to`?
6. Explain the AND vs OR difference when using `podSelector` and `namespaceSelector` in a `from` block.
7. What happens to a pod's network traffic the moment its first NetworkPolicy is applied?
8. Why must you always add a DNS egress allow policy alongside a default-deny-egress policy?
9. You have a default-deny-ingress policy. Your API pod needs to receive traffic from nginx. Write the exact `from` block needed.
10. A NetworkPolicy is applied but traffic that should be blocked still flows. What are three possible causes?

<details>
<summary>Answers</summary>

1. (1) Every Pod gets a unique IP address — no NAT between pods. (2) Any Pod can communicate with any other Pod without NAT. (3) Agents on a node (kubelet, system daemons) can communicate with all Pods on that node. (4) Pods see their own IP the same way others see it — no IP masquerading.

2. A CNI (Container Network Interface) plugin is software that configures networking for Pods when they are created. Kubernetes defines a standard interface (CNI) and calls the plugin to: create a network interface for the Pod, assign it an IP address, and set up routing so other Pods can reach it. Kubernetes needs CNI because networking differs fundamentally across cloud providers, on-premises setups, and operating systems — CNI abstracts this complexity behind a standard API.

3. Enforcing NetworkPolicies requires the network layer to inspect packets and allow/deny them based on source/destination labels. Flannel implements a simple overlay network (VXLAN) that forwards all packets without inspection — it has no mechanism to evaluate NetworkPolicy rules. Calico implements a full packet-filtering system (using iptables/eBPF) that can enforce allow/deny rules based on Pod labels and IP blocks. NetworkPolicy objects exist regardless, but without an enforcing CNI they have no effect.

4. All networking is completely open — every Pod can reach every other Pod on every port, across all namespaces. There is no isolation between Pods, between namespaces, or between services. Namespaces provide zero network isolation by default.

5. `podSelector` at the `spec` level identifies WHICH pods this NetworkPolicy applies TO — it defines the scope of the policy. `podSelector` inside `from`/`to` identifies WHERE traffic is allowed FROM or TO — it defines the traffic source or destination in a rule. They're completely different roles despite using the same syntax.

6. AND (same list item): `- podSelector: {matchLabels: {app: nginx}} namespaceSelector: {matchLabels: {env: prod}}` — allows only pods labelled app=nginx that are ALSO in a namespace labelled env=prod. Both conditions must match. OR (separate list items): `- podSelector: {matchLabels: {app: nginx}}` `- namespaceSelector: {matchLabels: {env: prod}}` — allows pods labelled app=nginx (in any namespace) OR any pod in a namespace labelled env=prod. Either condition matching is sufficient.

7. The moment the first NetworkPolicy selects a pod (via `podSelector`), all traffic NOT explicitly allowed by that policy (or subsequent policies) is denied for the traffic direction(s) specified in `policyTypes`. Before any policy: everything allowed. After first policy: only explicitly listed traffic is allowed, everything else is implicitly denied for that direction.

8. DNS resolution requires outgoing UDP/TCP connections to port 53 on the kube-dns service in kube-system. A default-deny-egress policy blocks ALL outgoing traffic including DNS queries. Without DNS, your Pod can't resolve any service names (`postgres-svc`, `api-svc`, external domains) — all connections will fail with name resolution errors even though the actual ports may be allowed. DNS is the foundation of all Kubernetes service discovery.

9. ```yaml
ingress:
  - from:
      - podSelector:
          matchLabels:
            app: nginx
    ports:
      - protocol: TCP
        port: 3000
```
The `from` block uses `podSelector` to match pods labelled `app: nginx` in the same namespace. The `ports` block specifies which port of the API pod is accessible.

10. Any three of: (1) CNI plugin doesn't support NetworkPolicies (e.g., Flannel) — policies exist but are never enforced. (2) Missing namespace labels — `namespaceSelector` can't match because the namespace isn't labelled with the expected key-value. (3) Missing pod labels — `podSelector` doesn't match because pods aren't labelled correctly. (4) Multiple policies are additive — another policy is allowing the traffic you expect to be blocked. (5) Wrong namespace — the NetworkPolicy is in a different namespace than the pods you expect to restrict.

</details>

---

## 16. Summary

After today you can explain and implement:
- **Kubernetes networking requirements** — four guarantees, no NAT between Pods
- **CNI plugins** — what they do, Calico vs Flannel vs Cilium, how pod IP assignment works
- **Default open networking** — the security risk and why it matters
- **NetworkPolicy anatomy** — `podSelector`, `policyTypes`, `ingress`/`egress`, `from`/`to`
- **AND vs OR selector logic** — same list item vs separate items in `from` blocks
- **Default deny pattern** — always start with deny-all, then add explicit allows
- **DNS exception** — why it's always needed alongside default-deny-egress
- **Complete zero-trust setup** — nginx → api → postgres/redis with full policy matrix
- **Advanced patterns** — monitoring scraping, namespace isolation, external IP access, environment isolation
- **Cilium L7 policies** — HTTP method + path level control
- **Debugging toolkit** — label checking, netshoot testing, common mistakes

**Coming up on Day 26:** Monitoring & Logging — Prometheus, Grafana, and kubectl logs. How to observe what's happening inside your cluster, set up metrics scraping, create dashboards, and configure alerts so you know about problems before users do.
