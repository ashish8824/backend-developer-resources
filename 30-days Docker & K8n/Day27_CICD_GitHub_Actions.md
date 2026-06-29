# Day 27: CI/CD — GitHub Actions Deploying to Kubernetes

**Goal for today:** Build a complete, production-grade CI/CD pipeline using GitHub Actions that automatically builds Docker images, scans for vulnerabilities, pushes to AWS ECR, runs database migrations, and deploys to Kubernetes via Helm — with automatic rollback on failure. By the end, you should be able to implement a zero-touch deployment pipeline for any Kubernetes application.

**Format:** WHAT → WHY → HOW for every concept.

---

## 0. Quick Recap (open your teaching session with this)

- Day 9: Registries — Docker Hub, AWS ECR, image tagging with Git SHA
- Day 24: Helm — versioned, parameterised deployments with rollback

**Today's connecting thought:**
> In Days 9 and 24 you learned to push images and deploy with Helm manually. Today we automate that entire process so it happens on every Git push — no human involvement between code commit and production deployment. This is the pipeline that modern engineering teams run hundreds of times per day.

---

## 1. What is CI/CD?

### WHAT
**CI (Continuous Integration):** Every code change is automatically built, tested, and validated.
**CD (Continuous Delivery):** Every validated build is automatically deployed to one or more environments.

```
Developer pushes code
        │
        ▼
  GitHub Actions triggered
        │
        ├── CI Phase
        │   ├── Run unit tests
        │   ├── Run integration tests
        │   ├── Build Docker image
        │   └── Scan image for vulnerabilities
        │
        └── CD Phase
            ├── Push image to ECR
            ├── Run DB migration Job
            ├── Deploy to Kubernetes (Helm upgrade)
            └── Verify deployment health
                    │
                    ├── Success → done, notify team
                    └── Failure → auto-rollback, notify team
```

### WHY CI/CD Matters
```
Without CI/CD (manual deployments):
  Developer finishes feature
  → Manually runs tests (maybe)
  → Manually builds Docker image
  → Manually tags and pushes to ECR
  → SSHs into server or runs kubectl manually
  → Checks if it worked
  → "Works on my machine" problems
  → Error-prone, slow, undocumented

With CI/CD:
  Developer pushes to main
  → Everything automated, reproducible
  → Every deployment identical
  → Full audit trail in GitHub
  → Rollback is one click
  → Team is unblocked — no "deploy person" bottleneck
```

**Teaching line:**
> "CI/CD is like a factory assembly line for software. A car factory doesn't have an engineer manually assembling each car — they define the process once and the line runs it the same way every time, thousands of times. CI/CD is your software factory."

---

## 2. GitHub Actions — Core Concepts

### WHAT
GitHub Actions is a CI/CD platform built into GitHub. Workflows are defined as YAML files in `.github/workflows/`.

### Key Terminology

```
Workflow    → a YAML file defining an automated process
              triggered by events (push, PR, schedule, manual)

Job         → a group of steps that run on the SAME runner (machine)
              jobs run in PARALLEL by default unless you declare dependencies

Step        → a single task in a job
              either runs a shell command (run:) or calls an Action (uses:)

Action      → a reusable step from the GitHub Marketplace
              e.g., actions/checkout, aws-actions/amazon-ecr-login

Runner      → the machine where the job executes
              ubuntu-latest = GitHub-hosted Ubuntu VM
              self-hosted = your own machine

Event       → what triggers the workflow
              push, pull_request, workflow_dispatch, schedule, etc.

Secret      → encrypted variable stored in GitHub repo settings
              accessed as ${{ secrets.MY_SECRET }}

Context     → built-in variables like github.sha, github.ref
```

---

## 3. The Complete Pipeline — File by File

### Project Structure

```
taskapi/
├── .github/
│   └── workflows/
│       ├── ci.yml              ← CI: test + build + scan (on every PR)
│       ├── deploy-staging.yml  ← CD: deploy to staging (on push to develop)
│       └── deploy-prod.yml     ← CD: deploy to production (on push to main)
├── charts/
│   └── taskapi/               ← Helm chart
├── src/
├── Dockerfile
└── ...
```

---

## 4. CI Workflow — Test, Build, Scan

### `.github/workflows/ci.yml`

```yaml
name: CI — Test, Build & Scan

# ── Triggers ──────────────────────────────────────────────────────────────────
on:
  pull_request:
    branches:
      - main
      - develop
  push:
    branches:
      - main
      - develop

# ── Environment Variables ─────────────────────────────────────────────────────
env:
  AWS_REGION: ap-south-1
  ECR_REPOSITORY: taskapi
  NODE_VERSION: "18"

jobs:
  # ── Job 1: Test ──────────────────────────────────────────────────────────────
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    
    services:
      # Spin up a real PostgreSQL for integration tests
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js ${{ env.NODE_VERSION }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'               # cache node_modules between runs

      - name: Install dependencies
        run: npm ci

      - name: Run linting
        run: npm run lint

      - name: Run unit tests
        run: npm run test:unit

      - name: Run integration tests
        env:
          DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379
          NODE_ENV: test
          JWT_SECRET: test-jwt-secret
        run: npm run test:integration

      - name: Upload test coverage
        uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: ./coverage/lcov.info

  # ── Job 2: Build & Scan ──────────────────────────────────────────────────────
  build-and-scan:
    name: Build & Scan Image
    runs-on: ubuntu-latest
    needs: test               # only run if test job passes
    
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}
      image-uri: ${{ steps.build-push.outputs.image-uri }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      # Set up Docker Buildx for advanced builds
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      # Configure AWS credentials
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      # Login to ECR
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      # Generate image tags
      - name: Generate image metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}
          tags: |
            type=sha,prefix=,format=short           # abc1234
            type=ref,event=branch                   # main, develop
            type=semver,pattern={{version}}          # v1.2.3 (from git tag)

      # Build and push image
      - name: Build and push Docker image
        id: build-push
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          platforms: linux/amd64              # always build for AMD64
          cache-from: type=gha               # GitHub Actions cache
          cache-to: type=gha,mode=max
          build-args: |
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
            GIT_SHA=${{ github.sha }}

      # Scan image for vulnerabilities
      - name: Scan image with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'              # fail pipeline on CRITICAL vulnerabilities

      # Upload scan results to GitHub Security tab
      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v2
        if: always()                  # upload even if scan failed
        with:
          sarif_file: 'trivy-results.sarif'
```

---

## 5. Staging Deployment Workflow

### `.github/workflows/deploy-staging.yml`

```yaml
name: Deploy to Staging

on:
  push:
    branches:
      - develop                  # deploy to staging on every push to develop

env:
  AWS_REGION: ap-south-1
  ECR_REPOSITORY: taskapi
  EKS_CLUSTER: taskapi-staging
  HELM_RELEASE: taskapi
  K8S_NAMESPACE: staging

jobs:
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    environment: staging          # GitHub environment (requires approval if configured)

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      # Configure AWS
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      # Login to ECR
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      # Get image URI
      - name: Set image tag
        id: image
        run: |
          SHORT_SHA=$(echo ${{ github.sha }} | cut -c1-7)
          echo "tag=${SHORT_SHA}" >> $GITHUB_OUTPUT
          echo "uri=${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${SHORT_SHA}" >> $GITHUB_OUTPUT

      # Configure kubectl for EKS
      - name: Configure kubectl for EKS
        run: |
          aws eks update-kubeconfig \
            --name ${{ env.EKS_CLUSTER }} \
            --region ${{ env.AWS_REGION }}

      # Install Helm
      - name: Setup Helm
        uses: azure/setup-helm@v3
        with:
          version: '3.13.0'

      # Create namespace if it doesn't exist
      - name: Ensure namespace exists
        run: |
          kubectl create namespace ${{ env.K8S_NAMESPACE }} \
            --dry-run=client -o yaml | kubectl apply -f -

      # Run DB migration BEFORE deploying new app version
      - name: Run database migration
        run: |
          # Delete old migration job if exists
          kubectl delete job db-migration-${{ steps.image.outputs.tag }} \
            -n ${{ env.K8S_NAMESPACE }} --ignore-not-found

          # Apply migration job
          helm upgrade --install db-migration-${{ steps.image.outputs.tag }} \
            ./charts/migration \
            --namespace ${{ env.K8S_NAMESPACE }} \
            --set image.repository=${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }} \
            --set image.tag=${{ steps.image.outputs.tag }} \
            --set secrets.databaseUrl=${{ secrets.STAGING_DATABASE_URL }} \
            --wait --timeout 5m

          # Wait for migration to complete
          kubectl wait job/db-migration-${{ steps.image.outputs.tag }}-migration \
            --for=condition=complete \
            --timeout=300s \
            -n ${{ env.K8S_NAMESPACE }}

      # Deploy with Helm
      - name: Deploy to Staging
        run: |
          helm upgrade --install ${{ env.HELM_RELEASE }} ./charts/taskapi \
            --namespace ${{ env.K8S_NAMESPACE }} \
            --create-namespace \
            -f charts/taskapi/values-staging.yaml \
            --set image.repository=${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }} \
            --set image.tag=${{ steps.image.outputs.tag }} \
            --set secrets.dbPassword=${{ secrets.STAGING_DB_PASSWORD }} \
            --set secrets.jwtSecret=${{ secrets.STAGING_JWT_SECRET }} \
            --atomic \
            --timeout 10m \
            --cleanup-on-fail

      # Verify deployment
      - name: Verify deployment
        run: |
          kubectl rollout status deployment/${{ env.HELM_RELEASE }} \
            -n ${{ env.K8S_NAMESPACE }} \
            --timeout=5m

          # Run smoke test
          STAGING_URL=$(kubectl get ingress ${{ env.HELM_RELEASE }} \
            -n ${{ env.K8S_NAMESPACE }} \
            -o jsonpath='{.spec.rules[0].host}')

          sleep 10    # wait for LB to route traffic

          response=$(curl -s -o /dev/null -w "%{http_code}" \
            http://${STAGING_URL}/health --max-time 10)

          if [ "$response" != "200" ]; then
            echo "Smoke test FAILED: /health returned $response"
            exit 1
          fi
          echo "Smoke test PASSED: /health returned 200"

      # Notify team
      - name: Notify Slack on success
        if: success()
        uses: slackapi/slack-github-action@v1.25.0
        with:
          channel-id: 'deployments'
          slack-message: |
            ✅ *Staging deployed successfully*
            Commit: `${{ github.sha }}`
            Branch: `${{ github.ref_name }}`
            Author: ${{ github.actor }}
            Image: `${{ steps.image.outputs.tag }}`
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}

      - name: Notify Slack on failure
        if: failure()
        uses: slackapi/slack-github-action@v1.25.0
        with:
          channel-id: 'deployments'
          slack-message: |
            ❌ *Staging deployment FAILED*
            Commit: `${{ github.sha }}`
            Author: ${{ github.actor }}
            Check: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

---

## 6. Production Deployment Workflow

### `.github/workflows/deploy-prod.yml`

```yaml
name: Deploy to Production

on:
  push:
    branches:
      - main                      # deploy to prod on push to main

  # Also allow manual trigger with custom image tag
  workflow_dispatch:
    inputs:
      image_tag:
        description: 'Image tag to deploy (leave empty to use latest SHA)'
        required: false
        type: string
      skip_migration:
        description: 'Skip database migration'
        required: false
        type: boolean
        default: false

env:
  AWS_REGION: ap-south-1
  ECR_REPOSITORY: taskapi
  EKS_CLUSTER: taskapi-production
  HELM_RELEASE: taskapi
  K8S_NAMESPACE: production

jobs:
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    environment: production        # GitHub environment — requires manual approval

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      # Determine image tag (manual input or auto SHA)
      - name: Set image tag
        id: image
        run: |
          if [ -n "${{ github.event.inputs.image_tag }}" ]; then
            TAG="${{ github.event.inputs.image_tag }}"
          else
            TAG=$(echo ${{ github.sha }} | cut -c1-7)
          fi
          echo "tag=${TAG}" >> $GITHUB_OUTPUT
          echo "Using image tag: ${TAG}"

      - name: Configure kubectl for EKS
        run: |
          aws eks update-kubeconfig \
            --name ${{ env.EKS_CLUSTER }} \
            --region ${{ env.AWS_REGION }}

      - name: Setup Helm
        uses: azure/setup-helm@v3
        with:
          version: '3.13.0'

      # Save current helm revision for rollback if needed
      - name: Get current helm revision
        id: current-revision
        run: |
          REVISION=$(helm history ${{ env.HELM_RELEASE }} \
            -n ${{ env.K8S_NAMESPACE }} \
            --max 1 -o json 2>/dev/null \
            | jq -r '.[0].revision // "0"')
          echo "revision=${REVISION}" >> $GITHUB_OUTPUT
          echo "Current revision: ${REVISION}"

      # Database migration
      - name: Run database migration
        if: ${{ !inputs.skip_migration }}
        run: |
          JOB_NAME="db-migration-${{ steps.image.outputs.tag }}"

          kubectl delete job ${JOB_NAME} \
            -n ${{ env.K8S_NAMESPACE }} --ignore-not-found

          helm upgrade --install ${JOB_NAME} ./charts/migration \
            --namespace ${{ env.K8S_NAMESPACE }} \
            --set image.repository=${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }} \
            --set image.tag=${{ steps.image.outputs.tag }} \
            --set secrets.databaseUrl=${{ secrets.PROD_DATABASE_URL }} \
            --wait --timeout 5m

          kubectl wait job/${JOB_NAME}-migration \
            --for=condition=complete \
            --timeout=300s \
            -n ${{ env.K8S_NAMESPACE }}

          if [ $? -ne 0 ]; then
            echo "Migration FAILED! Aborting deployment."
            kubectl logs -l job-name=${JOB_NAME}-migration -n ${{ env.K8S_NAMESPACE }}
            exit 1
          fi

      # Deploy to production
      - name: Deploy to Production
        id: deploy
        run: |
          helm upgrade --install ${{ env.HELM_RELEASE }} ./charts/taskapi \
            --namespace ${{ env.K8S_NAMESPACE }} \
            -f charts/taskapi/values-prod.yaml \
            --set image.repository=${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }} \
            --set image.tag=${{ steps.image.outputs.tag }} \
            --set secrets.dbPassword=${{ secrets.PROD_DB_PASSWORD }} \
            --set secrets.jwtSecret=${{ secrets.PROD_JWT_SECRET }} \
            --atomic \
            --timeout 15m \
            --cleanup-on-fail \
            --history-max 10

      # Post-deployment smoke test
      - name: Run smoke tests
        run: |
          echo "Waiting for deployment to stabilise..."
          sleep 20

          PROD_URL="api.taskapp.com"

          # Health check
          health_response=$(curl -s -o /dev/null -w "%{http_code}" \
            https://${PROD_URL}/health --max-time 15)

          if [ "$health_response" != "200" ]; then
            echo "Health check FAILED: returned ${health_response}"
            exit 1
          fi
          echo "Health check PASSED"

          # API smoke test — create and retrieve a task
          task_response=$(curl -s -X POST https://${PROD_URL}/api/tasks \
            -H "Content-Type: application/json" \
            -d '{"title":"smoke-test-${{ github.run_id }}"}' \
            --max-time 10)

          task_id=$(echo $task_response | jq -r '.id')
          if [ -z "$task_id" ] || [ "$task_id" == "null" ]; then
            echo "API smoke test FAILED: could not create task"
            echo "Response: $task_response"
            exit 1
          fi

          # Clean up smoke test data
          curl -s -X DELETE https://${PROD_URL}/api/tasks/${task_id} --max-time 10
          echo "API smoke test PASSED (task ${task_id})"

      # Automatic rollback on failure
      - name: Rollback on failure
        if: failure() && steps.deploy.outcome == 'failure'
        run: |
          echo "Deployment failed! Rolling back to revision ${{ steps.current-revision.outputs.revision }}..."

          helm rollback ${{ env.HELM_RELEASE }} \
            ${{ steps.current-revision.outputs.revision }} \
            -n ${{ env.K8S_NAMESPACE }} \
            --wait --timeout 10m

          echo "Rollback complete"

      # Tag the release in Git
      - name: Create GitHub Release
        if: success()
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: "deploy-prod-${{ steps.image.outputs.tag }}"
          release_name: "Production Deploy ${{ steps.image.outputs.tag }}"
          body: |
            ## Production Deployment
            - **Image:** `${{ steps.image.outputs.tag }}`
            - **Deployed by:** ${{ github.actor }}
            - **Commit:** ${{ github.sha }}
          draft: false
          prerelease: false

      # Success notification
      - name: Notify success
        if: success()
        uses: slackapi/slack-github-action@v1.25.0
        with:
          channel-id: 'deployments'
          slack-message: |
            🚀 *Production deployed successfully*
            Image: `${{ steps.image.outputs.tag }}`
            By: ${{ github.actor }}
            Commit: ${{ github.sha }}
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}

      # Failure notification
      - name: Notify failure
        if: failure()
        uses: slackapi/slack-github-action@v1.25.0
        with:
          channel-id: 'deployments-critical'
          slack-message: |
            🔥 *PRODUCTION DEPLOYMENT FAILED — ROLLBACK EXECUTED*
            Image: `${{ steps.image.outputs.tag }}`
            By: ${{ github.actor }}
            Run: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
            @oncall please investigate immediately
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

---

## 7. Reusable Workflows — DRY Principle

When staging and production share the same deployment logic, extract it into a reusable workflow:

```yaml
# .github/workflows/_deploy.yml (reusable)
name: Deploy (Reusable)

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      image_tag:
        required: true
        type: string
      helm_values_file:
        required: true
        type: string
      eks_cluster:
        required: true
        type: string
      namespace:
        required: true
        type: string
    secrets:
      AWS_ACCESS_KEY_ID:
        required: true
      AWS_SECRET_ACCESS_KEY:
        required: true
      DB_PASSWORD:
        required: true
      JWT_SECRET:
        required: true

jobs:
  deploy:
    name: Deploy to ${{ inputs.environment }}
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - uses: actions/checkout@v4
      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-south-1
      # ... full deploy steps ...
```

```yaml
# .github/workflows/deploy-staging.yml (calls reusable)
jobs:
  deploy:
    uses: ./.github/workflows/_deploy.yml
    with:
      environment: staging
      image_tag: ${{ needs.build.outputs.image-tag }}
      helm_values_file: values-staging.yaml
      eks_cluster: taskapi-staging
      namespace: staging
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      DB_PASSWORD: ${{ secrets.STAGING_DB_PASSWORD }}
      JWT_SECRET: ${{ secrets.STAGING_JWT_SECRET }}
```

---

## 8. GitHub Environments — Approval Gates

Configure deployment protection rules in GitHub:
```
Repository Settings → Environments → production
  ✅ Required reviewers: [senior-engineer, tech-lead]
  ✅ Wait timer: 5 minutes (force pause before prod)
  ✅ Deployment branches: main only
```

This means every production deployment requires:
1. Manual approval from a designated reviewer
2. A 5-minute mandatory wait
3. Can only deploy from the `main` branch

---

## 9. GitHub Secrets — What to Configure

```
Repository → Settings → Secrets and variables → Actions

Required secrets:
  AWS_ACCESS_KEY_ID          → AWS IAM key for ECR + EKS access
  AWS_SECRET_ACCESS_KEY      → AWS IAM secret

  PROD_DB_PASSWORD           → production PostgreSQL password
  PROD_JWT_SECRET            → production JWT signing secret
  PROD_DATABASE_URL          → full PostgreSQL connection URL

  STAGING_DB_PASSWORD        → staging PostgreSQL password
  STAGING_JWT_SECRET         → staging JWT secret
  STAGING_DATABASE_URL       → staging connection URL

  SLACK_BOT_TOKEN            → for deployment notifications
  CODECOV_TOKEN              → for test coverage reporting

AWS IAM permissions required:
  ecr:GetAuthorizationToken
  ecr:BatchCheckLayerAvailability
  ecr:GetDownloadUrlForLayer
  ecr:BatchGetImage
  ecr:PutImage
  ecr:InitiateLayerUpload
  ecr:UploadLayerPart
  ecr:CompleteLayerUpload
  eks:DescribeCluster        → for aws eks update-kubeconfig
```

---

## 10. Pipeline Optimisations

### Cache Dependencies

```yaml
# Cache node_modules between runs (saves 30-60s per run)
- name: Setup Node.js
  uses: actions/setup-node@v4
  with:
    node-version: '18'
    cache: 'npm'           # automatically caches ~/.npm

# Cache Docker layers (saves 2-4 min per build)
- name: Build and push
  uses: docker/build-push-action@v5
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

### Run Jobs in Parallel

```yaml
jobs:
  test-unit:
    runs-on: ubuntu-latest
    steps: [...]

  test-integration:
    runs-on: ubuntu-latest
    steps: [...]

  lint:
    runs-on: ubuntu-latest
    steps: [...]

  # Build only after ALL test/lint jobs pass
  build:
    needs: [test-unit, test-integration, lint]
    runs-on: ubuntu-latest
    steps: [...]
```

### Skip CI on Docs Changes

```yaml
on:
  push:
    branches: [main]
    paths-ignore:
      - '**.md'
      - 'docs/**'
      - '.gitignore'
```

---

## 11. The Full GitFlow → CI/CD Branch Strategy

```
feature/* → developer pushes code
              │
              ├── No CI/CD trigger (keep cost down)
              │
              └── PR opened → CI runs (test + build + scan)
                                │
                                ├── Tests pass → PR approved → merge to develop
                                │
develop   ←─────────────────────┘
              │
              └── Automatic deploy to STAGING
                  ├── Run migration
                  ├── Deploy
                  └── Smoke test
                         │
                         ├── Pass → QA tests staging
                         │
main      ←──────────────┘  PR from develop → main (manual approval)
              │
              └── Automatic deploy to PRODUCTION
                  ├── Approval gate (reviewer must approve)
                  ├── Run migration
                  ├── Deploy --atomic
                  ├── Smoke test
                  └── Failure → auto-rollback
```

---

## 12. Useful GitHub Actions for Kubernetes

```yaml
# Official AWS actions
uses: aws-actions/configure-aws-credentials@v4    # AWS auth
uses: aws-actions/amazon-ecr-login@v2             # ECR login
uses: aws-actions/amazon-eks-update-kubeconfig@v1 # EKS kubeconfig

# Docker
uses: docker/setup-buildx-action@v3       # enable BuildKit
uses: docker/metadata-action@v5           # generate tags
uses: docker/build-push-action@v5         # build + push

# Security
uses: aquasecurity/trivy-action@master    # vulnerability scan

# Helm
uses: azure/setup-helm@v3                 # install Helm CLI

# Kubernetes
uses: azure/k8s-set-context@v3            # configure kubectl (alternative)

# Notifications
uses: slackapi/slack-github-action@v1     # Slack messages

# Utilities
uses: actions/checkout@v4                 # checkout repo
uses: actions/setup-node@v4              # Node.js setup
uses: actions/cache@v3                   # cache files between runs
uses: actions/upload-artifact@v3         # save files from job
uses: actions/download-artifact@v3       # retrieve saved files
```

---

## 13. Debugging Failed Pipelines

```bash
# ── From GitHub UI ────────────────────────────────────────────────────────────
# Click on the failed run → expand failed step → read logs
# Re-run failed jobs (not the whole workflow) to save time

# ── Add debug output to steps ─────────────────────────────────────────────────
- name: Debug context
  run: |
    echo "SHA: ${{ github.sha }}"
    echo "Ref: ${{ github.ref }}"
    echo "Actor: ${{ github.actor }}"
    kubectl get pods -n ${{ env.K8S_NAMESPACE }}
    helm list -n ${{ env.K8S_NAMESPACE }}

# ── Enable debug logging (set secret) ────────────────────────────────────────
# Repository → Settings → Secrets → Add:
# ACTIONS_STEP_DEBUG = true
# ACTIONS_RUNNER_DEBUG = true

# ── kubectl debug from pipeline ───────────────────────────────────────────────
- name: Debug K8s state on failure
  if: failure()
  run: |
    echo "=== Pods ==="
    kubectl get pods -n ${{ env.K8S_NAMESPACE }} -o wide
    echo "=== Events ==="
    kubectl get events -n ${{ env.K8S_NAMESPACE }} --sort-by='.lastTimestamp' | tail -30
    echo "=== Helm History ==="
    helm history ${{ env.HELM_RELEASE }} -n ${{ env.K8S_NAMESPACE }}
    echo "=== Failed Pod Logs ==="
    kubectl logs -l app=taskapi -n ${{ env.K8S_NAMESPACE }} --tail=50 || true
```

---

## 14. Hands-On Practice Tasks

```bash
# 1. Create .github/workflows/ directory in your TaskAPI repo
mkdir -p .github/workflows

# 2. Write ci.yml with:
#    - Node.js test job with postgres service container
#    - Build and push to ECR (use your ECR repo from Day 9)
#    - Trivy scan

# 3. Configure GitHub Secrets:
#    AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY
#    (Create IAM user with ECR + EKS permissions)

# 4. Push to a feature branch and open a PR
#    Watch CI run in the Actions tab

# 5. Add deploy-staging.yml triggered on push to develop

# 6. Create a develop branch and push to it
#    Watch staging deployment run

# 7. Add the GitHub Environment protection rule for production
#    Require your own approval

# 8. Create deploy-prod.yml triggered on push to main

# 9. Merge develop → main and approve the deployment
#    Watch full prod pipeline run

# 10. Deliberately break the smoke test
#     Watch automatic rollback trigger
#     Check Slack notification
```

---

## 15. Quiz — Test Yourself / Test Others

1. What is the difference between CI and CD?
2. What is a GitHub Actions Workflow, Job, and Step — and how do they relate?
3. Why do we run `needs: test` in the build job?
4. What does `--atomic` do in `helm upgrade --atomic`?
5. Why do we run database migrations BEFORE deploying the new application version?
6. What is a GitHub Environment and what protection rules can it enforce?
7. What does `workflow_dispatch` trigger enable?
8. Why is caching (`cache: 'npm'` and `type=gha`) important for CI/CD performance?
9. What happens to the deployment if the smoke test fails after `helm upgrade --atomic`?
10. What information should a failed deployment Slack notification include?

<details>
<summary>Answers</summary>

1. CI (Continuous Integration) automatically builds, tests, and validates every code change — finding bugs before they reach users. CD (Continuous Delivery/Deployment) automatically deploys every validated build to target environments. CI answers "is the code good?" CD answers "get it to production." CI feeds CD — you only deploy what CI has validated.

2. A Workflow is a YAML file (`.github/workflows/*.yml`) defining an entire automated process, triggered by events. A Job is a group of Steps that execute on the same runner machine — jobs run in parallel by default. A Step is a single task within a job: either a shell command (`run:`) or a reusable Action (`uses:`). Workflow → contains Jobs → contains Steps.

3. `needs: test` creates a dependency — the build job only starts after the test job completes successfully. Without it, you'd waste time and money building and pushing a Docker image for code that fails tests. Dependencies also ensure the overall workflow fails fast — a broken test stops the entire pipeline before expensive operations like ECR pushes.

4. `--atomic` makes the Helm upgrade all-or-nothing: if any Kubernetes resources fail to become healthy within the timeout, Helm automatically rolls back to the previous release revision and marks the upgrade as failed. Without it, a failed upgrade leaves the cluster in a partially upgraded, potentially broken state requiring manual intervention.

5. The new application version may require database schema changes (new columns, tables, indexes) that the old version didn't need. If you deploy the new app before running migrations: the new app starts with an incompatible schema → crashes → CrashLoopBackOff. If migration runs first: schema is ready before the new app starts → clean startup. Also, the old app version continues serving traffic during the migration (it's compatible with the old schema), ensuring zero downtime.

6. A GitHub Environment is a deployment target (staging, production) that can have protection rules: required reviewers (specific people must approve before deployment proceeds), wait timers (mandatory pause), branch restrictions (only deploy from main), and secret scoping (environment-specific secrets only visible during deployments to that environment). It creates a formal approval gate for sensitive deployments.

7. `workflow_dispatch` allows the workflow to be triggered manually from the GitHub Actions UI, optionally with user-provided inputs (like an image tag or a boolean flag). This is essential for: deploying a specific image tag without a new code push, re-running a deployment, hotfixing with a specific version, and operators who need to trigger deploys outside of the normal git flow.

8. Caching saves time by reusing previously downloaded/built artifacts. `cache: 'npm'` caches the npm package cache (~/.npm) so `npm ci` downloads only changed packages instead of everything. `type=gha` caches Docker image layers — unchanged layers are pulled from cache instead of rebuilt, saving 2-4 minutes per build. These optimisations reduce pipeline runtime from 10-15 minutes to 3-5 minutes and reduce egress costs.

9. Since `--atomic` was used, Helm already automatically rolled back to the previous revision when the deployment itself failed. But if `--atomic` succeeded (pods became healthy) and then the smoke test step failed, you need the explicit rollback step (`if: failure() && steps.deploy.outcome == 'failure'`). The `helm rollback` command restores the previous revision — traffic routes back to the old version, and the team is notified.

10. A failed production deployment notification should include: what failed (deployment/rollback), the image tag that was being deployed, the git SHA or commit reference, who triggered the deployment (github.actor), a direct link to the failed Actions run for log access, and a clear urgency signal (e.g., @oncall mention). Including the rollback status (was it successful?) helps the on-call engineer know the current state of production.

</details>

---

## 16. Summary

After today you can build and explain a complete production CI/CD pipeline:
- **CI phase** — automated testing with real service containers, Docker build with BuildKit, Trivy vulnerability scanning, ECR push
- **CD phase** — kubeconfig setup for EKS, migration Job before deployment, `helm upgrade --install --atomic`, smoke tests
- **GitHub Actions concepts** — workflows, jobs, steps, actions, runners, contexts, secrets, environments
- **Reusable workflows** — DRY principle for multi-environment pipelines
- **GitHub Environments** — approval gates, required reviewers, branch restrictions
- **Automatic rollback** — detecting deployment failure, `helm rollback` to previous revision
- **Git branching strategy** — feature → PR → develop (staging) → main (production) with protection
- **Pipeline optimisations** — npm cache, Docker layer cache, parallel jobs, path-ignore
- **Debugging** — step debug logs, `if: failure()` diagnostic steps, ACTIONS_STEP_DEBUG

**Coming up on Day 28:** Managed Kubernetes on AWS EKS — how EKS differs from self-managed Kubernetes, node groups vs Fargate, IAM integration, load balancer provisioning, and connecting everything you've learned to real AWS infrastructure.
