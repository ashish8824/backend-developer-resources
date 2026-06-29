# Day 16: ConfigMaps & Secrets — Externalising Configuration in Kubernetes

**Goal for today:** Understand why configuration must be separated from application code, what ConfigMaps and Secrets are, every way to create and consume them (env vars, volume mounts, command args), how Secrets are stored and protected, and the production patterns for managing sensitive data. By the end, you should be able to fully externalise any application's configuration in Kubernetes following 12-factor app principles.

**Format:** WHAT → WHY → HOW for every concept.

---

## 0. Quick Recap (open your teaching session with this)

- Day 13: Pods — spec, health probes, multi-container patterns
- Day 14: Deployments — rolling updates, scaling, rollback
- Day 15: Services — ClusterIP, NodePort, LoadBalancer, DNS, kube-proxy

**Today's connecting thought:**
> Every app we've deployed so far has had its configuration either hardcoded in the image (bad) or passed as raw env vars in the Pod spec (acceptable but not scalable). In production, you have dozens of services, multiple environments (dev/staging/prod), and sensitive data like DB passwords and API keys. Today we learn the Kubernetes-native way to manage all of this cleanly, securely, and without touching your images.

---

## 1. The Problem — Why Configuration Needs to Be Separate

### The 12-Factor App Principle
The [12-Factor App](https://12factor.net/) methodology (the industry standard for cloud-native apps) states:

> **Factor III: Store config in the environment.** Config is everything that is likely to vary between deploys (staging, production, developer environments). An app's config is everything that does not vary across deploys (like code).

### What goes wrong without separation

```
Scenario 1: Config baked into image
  Image: taskapi:1.0 → DATABASE_URL=prod-db-1.rds.amazonaws.com
  → Want to test with different DB? Need a new image. Rebuild. Push. Slow.
  → Dev has different DB than prod? Two different images for same code. Chaos.

Scenario 2: Secrets in image
  Image: taskapi:1.0 → DB_PASSWORD=supersecret123
  → Anyone who can pull the image can read your DB password.
  → Secret is in your CI/CD logs. In Docker Hub. Forever.

Scenario 3: Everything hardcoded in Pod spec YAML committed to Git
  → DB passwords visible in Git history.
  → Can't rotate a secret without editing YAML and redeploying.
```

### The solution — Kubernetes native objects

```
ConfigMap → non-sensitive config (app settings, feature flags, URLs)
Secret    → sensitive data (passwords, API keys, TLS certs, tokens)

Both are:
- Stored in etcd (Kubernetes cluster state)
- Referenced by Pods at runtime (not baked into images)
- Updatable without rebuilding the image
- Scoped to a namespace (isolation)
- Manageable with RBAC (who can read Secrets)
```

**Teaching line:**
> "Separating config from code is like separating a recipe from the ingredients. The recipe (code/image) is the same everywhere. The ingredients (config) change per environment — dev uses local flour, production uses organic flour. You don't rewrite the recipe every time you change an ingredient."

---

## 2. ConfigMap — Non-Sensitive Configuration

### WHAT
A **ConfigMap** stores non-sensitive key-value configuration data that can be injected into Pods as environment variables, command-line arguments, or config files (via volume mounts).

### WHY
- Decouple environment-specific config from application images
- Change config without rebuilding or redeploying the image (with volume mounts)
- Share config across multiple Pods/Deployments
- Keep YAML files clean — reference a ConfigMap instead of listing 20 env vars

### HOW — Creating ConfigMaps

**Method 1: From a YAML file (declarative — preferred)**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: taskapi-config
  namespace: default
data:
  # Simple key-value pairs (become env vars)
  NODE_ENV: "production"
  PORT: "3000"
  LOG_LEVEL: "info"
  APP_NAME: "TaskAPI"
  MAX_CONNECTIONS: "100"

  # Multi-line values (become config files when volume-mounted)
  nginx.conf: |
    server {
      listen 80;
      location /api/ {
        proxy_pass http://api-service:3000/;
      }
    }

  app-config.json: |
    {
      "featureFlags": {
        "newDashboard": true,
        "betaSearch": false
      },
      "rateLimiting": {
        "windowMs": 60000,
        "max": 100
      }
    }
```

**Method 2: From literal values (imperative — quick testing)**
```bash
kubectl create configmap taskapi-config \
  --from-literal=NODE_ENV=production \
  --from-literal=PORT=3000 \
  --from-literal=LOG_LEVEL=info
```

**Method 3: From a file**
```bash
# ConfigMap key = filename, value = file contents
kubectl create configmap nginx-config --from-file=./nginx.conf
kubectl create configmap app-config --from-file=./config/   # all files in directory
```

```bash
# View a ConfigMap
kubectl get configmaps
kubectl describe configmap taskapi-config
kubectl get configmap taskapi-config -o yaml
```

---

## 3. Consuming ConfigMaps — Three Ways

### Method 1: As Individual Environment Variables

```yaml
spec:
  containers:
    - name: api
      image: taskapi:1.0
      env:
        # Individual key from ConfigMap
        - name: NODE_ENV
          valueFrom:
            configMapKeyRef:
              name: taskapi-config    # ConfigMap name
              key: NODE_ENV           # key inside the ConfigMap
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: taskapi-config
              key: LOG_LEVEL
```

### Method 2: All Keys as Environment Variables at Once

```yaml
spec:
  containers:
    - name: api
      image: taskapi:1.0
      envFrom:
        - configMapRef:
            name: taskapi-config    # ALL keys become env vars
        - configMapRef:
            name: common-config     # can reference multiple ConfigMaps
```

This injects ALL key-value pairs from the ConfigMap as environment variables. Clean, no repetition.

### Method 3: As a Volume Mount (Config Files)

```yaml
spec:
  containers:
    - name: nginx
      image: nginx:alpine
      volumeMounts:
        - name: nginx-config-vol
          mountPath: /etc/nginx/conf.d   # directory inside container
          readOnly: true

  volumes:
    - name: nginx-config-vol
      configMap:
        name: nginx-config              # ConfigMap to mount
        # Each key becomes a file in the mountPath directory
        # key: nginx.conf → file: /etc/nginx/conf.d/nginx.conf
```

**Mount specific keys only:**
```yaml
volumes:
  - name: config-vol
    configMap:
      name: taskapi-config
      items:                          # select specific keys
        - key: app-config.json        # ConfigMap key
          path: config.json           # filename to create in the container
```

**Why volume mounts are powerful for config:**
> When you update a ConfigMap, files mounted via volumes are **automatically updated inside running containers** (within ~1 minute) — without restarting the Pod. Environment variables, however, are only set at Pod startup and do NOT update live.

---

## 4. Secret — Sensitive Configuration

### WHAT
A **Secret** is like a ConfigMap but specifically for sensitive data — passwords, API keys, TLS certificates, SSH keys, tokens. Kubernetes treats Secrets differently from ConfigMaps:
- Stored with base64 encoding in etcd (NOT plaintext, but NOT encrypted by default)
- Only sent to nodes that have Pods needing them
- Stored in memory (tmpfs) on nodes — not written to disk
- Access controlled via RBAC — you can restrict who can read Secrets

### WHY Secrets and not just ConfigMaps
- Signals intent — anyone reading your YAML knows this value is sensitive
- RBAC can allow a service to read its ConfigMaps but not Secrets (least privilege)
- Kubernetes can encrypt Secrets at rest in etcd (with encryption config)
- Audit logging can specifically track Secret access
- External secret management (AWS Secrets Manager, Vault) integrates via the Secret API

### WHAT base64 is NOT
**Critical teaching point — base64 is encoding, not encryption:**
```bash
echo -n "mysecretpassword" | base64
# bXlzZWNyZXRwYXNzd29yZA==

echo "bXlzZWNyZXRwYXNzd29yZA==" | base64 -d
# mysecretpassword
```

Anyone with access to etcd or the Secret object can read the value. Base64 just prevents accidental exposure in logs. **Always enable etcd encryption at rest in production, or use external secret management.**

---

## 5. Creating Secrets

### Method 1: YAML with base64-encoded values (type: Opaque)

```bash
# First encode your values
echo -n "supersecretpassword" | base64
# c3VwZXJzZWNyZXRwYXNzd29yZA==

echo -n "taskuser" | base64
# dGFza3VzZXI=
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: default
type: Opaque                        # generic secret (most common type)
data:
  DB_PASSWORD: c3VwZXJzZWNyZXRwYXNzd29yZA==   # base64 encoded
  DB_USER: dGFza3VzZXI=                          # base64 encoded
```

**Simpler — use `stringData` (K8s encodes for you):**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:                         # plain text — K8s base64-encodes automatically
  DB_PASSWORD: supersecretpassword  # no manual encoding needed
  DB_USER: taskuser
  DATABASE_URL: "postgresql://taskuser:supersecretpassword@postgres-svc:5432/taskdb"
```

### Method 2: Imperative (quick — for dev/testing)

```bash
kubectl create secret generic db-secret \
  --from-literal=DB_PASSWORD=supersecretpassword \
  --from-literal=DB_USER=taskuser

kubectl create secret generic api-keys \
  --from-literal=STRIPE_SECRET_KEY=sk_live_... \
  --from-literal=JWT_SECRET=my-jwt-secret-key
```

### Method 3: From files (for TLS certs, SSH keys)

```bash
# TLS certificate and key
kubectl create secret tls taskapi-tls \
  --cert=tls.crt \
  --key=tls.key

# Generic secret from file
kubectl create secret generic ssh-key \
  --from-file=ssh-privatekey=~/.ssh/id_rsa
```

### Secret Types

| Type | Use case |
|---|---|
| `Opaque` | Generic — any key-value data (most common) |
| `kubernetes.io/tls` | TLS certificates and keys |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials |
| `kubernetes.io/ssh-auth` | SSH private keys |
| `kubernetes.io/service-account-token` | Service account tokens (auto-created) |

---

## 6. Consuming Secrets — Three Ways (Same as ConfigMap)

### Method 1: As Individual Environment Variables

```yaml
spec:
  containers:
    - name: api
      image: taskapi:1.0
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:             # ← secretKeyRef (not configMapKeyRef)
              name: db-secret
              key: DB_PASSWORD
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DB_USER
```

### Method 2: All Keys as Environment Variables

```yaml
spec:
  containers:
    - name: api
      envFrom:
        - secretRef:
            name: db-secret         # all Secret keys become env vars
        - configMapRef:
            name: taskapi-config    # can mix ConfigMap and Secret
```

### Method 3: As Volume Mount (for TLS certs, config files with secrets)

```yaml
spec:
  containers:
    - name: api
      volumeMounts:
        - name: secrets-vol
          mountPath: /app/secrets   # files appear here
          readOnly: true

  volumes:
    - name: secrets-vol
      secret:
        secretName: db-secret       # each key becomes a file
        defaultMode: 0400           # read-only permissions (owner only)
```

**Inside the container:**
```bash
# Each Secret key becomes a file:
ls /app/secrets
# DB_PASSWORD
# DB_USER

cat /app/secrets/DB_PASSWORD
# supersecretpassword

# Read in Node.js:
const dbPassword = fs.readFileSync('/app/secrets/DB_PASSWORD', 'utf8').trim();
```

---

## 7. Full Production Pod Spec — ConfigMap + Secret Together

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: taskapi
spec:
  replicas: 3
  selector:
    matchLabels:
      app: taskapi
  template:
    metadata:
      labels:
        app: taskapi
    spec:
      containers:
        - name: api
          image: taskapi:2.0
          ports:
            - containerPort: 3000

          # ── Inject all ConfigMap values as env vars ───────────────────────
          envFrom:
            - configMapRef:
                name: taskapi-config      # NODE_ENV, PORT, LOG_LEVEL, APP_NAME

          # ── Inject individual Secret values as env vars ───────────────────
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: DB_PASSWORD
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: DB_USER
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: DATABASE_URL
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: api-keys
                  key: JWT_SECRET

          # ── Mount TLS cert as volume ───────────────────────────────────────
          volumeMounts:
            - name: tls-certs
              mountPath: /app/certs
              readOnly: true

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

      volumes:
        - name: tls-certs
          secret:
            secretName: taskapi-tls
            defaultMode: 0400
```

---

## 8. Updating ConfigMaps and Secrets

### Updating a ConfigMap

```bash
# Edit in place
kubectl edit configmap taskapi-config

# Apply updated YAML file
kubectl apply -f configmap.yaml

# Patch a specific key
kubectl patch configmap taskapi-config \
  --patch '{"data": {"LOG_LEVEL": "debug"}}'
```

### What happens after an update?

| Consumption method | Update behaviour |
|---|---|
| `envFrom` / `env` (env vars) | ❌ NOT updated — env vars are set at Pod startup. Must restart Pod. |
| Volume mount | ✅ Updated automatically within ~1 minute (kubelet syncs) |

```bash
# Force Pod restart to pick up env var changes (triggers rolling update)
kubectl rollout restart deployment/taskapi
```

**Teaching line:**
> "Environment variables are like a printed name badge — you get it when you walk in the door and it doesn't change until you leave and come back. Volume-mounted config files are like a digital display board — it updates live as the source changes."

---

## 9. Docker Registry Secrets — Pulling from Private Registries

When your image is in a private registry (AWS ECR, Docker Hub private), Kubernetes needs credentials to pull it.

```bash
# Create a registry pull secret for AWS ECR
kubectl create secret docker-registry ecr-secret \
  --docker-server=123456789.dkr.ecr.ap-south-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region ap-south-1)

# Create for Docker Hub
kubectl create secret docker-registry dockerhub-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=yourusername \
  --docker-password=yourpassword \
  --docker-email=your@email.com
```

```yaml
# Reference in Pod/Deployment spec
spec:
  imagePullSecrets:
    - name: ecr-secret        # ← tells kubelet to use these creds for image pull
  containers:
    - name: api
      image: 123456789.dkr.ecr.ap-south-1.amazonaws.com/taskapi:2.0
```

---

## 10. Production Secret Management — Beyond Basic Secrets

Basic Kubernetes Secrets have limitations:
- Stored in etcd as base64 — not encrypted unless you explicitly configure it
- Secrets visible to anyone with RBAC access to Secrets in that namespace
- No secret rotation, versioning, or audit trail built-in

### Option 1: etcd Encryption at Rest

Enable in your cluster config — encrypts all Secrets in etcd using AES-GCM:
```yaml
# encryption-config.yaml (applied to API server)
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>
      - identity: {}   # fallback for unencrypted
```

### Option 2: External Secrets Operator (AWS Secrets Manager / Vault)

The production standard — secrets never live in Kubernetes etcd. They live in AWS Secrets Manager or HashiCorp Vault, and the **External Secrets Operator** syncs them into K8s Secrets automatically.

```yaml
# ExternalSecret — tells the operator to fetch from AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-secret
spec:
  refreshInterval: 1h             # re-sync every hour
  secretStoreRef:
    name: aws-secretsmanager      # which external store to use
    kind: ClusterSecretStore
  target:
    name: db-secret               # K8s Secret to create/update
    creationPolicy: Owner
  data:
    - secretKey: DB_PASSWORD      # key in K8s Secret
      remoteRef:
        key: production/taskapi/db  # path in AWS Secrets Manager
        property: password
```

```
Flow:
AWS Secrets Manager → External Secrets Operator → K8s Secret → Pod env var
     (source of truth)     (sync controller)       (K8s native)   (consumed by app)
```

**Benefits:**
- Secrets never committed to Git
- Automatic rotation — update in Secrets Manager → syncs to K8s automatically
- Full audit trail in AWS CloudTrail
- Access control via AWS IAM (not just K8s RBAC)

### Option 3: Sealed Secrets (GitOps-friendly)

Encrypt Secrets before committing them to Git using **Bitnami Sealed Secrets**:
```bash
# Encrypt a Secret into a SealedSecret (safe to commit to Git)
kubeseal < secret.yaml > sealed-secret.yaml

# sealed-secret.yaml is encrypted — safe to push to GitHub
# The Sealed Secrets controller in the cluster decrypts it back to a K8s Secret
```

---

## 11. ConfigMap and Secret — Side-by-Side Comparison

| | ConfigMap | Secret |
|---|---|---|
| For | Non-sensitive config | Sensitive data |
| Storage in etcd | Plaintext | Base64 encoded |
| Default encryption | ❌ No | ❌ No (unless configured) |
| Node storage | Disk | tmpfs (memory only) |
| RBAC granularity | Standard | Can restrict independently |
| Size limit | 1MB | 1MB |
| Live update (volume) | ✅ ~1 min | ✅ ~1 min |
| Live update (env var) | ❌ Restart needed | ❌ Restart needed |
| Use in `envFrom` | ✅ `configMapRef` | ✅ `secretRef` |
| Use in volumes | ✅ `configMap:` | ✅ `secret:` |

---

## 12. Namespace Isolation — Secrets Don't Cross Namespaces

An important security property:

```
Namespace: production
  Secret: db-secret   → only Pods in 'production' namespace can use it

Namespace: staging
  Secret: db-secret   → completely separate — different value, different RBAC

A Pod in 'staging' CANNOT reference a Secret in 'production'.
```

This is why multi-tenant clusters put different customer workloads in different namespaces — their Secrets are fully isolated.

---

## 13. CLI Reference — ConfigMaps and Secrets

```bash
# ── ConfigMaps ────────────────────────────────────────────────────────────────
kubectl get configmaps
kubectl get cm                                   # shorthand
kubectl describe configmap <name>
kubectl get configmap <name> -o yaml
kubectl create configmap <name> --from-literal=KEY=VALUE
kubectl create configmap <name> --from-file=<path>
kubectl apply -f configmap.yaml
kubectl edit configmap <name>
kubectl delete configmap <name>

# ── Secrets ───────────────────────────────────────────────────────────────────
kubectl get secrets
kubectl describe secret <name>                   # shows keys but NOT values
kubectl get secret <name> -o yaml                # shows base64-encoded values

# Decode a Secret value
kubectl get secret db-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d

# Create secrets
kubectl create secret generic <name> --from-literal=KEY=VALUE
kubectl create secret tls <name> --cert=tls.crt --key=tls.key
kubectl create secret docker-registry <name> \
  --docker-server=<registry> \
  --docker-username=<user> \
  --docker-password=<pass>

kubectl apply -f secret.yaml
kubectl edit secret <name>
kubectl delete secret <name>
```

---

## 14. Complete Working Example — Full Stack with Config & Secrets

```yaml
# ── ConfigMap ─────────────────────────────────────────────────────────────────
apiVersion: v1
kind: ConfigMap
metadata:
  name: taskapi-config
  namespace: default
data:
  NODE_ENV: "production"
  PORT: "3000"
  LOG_LEVEL: "info"
  REDIS_HOST: "redis-svc"
  REDIS_PORT: "6379"

---
# ── Secret ───────────────────────────────────────────────────────────────────
apiVersion: v1
kind: Secret
metadata:
  name: taskapi-secrets
  namespace: default
type: Opaque
stringData:
  DB_PASSWORD: "supersecretpassword"
  DB_USER: "taskuser"
  DATABASE_URL: "postgresql://taskuser:supersecretpassword@postgres-svc:5432/taskdb"
  JWT_SECRET: "my-very-long-jwt-signing-secret-key-minimum-32-chars"

---
# ── PostgreSQL Deployment ─────────────────────────────────────────────────────
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
              valueFrom:
                secretKeyRef:
                  name: taskapi-secrets
                  key: DB_PASSWORD
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: taskapi-secrets
                  key: DB_USER
            - name: POSTGRES_DB
              value: "taskdb"
          ports:
            - containerPort: 5432
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"

---
# ── TaskAPI Deployment ────────────────────────────────────────────────────────
apiVersion: apps/v1
kind: Deployment
metadata:
  name: taskapi
spec:
  replicas: 3
  selector:
    matchLabels:
      app: taskapi
  template:
    metadata:
      labels:
        app: taskapi
    spec:
      containers:
        - name: api
          image: taskapi:2.0
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: taskapi-config        # NODE_ENV, PORT, LOG_LEVEL, etc.
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: taskapi-secrets
                  key: DATABASE_URL
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: taskapi-secrets
                  key: JWT_SECRET
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
```

---

## 15. Hands-On Practice Tasks

```bash
# 1. Create the ConfigMap and Secret from the YAML above
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml

# 2. Verify they exist
kubectl get cm,secrets

# 3. Describe them — notice Secret values are hidden
kubectl describe configmap taskapi-config
kubectl describe secret taskapi-secrets    # values shown as <hidden>

# 4. Decode a secret value
kubectl get secret taskapi-secrets -o jsonpath='{.data.DB_PASSWORD}' | base64 -d

# 5. Deploy a Pod that uses them
kubectl apply -f deployment.yaml

# 6. Verify env vars are injected inside the pod
kubectl exec -it <pod-name> -- env | grep NODE_ENV
kubectl exec -it <pod-name> -- env | grep DB_PASSWORD

# 7. Update the ConfigMap (change LOG_LEVEL to debug)
kubectl edit configmap taskapi-config
# Restart pods to pick up the change
kubectl rollout restart deployment/taskapi

# 8. Mount the ConfigMap as a volume and verify the file appears
# Add a volumeMount to the deployment, apply, then:
kubectl exec -it <pod-name> -- ls /app/config
kubectl exec -it <pod-name> -- cat /app/config/app-config.json

# 9. Create a Docker registry secret for ECR
aws ecr get-login-password --region ap-south-1 | \
  kubectl create secret docker-registry ecr-secret \
    --docker-server=123456789.dkr.ecr.ap-south-1.amazonaws.com \
    --docker-username=AWS \
    --docker-password-stdin

# 10. Try to get a secret from a different namespace (should fail)
kubectl get secret taskapi-secrets -n kube-system
```

---

## 16. Quiz — Test Yourself / Test Others

1. What is the 12-Factor App principle around configuration, and how do ConfigMaps/Secrets implement it?
2. What is the difference between a ConfigMap and a Secret?
3. Is base64 encoding the same as encryption? Why does this matter for Secrets?
4. What are the three ways to consume a ConfigMap inside a Pod?
5. You update a ConfigMap. Which consumption method picks up the change automatically, and which requires a Pod restart?
6. What is `stringData` in a Secret and how does it differ from `data`?
7. Why would you use a volume mount for a Secret instead of environment variables?
8. What is the `imagePullSecrets` field in a Pod spec used for?
9. What are the limitations of native Kubernetes Secrets, and what are two production alternatives?
10. Can a Pod in namespace `staging` access a Secret defined in namespace `production`?

<details>
<summary>Answers</summary>

1. 12-Factor says config (anything that varies between deployments like dev, staging, prod) must be stored in the environment, not in the code. ConfigMaps implement this for non-sensitive config — you store env-specific values (URLs, feature flags, log levels) outside the image and inject them at runtime. Secrets implement it for sensitive data.

2. ConfigMap is for non-sensitive configuration (app settings, URLs, feature flags). Secret is for sensitive data (passwords, API keys, TLS certs). Secrets are base64-encoded in etcd, stored in memory-only (tmpfs) on nodes, and can be restricted more granularly with RBAC. Semantically they signal different intent.

3. No — base64 is encoding (reversible transformation for safe text transport), not encryption. Anyone can decode it instantly with `base64 -d`. It just prevents accidental exposure in logs. For real security, you need etcd encryption at rest and/or external secret management (AWS Secrets Manager, Vault).

4. (1) Individual env vars via `env.valueFrom.configMapKeyRef` — inject specific keys by name. (2) All keys as env vars via `envFrom.configMapRef` — inject every key at once. (3) Volume mount via `volumes.configMap` — each key becomes a file in a directory inside the container.

5. Volume mounts pick up ConfigMap changes automatically within ~1 minute (kubelet syncs the files). Environment variables (`env` and `envFrom`) are set only at Pod startup and do NOT update — you must restart the Pods (e.g., `kubectl rollout restart deployment`) to pick up changes.

6. `stringData` lets you write Secret values as plain text — Kubernetes automatically base64-encodes them when storing. `data` requires you to manually base64-encode values before writing them. `stringData` is write-only (it doesn't appear in `kubectl get secret -o yaml`), making Secrets easier to write without error.

7. TLS certificates and private keys are better as volume-mounted files because: (1) apps typically read them as files (not env vars), (2) they can be updated/rotated without restarting the Pod (kubelet syncs volume mounts), (3) files can have restrictive permissions (`defaultMode: 0400`), and (4) they don't appear in `env` output which reduces accidental exposure.

8. `imagePullSecrets` provides Docker registry credentials to the kubelet so it can authenticate and pull images from private registries (AWS ECR, private Docker Hub repos, GCR). Without it, Pods using private images would fail with `ImagePullBackOff`.

9. Limitations: base64 is not encryption, secrets visible to anyone with RBAC access to that namespace, no secret rotation/versioning/audit trail. Alternatives: (1) External Secrets Operator — syncs from AWS Secrets Manager or HashiCorp Vault, keeps secrets out of etcd entirely. (2) Sealed Secrets — encrypts Secrets with a cluster-specific key so they can be safely committed to Git.

10. No. Kubernetes Secrets are namespace-scoped. A Pod can only reference Secrets in its own namespace. This is a deliberate security boundary — namespaces provide full Secret isolation between teams and environments.

</details>

---

## 17. Summary

After today you can explain and implement:
- **Why** config must be separated from code (12-Factor App, environment parity, security)
- **ConfigMap** — creation (YAML, imperative, from-file), all 3 consumption methods, live update behaviour
- **Secret** — creation, types (Opaque, TLS, docker-registry), base64 vs encryption, `stringData`
- All 3 Secret consumption methods — env, envFrom, volume mount
- **Full production Pod spec** combining ConfigMap + Secret cleanly
- **Live update behaviour** — volume mounts update automatically, env vars need Pod restart
- **Docker registry secrets** — `imagePullSecrets` for private registries
- **Production secret management** — etcd encryption, External Secrets Operator, Sealed Secrets
- Namespace isolation of Secrets — pods can only access Secrets in their own namespace
- Complete CLI for managing ConfigMaps and Secrets

**Coming up on Day 17:** Storage in Kubernetes — PersistentVolumes, PersistentVolumeClaims, and StorageClasses. How Kubernetes abstracts away the underlying storage (EBS, NFS, local disk) and lets Pods request storage declaratively — the foundation for running databases and stateful workloads in Kubernetes.
