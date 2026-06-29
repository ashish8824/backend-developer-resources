# Day 30: Final Capstone — Production-Grade Kubernetes System From Scratch

**Goal for today:** Apply everything from Days 1–29 in one complete, deployable project. Design, build, and document a production-grade Kubernetes system that demonstrates Docker best practices, Kubernetes orchestration, Helm packaging, EKS deployment, CI/CD automation, monitoring, networking security, and RBAC — everything a senior engineer or hiring team would expect to see. By the end, you should have a portfolio-ready project and be able to teach the entire 30-day curriculum from memory.

**Format:** Architecture-first, then implementation layer by layer, then a complete 30-day review.

---

## 0. The Project — PulseBloom on Kubernetes

You're migrating **PulseBloom** (your production behavioral analytics platform) from ECS Fargate to Kubernetes. This is the perfect capstone because:
- You already understand the application
- It's real production software, not a toy
- It demonstrates migration from ECS to EKS — a highly valued skill
- It touches every concept from the 30 days

**Stack:**
| Service | Technology | K8s Resource |
|---|---|---|
| API | Node.js/TypeScript | Deployment + Service |
| Frontend | React (Nginx) | Deployment + Service + Ingress |
| Database | PostgreSQL 15 | StatefulSet + Headless Service + PVC |
| Cache | Redis 7 | StatefulSet + Headless Service |
| Worker | Background job processor | Deployment |
| Migrations | Prisma migrate | Job (runs before API) |
| Backups | pg_dump to S3 | CronJob |
| Log collector | Fluentd | DaemonSet |
| Monitoring | Prometheus + Grafana | Helm release |

---

## 1. Complete Repository Structure

```
pulsebloom-k8s/
├── .github/
│   └── workflows/
│       ├── ci.yml
│       ├── deploy-staging.yml
│       └── deploy-prod.yml
│
├── charts/
│   ├── pulsebloom/                  ← main Helm chart
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   ├── values-staging.yaml
│   │   ├── values-prod.yaml
│   │   └── templates/
│   │       ├── _helpers.tpl
│   │       ├── namespace.yaml
│   │       ├── resourcequota.yaml
│   │       ├── limitrange.yaml
│   │       ├── configmap.yaml
│   │       ├── secret.yaml          ← ExternalSecret (from AWS Secrets Manager)
│   │       ├── serviceaccount.yaml
│   │       ├── rbac.yaml
│   │       ├── networkpolicies.yaml
│   │       ├── postgres-statefulset.yaml
│   │       ├── redis-statefulset.yaml
│   │       ├── api-deployment.yaml
│   │       ├── worker-deployment.yaml
│   │       ├── frontend-deployment.yaml
│   │       ├── migration-job.yaml
│   │       ├── backup-cronjob.yaml
│   │       ├── fluentd-daemonset.yaml
│   │       ├── ingress.yaml
│   │       ├── prometheusrule.yaml
│   │       └── NOTES.txt
│   └── migration/                   ← standalone migration chart
│       ├── Chart.yaml
│       └── templates/
│           └── job.yaml
│
├── cluster/
│   ├── eks-cluster-config.yaml      ← eksctl config
│   ├── storageclass.yaml
│   ├── aws-lbc.yaml                 ← AWS Load Balancer Controller config
│   └── external-secrets.yaml       ← External Secrets Operator config
│
└── docs/
    ├── architecture.md
    ├── runbooks/
    │   ├── deployment.md
    │   ├── rollback.md
    │   └── incident-response.md
    └── adr/                         ← Architecture Decision Records
```

---

## 2. Layer 1 — Cluster Setup (Days 11, 28)

### `cluster/eks-cluster-config.yaml`

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: pulsebloom-prod
  region: ap-south-1
  version: "1.29"
  tags:
    Project: pulsebloom
    Environment: production
    ManagedBy: eksctl

vpc:
  cidr: "10.0.0.0/16"
  nat:
    gateway: HighlyAvailable

iam:
  withOIDC: true             # enables IRSA

managedNodeGroups:
  - name: general
    instanceType: t3.xlarge
    minSize: 2
    maxSize: 10
    desiredCapacity: 3
    availabilityZones: [ap-south-1a, ap-south-1b, ap-south-1c]
    volumeSize: 50
    labels:
      role: general
    iam:
      attachPolicyARNs:
        - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
        - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
        - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

addons:
  - name: vpc-cni
    version: latest
  - name: coredns
    version: latest
  - name: kube-proxy
    version: latest
  - name: aws-ebs-csi-driver
    version: latest
    wellKnownPolicies:
      ebsCSIController: true
```

```bash
# One-time cluster setup
eksctl create cluster -f cluster/eks-cluster-config.yaml
kubectl apply -f cluster/storageclass.yaml

# Install AWS Load Balancer Controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=pulsebloom-prod

# Install External Secrets Operator
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets --create-namespace
```

---

## 3. Layer 2 — Helm Chart Core

### `charts/pulsebloom/Chart.yaml`

```yaml
apiVersion: v2
name: pulsebloom
description: "PulseBloom — Behavioral Analytics Platform"
type: application
version: 2.1.0
appVersion: "2.1.0"
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
  - name: redis
    version: "17.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

### `charts/pulsebloom/values.yaml` (key sections)

```yaml
global:
  environment: production
  imageRegistry: 123456789.dkr.ecr.ap-south-1.amazonaws.com

api:
  replicaCount: 3
  image:
    repository: pulsebloom-api
    tag: "2.1.0"
    pullPolicy: Always
  resources:
    requests: { cpu: "250m", memory: "256Mi" }
    limits:   { cpu: "1",    memory: "1Gi" }
  autoscaling:
    enabled: true
    minReplicas: 3
    maxReplicas: 20
    targetCPUUtilizationPercentage: 70

worker:
  replicaCount: 2
  image:
    repository: pulsebloom-worker
    tag: "2.1.0"
  resources:
    requests: { cpu: "500m", memory: "512Mi" }
    limits:   { cpu: "2",    memory: "2Gi" }

frontend:
  replicaCount: 2
  image:
    repository: pulsebloom-frontend
    tag: "2.1.0"

ingress:
  enabled: true
  className: alb
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-south-1:123:certificate/abc
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: pulsebloom.com
      paths: [{ path: /, service: frontend, port: 80 }]
    - host: api.pulsebloom.com
      paths: [{ path: /, service: api, port: 3000 }]

postgresql:
  enabled: false   # using custom StatefulSet for more control
  external:
    host: postgres-headless
    port: 5432
    database: pulsebloom
    existingSecret: pulsebloom-secrets
    secretKey: DB_PASSWORD

redis:
  enabled: false   # using custom StatefulSet
  external:
    host: redis-headless
    port: 6379

monitoring:
  enabled: true
  prometheusRule: true

security:
  podSecurityStandard: restricted
  networkPolicies: true
```

---

## 4. Layer 3 — Namespace & Resource Governance (Day 18)

### `templates/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: {{ .Release.Namespace }}
  labels:
    env: {{ .Values.global.environment }}
    app: pulsebloom
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
    kubernetes.io/metadata.name: {{ .Release.Namespace }}
```

### `templates/resourcequota.yaml`

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pulsebloom-quota
  namespace: {{ .Release.Namespace }}
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"
    services: "20"
    persistentvolumeclaims: "10"
    secrets: "30"
    services.loadbalancers: "2"
```

---

## 5. Layer 4 — Security: RBAC & ServiceAccounts (Day 29)

### `templates/serviceaccount.yaml`

```yaml
# API ServiceAccount — IRSA for S3 uploads
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pulsebloom-api-sa
  namespace: {{ .Release.Namespace }}
  annotations:
    eks.amazonaws.com/role-arn: {{ .Values.irsa.apiRoleArn }}
automountServiceAccountToken: false

---
# Worker ServiceAccount — IRSA for SES email sending
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pulsebloom-worker-sa
  namespace: {{ .Release.Namespace }}
  annotations:
    eks.amazonaws.com/role-arn: {{ .Values.irsa.workerRoleArn }}
automountServiceAccountToken: false
```

### `templates/rbac.yaml`

```yaml
# Role: API can only read its own configmap and secret
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pulsebloom-api-role
  namespace: {{ .Release.Namespace }}
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["pulsebloom-config"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["pulsebloom-secrets"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pulsebloom-api-binding
  namespace: {{ .Release.Namespace }}
subjects:
  - kind: ServiceAccount
    name: pulsebloom-api-sa
    namespace: {{ .Release.Namespace }}
roleRef:
  kind: Role
  name: pulsebloom-api-role
  apiGroup: rbac.authorization.k8s.io
```

---

## 6. Layer 5 — Secrets from AWS (Days 16, 28, 29)

### `templates/secret.yaml`

```yaml
# Pull all secrets from AWS Secrets Manager — never store in Git
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: pulsebloom-secrets
  namespace: {{ .Release.Namespace }}
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: pulsebloom-secrets
    creationPolicy: Owner
  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: production/pulsebloom
        property: db_password
    - secretKey: JWT_SECRET
      remoteRef:
        key: production/pulsebloom
        property: jwt_secret
    - secretKey: REDIS_PASSWORD
      remoteRef:
        key: production/pulsebloom
        property: redis_password
    - secretKey: RAZORPAY_KEY_SECRET
      remoteRef:
        key: production/pulsebloom
        property: razorpay_key_secret
    - secretKey: SENDGRID_API_KEY
      remoteRef:
        key: production/pulsebloom
        property: sendgrid_api_key
```

---

## 7. Layer 6 — StatefulSets for PostgreSQL & Redis (Day 22)

### `templates/postgres-statefulset.yaml`

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: {{ .Release.Namespace }}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  serviceName: postgres-headless
  podManagementPolicy: OrderedReady
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0
  template:
    metadata:
      labels:
        app: postgres
        tier: database
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 999       # postgres user
        fsGroup: 999
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: postgres
          image: postgres:15-alpine
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: false  # postgres needs writable FS
            capabilities:
              drop: ["ALL"]
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: pulsebloom-secrets
                  key: DB_PASSWORD
            - name: POSTGRES_USER
              value: pulsebloom
            - name: POSTGRES_DB
              value: pulsebloom
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          ports:
            - containerPort: 5432
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2"
              memory: "4Gi"
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
          startupProbe:
            exec:
              command: ["sh", "-c", "pg_isready -U pulsebloom"]
            failureThreshold: 30
            periodSeconds: 5
          livenessProbe:
            exec:
              command: ["sh", "-c", "pg_isready -U pulsebloom"]
            periodSeconds: 20
            failureThreshold: 6
          readinessProbe:
            exec:
              command: ["sh", "-c", "pg_isready -U pulsebloom"]
            periodSeconds: 10
            failureThreshold: 3
      terminationGracePeriodSeconds: 120
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: gp3-encrypted
        resources:
          requests:
            storage: 50Gi

---
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: {{ .Release.Namespace }}
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

---

## 8. Layer 7 — API Deployment (Days 13, 14, 20)

### `templates/api-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pulsebloom-api
  namespace: {{ .Release.Namespace }}
  annotations:
    kubernetes.io/change-cause: "Deploy {{ .Values.api.image.tag }}"
spec:
  replicas: {{ .Values.api.replicaCount }}
  minReadySeconds: 15
  selector:
    matchLabels:
      app: pulsebloom-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    metadata:
      labels:
        app: pulsebloom-api
        version: {{ .Values.api.image.tag | quote }}
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
        prometheus.io/path: "/metrics"
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      serviceAccountName: pulsebloom-api-sa
      automountServiceAccountToken: false

      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        seccompProfile:
          type: RuntimeDefault

      initContainers:
        - name: wait-for-postgres
          image: busybox:1.36
          command: ["sh", "-c",
            "until nc -z postgres-headless 5432; do sleep 2; done"]
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]

      containers:
        - name: api
          image: "{{ .Values.global.imageRegistry }}/{{ .Values.api.image.repository }}:{{ .Values.api.image.tag }}"
          imagePullPolicy: {{ .Values.api.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 3000
            - name: metrics
              containerPort: 9090

          envFrom:
            - configMapRef:
                name: pulsebloom-config

          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: pulsebloom-secrets
                  key: DB_PASSWORD
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: pulsebloom-secrets
                  key: JWT_SECRET
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace

          resources:
            {{- toYaml .Values.api.resources | nindent 12 }}

          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]

          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: app-logs
              mountPath: /app/logs

          startupProbe:
            httpGet:
              path: /health
              port: 3000
            failureThreshold: 12
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
              path: /readyz
              port: 3000
            periodSeconds: 10
            failureThreshold: 3
            timeoutSeconds: 5

          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]

      volumes:
        - name: tmp
          emptyDir: {}
        - name: app-logs
          emptyDir: {}

      terminationGracePeriodSeconds: 35

      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: pulsebloom-api
```

---

## 9. Layer 8 — Batch Workloads (Day 23)

### `templates/migration-job.yaml`

```yaml
{{- if .Values.migration.run }}
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration-{{ .Values.api.image.tag | replace "." "-" }}
  namespace: {{ .Release.Namespace }}
  annotations:
    helm.sh/hook: pre-upgrade,pre-install
    helm.sh/hook-weight: "-5"
    helm.sh/hook-delete-policy: before-hook-creation
spec:
  backoffLimit: 3
  activeDeadlineSeconds: 600
  ttlSecondsAfterFinished: 86400
  template:
    spec:
      restartPolicy: OnFailure
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        seccompProfile:
          type: RuntimeDefault
      initContainers:
        - name: wait-for-postgres
          image: busybox:1.36
          command: ["sh", "-c",
            "until nc -z postgres-headless 5432; do sleep 2; done"]
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
      containers:
        - name: migrate
          image: "{{ .Values.global.imageRegistry }}/{{ .Values.api.image.repository }}:{{ .Values.api.image.tag }}"
          command: ["npx", "prisma", "migrate", "deploy"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: pulsebloom-secrets
                  key: DATABASE_URL
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
{{- end }}
```

### `templates/backup-cronjob.yaml`

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: {{ .Release.Namespace }}
spec:
  schedule: "0 2 * * *"        # 2am daily
  timeZone: "Asia/Kolkata"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 7
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 3600
      ttlSecondsAfterFinished: 604800
      template:
        spec:
          serviceAccountName: pulsebloom-api-sa    # IRSA for S3
          restartPolicy: OnFailure
          securityContext:
            runAsNonRoot: true
            runAsUser: 999
            seccompProfile:
              type: RuntimeDefault
          containers:
            - name: backup
              image: postgres:15-alpine
              command:
                - sh
                - -c
                - |
                  TIMESTAMP=$(date +%Y%m%d_%H%M%S)
                  FILENAME="pulsebloom_${TIMESTAMP}.sql.gz"
                  pg_dump -h postgres-headless -U pulsebloom pulsebloom \
                    | gzip \
                    | aws s3 cp - s3://pulsebloom-backups/postgres/${FILENAME}
                  echo "Backup uploaded: ${FILENAME}"
              env:
                - name: PGPASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: pulsebloom-secrets
                      key: DB_PASSWORD
              securityContext:
                allowPrivilegeEscalation: false
                capabilities:
                  drop: ["ALL"]
              resources:
                requests: { cpu: "200m", memory: "256Mi" }
                limits:   { cpu: "500m", memory: "512Mi" }
```

---

## 10. Layer 9 — Networking (Days 15, 19, 25)

### `templates/networkpolicies.yaml`

```yaml
# Default deny all
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: {{ .Release.Namespace }}
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]

---
# DNS egress for all pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: {{ .Release.Namespace }}
spec:
  podSelector: {}
  policyTypes: [Egress]
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
# API: ingress from ALB + egress to postgres/redis/external
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
  namespace: {{ .Release.Namespace }}
spec:
  podSelector:
    matchLabels:
      app: pulsebloom-api
  policyTypes: [Ingress, Egress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - port: 3000
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - port: 5432
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - port: 6379
    - ports:
        - port: 443     # external HTTPS (SendGrid, Razorpay, etc.)
    - ports:
        - port: 587     # SMTP
```

### `templates/ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: pulsebloom-ingress
  namespace: {{ .Release.Namespace }}
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: {{ .Values.ingress.certificateArn }}
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/group.name: pulsebloom-alb
spec:
  ingressClassName: alb
  rules:
    - host: pulsebloom.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: pulsebloom-frontend
                port:
                  number: 80
    - host: api.pulsebloom.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: pulsebloom-api
                port:
                  number: 3000
```

---

## 11. Layer 10 — Monitoring (Day 26)

### `templates/prometheusrule.yaml`

```yaml
{{- if .Values.monitoring.prometheusRule }}
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: pulsebloom-alerts
  namespace: {{ .Release.Namespace }}
  labels:
    release: prometheus
spec:
  groups:
    - name: pulsebloom.critical
      rules:
        - alert: APIHighErrorRate
          expr: |
            100 * sum(rate(http_requests_total{status=~"5..",namespace="{{ .Release.Namespace }}"}[5m]))
                / sum(rate(http_requests_total{namespace="{{ .Release.Namespace }}"}[5m])) > 5
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "PulseBloom API error rate above 5%"
            description: "Current error rate: {{ "{{" }} $value | humanizePercentage {{ "}}" }}"

        - alert: PodCrashLooping
          expr: |
            increase(kube_pod_container_status_restarts_total{namespace="{{ .Release.Namespace }}"}[1h]) > 3
          for: 0m
          labels:
            severity: critical
          annotations:
            summary: "Pod {{ "{{" }} $labels.pod {{ "}}" }} crash looping"

        - alert: DatabaseDown
          expr: |
            kube_pod_status_ready{namespace="{{ .Release.Namespace }}",
              pod=~"postgres-.*", condition="true"} < 1
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "PostgreSQL is down!"
{{- end }}
```

---

## 12. Layer 11 — CI/CD Pipeline (Day 27)

### `.github/workflows/deploy-prod.yml` (abbreviated key sections)

```yaml
name: Deploy PulseBloom to Production

on:
  push:
    branches: [main]

env:
  AWS_REGION: ap-south-1
  ECR_REGISTRY: 123456789.dkr.ecr.ap-south-1.amazonaws.com
  EKS_CLUSTER: pulsebloom-prod

jobs:
  build-scan-push:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      - uses: aws-actions/amazon-ecr-login@v2
        id: login-ecr
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ env.ECR_REGISTRY }}/pulsebloom-api:${{ github.sha }}
          platforms: linux/amd64
          cache-from: type=gha
          cache-to: type=gha,mode=max
      - uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.ECR_REGISTRY }}/pulsebloom-api:${{ github.sha }}
          severity: CRITICAL,HIGH
          exit-code: '1'

  deploy-production:
    needs: build-scan-push
    runs-on: ubuntu-latest
    environment: production      # requires manual approval
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      - run: aws eks update-kubeconfig --name ${{ env.EKS_CLUSTER }} --region ${{ env.AWS_REGION }}
      - uses: azure/setup-helm@v3
      - name: Deploy with Helm
        run: |
          helm upgrade --install pulsebloom ./charts/pulsebloom \
            --namespace production \
            --create-namespace \
            -f charts/pulsebloom/values-prod.yaml \
            --set api.image.tag=${{ github.sha }} \
            --set worker.image.tag=${{ github.sha }} \
            --set frontend.image.tag=${{ github.sha }} \
            --set migration.run=true \
            --atomic \
            --timeout 15m \
            --cleanup-on-fail
```

---

## 13. The Complete 30-Day Concept Map

This is your master teaching reference — every concept from the course applied in this capstone:

| Day | Concept | Applied in PulseBloom |
|---|---|---|
| 1 | Containers vs VMs | Every service runs in a container |
| 2 | Docker CLI, images, containers | Docker build/push in CI pipeline |
| 3 | Dockerfile, layers, caching | Multi-stage Dockerfile for API |
| 4 | Docker networking, DNS | Service names used in app config |
| 5 | Volumes, persistence | PostgreSQL StatefulSet PVC |
| 6 | Docker Compose | Development docker-compose.yml |
| 7 | Capstone | Day 7 project → now on K8s |
| 8 | Image optimization | Alpine base, multi-stage, non-root |
| 9 | Registries, ECR | CI pushes to ECR, EKS pulls from ECR |
| 10 | Why orchestration | SPOF, scaling, zero-downtime needs |
| 11 | K8s architecture | Control plane manages all objects |
| 12 | kubectl, Minikube | Local dev before EKS |
| 13 | Pods, probes | API Deployment with 3 probe types |
| 14 | Deployments, rolling updates | API/Worker/Frontend Deployments |
| 15 | Services, DNS | ClusterIP services + headless for StatefulSets |
| 16 | ConfigMaps, Secrets | ExternalSecret from AWS Secrets Manager |
| 17 | PVCs, StorageClass | gp3-encrypted StorageClass, postgres PVC |
| 18 | Namespaces, quotas | production namespace with ResourceQuota |
| 19 | Ingress | ALB Ingress with SSL for pulsebloom.com |
| 20 | Health probes | Separate /health (liveness) and /readyz (readiness) |
| 21 | Week 3 Capstone | Full deployment on Minikube |
| 22 | StatefulSets | PostgreSQL + Redis as StatefulSets |
| 23 | DaemonSets, Jobs, CronJobs | Fluentd DaemonSet, migration Job, backup CronJob |
| 24 | Helm | Complete Helm chart for entire stack |
| 25 | NetworkPolicies | Zero-trust: default-deny + explicit allows |
| 26 | Monitoring | Prometheus scraping + Grafana dashboards + AlertManager |
| 27 | CI/CD | GitHub Actions: build→scan→push→migrate→deploy |
| 28 | AWS EKS | EKS cluster, IRSA, AWS Load Balancer Controller |
| 29 | RBAC, Pod Security | Dedicated SAs, restricted PSS, securityContext |
| 30 | Capstone | All of the above, wired together |

---

## 14. Deployment Commands — Zero to Production

```bash
# ── ONE-TIME CLUSTER SETUP ─────────────────────────────────────────────────────
eksctl create cluster -f cluster/eks-cluster-config.yaml        # ~20 min
kubectl apply -f cluster/storageclass.yaml
helm install aws-load-balancer-controller eks/aws-load-balancer-controller ...
helm install external-secrets external-secrets/external-secrets ...
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring ...

# ── EVERY DEPLOYMENT (automated by CI/CD) ─────────────────────────────────────
helm upgrade --install pulsebloom ./charts/pulsebloom \
  --namespace production \
  --create-namespace \
  -f charts/pulsebloom/values-prod.yaml \
  --set api.image.tag=abc1234 \
  --atomic --timeout 15m

# ── ROLLBACK ──────────────────────────────────────────────────────────────────
helm rollback pulsebloom -n production           # to previous version
helm rollback pulsebloom 3 -n production         # to specific revision

# ── SCALE ─────────────────────────────────────────────────────────────────────
kubectl scale deployment pulsebloom-api --replicas=10 -n production

# ── DEBUG ─────────────────────────────────────────────────────────────────────
kubectl get all -n production
kubectl top pods -n production
kubectl logs -l app=pulsebloom-api -n production -f
helm history pulsebloom -n production
```

---

## 15. Final 30-Day Quiz — Test Everything

Test yourself on all 30 days with these 30 questions:

1. What is the kernel-level difference between a Docker container and a VM?
2. Explain Docker's layer caching with the `package.json` copy trick.
3. What does `FROM node:18-alpine AS builder` do in a multi-stage Dockerfile?
4. How does Docker DNS work in custom bridge networks?
5. What is the difference between a named volume and a bind mount?
6. What does `depends_on: condition: service_healthy` require in Docker Compose?
7. Name three production Dockerfile best practices from Day 8.
8. What tagging strategy should you use for production Docker images and why?
9. What are the 6 problems Docker Compose cannot solve at production scale?
10. Name all 4 Control Plane components and what each does.
11. What is the kubectl command to see what permissions a ServiceAccount has?
12. What is a Pod? What two things do containers in a Pod always share?
13. What does `maxUnavailable: 0, maxSurge: 1` mean for a rolling update?
14. What is the difference between a ClusterIP and a LoadBalancer Service?
15. When should you use `stringData` in a Kubernetes Secret?
16. What is `volumeBindingMode: WaitForFirstConsumer` and why is it critical on EKS?
17. What does `ResourceQuota` + `LimitRange` enforce together?
18. What is the difference between an Ingress and an Ingress Controller?
19. Why must liveness probes NOT check external dependencies?
20. What are the 5 StatefulSet guarantees that a Deployment doesn't provide?
21. What is the difference between a Job with `restartPolicy: Never` vs `OnFailure`?
22. What are the 3 things Helm adds over raw kubectl YAML?
23. What is the AND vs OR distinction in NetworkPolicy `from` selectors?
24. What is the difference between a Prometheus Counter and a Gauge?
25. What does `--atomic` do in `helm upgrade`?
26. What is IRSA and why is it better than static AWS credentials in Pods?
27. What are the three Pod Security Standard profiles?
28. What does `allowPrivilegeEscalation: false` prevent?
29. What is the EKS `aws-auth` ConfigMap used for?
30. Draw the full request lifecycle: user browser → pulsebloom.com → API pod responds.

<details>
<summary>Answers</summary>

1. VMs have a full guest OS kernel per VM. Containers share the host OS kernel and only isolate the application filesystem and processes via Linux namespaces and cgroups. Containers are MBs vs GBs, start in milliseconds vs minutes.

2. Copy package.json first, run npm install (this layer is cached since package.json rarely changes), THEN copy the rest of the code. When only app code changes, the npm install layer is served from cache — saving 1-3 minutes per build.

3. It starts a new build stage named "builder" using node:18-alpine. Subsequent instructions build in this stage. A later `FROM` starts a fresh stage, and `COPY --from=builder` pulls only specific files from the builder stage — leaving behind all build tools, devDependencies, and source files.

4. Containers on a custom bridge network register with Docker's embedded CoreDNS. Any container on the same network can reach another by container name. Docker resolves the name to the container's current IP automatically. The default bridge network does NOT have DNS — custom bridges are required.

5. Named volume: Docker-managed storage in Docker's internal area, referenced by name, portable across hosts. Bind mount: maps a specific host filesystem path into the container, tied to that exact path. Use named volumes for production persistence, bind mounts for local dev code synchronisation.

6. `depends_on: condition: service_healthy` makes Compose wait until the dependency container passes its defined `healthcheck` test (not just "container started") before starting the dependent service. Requires a `healthcheck` block on the dependency service.

7. Any three of: use alpine/slim base image, multi-stage builds (no build tools in final image), non-root USER instruction, pin image tags (no `latest`), CMD in exec form, `.dockerignore` to exclude secrets and junk, chain RUN commands with `&&`.

8. Use Git SHA tags (e.g., `abc1234`) for immutability and traceability — every image maps to exactly one commit. Also tag with branch names (`main`, `latest`) as mutable pointers. SHA tags enable rollback to any specific version; `latest` enables scripts to always use newest. Never use `latest` alone.

9. Single point of failure (one machine = one outage), no true horizontal scaling (all containers on same host), no zero-downtime deployments, no automatic cross-node self-healing, no intelligent resource-aware scheduling, limited service discovery and load balancing.

10. API Server: central hub, REST API, authentication, only component that reads/writes etcd. etcd: distributed key-value store, single source of truth for all cluster state. Scheduler: assigns unscheduled Pods to Nodes using filter+score algorithm. Controller Manager: runs reconciliation loops (ReplicaSet, Deployment, Node controllers) to maintain desired state.

11. `kubectl auth can-i --list -n <namespace> --as=system:serviceaccount:<namespace>:<sa-name>`

12. A Pod is the smallest deployable unit in Kubernetes — a wrapper around one or more containers that always run together on the same node. Containers in a Pod share: (1) network namespace — same IP, communicate via localhost; (2) volumes that are explicitly mounted to multiple containers.

13. During a rolling update, `maxUnavailable: 0` means the replica count never drops below the desired number (always full capacity). `maxSurge: 1` means one extra Pod can temporarily exist above desired count. The update proceeds by adding one new Pod, waiting for it to pass readiness, then removing one old Pod — zero downtime throughout.

14. ClusterIP creates a stable virtual IP accessible only within the cluster, load-balancing to matching Pods. LoadBalancer creates a ClusterIP AND provisions an external cloud load balancer (AWS ELB) with a public IP/DNS, making the Service accessible from the internet. Every LoadBalancer Service creates a separate cloud LB — use Ingress to share one LB.

15. `stringData` lets you write plain text values in a Secret YAML — Kubernetes automatically base64-encodes them on storage. `data` requires you to manually base64-encode every value before writing. Use `stringData` for readability and to avoid encoding errors. Note: `stringData` is write-only; `kubectl get secret -o yaml` only shows `data` (encoded).

16. `WaitForFirstConsumer` delays PVC binding until a Pod using the PVC is scheduled to a node. EBS volumes are AZ-specific — if the PVC binds in ap-south-1a but the Pod lands on a node in ap-south-1b, the Pod can't mount the volume. WaitForFirstConsumer ensures provisioning happens in the same AZ as the scheduled Pod.

17. ResourceQuota sets hard limits on TOTAL resources consumed by all objects in a namespace combined. LimitRange sets default and maximum resources PER CONTAINER. Together: LimitRange ensures every container has resource declarations (satisfying ResourceQuota's requirement), and ResourceQuota ensures the total across all containers stays within the namespace budget.

18. Ingress is a Kubernetes API object (YAML) that defines HTTP routing rules — which hostname+path maps to which Service. Ingress Controller is the running application (e.g., nginx-ingress, AWS Load Balancer Controller) that reads Ingress objects and actually implements the routing. Without a Controller, Ingress objects have zero effect.

19. If liveness checks a dependency (DB, Redis, external API) and that dependency goes down: liveness fails → Pod restarts → dependency still down → liveness fails again → CrashLoopBackOff. The Pod is healthy — the dependency isn't. Restarting the Pod doesn't fix the dependency but does take the healthy Pod out of service. Liveness should only verify the application process itself isn't stuck/deadlocked.

20. (1) Stable, unique Pod names with ordinal index (postgres-0, postgres-1). (2) Stable DNS hostname per Pod via Headless Service. (3) Stable persistent storage — dedicated PVC per Pod via volumeClaimTemplates, permanently bound. (4) Ordered deployment — each Pod must be Running+Ready before next starts. (5) Ordered scaling and rolling updates — reverse order.

21. `restartPolicy: Never`: if Pod fails, Job creates a BRAND NEW Pod for retry (old Pod preserved for debugging). `OnFailure`: if container exits non-zero, kubelet restarts the same container in the same Pod. `Never` accumulates failed Pods; `OnFailure` reuses the same Pod. Both respect `backoffLimit`.

22. (1) Parameterisation — one set of templates configurable for any environment via values.yaml and `-f overrides.yaml`. (2) Versioned releases — every deploy is a numbered revision; `helm rollback` restores any previous state instantly. (3) Dependency management — declare dependency on PostgreSQL chart version X; Helm handles downloading and deploying it.

23. AND (same list item with both `podSelector` and `namespaceSelector`): BOTH conditions must match — "pods labelled app=nginx that are also in namespace labelled env=prod." OR (separate list items — each with a dash): EITHER condition matches — "any pod labelled app=nginx regardless of namespace, OR any pod in a namespace labelled env=prod regardless of app label."

24. Counter: monotonically increasing, only goes up, resets to 0 on process restart. Use for total events (requests, errors, bytes). Always use `rate()` in PromQL, never raw value. Gauge: can go up or down, represents current state. Use for current measurements (memory usage, active connections, queue depth). Use raw value in PromQL.

25. `--atomic` makes the Helm upgrade all-or-nothing: if any resource fails to become healthy within the `--timeout`, Helm automatically rolls back to the previous successful release revision. Without it, failed upgrades leave the cluster partially upgraded, requiring manual investigation and rollback.

26. IRSA (IAM Roles for Service Accounts) links a Kubernetes ServiceAccount to an AWS IAM Role via OIDC federation. When a Pod uses that ServiceAccount, AWS STS provides temporary auto-rotating credentials. No static access keys are stored anywhere. Static keys are: a permanent security risk if leaked, require manual rotation, can't be scoped to individual Pods, and violate least-privilege. IRSA credentials auto-expire, auto-rotate, and are scoped precisely.

27. Privileged: no restrictions — for trusted infrastructure (DaemonSets). Baseline: blocks dangerous capabilities (NET_ADMIN, SYS_ADMIN, privileged containers, hostPath) — for general apps. Restricted: enforces all best practices (non-root required, all caps dropped, no privilege escalation, seccomp required) — for maximum security.

28. `allowPrivilegeEscalation: false` prevents a process from gaining more privileges than its parent. Specifically, it blocks execution of setuid/setgid binaries that could elevate to root. Without it, even a non-root process could run `sudo` or a setuid binary to become root, defeating `runAsNonRoot`.

29. The `aws-auth` ConfigMap in `kube-system` maps AWS IAM identities (IAM users, IAM roles) to Kubernetes RBAC identities (usernames and groups). When any tool authenticates to EKS using AWS credentials, the EKS API Server checks `aws-auth` to determine what Kubernetes permissions that identity has. Without an entry, no AWS identity can use `kubectl` against the cluster.

30. Browser → DNS: api.pulsebloom.com → AWS Route53 → ALB DNS. → ALB receives HTTPS request, terminates TLS using ACM certificate. → ALB checks target group (Pod IPs registered via AWS Load Balancer Controller). → ALB forwards HTTP to pulsebloom-api Pod IP:3000 directly (target-type: ip). → Pod receives request, middleware records metrics, checks DB/Redis for data. → Response flows back through ALB → browser. In parallel: kube-proxy maintains iptables rules, CoreDNS resolves service names, Prometheus scrapes /metrics every 15s, Fluentd ships logs to centralised storage.

</details>

---

## 16. You Are Now a Docker & Kubernetes Engineer

You have completed 30 days of structured, teach-ready learning. Here is what you can now do:

```
DOCKER (Days 1–9):
  ✅ Explain containers vs VMs from first principles
  ✅ Write production-grade Dockerfiles with multi-stage builds
  ✅ Design Docker Compose stacks with proper networking and volumes
  ✅ Optimise images, scan for vulnerabilities, push to ECR

KUBERNETES CORE (Days 10–21):
  ✅ Explain K8s architecture: control plane, nodes, reconciliation loop
  ✅ Deploy any stateless workload with Deployments
  ✅ Expose services with ClusterIP, NodePort, LoadBalancer, Ingress
  ✅ Manage config with ConfigMaps and Secrets
  ✅ Persist data with PVCs, StorageClasses, dynamic provisioning
  ✅ Organise clusters with Namespaces, ResourceQuotas, LimitRanges
  ✅ Design health probes that prevent rolling update downtime

KUBERNETES ADVANCED (Days 22–25):
  ✅ Run stateful workloads with StatefulSets
  ✅ Implement one-off and scheduled tasks with Jobs and CronJobs
  ✅ Deploy cluster-wide agents with DaemonSets
  ✅ Package applications as Helm Charts
  ✅ Implement zero-trust networking with NetworkPolicies

PRODUCTION OPERATIONS (Days 26–30):
  ✅ Set up Prometheus + Grafana monitoring with custom alerts
  ✅ Build CI/CD pipelines with GitHub Actions and automatic rollback
  ✅ Deploy on AWS EKS with IRSA, ALB Controller, EBS CSI
  ✅ Implement RBAC, Pod Security Standards, secrets hardening
  ✅ Architect and deploy a complete production system
```

**What to do next:**
1. **Deploy this capstone** to a real EKS cluster and put it on GitHub
2. **Get CKA certified** — Certified Kubernetes Administrator (90% of what you need is in this course)
3. **Get CKAD certified** — Certified Kubernetes Application Developer (even more directly aligned)
4. **Add to your resume** — list EKS, Helm, Prometheus, GitHub Actions, NetworkPolicies as skills
5. **Update your LinkedIn** — "Kubernetes | Docker | AWS EKS | Helm | CI/CD"
6. **Reference this in interviews** — "I deployed PulseBloom on EKS with StatefulSets, IRSA, ALB ingress, zero-trust networking, and a full GitHub Actions CI/CD pipeline"

**You now know more Kubernetes than 90% of developers. Go build.**
