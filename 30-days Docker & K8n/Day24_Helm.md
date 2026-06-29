# Day 24: Helm — The Kubernetes Package Manager

**Goal for today:** Understand what Helm is, why it exists, how Charts work, how templating with values.yaml makes deployments configurable and reusable, how to use public charts from artifact hub, and how to write your own chart from scratch. By the end, you should be able to package any Kubernetes application as a Helm chart, deploy it with a single command, and manage upgrades and rollbacks like a production team.

**Format:** WHAT → WHY → HOW for every concept.

---

## 0. Quick Recap (open your teaching session with this)

- Days 11–23: Built and deployed full Kubernetes applications using raw YAML files
- Day 21: Capstone — managed 15+ YAML files with a custom deploy.sh script

**Today's connecting thought:**
> In Day 21 you managed 15+ YAML files to deploy one application. A real company has dozens of services, each with its own set of YAML files, each needing different configurations for dev, staging, and production. Changing the image tag requires editing the Deployment. Changing the port requires updating both Deployment and Service. There's no versioning — you can't "rollback" to the YAML you had last week. Helm solves all of this by treating your entire Kubernetes application as a versioned, parameterised package.

---

## 1. The Problem — Raw YAML Doesn't Scale

```
Problems with raw YAML files at scale:

1. DUPLICATION: 10 microservices × 5 YAML files each = 50 nearly-identical files
   Change image registry? Edit 10 Deployments. Miss one? Broken environment.

2. ENVIRONMENT CONFIG: dev has 1 replica, staging has 2, prod has 5
   Keep 3 copies of each file? Use kustomize? sed substitution? All messy.

3. NO VERSIONING: You deployed something last Tuesday.
   Can you rollback to exactly that state? With raw YAML, no.

4. DEPENDENCY MANAGEMENT: Your app needs PostgreSQL and Redis.
   You hand-copy someone else's YAML? What version? How do you update it?

5. SHARING: You built a great NGINX configuration.
   How do you share it with another team? Email them the YAML files?

Helm solves all five problems with one tool.
```

---

## 2. What is Helm?

### WHAT
**Helm** is the package manager for Kubernetes. It introduces three core concepts:

- **Chart**: A package of pre-configured Kubernetes resources (like an npm package or apt package). Contains templates + default values.
- **Release**: A deployed instance of a Chart in a cluster. You can deploy the same Chart multiple times with different configurations — each deployment is a separate Release.
- **Repository**: A collection of Charts stored and shared online (like npm registry or Docker Hub for charts).

### WHY
Helm gives you:
- **Parameterisation**: One set of templates, infinite configurations via values
- **Versioning**: Every deploy is a numbered release — rollback to any previous state
- **Dependency management**: Declare "my app needs PostgreSQL chart version 12.x"
- **Sharing**: Publish your chart once, anyone can install it
- **Atomic deploys**: Either everything succeeds or everything rolls back

### HOW (the big picture)
```
Chart (templates + values.yaml)
         │
         │  helm install myapp ./my-chart --values prod-values.yaml
         ▼
  Helm renders templates with values
         │
         ▼
  Kubectl-equivalent API calls to cluster
         │
         ▼
  Release "myapp" created in cluster
  (versioned, rollbackable)
```

**Teaching line:**
> "Helm is to Kubernetes what npm is to Node.js. A Chart is like a package.json — it describes what you need and how to configure it. A Release is like node_modules — the actual installed instance. Helm install is npm install. Helm upgrade is npm update."

---

## 3. Installing Helm

```bash
# macOS
brew install helm

# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify
helm version
# version.BuildInfo{Version:"v3.x.x", ...}
```

Helm 3 (current) does NOT require Tiller (a server-side component that Helm 2 needed). Helm 3 talks directly to the Kubernetes API Server using your kubeconfig.

---

## 4. Helm Chart Structure — Anatomy

```
my-chart/                          ← chart root directory (name = chart name)
├── Chart.yaml                     ← chart metadata (name, version, description)
├── values.yaml                    ← default configuration values
├── charts/                        ← dependency charts (subcharts)
├── templates/                     ← Kubernetes YAML templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl               ← reusable template helpers (NOT rendered directly)
│   ├── NOTES.txt                  ← printed to user after helm install
│   └── tests/
│       └── test-connection.yaml   ← helm test pods
└── .helmignore                    ← like .gitignore for helm package
```

---

## 5. Chart.yaml — Chart Metadata

```yaml
# Chart.yaml
apiVersion: v2              # Helm chart API version (v2 for Helm 3)
name: taskapi               # chart name — must match directory name
description: "TaskAPI — A task management REST API"
type: application           # application | library

# Chart versioning — follow SemVer
version: 1.2.0              # Chart version (packaging/template changes)
appVersion: "2.1.0"         # App version (the image tag being deployed)
                             # appVersion is informational — used in templates as default image tag

# Optional metadata
home: https://taskapp.com
sources:
  - https://github.com/ashish8824/taskapi
maintainers:
  - name: Ashish Anand
    email: ashish@taskapp.com
keywords:
  - taskapi
  - nodejs
  - rest-api

# Dependencies (other charts this chart depends on)
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled    # only install if postgresql.enabled=true
  - name: redis
    version: "17.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

---

## 6. values.yaml — Default Configuration

This is the heart of Helm's configurability. Every value here can be overridden at deploy time.

```yaml
# values.yaml — default values for the chart

# ── Replica and image settings ───────────────────────────────────────────────
replicaCount: 3

image:
  repository: taskapi
  tag: "1.0"
  pullPolicy: IfNotPresent

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

# ── Service settings ─────────────────────────────────────────────────────────
service:
  type: ClusterIP
  port: 80
  targetPort: 3000

# ── Ingress settings ──────────────────────────────────────────────────────────
ingress:
  enabled: false
  className: "nginx"
  annotations: {}
  hosts:
    - host: taskapi.local
      paths:
        - path: /
          pathType: Prefix
  tls: []

# ── Resource settings ─────────────────────────────────────────────────────────
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"

# ── Environment config ────────────────────────────────────────────────────────
config:
  nodeEnv: "production"
  logLevel: "info"
  port: "3000"

# ── Secrets (filled by operator — never committed) ────────────────────────────
secrets:
  dbPassword: ""
  jwtSecret: ""

# ── Health probe settings ─────────────────────────────────────────────────────
probes:
  liveness:
    path: /health
    initialDelaySeconds: 15
    periodSeconds: 30
  readiness:
    path: /health
    initialDelaySeconds: 5
    periodSeconds: 10

# ── Autoscaling ───────────────────────────────────────────────────────────────
autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

# ── Node scheduling ───────────────────────────────────────────────────────────
nodeSelector: {}
tolerations: []
affinity: {}

# ── PostgreSQL (dependency chart) ─────────────────────────────────────────────
postgresql:
  enabled: true
  auth:
    username: taskuser
    password: ""          # override this!
    database: taskdb
  primary:
    persistence:
      size: 10Gi

# ── Redis (dependency chart) ──────────────────────────────────────────────────
redis:
  enabled: true
  auth:
    enabled: false
  master:
    persistence:
      size: 2Gi
```

---

## 7. Templates — The Rendering Engine

Templates are Kubernetes YAML files with Go template syntax. Helm renders them by substituting values.

### Template Syntax Basics

```yaml
# {{ .Values.xxx }}    → value from values.yaml
# {{ .Release.Name }}  → the helm release name
# {{ .Chart.Name }}    → chart name from Chart.yaml
# {{ .Chart.Version }} → chart version
# {{ if ... }}         → conditional
# {{ range ... }}      → loop
# {{ include ... }}    → include a helper template
# {{ - }}              → strip whitespace
```

### templates/deployment.yaml — Full Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "taskapi.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "taskapi.labels" . | nindent 4 }}
  annotations:
    helm.sh/chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
    app.kubernetes.io/managed-by: {{ .Release.Service }}

spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}

  selector:
    matchLabels:
      {{- include "taskapi.selectorLabels" . | nindent 6 }}

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1

  template:
    metadata:
      labels:
        {{- include "taskapi.selectorLabels" . | nindent 8 }}
      annotations:
        # Force pod restart when configmap changes (checksum trick)
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}

    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}

      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}

          ports:
            - name: http
              containerPort: {{ .Values.config.port }}
              protocol: TCP

          env:
            - name: NODE_ENV
              value: {{ .Values.config.nodeEnv | quote }}
            - name: PORT
              value: {{ .Values.config.port | quote }}
            - name: LOG_LEVEL
              value: {{ .Values.config.logLevel | quote }}
            {{- if .Values.secrets.dbPassword }}
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ include "taskapi.fullname" . }}-secret
                  key: DB_PASSWORD
            {{- end }}

          livenessProbe:
            httpGet:
              path: {{ .Values.probes.liveness.path }}
              port: http
            initialDelaySeconds: {{ .Values.probes.liveness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.liveness.periodSeconds }}

          readinessProbe:
            httpGet:
              path: {{ .Values.probes.readiness.path }}
              port: http
            initialDelaySeconds: {{ .Values.probes.readiness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.readiness.periodSeconds }}

          resources:
            {{- toYaml .Values.resources | nindent 12 }}

      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}

      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}

      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### templates/_helpers.tpl — Reusable Template Helpers

```yaml
{{/*
Expand the name of the chart.
*/}}
{{- define "taskapi.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
Truncates at 63 chars (K8s label limit).
*/}}
{{- define "taskapi.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Common labels — used across all resources
*/}}
{{- define "taskapi.labels" -}}
helm.sh/chart: {{ include "taskapi.chart" . }}
{{ include "taskapi.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels — used for matchLabels and pod labels
*/}}
{{- define "taskapi.selectorLabels" -}}
app.kubernetes.io/name: {{ include "taskapi.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### templates/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "taskapi.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "taskapi.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  selector:
    {{- include "taskapi.selectorLabels" . | nindent 4 }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
      protocol: TCP
      name: http
```

### templates/ingress.yaml — Conditional Resource

```yaml
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "taskapi.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "taskapi.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if .Values.ingress.className }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  {{- if .Values.ingress.tls }}
  tls:
    {{- range .Values.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
  {{- end }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ include "taskapi.fullname" $ }}
                port:
                  number: {{ $.Values.service.port }}
          {{- end }}
    {{- end }}
{{- end }}
```

---

## 8. Environment-Specific Values Files

The power of Helm: one chart, multiple environments.

```
my-chart/
├── values.yaml               ← defaults (base config)
├── values-dev.yaml           ← development overrides
├── values-staging.yaml       ← staging overrides
└── values-prod.yaml          ← production overrides
```

### values-dev.yaml
```yaml
replicaCount: 1

image:
  tag: "latest"
  pullPolicy: Always

resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "200m"
    memory: "256Mi"

ingress:
  enabled: true
  hosts:
    - host: taskapi.local
      paths:
        - path: /
          pathType: Prefix

config:
  nodeEnv: "development"
  logLevel: "debug"

postgresql:
  primary:
    persistence:
      size: 2Gi
```

### values-prod.yaml
```yaml
replicaCount: 5

image:
  repository: 123456789.dkr.ecr.ap-south-1.amazonaws.com/taskapi
  tag: "2.1.0"
  pullPolicy: Always

resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "2"
    memory: "2Gi"

service:
  type: ClusterIP

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
  hosts:
    - host: api.taskapp.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: taskapi-tls
      hosts:
        - api.taskapp.com

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70

config:
  nodeEnv: "production"
  logLevel: "info"

postgresql:
  primary:
    persistence:
      size: 50Gi
```

---

## 9. Core Helm Commands — Complete Reference

### Installing Charts

```bash
# Install a chart from local directory
helm install <release-name> <chart-path> [flags]

helm install taskapi ./taskapi-chart
helm install taskapi ./taskapi-chart --namespace production --create-namespace
helm install taskapi ./taskapi-chart -f values-prod.yaml
helm install taskapi ./taskapi-chart -f values.yaml -f values-prod.yaml   # merge files
helm install taskapi ./taskapi-chart --set image.tag=2.1.0               # override single value
helm install taskapi ./taskapi-chart --set image.tag=2.1.0 --set replicaCount=5

# Dry run — see what WOULD be deployed without actually deploying
helm install taskapi ./taskapi-chart --dry-run --debug

# Install from a repository
helm install my-postgres bitnami/postgresql --namespace production
```

### Upgrading Releases

```bash
# Upgrade an existing release (same as install but updates existing)
helm upgrade taskapi ./taskapi-chart -f values-prod.yaml -n production

# Upgrade and install if not exists (atomic upsert)
helm upgrade --install taskapi ./taskapi-chart -f values-prod.yaml -n production

# Upgrade with new image tag only
helm upgrade taskapi ./taskapi-chart -n production --reuse-values \
  --set image.tag=2.2.0

# Atomic upgrade: rollback automatically if upgrade fails
helm upgrade taskapi ./taskapi-chart -f values-prod.yaml \
  --atomic \
  --timeout 5m \
  -n production
```

### Rollback

```bash
# See release history
helm history taskapi -n production
# REVISION   UPDATED                    STATUS     CHART           APP VERSION
# 1          Mon Jan 13 10:00:00 2025   superseded taskapi-1.0.0   1.0
# 2          Mon Jan 13 11:00:00 2025   superseded taskapi-1.0.0   1.1
# 3          Mon Jan 13 12:00:00 2025   deployed   taskapi-1.0.0   2.0

# Rollback to previous release
helm rollback taskapi -n production

# Rollback to specific revision
helm rollback taskapi 2 -n production
```

### Inspecting Releases

```bash
# List all releases in namespace
helm list -n production
helm ls -n production

# List across all namespaces
helm list -A

# Get current values used in a release
helm get values taskapi -n production

# Get all values including defaults
helm get values taskapi --all -n production

# See rendered manifests for a release
helm get manifest taskapi -n production

# Status of a release
helm status taskapi -n production
```

### Uninstalling

```bash
# Uninstall a release (deletes all K8s resources)
helm uninstall taskapi -n production

# Uninstall but keep history (for rollback data)
helm uninstall taskapi -n production --keep-history
```

### Templating and Linting

```bash
# Render templates to stdout (without installing)
helm template taskapi ./taskapi-chart -f values-prod.yaml

# Render to a file (for GitOps — applying rendered YAML)
helm template taskapi ./taskapi-chart -f values-prod.yaml > rendered.yaml
kubectl apply -f rendered.yaml

# Lint a chart (check for errors)
helm lint ./taskapi-chart
helm lint ./taskapi-chart -f values-prod.yaml

# Package a chart into a .tgz archive
helm package ./taskapi-chart
# Creates: taskapi-1.0.0.tgz
```

---

## 10. Helm Repositories — Finding and Using Public Charts

```bash
# Add popular chart repositories
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add cert-manager https://charts.jetstack.io
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts

# Update repo index (like apt-get update)
helm repo update

# Search for charts
helm search repo postgresql
helm search repo bitnami/postgresql
helm search hub postgres               # search artifact hub (like npm search)

# Show chart info before installing
helm show chart bitnami/postgresql
helm show values bitnami/postgresql    # see all configurable values
helm show readme bitnami/postgresql

# List installed repos
helm repo list

# Remove a repo
helm repo remove bitnami
```

### Installing Real Infrastructure with Helm

```bash
# Install PostgreSQL (production-grade, HA capable)
helm install postgres bitnami/postgresql \
  --namespace production \
  --create-namespace \
  --set auth.username=taskuser \
  --set auth.password=supersecret \
  --set auth.database=taskdb \
  --set primary.persistence.size=20Gi

# Install Redis
helm install redis bitnami/redis \
  --namespace production \
  --set auth.enabled=false \
  --set master.persistence.size=5Gi

# Install Ingress Nginx (production-ready)
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer

# Install cert-manager
helm install cert-manager cert-manager/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

# Install Prometheus + Grafana monitoring stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

---

## 11. Creating Your Own Chart from Scratch

```bash
# Scaffold a chart (creates directory with all files)
helm create taskapi

# Generated structure:
# taskapi/
# ├── Chart.yaml
# ├── values.yaml        ← defaults (edit this heavily)
# ├── charts/
# └── templates/
#     ├── deployment.yaml    ← edit to match your app
#     ├── service.yaml
#     ├── ingress.yaml
#     ├── hpa.yaml          ← HorizontalPodAutoscaler (Day 25 topic)
#     ├── serviceaccount.yaml
#     ├── _helpers.tpl
#     ├── NOTES.txt
#     └── tests/
#         └── test-connection.yaml
```

```bash
# Development workflow:
# 1. Edit Chart.yaml, values.yaml, templates/*

# 2. Lint to catch errors
helm lint ./taskapi

# 3. Render to check output
helm template my-release ./taskapi -f values-dev.yaml

# 4. Install (dev)
helm install taskapi-dev ./taskapi \
  --namespace development \
  --create-namespace \
  -f values-dev.yaml

# 5. Test it
helm test taskapi-dev -n development

# 6. Upgrade after changes
helm upgrade taskapi-dev ./taskapi \
  -n development \
  -f values-dev.yaml

# 7. When ready for prod
helm upgrade --install taskapi-prod ./taskapi \
  --namespace production \
  --create-namespace \
  -f values-prod.yaml \
  --atomic
```

---

## 12. NOTES.txt — Post-Install Instructions

```
{{/* templates/NOTES.txt */}}
Thank you for installing {{ .Chart.Name }} v{{ .Chart.AppVersion }}!

Release name: {{ .Release.Name }}
Namespace:    {{ .Release.Namespace }}

{{- if .Values.ingress.enabled }}
Your application is available at:
  {{- range .Values.ingress.hosts }}
  http{{ if $.Values.ingress.tls }}s{{ end }}://{{ .host }}
  {{- end }}
{{- else }}
To access your application:

  kubectl port-forward service/{{ include "taskapi.fullname" . }} 8080:{{ .Values.service.port }} -n {{ .Release.Namespace }}
  Open: http://localhost:8080

{{- end }}

To upgrade:
  helm upgrade {{ .Release.Name }} ./taskapi-chart -n {{ .Release.Namespace }} -f values-prod.yaml

To rollback:
  helm rollback {{ .Release.Name }} -n {{ .Release.Namespace }}

Chart version: {{ .Chart.Version }}
App version:   {{ .Chart.AppVersion }}
```

---

## 13. Helm Dependencies — Using Subcharts

```yaml
# Chart.yaml — declare dependencies
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

```bash
# Download dependencies into charts/ directory
helm dependency update ./taskapi-chart

# Install with all dependencies
helm install taskapi ./taskapi-chart \
  -f values-prod.yaml \
  --set postgresql.auth.password=supersecret \
  --set redis.auth.enabled=false
```

---

## 14. Helm Secrets — Managing Sensitive Values

Never put real secrets in values.yaml (it gets committed to Git).

### Pattern 1: --set at deploy time (CI/CD)
```bash
helm upgrade --install taskapi ./taskapi-chart \
  -f values-prod.yaml \
  --set secrets.dbPassword=$DB_PASSWORD \
  --set secrets.jwtSecret=$JWT_SECRET
```

### Pattern 2: Separate secrets file (gitignored)
```bash
# secrets-prod.yaml (in .gitignore — NEVER commit)
secrets:
  dbPassword: "supersecretpassword"
  jwtSecret: "my-very-long-jwt-secret"

# Deploy with secrets file
helm upgrade --install taskapi ./taskapi-chart \
  -f values-prod.yaml \
  -f secrets-prod.yaml
```

### Pattern 3: helm-secrets plugin (encrypts secrets for Git)
```bash
# Install plugin
helm plugin install https://github.com/jkroepke/helm-secrets

# Encrypt secrets file with SOPS (uses AWS KMS, GCP KMS, or age)
helm secrets encrypt secrets-prod.yaml > secrets-prod.yaml.enc

# Deploy with encrypted secrets (decrypted in memory only)
helm secrets upgrade --install taskapi ./taskapi-chart \
  -f values-prod.yaml \
  -f secrets-prod.yaml.enc
```

---

## 15. Helm in CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy with Helm

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-south-1

      - name: Login to ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/taskapi:$IMAGE_TAG .
          docker push $ECR_REGISTRY/taskapi:$IMAGE_TAG

      - name: Setup Helm
        uses: azure/setup-helm@v3
        with:
          version: '3.13.0'

      - name: Configure kubectl
        uses: aws-actions/amazon-eks-update-kubeconfig@v1
        with:
          cluster-name: my-eks-cluster
          region: ap-south-1

      - name: Run DB migration
        run: |
          helm upgrade --install db-migration ./charts/migration \
            --namespace production \
            --set image.tag=${{ github.sha }} \
            --set secrets.dbPassword=${{ secrets.DB_PASSWORD }} \
            --wait --timeout 5m

      - name: Deploy application
        run: |
          helm upgrade --install taskapi ./charts/taskapi \
            --namespace production \
            --create-namespace \
            -f charts/taskapi/values-prod.yaml \
            --set image.tag=${{ github.sha }} \
            --set secrets.dbPassword=${{ secrets.DB_PASSWORD }} \
            --set secrets.jwtSecret=${{ secrets.JWT_SECRET }} \
            --atomic \
            --timeout 10m \
            --cleanup-on-fail
```

---

## 16. Debugging Helm Issues

```bash
# ── See what Helm would render before installing ──────────────────────────────
helm template taskapi ./taskapi-chart -f values-prod.yaml | less

# ── Debug install (verbose rendering + error details) ─────────────────────────
helm install taskapi ./taskapi-chart --dry-run --debug -f values-prod.yaml

# ── Check release status ───────────────────────────────────────────────────────
helm status taskapi -n production

# ── See what K8s resources a release owns ─────────────────────────────────────
helm get manifest taskapi -n production

# ── See what values are actually in use ───────────────────────────────────────
helm get values taskapi -n production           # user-supplied values only
helm get values taskapi -n production --all     # all values including defaults

# ── Troubleshoot a failed upgrade ─────────────────────────────────────────────
helm history taskapi -n production              # find last good revision
helm rollback taskapi <good-revision> -n production

# ── Release stuck in "pending-upgrade" ────────────────────────────────────────
# Happens when a previous upgrade was interrupted
helm rollback taskapi 0 -n production  # rollback to last deployed state
# OR
kubectl delete secret -n production -l owner=helm,name=taskapi,status=pending-upgrade
```

---

## 17. Hands-On Practice Tasks

```bash
# 1. Install Helm
helm version

# 2. Add Bitnami repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# 3. Explore a chart before installing
helm show values bitnami/postgresql | head -50

# 4. Install PostgreSQL using Helm (one command!)
helm install postgres bitnami/postgresql \
  --namespace helm-demo \
  --create-namespace \
  --set auth.postgresPassword=helmpassword \
  --set auth.database=demodb \
  --set primary.persistence.size=1Gi

# 5. Check what got deployed
helm list -n helm-demo
kubectl get all -n helm-demo

# 6. Scaffold your own chart
helm create my-taskapi
ls my-taskapi/

# 7. Edit values.yaml — change replicaCount to 2, set image.repository and image.tag

# 8. Lint the chart
helm lint ./my-taskapi

# 9. Render templates (see output)
helm template my-release ./my-taskapi

# 10. Install with dry-run first
helm install taskapi ./my-taskapi --dry-run --debug -n helm-demo

# 11. Install for real
helm install taskapi ./my-taskapi -n helm-demo

# 12. Upgrade with new value
helm upgrade taskapi ./my-taskapi -n helm-demo --set replicaCount=3

# 13. Check history and rollback
helm history taskapi -n helm-demo
helm rollback taskapi 1 -n helm-demo

# 14. Clean up
helm uninstall taskapi -n helm-demo
helm uninstall postgres -n helm-demo
kubectl delete namespace helm-demo
```

---

## 18. Quiz — Test Yourself / Test Others

1. What are the three core Helm concepts — Chart, Release, and Repository?
2. What is the difference between `chart.version` and `chart.appVersion` in Chart.yaml?
3. What is `values.yaml` and how does it enable environment-specific deployments?
4. What does `helm upgrade --install` do differently from plain `helm install`?
5. What does `--atomic` do in `helm upgrade`?
6. How do you override a specific value from `values.yaml` without editing the file?
7. What does `helm template` do and when would you use it?
8. What is `_helpers.tpl` used for in a Helm chart?
9. How do you rollback to revision 2 of a Helm release named `taskapi` in the `production` namespace?
10. Why should sensitive values (DB passwords, JWT secrets) NEVER be in `values.yaml`?

<details>
<summary>Answers</summary>

1. Chart: a package of pre-configured Kubernetes resource templates plus default values — like an npm package or apt package. Release: a deployed instance of a Chart in a Kubernetes cluster with a specific name and configuration — you can deploy the same Chart multiple times as separate Releases. Repository: a collection of Charts stored and distributed online — like Docker Hub for container images or npm registry for packages.

2. `chart.version` is the version of the Chart itself — it increments when you change templates, values, or chart packaging. `chart.appVersion` is the version of the application being deployed (typically the Docker image tag) — informational, used as the default image tag in templates. You can update the app version without changing the chart version or vice versa.

3. `values.yaml` contains default configuration values for all template variables. By overriding these values with environment-specific files (`-f values-prod.yaml`) or `--set` flags, you can deploy the same Chart templates to dev, staging, and production with different replicas, resource limits, image tags, and configuration — without duplicating YAML files.

4. `helm upgrade --install` is an idempotent "upsert" — if the release doesn't exist, it installs it; if it already exists, it upgrades it. Plain `helm install` fails if the release already exists. `upgrade --install` is the standard command for CI/CD pipelines where you don't know whether it's a first deployment or an update.

5. `--atomic` makes the upgrade all-or-nothing: if any part of the upgrade fails (Pods don't become Ready within the timeout), Helm automatically rolls back to the previous successful release. Without `--atomic`, a failed upgrade leaves the cluster in the partially-upgraded broken state, requiring manual intervention.

6. Use `--set key=value` flag: `helm upgrade taskapi ./chart --set image.tag=2.0.0`. For nested values: `--set image.repository=myrepo`. For multiple overrides: `--set key1=val1 --set key2=val2`. For values with special characters: `--set-string key=value`. These override values.yaml without editing it.

7. `helm template` renders all chart templates with the given values and prints the resulting YAML to stdout — without connecting to or modifying the Kubernetes cluster. Use it for: debugging template rendering, reviewing what will be deployed, generating static YAML for GitOps workflows (pipe to a file and `kubectl apply`), or validating template syntax.

8. `_helpers.tpl` (files starting with `_` are not rendered as Kubernetes resources) contains reusable Go template definitions that other templates can include with `{{ include "name" . }}`. Common helpers: `fullname` (generates a consistent resource name), `labels` (generates standard labels for all resources), `selectorLabels` (for matchLabels and pod labels). This avoids repeating the same logic in every template file.

9. `helm rollback taskapi 2 --namespace production`. This rolls the release back to revision 2 — Helm re-applies the Chart templates and values that were used at that revision. Run `helm history taskapi -n production` first to see available revisions.

10. `values.yaml` is typically committed to Git for version control. If secrets are in `values.yaml`, they're exposed in the Git history — accessible to anyone with repo access, CI/CD logs, and anyone who runs `helm get values`. Secrets should be passed via `--set` from CI/CD environment variables, via a separate gitignored secrets file, or via the `helm-secrets` plugin with encryption.

</details>

---

## 19. Summary

After today you can explain and implement:
- **Why Helm exists** — duplication, env config, versioning, dependency management, sharing
- **Three core concepts** — Chart (package), Release (deployed instance), Repository (store)
- **Chart structure** — Chart.yaml, values.yaml, templates/, _helpers.tpl, NOTES.txt
- **Template syntax** — `{{ .Values.x }}`, `{{ .Release.Name }}`, conditionals, loops, `toYaml`, `nindent`
- **values.yaml** — defaults overridden by environment-specific files or `--set`
- **Complete command set** — `install`, `upgrade`, `upgrade --install`, `rollback`, `history`, `list`, `template`, `lint`, `package`
- **Repository management** — `repo add`, `repo update`, `search`, `show values`
- **Public charts** — installing PostgreSQL, Redis, Ingress Nginx, cert-manager with one command
- **Creating charts** — `helm create`, edit templates, lint, test workflow
- **Secret management** — never in values.yaml, `--set` from CI/CD, helm-secrets plugin
- **CI/CD integration** — full GitHub Actions pipeline with `upgrade --install --atomic`
- **Debugging** — `--dry-run --debug`, `helm get manifest`, `helm history`, fixing pending-upgrade

**Coming up on Day 25:** Kubernetes Networking Deep Dive — CNI plugins, NetworkPolicies (firewall rules between Pods), and how to implement zero-trust networking in your cluster where services can only talk to what they're explicitly allowed to reach.
