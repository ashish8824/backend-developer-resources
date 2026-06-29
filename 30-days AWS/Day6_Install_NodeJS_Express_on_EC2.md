# Day 6: Install Node.js on EC2 & Run a Simple Express App
### *(Turning Your Bare Linux Server Into an Actual Web Server)*

> **Roadmap reference:** Week 1, Day 6 — "Install Node.js on EC2, Run simple Express app"

---

## Why This Matters

For the last two days, your EC2 instance has been a blank Ubuntu machine — secured, reachable, but doing nothing. Today, you turn it into what you actually came here to build: **a real web server running your code.**

This is the day where everything from Days 1–5 (Region, Security Group, SSH, Key Pair, Elastic IP) finally has a *purpose* — they all exist to safely get you to this exact moment.

---

## 1. The Big Picture — What You're Building Today

```
Your Laptop                    EC2 Instance (Ubuntu)
─────────────                  ──────────────────────
SSH in  ──────────────────►    1. Install Node.js
                                2. Create a simple Express app
                                3. Run it on port 5000
                                4. App listens for requests

Browser/curl request  ────────► http://<your-public-ip>:5000
                                         │
                                         ▼
                                "Hello from EC2!" response
```

By the end of today, you'll be able to open a browser, hit your server's public IP, and see a response coming from code running on a real AWS server — not localhost.

---

## 2. Step 1: SSH Into Your Instance

You did this on Day 4 — same command, just reconnecting:

```bash
ssh -i my-first-server-key.pem ubuntu@<your-elastic-ip>
```

> Notice you're now connecting via the **Elastic IP** from Day 5, not the old changing public IP — this is exactly why you set that up.

---

## 3. Step 2: Update the Server & Install Node.js

Always update package lists first — this is standard practice on any fresh Linux server.

```bash
# Update package lists
sudo apt update

# Upgrade existing packages (optional but good practice)
sudo apt upgrade -y
```

### Installing Node.js (Using NodeSource — the Recommended Method)

Ubuntu's default package manager often has an outdated Node.js version, so the standard practice is to use **NodeSource's setup script** to get a current LTS version.

```bash
# Download and run the NodeSource setup script for Node.js LTS
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -

# Install Node.js (this also installs npm)
sudo apt install -y nodejs

# Verify installation
node -v
npm -v
```

```
Expected output (versions will vary):
v20.x.x      ← Node.js version
10.x.x       ← npm version
```

> **Why NodeSource instead of `apt install nodejs` directly?** Ubuntu's default repositories often lag several major versions behind. NodeSource keeps an up-to-date LTS release, which matters if your app uses newer JS/Node features.

---

## 4. Step 3: Create a Simple Express App

```bash
# Create a project folder
mkdir my-first-api
cd my-first-api

# Initialize a Node.js project
npm init -y

# Install Express
npm install express
```

### Create the App File

```bash
nano index.js
```

Paste this in (then `Ctrl+O`, `Enter` to save, `Ctrl+X` to exit nano):

```javascript
const express = require('express');
const app = express();
const PORT = 5000;

app.get('/', (req, res) => {
  res.json({
    message: "Hello from EC2!",
    server: "AWS Ubuntu Instance",
    status: "running"
  });
});

app.listen(PORT, '0.0.0.0', () => {
  console.log(`Server running on port ${PORT}`);
});
```

### Why `'0.0.0.0'` and Not `'localhost'`?

This single detail trips up almost everyone the first time:

```
app.listen(PORT, 'localhost', ...)   ← WRONG for EC2
  → Only accepts connections from WITHIN the instance itself
  → Your browser, connecting from outside, gets a timeout

app.listen(PORT, '0.0.0.0', ...)     ← CORRECT for EC2
  → Accepts connections from any network interface,
    including external ones
  → Your Security Group (Day 5) still controls who's
    actually allowed to reach it
```

> Binding to `0.0.0.0` doesn't bypass your Security Group — it just makes the app *listen* on all interfaces. Your Security Group rule for port 5000 (which you added yesterday) is what actually controls external access.

---

## 5. Step 4: Run the App

```bash
node index.js
```

```
Expected output:
Server running on port 5000
```

### Test It — From Inside the Server First

Open a **second terminal tab**, SSH in again, and test locally on the server itself:

```bash
curl http://localhost:5000
```

```json
{"message":"Hello from EC2!","server":"AWS Ubuntu Instance","status":"running"}
```

### Test It — From Your Own Browser/Laptop

Now open a browser on your own laptop and visit:

```
http://<your-elastic-ip>:5000
```

If you see the JSON response, **you've successfully deployed your first application to AWS.**

```
If this DOESN'T work, check in this order:
  1. Is the app still running? (terminal should show no errors)
  2. Did you bind to '0.0.0.0', not 'localhost'?
  3. Does your Security Group allow inbound on port 5000
     from 0.0.0.0/0? (Day 5's task)
  4. Are you using the correct Elastic IP?
```

---

## 6. The Problem With Running It This Way (A Preview of Tomorrow)

Right now, your app is running directly in your SSH terminal session. Try this:

```bash
# Press Ctrl+C, or just close your terminal / disconnect SSH
```

```
Result: Your app STOPS immediately.

Why: The Node process is a "child" of your SSH session.
     When the session ends, the process dies with it.
```

This is obviously not how real production apps work — you can't keep an SSH session open forever. Tomorrow (Day 7), you'll fix this properly using a **process manager (PM2)** that keeps your app running in the background, restarts it if it crashes, and survives you disconnecting.

For today, just notice this limitation — it's the natural motivation for tomorrow's lesson.

---

## 7. Connecting This to Your Own Projects

This exact workflow — Node.js + Express, deployed to a Ubuntu EC2 instance — is the foundation of how you'd manually deploy something like an early version of **QueueCare's backend** before introducing containers or managed services. Many real companies still run simple backend services exactly this way: a basic EC2 instance, Node.js installed, Express listening on a port behind a Security Group.

```
What you did today, mapped to production thinking:

mkdir + npm init + npm install   →   same as any local setup
app.listen(PORT, '0.0.0.0')      →   the one EC2-specific change
Security Group port rule          →   replaces "firewall settings"
Elastic IP                        →   replaces "static domain pointer"
```

---

## 8. Interview Questions (Practice Explaining These Out Loud)

**Q1: What's the standard way to install an up-to-date Node.js version on Ubuntu?**
> Using the NodeSource setup script to add their repository, then installing via `apt`, since Ubuntu's default repositories often carry outdated Node.js versions.

**Q2: Why must you bind your Express app to `0.0.0.0` instead of `localhost` on EC2?**
> `localhost` only accepts connections originating from within the instance itself. `0.0.0.0` listens on all network interfaces, allowing external traffic to reach the app — assuming the Security Group also permits it.

**Q3: If your Express app is running and reachable on `0.0.0.0:5000`, but a Security Group still blocks port 5000, can outside users reach it?**
> No — binding to `0.0.0.0` only controls what the application itself listens on. The Security Group is a separate, additional layer that must also explicitly allow the traffic.

**Q4: Why does your Node.js app stop running when you close your SSH session?**
> Because the Node process is a child process of that SSH session by default — when the session terminates, so does the process. This is why production deployments use a process manager (like PM2) to keep apps running independently.

**Q5: What command would you use to quickly test if your API is responding, directly from the server itself?**
> `curl http://localhost:5000` — useful for confirming the app works locally before troubleshooting network/Security Group issues for external access.

---

## 9. Hands-On Assignment for Today

1. SSH into your Day 4/5 EC2 instance.
2. Install Node.js via the NodeSource LTS script — verify with `node -v` and `npm -v`.
3. Create the simple Express app exactly as shown in Section 4.
4. Run it, and confirm it responds via `curl` **from inside the server**.
5. Open it in your **own browser**, using your Elastic IP and port 5000.
6. Intentionally close your SSH session, then try refreshing the browser — confirm the app has actually stopped (this proves you understand *why* tomorrow's lesson exists).
7. Write a short note in your own words covering:
   - Why `0.0.0.0` is required instead of `localhost`
   - Why the app stops when you disconnect SSH
   - What you think tomorrow's fix (PM2) might need to solve

---

## 10. Day 6 Goal — Can You Teach This?

By the end of today, you should be able to explain, without notes:

> *"I installed Node.js on my EC2 instance using NodeSource for an up-to-date version, then ran a simple Express app bound to 0.0.0.0 so it accepts outside traffic — with my Security Group's port rule as the actual gatekeeper for who can reach it. Right now the app only survives as long as my SSH session stays open, which is exactly the problem a process manager like PM2 solves tomorrow."*

---

**Next up — Day 7: Deploy Express API on EC2 (Mini Project: Full Node.js REST API Deployment)**
