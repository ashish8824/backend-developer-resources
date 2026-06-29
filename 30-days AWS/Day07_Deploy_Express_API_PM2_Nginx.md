# Day 7: Deploy Express API on EC2 — Mini Project
### *(Making Your App Production-Ready with PM2 + Nginx)*

> **Roadmap reference:** Week 1, Day 7 — "Deploy Express API on EC2" | **Mini Project:** Deploy a Node.js REST API on EC2 | **Portfolio Project 1:** EC2 + PM2 + Nginx

---

## Why This Matters

Yesterday ended on a cliffhanger: your Express app died the moment you closed SSH. Today fixes that — and adds the second piece every real Node.js deployment needs. By the end of today you'll have completed **Week 1's mini project**, and have your first genuine portfolio piece: *"Node.js API deployed on AWS EC2 with PM2 and Nginx."*

Two new tools, two different jobs:

```
PM2    → Keeps your Node.js app running forever, restarts it on crash
Nginx  → Sits in front of your app, handles incoming traffic on port 80,
          and forwards it to your app running on port 5000
```

---

## 1. The Problem, Restated

```
Yesterday:
  You SSH in → run `node index.js` → app works
  You close SSH → app dies
        ↓
  Not acceptable for a real server — it must run 24/7,
  independent of whether you're connected to it.
```

---

## 2. PM2 — A Process Manager for Node.js

**PM2 is a process manager that keeps your Node.js application running continuously in the background, automatically restarts it if it crashes, and lets it survive your SSH session ending.**

```
Without PM2:
  SSH closes → Node process is a child of that session → it dies

With PM2:
  PM2 runs as an independent background daemon
  Your Node app runs UNDER PM2, not under your SSH session
  SSH closes → PM2 keeps running → your app keeps running
```

### Installing PM2

```bash
sudo npm install -g pm2
```

### Starting Your App with PM2 (Instead of `node index.js`)

```bash
cd my-first-api
pm2 start index.js --name my-api
```

```
Expected output: a status table showing your app as "online"
```

### Essential PM2 Commands

```bash
pm2 list              # see all running apps and their status
pm2 logs my-api       # view live logs (like watching console.log output)
pm2 restart my-api    # restart the app (e.g., after deploying new code)
pm2 stop my-api       # stop it
pm2 delete my-api     # remove it from PM2's management entirely
```

### Making PM2 Survive a Server Reboot

By default, if you **reboot the entire EC2 instance** (not just close SSH), PM2 itself also needs to restart. This one command fixes that permanently:

```bash
pm2 startup
# PM2 will print a command — copy and run that exact command it gives you

pm2 save
# Saves your current app list so it auto-resumes on every future reboot
```

```
Test it: close your SSH session entirely, wait a moment,
reconnect, run `pm2 list` — your app should still show "online".
```

---

## 3. Why That's Still Not Quite Production-Ready — Enter Nginx

Even with PM2 keeping your app alive, there's a remaining problem:

```
Your Express app runs on port 5000
        ↓
Users would have to visit:  http://your-ip:5000
                                          ^^^^^
        That's unusual — real websites don't make
        you type a port number.

Also: Node.js handling raw internet traffic directly
      isn't ideal — you'd rather have a dedicated,
      battle-tested web server in front of it.
```

**Nginx solves both problems** — it sits on the standard web port (80) and quietly forwards requests to your Express app running on 5000.

---

## 4. What is Nginx, and What Is a "Reverse Proxy"?

**Nginx is a high-performance web server, commonly used as a reverse proxy — a server that sits in front of your application and forwards incoming requests to it.**

```
                  Internet
                     │
                     │  http://your-ip  (port 80, no port needed)
                     ▼
              ┌─────────────┐
              │    Nginx      │   ← receives ALL public traffic
              │  (port 80)    │
              └─────────────┘
                     │
                     │  forwards internally to port 5000
                     ▼
              ┌─────────────┐
              │  Express App   │   ← managed by PM2, only talks
              │  (port 5000)   │      to Nginx, not the public internet
              └─────────────┘
```

> **Analogy:** Nginx is like a receptionist at a company. Visitors (internet traffic) talk to the receptionist (port 80), who then quietly walks them back to the right employee's desk (your app on port 5000). Visitors never need to know which floor/room number the employee sits in.

### Why Use a Reverse Proxy at All? (Real Reasons, Not Just Convention)

```
1. Clean URLs       → users visit your-domain.com, not your-domain.com:5000
2. SSL/HTTPS         → Nginx handles HTTPS certificates in one place,
                        your Node app doesn't need to deal with it
3. Load distribution → later, Nginx can forward to multiple app instances
4. Security buffer   → Nginx absorbs some attack patterns before they
                        ever reach your application code
5. Static file serving → Nginx serves images/CSS/JS faster than Node can
```

---

## 5. Installing & Configuring Nginx

### Install Nginx

```bash
sudo apt update
sudo apt install -y nginx
```

### Confirm It's Running

```bash
sudo systemctl status nginx
```

Visit `http://<your-elastic-ip>` in a browser right now (no port number) — you should see the **default Nginx welcome page**. That confirms Nginx itself is working, before you even touch configuration.

### Configure Nginx to Forward to Your Express App

```bash
sudo nano /etc/nginx/sites-available/default
```

Find the `location /` block inside the `server` block, and replace it with:

```nginx
location / {
    proxy_pass http://localhost:5000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_cache_bypass $http_upgrade;
}
```

```
Breaking this down:
  proxy_pass            → "send the request to my Express app on port 5000"
  proxy_set_header Host  → preserves the original domain in the request,
                            so your app knows what was actually requested
```

### Apply the Changes

```bash
# Test the config for syntax errors BEFORE applying
sudo nginx -t

# If the test passes, reload Nginx
sudo systemctl reload nginx
```

---

## 6. The Final Test

```bash
# Make sure your app is running under PM2
pm2 list
```

Now visit, **without any port number**:

```
http://<your-elastic-ip>
```

```json
{"message":"Hello from EC2!","server":"AWS Ubuntu Instance","status":"running"}
```

If you see this, **you've completed the Week 1 mini project** — a real, persistently-running Node.js REST API, properly fronted by a production-grade web server, exactly the way real backend services are deployed.

---

## 7. The Complete Architecture You Just Built

```
                       Internet
                          │
                          ▼
            ┌──────────────────────────┐
            │   Security Group           │  ← Day 5: allows 80, 443, 22(My IP)
            │   (port 80 open to all)     │
            └──────────────────────────┘
                          │
                          ▼
            ┌──────────────────────────┐
            │   Elastic IP                │  ← Day 5: fixed address
            │   13.234.99.50               │
            └──────────────────────────┘
                          │
                          ▼
            ┌──────────────────────────┐
            │   Nginx (port 80)            │  ← Today: reverse proxy
            └──────────────────────────┘
                          │
                          ▼
            ┌──────────────────────────┐
            │   PM2 (process manager)      │  ← Today: keeps app alive
            │     └── Express App (5000)    │
            └──────────────────────────┘
                          │
                          ▼
                  Your API Response
```

This single diagram is one of the best things you can draw in an interview when asked *"walk me through how you've deployed a Node.js app on AWS."*

---

## 8. Connecting This to Your Own Projects

This is essentially the manual, foundational version of how a service like an early **QueueCare backend** could be deployed before introducing Docker/ECS later in the roadmap (Week 4). Understanding this manual path first makes containerized deployment (Day 26-27) much easier to grasp later, because you'll already understand *why* each piece (process management, reverse proxy) exists — Docker and ECS just automate and formalize what you did by hand today.

---

## 9. Interview Questions (Practice Explaining These Out Loud)

**Q1: Why use PM2 instead of just running `node index.js` directly?**
> `node index.js` ties the process to your terminal/SSH session — it dies when that session ends. PM2 runs the app as an independent background process that survives disconnects and automatically restarts on crashes.

**Q2: What is a reverse proxy, and why use one in front of a Node.js app?**
> A reverse proxy (like Nginx) receives all public traffic and forwards it internally to your application. It enables clean URLs without port numbers, centralized SSL/HTTPS handling, better security, and the ability to later load-balance across multiple app instances.

**Q3: In the Nginx config, what does `proxy_pass http://localhost:5000;` actually do?**
> It tells Nginx to forward any matching incoming request to the application running locally on port 5000 — in this case, your Express app.

**Q4: How do you make PM2 automatically restart your app after the EC2 instance itself reboots?**
> Run `pm2 startup` (which generates and you then run a system startup script), followed by `pm2 save` to persist the current process list so it's restored automatically on every future boot.

**Q5: What's the difference between `pm2 restart` and `pm2 reload`?**
> (Good one to research further — at a high level: `restart` stops then starts the app, briefly dropping connections; `reload` is designed for zero-downtime restarts in cluster mode, more relevant once you scale beyond a single process.)

---

## 10. Hands-On Assignment for Today (Mini Project)

1. Install **PM2**, start your Day 6 Express app under it (`pm2 start index.js --name my-api`).
2. Run `pm2 startup` + `pm2 save`, and verify your app survives a full instance reboot.
3. Install **Nginx**, confirm the default welcome page loads on port 80.
4. Update the Nginx config to reverse-proxy to your Express app on port 5000.
5. Confirm `sudo nginx -t` passes, then `sudo systemctl reload nginx`.
6. Visit your Elastic IP **with no port number** and confirm your API's JSON response loads.
7. Write a short note in your own words covering:
   - Why PM2 was necessary
   - What a reverse proxy does, in your own words
   - Draw (even roughly, on paper) the full request path from browser to your Express app

---

## 11. Day 7 Goal — Can You Teach This?

By the end of today, you should be able to explain, without notes:

> *"PM2 keeps my Node.js app running independently of my SSH session and restarts it automatically if it crashes or the server reboots. Nginx sits in front of it as a reverse proxy, receiving public traffic on port 80 and forwarding it internally to my app on port 5000 — giving me clean URLs and a proper separation between the public-facing web server and my application code."*

---

## Week 1 Complete! 🎉

You've gone from "what is the cloud?" to a fully deployed, persistently-running Node.js API on a real AWS server — secured, addressed, process-managed, and reverse-proxied. That's a genuine, demoable, resume-worthy project: **Node.js API Deployment — EC2 + PM2 + Nginx.**

---

**Next up — Week 2 begins: Day 8: S3 Basics — Buckets & Objects**
