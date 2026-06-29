# Day 12: Local Kubernetes Setup — Minikube, Kind & kubectl Mastery

**Goal for today:** Set up a fully working local Kubernetes cluster using Minikube or Kind, and build real fluency with `kubectl` — the CLI tool you'll use for everything in Kubernetes. By the end, you should be able to spin up a cluster, deploy workloads, inspect every object, debug problems, and tear it all down — entirely from the command line.

**Format:** WHAT → WHY → HOW for every concept.

---

## 0. Quick Recap (open your teaching session with this)

- Day 10: Why orchestration? — the 6 problems Compose can't solve at scale
- Day 11: Kubernetes architecture — Control Plane (API Server, etcd, Scheduler, Controller Manager) + Worker Nodes (kubelet, kube-proxy, container runtime) + the reconciliation loop

**Today's connecting thought:**
> Day 11 was theory — understanding what's inside a Kubernetes cluster. Today is practice — you'll actually run one on your laptop and talk to it using `kubectl`. Think of today as your first day actually driving a car after studying how the engine works. The theory makes the practice make sense.

---

## 1. Local Kubernetes Options — What Exists and Why

### WHAT
Running a production Kubernetes cluster requires multiple VMs and complex networking. For learning and local development, several tools simulate a full cluster on your laptop.

### The Three Main Options

| Tool | What it is | Best for |
|---|---|---|
| **Minikube** | Runs a K8s cluster inside a VM or container on your laptop | Learning, matches real K8s closely |
| **Kind** (K8s in Docker) | Runs K8s nodes as Docker containers | CI/CD pipelines, fast startup |
| **k3d** | Lightweight K3s (minimal K8s) in Docker | Very fast, low resource usage |
| **Docker Desktop K8s** | Built into Docker Desktop | Quickest start if you have Docker Desktop |

### WHY choose Minikube for learning
- Creates a real multi-component K8s cluster (not a stripped-down version)
- Has built-in addons (dashboard, metrics-server, ingress) that mirror real cluster features
- Well-documented, largest community for beginners
- Supports multiple drivers (Docker, VirtualBox, Hyperkit)

### WHY choose Kind for CI/CD and speed
- Runs K8s nodes as Docker containers → faster to start/stop
- No VM overhead → lighter on RAM
- Supports multi-node clusters easily
- Official K8s project

**We'll cover both — use Minikube as your primary learning tool.**

---

## 2. Installing Minikube

### WHAT
Minikube is a CLI tool that creates and manages a local Kubernetes cluster.

### HOW

**macOS:**
```bash
# Install via Homebrew
brew install minikube

# Verify
minikube version
```

**Ubuntu/Debian (Linux):**
```bash
# Download the binary
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64

# Install it
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Verify
minikube version
```

**Windows:**
```powershell
# Using Chocolatey
choco install minikube

# Or download installer from minikube.sigs.k8s.io
```

---

## 3. Installing kubectl

### WHAT
`kubectl` is the official CLI for Kubernetes — you use it to communicate with any Kubernetes cluster (local, cloud, production). It sends REST API calls to the API Server.

### HOW

**macOS:**
```bash
brew install kubectl
kubectl version --client
```

**Ubuntu/Debian:**
```bash
# Download latest stable version
curl -LO "https://dl.k8s.io/release/$(curl -L -s \
  https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Install
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Verify
kubectl version --client
```

**Windows:**
```powershell
choco install kubernetes-cli
```

---

## 4. Starting and Managing Your Cluster

### Starting Minikube

```bash
# Start with Docker driver (recommended — no VM needed)
minikube start --driver=docker

# Start with specific Kubernetes version (pin it for reproducibility)
minikube start --driver=docker --kubernetes-version=v1.29.0

# Start with more resources (for running real apps)
minikube start --driver=docker --cpus=4 --memory=8192

# What Minikube does during start:
# ✅ Downloads the K8s component images
# ✅ Creates a Docker container acting as the K8s node
# ✅ Starts API Server, etcd, Scheduler, Controller Manager inside it
# ✅ Configures kubectl to talk to this cluster (kubeconfig)
```

### Essential Minikube Commands

```bash
# Check cluster status
minikube status

# Stop the cluster (saves state)
minikube stop

# Delete the cluster completely
minikube delete

# Open the Kubernetes dashboard in browser
minikube dashboard

# SSH into the Minikube node (the actual machine running K8s)
minikube ssh

# Get the cluster's IP address
minikube ip

# Pause the cluster (frees resources without deleting)
minikube pause

# List available addons
minikube addons list

# Enable useful addons
minikube addons enable metrics-server    # resource usage
minikube addons enable ingress           # HTTP routing (Day 19)
minikube addons enable dashboard         # web UI
```

---

## 5. Understanding kubeconfig — How kubectl Knows Which Cluster to Talk To

### WHAT
`kubectl` needs to know:
1. **Where** is the API Server? (URL)
2. **How** to authenticate? (certificates or tokens)
3. **Which** namespace/context to use by default?

All of this is stored in a file called **kubeconfig**, located at `~/.kube/config`.

### WHY
You'll often work with **multiple clusters** — local Minikube, a staging cluster, a production EKS cluster. kubeconfig manages all of them and lets you switch between them with one command.

### HOW

```bash
# View your entire kubeconfig
kubectl config view

# List all configured clusters
kubectl config get-clusters

# List all contexts (cluster + user + namespace combinations)
kubectl config get-contexts

# See which context you're currently using
kubectl config current-context

# Switch to a different cluster/context
kubectl config use-context minikube
kubectl config use-context my-eks-cluster

# Set a default namespace for current context (so you don't type -n every time)
kubectl config set-context --current --namespace=my-app
```

**kubeconfig structure (simplified):**
```yaml
# ~/.kube/config
apiVersion: v1
kind: Config
clusters:
  - name: minikube
    cluster:
      server: https://127.0.0.1:8443      # API Server URL
      certificate-authority: /path/to/ca.crt
  - name: my-eks-cluster
    cluster:
      server: https://abc123.eks.amazonaws.com

users:
  - name: minikube-user
    user:
      client-certificate: /path/to/client.crt
      client-key: /path/to/client.key

contexts:
  - name: minikube                         # ← context = cluster + user + namespace
    context:
      cluster: minikube
      user: minikube-user
      namespace: default

current-context: minikube                  # ← which context kubectl uses right now
```

**Teaching line:**
> "kubeconfig is like your SSH config file (`~/.ssh/config`) — it stores connection details for multiple servers so you don't type them out every time. `kubectl config use-context` is like switching which server you're SSH-ed into."

---

## 6. kubectl — Complete Mastery Guide

`kubectl` follows a consistent pattern for every command:

```
kubectl <verb> <resource-type> <resource-name> [flags]

Examples:
kubectl get     pods
kubectl get     pods        my-pod
kubectl describe deployment my-deployment
kubectl delete  service     my-service
kubectl apply   -f          my-file.yaml
```

### The Core Verbs

| Verb | What it does |
|---|---|
| `get` | List one or more resources |
| `describe` | Show detailed info about a resource |
| `apply` | Create or update resources from a YAML file |
| `create` | Create a resource (imperative — less preferred) |
| `delete` | Delete a resource |
| `edit` | Open a resource in your text editor to modify it live |
| `logs` | View logs from a Pod/container |
| `exec` | Run a command inside a running container |
| `port-forward` | Forward a local port to a Pod port |
| `rollout` | Manage deployment rollouts |
| `scale` | Scale a deployment |
| `top` | Show CPU/memory usage |

---

### 6.1 `kubectl get` — List Resources

```bash
# Get all pods in current namespace
kubectl get pods

# Get pods in a specific namespace
kubectl get pods -n kube-system

# Get pods across ALL namespaces
kubectl get pods -A

# Get with more detail (shows IPs, node assignment)
kubectl get pods -o wide

# Get in YAML format (see full spec + status)
kubectl get pod my-pod -o yaml

# Get in JSON format
kubectl get pod my-pod -o json

# Get multiple resource types at once
kubectl get pods,services,deployments

# Get all resource types in namespace
kubectl get all

# Watch for real-time changes (like tail -f)
kubectl get pods --watch
kubectl get pods -w          # shorthand

# Filter by label
kubectl get pods -l app=api
kubectl get pods -l app=api,env=production

# Get just the names (useful for scripting)
kubectl get pods -o name

# Custom columns output
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName
```

---

### 6.2 `kubectl describe` — Deep Inspection

```bash
# Describe a specific pod (events, conditions, volumes, containers)
kubectl describe pod my-pod

# Describe a node (capacity, allocated resources, running pods)
kubectl describe node minikube

# Describe a deployment
kubectl describe deployment my-api

# Describe a service
kubectl describe service my-service
```

**`describe` output includes `Events` at the bottom — this is where errors appear. Always check events when something is broken.**

```
Events:
  Type     Reason            Age    From               Message
  ----     ------            ----   ----                -------
  Warning  FailedScheduling  5m     default-scheduler   0/1 nodes available: insufficient memory
  Warning  BackOff           2m     kubelet             Back-off pulling image "myapp:wrong-tag"
  Normal   Pulled            1m     kubelet             Successfully pulled image
  Normal   Started           1m     kubelet             Started container api
```

---

### 6.3 `kubectl apply` vs `kubectl create` — Declarative vs Imperative

#### WHAT
There are two ways to create resources in Kubernetes:

**Imperative (avoid in production):**
```bash
kubectl create deployment my-api --image=nginx
kubectl expose deployment my-api --port=80
```
Direct commands. Fast for experimenting, but not reproducible or version-controllable.

**Declarative (use this always):**
```bash
kubectl apply -f deployment.yaml
kubectl apply -f ./k8s/              # apply all YAML files in a directory
kubectl apply -f https://raw.githubusercontent.com/.../file.yaml  # from URL
```
Applies YAML files that describe the desired state. Idempotent — running it again with the same file does nothing if nothing changed. Changes are visible in Git.

**Teaching line:**
> "Imperative is like giving someone verbal instructions — hard to repeat, hard to track. Declarative is like giving them a written spec — reproducible, version-controlled, and clear about intent."

---

### 6.4 `kubectl logs` — View Container Output

```bash
# Logs from a pod (if single container)
kubectl logs my-pod

# Follow logs live (like docker logs -f)
kubectl logs -f my-pod

# Last 100 lines
kubectl logs --tail=100 my-pod

# Logs from a specific container inside a multi-container pod
kubectl logs my-pod -c my-container

# Logs from previous container instance (useful after a crash)
kubectl logs my-pod --previous

# Logs from all pods matching a label
kubectl logs -l app=api --all-containers=true

# Show timestamps
kubectl logs my-pod --timestamps
```

---

### 6.5 `kubectl exec` — Run Commands Inside Containers

```bash
# Open an interactive shell inside a running container
kubectl exec -it my-pod -- bash
kubectl exec -it my-pod -- sh          # if bash not available (alpine)

# Run a single command
kubectl exec my-pod -- ls /app
kubectl exec my-pod -- env             # see environment variables
kubectl exec my-pod -- cat /etc/hosts  # check DNS config

# Exec into specific container in multi-container pod
kubectl exec -it my-pod -c api-container -- bash
```

**This is the kubectl equivalent of `docker exec` — same concept, same use case.**

---

### 6.6 `kubectl port-forward` — Access Pods Locally Without a Service

```bash
# Forward local port 8080 to pod port 3000
kubectl port-forward pod/my-pod 8080:3000

# Forward to a deployment (picks any healthy pod)
kubectl port-forward deployment/my-api 8080:3000

# Forward to a service
kubectl port-forward service/my-service 8080:80

# Keep it running in background
kubectl port-forward pod/my-pod 8080:3000 &
```

**WHY:** You can test a Pod directly from your browser/Postman without needing a Service or Ingress set up. Essential for debugging.

---

### 6.7 `kubectl delete` — Remove Resources

```bash
# Delete by name
kubectl delete pod my-pod
kubectl delete deployment my-api
kubectl delete service my-service

# Delete using the same YAML you used to create
kubectl delete -f deployment.yaml

# Delete all pods matching a label
kubectl delete pods -l app=api

# Delete everything in a namespace
kubectl delete all --all -n my-namespace

# Force delete (skip graceful shutdown — use carefully)
kubectl delete pod my-pod --force --grace-period=0
```

**Important:** If you delete a Pod that's managed by a Deployment, it comes back automatically (Controller Manager reconciles). To truly remove it, delete the Deployment.

---

### 6.8 `kubectl rollout` — Manage Deployments

```bash
# Check rollout status (is it complete? stuck?)
kubectl rollout status deployment/my-api

# See rollout history
kubectl rollout history deployment/my-api

# Roll back to previous version
kubectl rollout undo deployment/my-api

# Roll back to specific revision
kubectl rollout undo deployment/my-api --to-revision=2

# Pause a rollout (to inspect mid-deployment)
kubectl rollout pause deployment/my-api

# Resume a paused rollout
kubectl rollout resume deployment/my-api

# Trigger a fresh rollout (restart all pods) without changing the image
kubectl rollout restart deployment/my-api
```

---

### 6.9 `kubectl scale` — Change Replica Count

```bash
# Scale a deployment to 5 replicas
kubectl scale deployment my-api --replicas=5

# Scale to 0 (effectively pause/stop the app)
kubectl scale deployment my-api --replicas=0
```

---

### 6.10 `kubectl top` — Resource Usage

```bash
# CPU/memory for all nodes
kubectl top nodes

# CPU/memory for all pods
kubectl top pods

# CPU/memory for pods in specific namespace
kubectl top pods -n my-namespace

# Requires metrics-server addon: minikube addons enable metrics-server
```

---

## 7. Namespaces — Virtual Clusters Within a Cluster

### WHAT
A **namespace** is a way to divide a single Kubernetes cluster into isolated sections. Resources in one namespace don't interfere with resources in another namespace with the same name.

### WHY
- **Team isolation:** team-a and team-b each get their own namespace — their Services, ConfigMaps, etc. don't collide
- **Environment separation:** `dev`, `staging`, `production` namespaces in the same cluster
- **Resource quotas:** limit how much CPU/memory a namespace can consume
- **Access control:** RBAC can grant permissions per namespace

### HOW

```bash
# List all namespaces
kubectl get namespaces
kubectl get ns              # shorthand

# Default namespaces in every cluster:
# default       → where resources go if you don't specify a namespace
# kube-system   → K8s internal components (API Server, etcd, etc.)
# kube-public   → publicly readable config
# kube-node-lease → node heartbeat data

# Create a namespace
kubectl create namespace my-app
# OR declaratively:
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
EOF

# Run any command in a specific namespace
kubectl get pods -n my-app
kubectl apply -f deployment.yaml -n my-app

# Set default namespace for current session
kubectl config set-context --current --namespace=my-app
# Now all commands use my-app namespace without -n flag

# Delete a namespace (deletes EVERYTHING inside it!)
kubectl delete namespace my-app
```

---

## 8. Kind — Kubernetes in Docker (Alternative Setup)

### WHAT
**Kind** (Kubernetes in Docker) runs each Kubernetes node as a Docker container. It's faster to start than Minikube and easier to create multi-node clusters.

### HOW

```bash
# Install Kind
# macOS
brew install kind

# Linux
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
chmod +x kind && sudo mv kind /usr/local/bin/

# Create a single-node cluster
kind create cluster

# Create a named cluster
kind create cluster --name my-cluster

# Create a MULTI-NODE cluster (1 control plane + 3 workers)
# Create a config file: kind-config.yaml
cat > kind-config.yaml << 'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
EOF

kind create cluster --config kind-config.yaml --name multi-node

# List clusters
kind get clusters

# Load a local Docker image into Kind cluster
# (Kind can't pull from local Docker daemon directly)
kind load docker-image my-app:1.0 --name my-cluster

# Delete cluster
kind delete cluster --name my-cluster
```

**When to use Kind vs Minikube:**
| Situation | Use |
|---|---|
| Learning Kubernetes concepts | Minikube |
| Testing multi-node behaviour | Kind |
| CI/CD pipeline testing | Kind |
| Using addons (dashboard, ingress) | Minikube |
| Fastest iteration on app code | Kind |

---

## 9. Your First Real Deployment — End to End

Let's deploy a real app to your Minikube cluster using pure `kubectl`:

```bash
# Step 1: Ensure cluster is running
minikube start
kubectl get nodes              # should show: minikube   Ready

# Step 2: Deploy an nginx web server (imperative — for exploration)
kubectl create deployment nginx-demo --image=nginx:alpine --replicas=3

# Step 3: Watch pods come up
kubectl get pods -w

# Step 4: Expose it (create a Service — more on Day 15)
kubectl expose deployment nginx-demo --port=80 --type=NodePort

# Step 5: Get the URL to access it
minikube service nginx-demo --url

# Step 6: Inspect everything
kubectl get all                          # see deployment, replicaset, pods, service
kubectl describe deployment nginx-demo
kubectl describe pod <pod-name>          # copy a pod name from get pods

# Step 7: See the reconciliation loop in action
kubectl delete pod <any-pod-name>        # delete one pod
kubectl get pods -w                      # watch it come back automatically

# Step 8: Scale it
kubectl scale deployment nginx-demo --replicas=5
kubectl get pods -w                      # watch 2 more pods appear

# Step 9: Update the image (rolling update)
kubectl set image deployment/nginx-demo nginx=nginx:1.25-alpine
kubectl rollout status deployment/nginx-demo   # watch it roll

# Step 10: Roll back
kubectl rollout undo deployment/nginx-demo
kubectl rollout status deployment/nginx-demo

# Step 11: Clean up
kubectl delete deployment nginx-demo
kubectl delete service nginx-demo
```

---

## 10. Declarative Workflow — The Right Way

Now do the same thing properly — using YAML files like you would in a real project:

### `nginx-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
  namespace: default
  labels:
    app: nginx-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"       # 0.1 CPU
              memory: "64Mi"    # 64 megabytes
            limits:
              cpu: "200m"       # 0.2 CPU
              memory: "128Mi"   # 128 megabytes
```

### `nginx-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-demo
  namespace: default
spec:
  selector:
    app: nginx-demo           # routes to pods with this label
  ports:
    - port: 80
      targetPort: 80
  type: NodePort              # accessible from outside cluster
```

```bash
# Apply both files
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-service.yaml

# Or apply entire directory
kubectl apply -f ./k8s/

# See what was created
kubectl get all

# Access the app
minikube service nginx-demo --url

# Make a change: update replicas to 5 in nginx-deployment.yaml, then:
kubectl apply -f nginx-deployment.yaml      # idempotent — only changes what's different

# Clean up
kubectl delete -f nginx-deployment.yaml
kubectl delete -f nginx-service.yaml
```

---

## 11. kubectl Shortcuts and Productivity Tips

```bash
# ── Resource type shortcuts ────────────────────────────────────────────────────
po    → pods
svc   → services
deploy → deployments
rs    → replicasets
ns    → namespaces
cm    → configmaps
pv    → persistentvolumes
pvc   → persistentvolumeclaims
sa    → serviceaccounts
ep    → endpoints
no    → nodes

# Use shortcuts:
kubectl get po
kubectl get svc
kubectl get deploy

# ── Shell aliases (add to ~/.bashrc or ~/.zshrc) ───────────────────────────────
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get services'
alias kgd='kubectl get deployments'
alias kaf='kubectl apply -f'
alias kdel='kubectl delete'
alias klogs='kubectl logs -f'
alias kexec='kubectl exec -it'

# ── Auto-completion ────────────────────────────────────────────────────────────
# bash
echo 'source <(kubectl completion bash)' >> ~/.bashrc

# zsh
echo 'source <(kubectl completion zsh)' >> ~/.zshrc

# ── kubectx and kubens — fastest context/namespace switching ──────────────────
brew install kubectx         # installs both kubectx and kubens

kubectx                      # list contexts
kubectx minikube             # switch to minikube context
kubectx -                    # switch back to previous context

kubens                       # list namespaces
kubens my-app                # switch default namespace
kubens -                     # switch back to previous namespace

# ── Dry run — see what WOULD be created without actually creating ─────────────
kubectl apply -f deployment.yaml --dry-run=client
kubectl apply -f deployment.yaml --dry-run=server    # validates on server side too

# ── Output YAML for any existing resource ─────────────────────────────────────
# Great for learning: see the YAML of any object
kubectl get deployment nginx-demo -o yaml
kubectl get service nginx-demo -o yaml

# Generate a YAML template without applying it
kubectl create deployment my-app --image=nginx --dry-run=client -o yaml > deployment.yaml
kubectl create service nodeport my-app --tcp=80:80 --dry-run=client -o yaml >> service.yaml

# ── Explain any field in a resource ───────────────────────────────────────────
kubectl explain pod
kubectl explain pod.spec
kubectl explain pod.spec.containers
kubectl explain deployment.spec.strategy
```

---

## 12. Debugging Toolkit — What to Do When Things Go Wrong

```bash
# Pod not starting? — Check events
kubectl describe pod <pod-name>          # scroll to Events section at bottom

# Common events and what they mean:
# "Back-off pulling image"     → wrong image name/tag, or registry auth issue
# "insufficient cpu/memory"    → node doesn't have enough resources
# "0/1 nodes available"        → scheduling failed — check constraints
# "CrashLoopBackOff"           → container starts then exits — check logs

# Container crashing?
kubectl logs <pod-name>                  # see stdout/stderr
kubectl logs <pod-name> --previous       # logs from the PREVIOUS crashed instance
kubectl logs <pod-name> -f               # follow live

# Need to debug inside the pod?
kubectl exec -it <pod-name> -- sh

# Pod status meanings:
# Pending      → waiting to be scheduled or image pull in progress
# Running      → at least one container running
# Completed    → all containers exited successfully
# CrashLoopBackOff → container repeatedly crashing
# OOMKilled    → container exceeded memory limit (increase limits)
# ImagePullBackOff → can't pull image (wrong name, no credentials)
# Error        → container exited with non-zero exit code
# Terminating  → being deleted (stuck? use --force --grace-period=0)

# Node issues?
kubectl describe node <node-name>        # check Conditions and Events
kubectl top nodes                        # check resource pressure

# Something seems broken but no obvious error?
kubectl get events --sort-by='.lastTimestamp'    # all events, newest last
kubectl get events -n my-namespace --watch       # watch events live
```

---

## 13. Hands-On Practice Tasks

1. Start Minikube: `minikube start --driver=docker`
2. View your kubeconfig: `kubectl config view` — identify the cluster, user, and context
3. Explore system components: `kubectl get pods -n kube-system` — identify every component from Day 11
4. Run the full nginx deployment workflow from Section 9 (imperative)
5. Delete a pod manually and watch the reconciliation loop bring it back
6. Scale to 5 and back to 1 — watch pods appear and disappear with `-w`
7. Write the YAML files from Section 10 by hand, apply them, and use `kubectl get all` to verify
8. Use `kubectl get deployment nginx-demo -o yaml` to see the full spec Kubernetes generated
9. Try `kubectl explain deployment.spec.strategy` to see field documentation
10. Enable the dashboard: `minikube addons enable dashboard && minikube dashboard`

---

## 14. Quiz — Test Yourself / Test Others

1. What is kubeconfig and what three things does it store per cluster?
2. What is the difference between `kubectl apply` and `kubectl create`? Which should you prefer and why?
3. Why would you use `kubectl port-forward` instead of creating a Service?
4. If you `kubectl delete pod my-pod` and it's part of a Deployment, what happens?
5. What does `kubectl rollout undo deployment/my-api` do?
6. What is a Kubernetes namespace? Give two real use cases.
7. What is the difference between Minikube and Kind? When would you use each?
8. You run `kubectl get pods` and see a pod in `CrashLoopBackOff` status. What are your next two debugging steps?
9. What does `--dry-run=client -o yaml` do and why is it useful?
10. What is the difference between `kubectl logs my-pod` and `kubectl logs my-pod --previous`?

<details>
<summary>Answers</summary>

1. kubeconfig stores: (1) clusters — API Server URL and certificate for each cluster, (2) users — credentials (cert/key or token) for authenticating, (3) contexts — named combinations of cluster + user + default namespace. It lives at `~/.kube/config`.

2. `kubectl create` is imperative — you describe what action to take (create this). `kubectl apply` is declarative — you describe the desired state and Kubernetes figures out what to change. Prefer `apply` because it's idempotent, version-controllable in YAML files, and works for both creating and updating resources.

3. `port-forward` lets you access a Pod directly from your local machine without setting up a Service or exposing anything externally — ideal for debugging, testing specific pods, or accessing admin UIs temporarily without permanent exposure.

4. The pod is deleted, but the Deployment's Controller Manager detects the actual replica count (e.g., 2) is below desired (3) and immediately creates a replacement pod. You cannot permanently remove a pod by deleting it if a Deployment manages it — you must delete the Deployment.

5. It reverts the Deployment to the previous version — restoring the previous ReplicaSet and triggering a rolling update back to the previous image/configuration. K8s keeps a history of revisions for exactly this purpose.

6. A namespace is a virtual cluster within a cluster that isolates resources. Use cases: (1) Team isolation — team-a and team-b each have their own namespace so their Services/ConfigMaps don't collide. (2) Environment separation — `dev`, `staging`, `prod` namespaces in the same cluster with different resource quotas and access controls.

7. Minikube runs K8s in a single VM or Docker container and has built-in addons (dashboard, ingress, metrics) — best for learning and matching real cluster behaviour. Kind runs K8s nodes as Docker containers, starts faster, and easily supports multi-node clusters — best for CI/CD pipelines and testing multi-node scenarios.

8. First: `kubectl logs <pod-name> --previous` — check logs from the crashed container instance to see the error. Second: `kubectl describe pod <pod-name>` — check the Events section for scheduling issues, image pull failures, or exit codes.

9. `--dry-run=client -o yaml` shows what YAML Kubernetes would create/apply without actually doing it. Useful for: generating YAML templates from imperative commands, validating syntax before applying, and learning the full YAML structure of any resource type.

10. `kubectl logs my-pod` shows logs from the currently running container. `kubectl logs my-pod --previous` shows logs from the previous container instance — essential after a crash (CrashLoopBackOff) because the current container may have just started with no logs yet, but the previous one contains the error that caused the crash.

</details>

---

## 15. Summary

After today you can:
- Install and start a local Kubernetes cluster with Minikube or Kind
- Understand kubeconfig — what it stores and how to switch between clusters
- Use every core kubectl verb: `get`, `describe`, `apply`, `delete`, `logs`, `exec`, `port-forward`, `rollout`, `scale`, `top`
- Work with namespaces — create, switch, isolate resources
- Write and apply declarative YAML files — the right way to manage K8s resources
- Use `kubectl` productivity tools: shortcuts, aliases, auto-completion, kubectx/kubens
- Debug the full range of Pod failure states with a systematic approach
- Understand why declarative (`apply`) beats imperative (`create`) for production workflows

**Coming up on Day 13:** Pods — the smallest deployable unit in Kubernetes. You've been creating Pods indirectly via Deployments. Tomorrow you go deep on what a Pod actually is, multi-container Pod patterns (sidecar, init containers, ambassador), Pod lifecycle, and why you almost never create bare Pods in production.
