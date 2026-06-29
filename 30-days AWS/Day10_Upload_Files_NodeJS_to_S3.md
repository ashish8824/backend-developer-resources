# Day 10: Upload Files from Node.js to S3
### *(Connecting Your Express App to Real Cloud Storage)*

> **Roadmap reference:** Week 2, Day 10 — "Upload files from Node.js to S3"

---

## Why This Matters

The last two days were AWS Console work. Today, you write **actual application code** that talks to S3 — this is the exact pattern you'd use in a real product to let users upload profile pictures, documents, or any other file. By the end of today, you'll have a working Express endpoint: client uploads a file → your code sends it to S3 → you get back a URL to store in your database.

---

## 1. The Full Flow You're Building Today

```
Client (Postman/Browser)
        │  POST /upload  (multipart/form-data, with a file attached)
        ▼
Express App
        │  receives file via Multer middleware
        ▼
AWS SDK (S3 Client)
        │  uploads file buffer to S3
        ▼
S3 Bucket
        │  stores the object, returns success
        ▼
Express App
        │  responds with the file's S3 URL/key
        ▼
Client receives the URL
        (in a real app, you'd now save this URL in your
         PostgreSQL/RDS database — covered properly Week 2,
         once RDS is set up)
```

---

## 2. Step 1: Set Up IAM Credentials the Right Way

**Do not hardcode your AWS Access Key and Secret Key directly in your code.** Recall Day 3's lesson on Roles vs hardcoded keys — today you'll use the safer approach appropriate for local development.

```
For LOCAL development (today, on your own machine):
  → Create a dedicated IAM User with S3-only permissions
  → Store its keys in a .env file (never committed to Git)

For PRODUCTION (later, once deployed to EC2/ECS):
  → Use an IAM Role attached to the instance/task instead
  → No keys stored anywhere — AWS handles it automatically
```

### Create a Scoped IAM User for This Project

```
AWS Console → IAM → Users → Create User
  Name: my-app-s3-uploader
  Attach policy: AmazonS3FullAccess
    (for learning purposes — in production you'd write a custom
     policy scoped to just YOUR bucket, following Day 3's
     Principle of Least Privilege)

  → Generate Access Key → choose "Application running outside AWS"
  → Copy the Access Key ID and Secret Access Key
    (shown ONLY ONCE — save them now)
```

### Store Credentials Safely

```bash
# In your project root
nano .env
```

```env
AWS_ACCESS_KEY_ID=AKIA...........
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI...........
AWS_REGION=ap-south-1
S3_BUCKET_NAME=your-bucket-name-from-day-8
```

```bash
# CRITICAL — make sure .env is never committed to Git
echo ".env" >> .gitignore
```

> **This single `.gitignore` line prevents the #1 most common way student/junior-dev AWS keys leak — accidentally pushing `.env` to a public GitHub repo.**

---

## 3. Step 2: Install Required Packages

```bash
npm install @aws-sdk/client-s3 multer dotenv
```

```
@aws-sdk/client-s3  → AWS's official SDK (v3) for talking to S3
multer               → Express middleware for handling file uploads
                        (multipart/form-data parsing)
dotenv                → loads your .env file into process.env
```

> **Note:** AWS SDK v3 (modular, what we're using) replaced the older AWS SDK v2. v3 is more efficient — you only import the specific service clients you need (here, just S3) instead of one giant package.

---

## 4. Step 3: Write the Upload Code

```bash
nano index.js
```

```javascript
require('dotenv').config();
const express = require('express');
const multer = require('multer');
const { S3Client, PutObjectCommand } = require('@aws-sdk/client-s3');

const app = express();
const PORT = 5000;

// Configure the S3 client
const s3Client = new S3Client({
  region: process.env.AWS_REGION,
  credentials: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY,
  },
});

// Multer stores the uploaded file in memory temporarily,
// as a buffer, before we forward it to S3
const upload = multer({ storage: multer.memoryStorage() });

app.post('/upload', upload.single('file'), async (req, res) => {
  try {
    if (!req.file) {
      return res.status(400).json({ error: 'No file uploaded' });
    }

    const fileKey = `uploads/${Date.now()}-${req.file.originalname}`;

    const command = new PutObjectCommand({
      Bucket: process.env.S3_BUCKET_NAME,
      Key: fileKey,
      Body: req.file.buffer,
      ContentType: req.file.mimetype,
    });

    await s3Client.send(command);

    const fileUrl = `https://${process.env.S3_BUCKET_NAME}.s3.${process.env.AWS_REGION}.amazonaws.com/${fileKey}`;

    res.status(200).json({
      message: 'File uploaded successfully',
      key: fileKey,
      url: fileUrl,
    });

  } catch (error) {
    console.error('Upload error:', error);
    res.status(500).json({ error: 'Failed to upload file' });
  }
});

app.listen(PORT, '0.0.0.0', () => {
  console.log(`Server running on port ${PORT}`);
});
```

### Breaking Down the Important Parts

```
multer.memoryStorage()
  → keeps the uploaded file as a Buffer in RAM, rather than
    writing it to disk first — efficient for forwarding
    straight to S3 without an intermediate file

fileKey = `uploads/${Date.now()}-${originalname}`
  → prepends a timestamp to avoid filename collisions if two
    users upload files with the same name (e.g., "photo.jpg")

PutObjectCommand
  → the SDK v3 way of constructing an S3 "upload this object"
    request — Bucket, Key, Body, and ContentType are the
    essential fields

ContentType: req.file.mimetype
  → tells S3 (and later, browsers) what kind of file this is,
    so it displays/downloads correctly instead of as raw bytes
```

---

## 5. Step 4: Test It

### Using Postman or curl

```bash
curl -X POST http://localhost:5000/upload \
  -F "file=@/path/to/your/image.jpg"
```

### Expected Response

```json
{
  "message": "File uploaded successfully",
  "key": "uploads/1719500000000-image.jpg",
  "url": "https://your-bucket-name.s3.ap-south-1.amazonaws.com/uploads/1719500000000-image.jpg"
}
```

### Verify in the AWS Console

```
S3 → your bucket → uploads/ folder
  → your file should now appear, uploaded directly from your code
```

> If the URL doesn't load in a browser, that's expected — recall Day 9. Your bucket is still private by design. To make this URL work publicly, you'd apply the scoped Bucket Policy approach from yesterday, or generate a pre-signed URL for safe temporary access.

---

## 6. Common Errors & How to Read Them

```
Error: "Access Denied"
  → Your IAM User's policy doesn't include s3:PutObject permission,
    OR your credentials in .env are wrong/expired

Error: "The bucket does not allow ACLs"
  → You tried to set an ACL (e.g., 'public-read') on upload —
    modern S3 buckets disable ACLs by default; use Bucket
    Policies (Day 9) instead, not per-object ACLs

Error: "Region mismatch" / SignatureDoesNotMatch
  → Your AWS_REGION in .env doesn't match the actual region
    your bucket was created in

Error: "Cannot read property 'buffer' of undefined"
  → The form field name in your request doesn't match
    upload.single('file') — make sure your form field
    is literally named "file"
```

---

## 7. The Bigger Picture — Where This Fits in a Real App

```
This pattern is exactly what powers:

  PulseBloom: user uploads a mood-journal photo
    → Express receives it → uploads to S3 → saves the
      returned S3 URL in the user's PostgreSQL record

  QueueCare: patient uploads a medical document
    → same flow, but bucket would stay fully private,
      accessed only via pre-signed URLs (Day 9) since
      this is sensitive data
```

```
Database row (conceptual):

  user_uploads table
  ┌────┬─────────┬──────────────────────────────────────────┐
  │ id │ user_id │ file_url                                   │
  ├────┼─────────┼──────────────────────────────────────────┤
  │ 1  │ 42      │ https://bucket.s3.ap-south-1.amazonaws.com/│
  │    │         │ uploads/1719500000000-avatar.jpg            │
  └────┴─────────┴──────────────────────────────────────────┘
```

This is the exact "store the file in S3, store only the reference in the database" pattern introduced on Day 8 — today you finally implemented it in real code.

---

## 8. Interview Questions (Practice Explaining These Out Loud)

**Q1: How would you handle file uploads in an Express + S3 backend?**
> Use Multer with memory storage to receive the file as a buffer, then use AWS SDK v3's S3Client and PutObjectCommand to upload that buffer directly to an S3 bucket, returning the resulting key/URL to the client.

**Q2: Why use `multer.memoryStorage()` instead of writing the file to disk first?**
> It's more efficient — the file stays as an in-memory buffer and gets forwarded straight to S3 without an unnecessary intermediate disk write/read cycle.

**Q3: Why prepend a timestamp to the S3 object key?**
> To avoid filename collisions — if two users upload files with the same original name, the timestamp ensures each gets a unique key in the bucket.

**Q4: Why shouldn't AWS credentials be hardcoded in your source code?**
> If the code or repository is ever leaked or made public, hardcoded credentials grant immediate, often broad access to your AWS account. Credentials belong in environment variables (excluded from Git), or better, an IAM Role when running on AWS infrastructure.

**Q5: Should you store the uploaded file's binary data in your PostgreSQL database?**
> No — store the file itself in S3, and store only the reference (URL or key) as a column in the database. This keeps the database lean, fast, and cost-efficient.

---

## 9. Hands-On Assignment for Today

1. Create a scoped IAM User for this project, store its keys in a `.env` file, and confirm `.env` is in `.gitignore`.
2. Install `@aws-sdk/client-s3`, `multer`, and `dotenv`.
3. Implement the `/upload` endpoint exactly as shown in Section 4.
4. Test it with `curl` or Postman, uploading a real file.
5. Confirm the file appears in your S3 bucket via the AWS Console.
6. **Bonus:** Add a second endpoint, `GET /files`, that uses `ListObjectsV2Command` from the same SDK to list all objects currently in your bucket (a good way to confirm uploads are landing where expected).
7. Write a short note in your own words covering:
   - The full request flow from client to S3
   - Why credentials live in `.env`, not in the code itself
   - What the returned S3 URL would typically be used for in a real app

---

## 10. Day 10 Goal — Can You Teach This?

By the end of today, you should be able to explain, without notes:

> *"My Express app uses Multer to receive an uploaded file as an in-memory buffer, then the AWS SDK v3's S3Client sends that buffer to my S3 bucket using PutObjectCommand. I never hardcode AWS credentials — they live in a gitignored .env file locally, and would be replaced by an IAM Role once deployed to AWS. The endpoint returns the file's S3 key/URL, which is what actually gets saved in the application's database — never the raw file itself."*

---

**Next up — Day 11: RDS Overview — MySQL/PostgreSQL Managed Databases**
