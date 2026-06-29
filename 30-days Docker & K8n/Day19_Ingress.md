# Day 19: Ingress — HTTP/S Routing, TLS Termination & Production Traffic Management

**Goal for today:** Understand what Ingress is, why it exists, how Ingress Controllers work, every type of routing rule (path-based, host-based, TLS), how to configure TLS termination with certificates, and how to build a production HTTP routing layer for multiple services behind a single load balancer. By the end, you should be able to design and implement the complete external traffic management layer for any Kubernetes application.

**Format:** WHAT → WHY → HOW for every concept.

---

## 0. Quick Recap (open your teaching session with this)

- Day 15: Services — ClusterIP, NodePort, LoadBalancer, DNS
- Day 17: Storage — PV, PVC, StorageClass
- Day 18: Namespaces & Resource Management — quotas, limits, cluster structure

**Today's connecting thought:**
> On Day 15 you learned that LoadBalancer Services expose individual services externally — but each one creates a separate cloud load balancer. A real application has dozens of services (API, auth, admin, websockets, static files). Creating one load balancer per service would cost a fortune and create a URL management nightmare. Ingress is the solution: one load balancer, one IP address, and intelligent HTTP routing that sends traffic to the right service based on the URL path or hostname. This is how every real production Kubernetes app handles external traffic.

---

## 1. The Problem — Why LoadBalancer Services Are Not Enough

### The Cost and Complexity Problem

```
Without Ingress (one LoadBalancer per service):

Internet
  │
  ├──► ELB-1 (api.myapp.com)      → api-service      → api pods
  ├──► ELB-2 (auth.myapp.com)     → auth-service      → auth pods
  ├──► ELB-3 (admin.myapp.com)    → admin-service     → admin pods
  ├──► ELB-4 (ws.myapp.com)       → websocket-service → ws pods
  └──► ELB-5 (myapp.com/static)   → static-service    → static pods

5 Load Balancers × ~$20/month each = $100+/month just for LBs
5 separate SSL certificates to manage
5 separate DNS records
5 separate health check configs
No central place to add authentication, rate limiting, CORS headers
```

### What You Actually Want

```
With Ingress (one load balancer, intelligent routing):

Internet
  │
  └──► ELB (single entry point)
          │
          └──► Ingress Controller (nginx/traefik/etc.)
                    │
                    ├── myapp.com/api/*     → api-service:3000
                    ├── myapp.com/auth/*    → auth-service:4000
                    ├── admin.myapp.com/*   → admin-service:5000
                    ├── ws.myapp.com/*      → websocket-service:8080
                    └── myapp.com/*         → frontend-service:80

1 Load Balancer × $20/month = $20/month
1 SSL certificate (wildcard *.myapp.com)
Centralised: auth, rate limiting, CORS, compression, logging
```

**Teaching line:**
> "Ingress is like a smart hotel concierge standing at one main entrance. When guests arrive, the concierge reads their reservation (the URL) and directs them to the right floor and room — Restaurant on floor 2, Spa on floor 3, Conference rooms on floor 4. Without the concierge, you'd need a separate entrance door for every floor."

---

## 2. Ingress vs Ingress Controller — The Critical Distinction

This confuses almost every beginner. Clarify this upfront.

| Object | What it is |
|---|---|
| **Ingress** | A Kubernetes API object that defines routing rules (YAML you write) |
| **Ingress Controller** | A running application (Pod) that reads Ingress rules and actually implements them |

```
You write:  Ingress YAML (routing rules)
                │
                │ read by
                ▼
Ingress Controller (e.g., nginx-ingress)
  → configures itself based on your rules
  → handles actual traffic routing

Without an Ingress Controller, Ingress objects do NOTHING.
An Ingress is just a config — the Controller is the engine.
```

**Teaching line:**
> "An Ingress object is like a menu card in a restaurant — it lists what's available and where. The Ingress Controller is the waiter who actually reads the menu and brings you what you ordered. Without the waiter, the menu card is just paper."

---

## 3. Popular Ingress Controllers

| Controller | Notes | Best for |
|---|---|---|
| **nginx-ingress** (ingress-nginx) | Most popular, mature, feature-rich | General purpose |
| **Traefik** | Auto-discovers services, has dashboard | Microservices, auto-config |
| **AWS ALB Ingress Controller** | Provisions AWS ALB natively | AWS EKS |
| **GCE Ingress** | Provisions GCP Load Balancer | GKE |
| **HAProxy Ingress** | High performance | High-traffic |
| **Kong** | API gateway features | APIs needing auth/rate-limiting |

**We'll use `ingress-nginx`** (the community version, not the NGINX Inc one) — it's the most common, has the best documentation, and works on any cluster.

---

## 4. Installing the Ingress Controller

### On Minikube (easiest)

```bash
# Enable the built-in ingress addon
minikube addons enable ingress

# Verify the controller is running
kubectl get pods -n ingress-nginx
# NAME                                        READY   STATUS
# ingress-nginx-controller-xxxx-yyyy          1/1     Running

# Get the Minikube IP to access Ingress
minikube ip
# 192.168.49.2
# → your ingress is accessible at http://192.168.49.2
```

### On a real cluster (Helm — production approach)

```bash
# Install using Helm (Day 24 covers Helm in depth)
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer

# Wait for LoadBalancer external IP
kubectl get service ingress-nginx-controller -n ingress-nginx -w
# NAME                       TYPE           CLUSTER-IP    EXTERNAL-IP
# ingress-nginx-controller   LoadBalancer   10.96.50.1    a1b2.elb.amazonaws.com
```

### Using YAML manifests directly

```bash
kubectl apply -f \
  https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.4/deploy/static/provider/cloud/deploy.yaml
```

---

## 5. The Basic Ingress Object

### WHAT — Anatomy of an Ingress YAML

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: taskapi-ingress
  namespace: production
  annotations:                          # controller-specific config goes here
    nginx.ingress.kubernetes.io/rewrite-target: /

spec:
  ingressClassName: nginx               # which Ingress Controller handles this

  rules:                                # routing rules
    - host: api.taskapp.com             # match this hostname
      http:
        paths:
          - path: /                     # match this path
            pathType: Prefix            # Prefix | Exact | ImplementationSpecific
            backend:
              service:
                name: api-service       # forward to this Service
                port:
                  number: 3000          # on this port
```

```bash
kubectl apply -f ingress.yaml
kubectl get ingress
# NAME               CLASS   HOSTS             ADDRESS          PORTS   AGE
# taskapi-ingress    nginx   api.taskapp.com   192.168.49.2     80      1m

kubectl describe ingress taskapi-ingress
```

---

## 6. Path Types — Exact, Prefix, ImplementationSpecific

| PathType | Behaviour |
|---|---|
| `Exact` | URL must match exactly — `/api/v1` matches ONLY `/api/v1`, not `/api/v1/` |
| `Prefix` | URL must start with prefix — `/api` matches `/api`, `/api/users`, `/api/v1/tasks` |
| `ImplementationSpecific` | Matching behaviour defined by the Ingress Controller (avoid unless needed) |

```yaml
paths:
  - path: /api
    pathType: Prefix    # /api, /api/users, /api/v1/tasks all match
  - path: /health
    pathType: Exact     # ONLY /health matches, not /health/ or /healthz
```

---

## 7. Path-Based Routing — One Host, Multiple Services

Route different URL paths to different backend services. Most common pattern for monolith → microservice migration.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: taskapp-path-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: taskapp.com
      http:
        paths:
          # Route /api/* to the backend API
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 3000

          # Route /auth/* to the auth service
          - path: /auth
            pathType: Prefix
            backend:
              service:
                name: auth-service
                port:
                  number: 4000

          # Route /admin/* to the admin panel
          - path: /admin
            pathType: Prefix
            backend:
              service:
                name: admin-service
                port:
                  number: 5000

          # Everything else → frontend (React app)
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

**Traffic flow:**
```
taskapp.com/api/tasks    → api-service:3000
taskapp.com/auth/login   → auth-service:4000
taskapp.com/admin/users  → admin-service:5000
taskapp.com/             → frontend-service:80
taskapp.com/about        → frontend-service:80 (catch-all)
```

**Important: path ordering matters — more specific paths first.**
The Ingress Controller matches paths in order — put specific paths before catch-all `/`.

---

## 8. Host-Based Routing — Multiple Domains, One Cluster

Route different hostnames (domains) to different services. Used for multi-tenant systems or when different teams own different subdomains.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-host-ingress
  namespace: production
spec:
  ingressClassName: nginx
  rules:
    # ── API subdomain ───────────────────────────────────────────────────────
    - host: api.taskapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 3000

    # ── Admin subdomain ─────────────────────────────────────────────────────
    - host: admin.taskapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin-service
                port:
                  number: 5000

    # ── Main domain ─────────────────────────────────────────────────────────
    - host: taskapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

**Traffic flow:**
```
api.taskapp.com/users      → api-service:3000
admin.taskapp.com/         → admin-service:5000
taskapp.com/               → frontend-service:80
```

---

## 9. TLS/HTTPS — Terminating SSL at the Ingress

### WHAT
TLS termination at the Ingress means: external clients connect over HTTPS, the Ingress Controller decrypts the traffic, and forwards plain HTTP to your backend services. Your backend services don't need to handle TLS at all — the Ingress handles it.

### WHY
- Centralise certificate management — one place for all certs
- Backend services stay simple (plain HTTP internally)
- Easier certificate rotation — update once at Ingress, affects all services
- Internal cluster traffic doesn't need encryption overhead

### HOW — Step by step

**Step 1: Create a TLS Secret with certificate and key**

```bash
# For testing: generate a self-signed certificate
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=taskapp.com/O=TaskApp"

# Create Kubernetes TLS Secret
kubectl create secret tls taskapp-tls \
  --cert=tls.crt \
  --key=tls.key \
  -n production
```

**Step 2: Reference the TLS Secret in the Ingress**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: taskapp-tls-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"     # HTTP → HTTPS redirect
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"

spec:
  ingressClassName: nginx

  # ── TLS Configuration ──────────────────────────────────────────────────────
  tls:
    - hosts:
        - taskapp.com
        - api.taskapp.com
      secretName: taskapp-tls       # ← TLS Secret created above

  # ── Routing Rules ──────────────────────────────────────────────────────────
  rules:
    - host: taskapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80

    - host: api.taskapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 3000
```

**Traffic flow with TLS:**
```
Client → HTTPS:443 to ELB → Ingress Controller
                                   │ decrypts TLS
                                   ▼
                           HTTP → api-service:3000
```

---

## 10. cert-manager — Automatic TLS Certificate Management

### WHAT
**cert-manager** is a Kubernetes add-on that automatically provisions and renews TLS certificates from Let's Encrypt (or other CAs). Instead of manually managing certificates, you annotate your Ingress and cert-manager handles everything.

### WHY
- Manual certificate management is error-prone (forgetting to renew = outage)
- Let's Encrypt certificates expire every 90 days
- cert-manager automates issuance and renewal — zero manual work

### HOW

**Install cert-manager:**
```bash
kubectl apply -f \
  https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Wait for pods to be ready
kubectl get pods -n cert-manager
```

**Create a ClusterIssuer (one-time cluster setup):**
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com          # for expiry notifications
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: nginx                   # use nginx ingress for ACME challenge
```

**Annotate your Ingress — cert-manager does the rest:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: taskapp-ingress
  namespace: production
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"   # ← trigger cert-manager

spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - taskapp.com
        - api.taskapp.com
      secretName: taskapp-tls             # cert-manager creates/updates this Secret

  rules:
    - host: taskapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

**What cert-manager does automatically:**
```
1. Sees Ingress with cert-manager annotation
2. Creates ACME challenge (proves you own the domain)
3. Let's Encrypt validates → issues certificate
4. cert-manager creates/updates the TLS Secret
5. Ingress Controller picks up the new cert
6. Every 60-75 days: automatically renews before expiry
→ Zero manual certificate work ever again
```

---

## 11. Ingress Annotations — Controller-Specific Configuration

Annotations are how you pass configuration to the Ingress Controller. The nginx-ingress controller has hundreds of annotations:

```yaml
metadata:
  annotations:
    # ── Rewriting ──────────────────────────────────────────────────────────
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    # Example: /api(/|$)(.*) → /$2 strips /api prefix before forwarding

    # ── Rate Limiting ───────────────────────────────────────────────────────
    nginx.ingress.kubernetes.io/limit-rps: "100"          # max 100 req/s per IP
    nginx.ingress.kubernetes.io/limit-connections: "20"   # max 20 concurrent per IP

    # ── CORS ───────────────────────────────────────────────────────────────
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://taskapp.com"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"

    # ── Timeouts ───────────────────────────────────────────────────────────
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"

    # ── Body size ──────────────────────────────────────────────────────────
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"    # max upload: 10MB

    # ── SSL ────────────────────────────────────────────────────────────────
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-protocols: "TLSv1.2 TLSv1.3"

    # ── Upstream hashing (sticky sessions) ─────────────────────────────────
    nginx.ingress.kubernetes.io/upstream-hash-by: "$request_uri"

    # ── Authentication (basic auth) ────────────────────────────────────────
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth-secret
    nginx.ingress.kubernetes.io/auth-realm: "Protected Area"

    # ── Whitelist IP ranges ─────────────────────────────────────────────────
    nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8,192.168.0.0/16"

    # ── Custom headers ──────────────────────────────────────────────────────
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
```

---

## 12. URL Rewriting — Stripping Path Prefixes

A very common requirement: your service internally handles `/` but external URL is `/api/`.

**Without rewrite** (service must handle `/api/` prefix):
```
Request: GET /api/users
Forwarded to service: GET /api/users   ← service must handle /api prefix
```

**With rewrite** (strip the `/api` prefix):
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2

spec:
  rules:
    - host: taskapp.com
      http:
        paths:
          - path: /api(/|$)(.*)         # capture group $2 = everything after /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 3000
```

```
Request: GET /api/users
Forwarded to service: GET /users    ← /api stripped, service handles /users
```

---

## 13. Default Backend — Handling Unmatched Routes

```yaml
spec:
  defaultBackend:              # fallback for any request matching no rules
    service:
      name: default-404-service
      port:
        number: 80

  rules:
    - host: api.taskapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 3000
```

The `defaultBackend` handles requests that don't match any rule — useful for custom 404 pages or redirect-to-main-site logic.

---

## 14. Complete Production Ingress — Full Example

This is the ingress configuration for a real production application:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: taskapp-production
  namespace: production
  annotations:
    # Certificate management
    cert-manager.io/cluster-issuer: "letsencrypt-prod"

    # SSL
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"

    # Security headers
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "Strict-Transport-Security: max-age=31536000; includeSubDomains";
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "Referrer-Policy: strict-origin-when-cross-origin";

    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "100"

    # CORS (for API)
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://taskapp.com"

    # Body size limit (for file uploads)
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"

    # Timeouts
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "10"

spec:
  ingressClassName: nginx

  tls:
    - hosts:
        - taskapp.com
        - api.taskapp.com
        - admin.taskapp.com
      secretName: taskapp-tls     # cert-manager populates this

  rules:
    # ── Main domain: React frontend ────────────────────────────────────────────
    - host: taskapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80

    # ── API subdomain: Node.js API ─────────────────────────────────────────────
    - host: api.taskapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 3000

    # ── Admin subdomain: Admin panel (restricted by IP) ────────────────────────
    - host: admin.taskapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin-service
                port:
                  number: 5000
```

---

## 15. Testing Ingress Locally (Minikube + /etc/hosts)

Since you're using a local Minikube cluster, real DNS doesn't point to it. Use `/etc/hosts` to test:

```bash
# Get Minikube IP
minikube ip
# 192.168.49.2

# Add entries to /etc/hosts (macOS/Linux)
sudo sh -c 'echo "192.168.49.2 taskapp.local api.taskapp.local admin.taskapp.local" >> /etc/hosts'

# Now you can access:
curl http://taskapp.local
curl http://api.taskapp.local/health

# Or use --resolve with curl (no /etc/hosts edit needed)
curl --resolve "taskapp.local:80:192.168.49.2" http://taskapp.local

# Undo /etc/hosts later by removing the lines you added
```

---

## 16. Debugging Ingress

```bash
# ── Check Ingress is created and has an address ────────────────────────────────
kubectl get ingress -n production

# NAME               CLASS   HOSTS              ADDRESS          PORTS     AGE
# taskapp-ingress    nginx   api.taskapp.com    192.168.49.2    80, 443   5m
#                                               ↑ empty means controller not ready

# ── Describe ingress — check rules and events ─────────────────────────────────
kubectl describe ingress taskapp-ingress -n production

# ── Check ingress controller logs ─────────────────────────────────────────────
kubectl logs -n ingress-nginx \
  $(kubectl get pods -n ingress-nginx -l app.kubernetes.io/component=controller -o name) \
  --tail=50 -f

# ── Common problems ────────────────────────────────────────────────────────────
```

| Problem | Symptom | Fix |
|---|---|---|
| No address in `get ingress` | ADDRESS column empty | Ingress controller not running — check `kubectl get pods -n ingress-nginx` |
| 404 from Ingress | Ingress returns 404 | No rule matches the request — check host + path exactly |
| 502 Bad Gateway | Ingress can't reach service | Check Service name/port in Ingress matches actual Service |
| 503 Service Unavailable | Backend Pods not ready | Check Pod readiness probes and Service endpoints |
| TLS certificate error | Browser security warning | Check TLS Secret contents and cert-manager logs |
| Rule not matching | Wrong service getting traffic | Path order matters — specific paths before catch-all `/` |

---

## 17. IngressClass — Managing Multiple Controllers

```yaml
# When you have multiple Ingress Controllers, specify which one handles each Ingress
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"  # default if ingressClassName omitted
spec:
  controller: k8s.io/ingress-nginx

---
# In your Ingress:
spec:
  ingressClassName: nginx     # must match IngressClass name above
```

---

## 18. Ingress vs Gateway API (the future)

Ingress is being gradually superseded by the **Gateway API** — a more powerful, extensible, and role-aware way to manage HTTP routing. Worth knowing it exists:

```yaml
# Gateway API (newer, more powerful — not yet universal)
kind: HTTPRoute    # replaces Ingress
kind: Gateway      # replaces IngressClass + LoadBalancer Service
kind: GatewayClass # replaces IngressClass provisioner
```

Gateway API separates concerns — infra team manages Gateways, app teams manage Routes. For now, Ingress is still the standard. Gateway API will be the standard in 2-3 years.

---

## 19. Complete Workflow — Full Ingress Setup

```bash
# ── Step 1: Enable/install Ingress Controller ──────────────────────────────────
minikube addons enable ingress
kubectl get pods -n ingress-nginx -w    # wait for Running

# ── Step 2: Deploy your services ──────────────────────────────────────────────
kubectl apply -f frontend-deployment.yaml
kubectl apply -f frontend-service.yaml
kubectl apply -f api-deployment.yaml
kubectl apply -f api-service.yaml

# ── Step 3: Create Ingress ──────────────────────────────────────────────────────
kubectl apply -f ingress.yaml

# ── Step 4: Get the Ingress address ────────────────────────────────────────────
kubectl get ingress
# Wait for ADDRESS to populate

# ── Step 5: Add to /etc/hosts ──────────────────────────────────────────────────
echo "$(minikube ip) taskapp.local api.taskapp.local" | sudo tee -a /etc/hosts

# ── Step 6: Test ───────────────────────────────────────────────────────────────
curl http://taskapp.local
curl http://api.taskapp.local/health

# ── Step 7: Watch the controller handle requests ──────────────────────────────
kubectl logs -n ingress-nginx \
  $(kubectl get pods -n ingress-nginx -o name | head -1) -f
```

---

## 20. Hands-On Practice Tasks

1. Enable Ingress on Minikube: `minikube addons enable ingress`
2. Deploy two services: a simple frontend (`nginx`) and a simple API (`httpd` as mock)
3. Write an Ingress with path-based routing: `/api/*` → api-service, `/` → frontend-service
4. Add `/etc/hosts` entry and verify both paths work with `curl`
5. Write a host-based Ingress: `api.taskapp.local` and `taskapp.local` routing to different services
6. Add a TLS Secret and enable HTTPS on the Ingress
7. Add the `ssl-redirect: "true"` annotation and verify HTTP redirects to HTTPS
8. Add rate limiting annotation and test it
9. Deliberately break the service name in the Ingress — observe the 503 error — fix it
10. Read the Ingress controller logs while making requests — see each request logged

---

## 21. Quiz — Test Yourself / Test Others

1. What is the difference between an Ingress and an Ingress Controller?
2. Why is a single Ingress preferable to multiple LoadBalancer Services in production?
3. What are the two main routing strategies in Ingress? Give an example of each.
4. What is the difference between `Prefix` and `Exact` path types?
5. What does TLS termination at the Ingress mean? What are its benefits?
6. What is cert-manager and what problem does it solve?
7. If an Ingress has rules for `/api` and `/`, which gets matched for a request to `/api/users`?
8. What does the `nginx.ingress.kubernetes.io/rewrite-target` annotation do?
9. An Ingress has been applied but the ADDRESS column is empty. What does this mean and how do you fix it?
10. What is the IngressClass object used for?

<details>
<summary>Answers</summary>

1. An Ingress is a Kubernetes API object (YAML spec) that defines HTTP routing rules — which hostname and path should go to which backend Service. An Ingress Controller is a running application (typically a Pod running Nginx, Traefik, or similar) that reads those Ingress rules and actually implements the routing. Without a Controller, Ingress objects do absolutely nothing.

2. Each LoadBalancer Service creates a separate cloud load balancer (e.g., AWS ELB) — with its own cost (~$20/month each), DNS name, SSL certificate, and config. With Ingress, one load balancer serves all services, using path or hostname routing to direct traffic. This reduces cost dramatically and centralises TLS, rate limiting, CORS, and logging in one place.

3. Path-based routing: different URL paths on the same host go to different services — e.g., `myapp.com/api` → api-service, `myapp.com/` → frontend-service. Host-based routing: different hostnames (domains/subdomains) go to different services — e.g., `api.myapp.com` → api-service, `admin.myapp.com` → admin-service.

4. `Prefix` matches any URL that starts with the given path — `/api` matches `/api`, `/api/users`, `/api/v1/tasks`. `Exact` matches only the exact path — `/health` matches ONLY `/health`, not `/health/` or `/healthz`.

5. TLS termination means the Ingress Controller handles HTTPS — it decrypts incoming encrypted traffic and forwards plain HTTP to backend services. Benefits: backend services don't need TLS configuration, certificates are managed centrally in one place, internal cluster traffic stays simple HTTP, and certificate rotation only needs to happen once.

6. cert-manager is a Kubernetes add-on that automatically provisions and renews TLS certificates from Let's Encrypt or other CAs. It solves the problem of manual certificate management — certificates expire every 90 days and manual renewal is error-prone. With cert-manager, you annotate your Ingress once and certificates are issued and renewed automatically forever.

7. The more specific path `/api` is matched first — `/api/users` starts with `/api` (Prefix), so it goes to the api-service. The `/` catch-all only matches requests that don't match any more specific path. Path specificity: longer/more specific paths take priority over shorter/less specific ones.

8. `rewrite-target` rewrites the request URL before forwarding it to the backend service. Example: incoming request `/api/users` with `rewrite-target: /$2` (and path `/api(/|$)(.*)`) strips the `/api` prefix and forwards just `/users` to the backend service. This lets your service handle clean paths internally while being exposed under a prefix externally.

9. An empty ADDRESS column means the Ingress Controller has not assigned an external IP/address to the Ingress. This usually means: the Ingress Controller is not running or not ready (check `kubectl get pods -n ingress-nginx`), the `ingressClassName` in the Ingress doesn't match any IngressClass, or on a local cluster, the LoadBalancer Service for the controller has no external IP (run `minikube tunnel` to fix on Minikube).

10. IngressClass is an object that links an Ingress resource to a specific Ingress Controller implementation. When multiple controllers are installed (e.g., nginx AND traefik), each Ingress specifies `ingressClassName: nginx` or `ingressClassName: traefik` to indicate which controller should handle it. One IngressClass can be marked as default so Ingresses without an explicit class are handled by it.

</details>

---

## 22. Summary

After today you can explain and implement:
- **Why Ingress exists** — one load balancer replacing many, centralised routing/TLS/auth
- **Ingress vs Ingress Controller** — the spec vs the engine that implements it
- **Installing ingress-nginx** — Minikube addon vs Helm for production
- **Path-based routing** — multiple services behind one host
- **Host-based routing** — multiple subdomains to different services
- **Path types** — Prefix, Exact, ImplementationSpecific
- **TLS termination** — TLS Secret creation, tls spec block, HTTP→HTTPS redirect
- **cert-manager** — automatic Let's Encrypt certificate provisioning and renewal
- **Annotations** — rate limiting, CORS, timeouts, rewrites, security headers
- **URL rewriting** — stripping path prefixes before forwarding
- **Default backend** — catch-all for unmatched routes
- **IngressClass** — managing multiple controllers
- Complete debugging toolkit for Ingress issues

**Coming up on Day 20:** Health Checks — Liveness, Readiness, and Startup Probes deep dive with real-world probe design patterns, how probes integrate with rolling updates and Services, and the exact probe settings for Node.js, Java, and database workloads.
