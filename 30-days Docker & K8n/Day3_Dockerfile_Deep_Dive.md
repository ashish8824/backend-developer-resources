# Day 3: Dockerfile Deep Dive — Build Your Own Images

**Goal for today:** Understand what a Dockerfile is, every important instruction in it, how layer caching works in practice, and build a real custom image from a Node.js/Express app (like your QueueCare/PulseBloom backend). By the end, you should be able to Dockerize *any* app from scratch and explain every line to a student.

**Format:** WHAT → WHY → HOW for every concept.

---

## 0. Quick Recap (open your teaching session with this)

- Day 1: Containers vs VMs, Docker = Client + Daemon + Registry, Image = blueprint, Container = running instance
- Day 2: Images are layered (Union FS), core CLI (`run`, `exec`, `logs`, `inspect`, `ps`, etc.), `run` vs `exec` distinction

**Today's connecting thought:**
> So far we've only used images that *someone else* built (nginx, ubuntu, node). Today we learn how to build our *own* image — packaging our own application into a portable, shareable container. This is the skill that makes Docker truly useful in real projects.

---

## 1. What is a Dockerfile?

### WHAT
A **Dockerfile** is a plain text file named literally `Dockerfile` (no extension) that contains a series of **instructions** — each instruction tells Docker how to build one layer of your image.

When you run `docker build`, Docker reads this file top-to-bottom, executes each instruction, and produces a final image you can run anywhere.

### WHY
Without a Dockerfile, you'd have to manually configure every container by hand every single time — not repeatable, not shareable, not automatable. The Dockerfile is the **source of truth** for how your image is constructed:
- Anyone can read it and understand exactly what's in the image
- You can version-control it in Git alongside your code
- CI/CD pipelines can build it automatically on every push
- It makes image builds **reproducible** — same Dockerfile = same image, always

### HOW
```
Dockerfile
    │
    └── docker build -t my-app:1.0 .
              │
              ▼
          Docker Image (layered)
              │
              └── docker run my-app:1.0
                        │
                        ▼
                  Running Container
```

**Teaching line:**
> "A Dockerfile is a recipe. `docker build` is the act of cooking that recipe. The image is the finished dish. A container is someone actually eating that dish."

---

## 2. The General Syntax Rule

Every Dockerfile instruction follows this pattern:

```dockerfile
INSTRUCTION argument(s)
```

- **Instructions are written in UPPERCASE** by convention (not required, but universal practice)
- Each instruction (almost always) creates a new layer in the image
- Instructions are executed **top to bottom**, in order

---

## 3. Every Important Dockerfile Instruction — WHAT, WHY, HOW

---

### 3.1 `FROM` — Set the base image

**WHAT:** The very first instruction in every Dockerfile. It declares which existing image yours is built *on top of*. You almost never build from absolute zero — you start with a base.

**WHY:** Real applications need an OS and runtime environment (like Node.js, Python, Java). Instead of manually installing all of that yourself, you start from an image that already has it. This is layer reuse in action.

**HOW:**
```dockerfile
FROM node:18
FROM node:18-alpine
FROM ubuntu:22.04
FROM python:3.11-slim
```

**Tag types to know (teach this — it matters for image size):**

| Tag suffix | What it means |
|---|---|
| (none / `latest`) | Full official image, largest |
| `-alpine` | Based on Alpine Linux — tiny (~5MB base), fewer tools |
| `-slim` | Stripped-down Debian — smaller than full, more than alpine |
| `-buster` / `-bullseye` | Specific Debian version name |

**Teaching line:**
> "Choosing your base image is like choosing which pre-furnished apartment to move into. Alpine is a studio flat with just the essentials — light and cheap. Ubuntu is a full house — bigger but comes with more tools."

```dockerfile
FROM node:18-alpine    # preferred for production — smallest size
```

---

### 3.2 `WORKDIR` — Set the working directory

**WHAT:** Sets the current working directory inside the container for all subsequent instructions (`RUN`, `COPY`, `CMD`, etc.). Creates the directory if it doesn't exist.

**WHY:** Without it, files would pile up in the root (`/`) which is messy and error-prone. `WORKDIR` is like `cd` + `mkdir` in one, giving your application a clean home inside the container.

**HOW:**
```dockerfile
WORKDIR /app
```
Now every subsequent instruction runs *from* `/app`. Any relative paths are relative to `/app`.

---

### 3.3 `COPY` — Copy files from host into the image

**WHAT:** Copies files or directories from your local machine (the **build context** — usually your project folder) into the image's filesystem.

**WHY:** Your application code, configuration files, and assets need to be inside the image for it to run.

**HOW:**
```dockerfile
COPY package.json .         # copy package.json into the current WORKDIR
COPY package*.json ./       # copy package.json AND package-lock.json
COPY . .                    # copy everything from local folder into WORKDIR
```

**Syntax:** `COPY <source-on-host> <destination-in-image>`

**Teaching nuance — copy in two stages (this is critical for layer caching, Section 6):**
```dockerfile
# Stage 1: Copy only dependency file first
COPY package*.json ./

# Stage 2: Install dependencies
RUN npm install

# Stage 3: THEN copy the rest of the app code
COPY . .
```
We'll explain *exactly why* this order matters in Section 6.

---

### 3.4 `ADD` — Like COPY, but more powerful

**WHAT:** Does everything `COPY` does, but also:
- Can unpack `.tar.gz` archives automatically
- Can fetch files from a URL

**WHY:** Rarely needed over `COPY`. Most Docker style guides recommend preferring `COPY` because it's explicit and predictable. Use `ADD` only when you specifically need its extra features.

**HOW:**
```dockerfile
ADD archive.tar.gz /app/     # auto-extracts into /app/
ADD https://example.com/file.txt /app/  # download into image
```

**Teaching rule:** Default to `COPY`. Use `ADD` only when you need unpacking or downloading.

---

### 3.5 `RUN` — Execute a command during build

**WHAT:** Executes a shell command *during the image build* and saves the result as a new layer.

**WHY:** This is how you install packages, compile code, create directories, set permissions, or do any transformation that should be "baked into" the image.

**HOW:**
```dockerfile
RUN npm install
RUN apt-get update && apt-get install -y curl
RUN mkdir -p /app/logs
```

**Critical tip — chain commands with `&&` to keep layers small:**
```dockerfile
# BAD: 3 separate RUN = 3 separate layers
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean

# GOOD: 1 RUN = 1 layer (smaller, cleaner image)
RUN apt-get update && \
    apt-get install -y curl && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

**Teaching line:**
> "`RUN` is what you'd type in a terminal to set up the environment. Whatever you'd do by hand to set up a server, you write as `RUN` commands in the Dockerfile — but they run during *build* time, not when the container starts."

---

### 3.6 `ENV` — Set environment variables

**WHAT:** Sets environment variables that are available **both** during the build and at runtime when the container runs.

**WHY:** Apps need configuration that shouldn't be hardcoded. `ENV` lets you set defaults in the image (e.g., `NODE_ENV=production`, `PORT=3000`).

**HOW:**
```dockerfile
ENV NODE_ENV=production
ENV PORT=3000
ENV APP_NAME=QueueCare
```

**Teaching distinction:**
- `ENV` in Dockerfile → baked into the image, available always
- `-e KEY=value` in `docker run` → overrides/adds at runtime without changing the image

```bash
docker run -e NODE_ENV=development my-app   # override ENV at runtime
```

---

### 3.7 `EXPOSE` — Document which port the app listens on

**WHAT:** Declares that the container will listen on a specific network port at runtime. It is **documentation only** — it does NOT actually publish the port to your host machine.

**WHY:** It tells other developers (and Docker itself) what port the app inside uses. You still need `-p` in `docker run` to actually make it reachable from outside.

**HOW:**
```dockerfile
EXPOSE 3000     # your Express app listens on port 3000
EXPOSE 8080
```

**Common confusion to clear up when teaching:**
> "`EXPOSE` is like writing 'this door exists' on a blueprint. But the door is still locked — you need `-p 3000:3000` in `docker run` to actually unlock and open it."

---

### 3.8 `CMD` — Default command when the container starts

**WHAT:** Specifies the default command that runs when a container starts from this image (if no command is provided in `docker run`).

**WHY:** Every container needs to *do* something when it starts — `CMD` defines what that "something" is by default.

**HOW:**
```dockerfile
CMD ["node", "server.js"]         # preferred: exec form (array)
CMD ["npm", "start"]
CMD node server.js                # shell form (not recommended)
```

**Exec form (array) vs Shell form:**
- **Exec form** `["node", "server.js"]` → runs the process directly as PID 1; signals (like Ctrl+C / SIGTERM) go straight to your process → cleanest shutdown
- **Shell form** `node server.js` → runs via `/bin/sh -c` as a wrapper process → signal handling is less reliable

**Always use exec form in production.**

---

### 3.9 `ENTRYPOINT` — The non-overridable starting command

**WHAT:** Like `CMD`, but it cannot be overridden by arguments passed to `docker run`. It defines the *fixed executable* the container always runs. CMD then becomes the *default arguments* to that executable.

**WHY:** Use `ENTRYPOINT` when you want the container to behave like a specific executable — the user can pass arguments but can't accidentally replace the executable entirely.

**HOW:**
```dockerfile
ENTRYPOINT ["node"]
CMD ["server.js"]
```
- `docker run my-app` → runs `node server.js`
- `docker run my-app app.js` → runs `node app.js` (CMD overridden by user arg)
- `docker run --entrypoint bash my-app` → runs `bash` (only way to override ENTRYPOINT)

**CMD vs ENTRYPOINT — the golden rule to teach:**
| | Overridable via `docker run args`? |
|---|---|
| `CMD` | Yes — user args replace CMD entirely |
| `ENTRYPOINT` | No — user args are *appended* as arguments to ENTRYPOINT |

**Most common pattern for Node.js apps:**
```dockerfile
# Either:
CMD ["node", "server.js"]          # simple apps — CMD is enough

# Or:
ENTRYPOINT ["node"]
CMD ["server.js"]                  # when you want to allow swapping the entry file
```

---

### 3.10 `ARG` — Build-time variables

**WHAT:** Declares variables that can be passed in at **build time** using `--build-arg`. Unlike `ENV`, these are NOT available at runtime in the running container.

**WHY:** Useful for things like version numbers or build-time tokens that shouldn't exist in the final image.

**HOW:**
```dockerfile
ARG NODE_VERSION=18
FROM node:${NODE_VERSION}
```
```bash
docker build --build-arg NODE_VERSION=20 -t my-app .
```

---

### 3.11 `VOLUME` — Declare mount points

**WHAT:** Declares that a directory inside the container should be persisted externally (as a volume). We cover this fully on Day 5 — just know the instruction exists.

```dockerfile
VOLUME ["/app/data"]
```

---

## 4. Complete Dockerfile for a Node.js/Express App (like QueueCare / PulseBloom)

Here is a **production-quality** Dockerfile for a Node.js Express API. Read through it line by line — every line uses something we just learned.

```dockerfile
# ── Base Image ──────────────────────────────────────────
# Use Node 18 on Alpine Linux → smallest official Node image
FROM node:18-alpine

# ── Working Directory ────────────────────────────────────
# All subsequent commands run from /app inside the container
WORKDIR /app

# ── Copy dependency files FIRST (Layer caching trick) ────
# Copy only package.json and package-lock.json, NOT the app code yet
# Reason: npm install only re-runs if these files change (see Section 6)
COPY package*.json ./

# ── Install Dependencies ─────────────────────────────────
# --omit=dev: skip devDependencies in production image (smaller, safer)
RUN npm install --omit=dev

# ── Copy Application Code ────────────────────────────────
# Now copy the rest of the code
# This layer changes frequently (every code edit) — so it goes last
COPY . .

# ── Environment Variables ─────────────────────────────────
ENV NODE_ENV=production
ENV PORT=3000

# ── Documentation ─────────────────────────────────────────
EXPOSE 3000

# ── Start Command ─────────────────────────────────────────
# Exec form: node gets SIGTERM directly → clean graceful shutdown
CMD ["node", "server.js"]
```

Build it:
```bash
docker build -t queuecare-api:1.0 .
```

Run it:
```bash
docker run -d --name queuecare -p 3000:3000 -e PORT=3000 queuecare-api:1.0
```

---

## 5. `.dockerignore` — What NOT to Copy

### WHAT
A `.dockerignore` file (in the same folder as your Dockerfile) lists files and folders that should be **excluded** from the build context — i.e., not sent to the Docker Daemon and not available for `COPY` instructions.

### WHY
Without it, every `COPY . .` would accidentally include:
- `node_modules/` → hundreds of MBs of packages (you install these fresh inside with `RUN npm install` anyway)
- `.git/` → your entire git history
- `.env` files → secrets/API keys
- Log files, local config, test results

This slows down builds, bloats images, and risks leaking secrets.

### HOW
```
# .dockerignore
node_modules
.git
.env
*.log
coverage
dist
.DS_Store
```

**Teaching line:**
> "`.dockerignore` is to Docker what `.gitignore` is to Git — it stops you from accidentally including junk or secrets."

---

## 6. Layer Caching — Why Instruction Order Matters Deeply

### WHAT
When Docker builds an image, it checks if a layer can be **reused from cache** instead of rebuilt from scratch. A layer is reused if: the instruction hasn't changed AND all previous layers are also unchanged.

**Critical rule: if a layer is invalidated (changed), all layers AFTER it are also rebuilt, even if those instructions haven't changed.**

### WHY
This is the #1 practical optimization in Dockerfile writing. Getting the order right means your builds go from minutes to seconds.

### HOW — Visualize it with the Node app example

**Bad order (naive):**
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY . .             # copies ALL code — changes every time you edit any file
RUN npm install      # ← THIS runs every single time, even if package.json didn't change
CMD ["node", "server.js"]
```
Every time you fix a typo in `server.js`, the entire `npm install` runs again. That's potentially minutes of wasted time.

**Good order (layer-cache-aware):**
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./   # Step A: only changes when dependencies change
RUN npm install          # Step B: only re-runs when Step A changes
COPY . .                 # Step C: changes every code edit — but npm install is already cached
CMD ["node", "server.js"]
```

**What happens on a code-only change (e.g., you fix a bug in `server.js`):**
```
FROM node:18-alpine     ✅ cached
WORKDIR /app            ✅ cached
COPY package*.json      ✅ cached (package.json didn't change)
RUN npm install         ✅ cached (skipped! huge time save)
COPY . .                ⚙️  rebuilt (code changed)
CMD [...]               ⚙️  rebuilt
```

**Teaching line:**
> "Docker layer caching is like a smart chef who remembers steps they've already done. Put the steps that change least (installing dependencies) before the steps that change most (copying code). That way, the chef only redoes the work that actually changed."

---

## 7. Multi-Stage Builds (Introduction — key concept for production)

### WHAT
A multi-stage build uses **multiple `FROM` instructions** in one Dockerfile. Each `FROM` starts a new build stage. You can selectively copy artifacts from one stage to the next.

### WHY
The best example: a compiled language (Java, TypeScript, Go). To compile, you need build tools (compiler, npm, Maven, etc.). But to *run* the compiled output, you don't need those tools. Without multi-stage builds, your final image would carry all the heavy build tools, even though they're useless at runtime.

### HOW — TypeScript Express app example
```dockerfile
# ── Stage 1: Build Stage ──────────────────────────────────
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install                  # installs dev deps too (tsc needs them)
COPY . .
RUN npm run build                # compiles TypeScript → JavaScript (into /app/dist)

# ── Stage 2: Production Stage ─────────────────────────────
FROM node:18-alpine AS production
WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev       # only production deps
COPY --from=builder /app/dist ./dist   # ← copy ONLY compiled output from stage 1
ENV NODE_ENV=production
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

**Result:**
- Stage 1 image: ~400MB (has TypeScript compiler, all devDependencies, source code)
- Stage 2 final image: ~80MB (only compiled JS, production deps, no compiler)

**Teaching line:**
> "Multi-stage builds are like building a piece of furniture. You need a full workshop with saws, drills, and sanders to make it. But you don't ship the workshop to the customer — you only ship the finished furniture."

---

## 8. Common Dockerfile Mistakes to Teach/Avoid

| Mistake | Problem | Fix |
|---|---|---|
| `COPY . .` before `RUN npm install` | Cache miss on every code change | Swap the order |
| No `.dockerignore` | Huge build context, secrets leaked | Always create `.dockerignore` |
| Running as root | Security risk | Add `USER node` before CMD |
| One `RUN` per apt-get command | Bloated layers | Chain with `&&` |
| Using `latest` tag | Non-reproducible builds | Pin to specific version: `node:18.20-alpine` |
| `CMD node server.js` (shell form) | Bad signal handling, zombie processes | Use exec form: `CMD ["node", "server.js"]` |

### Running as non-root (security best practice)
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY --chown=node:node package*.json ./  # give node user ownership of files
RUN npm install --omit=dev
COPY --chown=node:node . .
USER node                                # switch to non-root user before CMD
EXPOSE 3000
CMD ["node", "server.js"]
```

---

## 9. Full Dockerfile Reference Card

```dockerfile
FROM <image>:<tag>                  # base image
WORKDIR /path                       # set working directory
COPY <src> <dest>                   # copy from host to image
ADD <src> <dest>                    # like COPY + untar/download
RUN <command>                       # execute at build time → becomes a layer
ENV KEY=value                       # environment variable (build + runtime)
ARG NAME=default                    # build-time variable only
EXPOSE <port>                       # document which port the app uses
CMD ["executable", "arg"]           # default startup command (overridable)
ENTRYPOINT ["executable"]           # fixed startup command (not overridable)
VOLUME ["/path"]                    # declare persistent mount point
USER <username>                     # switch to non-root user
```

---

## 10. Hands-On Practice Tasks

1. Create a simple Express app (`server.js` that returns `"Hello from Docker!"` on `GET /`), write a Dockerfile for it using the Node.js example from Section 4, and build + run it.
2. Add a `.dockerignore` file and confirm `node_modules` is excluded from the build context.
3. Deliberately put `COPY . .` *before* `RUN npm install`, then run `docker build` twice — time the second build. Then flip the order and build twice again — observe the caching difference.
4. Run `docker image inspect <your-image>` and look for the `Layers` field — count how many layers your image has.
5. Run `docker images` and compare the size of `node:18` vs `node:18-alpine` — it's a dramatic difference.

---

## 11. Quiz — Test Yourself / Test Others

1. What does `WORKDIR` do? What happens if the directory doesn't exist?
2. What is the difference between `COPY` and `ADD`? When should you prefer each?
3. What is the difference between `CMD` and `ENTRYPOINT`?
4. Why do we copy `package.json` *before* copying all the app code?
5. What does `.dockerignore` do, and why is it important for security?
6. In a multi-stage build, what does `COPY --from=builder` mean?
7. Why is exec form `CMD ["node", "server.js"]` preferred over shell form `CMD node server.js`?
8. What happens to ALL layers after a changed layer in Docker's build cache?

<details>
<summary>Answers</summary>

1. Sets the working directory for all subsequent instructions; Docker creates it automatically if it doesn't exist.
2. `COPY` is explicit — only copies local files. `ADD` also unpacks tars and can fetch URLs. Prefer `COPY` by default; use `ADD` only when you need its extras.
3. `CMD` is the default command and can be completely overridden by passing arguments to `docker run`. `ENTRYPOINT` is fixed — user arguments are *appended* to it, not replaced.
4. Layer caching: `package.json` changes rarely. If you copy it first, `npm install` is cached and only re-runs when actual dependencies change — not every time you edit code.
5. It excludes files/folders from the build context so they don't get sent to the daemon or included via `COPY . .`. Without it, secrets in `.env` files or huge `node_modules` can leak into your image.
6. Copies a file or folder from a previous named build stage (`AS builder`) into the current stage — the core mechanism of multi-stage builds.
7. Exec form makes the process PID 1 directly, so OS signals like SIGTERM go straight to it — enabling clean, graceful shutdowns. Shell form wraps it in `/bin/sh -c`, which may absorb or mishandle signals.
8. All subsequent layers are also invalidated and rebuilt — even if those instructions themselves haven't changed. This is why order matters.

</details>

---

## 12. Summary

After today you can:
- Write a complete, production-quality Dockerfile from scratch for any Node.js/Express app
- Explain every major instruction: `FROM`, `WORKDIR`, `COPY`, `RUN`, `ENV`, `EXPOSE`, `CMD`, `ENTRYPOINT`, `ARG`
- Use `.dockerignore` correctly
- Understand and exploit Docker's layer caching with the correct instruction order
- Explain multi-stage builds and why they matter for production image size
- Avoid the most common Dockerfile anti-patterns

**Coming up on Day 4:** Docker Networking — how containers talk to each other and the outside world: bridge networks, host networks, custom networks, and container-to-container DNS resolution.
