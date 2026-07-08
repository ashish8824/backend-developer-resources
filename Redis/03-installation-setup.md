# Module 3: Installation & Setup

**Level:** Beginner

Before we can write any code, we need Redis running somewhere we can connect to. Let's cover every common option, since your setup depends on your OS.

---

## 1. Option A: Docker (Recommended for Everyone)

This is the easiest, most consistent way to run Redis, regardless of whether you're on Windows, Mac, or Linux. It avoids "works on my machine" issues.

**Step 1: Install Docker Desktop** (if you don't have it) from docker.com.

**Step 2: Run Redis in a container:**

```bash
# -d          → run in the background (detached mode)
# --name      → give the container a friendly name
# -p 6379:6379 → map port 6379 on your machine to port 6379 inside the container
#                 (6379 is Redis's default port)
docker run -d --name my-redis -p 6379:6379 redis
```

**Step 3: Confirm it's running:**

```bash
docker ps
# You should see "my-redis" listed with status "Up"
```

**Step 4: Talk to Redis directly using its CLI (inside the container):**

```bash
docker exec -it my-redis redis-cli
# This drops you into an interactive Redis shell, e.g.:
# 127.0.0.1:6379>
```

Try a quick test:

```bash
127.0.0.1:6379> PING
PONG
```

If you see `PONG`, Redis is alive and working.

---

## 2. Option B: Windows 11 (Native)

Redis doesn't officially support Windows anymore, but you have two solid choices:

**Option B1 — WSL2 (Windows Subsystem for Linux) — Recommended**

```bash
# 1. Open WSL2 (Ubuntu) terminal
# 2. Update packages
sudo apt update

# 3. Install Redis server
sudo apt install redis-server -y

# 4. Start Redis
sudo service redis-server start

# 5. Test it
redis-cli ping
# Should return: PONG
```

**Option B2 — Memurai (a Redis-compatible Windows-native build)**

Download from memurai.com and install like any normal Windows app if you don't want to deal with WSL2.

---

## 3. Option C: macOS

```bash
# Using Homebrew (the standard package manager for Mac)
brew install redis

# Start Redis as a background service
brew services start redis

# Test it
redis-cli ping
# Should return: PONG
```

---

## 4. Option D: Linux (Ubuntu/Debian)

```bash
sudo apt update
sudo apt install redis-server -y

# Start the service
sudo systemctl start redis-server

# Enable it to auto-start on boot
sudo systemctl enable redis-server

# Test it
redis-cli ping
```

---

## 5. Option E: Free Cloud Redis (No Install At All)

If you don't want to install anything locally, you can spin up a free-tier Redis instance on services like **Redis Cloud** (redis.io/try-free) or **Upstash** (upstash.com). They give you a hostname, port, and password to connect with — perfect for quick testing or small production apps, and great for deploying your Node.js app later since you won't need to manage a Redis server yourself.

---

## 6. Connecting to Redis from Your Terminal (`redis-cli`)

`redis-cli` is Redis's built-in command-line tool. Think of it as a direct chat window with Redis — before we bring Node.js into the picture, we should get comfortable talking to Redis directly.

```bash
# Connect to a local Redis instance
redis-cli

# Connect to a remote Redis instance (e.g., cloud-hosted)
redis-cli -h <host> -p <port> -a <password>
```

Try these commands once connected:

```bash
127.0.0.1:6379> PING
PONG

127.0.0.1:6379> SET greeting "Hello Redis"
OK

127.0.0.1:6379> GET greeting
"Hello Redis"

127.0.0.1:6379> DEL greeting
(integer) 1

127.0.0.1:6379> GET greeting
(nil)
```

We'll explain exactly what `SET`, `GET`, and `DEL` mean in Module 5 — for now, just confirm your setup works.

---

## 7. Setting Up Your Node.js Project

```bash
# 1. Create a project folder
mkdir redis-node-course
cd redis-node-course

# 2. Initialize a Node.js project (creates package.json)
npm init -y

# 3. Install Express (our web framework) and ioredis (our Redis client)
npm install express ioredis
```

**Why `ioredis` and not the official `redis` package?**

Both are excellent, actively maintained libraries. We'll use `ioredis` throughout this course because:

- It has a very clean Promise-based API (works great with `async/await`).
- It has excellent built-in support for Redis Cluster and Sentinel (Module 16).
- It's widely used in production Node.js apps.

(If you prefer the official `redis` npm package, everything you learn conceptually still applies — just the connection syntax differs slightly.)

---

## 8. Your First Connection to Redis from Node.js

```js
// redisClient.js

// Import the ioredis library
const Redis = require("ioredis");

// Create a new Redis client instance.
// By default, ioredis connects to 127.0.0.1:6379 (localhost, default Redis port)
const redis = new Redis({
  host: "127.0.0.1",   // where Redis is running
  port: 6379,           // Redis's default port
  // password: "yourpassword", // uncomment if your Redis instance requires a password
});

// ioredis emits a "connect" event once the connection succeeds
redis.on("connect", () => {
  console.log("✅ Connected to Redis successfully!");
});

// Always listen for errors so a broken Redis connection doesn't crash silently
redis.on("error", (err) => {
  console.error("❌ Redis connection error:", err);
});

// Export the client so other files in our app can reuse this same connection
module.exports = redis;
```

**Test it:**

```js
// test.js
const redis = require("./redisClient");

async function run() {
  // SET a value: key = "message", value = "Hello from Node.js"
  await redis.set("message", "Hello from Node.js");

  // GET the value back
  const value = await redis.get("message");

  console.log("Value from Redis:", value); // "Hello from Node.js"

  // Close the connection cleanly when done
  redis.disconnect();
}

run();
```

Run it:

```bash
node test.js
```

Expected output:

```
✅ Connected to Redis successfully!
Value from Redis: Hello from Node.js
```

If you see this, your entire pipeline — Redis server + Node.js client — is working correctly. 🎉

---

## 9. Common Setup Problems & Fixes

| Problem | Likely Cause | Fix |
|---|---|---|
| `ECONNREFUSED 127.0.0.1:6379` | Redis server isn't running | Start Redis (`docker start my-redis`, `brew services start redis`, etc.) |
| `NOAUTH Authentication required` | Redis requires a password but you didn't provide one | Add `password: "..."` to your client config |
| Commands work in `redis-cli` but not Node.js | Wrong host/port in your Node.js config | Double check host/port match your running Redis instance |
| Docker container exits immediately | Port 6379 already in use by another process | Stop the other process, or map to a different port like `-p 6380:6379` |

---

## 10. Summary

- Redis can be installed via Docker (easiest/most consistent), WSL2 on Windows, Homebrew on Mac, apt on Linux, or used as a free managed cloud service.
- `redis-cli` lets you talk to Redis directly from the terminal — useful for quick testing and debugging.
- We installed `ioredis` in our Node.js project and successfully connected to Redis.
- We ran our first real `SET`/`GET` commands from Node.js code.

---

## 11. Homework

1. Get Redis running using whichever method fits your OS, and confirm `PING` returns `PONG`.
2. Run the Node.js connection test above and confirm you see "Connected to Redis successfully!"
3. Try changing the `SET`/`GET` key/value pair to store your own name and print it.

---

**Next up:** [Module 4 — Redis Data Types](./04-redis-data-types.md)
