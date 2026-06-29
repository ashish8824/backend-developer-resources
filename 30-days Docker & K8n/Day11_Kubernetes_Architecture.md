# Day 11: Kubernetes Architecture — The Brain and the Body

**Goal for today:** Understand every component of a Kubernetes cluster — what each one is, why it exists, what it does, and how they all communicate. By the end, you should be able to draw the full Kubernetes architecture from memory and explain every component in plain English to someone who has never heard of Kubernetes.

**Format:** WHAT → WHY → HOW for every concept.

---

## 0. Quick Recap (open your teaching session with this)

- Days 1–9: Docker fundamentals — images, containers, networking, volumes, Compose, registries
- Day 10: Why orchestration? — SPOF, no self-healing, fake scaling, deployment downtime
- The desired-state model: you declare *what* you want, Kubernetes figures out *how* to achieve and maintain it

**Today's connecting thought:**
> On Day 10 you understood *why* Kubernetes exists. Today you open the hood and see *how* it works. Every component we study today exists to solve one of the problems from Day 10. Keep asking "which problem does this solve?" as we go through each component.

---

## 1. The Two Halves of a Kubernetes Cluster

Every Kubernetes cluster has exactly two types of machines:

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Kubernetes Cluster                              │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    CONTROL PLANE                              │   │
│  │            (The Brain — makes all decisions)                  │   │
│  │                                                               │   │
│  │  API Server │ etcd │ Scheduler │ Controller Manager           │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                   communicates with                                  │
│                              │                                       │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────────┐  │
│  │  Worker     │  │  Worker     │  │  Worker     │  │  Worker      │  │
│  │  Node 1     │  │  Node 2     │  │  Node 3     │  │  Node 4      │  │
│  │             │  │             │  │             │  │              │  │
│  │  kubelet    │  │  kubelet    │  │  kubelet    │  │  kubelet     │  │
│  │  kube-proxy │  │  kube-proxy │  │  kube-proxy │  │  kube-proxy  │  │
│  │  container  │  │  container  │  │  container  │  │  container   │  │
│  │  runtime    │  │  runtime    │  │  runtime    │  │  runtime     │  │
│  │             │  │             │  │             │  │              │  │
│  │  [Pod][Pod] │  │  [Pod][Pod] │  │  [Pod][Pod] │  │  [Pod][Pod]  │  │
│  └────────────┘  └────────────┘  └────────────┘  └──────────────┘  │
│                    DATA PLANE                                        │
│              (The Body — does the actual work)                       │
└─────────────────────────────────────────────────────────────────────┘
```

| Half | Also called | Role | Analogy |
|---|---|---|---|
| **Control Plane** | Master | Makes all decisions — schedules Pods, watches state, responds to changes | The brain / the manager |
| **Worker Nodes** | Data Plane | Actually runs your containers | The workers / the body |

**Teaching line:**
> "Think of a Kubernetes cluster as a company. The Control Plane is the management team — they decide what gets done, who does it, and watch for problems. The Worker Nodes are the employees — they actually do the work (run your containers). The management team never does the work themselves; they only manage."

---

## 2. The Control Plane — Component by Component

The Control Plane has four main components. Each solves a specific problem.

---

### 2.1 API Server (`kube-apiserver`) — The Front Door

#### WHAT
The **API Server** is the central communication hub of the entire cluster. Every interaction — from `kubectl` commands you type, to internal component communication, to automation tools — goes through the API Server. It is the **only** component that reads from and writes to etcd (the database).

#### WHY
Centralising all communication through one component means:
- One place to authenticate and authorise every request
- One consistent API for everything (humans, tools, internal components)
- One place to validate requests before any action is taken
- Easier auditing — every change to cluster state goes through here

#### HOW
```
kubectl apply -f deployment.yaml
        │
        │  HTTP/REST request
        ▼
  ┌─────────────┐
  │  API Server  │  ← validates request, checks auth, writes to etcd
  └──────┬──────┘
         │  notifies other components
         ▼
  Scheduler, Controller Manager, kubelets
  (all watch the API Server for changes)
```

**Key facts:**
- Exposes a REST API (you can even call it with `curl` — `kubectl` is just a wrapper)
- Stateless — can be horizontally scaled (run multiple API Server replicas for HA)
- All cluster communication is signed with TLS certificates
- Supports RBAC (Role-Based Access Control) — who can do what

**Teaching line:**
> "The API Server is the reception desk of the Kubernetes office building. Every request — whether from a developer typing `kubectl`, a CI/CD pipeline, or an internal component — must go through reception. Reception validates your ID, checks your permissions, logs your visit, and routes you to the right department."

---

### 2.2 etcd — The Cluster's Database

#### WHAT
**etcd** is a distributed, consistent key-value store that serves as Kubernetes' **single source of truth**. Every piece of cluster state — what Pods should be running, what Nodes exist, what Services are defined, what ConfigMaps contain — is stored in etcd.

#### WHY
Kubernetes needs somewhere to store the desired state you declare and the current state it observes. etcd was chosen because:
- **Distributed:** can run across multiple machines — no single point of failure for state storage
- **Consistent:** all reads see the latest write (uses Raft consensus algorithm)
- **Watch API:** components can subscribe to changes — "notify me when anything in /pods changes"
- **Fast:** designed for high-throughput, low-latency reads and writes

#### HOW
```
etcd stores everything as key-value pairs:
/registry/pods/default/my-api-pod-1        → { spec, status, metadata }
/registry/pods/default/my-api-pod-2        → { spec, status, metadata }
/registry/deployments/default/my-api       → { replicas: 3, image: myapp:1.0 }
/registry/services/default/my-api-service  → { clusterIP, ports, selector }
/registry/nodes/worker-1                   → { capacity, conditions, addresses }
```

**Critical rules to teach:**
- **Only the API Server** talks to etcd directly. No other component reads/writes etcd.
- etcd is the most critical component — if etcd data is lost, the entire cluster state is lost. **Back it up.**
- In production, etcd runs as a 3 or 5-node cluster (odd numbers for Raft quorum)

**Teaching line:**
> "etcd is the cluster's memory — its long-term, persistent, reliable memory. Everything Kubernetes knows and does is ultimately stored here. The API Server is the only one allowed to write in this notebook, but everyone reads from it (through the API Server)."

---

### 2.3 Scheduler (`kube-scheduler`) — The Placement Engine

#### WHAT
The **Scheduler** watches for newly created Pods that haven't been assigned to a Node yet, and decides which Node each Pod should run on.

#### WHY
This directly solves Problem 5 from Day 10 — intelligent placement. The Scheduler considers:
- **Resource requests:** "This Pod needs 2 CPUs and 4GB RAM — only nodes with that available qualify"
- **Affinity/Anti-affinity rules:** "Put this Pod on a node with an SSD" or "Don't put two replicas of this Pod on the same node"
- **Taints and Tolerations:** "This node is reserved for GPU workloads — only schedule GPU Pods here"
- **Node conditions:** "Node 3 is under memory pressure — don't schedule there"

#### HOW
```
Scheduling algorithm (simplified):

New Pod created (no Node assigned)
        │
        ▼
  ┌─────────────────────────────────────────────┐
  │  Step 1: FILTERING                           │
  │  Remove nodes that cannot run this Pod       │
  │  (not enough CPU, wrong architecture, etc.)  │
  └────────────────────┬────────────────────────┘
                       │
                       ▼
  ┌─────────────────────────────────────────────┐
  │  Step 2: SCORING                             │
  │  Rank remaining nodes:                       │
  │  - most available resources → higher score   │
  │  - least Pods already running → higher score │
  │  - node affinity match → higher score        │
  └────────────────────┬────────────────────────┘
                       │
                       ▼
  ┌─────────────────────────────────────────────┐
  │  Step 3: BINDING                             │
  │  Write chosen node to Pod spec in etcd       │
  │  (via API Server)                            │
  └─────────────────────────────────────────────┘
```

**Teaching line:**
> "The Scheduler is like a hotel concierge who assigns rooms to guests. They check room availability (resources), guest preferences (affinity rules), and hotel policy (taints). They never actually put the guest in the room — they just decide which room and hand the assignment to housekeeping (kubelet)."

---

### 2.4 Controller Manager (`kube-controller-manager`) — The Reconciliation Engine

#### WHAT
The **Controller Manager** runs a collection of **controllers** — each controller is a control loop that continuously watches the cluster state and takes action to move actual state toward desired state. This is the component that implements the reconciliation model from Day 10.

#### WHY
This directly solves the self-healing problem. Examples of controllers inside the Controller Manager:

| Controller | What it watches | What it does |
|---|---|---|
| **ReplicaSet Controller** | Number of running Pods for a ReplicaSet | Creates/deletes Pods to match desired replica count |
| **Deployment Controller** | Deployment objects | Creates/updates ReplicaSets for rolling deployments |
| **Node Controller** | Node health | Marks nodes as unreachable, evicts Pods from dead nodes |
| **Service Account Controller** | Namespaces | Creates default service accounts in new namespaces |
| **Job Controller** | Job objects | Creates Pods for jobs, tracks completion |

#### HOW (the reconciliation loop — memorise this)
```
Every controller runs this loop continuously:

while (true) {
    desired_state = read from etcd (via API Server)
    actual_state  = observe real cluster
    
    if (actual_state != desired_state) {
        take_action_to_close_the_gap()
    }
    
    sleep(a few seconds)
}

Example — ReplicaSet Controller:
  desired: 3 Pods running for "api"
  actual:  2 Pods running (one crashed)
  action:  create 1 new Pod → submit to API Server → Scheduler assigns node → kubelet starts it
```

**Teaching line:**
> "Each controller is like a thermostat. You set the desired temperature (desired state). The thermostat continuously checks the actual temperature (actual state). If it's too cold, it turns on the heater (takes action). It doesn't do it once — it does it in a continuous loop, forever."

---

### 2.5 Cloud Controller Manager (optional — for cloud deployments)

When Kubernetes runs on a cloud (AWS, GCP, Azure), a **Cloud Controller Manager** integrates Kubernetes with the cloud provider's APIs:
- Creating an AWS Load Balancer when a Service of type `LoadBalancer` is created
- Provisioning an EBS volume when a PersistentVolumeClaim is created
- Updating Route53 DNS entries

This is why Kubernetes on EKS "just works" with AWS services — the Cloud Controller Manager bridges them.

---

## 3. The Worker Nodes — Component by Component

Every worker node runs three things:

---

### 3.1 kubelet — The Node Agent

#### WHAT
The **kubelet** is an agent that runs on every worker node. It is the only component that actually creates, starts, stops, and monitors containers. The control plane decides *what* should run; the kubelet makes it happen.

#### WHY
The control plane can't directly SSH into nodes and run Docker commands — that would be tightly coupled and fragile. Instead, each node has a kubelet that:
- Watches the API Server for Pods assigned to its node
- Talks to the container runtime (like Docker or containerd) to start/stop containers
- Reports the node's status and Pod health back to the API Server
- Runs container health checks (liveness/readiness probes)

#### HOW
```
Kubelet's job on each node:

1. Watches API Server: "Any Pods assigned to my node?"
2. Pod assigned → reads Pod spec
3. Tells container runtime: "Start this container with these settings"
4. Monitors container → if it fails a health check → restart it
5. Reports back to API Server: "Pod X is Running / Failed / Pending"
6. Reports node resources: "I have 2 CPU and 3GB RAM available"
```

**Teaching line:**
> "If the Scheduler is the hotel concierge who assigns rooms, the kubelet is the housekeeping team on each floor who actually prepares and maintains the rooms. The concierge tells them which room to prepare — they do the physical work."

---

### 3.2 kube-proxy — The Network Rules Engine

#### WHAT
**kube-proxy** runs on every node and maintains network rules (using Linux iptables or IPVS) that implement Kubernetes Services — the stable network endpoints that load-balance traffic across Pods.

#### WHY
Pods get new IP addresses every time they restart. You can't hardcode Pod IPs anywhere. Services provide a stable virtual IP (ClusterIP) that always routes to healthy Pods. kube-proxy is what makes this work at the network level.

#### HOW
```
Without kube-proxy:
  client → Pod-1 IP (172.10.1.5) → but Pod-1 died and IP changed → broken

With kube-proxy:
  client → Service ClusterIP (10.96.100.1) → kube-proxy rule →
  routes to any of: Pod-1 (172.10.1.5), Pod-2 (172.10.1.8), Pod-3 (172.10.2.3)
  → one of them responds
  → Pod-1 dies → kube-proxy updates rules → no longer routes to dead Pod-1
```

**Teaching line:**
> "kube-proxy is like a switchboard operator on every floor of the building. When a call comes in for 'the sales team' (a Service), they don't care exactly which sales person picks up (which Pod). They route it to whoever is available and healthy."

---

### 3.3 Container Runtime — The Container Engine

#### WHAT
The **container runtime** is the software that actually runs containers on the node. Kubernetes doesn't run containers directly — it delegates to a container runtime via the **CRI (Container Runtime Interface)**.

#### WHY
Kubernetes is container-runtime-agnostic — it defines a standard interface (CRI) and any runtime implementing it can be used. This decouples Kubernetes from Docker's specific implementation.

#### Common runtimes
| Runtime | Notes |
|---|---|
| **containerd** | Most common today — lightweight, Docker's own runtime extracted |
| **CRI-O** | Lightweight runtime specifically built for Kubernetes |
| **Docker (dockershim)** | Was supported until K8s v1.24, then removed |

**Important nuance to teach:**
> Kubernetes stopped supporting Docker as a runtime in v1.24 (2022) — this caused confusion but doesn't affect YOU as a developer. Your Dockerfiles and images still work fine. The runtime that actually executes containers on nodes just changed from Docker to containerd, which runs the same OCI-standard images your Dockerfile produces.

---

## 4. Communication Flow — How It All Works Together

Walk through the complete lifecycle of a Pod from `kubectl apply` to running container. This is the most important explanation to be able to give from memory.

```
You run: kubectl apply -f deployment.yaml

Step 1: kubectl → API Server
  - kubectl sends the Deployment spec as a REST API call
  - API Server authenticates + authorises the request
  - API Server validates the Deployment spec
  - API Server writes the Deployment to etcd
  - API Server returns: "Deployment created"

Step 2: Deployment Controller (in Controller Manager) watches API Server
  - "A new Deployment appeared! I need to create a ReplicaSet for it"
  - Creates a ReplicaSet with 3 replicas → writes to etcd via API Server

Step 3: ReplicaSet Controller watches API Server
  - "A new ReplicaSet appeared with 3 replicas! I need 3 Pods"
  - Creates 3 Pod objects (no Node assigned yet) → writes to etcd via API Server

Step 4: Scheduler watches API Server
  - "3 new Pods with no Node assigned!"
  - Runs filtering + scoring for each Pod
  - Assigns each Pod to a Node → updates Pod spec in etcd via API Server

Step 5: kubelet on each assigned Node watches API Server
  - "A Pod has been assigned to MY node!"
  - Reads the Pod spec (which containers, which image, ports, volumes, etc.)
  - Tells containerd: "Pull image myapp:1.0, create and start this container"
  - Container starts running
  - kubelet monitors it, runs health checks
  - kubelet reports back: "Pod is Running" → updates etcd via API Server

Step 6: kube-proxy on every Node
  - "A new Service exists for this Deployment"
  - Updates iptables rules: Service ClusterIP → route to all 3 Pod IPs
  - Traffic now load-balances across all 3 Pods

You see:
  kubectl get pods → 3 pods, all Running
```

**Teaching line:**
> "Nothing in Kubernetes happens directly. Every action is a message through the API Server. Every component watches for changes and reacts. No component tells another component what to do directly — they all just watch the shared state and do their part."

---

## 5. etcd Disaster Recovery — Why This Matters

Since etcd is the source of truth for the entire cluster, losing it means losing all cluster state (which Pods should run, which Secrets exist, all your configs). Production setups:

1. **Run etcd as a 3-node cluster** — tolerates 1 node failure (5-node tolerates 2)
2. **Back up etcd regularly** — `etcdctl snapshot save`
3. **Encrypt etcd data at rest** — it contains Secrets in encoded form

```bash
# Backup etcd (run on control plane node)
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key
```

---

## 6. Managed Kubernetes (EKS, GKE, AKS) — What Cloud Does For You

When you use a managed Kubernetes service like AWS EKS (relevant to your existing ECS Fargate experience):

**Cloud manages:**
- The entire Control Plane (API Server, etcd, Scheduler, Controller Manager)
- Control plane HA (multi-AZ)
- Control plane upgrades
- etcd backups
- Control plane security patches

**You manage:**
- Worker Nodes (EC2 instances in EKS, or use Fargate for serverless nodes)
- Node upgrades
- Application deployments
- Networking configuration

**Why this matters:**
> Running your own Kubernetes control plane is complex. EKS/GKE/AKS let you use Kubernetes without becoming a Kubernetes infrastructure expert. You just use it. This is why managed K8s dominates production usage.

---

## 7. Kubernetes Architecture — Full Visual Summary

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          CONTROL PLANE                                       │
│                                                                              │
│  ┌───────────────────┐    ┌──────────────────────────────────────────────┐  │
│  │      etcd          │    │                API Server                    │  │
│  │                    │◄───│  (only component that reads/writes etcd)     │  │
│  │  cluster state     │    │  (all communication flows through here)      │  │
│  │  (key-value store) │    └────────────────┬─────────────────────────────┘  │
│  └───────────────────┘                     │ watches for changes             │
│                             ┌──────────────┼──────────────────┐             │
│                             ▼              ▼                  ▼             │
│                    ┌──────────────┐ ┌────────────────┐  ┌───────────────┐  │
│                    │  Scheduler   │ │   Controller   │  │    Cloud      │  │
│                    │              │ │   Manager      │  │   Controller  │  │
│                    │  assigns     │ │                │  │   Manager     │  │
│                    │  Pods to     │ │  runs control  │  │   (optional)  │  │
│                    │  Nodes       │ │  loops for     │  │               │  │
│                    └──────────────┘ │  self-healing  │  └───────────────┘  │
│                                     └────────────────┘                      │
└─────────────────────────────────────────────────────────────────────────────┘
                          │ API calls over TLS
                          ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                              WORKER NODES                                     │
│                                                                               │
│  ┌─────────────────────────────┐  ┌─────────────────────────────┐            │
│  │         Worker Node 1        │  │         Worker Node 2        │            │
│  │                              │  │                              │            │
│  │  ┌─────────┐  ┌──────────┐  │  │  ┌─────────┐  ┌──────────┐  │            │
│  │  │ kubelet  │  │kube-proxy│  │  │  │ kubelet  │  │kube-proxy│  │            │
│  │  │          │  │          │  │  │  │          │  │          │  │            │
│  │  │ watches  │  │ network  │  │  │  │ watches  │  │ network  │  │            │
│  │  │ API svr  │  │  rules   │  │  │  │ API svr  │  │  rules   │  │            │
│  │  └────┬─────┘  └──────────┘  │  │  └────┬─────┘  └──────────┘  │            │
│  │       │                      │  │       │                        │            │
│  │       ▼                      │  │       ▼                        │            │
│  │  ┌──────────────┐            │  │  ┌──────────────┐              │            │
│  │  │  containerd  │            │  │  │  containerd  │              │            │
│  │  │  (runtime)   │            │  │  │  (runtime)   │              │            │
│  │  └──────┬───────┘            │  │  └──────┬───────┘              │            │
│  │         │                    │  │         │                      │            │
│  │  [Pod]  [Pod]   [Pod]        │  │  [Pod]  [Pod]                  │            │
│  └─────────────────────────────┘  └─────────────────────────────┘  │            │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Component Cheat Sheet — Quick Reference Card

| Component | Lives on | Role | Problem it solves |
|---|---|---|---|
| **API Server** | Control Plane | Central hub, REST API, auth, validation | Single communication interface |
| **etcd** | Control Plane | Key-value store, cluster state | Persistent, distributed state storage |
| **Scheduler** | Control Plane | Assigns Pods to Nodes | Intelligent placement |
| **Controller Manager** | Control Plane | Runs reconciliation loops | Self-healing, desired state |
| **Cloud Controller Manager** | Control Plane | Cloud provider integration | ELBs, EBS, DNS from K8s |
| **kubelet** | Every Node | Starts/stops containers via CRI | Actual container lifecycle |
| **kube-proxy** | Every Node | Network rules for Services | Stable endpoints, load balancing |
| **Container Runtime** | Every Node | Runs containers (containerd/CRI-O) | OCI container execution |

---

## 9. Hands-On Tasks — Install kubectl and Explore

```bash
# ── Install kubectl ───────────────────────────────────────────────────────────
# macOS
brew install kubectl

# Ubuntu/Debian
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# Verify
kubectl version --client

# ── Install Minikube (local single-node K8s cluster) ─────────────────────────
# macOS
brew install minikube

# Ubuntu
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start a local cluster
minikube start

# ── Explore the cluster components ───────────────────────────────────────────
# See all nodes in the cluster
kubectl get nodes

# See all system components running in the kube-system namespace
kubectl get pods -n kube-system

# You'll see: etcd, kube-apiserver, kube-controller-manager, kube-scheduler,
#             coredns (DNS), kube-proxy, etc.

# Describe the control plane node
kubectl describe node minikube

# See cluster info
kubectl cluster-info

# Watch the API Server event stream
kubectl get events --watch
```

**What to look for in `kubectl get pods -n kube-system`:**
```
NAME                               READY   STATUS    RESTARTS
coredns-xxx                        1/1     Running   0          ← DNS server
etcd-minikube                      1/1     Running   0          ← THE etcd
kube-apiserver-minikube            1/1     Running   0          ← THE API Server
kube-controller-manager-minikube   1/1     Running   0          ← THE Controller Manager
kube-proxy-xxx                     1/1     Running   0          ← node-level kube-proxy
kube-scheduler-minikube            1/1     Running   0          ← THE Scheduler
```

Every component you studied today is running as a Pod in the `kube-system` namespace — even Kubernetes runs itself using Kubernetes. This is called **self-hosting**.

---

## 10. Quiz — Test Yourself / Test Others

1. What are the two halves of a Kubernetes cluster called? What does each do?
2. Which component is the ONLY one that reads from and writes to etcd directly?
3. What does the Scheduler do and what factors does it consider when placing a Pod?
4. What is the reconciliation loop? Give a concrete example of it in action.
5. What is the kubelet and where does it run?
6. Why did Kubernetes remove Docker as a supported container runtime in v1.24? Does this break your Dockerfiles?
7. What does kube-proxy do? Why do we need it?
8. Name three controllers inside the Controller Manager and what each watches/does.
9. When you use AWS EKS, which parts of Kubernetes does AWS manage for you?
10. Describe the full lifecycle of a Pod from `kubectl apply` to a running container, naming every component involved in order.

<details>
<summary>Answers</summary>

1. Control Plane (the brain — makes all decisions: scheduling, healing, deploying) and Worker Nodes / Data Plane (the body — actually runs your containers). Control Plane = API Server + etcd + Scheduler + Controller Manager. Nodes = kubelet + kube-proxy + container runtime.

2. The API Server. It is the single gateway between all other components and etcd. No other component communicates with etcd directly — they all go through the API Server.

3. The Scheduler watches for newly created Pods with no Node assigned, then decides which Node each Pod should run on. It considers: resource requests/limits (CPU, memory), node affinity/anti-affinity rules, taints and tolerations, node conditions (disk pressure, memory pressure), and pod topology constraints.

4. The reconciliation loop is a continuous control loop that compares desired state (from etcd) with actual state (observed cluster), then takes action to close the gap. Example: a Deployment has desired replicas = 3, but one Pod crashes, making actual = 2. The ReplicaSet Controller detects the mismatch and creates a new Pod to restore desired state.

5. kubelet is an agent that runs on every worker node. It watches the API Server for Pods assigned to its node, instructs the container runtime (containerd) to start/stop containers, monitors their health, and reports status back to the API Server.

6. Kubernetes removed the dockershim (the Docker-specific adapter) because Docker as a runtime had too much overhead — it added an unnecessary layer (Docker daemon → containerd → container). Kubernetes now talks directly to containerd via CRI. This does NOT break Dockerfiles or images — your images are standard OCI format which containerd runs perfectly.

7. kube-proxy runs on every node and maintains network rules (iptables/IPVS) that implement Kubernetes Services. It ensures that traffic sent to a Service's ClusterIP is load-balanced across all healthy Pods backing that Service, and that routing rules are updated when Pods start or stop.

8. Any three: ReplicaSet Controller (watches ReplicaSets, creates/deletes Pods to match replica count), Deployment Controller (watches Deployments, manages rolling updates by creating/updating ReplicaSets), Node Controller (watches Nodes, evicts Pods from unhealthy/dead nodes), Job Controller (watches Jobs, creates Pods to execute them), Service Account Controller (creates default service accounts in new namespaces).

9. AWS manages the entire control plane: API Server (multi-AZ HA), etcd (backups, HA), Scheduler, Controller Manager, control plane security patches and upgrades, and cloud integration (Cloud Controller Manager for ELB, EBS, Route53). You manage worker nodes (EC2) or use Fargate for serverless nodes.

10. kubectl sends Deployment spec → API Server (auth, validate, write to etcd) → Deployment Controller sees new Deployment → creates ReplicaSet (via API Server → etcd) → ReplicaSet Controller sees new ReplicaSet → creates Pods with no Node (via API Server → etcd) → Scheduler sees unscheduled Pods → assigns Nodes (writes to etcd via API Server) → kubelet on assigned Node sees Pod assigned to it → tells containerd to pull image and start container → container starts → kubelet reports Running → kube-proxy updates iptables rules → Service routes traffic to new Pod.

</details>

---

## 11. Summary

After today you can explain from memory:
- The two halves of every Kubernetes cluster: Control Plane and Worker Nodes
- All 4 Control Plane components: **API Server** (central hub), **etcd** (state store), **Scheduler** (placement), **Controller Manager** (reconciliation loops)
- All 3 Node components: **kubelet** (container lifecycle), **kube-proxy** (network rules), **container runtime** (runs containers)
- The complete Pod lifecycle from `kubectl apply` to running container — naming every component in order
- Why the API Server is the central, only-etcd-writer hub
- The reconciliation loop — the engine behind all self-healing
- Why Kubernetes dropped Docker as a runtime and why it doesn't matter for developers
- What managed Kubernetes (EKS) handles vs what you manage

**Coming up on Day 12:** Setting up a local Kubernetes cluster with Minikube/Kind and learning `kubectl` — the CLI tool you'll use for everything in Kubernetes, equivalent to what the `docker` CLI is for Docker.
