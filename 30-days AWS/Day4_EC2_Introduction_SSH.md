# Day 4: EC2 Introduction — Launch an Ubuntu Instance & Connect via SSH
### *(Your First Real AWS Server — Where Theory Becomes Practice)*

> **Roadmap reference:** Week 1, Day 4 — "EC2 Introduction, Launch Ubuntu Instance, Connect using SSH"

---

## Why This Matters

Days 1–3 were all conceptual. **Today you launch your first real, running computer in the cloud.** This is the moment AWS stops being an abstract idea and becomes a server you can SSH into and run code on — exactly like the ones powering real production apps.

EC2 is also the single most commonly used AWS service in junior backend interviews, because it's the most direct equivalent to "a server," which every backend developer already understands.

---

## 1. What is EC2?

**EC2 (Elastic Compute Cloud) is AWS's service for renting virtual servers — called "instances" — that you can configure, install software on, and run applications from, just like a physical computer.**

```
Traditional Server               EC2 Instance
─────────────────                ─────────────
Buy physical hardware       →    Click "Launch Instance"
Install OS yourself          →    Choose an AMI (pre-built OS image)
Wait days/weeks for setup    →    Server ready in ~2 minutes
Pay for the hardware forever →    Pay only while it's running
```

The word "Elastic" matters: you can resize, stop, start, or scale these servers on demand — something a physical machine in your office could never do.

---

## 2. Key EC2 Concepts You Need Before Launching

### Instance — The Virtual Server Itself
A single running virtual machine. Each one has its own CPU, memory, storage, and network interface.

### AMI (Amazon Machine Image) — The OS Template
**An AMI is a pre-configured template containing the operating system (and sometimes pre-installed software) that your instance boots from.**

```
AMI = "the blueprint"
Instance = "the actual house built from that blueprint"
```

> Today you'll use the **Ubuntu Server AMI** — a clean Linux environment, perfect for running Node.js.

### Instance Type — The Hardware Size
Defines how much CPU, memory, and network performance your instance gets.

```
t2.micro / t3.micro  → 1 vCPU, 1 GB RAM   → Free Tier eligible, fine for learning
t3.medium             → 2 vCPU, 4 GB RAM   → Small production workloads
m5.large               → 2 vCPU, 8 GB RAM   → Real production backend traffic
```

> For today's hands-on, you'll use **t2.micro or t3.micro** — it's covered under the 12-Months Free Tier you set up budget alerts for on Day 2.

---

## 3. The EC2 Launch Process — Step by Step

```
1. Choose an AMI            → Ubuntu Server (latest LTS)
2. Choose an Instance Type  → t2.micro / t3.micro
3. Create/select a Key Pair → for SSH login (covered below)
4. Configure Network        → which VPC/subnet it lives in (Day 15 goes deeper)
5. Configure Security Group → which ports are open (Day 5 goes deeper)
6. Configure Storage        → EBS volume size (default is fine for now)
7. Launch                   → instance boots in ~1-2 minutes
```

Don't worry if Security Groups and VPC feel unfamiliar right now — today you'll use sensible defaults, and you'll go deep on both in upcoming days. The goal today is just: **get a server running and connect to it.**

---

## 4. What is a Key Pair, and Why Do You Need One?

**A Key Pair is how you securely log into your EC2 instance — instead of a traditional username/password.**

```
Public Key  → Stored by AWS, embedded into your instance automatically
Private Key → Downloaded ONLY by you, as a .pem file
              (AWS never stores a copy — if you lose it, it's gone)
```

```
Your Computer (.pem private key)
        │
        │   SSH connection attempt
        ▼
EC2 Instance (has matching public key)
        │
        │   Keys match → access granted
        ▼
   You're logged in
```

> **Critical rule:** Treat your `.pem` file like a password. Never commit it to GitHub, never share it. If it leaks, anyone can SSH into your server.

---

## 5. Launching Your Instance (Hands-On Walkthrough)

```
AWS Console → EC2 → "Launch Instance"

  Name: my-first-server

  AMI: Ubuntu Server 22.04 LTS (or latest LTS available)

  Instance Type: t2.micro (Free Tier eligible)

  Key Pair: 
    → Create new key pair
    → Name it (e.g., "my-first-server-key")
    → Type: RSA, Format: .pem (for Mac/Linux) or .ppk (for PuTTY on Windows)
    → DOWNLOAD IT NOW — you cannot download it again later

  Network Settings:
    → Leave default VPC for now
    → Allow SSH traffic from "My IP" (NOT 0.0.0.0/0 — recall Day 4's lesson
      on the dangers of open SSH to the entire internet)

  Storage: 
    → Default 8 GB gp3 is fine for learning

  → Click "Launch Instance"
```

Within about a minute, your instance status will show **"Running"** — congratulations, that's a real virtual server, live on the internet.

---

## 6. Connecting via SSH

**SSH (Secure Shell) is a protocol that lets you securely log into and control a remote computer from your terminal.**

### On Mac/Linux Terminal

```bash
# First, lock down permissions on your key file (required by SSH)
chmod 400 my-first-server-key.pem

# Connect to your instance
ssh -i my-first-server-key.pem ubuntu@<your-instance-public-ip>
```

### On Windows
Use **PuTTY** with the `.ppk` key, or use **Windows Terminal/PowerShell** with the same `ssh -i` command if you have OpenSSH installed (most modern Windows versions do).

### Breaking Down the SSH Command

```
ssh -i my-first-server-key.pem  ubuntu@13.234.XX.XX
     │       │                    │        │
     │       │                    │        └── Your instance's Public IP
     │       │                    └── Default username for Ubuntu AMIs
     │       └── Path to your downloaded private key
     └── The SSH command itself
```

> **Note:** The default SSH username depends on the AMI. Ubuntu AMIs use `ubuntu`. Amazon Linux AMIs use `ec2-user`. This trips up a lot of beginners — always check the AMI's documentation.

### What Success Looks Like

```
$ ssh -i my-first-server-key.pem ubuntu@13.234.XX.XX

Welcome to Ubuntu 22.04 LTS

ubuntu@ip-172-31-XX-XX:~$
```

That prompt means you're now operating *inside* your cloud server, exactly as if you'd plugged a monitor and keyboard into a physical machine sitting in a data center in Mumbai.

---

## 7. Where Your EC2 Instance Lives (Connecting the Dots from Earlier Days)

```
Region: ap-south-1 (Mumbai)         ← Day 1
   └── Availability Zone: ap-south-1a
         └── Your EC2 Instance
               ├── Secured by: IAM (if a Role is attached) ← Day 3
               ├── Secured by: Security Group rules        ← Day 5
               └── Billed per running hour                  ← Day 2
```

Every concept from the last three days is now physically grounded in this one running server.

---

## 8. Common Beginner Mistakes (Avoid These Today)

```
Mistake 1: Losing the .pem file
  → There's no "forgot password" option. If lost, you must create
    a new key pair and reattach it via a more advanced recovery process.

Mistake 2: Wrong SSH username
  → ec2-user vs ubuntu vs admin — depends entirely on the AMI used.

Mistake 3: Security Group blocks your IP
  → If your home IP changes (common with mobile/dynamic IPs), SSH
    will suddenly stop connecting — you'll need to update the rule.

Mistake 4: Forgetting to STOP the instance after practicing
  → Remember Day 2 — an idle running instance still burns Free Tier
    hours and, eventually, real money.
```

---

## 9. Interview Questions (Practice Explaining These Out Loud)

**Q1: What is EC2?**
> AWS's service for renting resizable virtual servers (instances) in the cloud, where you choose the OS, hardware size, and configuration.

**Q2: What is an AMI?**
> Amazon Machine Image — a template containing the operating system (and optionally pre-installed software) that an EC2 instance boots from.

**Q3: What's the difference between an Instance and an Instance Type?**
> An Instance is the actual running virtual server. An Instance Type defines its hardware specs — CPU, memory, and network performance (e.g., t2.micro vs m5.large).

**Q4: What is a Key Pair used for?**
> Secure SSH authentication into an EC2 instance — a public key is embedded in the instance, and only the matching private key (downloaded once, by you) can unlock access.

**Q5: Why shouldn't you allow SSH access from 0.0.0.0/0?**
> It opens port 22 to the entire internet, making the instance an easy target for automated brute-force login attempts. Best practice is to restrict SSH to your specific IP address.

**Q6: What happens if you lose your .pem private key file?**
> AWS never stores a copy, so you cannot directly recover it. You'd need to create a new key pair and use a recovery process (e.g., attaching the new key via an EC2 Instance Connect session or volume detach/reattach trick) to regain access.

---

## 10. Hands-On Assignment for Today

1. **Launch an EC2 instance** following the exact steps in Section 5 — Ubuntu Server AMI, t2.micro, new key pair, SSH allowed only from "My IP."
2. **Download and secure your `.pem` key** — move it somewhere safe, never inside a Git repo.
3. **SSH into your instance** using the command from Section 6.
4. Once connected, run a couple of basic Linux commands to confirm you're really in:
   ```bash
   whoami
   pwd
   lsb_release -a   # confirms Ubuntu version
   ```
5. **Stop (don't terminate) your instance** when you're done for the day, to avoid burning unnecessary Free Tier hours.
6. Write a short note in your own words covering:
   - What an AMI is, in your own words
   - Why SSH should be restricted to your IP, not the whole internet
   - The difference between stopping and terminating an instance

---

## 11. Day 4 Goal — Can You Teach This?

By the end of today, you should be able to explain, without notes:

> *"EC2 lets me rent a virtual server in the cloud. I choose an AMI as the OS template, an instance type for hardware size, and a key pair for secure SSH access. I launch it, connect via SSH using my private key, and remember to stop it when I'm not actively using it to avoid unnecessary cost."*

---

**Next up — Day 5: Security Groups, Key Pairs (deeper dive), and Elastic IP**
