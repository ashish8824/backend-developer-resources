# Day 8: Docker Best Practices — Image Optimization, Security & Production Readiness

**Goal for today:** Learn how to build Docker images that are small, fast, secure, and production-ready. Understand why image size matters, how to systematically reduce it, the security risks in poorly written Dockerfiles, and the professional practices used by engineering teams shipping to production. By the end, you should be able to audit any Dockerfile and make it better.

**Format:** WHAT → WHY → HOW for every concept.

---

## 0. Quick Recap (open your teaching session with this)

Week 1 covered all four Docker primitives:
- Images (Dockerfile, layers, caching)
- Containers (lifecycle, CLI)
- Networks (bridge, DNS, port mapping)
- Volumes (named volumes, bind mounts)
- Docker Compose (multi-container orchestration)

**Today's connecting thought:**
> In Week 1 you learned *how* to use Docker. Today you learn how to use it *well*. There's a big difference between a Dockerfile that works and one that's production-ready. The gap between those two is what today covers.

---

## 1. Why Image Size Matters

Before diving into techniques, understand *why* this is worth caring about:

| Impact | Why size matters |
|---|---|
| **Pull time** | Larger images take longer to pull from registry on every deploy — slowing CI/CD pipelines and cold starts |
| **Storage cost** | Registry storage costs money — hundreds of services × large images = significant cost |
| **Attack surface** | Every tool/binary in an image is a potential exploit vector. Smaller images = fewer things that can be attacked |
| **Startup time** | Containers start faster from cached layers — smaller images cache better |
| **Network bandwidth** | In Kubernetes, images are pulled on every new node — bandwidth costs multiply across a cluster |

**Teaching line:**
> "A 1.2GB image vs a 120MB image doesn't just save disk space — it saves 10x pull time on every deploy, on every node, forever. In a 50-microservice system, that compounds enormously."

---

## 2. Technique 1 — Choose the Right Base Image

### WHAT
Your base image (`FROM`) is the single biggest lever for image size. It's the foundation — everything you add on top can only make it larger.

### WHY
The same application logic packaged into different base images can produce dramatically different image sizes:

```
node:18               → 950MB  (full Debian, all tools)
node:18-slim          → 240MB  (stripped Debian)
node:18-alpine        → 170MB  (Alpine Linux — minimal)
gcr.io/distroless/nodejs18-debian12 → 110MB  (no shell, no package manager)
```

### HOW

**Tier 1: Alpine-based images (use these by default)**
```dockerfile
FROM node:18-alpine      # ✅ great default — small, well-maintained
FROM python:3.11-alpine
FROM nginx:alpine
```
Alpine Linux is built around `musl libc` and `busybox` — extremely minimal (~5MB base).

**Tier 2: Slim/Debian-slim (when Alpine causes compatibility issues)**
```dockerfile
FROM node:18-slim        # Debian-based but stripped — use when Alpine causes issues
```
Some npm packages with native C++ bindings compile differently against Alpine's `musl libc` vs Debian's `glibc`. If `npm install` fails on Alpine, use `-slim`.

**Tier 3: Distroless images (maximum security)**
```dockerfile
FROM gcr.io/distroless/nodejs18-debian12    # no shell, no package manager at all
```
Google's distroless images contain only the application and its runtime dependencies — no `bash`, no `sh`, no package manager. You literally cannot `docker exec` into a distroless container and get a shell. Used when security is paramount.

**Tier 4: Scratch (for compiled binaries — Go, Rust)**
```dockerfile
FROM scratch             # completely empty image — nothing at all
```
Used for compiled languages that produce a single static binary. The entire image is just the binary. Not applicable to Node.js.

**Teaching rule:**
> Start with Alpine. Move to slim if Alpine causes compatibility issues. Use distroless if you need maximum security and don't need shell access. Use scratch only for compiled static binaries.

---

## 3. Technique 2 — Multi-Stage Builds (Production Standard)

You saw this in Day 3 — but now we go deeper on the optimization angle.

### WHAT
Multi-stage builds let you use a heavy build environment but produce a lean final image by copying only the built artifacts across.

### WHY
Without multi-stage builds:
- TypeScript compiler lives in your production image (useless, ~100MB)
- Test frameworks live in production (security risk, wasted space)
- Build caches and intermediate files live in production image
- All devDependencies in production (nodemon, jest, ts-node — all junk in production)

### HOW — Full comparison

**Before (single stage — naive):**
```dockerfile
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install          # installs ALL deps including devDependencies
COPY . .
RUN npm run build        # compiles TypeScript
# Final image: ~950MB, contains: TypeScript, jest, nodemon, source .ts files, everything
CMD ["node", "dist/server.js"]
```

**After (multi-stage — professional):**
```dockerfile
# ── Stage 1: Install all deps + build ─────────────────────────────────────────
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci                           # npm ci: faster, stricter than npm install
COPY . .
RUN npm run build                    # compile TypeScript → dist/

# ── Stage 2: Lean production image ───────────────────────────────────────────
FROM node:18-alpine AS production
WORKDIR /app

# Copy only production deps (reinstall cleanly in production stage)
COPY package*.json ./
RUN npm ci --omit=dev

# Copy only the compiled output — nothing else
COPY --from=builder /app/dist ./dist

# Final image: ~170MB, contains ONLY: node runtime, prod deps, compiled JS
CMD ["node", "dist/server.js"]
```

**Result:** ~950MB → ~170MB. No TypeScript compiler. No source files. No devDependencies.

---

## 4. Technique 3 — Layer Optimization

### 4.1 Chain RUN commands (reduce layer count)

```dockerfile
# ❌ BAD: 4 layers, intermediate apt cache stored in image
RUN apt-get update
RUN apt-get install -y curl wget
RUN apt-get clean
RUN rm -rf /var/lib/apt/lists/*

# ✅ GOOD: 1 layer, cleanup happens in same layer (not baked in)
RUN apt-get update && \
    apt-get install -y curl wget && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

**Why it matters:** Docker layers are additive. Even if you `rm -rf` something in a later layer, the data still exists in the earlier layer — it's just hidden. Only cleaning up IN THE SAME `RUN` command truly removes it from the image.

### 4.2 Use `npm ci` instead of `npm install`

```dockerfile
# ❌ npm install: modifies package-lock.json, slower, non-deterministic
RUN npm install

# ✅ npm ci: uses package-lock.json exactly, faster, deterministic, fails if lock is out of sync
RUN npm ci --omit=dev
```

`npm ci` is purpose-built for CI/CD and Docker builds:
- **Faster** — skips resolution step, uses lock file directly
- **Deterministic** — exact same versions every time
- **Strict** — fails if `package-lock.json` is out of sync with `package.json` (catches drift early)

### 4.3 Order layers by change frequency (recap from Day 3)

```dockerfile
FROM node:18-alpine
WORKDIR /app

# Changes rarely → put first (cache-friendly)
COPY package*.json ./
RUN npm ci --omit=dev

# Changes constantly → put last (cache misses here are unavoidable but cheap)
COPY . .

CMD ["node", "server.js"]
```

---

## 5. Technique 4 — Security Hardening

This is the most important section for production. Insecure images are one of the most common real-world vulnerabilities.

### 5.1 Never run as root

**WHAT:** By default, processes inside a Docker container run as the `root` user. This means if an attacker exploits your application (e.g., via a dependency vulnerability), they have root access inside the container — and potentially to the host.

**WHY:** The principle of least privilege — the process should only have the permissions it actually needs. A Node.js web server needs to read files and bind to a port, not run arbitrary system commands as root.

**HOW:**
```dockerfile
FROM node:18-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .

# The official node image already has a non-root user called 'node'
# Transfer ownership of the app files to this user
RUN chown -R node:node /app

# Switch to the non-root user BEFORE the CMD instruction
USER node

EXPOSE 3000
CMD ["node", "server.js"]
```

For images without a built-in non-root user (e.g., Ubuntu-based):
```dockerfile
# Create a dedicated user and group
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
USER appuser
```

### 5.2 Use specific image tags, never `latest`

```dockerfile
# ❌ BAD: 'latest' changes without warning — your build becomes non-reproducible
FROM node:latest
FROM postgres:latest

# ✅ GOOD: pin to a specific version
FROM node:18.20.4-alpine3.19
FROM postgres:15.6-alpine3.19
```

**Why `latest` is dangerous:**
- Someone updates the base image → your app silently starts using a different Node version
- A security vulnerability is patched in a new version → you might unknowingly pull it on next build (or not)
- Builds stop being deterministic → "it worked yesterday" bugs appear

### 5.3 Use `.dockerignore` religiously

```
# .dockerignore
.git
.env
.env.*
node_modules
npm-debug.log
*.log
coverage/
.nyc_output/
dist/
.DS_Store
Thumbs.db
*.md
docker-compose*.yml
Dockerfile*
.github/
tests/
```

**Never let `.env` files reach the build context.** If a `.env` file gets `COPY`-ed into an image and that image is pushed to a registry, your secrets are exposed to anyone who can pull the image — even if you delete the file in a later layer (the data is still in the earlier layer).

### 5.4 Don't store secrets in ENV instructions

```dockerfile
# ❌ NEVER DO THIS — secret is baked into every layer of the image forever
ENV DB_PASSWORD=mysecretpassword
ENV API_KEY=sk-1234567890abcdef

# ✅ Pass secrets at runtime, never bake into image
# docker run -e DB_PASSWORD=$DB_PASSWORD my-image
# Or use Docker Secrets / AWS Secrets Manager / Vault
```

**Why it's dangerous even if "private":** `docker history my-image` shows the content of every layer including `ENV` instructions. Anyone with access to the image can read your secrets.

### 5.5 Set a read-only filesystem where possible

```bash
# Run container with read-only filesystem — can only write to explicitly mounted volumes
docker run --read-only \
  --tmpfs /tmp \              # allow writes to /tmp (in RAM)
  -v uploads:/app/uploads \   # allow writes to uploads (via volume)
  my-app
```

In Docker Compose:
```yaml
services:
  api:
    read_only: true
    tmpfs:
      - /tmp
```

### 5.6 Drop unnecessary Linux capabilities

```bash
# Drop ALL capabilities then add back only what's needed
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE my-app
```

In Compose:
```yaml
services:
  api:
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE    # only if binding to port < 1024
```

---

## 6. Technique 5 — Image Scanning for Vulnerabilities

### WHAT
Image scanning analyses every layer of your image, identifies the software packages installed, and checks them against CVE (Common Vulnerability and Exposure) databases — flagging known security vulnerabilities.

### WHY
Your base image ships with dozens of OS packages. Some will have known vulnerabilities. Without scanning you're blind to these — you might be shipping a container with a critical vulnerability that's trivial to exploit.

### HOW

**Option 1: Docker Scout (built into Docker Desktop)**
```bash
# Scan a local image
docker scout cves my-app:1.0

# Quick overview
docker scout quickview my-app:1.0

# Compare with another version
docker scout compare my-app:1.0 my-app:2.0
```

**Option 2: Trivy (free, open source — excellent)**
```bash
# Install trivy
brew install aquasecurity/trivy/trivy     # macOS
apt-get install trivy                      # Ubuntu

# Scan an image
trivy image my-app:1.0
trivy image node:18-alpine

# Only show CRITICAL and HIGH severity
trivy image --severity CRITICAL,HIGH my-app:1.0

# Scan and output as JSON (for CI/CD integration)
trivy image --format json --output results.json my-app:1.0
```

**Option 3: Snyk**
```bash
# Free tier available
docker scan my-app:1.0     # integrates with Docker Hub
```

**Integrating scanning into CI/CD:**
```yaml
# GitHub Actions example
- name: Scan image for vulnerabilities
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'my-app:${{ github.sha }}'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'          # fail the pipeline if CRITICAL vulns found
```

---

## 7. Technique 6 — Understand and Reduce Image Layers

### WHAT
`docker history` shows you every layer of an image — what instruction created it, how large it is, and what command was run.

### HOW
```bash
# See all layers of an image
docker history my-app:1.0

# More readable format
docker history --no-trunc my-app:1.0

# Sample output:
# IMAGE         CREATED BY                              SIZE
# a1b2c3d4     CMD ["node" "server.js"]                0B
# e5f6g7h8     EXPOSE 3000                             0B
# i9j0k1l2     USER node                               0B
# m3n4o5p6     COPY . .                                45kB
# q7r8s9t0     RUN npm ci --omit=dev                   95MB
# u1v2w3x4     COPY package*.json ./                   3kB
# y5z6a7b8     WORKDIR /app                            0B
# c9d0e1f2     /bin/sh -c apk add --no-cache wget      2.3MB
# g3h4i5j6     FROM node:18-alpine                     167MB
```

**What to look for:**
- Any unexpectedly large layers → what's being installed/copied there?
- Are cleanup commands in the same `RUN` as install commands?
- Are secrets visible in `CMD`, `ENV`, or `RUN` layer commands?

---

## 8. Technique 7 — Use `HEALTHCHECK` in the Dockerfile

You used healthchecks in Compose on Day 6. You can also bake them directly into the Dockerfile so they work even when the container runs without Compose.

```dockerfile
FROM node:18-alpine

# Install wget for the healthcheck command
RUN apk add --no-cache wget

WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .

RUN chown -R node:node /app
USER node

ENV NODE_ENV=production
EXPOSE 3000

# Healthcheck baked into the image
HEALTHCHECK --interval=30s --timeout=10s --start-period=15s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

CMD ["node", "server.js"]
```

Now `docker ps` shows `(healthy)` or `(unhealthy)` for this container regardless of whether it was started with Compose or plain `docker run`.

---

## 9. The Production-Ready Dockerfile — Complete Example

Taking every technique from today and applying it to a Node.js/Express API:

```dockerfile
# ══════════════════════════════════════════════════════════════════════
# Stage 1 — Install production dependencies
# ══════════════════════════════════════════════════════════════════════
FROM node:18.20.4-alpine3.19 AS deps

WORKDIR /app

# Copy dependency manifests only (layer cache trick)
COPY package*.json ./

# npm ci: deterministic, strict, fast
RUN npm ci --omit=dev


# ══════════════════════════════════════════════════════════════════════
# Stage 2 — Build (only needed for TypeScript / bundling)
# ══════════════════════════════════════════════════════════════════════
FROM node:18.20.4-alpine3.19 AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci                # full deps including devDeps for build

COPY . .
RUN npm run build         # compile TS → JS into /app/dist


# ══════════════════════════════════════════════════════════════════════
# Stage 3 — Production image (lean, secure, final)
# ══════════════════════════════════════════════════════════════════════
FROM node:18.20.4-alpine3.19 AS production

# Install only what's needed for healthcheck
RUN apk add --no-cache wget && \
    rm -rf /var/cache/apk/*

WORKDIR /app

# Copy production deps from stage 1
COPY --from=deps /app/node_modules ./node_modules

# Copy compiled output from stage 2
COPY --from=builder /app/dist ./dist

# Copy package.json (needed by Node.js for module resolution)
COPY package.json ./

# Transfer ownership to non-root user
RUN chown -R node:node /app

# Drop to non-root user
USER node

# Runtime environment
ENV NODE_ENV=production
ENV PORT=3000

EXPOSE 3000

# Health check baked into image
HEALTHCHECK --interval=30s --timeout=10s --start-period=15s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

# Exec form: PID 1 = node → signals handled correctly → graceful shutdown
CMD ["node", "dist/server.js"]
```

---

## 10. Docker Best Practices — Quick Audit Checklist

Use this to review any Dockerfile — yours or someone else's:

```
✅ BASE IMAGE
  [ ] Using specific tag (not 'latest')?
  [ ] Using alpine or slim variant?
  [ ] Is the base image necessary? Could a smaller one work?

✅ LAYER CACHING
  [ ] package.json / requirements.txt copied BEFORE source code?
  [ ] Dependency install runs BEFORE source copy?
  [ ] apt/apk installs cleaned up in the same RUN command?

✅ SECURITY
  [ ] Running as non-root user (USER instruction)?
  [ ] No secrets in ENV, ARG, or RUN instructions?
  [ ] .dockerignore present and covers .env files?
  [ ] Unnecessary packages not installed?

✅ MULTI-STAGE
  [ ] Build tools absent from final image?
  [ ] DevDependencies absent from final image?
  [ ] Source files absent from final image (if compiled)?

✅ PRODUCTION READINESS
  [ ] CMD uses exec form (array), not shell form?
  [ ] EXPOSE documents the correct port?
  [ ] HEALTHCHECK defined?
  [ ] Image scanned for vulnerabilities?
```

---

## 11. Image Size — Before vs After Comparison

| Dockerfile quality | Approximate size (Node.js app) |
|---|---|
| `FROM node:18` + single stage, no cleanup | 950MB+ |
| `FROM node:18-alpine` + single stage | ~200MB |
| `FROM node:18-alpine` + multi-stage, prod deps only | ~120MB |
| `FROM node:18-alpine` + multi-stage + TypeScript compiled | ~100MB |
| `FROM gcr.io/distroless/nodejs18` + compiled | ~80MB |

Reducing from 950MB to 100MB is a 9.5x improvement. Across a 20-service system pulling images on every CI/CD run, this can save hours per week in pipeline time.

---

## 12. Hands-On Practice Tasks

1. Take the Dockerfile from Day 7 and run `docker history taskapi-api` — identify the biggest layers
2. Run `trivy image taskapi-api` (install Trivy first) — review the vulnerability report
3. Refactor the Day 7 Dockerfile using today's techniques:
   - Pin the base image to a specific version
   - Add multi-stage build if not already there
   - Add `USER node` with `chown`
   - Add `HEALTHCHECK` instruction
   - Chain any `RUN` commands that can be combined
4. Run `docker images` — compare the size before and after your refactor
5. Run `docker history` on both the old and new image — compare layer sizes
6. Try building with `FROM node:18` vs `FROM node:18-alpine` vs `FROM node:18-slim` — just run `docker build` and compare sizes with `docker images`
7. Add `read_only: true` to the api service in your Compose file and confirm it still starts

---

## 13. Quiz — Test Yourself / Test Others

1. Why does a smaller Docker image improve security (not just storage)?
2. What is the difference between `FROM node:18-alpine`, `FROM node:18-slim`, and `FROM node:18`?
3. Why is `FROM node:latest` dangerous in production?
4. Why must `apt-get clean` be in the **same** `RUN` instruction as `apt-get install`?
5. What is the difference between `npm install` and `npm ci` — and why prefer `npm ci` in Docker?
6. Why should you never write `ENV DB_PASSWORD=mysecretpassword` in a Dockerfile?
7. What does `USER node` do, and why is it a security best practice?
8. What does `docker history my-image` show you?
9. What is a distroless image and when would you use one?
10. Name three things that should always be in `.dockerignore`.

<details>
<summary>Answers</summary>

1. Every tool and binary in an image is a potential exploit vector. A smaller image has fewer packages, fewer binaries, and fewer known CVEs — reducing the attack surface. Even if the app is compromised, there are fewer tools available for an attacker to use inside the container.
2. `node:18` is full Debian (~950MB), all tools included. `node:18-slim` is stripped Debian (~240MB), most tools removed. `node:18-alpine` is Alpine Linux (~170MB), minimal OS with only essentials. Prefer Alpine; fall back to slim for native module compatibility issues.
3. `latest` resolves to a different image over time — builds become non-deterministic. A security patch or major version bump can silently break your application on the next build without any change to your Dockerfile.
4. Docker layers are immutable and additive. If you install in one `RUN` and clean in another, the install data still exists in the first layer — the cleanup only hides it in subsequent layers. Cleanup must be in the same `RUN` to actually reduce layer size.
5. `npm install` re-resolves dependencies and can modify `package-lock.json`. `npm ci` uses the lock file exactly, is faster (skips resolution), is deterministic, and fails if the lock file is inconsistent with `package.json`. This makes builds reproducible and catches version drift early.
6. `ENV` values are baked into the image's layer history. Anyone running `docker history my-image` can read them. If the image is pushed to a registry, secrets become accessible to anyone who can pull it — even across private registries if access controls are weak.
7. `USER node` switches the running process from root to the non-root `node` user. If the application is exploited, the attacker only has the limited permissions of the `node` user rather than full root access to the container (and potentially the host).
8. `docker history` shows every layer of an image — the instruction that created each layer, its size, and the command run. Used to audit image size, find unexpectedly large layers, and check if secrets were accidentally baked in.
9. Distroless images (by Google) contain only the application runtime and its dependencies — no shell, no package manager, no OS utilities. Use when maximum security is required and you don't need to shell into the container for debugging.
10. Any three of: `.env`, `node_modules`, `.git`, `*.log`, `coverage/`, `dist/`, `.DS_Store`, `Dockerfile*`, `docker-compose*.yml`, `tests/`.

</details>

---

## 14. Summary

After today you can:
- Choose the right base image for any situation (alpine vs slim vs distroless vs scratch)
- Write multi-stage Dockerfiles that separate build and production environments
- Optimize layers with `&&` chaining, `npm ci`, and correct instruction ordering
- Apply security hardening: non-root user, pinned tags, no secrets in ENV, read-only filesystem
- Scan images for vulnerabilities using Docker Scout or Trivy
- Use `docker history` to audit any image
- Audit any Dockerfile against the 20-point production checklist

**Coming up on Day 9:** Docker Registries — pushing your image to Docker Hub and AWS ECR, image tagging strategies, and how CI/CD pipelines build and push images automatically. This connects directly to how your PulseBloom app is already deployed on ECS Fargate.
