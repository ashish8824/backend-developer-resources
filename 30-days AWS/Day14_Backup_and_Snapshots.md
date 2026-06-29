# Day 14: Backup and Snapshots
### *(Your Last Line of Defense — Before the Mini Project)*

> **Roadmap reference:** Week 2, Day 14 — "Backup and Snapshots" | **Mini Project:** File Upload API using S3 + PostgreSQL RDS

---

## Why This Matters

You now have a real database (RDS) and a real server (EC2) running your application. But consider this scenario:

```
9:00 AM  → Developer runs a migration script on the RDS database
9:01 AM  → Script has a bug — it drops the wrong table
9:01 AM  → Weeks of user data is gone
9:02 AM  → Panic
        ↓
With no backups:   "The data is just... gone."
With backups:      "Restore to 8:59 AM, crisis averted."
```

Backups are the difference between a bad morning and a catastrophe. Today you learn AWS's two backup mechanisms — for both EC2 and RDS — so you're never in that second scenario.

---

## 1. Two Types of Backups in AWS

```
AWS backup mechanisms:
        │
        ├── SNAPSHOTS
        │     → Manual, point-in-time copy you take yourself
        │     → Like taking a photograph of your data right now
        │     → Stored in S3 (managed by AWS, you never see the bucket)
        │     → Persists even after you delete the original instance
        │
        └── AUTOMATED BACKUPS (RDS only)
              → AWS takes these automatically on a schedule
              → Enables Point-in-Time Recovery (PITR)
              → Kept for a configurable retention window (1–35 days)
              → Deleted automatically when the RDS instance is deleted
                (unless you configure otherwise)
```

---

## 2. EC2 Snapshots — Backing Up Your Server

**An EC2 Snapshot is a point-in-time copy of an EBS (Elastic Block Store) volume — the disk attached to your EC2 instance.**

```
EC2 Instance
      │
      └── EBS Volume (your server's disk — OS, app code, Node.js, etc.)
                │
                │  "Take Snapshot"
                ▼
           EBS Snapshot
           (stored in S3, managed by AWS)
           → Can be used to:
               1. Restore the volume to this exact state later
               2. Create an AMI from it (launch new instances
                  from this exact server state)
               3. Copy to another Region for disaster recovery
```

### When to Snapshot Your EC2 Instance

```
Before any risky operation, such as:
  - Major OS/package upgrades
  - Installing new software that might conflict
  - Changing critical config files
  - Before running a new deployment for the first time

Rule of thumb:
  "If it would take more than 30 minutes to rebuild if it
   broke, snapshot it first."
```

### Taking an EC2 Snapshot (Console)

```
EC2 → Instances → select your instance
  → Actions → Monitor and troubleshoot → Create snapshot of root volume

  OR

EC2 → Volumes → select your EBS volume
  → Actions → Create snapshot
  → Description: "Before Node.js upgrade - June 2026"
  → Create snapshot
```

### Restoring from an EC2 Snapshot

```
EC2 → Snapshots → select your snapshot
  → Actions → Create volume from snapshot
  → Attach the new volume to a running instance
  → (More common: Actions → Create image from snapshot
     → Creates an AMI you can launch a fresh instance from)
```

---

## 3. RDS Snapshots — Backing Up Your Database

**An RDS Snapshot is a complete point-in-time copy of your entire RDS database instance.**

```
RDS Snapshot types:

  MANUAL SNAPSHOT
    → You trigger it yourself, any time
    → Persists INDEFINITELY until you delete it manually
    → Not deleted when the RDS instance is deleted

  AUTOMATED SNAPSHOT
    → AWS creates one every day during the backup window
    → Kept for your configured retention period (1–35 days)
    → Automatically deleted when retention period expires
    → The backbone of Point-in-Time Recovery (PITR)
```

### Taking a Manual RDS Snapshot

```
RDS → Databases → select your instance
  → Actions → Take snapshot
  → Snapshot name: "before-schema-migration-june-2026"
  → Take snapshot
```

> **Best practice:** Always take a manual snapshot before running any significant migration against a production database. Automated backups are for day-to-day safety; manual snapshots are your deliberate "save point" before something potentially risky.

### Restoring from an RDS Snapshot

```
RDS → Snapshots → select snapshot → Restore snapshot
  → This creates a BRAND NEW RDS instance from that snapshot
  → The original instance is untouched
  → You then update your application's DB_HOST to point
    to the restored instance
```

```
Important nuance:
  You can't "restore in place" — RDS always creates a new
  instance from a snapshot. This is actually safer — the
  original stays available while you verify the restored
  data before switching traffic to it.
```

---

## 4. Point-in-Time Recovery (PITR) — The Most Powerful RDS Feature

This was introduced on Day 11 — today you understand the mechanics behind it.

```
How PITR works under the hood:

  Daily automated snapshot (e.g., taken at 3:00 AM)
          │
          │  PLUS
          ▼
  Continuous transaction logs (written every 5 minutes)
          │
          │  COMBINED
          ▼
  Ability to reconstruct your database at ANY second
  within your retention window

Example:
  3:00 AM  → Automated snapshot taken
  11:47 AM → Bad data migration runs, corrupts critical table
  11:50 AM → You detect the problem
          ↓
  PITR: "Restore to 11:46:59 AM"
  → AWS replays the daily snapshot + transaction logs
    up to exactly that second
  → New RDS instance created at that precise state
  → Switch your app's connection string to it
  → Crisis averted, less than 1 minute of data lost
```

### Enabling/Configuring PITR

```
RDS → Databases → select instance → Modify
  → Backup section:
      Backup retention period: 7 days (minimum for PITR)
      Backup window: choose a low-traffic time (e.g., 3:00 AM–4:00 AM)
  → Apply immediately
```

> You enabled 7-day retention during creation on Day 12. That means you can recover to any second in the last 7 days — right now, that's already protecting you.

---

## 5. RDS Automated Backups vs Manual Snapshots — Side by Side

| | Automated Backups | Manual Snapshots |
|---|---|---|
| **Triggered by** | AWS, on a schedule | You, manually |
| **Retention** | 1–35 days (auto-deleted after) | Forever (until you delete) |
| **Enables PITR** | Yes | No (only restores to snapshot moment) |
| **Deleted with instance** | Yes (unless retained) | No — persists independently |
| **Best for** | Day-to-day protection, PITR | Before risky operations, long-term archiving |

---

## 6. EC2 Snapshots vs AMIs — What's the Difference?

This confuses many beginners:

```
EBS SNAPSHOT
  → A backup of a DISK (volume)
  → Low-level — just raw data
  → Can be used to restore a volume

AMI (Amazon Machine Image)
  → A complete template for launching a NEW INSTANCE
  → Built from one or more snapshots (plus launch config)
  → Think of it as: snapshot + metadata + launch instructions

Relationship:
  Snapshot  ──(create image from snapshot)──►  AMI
                                                 │
                                                 ▼
                                           Launch new EC2 instance
                                           (identical to original)
```

> **Use case:** If you've perfectly configured your EC2 — Node.js installed, Nginx configured, PM2 set up, all security hardened — creating an AMI from it lets you launch identical copies instantly. This becomes very important later when you learn Auto Scaling (Day 19), which needs an AMI to launch new instances automatically.

---

## 7. Backup Strategy for a Real App (PulseBloom / QueueCare)

```
COMPONENT        BACKUP APPROACH
─────────────    ──────────────────────────────────────────
EC2 Instance     Snapshot before major changes/deployments
                 Create an AMI once your server is fully
                 configured, for auto-scaling later

RDS Database     7-day automated backup retention (minimum)
                 Manual snapshot before every migration
                 PITR enabled always

S3 Files         S3 itself has 11 nines (99.999999999%)
                 durability — AWS replicates your objects
                 across multiple AZs automatically.
                 For extra safety: enable S3 Versioning (Day 11)
                 to recover from accidental overwrites/deletes
```

---

## 8. Cost of Snapshots

```
EC2 Snapshots (EBS):
  First snapshot: full copy of volume (charged per GB)
  Subsequent snapshots: INCREMENTAL — only changed blocks
  → This means: if you have a 20 GB volume but only 1 GB changed
    since the last snapshot, you only pay for 1 GB

RDS Snapshots:
  Free storage up to the size of your database (per month)
  → A 20 GB RDS database gets 20 GB of backup storage free
  → Beyond that, standard S3 rates apply
  → Automated backups within the retention window are
    included in that free storage allocation
```

> **Practical implication:** Snapshots are cheap — almost free for the scale you're working at right now. There's no good reason to skip them.

---

## 9. Interview Questions (Practice Explaining These Out Loud)

**Q1: What's the difference between an automated backup and a manual snapshot in RDS?**
> Automated backups are taken daily by AWS and kept for a configured retention period — they enable Point-in-Time Recovery. Manual snapshots are taken deliberately by the user, persist indefinitely until deleted, and don't enable PITR — but they survive instance deletion and are best used before risky operations.

**Q2: What is Point-in-Time Recovery and how does it work?**
> PITR lets you restore an RDS database to any specific second within the retention window, by combining a daily automated snapshot with continuously captured transaction logs. It creates a new RDS instance at that precise state — the original remains untouched.

**Q3: What's the difference between an EC2 Snapshot and an AMI?**
> A snapshot is a backup of a disk (EBS volume) — raw data. An AMI is a complete template for launching a new EC2 instance, built from one or more snapshots plus metadata and launch configuration. You'd use a snapshot to restore a disk, and an AMI to replicate a fully configured server.

**Q4: Does restoring an RDS snapshot overwrite the existing database?**
> No — restoring always creates a brand new RDS instance from the snapshot. The original instance remains untouched, which lets you verify the restored data before switching application traffic to it.

**Q5: Are EC2 snapshots incremental?**
> Yes — after the first full snapshot, subsequent snapshots only capture blocks that have changed, keeping storage costs low even with frequent snapshots.

---

## 10. Hands-On Assignment for Today

1. **Take a manual EC2 snapshot:**
   EC2 → Volumes → select your volume → Create snapshot → describe it clearly.

2. **Take a manual RDS snapshot:**
   RDS → your instance → Actions → Take snapshot → name it `"manual-backup-week2"`.

3. **Verify PITR is enabled:**
   RDS → your instance → Maintenance & backups tab → confirm backup retention is 7+ days.

4. **Create an AMI from your EC2 instance:**
   EC2 → Instances → Actions → Image and templates → Create image.
   Name it `"node-api-configured-v1"`. This is your server blueprint for later.

5. **Cost check:** Go to S3 console — you won't see the backup bucket directly (AWS manages it), but check RDS → Snapshots to confirm your manual and automated snapshots are listed.

6. Write a short note covering:
   - When you'd use PITR vs restoring from a manual snapshot
   - Why restoring creates a new RDS instance instead of overwriting
   - What you'd do immediately before running a risky database migration

---

## 11. Week 2 Mini Project — File Upload API (S3 + PostgreSQL RDS)

You now have everything needed to build this week's portfolio project. Here's the complete picture:

```
POST /upload
  → Multer receives file (Day 10)
  → Upload to S3, get back a URL (Day 10)
  → Save file metadata + URL to RDS PostgreSQL (Day 13)
  → Return the full record to the client

GET /files
  → Query RDS for all uploaded file records
  → Return list with S3 URLs
  → Client can use the URL to fetch the actual file from S3
```

```sql
-- Schema for the mini project
CREATE TABLE file_uploads (
  id          SERIAL PRIMARY KEY,
  filename    VARCHAR(255) NOT NULL,
  s3_key      VARCHAR(500) NOT NULL,
  s3_url      TEXT NOT NULL,
  file_size   INTEGER,
  mime_type   VARCHAR(100),
  uploaded_at TIMESTAMP DEFAULT NOW()
);
```

```javascript
// Mini project endpoint tying it all together
app.post('/upload', upload.single('file'), async (req, res) => {
  // 1. Upload to S3 (Day 10 code)
  const fileKey = `uploads/${Date.now()}-${req.file.originalname}`;
  await s3Client.send(new PutObjectCommand({
    Bucket: process.env.S3_BUCKET_NAME,
    Key: fileKey,
    Body: req.file.buffer,
    ContentType: req.file.mimetype,
  }));

  const fileUrl = `https://${process.env.S3_BUCKET_NAME}.s3.${process.env.AWS_REGION}.amazonaws.com/${fileKey}`;

  // 2. Save metadata to RDS (Day 13 code)
  const result = await pool.query(
    `INSERT INTO file_uploads
       (filename, s3_key, s3_url, file_size, mime_type)
     VALUES ($1, $2, $3, $4, $5)
     RETURNING *`,
    [req.file.originalname, fileKey, fileUrl,
     req.file.size, req.file.mimetype]
  );

  res.status(201).json(result.rows[0]);
});
```

> This single endpoint connects EC2 (running Express), S3 (file storage), and RDS (metadata) — Portfolio Project 2 is essentially this, deployed and working end-to-end.

---

## 12. Day 14 Goal — Can You Teach This?

By the end of today, you should be able to explain, without notes:

> *"AWS gives me two backup mechanisms — automated backups for day-to-day protection and Point-in-Time Recovery, and manual snapshots for deliberate save points before risky operations. Snapshots always create new instances on restore, never overwrite existing ones. For EC2, I can create an AMI from a snapshot to replicate my fully-configured server later. Combined with the S3 + RDS integration from this week, I can now protect every layer of my application's data."*

---

## Week 2 Complete! 🎉

You've covered S3 (Days 8–10) and RDS (Days 11–14) — plus backups tying it all together. Portfolio Project 2 (File Upload API using S3 + PostgreSQL RDS) is fully buildable from today's code.

---

**Next up — Week 3 begins: Day 15: VPC Basics & Subnets**
