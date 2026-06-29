# Day 9: Docker Registries — Docker Hub, AWS ECR & Pushing Images to Production

**Goal for today:** Understand what a Docker registry is, how image tagging works, how to push and pull images to/from Docker Hub and AWS ECR, what image tagging strategies professional teams use, and how CI/CD pipelines automate the entire build-tag-push cycle. By the end, you should be able to take any image from your machine to a remote registry and deploy it from there — the foundation of every real deployment pipeline.

**Format:** WHAT → WHY → HOW for every concept.

---

## 0. Quick Recap (open your teaching session with this)

- Day 7: Capstone — full stack app Dockerized end to end
- Day 8: Best practices — image optimization, security, scanning, production-ready Dockerfiles

**Today's connecting thought:**
> Every image we built so far lives only on your laptop. The moment you want to deploy it to a server, share it with a teammate, or run it in Kubernetes — you need somewhere to store and distribute it. That's what a registry does. And since you've already deployed PulseBloom and QueueCare to AWS ECS Fargate, you've already *used* a registry (ECR) — today you'll understand exactly how it works.

---

## 1. What is a Docker Registry?

### WHAT
A **Docker registry** is a storage and distribution system for Docker images. It's a server that stores named, versioned images and lets you push (upload) and pull (download) them from anywhere with internet access and correct credentials.

### WHY
Without a registry:
- Images live only on the machine that built them
- Deploying to a server means manually copying images (impractical)
- Teammates can't pull your image without you sending them a tar file
- CI/CD pipelines can't push built images for deployment

With a registry:
- Build once on CI → push to registry → pull anywhere (dev machines, staging, production, Kubernetes nodes)
- The entire team shares the same versioned images
- Rollback = just tell the server to pull an older image tag

### HOW (the conceptual flow)
```
Developer / CI Pipeline               Registry                  Deployment Target
                                    (Docker Hub /
                                     AWS ECR / etc.)

  docker build → image              ←── stores ───→        docker pull → container
  docker push  ─────────────────────────►
                                                     ◄───── docker pull ──── ECS / K8s node
```

**Teaching line:**
> "A registry is to Docker images what GitHub is to source code — a central place to store, version, and distribute your work. Just as you `git push` code to GitHub and `git pull` it on another machine, you `docker push` images to a registry and `docker pull` them wherever you need to run them."

---

## 2. Registry Types — Public vs Private

| Type | Examples | Use case |
|---|---|---|
| **Public registry** | Docker Hub (hub.docker.com) | Open source images, personal projects |
| **Private cloud registry** | AWS ECR, Google GCR, Azure ACR | Production workloads, private images |
| **Self-hosted registry** | Harbor, Nexus, `registry:2` | On-premises, air-gapped environments |
| **CI-integrated registry** | GitHub Container Registry (ghcr.io), GitLab Registry | Images tied directly to your repo |

---

## 3. Image Naming and Tagging — The Full Picture

### WHAT
Every Docker image has a **fully qualified name** that tells Docker exactly where to find it:

```
registry/namespace/repository:tag

Examples:
docker.io/library/nginx:1.25-alpine       → official nginx on Docker Hub
docker.io/ashish8824/taskapi:1.0.0        → your image on Docker Hub
123456789.dkr.ecr.ap-south-1.amazonaws.com/pulsebloom-api:latest  → AWS ECR
ghcr.io/ashish8824/taskapi:main-abc1234   → GitHub Container Registry
```

**Parts explained:**
| Part | Meaning | Default if omitted |
|---|---|---|
| `registry` | Hostname of the registry | `docker.io` (Docker Hub) |
| `namespace` | Organisation or username | `library` (official images) |
| `repository` | The image name | Required |
| `tag` | Version label | `latest` |

### WHY tags matter
Tags are how you version your images. Without a thoughtful tagging strategy:
- `latest` gets overwritten every build — you can't roll back
- You don't know which code version is running in production
- Debugging production incidents becomes guesswork

### HOW — The `docker tag` command

```bash
# Tag a locally built image with a full registry path
docker tag my-local-image:1.0 ashish8824/taskapi:1.0.0

# Tag the same image with multiple tags
docker tag my-local-image:1.0 ashish8824/taskapi:1.0.0
docker tag my-local-image:1.0 ashish8824/taskapi:latest
docker tag my-local-image:1.0 ashish8824/taskapi:stable

# Tag for AWS ECR
docker tag my-local-image:1.0 \
  123456789.dkr.ecr.ap-south-1.amazonaws.com/taskapi:1.0.0
```

`docker tag` does NOT create a new image — it just adds a new name/alias pointing to the same image layers. No storage cost for multiple tags.

---

## 4. Tagging Strategies — What Professional Teams Use

This is one of the most practically useful sections for job interviews and real work.

### Strategy 1: Semantic Versioning (SemVer) — for versioned releases
```
myapp:1.0.0       → major.minor.patch
myapp:1.0         → points to latest 1.0.x patch
myapp:1           → points to latest 1.x version
myapp:latest      → always points to latest stable release
```

```bash
# Build and tag with all SemVer aliases at once
docker build -t myapp:1.2.3 .
docker tag myapp:1.2.3 myapp:1.2
docker tag myapp:1.2.3 myapp:1
docker tag myapp:1.2.3 myapp:latest
```

### Strategy 2: Git SHA tagging — for traceability (most common in CI/CD)
```
myapp:a3f8c2d       → exact git commit SHA (short)
myapp:main-a3f8c2d  → branch + SHA
```

**Why SHA tags are powerful:**
- Every image is tied to an exact commit — you know exactly what code is in any image
- Fully auditable: "which image is running in prod?" → check the tag → check the commit → see the diff
- Never ambiguous — two different builds never have the same SHA

```bash
# In CI/CD — get short git SHA automatically
GIT_SHA=$(git rev-parse --short HEAD)
docker build -t myapp:${GIT_SHA} .
docker push myapp:${GIT_SHA}
```

### Strategy 3: Branch tagging — for environments
```
myapp:main         → latest build from main branch (production)
myapp:develop      → latest build from develop branch (staging)
myapp:feature-xyz  → latest build from a feature branch (testing)
```

### Strategy 4: Combined (recommended production pattern)
```bash
# Tag with BOTH git SHA (for traceability) AND branch (for environment)
docker build -t myapp:${GIT_SHA} .
docker tag myapp:${GIT_SHA} myapp:main
docker tag myapp:${GIT_SHA} myapp:latest

docker push myapp:${GIT_SHA}    # push the immutable, traceable tag
docker push myapp:main           # push the mutable branch pointer
docker push myapp:latest         # push the mutable latest pointer
```

**Teaching line:**
> "Git SHA tags are like a permanent address — they never change and always point to exactly the right place. Branch tags like `latest` are like a signpost — they get moved to point at the newest address. You need both: SHA for reliability, branch tags for convenience."

---

## 5. Docker Hub — Push and Pull

### WHAT
Docker Hub (`hub.docker.com`) is Docker's official public registry — the default registry when no registry hostname is specified. Free tier allows unlimited public images and one private image.

### HOW

**Step 1: Create account at hub.docker.com**

**Step 2: Login from CLI**
```bash
docker login
# Enter your Docker Hub username and password when prompted
# ✅ Login Succeeded

# Or login with token (more secure — generate at hub.docker.com → Account Settings → Security)
docker login -u ashish8824 --password-stdin <<< "your-access-token"
```

**Step 3: Tag your image with your Docker Hub namespace**
```bash
# Format: <username>/<repository>:<tag>
docker build -t ashish8824/taskapi:1.0.0 .
# OR tag an existing local image
docker tag taskapi:1.0.0 ashish8824/taskapi:1.0.0
```

**Step 4: Push to Docker Hub**
```bash
docker push ashish8824/taskapi:1.0.0

# Output shows each layer being pushed (or "Layer already exists" if cached)
# The push refers to repository [docker.io/ashish8824/taskapi]
# 1.0.0: digest: sha256:abc123... size: 1234
```

**Step 5: Pull from anywhere**
```bash
# On any machine logged into Docker Hub
docker pull ashish8824/taskapi:1.0.0

# Or just run it directly — Docker pulls automatically if not local
docker run -d ashish8824/taskapi:1.0.0
```

**Step 6: Logout**
```bash
docker logout
```

---

## 6. AWS ECR — The Production Registry

### WHAT
**Amazon Elastic Container Registry (ECR)** is AWS's fully managed private Docker registry. Since you already have PulseBloom and QueueCare deployed on ECS Fargate, your images are already stored in ECR — today we'll understand and master exactly how that pipeline works.

### WHY ECR over Docker Hub for production
| Feature | Docker Hub (free) | AWS ECR |
|---|---|---|
| Private repos | 1 only | Unlimited |
| Integrated with AWS | No | ✅ Native (IAM auth) |
| ECS/EKS integration | Manual | ✅ Seamless |
| Image scanning | Basic | ✅ Deep (Amazon Inspector) |
| Lifecycle policies | No | ✅ Auto-delete old images |
| Network cost (in AWS) | Egress cost | ✅ Free within same region |

### HOW — Complete ECR Workflow

**Prerequisites:**
```bash
# Install AWS CLI (if not already installed)
pip install awscli --break-system-packages

# Configure with your credentials
aws configure
# AWS Access Key ID: <your key>
# AWS Secret Access Key: <your secret>
# Default region name: ap-south-1     ← Mumbai / your region
# Default output format: json
```

**Step 1: Create an ECR repository**
```bash
# Create the repository (one-time setup per image name)
aws ecr create-repository \
  --repository-name taskapi \
  --region ap-south-1

# Output includes the repositoryUri — save this!
# 123456789.dkr.ecr.ap-south-1.amazonaws.com/taskapi
```

**Step 2: Authenticate Docker with ECR**
```bash
# ECR uses temporary tokens (valid 12 hours) — run this before every push/pull session
aws ecr get-login-password --region ap-south-1 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.ap-south-1.amazonaws.com

# ✅ Login Succeeded
```

**Step 3: Build and tag for ECR**
```bash
# Set variables for cleaner commands
AWS_ACCOUNT_ID=123456789
REGION=ap-south-1
REPO_NAME=taskapi
GIT_SHA=$(git rev-parse --short HEAD)
ECR_URI=${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/${REPO_NAME}

# Build the image
docker build -t ${REPO_NAME}:${GIT_SHA} .

# Tag for ECR (two tags: SHA for traceability, latest for convenience)
docker tag ${REPO_NAME}:${GIT_SHA} ${ECR_URI}:${GIT_SHA}
docker tag ${REPO_NAME}:${GIT_SHA} ${ECR_URI}:latest
```

**Step 4: Push to ECR**
```bash
docker push ${ECR_URI}:${GIT_SHA}
docker push ${ECR_URI}:latest
```

**Step 5: Pull from ECR (on any authorized machine/service)**
```bash
docker pull ${ECR_URI}:${GIT_SHA}
```

**Step 6: Set up lifecycle policy (auto-delete old images — cost saving)**
```bash
# Keep only the last 10 images tagged with 'latest' or any SHA
aws ecr put-lifecycle-policy \
  --repository-name taskapi \
  --lifecycle-policy '{
    "rules": [
      {
        "rulePriority": 1,
        "description": "Keep last 10 images",
        "selection": {
          "tagStatus": "any",
          "countType": "imageCountMoreThan",
          "countNumber": 10
        },
        "action": { "type": "expire" }
      }
    ]
  }'
```

---

## 7. CI/CD Pipeline — Automated Build → Tag → Push

### WHAT
A CI/CD pipeline automates the entire process: on every push to Git, it builds the Docker image, tags it, pushes it to the registry, and (optionally) deploys it — without any manual steps.

### WHY
Manual `docker build` → `docker push` is:
- Error-prone (forget to push, wrong tag, wrong registry)
- Not reproducible (different developer machines = different builds)
- Not auditable (no record of who pushed what when)

CI/CD gives you:
- Every image built in a clean, consistent environment
- Every push produces a traceable image (SHA tag)
- Automated deployment after successful build
- Build failures prevent bad images from reaching production

### HOW — GitHub Actions Pipeline (complete example)

This is the exact pipeline pattern used for services like PulseBloom/QueueCare deploying to ECS Fargate.

#### `.github/workflows/docker-build-push.yml`

```yaml
name: Build, Push & Deploy

on:
  push:
    branches:
      - main          # trigger on every push to main
  pull_request:
    branches:
      - main          # also run on PRs (build + test only, no push)

env:
  AWS_REGION: ap-south-1
  ECR_REPOSITORY: taskapi
  ECS_SERVICE: taskapi-service
  ECS_CLUSTER: taskapi-cluster

jobs:
  build-and-push:
    name: Build, Scan & Push
    runs-on: ubuntu-latest

    steps:
      # ── 1. Checkout code ──────────────────────────────────────────────────
      - name: Checkout repository
        uses: actions/checkout@v4

      # ── 2. Set up Docker Buildx (for advanced builds) ─────────────────────
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      # ── 3. Configure AWS credentials ─────────────────────────────────────
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      # ── 4. Login to Amazon ECR ────────────────────────────────────────────
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      # ── 5. Generate image metadata (tags) ────────────────────────────────
      - name: Extract metadata for Docker
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}
          tags: |
            type=sha,prefix=,suffix=,format=short   # git SHA tag: abc1234
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

      # ── 6. Build and push to ECR ──────────────────────────────────────────
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile
          push: ${{ github.event_name != 'pull_request' }}  # don't push on PRs
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha          # GitHub Actions cache (speeds up builds)
          cache-to: type=gha,mode=max

      # ── 7. Scan image for vulnerabilities ─────────────────────────────────
      - name: Scan image with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:latest
          severity: 'CRITICAL,HIGH'
          exit-code: '1'               # fail pipeline on critical vulnerabilities

      # ── 8. Deploy to ECS Fargate ──────────────────────────────────────────
      - name: Deploy to Amazon ECS
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ecs-task-def.json
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true
```

**What this pipeline does on every push to `main`:**
1. ✅ Checks out your code
2. ✅ Logs into ECR automatically (using stored secrets)
3. ✅ Generates tags: `abc1234` (SHA) + `latest`
4. ✅ Builds the image using your Dockerfile
5. ✅ Pushes to ECR (skips on PRs)
6. ✅ Scans for CRITICAL/HIGH vulnerabilities — fails if found
7. ✅ Deploys the new image to ECS Fargate

**GitHub Secrets to configure** (Settings → Secrets → Actions):
```
AWS_ACCESS_KEY_ID      → your AWS access key
AWS_SECRET_ACCESS_KEY  → your AWS secret key
```

---

## 8. GitHub Container Registry (ghcr.io) — Free Private Registry

If you don't want to use AWS ECR, GitHub Container Registry is free for public repos and included with GitHub Pro/Team for private repos — and it integrates directly with GitHub Actions.

```yaml
# In GitHub Actions:
- name: Login to GitHub Container Registry
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}   # automatic! No manual secret needed

- name: Build and push
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: ghcr.io/ashish8824/taskapi:${{ github.sha }}
```

Images are at: `ghcr.io/<github-username>/<repo-name>:<tag>`

---

## 9. Registry CLI Commands — Full Reference

```bash
# ── Authentication ─────────────────────────────────────────────────────────────
docker login                                    # Docker Hub (prompts for creds)
docker login -u username --password-stdin       # pipe password in (safer)
docker login ghcr.io                            # GitHub Container Registry
docker login 123456789.dkr.ecr.ap-south-1.amazonaws.com  # AWS ECR

docker logout                                   # logout from Docker Hub
docker logout 123456789.dkr.ecr.ap-south-1.amazonaws.com # logout from ECR

# ── Tagging ───────────────────────────────────────────────────────────────────
docker tag <source-image>:<tag> <target-image>:<tag>

# ── Push ──────────────────────────────────────────────────────────────────────
docker push ashish8824/taskapi:1.0.0
docker push ashish8824/taskapi:latest

# ── Pull ──────────────────────────────────────────────────────────────────────
docker pull ashish8824/taskapi:1.0.0
docker pull --platform linux/amd64 myimage:1.0   # pull specific architecture

# ── Inspect remote image (without pulling) ────────────────────────────────────
docker manifest inspect ashish8824/taskapi:1.0.0

# ── Remove image from local cache ─────────────────────────────────────────────
docker rmi ashish8824/taskapi:1.0.0

# ── AWS ECR specific ──────────────────────────────────────────────────────────
aws ecr describe-repositories                    # list all ECR repos
aws ecr list-images --repository-name taskapi    # list images in a repo
aws ecr describe-images --repository-name taskapi  # detailed image info
aws ecr batch-delete-image \
  --repository-name taskapi \
  --image-ids imageTag=old-tag                   # delete specific image
```

---

## 10. Multi-Platform Builds (ARM + AMD64)

### WHAT
Modern development often involves M1/M2 Macs (ARM architecture) but production runs on AMD64 (x86) servers. If you build on an M1 Mac without specifying platform, you'll create an ARM image that won't run on AMD64 servers.

### WHY
ECR, ECS Fargate, and most cloud servers run AMD64. Building for the wrong architecture is a common source of "it works on my machine but crashes in prod" bugs.

### HOW
```bash
# Build specifically for AMD64 (even on ARM Mac)
docker build --platform linux/amd64 -t myapp:1.0 .

# Build for BOTH platforms simultaneously (multi-platform build)
# Requires Docker Buildx (built into Docker Desktop)
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ashish8824/taskapi:1.0.0 \
  --push .    # must push directly, can't load multi-arch into local daemon

# In GitHub Actions — always specify platform for consistent prod builds:
- name: Build and push
  uses: docker/build-push-action@v5
  with:
    platforms: linux/amd64    # explicitly AMD64 for production
    push: true
    tags: ...
```

---

## 11. Image Pull Policy — How Kubernetes and ECS Use Registry Images

Even though Kubernetes comes in Week 2–3, this is the right moment to preview this concept.

When a container platform (ECS, Kubernetes) needs to run a container:
1. It checks if the image is already cached on the node
2. Based on the **pull policy**, it decides whether to pull from the registry

| Pull Policy | Behaviour |
|---|---|
| `Always` | Always pull from registry (even if cached) — ensures latest version |
| `IfNotPresent` | Use cached if available, pull only if missing |
| `Never` | Never pull — fail if not cached |

**Why this matters for your tagging strategy:**
- If you use `latest` tag with `IfNotPresent` policy → you'll **never** pick up new versions (cached `latest` is used forever)
- Using SHA tags + `Always` policy → always pulls the exact version you deployed
- **Production recommendation:** use SHA tags + `Always` pull policy

---

## 12. Hands-On Practice Tasks

1. Create a Docker Hub account and login with `docker login`
2. Build your Day 7 TaskAPI image and push it to Docker Hub:
   ```bash
   docker build -t <your-dockerhub-username>/taskapi:1.0.0 .
   docker push <your-dockerhub-username>/taskapi:1.0.0
   ```
3. Pull it on a different terminal (simulate another machine):
   ```bash
   docker rmi <your-username>/taskapi:1.0.0   # remove local cache
   docker pull <your-username>/taskapi:1.0.0  # pull from Hub
   ```
4. Tag the same image with 3 different tags (SemVer pattern) and push all three
5. Create an ECR repository using the AWS CLI and push an image to it
6. Run `docker manifest inspect <your-image>` and read the output
7. Create the GitHub Actions workflow from Section 7 in your TaskAPI repo — set up the AWS secrets and push to `main` to trigger it

---

## 13. Quiz — Test Yourself / Test Others

1. What is a Docker registry and why is it necessary for deployment?
2. What do the four parts of a fully qualified image name represent: `registry/namespace/repository:tag`?
3. Why is tagging with `latest` alone insufficient for production?
4. What is a Git SHA tag and why is it preferred in CI/CD?
5. What does `docker tag` actually do to the image data?
6. What is the difference between Docker Hub and AWS ECR — when would you use each?
7. Why do you need to re-authenticate with ECR every 12 hours, and what command does this?
8. In the GitHub Actions pipeline, why do we skip the push step on pull requests?
9. What problem do multi-platform builds solve?
10. How does an ECR lifecycle policy save money?

<details>
<summary>Answers</summary>

1. A registry is a central server for storing, versioning, and distributing Docker images. Without it, images can only exist on the machine that built them — you couldn't deploy to servers, share with teammates, or use CI/CD pipelines.
2. `registry` = the hostname of the server storing images (default: docker.io). `namespace` = organisation or username (default: `library` for official images). `repository` = the image name. `tag` = the version label (default: `latest`).
3. `latest` is mutable — it gets overwritten on every build. You lose the ability to roll back to a specific version, audit what's in production, or reproduce a past build. You need immutable version tags alongside it.
4. A Git SHA tag ties the image directly to the exact commit that produced it. It's unique, immutable, and fully traceable — given a SHA tag, you can look up the exact code, diff, and author. No two builds ever share the same SHA.
5. `docker tag` creates a new name/alias pointing to the same image layers — it does NOT copy or duplicate any data. Multiple tags share the same underlying layers with zero extra storage cost.
6. Docker Hub is a public registry — free for public images, suitable for open source and personal projects. AWS ECR is a private, AWS-managed registry — better for production, integrates natively with IAM, ECS, and EKS, and avoids egress costs when used within AWS.
7. ECR uses temporary authentication tokens that expire after 12 hours for security. The command is: `aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <ecr-uri>`
8. PRs may contain unreviewed code that shouldn't be pushed to the registry or deployed. Building on PRs validates the Dockerfile works, but pushing is reserved for verified, merged code on `main`.
9. Multi-platform builds solve the architecture mismatch between development machines (often ARM — Apple M1/M2) and production servers (usually AMD64/x86). Building for the wrong architecture produces images that crash on deployment.
10. ECR charges per GB stored. Without lifecycle policies, old image versions accumulate indefinitely. A lifecycle policy automatically deletes images beyond a defined count or age — keeping storage costs bounded without manual cleanup.

</details>

---

## 14. Summary

After today you can:
- Explain what a registry is and why it's essential for real deployments
- Understand the full anatomy of a Docker image name (`registry/namespace/repo:tag`)
- Apply professional tagging strategies: SemVer, Git SHA, branch tags, combined
- Push and pull images to/from Docker Hub with full authentication
- Create ECR repositories, authenticate, tag, push, and set lifecycle policies
- Write a complete GitHub Actions CI/CD pipeline that builds, scans, pushes, and deploys
- Handle multi-platform builds for ARM Mac → AMD64 production
- Use GitHub Container Registry as a free Docker Hub alternative

**Coming up on Day 10:** Why orchestration? The problems Docker Compose can't solve at scale — single point of failure, no auto-healing, no rolling deployments, no traffic management. This is the motivation lecture for Kubernetes — after Day 10, you'll understand not just *how* Kubernetes works, but *why* it had to be invented.
