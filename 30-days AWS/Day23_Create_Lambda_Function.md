# Day 23: Create Lambda Function — Hands-On Deep Dive
### *(Real Functions, Real Triggers, Real Results)*

> **Roadmap reference:** Week 4, Day 23 — "Create Lambda Function"

---

## Why This Matters

Yesterday was conceptual — today you build real Lambda functions that do real things. By the end of today you'll have three working functions covering the most common Lambda patterns: a standalone utility function, an S3-triggered image processor, and a scheduled cron job. These three patterns cover about 80% of real-world Lambda use cases you'll encounter as a backend developer.

---

## 1. Today's Plan

```
Function 1: User Validator
  → A utility Lambda that validates user input data
  → Tests the core handler structure and environment variables
  → No trigger yet — invoked directly (good for learning mechanics)

Function 2: S3 File Processor
  → Triggered automatically when a file is uploaded to S3
  → Reads the file details, logs metadata, simulates processing
  → The most common real-world Lambda pattern

Function 3: Scheduled Cleanup Job
  → Triggered by EventBridge on a cron schedule
  → Simulates a nightly database cleanup task
  → The "background job" pattern every backend needs
```

---

## 2. Function 1: User Validator Lambda

### What It Does
Validates incoming user data — checks required fields, email format, and returns clean errors if validation fails. This represents any Lambda used as a utility function called from other services.

### Create the Function

```
AWS Console → Lambda → Create function
  Author from scratch
  Function name: user-validator
  Runtime: Node.js 20.x
  Architecture: x86_64

  Permissions:
    → Create a new role with basic Lambda permissions
      (just CloudWatch Logs access — this function doesn't
       touch any other AWS service)

  → Create function
```

### The Code

```javascript
// index.js — User Validator Lambda

exports.handler = async (event) => {
  console.log('Incoming event:', JSON.stringify(event, null, 2));

  // Extract user data from the event
  const { name, email, age } = event;
  const errors = [];

  // ── Validation rules ──────────────────────────────────────
  if (!name || typeof name !== 'string' || name.trim().length < 2) {
    errors.push('name must be at least 2 characters');
  }

  if (!email || !isValidEmail(email)) {
    errors.push('email must be a valid email address');
  }

  if (age !== undefined) {
    if (typeof age !== 'number' || age < 0 || age > 150) {
      errors.push('age must be a number between 0 and 150');
    }
  }

  // ── Return result ─────────────────────────────────────────
  if (errors.length > 0) {
    console.log('Validation failed:', errors);
    return {
      statusCode: 400,
      valid: false,
      errors: errors
    };
  }

  console.log('Validation passed for:', email);
  return {
    statusCode: 200,
    valid: true,
    sanitized: {
      name: name.trim(),
      email: email.toLowerCase(),
      age: age || null
    }
  };
};

function isValidEmail(email) {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
}
```

### Test Events to Try

```
Test 1 — Valid input:
{
  "name": "Ashish Anand",
  "email": "ASHISH@EXAMPLE.COM",
  "age": 25
}
Expected: statusCode 200, valid: true, email lowercased

Test 2 — Missing email:
{
  "name": "A",
  "age": 25
}
Expected: statusCode 400, valid: false, two errors
  (name too short + email missing)

Test 3 — Invalid email format:
{
  "name": "Ashish",
  "email": "not-an-email"
}
Expected: statusCode 400, errors: ["email must be a valid email address"]
```

### Using Environment Variables in Lambda

```javascript
// Instead of hardcoding values, use environment variables:
const MAX_NAME_LENGTH = parseInt(process.env.MAX_NAME_LENGTH) || 100;
const REQUIRED_DOMAIN = process.env.REQUIRED_DOMAIN || null;

// Set them in the console:
// Lambda → your function → Configuration → Environment variables
// KEY: MAX_NAME_LENGTH   VALUE: 50
// KEY: REQUIRED_DOMAIN   VALUE: company.com
```

> Environment variables in Lambda work exactly like `.env` files in local development — secure, configurable without redeploying code. For secrets (database passwords, API keys), use **AWS Secrets Manager** instead of environment variables directly, since Secrets Manager encrypts them and rotates them automatically.

---

## 3. Function 2: S3 File Processor (Event-Driven)

### What It Does
Triggers automatically whenever a new file is uploaded to an S3 bucket. Reads the file's metadata, logs it to CloudWatch, and returns a processing result. This is the pattern for image resizing, CSV processing, document scanning, etc.

### Create the Function

```
Lambda → Create function
  Name: s3-file-processor
  Runtime: Node.js 20.x

  Permissions:
    → Create a new role with basic Lambda permissions
    → After creation, ADD this permission to its role:
        IAM → Roles → s3-file-processor-role
          → Attach policy: AmazonS3ReadOnlyAccess
```

### The Code

```javascript
// S3 File Processor Lambda
const { S3Client, GetObjectCommand } = require('@aws-sdk/client-s3');

const s3Client = new S3Client({ region: process.env.AWS_REGION });

exports.handler = async (event) => {
  console.log('S3 Event:', JSON.stringify(event, null, 2));

  // event.Records contains one entry per file — S3 can batch
  // multiple events, so always loop through Records
  const results = [];

  for (const record of event.Records) {

    const bucketName = record.s3.bucket.name;
    const objectKey  = decodeURIComponent(
                         record.s3.object.key.replace(/\+/g, ' ')
                       );
    const fileSize   = record.s3.object.size;
    const eventTime  = record.eventTime;

    console.log(`Processing file: ${objectKey} (${fileSize} bytes)`);

    // Determine file type from extension
    const extension = objectKey.split('.').pop().toLowerCase();
    const fileType  = getFileType(extension);

    // Simulate different processing based on file type
    let processingResult;
    if (fileType === 'image') {
      processingResult = await processImage(bucketName, objectKey);
    } else if (fileType === 'document') {
      processingResult = await processDocument(bucketName, objectKey);
    } else {
      processingResult = { status: 'skipped', reason: 'unsupported file type' };
    }

    results.push({
      file:      objectKey,
      bucket:    bucketName,
      size:      fileSize,
      type:      fileType,
      uploadedAt: eventTime,
      processing: processingResult
    });
  }

  console.log('Processing complete:', JSON.stringify(results, null, 2));
  return { processed: results.length, results };
};

// ── Helper functions ──────────────────────────────────────────

function getFileType(extension) {
  const imageExts    = ['jpg', 'jpeg', 'png', 'gif', 'webp'];
  const documentExts = ['pdf', 'doc', 'docx', 'txt'];
  if (imageExts.includes(extension))    return 'image';
  if (documentExts.includes(extension)) return 'document';
  return 'other';
}

async function processImage(bucket, key) {
  // In a real function, you'd use sharp library to resize here
  // For now, simulate processing
  console.log(`[IMAGE] Would resize: s3://${bucket}/${key}`);
  return {
    status: 'processed',
    action: 'resize',
    outputKey: key.replace('uploads/', 'thumbnails/')
  };
}

async function processDocument(bucket, key) {
  // In a real function, you'd extract text, scan for viruses, etc.
  console.log(`[DOCUMENT] Would scan: s3://${bucket}/${key}`);
  return {
    status: 'processed',
    action: 'virus_scan',
    clean: true
  };
}
```

### Add the S3 Trigger

```
Lambda → s3-file-processor → Configuration → Triggers
  → Add trigger
  → Select: S3
  → Bucket: your-bucket-from-day-8
  → Event types: PUT (ObjectCreated:Put)
  → Prefix: uploads/   ← only trigger for files in uploads/ folder
                          (important: avoids infinite loops if Lambda
                           also writes to the same bucket)
  → Add
```

> **Critical warning:** If your Lambda reads from S3 bucket A and writes back to S3 bucket A without a prefix/suffix filter, it will trigger itself infinitely, racking up enormous costs in seconds. Always use prefix or suffix filters, or use separate buckets for input/output.

### Test It

```bash
# Upload a file to your S3 bucket's uploads/ folder
aws s3 cp ./test-image.jpg s3://your-bucket-name/uploads/test-image.jpg

# Or via the console: S3 → your bucket → uploads/ → Upload
```

```
Then check CloudWatch Logs:
  Log Groups → /aws/lambda/s3-file-processor
  → Find the latest log stream
  → You should see the file's metadata logged
```

---

## 4. Function 3: Scheduled Cleanup Job (Cron)

### What It Does
Runs automatically on a schedule — simulates a nightly database cleanup task. This pattern handles any background job that needs to run periodically without a server.

### Create the Function

```
Lambda → Create function
  Name: nightly-cleanup-job
  Runtime: Node.js 20.x
  Permissions: Create new role with basic Lambda permissions
```

### The Code

```javascript
// Nightly Cleanup Job Lambda
exports.handler = async (event) => {
  console.log('Cleanup job started at:', new Date().toISOString());
  console.log('Triggered by:', JSON.stringify(event, null, 2));

  const results = {
    startTime:        new Date().toISOString(),
    tasksCompleted:   [],
    errors:           []
  };

  // ── Task 1: Clean up expired sessions ─────────────────────
  try {
    const expiredCount = await cleanExpiredSessions();
    results.tasksCompleted.push({
      task: 'clean_expired_sessions',
      count: expiredCount,
      status: 'success'
    });
  } catch (err) {
    console.error('Session cleanup failed:', err.message);
    results.errors.push({ task: 'clean_expired_sessions', error: err.message });
  }

  // ── Task 2: Archive old records ────────────────────────────
  try {
    const archivedCount = await archiveOldRecords();
    results.tasksCompleted.push({
      task: 'archive_old_records',
      count: archivedCount,
      status: 'success'
    });
  } catch (err) {
    console.error('Archive task failed:', err.message);
    results.errors.push({ task: 'archive_old_records', error: err.message });
  }

  results.endTime   = new Date().toISOString();
  results.success   = results.errors.length === 0;

  console.log('Cleanup complete:', JSON.stringify(results, null, 2));

  // If there were errors, throw so CloudWatch alarm can detect it
  if (results.errors.length > 0) {
    throw new Error(`Cleanup completed with ${results.errors.length} error(s)`);
  }

  return results;
};

// ── Simulated database operations ────────────────────────────
// In production, these would connect to RDS and run real queries

async function cleanExpiredSessions() {
  console.log('Cleaning sessions older than 30 days...');
  // Real code: await pool.query("DELETE FROM sessions WHERE created_at < NOW() - INTERVAL '30 days'")
  await sleep(100); // simulate DB query time
  const deletedCount = Math.floor(Math.random() * 50);
  console.log(`Deleted ${deletedCount} expired sessions`);
  return deletedCount;
}

async function archiveOldRecords() {
  console.log('Archiving records older than 1 year...');
  // Real code: await pool.query("INSERT INTO archive SELECT * FROM logs WHERE created_at < NOW() - INTERVAL '1 year'")
  await sleep(150);
  const archivedCount = Math.floor(Math.random() * 200);
  console.log(`Archived ${archivedCount} old records`);
  return archivedCount;
}

function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}
```

### Add the EventBridge (Cron) Trigger

```
Lambda → nightly-cleanup-job → Configuration → Triggers
  → Add trigger
  → Select: EventBridge (CloudWatch Events)
  → Rule: Create a new rule
  → Rule name: nightly-cleanup-schedule
  → Rule type: Schedule expression
  → Schedule expression: cron(0 21 * * ? *)
                          ↑  ↑
                          │  └── 9 PM UTC = 2:30 AM IST
                          └───── minute 0

  → Add
```

```
Cron expression format in AWS EventBridge:
  cron(minute  hour  day-of-month  month  day-of-week  year)

  Examples:
    cron(0 21 * * ? *)     → Every day at 9 PM UTC
    cron(0 */6 * * ? *)    → Every 6 hours
    cron(0 9 ? * MON-FRI *) → Every weekday at 9 AM UTC
    cron(0 0 1 * ? *)       → First day of every month at midnight
```

---

## 5. Deploying Lambda from Local Code (ZIP Upload)

For larger functions with npm dependencies, you need to package and upload a ZIP instead of using the inline editor:

```bash
# On your local machine
mkdir my-lambda && cd my-lambda

npm init -y
npm install axios          # example dependency

# Create your handler
cat > index.js << 'EOF'
const axios = require('axios');

exports.handler = async (event) => {
  const response = await axios.get('https://api.github.com/users/ashish');
  return {
    statusCode: 200,
    body: JSON.stringify({
      login: response.data.login,
      name:  response.data.name
    })
  };
};
EOF

# Package everything including node_modules
zip -r function.zip index.js node_modules package.json
```

```
Upload to Lambda:
  Lambda → your function → Code → Upload from → .zip file
  → Upload function.zip → Save
```

> **Important:** Always run `npm install` BEFORE zipping, and include `node_modules/` in the ZIP. Lambda doesn't run `npm install` — it expects everything already packaged. This is different from how you'd push to Heroku or Beanstalk.

---

## 6. Lambda Layers — Sharing Code Across Functions

If multiple Lambda functions use the same dependencies (e.g., `pg`, `axios`, `lodash`), you can package them as a **Layer** and attach it to multiple functions — rather than bundling the same `node_modules` in every function's ZIP.

```
Without Layers:
  function-A.zip (50MB: code + pg + axios + lodash)
  function-B.zip (50MB: code + pg + axios + lodash)
  function-C.zip (50MB: code + pg + axios + lodash)
  → Duplicated dependencies, large ZIPs

With a Layer:
  shared-layer.zip (48MB: pg + axios + lodash)
  function-A.zip (2MB: just your code)
  function-B.zip (2MB: just your code)
  function-C.zip (2MB: just your code)
  → One layer attached to all three functions
  → Fast deploys, shared updates
```

---

## 7. Common Errors and Fixes

```
Error: "Task timed out after 3.00 seconds"
  Fix: Lambda → Configuration → General configuration
       Increase timeout to 30s or more for DB-connected functions
       Default is 3 seconds — too low for anything real

Error: "Cannot find module 'pg'"
  Fix: You're using the inline editor but the function needs
       external npm packages. Switch to ZIP upload including
       node_modules/, or use a Lambda Layer.

Error: "AccessDenied: User is not authorized to perform s3:GetObject"
  Fix: Your Lambda Execution Role lacks S3 permissions.
       IAM → Roles → your function's role → Attach policy
       → AmazonS3ReadOnlyAccess (or a custom scoped policy)

Error: "Process exited before completing request"
  Fix: Unhandled promise rejection or synchronous throw.
       Always wrap handler in try/catch, and use async/await
       consistently — never mix callbacks with async/await.

Error: Lambda triggered infinitely from S3
  Fix: Add a prefix filter to the S3 trigger — ensure the Lambda
       only fires on uploads/, never on the output path it writes to.
```

---

## 8. Interview Questions (Practice Explaining These Out Loud)

**Q1: Walk me through creating a Lambda function triggered by S3.**
> Create the function with Node.js runtime, attach an Execution Role with S3 read permissions and CloudWatch Logs write permissions. In the function code, read `event.Records` to extract bucket name and object key. Add an S3 trigger in the Lambda console, specifying the bucket, event type (ObjectCreated), and a prefix filter to avoid infinite trigger loops.

**Q2: What is a Lambda Layer, and when would you use one?**
> A Layer is a ZIP archive of shared dependencies or utilities that multiple Lambda functions can reference, avoiding duplication. Use it when several functions share the same npm packages — one layer update propagates to all attached functions simultaneously.

**Q3: How does a scheduled Lambda function work?**
> An EventBridge rule with a cron or rate expression triggers the Lambda function on schedule. The function receives an event object describing the schedule rule, executes its logic, and exits. Lambda handles all the infrastructure — no cron server or EC2 instance needed.

**Q4: Why must you zip `node_modules` before uploading a Lambda function?**
> Lambda doesn't run `npm install` during deployment — it expects the entire runtime environment pre-packaged. Unlike platforms like Heroku, Lambda runs your ZIP as-is, so all dependencies must be included in the package.

**Q5: What's the risk of an S3-triggered Lambda writing back to the same S3 bucket?**
> An infinite loop — the Lambda writes a file, which triggers itself again, which writes another file, which triggers itself again. This generates exponential invocations in seconds, causing massive unexpected cost. Always use prefix/suffix filters or separate input/output buckets.

---

## 9. Hands-On Assignment for Today

1. **Create and test Function 1** (user-validator) — run all three test events and confirm the outputs match expectations.

2. **Add environment variables** to Function 1 — a `MAX_NAME_LENGTH` of 50 — and update the validation code to use it.

3. **Create Function 2** (s3-file-processor) — attach the S3 trigger with an `uploads/` prefix filter.
   - Upload a `.jpg` and a `.pdf` to your bucket's `uploads/` folder.
   - Confirm Lambda fires and the file type logic routes correctly in CloudWatch Logs.

4. **Create Function 3** (nightly-cleanup-job) — add the EventBridge cron trigger.
   - Change the schedule to `rate(2 minutes)` temporarily to test it fires.
   - Watch CloudWatch Logs for the cleanup output.
   - Switch back to a nightly cron when confirmed working.

5. **Package a local function with dependencies:**
   ```bash
   mkdir lambda-test && cd lambda-test
   npm init -y && npm install axios
   # Write a simple handler using axios
   zip -r function.zip .
   # Upload to a new Lambda function
   ```

6. Write a short note covering:
   - The S3 infinite loop problem and how prefix filters solve it
   - When you'd use a Layer vs bundling dependencies in the ZIP
   - The cron expression for "every Monday at 9 AM IST" (convert to UTC)

---

## 10. Day 23 Goal — Can You Teach This?

By the end of today, you should be able to explain, without notes:

> *"Lambda functions are created with a runtime, an execution role, and a trigger. For S3-triggered functions, the event object contains Records with bucket name and object key — always use prefix filters to prevent infinite loops. Scheduled functions use EventBridge cron expressions and need no infrastructure beyond the function itself. For functions with npm dependencies, I package everything including node_modules into a ZIP and upload it — or use a Layer for dependencies shared across multiple functions."*

---

**Next up — Day 24: API Gateway**
