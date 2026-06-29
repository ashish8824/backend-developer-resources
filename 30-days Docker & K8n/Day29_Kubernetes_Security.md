# Day 29: Kubernetes Security — RBAC, Pod Security & Secrets Hardening

**Goal for today:** Understand the complete Kubernetes security model — RBAC (who can do what in the cluster), ServiceAccounts (identity for Pods), Pod Security Standards (restricting what Pods can do on nodes), and secrets management best practices. By the end, you should be able to implement a defence-in-depth security posture for any production Kubernetes cluster.

**Format:** WHAT → WHY → HOW for every concept.

---

## 0. Quick Recap (open your teaching session with this)

- Day 16: ConfigMaps & Secrets — base64 encoding, not encryption
- Day 25: NetworkPolicies — restricting pod-to-pod communication
- Day 28: IRSA — IAM roles for pods (no static credentials)

**Today's connecting thought:**
> Security in Kubernetes has four layers: what can reach your Pods (NetworkPolicies — Day 25), what AWS permissions your Pods have (IRSA — Day 28), what Kubernetes API actions users and Pods can perform (RBAC — today), and what your Pods themselves can do on the underlying node (Pod Security Standards — today). Today completes the security picture.

---

## 1. The Kubernetes Security Model — Four Layers

```
┌─────────────────────────────────────────────────────────────────────┐
│             KUBERNETES SECURITY — DEFENCE IN DEPTH                   │
├─────────────────────────────────────────────────────────────────────┤
│  Layer 1: NETWORK      (Who can reach my Pods?)                      │
│           NetworkPolicies — covered Day 25                           │
├─────────────────────────────────────────────────────────────────────┤
│  Layer 2: CLOUD IAM    (What AWS/cloud resources can Pods access?)   │
│           IRSA / Workload Identity — covered Day 28                  │
├─────────────────────────────────────────────────────────────────────┤
│  Layer 3: K8S RBAC     (Who can do what in the K8s API?)             │
│           Roles, ClusterRoles, RoleBindings — TODAY                  │
├─────────────────────────────────────────────────────────────────────┤
│  Layer 4: POD SECURITY (What can a Pod do on the host node?)         │
│           Pod Security Standards, securityContext — TODAY            │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. RBAC — Role-Based Access Control

### WHAT
**RBAC** controls who (users, groups, ServiceAccounts) can perform which actions (verbs) on which Kubernetes API resources (pods, deployments, secrets, etc.) in which namespaces.

### WHY
Without RBAC:
- Any developer can delete production Pods
- Any Pod can read all Secrets in the cluster (including other namespaces)
- A compromised CI/CD pipeline can modify any resource
- No audit trail of who changed what

With RBAC:
- Developers can deploy to staging but only read production
- Pods only have access to their own namespace's resources
- CI/CD pipelines have exactly the permissions they need — nothing more

### The Four RBAC Objects

| Object | Scope | Purpose |
|---|---|---|
| `Role` | Namespace | Defines permissions within one namespace |
| `ClusterRole` | Cluster-wide | Defines permissions across all namespaces or for cluster-scoped resources |
| `RoleBinding` | Namespace | Grants a Role to a user/group/ServiceAccount in one namespace |
| `ClusterRoleBinding` | Cluster-wide | Grants a ClusterRole cluster-wide |

```
WHO (Subject)          WHAT (Role/ClusterRole)     WHERE (Namespace)
─────────────────────────────────────────────────────────────────────
User: ashish           Role: developer              Namespace: staging
ServiceAccount: api    Role: pod-reader             Namespace: production
Group: team-backend    ClusterRole: read-only       Cluster-wide
```

---

## 3. Roles and ClusterRoles — Defining Permissions

### Role (namespace-scoped)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-role
  namespace: staging          # permissions only in 'staging' namespace
rules:
  # ── Rule 1: Full access to deployments and pods ──────────────────────────
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  - apiGroups: [""]            # "" = core API group (pods, services, etc.)
    resources: ["pods", "pods/log", "pods/exec"]
    verbs: ["get", "list", "watch", "create", "delete"]

  # ── Rule 2: Read-only access to services and configmaps ───────────────────
  - apiGroups: [""]
    resources: ["services", "configmaps"]
    verbs: ["get", "list", "watch"]

  # ── Rule 3: NO access to secrets (security) ───────────────────────────────
  # (simply don't add secrets to the rules)

  # ── Rule 4: Access to ingresses ───────────────────────────────────────────
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
```

### Common API Groups Reference

```
""                        → core: pods, services, configmaps, secrets, pvc, sa, nodes
"apps"                    → deployments, replicasets, statefulsets, daemonsets
"batch"                   → jobs, cronjobs
"networking.k8s.io"       → ingresses, networkpolicies
"rbac.authorization.k8s.io" → roles, clusterroles, rolebindings
"storage.k8s.io"          → storageclasses, persistentvolumes
"autoscaling"             → horizontalpodautoscalers
"policy"                  → poddisruptionbudgets
```

### Common Verbs Reference

```
"get"     → read one resource by name
"list"    → read all resources of a type
"watch"   → stream updates to resources
"create"  → create new resources
"update"  → full update (PUT)
"patch"   → partial update (PATCH)
"delete"  → delete a resource
"exec"    → kubectl exec into a pod (pods/exec sub-resource)
"log"     → kubectl logs (pods/log sub-resource)
```

### ClusterRole (cluster-wide)

```yaml
# ClusterRole for Prometheus — needs to read all pods in any namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-reader
rules:
  - apiGroups: [""]
    resources:
      - nodes
      - nodes/proxy
      - nodes/metrics
      - services
      - endpoints
      - pods
    verbs: ["get", "list", "watch"]

  - apiGroups: ["extensions", "networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list", "watch"]

  - nonResourceURLs: ["/metrics"]    # HTTP path (not K8s resource)
    verbs: ["get"]
```

```yaml
# ClusterRole for read-only cluster access
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-viewer
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["get", "list", "watch"]
  # No create/update/delete/patch
```

---

## 4. RoleBindings and ClusterRoleBindings

### RoleBinding — Grant permissions in one namespace

```yaml
# Grant developer-role to a user in staging namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ashish-developer-binding
  namespace: staging
subjects:
  # Subject type 1: A human user
  - kind: User
    name: ashish@company.com    # must match the username in kubeconfig
    apiGroup: rbac.authorization.k8s.io

  # Subject type 2: A group
  - kind: Group
    name: backend-team
    apiGroup: rbac.authorization.k8s.io

  # Subject type 3: A ServiceAccount (for Pods)
  - kind: ServiceAccount
    name: ci-cd-sa
    namespace: staging          # ServiceAccount namespace
roleRef:
  kind: Role                    # Role or ClusterRole
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRoleBinding — Grant permissions cluster-wide

```yaml
# Grant Prometheus the cluster-wide reader role
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus-binding
subjects:
  - kind: ServiceAccount
    name: prometheus
    namespace: monitoring
roleRef:
  kind: ClusterRole
  name: prometheus-reader
  apiGroup: rbac.authorization.k8s.io
```

### Useful trick: RoleBinding referencing a ClusterRole

```yaml
# Grant a ClusterRole but only in one namespace
# (ClusterRole defines permissions, RoleBinding scopes them to a namespace)
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: view-production
  namespace: production        # scoped to production only
subjects:
  - kind: Group
    name: developers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole            # reference ClusterRole (not Role)
  name: view                   # built-in ClusterRole: read-only access
  apiGroup: rbac.authorization.k8s.io
```

---

## 5. Built-in ClusterRoles — Don't Reinvent the Wheel

```bash
kubectl get clusterroles | grep -v system

# Most important built-in ClusterRoles:
# cluster-admin  → God mode: full access to everything
# admin          → Full namespace access (read + write everything except resource quotas)
# edit           → Read/write most resources, no access to roles/rolebindings
# view           → Read-only access to most namespace resources (no secrets)
```

```yaml
# Use built-in roles when they fit
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-view-prod
  namespace: production
subjects:
  - kind: Group
    name: developers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view          # built-in: read-only, no secrets access
  apiGroup: rbac.authorization.k8s.io
```

---

## 6. ServiceAccounts — Pod Identity

### WHAT
A **ServiceAccount** is a Kubernetes identity for Pods (not humans). Every Pod runs with a ServiceAccount that determines what Kubernetes API resources it can access.

### WHY
Pods often need to interact with the Kubernetes API — for example:
- Prometheus reads pod/node metrics via the API
- CI/CD deployments need to create/update Deployments
- Admission webhooks need to watch resources
- Your application might need to list ConfigMaps

Without ServiceAccounts, Pods would either have no API access (safe but limiting) or use the default ServiceAccount (which may have too many permissions).

### The Default ServiceAccount Problem

```bash
# Every namespace has a 'default' ServiceAccount
kubectl get serviceaccounts -n production

# By default, ALL pods use the 'default' ServiceAccount unless specified
# The default ServiceAccount has minimal permissions — but it still mounts
# a token into every Pod at /var/run/secrets/kubernetes.io/serviceaccount/
# This token can be used to talk to the K8s API

# Best practice: disable auto-mounting for the default SA
kubectl patch serviceaccount default -n production \
  -p '{"automountServiceAccountToken": false}'
```

### Creating a Dedicated ServiceAccount

```yaml
# ServiceAccount with RBAC for Prometheus
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring
  annotations:
    # For IRSA (AWS EKS) — attach IAM role
    eks.amazonaws.com/role-arn: arn:aws:iam::123:role/prometheus-role
automountServiceAccountToken: true   # explicitly enable (disabled by default in K8s 1.24+)

---
# Role for Prometheus
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
  - apiGroups: [""]
    resources: ["nodes", "pods", "services", "endpoints"]
    verbs: ["get", "list", "watch"]
  - nonResourceURLs: ["/metrics"]
    verbs: ["get"]

---
# Bind the Role to the ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
subjects:
  - kind: ServiceAccount
    name: prometheus
    namespace: monitoring
roleRef:
  kind: ClusterRole
  name: prometheus
  apiGroup: rbac.authorization.k8s.io
```

```yaml
# Reference the ServiceAccount in a Deployment
spec:
  template:
    spec:
      serviceAccountName: prometheus
      automountServiceAccountToken: true
      containers:
        - name: prometheus
          image: prom/prometheus:v2.48.0
```

### Minimal Permissions ServiceAccount for Your API

```yaml
# Your API probably doesn't need to talk to K8s API at all
apiVersion: v1
kind: ServiceAccount
metadata:
  name: taskapi-sa
  namespace: production
automountServiceAccountToken: false   # don't mount K8s API token
                                       # reduces attack surface

---
# In Deployment:
spec:
  template:
    spec:
      serviceAccountName: taskapi-sa
      automountServiceAccountToken: false
```

---

## 7. RBAC Debugging

```bash
# ── Check what permissions a user/SA has ─────────────────────────────────────
# Can I list pods in production namespace?
kubectl auth can-i list pods -n production

# Can a specific ServiceAccount do something?
kubectl auth can-i get secrets -n production \
  --as=system:serviceaccount:production:taskapi-sa

# Can the CI/CD user create deployments?
kubectl auth can-i create deployments -n production \
  --as=github-actions

# ── List all roles and bindings ──────────────────────────────────────────────
kubectl get roles,rolebindings -n production
kubectl get clusterroles,clusterrolebindings | grep -v system

# ── See all permissions for a role ───────────────────────────────────────────
kubectl describe role developer-role -n staging
kubectl describe clusterrole prometheus

# ── Who has access to what (audit) ───────────────────────────────────────────
# Which subjects are bound to a role?
kubectl describe rolebinding ashish-developer-binding -n staging

# What ClusterRoleBindings exist for a user?
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.subjects[]?.name == "github-actions") | .metadata.name'

# ── Common error messages ─────────────────────────────────────────────────────
# Error: "User cannot list resource 'pods' in API group '' in the namespace 'production'"
# → Need to add: { resources: ["pods"], verbs: ["list"], apiGroups: [""] }

# Error: "Error from server (Forbidden): pods is forbidden"
# → Check RoleBinding subject name matches exactly
# → Check the ServiceAccount namespace in RoleBinding
```

---

## 8. Pod Security Standards — Restricting Pod Capabilities

### WHAT
**Pod Security Standards (PSS)** define three security profiles that restrict what Pods can do on the host node:

| Profile | Use case | What it restricts |
|---|---|---|
| `Privileged` | Trusted system workloads | Nothing — all capabilities allowed |
| `Baseline` | Most application workloads | Blocks most dangerous capabilities |
| `Restricted` | Maximum security | Enforces security best practices strictly |

### WHY
Without Pod Security Standards:
- A Pod could run as root and escape to the host
- A Pod could mount the host filesystem and read all data
- A Pod could gain all Linux capabilities (NET_ADMIN, SYS_ADMIN, etc.)
- A compromised Pod = compromised node = compromised cluster

### Enabling Pod Security Standards via Namespace Labels

```bash
# Apply to a namespace (three modes: enforce, audit, warn)
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted
```

**Three modes:**
- `enforce` — pods that violate are REJECTED (admission blocked)
- `audit` — violations logged but pods allowed
- `warn` — user gets a warning but pods allowed

**Applying profiles gradually:**
```bash
# Step 1: Start with audit (see what would fail without breaking anything)
kubectl label namespace production \
  pod-security.kubernetes.io/audit=restricted

# Step 2: Add warnings for developers
kubectl label namespace production \
  pod-security.kubernetes.io/warn=restricted

# Step 3: Enforce (once you've fixed all violations)
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted
```

---

## 9. securityContext — Pod and Container Security Settings

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: taskapi
  namespace: production
spec:
  template:
    spec:
      # ── Pod-level security context ──────────────────────────────────────────
      securityContext:
        runAsNonRoot: true          # MUST run as non-root user
        runAsUser: 1000             # run as UID 1000
        runAsGroup: 3000            # run as GID 3000
        fsGroup: 2000               # files in volumes owned by GID 2000
        seccompProfile:
          type: RuntimeDefault      # use default seccomp profile (blocks dangerous syscalls)

      containers:
        - name: api
          image: taskapi:2.0
          # ── Container-level security context ─────────────────────────────────
          securityContext:
            allowPrivilegeEscalation: false   # cannot gain more privileges than parent
            readOnlyRootFilesystem: true        # container's root FS is read-only
            capabilities:
              drop:
                - ALL               # drop ALL Linux capabilities
              add:
                - NET_BIND_SERVICE  # only add back what's needed (bind port < 1024)

          # Since root FS is read-only, need writable volume for temp files
          volumeMounts:
            - name: tmp-dir
              mountPath: /tmp
            - name: app-logs
              mountPath: /app/logs

      volumes:
        - name: tmp-dir
          emptyDir: {}
        - name: app-logs
          emptyDir: {}
```

### What Each Setting Does

```
runAsNonRoot: true
  → Rejects any container whose image runs as UID 0 (root)
  → Pod fails to start if image doesn't specify a non-root USER

runAsUser: 1000
  → Force the container process to run as UID 1000
  → Overrides the USER in the Dockerfile

allowPrivilegeEscalation: false
  → Prevents processes from gaining more privileges than their parent
  → Blocks setuid binaries from elevating to root
  → Always set to false unless you have a specific need

readOnlyRootFilesystem: true
  → Makes the container's filesystem immutable at runtime
  → Attacker can't write malware, modify configs, or tamper with binaries
  → Requires explicit writable volumes for /tmp, logs, etc.

capabilities.drop: [ALL]
  → Removes ALL Linux capabilities (NET_ADMIN, SYS_ADMIN, etc.)
  → Dramatically reduces what a compromised container can do
  → Add back only what's genuinely needed

seccompProfile.type: RuntimeDefault
  → Applies a filter on Linux system calls the container can make
  → Blocks ~300+ dangerous syscalls by default
  → RuntimeDefault uses the container runtime's default profile
```

---

## 10. The Restricted Profile — What it Requires

For a Pod to pass the `restricted` Pod Security Standard, it MUST:

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault       # or Localhost
  containers:
    - name: app
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop: ["ALL"]
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      # volumes: only emptyDir, configMap, secret, projected, ephemeral allowed
      # No hostPath, no hostNetwork, no hostPID, no hostIPC
```

---

## 11. Secrets Management Best Practices — The Full Picture

### Problem recap from Day 16

```
Kubernetes Secrets are:
  ✅ Base64-encoded (not plaintext in YAML)
  ❌ NOT encrypted by default in etcd
  ❌ Visible to anyone with RBAC access to secrets
  ❌ Stored permanently in Git if you commit them

Solutions in order of security:
  Level 1: Basic Kubernetes Secrets (acceptable for dev)
  Level 2: RBAC restrictions on Secret access
  Level 3: etcd encryption at rest
  Level 4: External Secrets (AWS Secrets Manager / Vault)
  Level 5: Sealed Secrets (encrypted, Git-safe)
```

### Level 2: Restrict Secret Access with RBAC

```yaml
# Most pods should NOT have access to Secrets
# Only give secret access to specific service accounts that need it

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["taskapi-secrets"]  # only THIS specific secret, not all secrets
    verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: taskapi-secret-access
  namespace: production
subjects:
  - kind: ServiceAccount
    name: taskapi-sa
    namespace: production
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

### Level 3: etcd Encryption at Rest

```yaml
# encryption-config.yaml (applied to API Server)
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:                    # AES-CBC encryption
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>
      - identity: {}               # fallback for unencrypted existing secrets
```

On AWS EKS:
```bash
# Enable envelope encryption for Secrets using KMS
eksctl utils enable-secrets-encryption \
  --cluster taskapi-prod \
  --key-arn arn:aws:kms:ap-south-1:123:key/abc \
  --region ap-south-1
```

### Level 4: External Secrets from AWS Secrets Manager

Covered in Day 28 — the gold standard for production.

### Level 5: Sealed Secrets (GitOps-safe)

```bash
# Install Sealed Secrets controller
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system

# Install kubeseal CLI
brew install kubeseal

# Create a regular Secret YAML (don't apply yet)
kubectl create secret generic db-secret \
  --from-literal=DB_PASSWORD=supersecret \
  --dry-run=client -o yaml > secret.yaml

# Encrypt it into a SealedSecret (safe to commit to Git)
kubeseal --format yaml < secret.yaml > sealed-secret.yaml
# sealed-secret.yaml is encrypted with the cluster's public key
# Only the cluster can decrypt it

# Commit sealed-secret.yaml to Git — completely safe
git add sealed-secret.yaml
git commit -m "add db sealed secret"

# Apply to cluster — controller decrypts into real Secret
kubectl apply -f sealed-secret.yaml
```

---

## 12. Complete Security Hardening Checklist

```
RBAC:
  [ ] No user or SA has cluster-admin unless absolutely necessary
  [ ] CI/CD pipeline has minimal permissions (create/update deployments only)
  [ ] Pods use dedicated ServiceAccounts (not default)
  [ ] automountServiceAccountToken: false on pods that don't need K8s API access
  [ ] Secret access restricted by resourceNames to specific secrets
  [ ] Regular audit: kubectl auth can-i --list --as=system:serviceaccount:ns:sa

POD SECURITY:
  [ ] runAsNonRoot: true on all containers
  [ ] allowPrivilegeEscalation: false on all containers
  [ ] capabilities.drop: [ALL] on all containers
  [ ] readOnlyRootFilesystem: true (with explicit writable volumes)
  [ ] seccompProfile: RuntimeDefault
  [ ] Pod Security Standards enforced on production namespace
  [ ] No hostPath volumes in production (except DaemonSets)
  [ ] No hostNetwork, hostPID, hostIPC in production

SECRETS:
  [ ] No secrets in Git (even base64)
  [ ] etcd encryption at rest enabled
  [ ] Secrets accessed via IRSA/ExternalSecrets where possible
  [ ] RBAC limits which SAs can read which secrets
  [ ] Secret rotation policy defined

NETWORK:
  [ ] Default-deny NetworkPolicies in production
  [ ] Explicit allow rules for each communication path
  [ ] Ingress restricted to known sources
  [ ] Egress restricted to known destinations + DNS

IMAGES:
  [ ] All images scanned with Trivy in CI (covered Day 27)
  [ ] No latest tags in production
  [ ] Images from private registry only (ECR)
  [ ] Pulling from trusted base images only
```

---

## 13. Admission Controllers — The Last Line of Defence

### WHAT
**Admission Controllers** intercept API Server requests BEFORE they're persisted to etcd. They can validate (reject invalid requests) or mutate (modify) incoming resources.

### Built-in admission controllers to know

```
NamespaceLifecycle    → prevents creation in terminating namespaces
ResourceQuota         → enforces ResourceQuota limits
LimitRanger           → enforces LimitRange defaults
ServiceAccount        → auto-creates and assigns service accounts
PodSecurity           → enforces Pod Security Standards
MutatingAdmissionWebhook  → custom mutation via webhooks
ValidatingAdmissionWebhook → custom validation via webhooks
```

### OPA Gatekeeper — Policy as Code

```yaml
# Gatekeeper ConstraintTemplate — define a custom policy
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items: {type: string}
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels
        violation[{"msg": msg}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("Missing required labels: %v", [missing])
        }

---
# Apply the constraint: all Deployments must have 'team' and 'app' labels
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: deployment-must-have-labels
spec:
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
  parameters:
    labels: ["app", "team"]
```

---

## 14. Audit Logging — Who Did What

Enable API Server audit logging to track every action:

```yaml
# audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log all secret access at RequestResponse level (full details)
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["secrets"]

  # Log pod creation/deletion
  - level: Request
    verbs: ["create", "delete"]
    resources:
      - group: ""
        resources: ["pods"]

  # Log all cluster-admin actions
  - level: RequestResponse
    users: ["cluster-admin"]

  # Log nothing for health checks (too noisy)
  - level: None
    nonResourceURLs:
      - "/healthz"
      - "/readyz"
      - "/livez"
```

On EKS:
```bash
# Enable audit logging to CloudWatch
eksctl utils update-cluster-logging \
  --cluster taskapi-prod \
  --region ap-south-1 \
  --enable-types audit,api,authenticator
```

---

## 15. Hands-On Practice Tasks

```bash
# 1. Create namespace with restricted Pod Security Standard
kubectl create namespace secure-demo
kubectl label namespace secure-demo \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted

# 2. Try deploying a Pod running as root — should be rejected
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: root-pod
  namespace: secure-demo
spec:
  containers:
    - name: app
      image: nginx:alpine
EOF
# Should fail: "Error: violates PodSecurity"

# 3. Deploy a correctly secured Pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  namespace: secure-demo
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: nginx:alpine
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
      volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /var/cache/nginx
        - name: run
          mountPath: /var/run
  volumes:
    - name: tmp
      emptyDir: {}
    - name: cache
      emptyDir: {}
    - name: run
      emptyDir: {}
EOF

# 4. Check what YOU can do in a namespace
kubectl auth can-i --list -n production

# 5. Create a ServiceAccount with limited permissions
kubectl create serviceaccount limited-sa -n default
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n default
kubectl create rolebinding limited-binding \
  --role=pod-reader \
  --serviceaccount=default:limited-sa -n default

# 6. Test the limited SA's permissions
kubectl auth can-i list pods -n default \
  --as=system:serviceaccount:default:limited-sa
# Should be: yes

kubectl auth can-i delete pods -n default \
  --as=system:serviceaccount:default:limited-sa
# Should be: no

kubectl auth can-i get secrets -n default \
  --as=system:serviceaccount:default:limited-sa
# Should be: no

# 7. Disable auto-mount on default SA
kubectl patch serviceaccount default -n default \
  -p '{"automountServiceAccountToken": false}'

# 8. Clean up
kubectl delete namespace secure-demo
```

---

## 16. Quiz — Test Yourself / Test Others

1. What are the four Kubernetes RBAC objects and how do they relate to each other?
2. What is the difference between a Role and a ClusterRole?
3. What is the difference between a RoleBinding and a ClusterRoleBinding?
4. What is a ServiceAccount and why should Pods not use the default one?
5. What do the three Pod Security Standard profiles (Privileged, Baseline, Restricted) restrict?
6. What does `allowPrivilegeEscalation: false` prevent?
7. What does `readOnlyRootFilesystem: true` protect against?
8. What does `capabilities.drop: [ALL]` do and why is it important?
9. What is the difference between `audit`, `warn`, and `enforce` modes for Pod Security Standards?
10. A Pod needs to read Secrets from Kubernetes API. What is the correct RBAC setup and what security precautions should you take?

<details>
<summary>Answers</summary>

1. Role: defines permissions (verbs on resources) within one namespace. ClusterRole: defines permissions cluster-wide or for cluster-scoped resources. RoleBinding: grants a Role or ClusterRole to a subject (user/group/SA) within one namespace. ClusterRoleBinding: grants a ClusterRole to a subject cluster-wide. The pattern is: define permissions in a Role/ClusterRole, then grant them to subjects via RoleBinding/ClusterRoleBinding.

2. A Role is namespace-scoped — it defines what API resources and verbs are allowed within a specific namespace. A ClusterRole is cluster-scoped — it can define permissions for resources across all namespaces OR for cluster-scoped resources like Nodes, PersistentVolumes, and StorageClasses. A Role cannot grant access to cluster-scoped resources; only a ClusterRole can.

3. A RoleBinding grants a Role or ClusterRole to subjects but scopes the permissions to a single namespace — even if a ClusterRole is referenced, the permissions only apply in that namespace. A ClusterRoleBinding grants a ClusterRole to subjects cluster-wide — the subject gets those permissions in all namespaces and for all cluster-scoped resources. Use RoleBinding to restrict scope; use ClusterRoleBinding for truly cluster-wide access.

4. A ServiceAccount is a Kubernetes identity for Pods (not humans), used to authenticate to the Kubernetes API Server. Every Pod runs under a ServiceAccount. The default ServiceAccount is shared by all Pods that don't specify one — if it's compromised or misconfigured, all pods are affected. Dedicated ServiceAccounts follow least-privilege: each service gets only the minimal permissions it needs, and a compromise affects only that service's access.

5. Privileged: no restrictions — all capabilities allowed, host network/PID/IPC allowed, any volume type, any UID. Used only for trusted infrastructure (DaemonSets like node exporters). Baseline: blocks the most dangerous capabilities (like NET_ADMIN, SYS_ADMIN), prohibits hostPath, hostNetwork, privileged containers, and dangerous volume types. Appropriate for general application workloads. Restricted: enforces all security best practices — non-root user required, all capabilities dropped, privilege escalation blocked, seccomp profile required, read-only root filesystem encouraged, limited volume types.

6. `allowPrivilegeEscalation: false` prevents a process inside the container from gaining more privileges than its parent process. Specifically, it blocks execution of setuid and setgid binaries that could elevate a process to root. Without this, even a non-root process could run a setuid binary (like su or sudo) to become root, defeating the `runAsNonRoot` protection.

7. `readOnlyRootFilesystem: true` makes the container's entire filesystem immutable at runtime — nothing can be written to the container filesystem. This prevents an attacker who gains code execution from: writing malware or backdoors, modifying binaries or configurations, persisting changes that survive container restarts. The attacker can run code but can't permanently alter the container's state. Legitimate writable paths must be explicitly provided via emptyDir volumes.

8. `capabilities.drop: [ALL]` removes all Linux capabilities from the container process. Linux capabilities are fine-grained privileges beyond basic file permissions — things like NET_ADMIN (configure network interfaces), SYS_ADMIN (mount filesystems, many other dangerous ops), SYS_PTRACE (debug other processes), and ~38 others. By dropping all, you prevent a compromised container from performing privileged OS operations even if it runs as root or escapes its user context. You then add back only what's specifically needed.

9. `enforce`: pods that violate the policy are rejected at admission — they cannot be created. `warn`: pods are created but the user receives a warning message — useful for informing developers of future violations. `audit`: violations are recorded in audit logs but pods are created normally — useful for discovering violations without breaking existing workloads. The recommended migration path: start with audit → add warn → then enforce once all violations are fixed.

10. Create a dedicated ServiceAccount (`kubectl create serviceaccount secret-reader-sa`). Create a Role with only `get` verb on `secrets` resource, scoped to specific `resourceNames` (only the secrets it actually needs, not all secrets). Create a RoleBinding linking the Role to the ServiceAccount. In the Pod/Deployment spec, set `serviceAccountName: secret-reader-sa`. Security precautions: use `resourceNames` to limit to specific secrets, never grant `list` on secrets (returns all secret data), enable etcd encryption, audit secret access via audit logs, prefer External Secrets Operator over direct K8s Secret access.

</details>

---

## 17. Summary

After today you can explain and implement the complete Kubernetes security model:

- **RBAC** — Roles, ClusterRoles, RoleBindings, ClusterRoleBindings — the four objects and how they compose
- **API groups and verbs** — how to write precise permission rules
- **Built-in ClusterRoles** — cluster-admin, admin, edit, view
- **ServiceAccounts** — pod identity, dedicated SAs per service, disabling auto-mount
- **RBAC debugging** — `kubectl auth can-i`, listing bindings, common error messages
- **Pod Security Standards** — Privileged, Baseline, Restricted profiles
- **securityContext** — runAsNonRoot, allowPrivilegeEscalation, readOnlyRootFilesystem, capabilities.drop, seccompProfile
- **PSS namespace labels** — enforce/audit/warn modes, gradual adoption
- **Secrets hardening** — RBAC restrictions, etcd encryption, External Secrets, Sealed Secrets
- **Admission Controllers** — what they do, OPA Gatekeeper for policy-as-code
- **Audit logging** — tracking who did what, EKS CloudWatch integration

**Coming up on Day 30:** The Final Capstone — deploy a complete, production-grade application with everything from Days 1–29: Docker, Kubernetes, Helm, EKS, CI/CD, monitoring, networking, and security — all woven together into one deployable system you can take into any job interview or production environment.
