# Day 5: Security Groups, Key Pairs & Elastic IP
### *(Locking Down Your Server — and Giving It a Permanent Address)*

> **Roadmap reference:** Week 1, Day 5 — "Security Groups, Key Pairs, Elastic IP"

---

## Why This Matters

Yesterday you launched a server and connected to it. Today you go one level deeper into **two questions every backend developer must answer for every server they deploy:**

1. *"Exactly who is allowed to reach this server, and on which ports?"* → **Security Groups**
2. *"Will this server's IP address change every time I restart it?"* → **Elastic IP**

Get these wrong, and you either lock yourself out of your own server, or accidentally expose it to the entire internet (remember Day 4's SSH warning — today you'll understand exactly *why* that rule exists, mechanically).

---

## 1. Security Groups — Your Instance's Personal Firewall

**A Security Group is a virtual firewall that controls inbound and outbound traffic for one or more EC2 instances.**

You actually already used one on Day 4 without fully unpacking it — when you set "Allow SSH from My IP." Today, let's understand the full mechanics.

```
                    ┌─────────────────────────┐
   Internet  ─────► │     Security Group        │ ─────► EC2 Instance
                    │   (checks every request)  │
                    └─────────────────────────┘

   Request allowed in?  → Checked against INBOUND rules
   Response allowed out? → Checked against OUTBOUND rules
```

### Inbound vs Outbound Rules

| Rule Type | Controls | Example |
|---|---|---|
| **Inbound** | Traffic coming INTO your instance | "Allow SSH (port 22) from my IP" |
| **Outbound** | Traffic leaving your instance | "Allow all outbound" (default, so your server can call external APIs) |

### A Typical Security Group for a Node.js Backend

```
INBOUND RULES:
┌──────────┬──────────┬─────────────────┬───────────────────────┐
│ Type     │ Port     │ Source          │ Purpose                │
├──────────┼──────────┼─────────────────┼───────────────────────┤
│ SSH      │ 22       │ My IP only      │ Your admin access      │
│ HTTP     │ 80       │ 0.0.0.0/0       │ Public web traffic     │
│ HTTPS    │ 443      │ 0.0.0.0/0       │ Public secure traffic  │
│ Custom   │ 5000     │ 0.0.0.0/0       │ Your Express API port  │
└──────────┴──────────┴─────────────────┴───────────────────────┘

OUTBOUND RULES:
┌──────────┬──────────┬─────────────────┬───────────────────────┐
│ Type     │ Port     │ Destination     │ Purpose                │
├──────────┼──────────┼─────────────────┼───────────────────────┤
│ All      │ All      │ 0.0.0.0/0       │ Server can call out    │
│          │          │                 │ (npm installs, APIs)   │
└──────────┴──────────┴─────────────────┴───────────────────────┘
```

> Notice the difference: **HTTP/HTTPS/your API port** are meant for the public (0.0.0.0/0 = "anyone"), but **SSH is restricted to your IP only**. This is the core lesson from Day 4, now made concrete — public-facing services need to be open, but admin access should never be.

### Security Groups Are "Stateful" — An Important, Often-Asked Detail

```
Stateful firewall (Security Groups):
  Inbound request allowed in
        ↓
  Response is AUTOMATICALLY allowed back out
  (you don't need a matching outbound rule for the reply)
```

This is different from a traditional ("stateless") firewall, where you'd need to explicitly write rules for both directions of every single conversation. Security Groups handle the "reply" traffic for you automatically — this distinction comes up often in interviews.

### Security Groups vs Network ACLs (a quick preview)

You don't need to master this today — it's covered properly when you reach VPC (Day 15) — but it's worth knowing this exists:

| | Security Group | Network ACL |
|---|---|---|
| Applies to | Instance level | Subnet level |
| State | Stateful (auto-allows replies) | Stateless (must allow both directions) |
| Rules | Allow only | Allow AND Deny |

---

## 2. Key Pairs — A Deeper Look

You used a Key Pair on Day 4 to SSH in. Let's understand *why* this system is more secure than a traditional password.

```
PASSWORD-BASED LOGIN (weaker):
  Password can be guessed, brute-forced, or phished
  Same password often reused across systems
  Easy to share accidentally (typed, texted, etc.)

KEY-PAIR LOGIN (stronger):
  Private key is a long cryptographic file — effectively
    impossible to brute-force
  Never transmitted over the network during login (cryptographic
    proof is used instead — math, not the actual key, crosses the wire)
  Can't be "guessed" the way a weak password can
```

### Managing Key Pairs Responsibly

```
One Key Pair per project/team is common practice, but:

GOOD: Each developer ideally has their own key + own IAM user
       so access can be individually revoked

RISKY: Sharing one .pem file across an entire team
       → If one person leaves, you can't revoke just their access
         without rotating the key for everyone
```

> **Practical tip for your own projects:** For something like QueueCare or PulseBloom, if you ever bring on a collaborator, give them their *own* key pair rather than sharing yours — it keeps access auditable and revocable.

---

## 3. Elastic IP — A Permanent Public Address

### The Problem Elastic IP Solves

By default, every time you **stop and start** (not just reboot) an EC2 instance, **its public IP address changes.**

```
Day 1: Launch instance → Public IP: 13.234.50.10
Day 2: Stop instance, then start it again
Day 2: New Public IP: 13.234.78.92   ← completely different!
```

If you'd pointed a domain name or shared that IP with anyone, it's now broken. This is a very common "why did my server stop working?!" moment for beginners.

### What is an Elastic IP?

**An Elastic IP is a static, public IPv4 address that you reserve and can attach to (or move between) your EC2 instances — it never changes until you explicitly release it.**

```
Without Elastic IP:
  Instance restarts → IP changes → DNS/bookmarks break

With Elastic IP:
  Instance restarts → IP stays the SAME → nothing breaks
```

```
       Elastic IP: 13.234.99.50  (fixed, reserved by you)
              │
              │  attached to
              ▼
        EC2 Instance
   (can stop/start freely — the
    Elastic IP stays pointed at it)
```

### Important Cost Detail (Connects Back to Day 2)

```
Elastic IP attached to a RUNNING instance     → FREE
Elastic IP attached to a STOPPED instance     → CHARGED
Elastic IP reserved but NOT attached to anything → CHARGED
```

> AWS deliberately charges for *unused* Elastic IPs — this discourages people from hoarding scarce public IPv4 addresses they're not actually using. This is exactly the kind of "hidden cost" Day 2's billing alerts are designed to catch.

### When Do You Actually Need an Elastic IP?

```
YES, use one when:
  - You're pointing a domain name (Route 53) at this server
  - External services need to whitelist your server's IP
  - You restart your instance frequently during development

NO, skip it when:
  - You're behind a Load Balancer (Day 18) — the Load Balancer
    has its own stable DNS name, individual instance IPs don't matter
  - This is a short-lived test instance you'll terminate soon
```

---

## 4. Putting It All Together — Today's Full Picture

```
                 Internet
                    │
                    ▼
         ┌─────────────────────┐
         │   Elastic IP          │  ← Fixed address, survives restarts
         │   13.234.99.50         │
         └─────────────────────┘
                    │
                    ▼
         ┌─────────────────────┐
         │   Security Group       │  ← Firewall: only allows
         │   - SSH: My IP only     │     specific ports/sources
         │   - HTTP/HTTPS: public  │
         └─────────────────────┘
                    │
                    ▼
         ┌─────────────────────┐
         │   EC2 Instance          │
         │   (accessed via          │  ← Key Pair required
         │    Key Pair for SSH)     │     to actually log in
         └─────────────────────┘
```

Three independent layers of control, all working together to keep your server both **reachable** and **secure**.

---

## 5. Interview Questions (Practice Explaining These Out Loud)

**Q1: What is a Security Group?**
> A virtual firewall attached to EC2 instances that controls inbound and outbound traffic based on rules you define for ports, protocols, and source/destination IPs.

**Q2: What does it mean that Security Groups are "stateful"?**
> If inbound traffic is allowed in, the response traffic is automatically allowed back out — you don't need a separate matching outbound rule for replies.

**Q3: Why is it dangerous to allow SSH from 0.0.0.0/0?**
> It opens port 22 to literally anyone on the internet, making the server an easy target for automated brute-force login attempts. SSH should be restricted to specific trusted IPs.

**Q4: What problem does an Elastic IP solve?**
> By default, an EC2 instance's public IP changes every time it's stopped and started. An Elastic IP gives you a fixed, static public IP that stays the same across restarts.

**Q5: Is an Elastic IP always free?**
> No — it's free only while attached to a running instance. If it's attached to a stopped instance, or reserved but not attached to anything at all, AWS charges for it.

**Q6: What's the difference between a Security Group and a Network ACL?**
> Security Groups operate at the instance level and are stateful (auto-allow replies). Network ACLs operate at the subnet level and are stateless (you must explicitly allow both inbound and outbound directions).

---

## 6. Hands-On Assignment for Today

1. Go to your **Day 4 EC2 instance → Security tab → Security Groups**.
2. **Review the inbound rules** — confirm SSH is still restricted to "My IP," not 0.0.0.0/0.
3. **Add a new inbound rule** allowing custom TCP port `5000` from `0.0.0.0/0` (you'll need this tomorrow when you run an Express app on this port).
4. Go to **EC2 → Elastic IPs → Allocate Elastic IP address**.
5. **Associate it with your Day 4 instance.**
6. **Stop and start your instance** — confirm the public IP stays exactly the same now (compare against what it was before).
7. **Important:** If you're done practicing for the day, either keep the Elastic IP attached to a running instance, or release it entirely — don't leave it reserved-but-unattached, since that gets billed.
8. Write a short note in your own words covering:
   - What inbound vs outbound rules control
   - Why Elastic IP is useful, and the one cost trap to avoid
   - Why key-pair login is more secure than password login

---

## 7. Day 5 Goal — Can You Teach This?

By the end of today, you should be able to explain, without notes:

> *"Security Groups act as a stateful firewall around my EC2 instance, controlling exactly which ports and sources can reach it — public services stay open, admin access like SSH stays restricted to my IP. Elastic IP gives my instance a permanent public address so it doesn't change every time I restart it — but I have to remember it's only free while actually attached to a running instance."*

---

**Next up — Day 6: Install Node.js on EC2, Run a Simple Express App**
