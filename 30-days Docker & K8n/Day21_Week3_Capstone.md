# Day 21: Week 3 Capstone — Deploy a Full-Stack App on Kubernetes End-to-End

**Goal for today:** Apply everything from Days 11–20 in one real, complete project. Deploy a production-grade full-stack application (Node.js API + PostgreSQL + Redis + Nginx) to your local Kubernetes cluster with proper Namespaces, ResourceQuotas, LimitRanges, ConfigMaps, Secrets, PVCs, Deployments, Services, Ingress, and health probes. By the end, you should have a running Kubernetes application you understand at every layer.

---

## 0. What You Are Building

A production-pattern deployment of **TaskAPI** on Kubernetes.

| Component | Technology | K8s Object |
|---|---|---|
| API | Node.js/Express | Deployment + Service |
| Database | PostgreSQL 15 | Deployment + Service + PVC |
| Cache | Redis 7 | Deployment + Service |
| Reverse Proxy | Nginx | Deployment + Service + Ingress |
| Namespace config | production | Namespace + ResourceQuota + LimitRange |
| App config | settings | ConfigMap |
| Credentials | DB password, JWT | Secret |
| Storage | PostgreSQL data | PersistentVolumeClaim |

---

## 1. Project File Structure

```
k8s-taskapi/
├── namespace/
│   ├── namespace.yaml
│   ├── resourcequota.yaml
│   └── limitrange.yaml
├── config/
│   ├── configmap.yaml
│   └── secret.yaml
├── storage/
│   └── postgres-pvc.yaml
├── postgres/
│   ├── deployment.yaml
│   └── service.yaml
├── redis/
│   ├── deployment.yaml
│   └── service.yaml
├── api/
│   ├── deployment.yaml
│   └── service.yaml
├── nginx/
│   ├── deployment.yaml
│   └── service.yaml
├── ingress/
│   └── ingress.yaml
└── deploy.sh
```

---

## 2. Namespace, Quota, LimitRange

### namespace/namespace.yaml
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    env: production
    app: taskapi
  annotations:
    purpose: "TaskAPI production workloads"
```

### namespace/resourcequota.yaml
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
    services: "10"
    persistentvolumeclaims: "5"
    secrets: "20"
    configmaps: "20"
    services.loadbalancers: "1"
```

### namespace/limitrange.yaml
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: production-limitrange
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
        cpu: "2"
        memory: "4Gi"
      min:
        cpu: "50m"
        memory: "64Mi"
    - type: PersistentVolumeClaim
      max:
        storage: 50Gi
      min:
        storage: 1Gi
```

---

## 3. ConfigMap and Secret

### config/configmap.yaml
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: taskapi-config
  namespace: production
data:
  NODE_ENV: "production"
  PORT: "3000"
  LOG_LEVEL: "info"
  APP_NAME: "TaskAPI"
  DB_HOST: "postgres-svc"
  DB_PORT: "5432"
  DB_NAME: "taskdb"
  REDIS_HOST: "redis-svc"
  REDIS_PORT: "6379"
  nginx.conf: |
    events { worker_connections 1024; }
    http {
      upstream api_backend { server api-svc:3000; }
      server {
        listen 80;
        location /api/ {
          proxy_pass http://api_backend/;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_read_timeout 60s;
        }
        location /health { proxy_pass http://api_backend/health; }
        location / {
          return 200 '{"service":"TaskAPI Gateway","status":"running"}';
          add_header Content-Type application/json;
        }
      }
    }
```

### config/secret.yaml
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: taskapi-secrets
  namespace: production
type: Opaque
stringData:
  DB_USER: "taskuser"
  DB_PASSWORD: "supersecretpassword123"
  JWT_SECRET: "my-very-long-jwt-signing-secret-key-must-be-at-least-32-chars"
  POSTGRES_USER: "taskuser"
  POSTGRES_PASSWORD: "supersecretpassword123"
  POSTGRES_DB: "taskdb"
```

---

## 4. Storage

### storage/postgres-pvc.yaml
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data-pvc
  namespace: production
  labels:
    app: postgres
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
```

---

## 5. PostgreSQL

### postgres/deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: production
  labels:
    app: postgres
    tier: database
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  strategy:
    type: Recreate        # RWO volume - must use Recreate
  template:
    metadata:
      labels:
        app: postgres
        tier: database
    spec:
      containers:
        - name: postgres
          image: postgres:15-alpine
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 5432
          envFrom:
            - secretRef:
                name: taskapi-secrets
          env:
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "1"
              memory: "1Gi"
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
          startupProbe:
            exec:
              command: ["sh", "-c", "pg_isready -U $POSTGRES_USER -d $POSTGRES_DB"]
            failureThreshold: 20
            periodSeconds: 5
            timeoutSeconds: 3
          livenessProbe:
            exec:
              command: ["sh", "-c", "pg_isready -U $POSTGRES_USER -d $POSTGRES_DB"]
            periodSeconds: 20
            failureThreshold: 6
            timeoutSeconds: 5
          readinessProbe:
            exec:
              command: ["sh", "-c", "pg_isready -U $POSTGRES_USER -d $POSTGRES_DB"]
            periodSeconds: 10
            failureThreshold: 3
            timeoutSeconds: 5
      volumes:
        - name: postgres-data
          persistentVolumeClaim:
            claimName: postgres-data-pvc
      terminationGracePeriodSeconds: 60
```

### postgres/service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-svc
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: postgres
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
```

---

## 6. Redis

### redis/deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: production
  labels:
    app: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 6379
          command: ["redis-server", "--appendonly", "yes", "--appendfsync", "everysec"]
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          volumeMounts:
            - name: redis-data
              mountPath: /data
          startupProbe:
            exec:
              command: ["redis-cli", "ping"]
            failureThreshold: 10
            periodSeconds: 3
            timeoutSeconds: 2
          livenessProbe:
            exec:
              command: ["redis-cli", "ping"]
            periodSeconds: 20
            failureThreshold: 3
            timeoutSeconds: 3
          readinessProbe:
            exec:
              command: ["redis-cli", "ping"]
            periodSeconds: 10
            failureThreshold: 3
            timeoutSeconds: 3
      volumes:
        - name: redis-data
          emptyDir: {}
      terminationGracePeriodSeconds: 30
```

### redis/service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-svc
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: redis
  ports:
    - name: redis
      port: 6379
      targetPort: 6379
```

---

## 7. TaskAPI

### api/deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: taskapi
  namespace: production
  labels:
    app: taskapi
  annotations:
    kubernetes.io/change-cause: "Initial deployment v1.0"
spec:
  replicas: 3
  minReadySeconds: 10
  selector:
    matchLabels:
      app: taskapi
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    metadata:
      labels:
        app: taskapi
        tier: backend
        version: "1.0"
    spec:
      initContainers:
        - name: wait-for-postgres
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              until nc -z postgres-svc 5432; do
                echo "Waiting for postgres..."; sleep 3
              done
              echo "Postgres ready!"
        - name: wait-for-redis
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              until nc -z redis-svc 6379; do
                echo "Waiting for redis..."; sleep 2
              done
              echo "Redis ready!"
      containers:
        - name: api
          image: taskapi:1.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 3000
          envFrom:
            - configMapRef:
                name: taskapi-config
          env:
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: taskapi-secrets
                  key: DB_USER
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: taskapi-secrets
                  key: DB_PASSWORD
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
          startupProbe:
            httpGet:
              path: /health
              port: 3000
            failureThreshold: 10
            periodSeconds: 5
            timeoutSeconds: 3
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            periodSeconds: 30
            failureThreshold: 3
            timeoutSeconds: 5
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            periodSeconds: 10
            failureThreshold: 3
            successThreshold: 1
            timeoutSeconds: 5
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]
      terminationGracePeriodSeconds: 35
```

### api/service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-svc
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: taskapi
  ports:
    - name: http
      port: 3000
      targetPort: 3000
```

---

## 8. Nginx

### nginx/deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: production
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    metadata:
      labels:
        app: nginx
        tier: proxy
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
          volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/nginx.conf
              subPath: nginx.conf
              readOnly: true
          resources:
            requests:
              cpu: "100m"
              memory: "64Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          livenessProbe:
            httpGet:
              path: /health
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 30
            failureThreshold: 3
            timeoutSeconds: 5
          readinessProbe:
            httpGet:
              path: /health
              port: 80
            initialDelaySeconds: 3
            periodSeconds: 10
            failureThreshold: 3
            timeoutSeconds: 5
      volumes:
        - name: nginx-config
          configMap:
            name: taskapi-config
            items:
              - key: nginx.conf
                path: nginx.conf
      terminationGracePeriodSeconds: 15
```

### nginx/service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: production
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - name: http
      port: 80
      targetPort: 80
      nodePort: 30080
```

---

## 9. Ingress

### ingress/ingress.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: taskapi-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/limit-rps: "50"
spec:
  ingressClassName: nginx
  rules:
    - host: taskapi.local
      http:
        paths:
          - path: /api(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 3000
          - path: /health
            pathType: Exact
            backend:
              service:
                name: api-svc
                port:
                  number: 3000
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-svc
                port:
                  number: 80
```

---

## 10. Deploy Script

### deploy.sh
```bash
#!/bin/bash
set -e

echo "========================================="
echo "  TaskAPI Kubernetes Deployment"
echo "========================================="

echo "Checking Minikube..."
minikube status || minikube start --driver=docker

echo "Enabling Ingress addon..."
minikube addons enable ingress

echo "Step 1: Namespace + Policies..."
kubectl apply -f namespace/namespace.yaml
kubectl apply -f namespace/resourcequota.yaml
kubectl apply -f namespace/limitrange.yaml

echo "Step 2: ConfigMap + Secrets..."
kubectl apply -f config/configmap.yaml
kubectl apply -f config/secret.yaml

echo "Step 3: Storage..."
kubectl apply -f storage/postgres-pvc.yaml

echo "Step 4: PostgreSQL..."
kubectl apply -f postgres/service.yaml
kubectl apply -f postgres/deployment.yaml
kubectl rollout status deployment/postgres -n production --timeout=120s

echo "Step 5: Redis..."
kubectl apply -f redis/service.yaml
kubectl apply -f redis/deployment.yaml
kubectl rollout status deployment/redis -n production --timeout=60s

echo "Step 6: TaskAPI..."
kubectl apply -f api/service.yaml
kubectl apply -f api/deployment.yaml
kubectl rollout status deployment/taskapi -n production --timeout=120s

echo "Step 7: Nginx..."
kubectl apply -f nginx/service.yaml
kubectl apply -f nginx/deployment.yaml
kubectl rollout status deployment/nginx -n production --timeout=60s

echo "Step 8: Ingress..."
kubectl apply -f ingress/ingress.yaml

MINIKUBE_IP=$(minikube ip)
echo ""
echo "========================================="
echo "Deployment Complete!"
echo ""
echo "Add to /etc/hosts:"
echo "  ${MINIKUBE_IP}  taskapi.local"
echo ""
echo "Access URLs:"
echo "  NodePort:  http://${MINIKUBE_IP}:30080"
echo "  Ingress:   http://taskapi.local/api/tasks"
echo "========================================="
kubectl get all -n production
```

---

## 11. Verification Checklist

Run every command and understand every output:

```bash
# 1. All pods Running
kubectl get pods -n production

# 2. All services exist
kubectl get services -n production

# 3. PVC Bound
kubectl get pvc -n production

# 4. Quota usage
kubectl describe resourcequota production-quota -n production

# 5. Ingress has address
kubectl get ingress -n production

# 6. Test via port-forward
kubectl port-forward service/api-svc 8080:3000 -n production &
curl http://localhost:8080/health
kill %1

# 7. Test via NodePort
curl http://$(minikube ip):30080/health

# 8. Test via Ingress
echo "$(minikube ip) taskapi.local" | sudo tee -a /etc/hosts
curl http://taskapi.local/health
curl http://taskapi.local/api/tasks

# 9. DNS resolution from inside pod
kubectl exec -it deploy/taskapi -n production -- wget -qO- http://postgres-svc:5432

# 10. Endpoints show 3 pod IPs
kubectl get endpoints api-svc -n production

# 11. Data persistence test
curl -X POST http://taskapi.local/api/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Test persistence"}'
kubectl rollout restart deployment/postgres -n production
kubectl rollout status deployment/postgres -n production
curl http://taskapi.local/api/tasks   # data still there!

# 12. Rolling update
kubectl set image deployment/taskapi api=taskapi:1.1 -n production
kubectl rollout status deployment/taskapi -n production

# 13. Rollback
kubectl rollout undo deployment/taskapi -n production

# 14. Check all logs
kubectl logs -l app=taskapi -n production --all-containers=true --tail=20

# 15. Resource usage
kubectl top pods -n production
```

---

## 12. Common Debug Scenarios

```bash
# Pod stuck in Init:
kubectl logs <pod> -c wait-for-postgres -n production

# CrashLoopBackOff:
kubectl logs <pod> -n production --previous
kubectl describe pod <pod> -n production   # check exit code in Last State

# Service not routing (endpoints empty):
kubectl get endpoints api-svc -n production
kubectl get pods -l app=taskapi --show-labels -n production

# PVC Pending:
kubectl describe pvc postgres-data-pvc -n production
kubectl get storageclass

# Ingress 404:
kubectl describe ingress taskapi-ingress -n production
kubectl logs -n ingress-nginx $(kubectl get pods -n ingress-nginx -o name | head -1) --tail=30
```

---

## 13. Week 2-3 Concept-to-File Map

| Day | Concept | Applied in |
|---|---|---|
| Day 11 | Control Plane + Nodes | Every object creation — API Server processes each |
| Day 12 | kubectl mastery | deploy.sh + verification commands |
| Day 13 | Pod spec, probes, init containers | api/deployment.yaml |
| Day 14 | Deployments, rolling updates | api/deployment.yaml strategy block |
| Day 15 | Services, ClusterIP, DNS | All */service.yaml files |
| Day 16 | ConfigMap + Secret | config/configmap.yaml + config/secret.yaml |
| Day 17 | PVC, StorageClass | storage/postgres-pvc.yaml |
| Day 18 | Namespace, Quota, LimitRange | namespace/ directory |
| Day 19 | Ingress, path routing | ingress/ingress.yaml |
| Day 20 | Liveness, readiness, startup probes | All deployment.yaml probe sections |

---

## 14. Teardown

```bash
# Remove all workloads
kubectl delete all --all -n production

# Remove PVCs (data loss!)
kubectl delete pvc --all -n production

# Remove configs
kubectl delete configmap,secret --all -n production

# Remove ingress
kubectl delete ingress --all -n production

# Remove entire namespace
kubectl delete namespace production

# Stop Minikube
minikube stop
```

---

## 15. Week 2-3 Final Quiz

1. Name all 4 Control Plane components and what each does.
2. What is the reconciliation loop? Give a specific example from today.
3. Why did we use `strategy: Recreate` for PostgreSQL and `RollingUpdate` for the API?
4. What DNS name does the API use to connect to PostgreSQL?
5. Why did we use `initContainers` in the API deployment?
6. What is the difference between liveness and readiness probes?
7. Why must liveness NOT check the database connection?
8. What does `preStop: sleep 5` do and why is it needed?
9. What is the difference between ConfigMap and Secret?
10. Why does `maxUnavailable: 0, maxSurge: 1` give zero-downtime deployments?
11. What would happen if we deleted the PVC with the postgres pod?
12. What does `subPath: nginx.conf` do in the nginx volume mount?
13. Why does nginx use `NodePort` while the API uses `ClusterIP`?
14. What does the ResourceQuota `services.loadbalancers: "1"` rule enforce?
15. What Ingress annotation strips `/api` prefix before forwarding to api-svc?

<details>
<summary>Answers</summary>

1. API Server (central hub, auth, only etcd writer), etcd (key-value state store, cluster memory), Scheduler (assigns Pods to Nodes using filter+score), Controller Manager (runs reconciliation loops — ReplicaSet, Deployment, Node controllers).

2. Reconciliation: desired state vs actual state, action taken to close the gap. Example: `replicas: 3` in taskapi Deployment. If one pod crashes, ReplicaSet Controller detects actual=2 ≠ desired=3 and immediately creates a replacement pod on any healthy node.

3. PostgreSQL PVC is ReadWriteOnce — only one node can mount it at a time. RollingUpdate creates new pod before removing old one, both trying to mount the same RWO volume simultaneously — new pod gets stuck in Pending. Recreate terminates old pod first (unmounting volume), then starts new pod. API has no volume contention — RollingUpdate gives zero-downtime deployments.

4. `postgres-svc` (short DNS name within the same namespace). Kubernetes CoreDNS resolves this to `postgres-svc.production.svc.cluster.local` → ClusterIP → postgres pod IP.

5. Init containers run to completion before main containers start. Without them, the API starts while PostgreSQL and Redis are still initialising → connection failures → CrashLoopBackOff. Init containers guarantee dependencies are ready before the app boots.

6. Liveness: "Is the container still functioning?" — failure triggers RESTART. Readiness: "Is the container ready for traffic?" — failure triggers REMOVAL from Service Endpoints (no traffic), no restart. Container can pass liveness (alive) but fail readiness (temporarily not ready for requests).

7. If liveness checks DB and DB goes down: liveness fails → API restarts → DB still down → liveness fails again → CrashLoopBackOff. The API pod is perfectly healthy — restarting it doesn't fix the database. Liveness should only check the application process itself.

8. When a Pod is deleted it's immediately removed from Service Endpoints, but iptables rules take 1-2 seconds to propagate across all nodes. During this window requests may still arrive at the terminating pod. `preStop: sleep 5` delays SIGTERM by 5 seconds, giving iptables propagation time to complete before the app starts refusing connections — eliminating dropped requests.

9. ConfigMap: non-sensitive config (app settings, nginx.conf, URLs) — stored as plaintext in etcd. Secret: sensitive data (passwords, JWT keys) — stored as base64 in etcd, memory-only on nodes, independently RBAC-controllable. Both inject the same ways (env, envFrom, volumes) but signal different intent and have different security controls.

10. `maxUnavailable: 0`: never reduce below 3 running pods (always full capacity). `maxSurge: 1`: allow 1 extra pod (4 total during update). Process: start new pod → passes readiness → Kubernetes counts it as available → removes one old pod. At every moment: 3-4 pods running, always at least 3 serving traffic. Zero downtime throughout.

11. Minikube's standard StorageClass has `reclaimPolicy: Delete`. Deleting the PVC would trigger automatic deletion of the underlying PersistentVolume and all PostgreSQL data stored on it — unrecoverable. In production always use `reclaimPolicy: Retain` for databases.

12. `subPath: nginx.conf` mounts only the specific `nginx.conf` key from the ConfigMap as a single file at `/etc/nginx/nginx.conf` — NOT replacing the entire `/etc/nginx/` directory. Without subPath, mounting at that path would replace the whole directory, removing all other nginx config files needed for operation.

13. Nginx is the entry point for external traffic — it needs to be reachable from outside the cluster, hence NodePort (port 30080). The API only needs to be reachable from Nginx and the Ingress controller — both inside the cluster — so ClusterIP is sufficient and more secure (no external exposure).

14. It limits the namespace to creating at most 1 LoadBalancer-type Service. LoadBalancer Services create real cloud load balancers (AWS ELB etc.) which cost money. This quota prevents accidental or malicious creation of expensive cloud resources from this namespace.

15. `nginx.ingress.kubernetes.io/rewrite-target: /$2` combined with path `/api(/|$)(.*)` captures everything after `/api` in capture group `$2` and rewrites the URL to just that captured part. Request `/api/tasks` → forwarded as `/tasks` to api-svc. This strips the `/api` prefix before the request reaches the backend.

</details>

---

## 16. Summary

Today you built and deployed a complete production-pattern Kubernetes application:
- Namespace with ResourceQuota and LimitRange
- ConfigMap with app settings and nginx.conf mounted as a file
- Secret with DB credentials and JWT key using stringData
- PVC for PostgreSQL data persistence
- PostgreSQL with Recreate strategy and exec probes
- Redis with persistence configuration
- TaskAPI with 3 replicas, init containers, separate probes, preStop hook
- Nginx with ConfigMap-mounted config using subPath
- All ClusterIP Services for internal DNS-based discovery
- NodePort for external Nginx access
- Ingress with path-based routing and rewrite annotation
- Ordered deploy.sh script with rollout status checks
- 15-item verification checklist testing every layer

**Week 4 starts on Day 22:** StatefulSets — the right way to run databases with stable pod identities, ordered deployment, and stable network names.
