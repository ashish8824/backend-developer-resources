# Day 8: S3 Basics — Buckets & Objects
### *(Moving Beyond the Server — Storing Files the AWS Way)*

> **Roadmap reference:** Week 2, Day 8 — "S3 Basics, Buckets, Objects"

---

## Why This Matters

Week 1 was all about compute (EC2) — a server running your code. **Week 2 starts with storage** — and S3 is, without exaggeration, one of the most widely used services in all of AWS. Profile pictures, uploaded documents, app backups, static website files, ML training data — an enormous share of it sits in S3.

If your EC2 instance died right now, you'd lose anything stored only on its local disk. S3 exists precisely to solve that — durable, independent storage that lives outside any single server.

---

## 1. What is S3?

**S3 (Simple Storage Service) is AWS's object storage service — designed to store and retrieve any amount of data, from anywhere, with extremely high durability.**

```
EC2 Instance Storage (EBS)        S3
─────────────────────────         ─────────────────
Tied to one specific instance  vs  Independent of any server
Disappears if instance is        Data persists regardless of
  terminated (by default)          what's running or not
Good for: OS, app code,           Good for: file uploads, backups,
  databases                         images, videos, static assets
```

### "Object Storage" — What Does That Actually Mean?

This is a key distinction interviewers love to probe:

```
FILE STORAGE (traditional)        OBJECT STORAGE (S3)
───────────────────────────       ──────────────────────
Organized in folders/directories  Flat structure — everything is
  with a real file system           an "object" with a unique key
Good for: OS files, structured    Good for: unstructured data —
  file hierarchies                  images, videos, backups, logs
```

S3 doesn't actually have "real" folders — what looks like a folder (e.g., `uploads/profile-pics/`) is really just part of the object's **key** (its full name/path string). AWS's console *displays* it folder-like for convenience, but underneath, it's flat.

---

## 2. Core S3 Concepts: Buckets & Objects

```
        AWS Account
             │
             ▼
        S3 BUCKET                ← a container (like a top-level folder)
   "pulsebloom-uploads"
             │
        ┌────┴─────┬──────────────┐
        ▼            ▼              ▼
     OBJECT        OBJECT         OBJECT
  profile.jpg    report.pdf    backup-2026.zip
```

### Bucket
**A Bucket is a top-level container in S3 that holds objects. Think of it as your storage "namespace."**

```
Bucket name rules (commonly tested in quizzes/interviews):
  - Must be globally unique across ALL of AWS, not just your account
  - Lowercase letters, numbers, hyphens only
  - 3–63 characters
  - No uppercase, no underscores
```

> **Why "globally unique"?** Because every bucket gets a public-style URL like `https://pulsebloom-uploads.s3.amazonaws.com/`. If two AWS customers could name their bucket the same thing, that URL would be ambiguous. This is genuinely different from EC2 instance names, which only need to be unique within your own account.

### Object
**An Object is a single file stored in a bucket, along with its metadata.**

```
An object consists of:
  Key          → the full "path/filename" string (e.g., "uploads/avatar.jpg")
  Value/Data   → the actual file content (bytes)
  Metadata     → info about the object (content-type, size, last modified)
  Version ID   → if versioning is enabled (you'll learn this on Day 11)
```

```
Example object:
  Key:   uploads/users/42/avatar.jpg
  Size:  245 KB
  Type:  image/jpeg
```

> Notice that `uploads/users/42/` *looks* like a folder path — but it's really just text inside the Key. There's no actual nested directory structure underneath; S3 is fundamentally flat.

---

## 3. Creating Your First Bucket (Hands-On Walkthrough)

```
AWS Console → S3 → "Create bucket"

  Bucket name: your-name-first-bucket-2026
    (must be globally unique — try variations if taken)

  Region: ap-south-1 (Mumbai)
    — keep it consistent with where your EC2 instance lives,
      to avoid unnecessary cross-region data transfer

  Object Ownership: Bucket owner enforced (default, recommended)

  Block Public Access: Leave ALL boxes checked (blocked) for now
    — you'll deliberately learn how to safely open specific
      access on Day 9, not by accident today

  Versioning: Disable for now (covered properly on Day 11)

  Encryption: Leave default (SSE-S3) enabled — it's free and
    automatic, no reason to skip it

  → Click "Create bucket"
```

### Uploading Your First Object

```
Open your new bucket → "Upload" → "Add files"
  → Select any file from your computer
  → Click "Upload"
```

```
After upload, you'll see:
  Key:          your-file.jpg
  Size:         (file size)
  Type:         (auto-detected, e.g., image/jpeg)
  Last modified: (timestamp)
```

Congratulations — that file now lives in AWS's object storage, completely independent of any EC2 instance.

---

## 4. Storage Classes — Not All S3 Storage Is Priced the Same

A frequently asked interview topic: **S3 offers multiple storage classes, trading cost against retrieval speed/frequency.**

```
STANDARD             → Frequently accessed data, most common default
                        (e.g., active user profile pictures)

STANDARD-IA           → Infrequently Accessed — cheaper storage,
  (Infrequent Access)    small retrieval fee (e.g., monthly reports)

GLACIER                → Very cheap, but slow retrieval (minutes-hours)
                        (e.g., compliance archives, old backups)

GLACIER DEEP ARCHIVE   → Cheapest of all, retrieval takes 12+ hours
                        (e.g., 7-year-old data you must legally retain
                          but almost never access)
```

```
Decision logic:
  Will I access this file often?      → STANDARD
  Will I access this file rarely?      → STANDARD-IA
  Do I just need to legally retain it,
    and almost never read it back?     → GLACIER / DEEP ARCHIVE
```

You'll go deeper into automating these transitions on **Day 12 (S3 Lifecycle Policies)** — for now, just know they exist and why.

---

## 5. Real Use Cases for S3 in Backend Development

```
1. User-uploaded content
   PulseBloom mood-entry photos, QueueCare patient documents
   → stored as S3 objects, referenced by their key/URL in your
     PostgreSQL database (the DB stores a pointer, not the file itself)

2. Application backups
   Database dumps, config snapshots → stored in S3, often with
   STANDARD-IA or Glacier for cost efficiency

3. Static website/asset hosting
   Frontend build files, images, CSS → S3 can even serve a
   static website directly (no server needed at all)

4. Logs and data lakes
   Application logs, CloudWatch log exports, data for analytics
```

### The Pattern You'll Use Constantly

```
Database (RDS/PostgreSQL)              S3 Bucket
─────────────────────────              ──────────
Stores: structured data            Stores: the actual file
  user_id, file_url, uploaded_at         (image, PDF, video)

  file_url column value:
  "https://pulsebloom-uploads.s3.amazonaws.com/uploads/users/42/avatar.jpg"
```

> This is a critically important pattern: **never store large binary files directly inside your relational database.** Store the file in S3, and store only the *reference* (the S3 key or URL) in PostgreSQL/MySQL. You'll implement this exact flow hands-on on **Day 10**.

---

## 6. Interview Questions (Practice Explaining These Out Loud)

**Q1: What is S3?**
> AWS's object storage service, designed to store and retrieve any amount of unstructured data (files) with high durability, independent of any single server.

**Q2: What's the difference between a Bucket and an Object?**
> A Bucket is the top-level container — like a namespace — that holds your data. An Object is an individual file stored inside that bucket, along with its key (path/name) and metadata.

**Q3: Why must S3 bucket names be globally unique, unlike EC2 instance names?**
> Because each bucket maps to a public-style URL (e.g., `bucket-name.s3.amazonaws.com`) that must be unambiguous across all of AWS, not just within one account.

**Q4: Does S3 actually have real folders?**
> No — S3 is fundamentally a flat object store. What appears as folder structure (e.g., `uploads/avatars/`) is really just part of the object's key (its full name string), displayed in a folder-like way by the console for convenience.

**Q5: Why shouldn't you store large files (images, videos) directly in a relational database?**
> It bloats the database, slows down queries/backups, and is far more expensive per GB than S3. The standard pattern is to store the actual file in S3 and keep only a reference (URL/key) in the database.

**Q6: Name the main S3 storage classes and when you'd use each.**
> Standard for frequently accessed data, Standard-IA for infrequently accessed data, and Glacier/Glacier Deep Archive for long-term archival where retrieval speed doesn't matter and cost minimization does.

---

## 7. Hands-On Assignment for Today

1. Create a new S3 bucket in **`ap-south-1`**, following the exact settings in Section 3 (Block Public Access left ON for now).
2. Upload **2–3 files** of different types (an image, a PDF, a text file).
3. Click into one uploaded object and explore its **Properties tab** — note the Key, Size, and Storage Class shown.
4. Try uploading a file inside a "folder" (e.g., type `test-folder/myfile.txt` as the key during upload) — notice how the console displays it, and think about why it's actually still flat underneath.
5. Write a short note in your own words covering:
   - The difference between a Bucket and an Object
   - Why S3 bucket names must be globally unique
   - One real example of when you'd use Standard vs Glacier storage class

---

## 8. Day 8 Goal — Can You Teach This?

By the end of today, you should be able to explain, without notes:

> *"S3 is AWS's object storage service — buckets are containers, objects are the individual files inside them, identified by a unique key. Unlike EC2's local storage, S3 is independent of any server and built for durability. I'd use it to store user-uploaded files, keeping only a reference URL in my actual database, and I'd choose a storage class — Standard, Standard-IA, or Glacier — based on how often that data actually needs to be accessed."*

---

**Next up — Day 9: S3 Permissions & Bucket Policies**
