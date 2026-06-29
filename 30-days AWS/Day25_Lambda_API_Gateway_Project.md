# Day 25: Lambda + API Gateway Project
### *(Portfolio Project 3: Serverless URL Shortener — Lambda + API Gateway + DynamoDB)*

> **Roadmap reference:** Week 4, Day 25 — "Lambda + API Gateway Project" | **Portfolio Project 3:** Serverless URL Shortener (Lambda + API Gateway + DynamoDB)

---

## Why This Matters

Days 22–24 built the components. Today you combine them into a complete, production-worthy project: a **URL Shortener** — one of the most classic serverless architectures because it perfectly demonstrates Lambda's strengths (lightweight, event-driven, scales to zero) and introduces **DynamoDB** as the natural data store for this use case.

By the end of today you'll have a working, deployable portfolio project with a real URL you can share.

---

## 1. What You're Building

```
POST /shorten
  Input:  { "url": "https://www.example.com/very/long/path?with=params" }
  Output: { "shortUrl": "https://abc123.execute-api.../r/Xk9mP2" }

GET /r/{code}
  Input:  short code in the path (e.g., Xk9mP2)
  Output: HTTP 301 redirect → original long URL

GET /stats/{code}
  Input:  short code
  Output: { "shortCode": "Xk9mP2", "originalUrl": "...", "clicks": 47 }

DELETE /r/{code}
  Input:  short code
  Output: { "message": "URL deleted" }
```

---

## 2. Why DynamoDB — Not RDS?

This is a design decision worth explaining in interviews:

```
RDS (PostgreSQL):
  → Relational, good for complex joins and transactions
  → Always running = always billing
  → Connection pooling complexity with Lambda
    (Lambda's stateless nature + RDS's connection limits
     can cause "too many connections" under load)

DynamoDB:
  → NoSQL key-value/document store
  → Pay per request (no idle cost — perfect for Lambda)
  → Scales automatically, no connection management
  → Read/write by key in < 10ms typically
  → Free Tier: 25 GB storage + 25 RCU + 25 WCU ALWAYS FREE

For a URL shortener:
  → Every operation is a simple key lookup (shortCode → url)
  → No complex joins needed
  → Traffic can be extremely spiky (viral links)
  → DynamoDB is the obvious choice
```

---

## 3. DynamoDB Basics — Just Enough to Build This

```
TABLE
  → Top-level container (like a table in SQL, but schema-free)

PRIMARY KEY
  → Uniquely identifies each item in the table
  → For a URL shortener: shortCode is the primary key
  → Every other attribute is flexible (no fixed schema)

ITEM
  → A single record (like a row in SQL)
  → Can have any attributes beyond the primary key

{
  "shortCode": "Xk9mP2",          ← Primary Key (Partition Key)
  "originalUrl": "https://...",    ← Attribute
  "createdAt":   "2026-06-22...",  ← Attribute
  "clicks":      47,               ← Attribute
  "expiresAt":   "2026-07-22..."   ← Attribute (optional TTL field)
}
```

### Create the DynamoDB Table

```
AWS Console → DynamoDB → Create table

  Table name:        url-shortener
  Partition key:     shortCode (String)

  Table settings: Default settings (On-demand capacity mode)
    → On-demand: you pay per request, no minimum cost
    → Perfect for Lambda — both scale to zero together

  → Create table
```

---

## 4. IAM Role for the Lambda Functions

Your Lambda functions need DynamoDB access:

```
IAM → Roles → Create role
  Trusted entity: AWS service → Lambda
  Attach policies:
    AWSLambdaBasicExecutionRole   ← CloudWatch Logs
    AmazonDynamoDBFullAccess       ← DynamoDB read/write
  Name: url-shortener-lambda-role
```

---

## 5. The Lambda Functions

### Function 1: shorten-url (POST /shorten)

```javascript
// shorten-url/index.js
const { DynamoDBClient, PutItemCommand,
        GetItemCommand } = require('@aws-sdk/client-dynamodb');
const { marshall, unmarshall } = require('@aws-sdk/util-dynamodb');

const client    = new DynamoDBClient({ region: process.env.AWS_REGION });
const TABLE     = process.env.TABLE_NAME || 'url-shortener';
const BASE_URL  = process.env.BASE_URL;   // your API Gateway URL

exports.handler = async (event) => {
  try {
    const body       = JSON.parse(event.body || '{}');
    const originalUrl = body.url;

    // ── Validate input ──────────────────────────────────────
    if (!originalUrl) {
      return response(400, { error: 'url is required' });
    }

    if (!isValidUrl(originalUrl)) {
      return response(400, { error: 'url must be a valid http/https URL' });
    }

    // ── Generate a unique short code ────────────────────────
    const shortCode = generateCode(6);

    // ── Save to DynamoDB ────────────────────────────────────
    const item = {
      shortCode,
      originalUrl,
      createdAt: new Date().toISOString(),
      clicks:    0,
      expiresAt: Math.floor(Date.now() / 1000) + (30 * 24 * 60 * 60) // 30 days TTL
    };

    await client.send(new PutItemCommand({
      TableName: TABLE,
      Item:      marshall(item),
      ConditionExpression: 'attribute_not_exists(shortCode)' // avoid overwrites
    }));

    // ── Return the short URL ────────────────────────────────
    return response(201, {
      shortCode,
      shortUrl:    `${BASE_URL}/r/${shortCode}`,
      originalUrl,
      expiresAt:   new Date(item.expiresAt * 1000).toISOString()
    });

  } catch (err) {
    console.error('Error shortening URL:', err);
    return response(500, { error: 'Failed to shorten URL' });
  }
};

// ── Helpers ────────────────────────────────────────────────────

function generateCode(length) {
  const chars = 'ABCDEFGHJKMNPQRSTUVWXYZabcdefghjkmnpqrstuvwxyz23456789';
  // Removed confusing chars: 0,O,1,I,l
  return Array.from({ length }, () =>
    chars[Math.floor(Math.random() * chars.length)]
  ).join('');
}

function isValidUrl(str) {
  try {
    const url = new URL(str);
    return url.protocol === 'http:' || url.protocol === 'https:';
  } catch {
    return false;
  }
}

function response(statusCode, body) {
  return {
    statusCode,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*'
    },
    body: JSON.stringify(body)
  };
}
```

### Function 2: redirect-url (GET /r/{code})

```javascript
// redirect-url/index.js
const { DynamoDBClient, GetItemCommand,
        UpdateItemCommand } = require('@aws-sdk/client-dynamodb');
const { marshall, unmarshall } = require('@aws-sdk/util-dynamodb');

const client = new DynamoDBClient({ region: process.env.AWS_REGION });
const TABLE  = process.env.TABLE_NAME || 'url-shortener';

exports.handler = async (event) => {
  try {
    const shortCode = event.pathParameters?.code;

    if (!shortCode) {
      return response(400, { error: 'Short code is required' });
    }

    // ── Look up the short code ──────────────────────────────
    const result = await client.send(new GetItemCommand({
      TableName: TABLE,
      Key:       marshall({ shortCode })
    }));

    if (!result.Item) {
      return response(404, { error: 'Short URL not found or expired' });
    }

    const item = unmarshall(result.Item);

    // ── Increment click counter (fire-and-forget) ───────────
    // We don't await this — let the redirect happen immediately
    // and update the counter asynchronously
    client.send(new UpdateItemCommand({
      TableName:                 TABLE,
      Key:                       marshall({ shortCode }),
      UpdateExpression:          'ADD clicks :inc',
      ExpressionAttributeValues: marshall({ ':inc': 1 })
    })).catch(err => console.error('Failed to update click count:', err));

    // ── 301 Redirect to the original URL ───────────────────
    return {
      statusCode: 301,
      headers: {
        Location:                     item.originalUrl,
        'Cache-Control':              'no-cache',   // don't cache redirects
        'Access-Control-Allow-Origin': '*'
      },
      body: ''
    };

  } catch (err) {
    console.error('Redirect error:', err);
    return response(500, { error: 'Failed to redirect' });
  }
};

function response(statusCode, body) {
  return {
    statusCode,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*'
    },
    body: JSON.stringify(body)
  };
}
```

### Function 3: get-stats (GET /stats/{code})

```javascript
// get-stats/index.js
const { DynamoDBClient, GetItemCommand } = require('@aws-sdk/client-dynamodb');
const { marshall, unmarshall } = require('@aws-sdk/util-dynamodb');

const client = new DynamoDBClient({ region: process.env.AWS_REGION });
const TABLE  = process.env.TABLE_NAME || 'url-shortener';

exports.handler = async (event) => {
  try {
    const shortCode = event.pathParameters?.code;

    const result = await client.send(new GetItemCommand({
      TableName: TABLE,
      Key:       marshall({ shortCode })
    }));

    if (!result.Item) {
      return response(404, { error: 'Short URL not found' });
    }

    const item = unmarshall(result.Item);

    return response(200, {
      shortCode:   item.shortCode,
      originalUrl: item.originalUrl,
      clicks:      item.clicks || 0,
      createdAt:   item.createdAt,
      expiresAt:   new Date(item.expiresAt * 1000).toISOString()
    });

  } catch (err) {
    console.error('Stats error:', err);
    return response(500, { error: 'Failed to get stats' });
  }
};

function response(statusCode, body) {
  return {
    statusCode,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*'
    },
    body: JSON.stringify(body)
  };
}
```

### Function 4: delete-url (DELETE /r/{code})

```javascript
// delete-url/index.js
const { DynamoDBClient, DeleteItemCommand,
        GetItemCommand } = require('@aws-sdk/client-dynamodb');
const { marshall, unmarshall } = require('@aws-sdk/util-dynamodb');

const client = new DynamoDBClient({ region: process.env.AWS_REGION });
const TABLE  = process.env.TABLE_NAME || 'url-shortener';

exports.handler = async (event) => {
  try {
    const shortCode = event.pathParameters?.code;

    // Check it exists first
    const existing = await client.send(new GetItemCommand({
      TableName: TABLE,
      Key:       marshall({ shortCode })
    }));

    if (!existing.Item) {
      return response(404, { error: 'Short URL not found' });
    }

    await client.send(new DeleteItemCommand({
      TableName: TABLE,
      Key:       marshall({ shortCode })
    }));

    return response(200, {
      message:   'Short URL deleted successfully',
      shortCode
    });

  } catch (err) {
    console.error('Delete error:', err);
    return response(500, { error: 'Failed to delete URL' });
  }
};

function response(statusCode, body) {
  return {
    statusCode,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*'
    },
    body: JSON.stringify(body)
  };
}
```

---

## 6. Deploying — ZIP Upload with Dependencies

```bash
# For each function:
mkdir shorten-url && cd shorten-url
npm init -y
npm install @aws-sdk/client-dynamodb @aws-sdk/util-dynamodb
# paste the function code into index.js
zip -r ../shorten-url.zip .
cd ..

# Repeat for redirect-url, get-stats, delete-url
# Upload each ZIP to its respective Lambda function
```

### Environment Variables for Each Function

```
Lambda → each function → Configuration → Environment variables:
  TABLE_NAME: url-shortener
  BASE_URL:   https://YOUR-API-ID.execute-api.ap-south-1.amazonaws.com
```

---

## 7. API Gateway Setup

```
API Gateway → Create API → HTTP API

  Integrations — add all four Lambda functions:
    Lambda: shorten-url
    Lambda: redirect-url
    Lambda: get-stats
    Lambda: delete-url

  Routes:
    POST    /shorten     → shorten-url
    GET     /r/{code}    → redirect-url
    GET     /stats/{code}→ get-stats
    DELETE  /r/{code}    → delete-url

  Stage: prod, auto-deploy enabled
  → Create

Copy the API URL → paste into BASE_URL env var of shorten-url function
```

---

## 8. End-to-End Testing

```bash
BASE="https://YOUR-API-ID.execute-api.ap-south-1.amazonaws.com"

# 1. Shorten a URL
curl -X POST $BASE/shorten \
  -H "Content-Type: application/json" \
  -d '{"url":"https://docs.aws.amazon.com/lambda/latest/dg/welcome.html"}'

# Expected:
# {
#   "shortCode": "Xk9mP2",
#   "shortUrl": "https://YOUR-API.../r/Xk9mP2",
#   "originalUrl": "https://docs.aws.amazon.com/...",
#   "expiresAt": "2026-07-22T..."
# }

# 2. Use the short URL (open in browser — should redirect!)
curl -L $BASE/r/Xk9mP2
# -L follows redirects — you should land on the AWS docs page

# 3. Check stats (click count should be 1 now)
curl $BASE/stats/Xk9mP2

# Expected:
# { "shortCode":"Xk9mP2", "clicks":1, "originalUrl":"...", ... }

# 4. Test invalid URL
curl -X POST $BASE/shorten \
  -H "Content-Type: application/json" \
  -d '{"url":"not-a-valid-url"}'
# Expected: 400 error

# 5. Test missing URL
curl -X POST $BASE/shorten \
  -H "Content-Type: application/json" \
  -d '{}'
# Expected: 400, "url is required"

# 6. Test unknown short code
curl $BASE/r/XXXXXX
# Expected: 404, "Short URL not found or expired"

# 7. Delete the URL
curl -X DELETE $BASE/r/Xk9mP2
# Expected: 200, "Short URL deleted successfully"

# 8. Confirm it's gone
curl $BASE/r/Xk9mP2
# Expected: 404
```

---

## 9. DynamoDB TTL — Auto-Expiry for Short URLs

**TTL (Time To Live) is a DynamoDB feature that automatically deletes items when a specified timestamp attribute expires — at zero cost.**

```
DynamoDB → url-shortener table → Additional settings
  → Time to Live (TTL)
  → TTL attribute name: expiresAt
  → Enable TTL

Now, any item where expiresAt (a Unix timestamp) is in the past
will be automatically deleted by DynamoDB within ~48 hours.

Your shorten-url function already sets expiresAt to 30 days from now —
after 30 days, the short URL disappears without any Lambda or cron job.
```

> This is an elegant, zero-cost alternative to running a nightly cleanup Lambda — DynamoDB handles expiry natively. A great pattern to mention in interviews.

---

## 10. The Complete Architecture

```
                        Internet
                            │
               ┌────────────┴─────────────┐
               │                            │
         POST /shorten              GET /r/{code}
         GET /stats/{code}          DELETE /r/{code}
               │                            │
               ▼                            ▼
        ┌───────────────────────────────────────┐
        │           API Gateway (HTTP API)        │
        │                                         │
        │  POST /shorten    → shorten-url λ       │
        │  GET  /r/{code}   → redirect-url λ      │
        │  GET  /stats/{code}→ get-stats λ        │
        │  DELETE /r/{code} → delete-url λ        │
        └───────────────────────────────────────┘
               │           │          │        │
               ▼           ▼          ▼        ▼
          ┌──────┐    ┌──────┐  ┌──────┐  ┌──────┐
          │shorten│    │redir │  │stats │  │delete│
          │  λ   │    │  λ   │  │  λ   │  │  λ   │
          └──────┘    └──────┘  └──────┘  └──────┘
               │           │          │        │
               └───────────┴──────────┴────────┘
                                 │
                    ┌────────────▼────────────┐
                    │   DynamoDB                │
                    │   url-shortener table      │
                    │   TTL: auto-expire 30 days │
                    └───────────────────────────┘

Total infrastructure managed by YOU:
  → 4 Lambda functions (code only)
  → 1 DynamoDB table
  → 1 API Gateway

Total infrastructure managed by AWS:
  → Servers, scaling, patching, availability — everything else
```

---

## 11. Interview Questions (Practice Explaining These Out Loud)

**Q1: Walk me through your URL Shortener architecture.**
> It's fully serverless — API Gateway receives HTTP requests and routes them to one of four Lambda functions. Each function reads from or writes to a DynamoDB table keyed on the short code. DynamoDB TTL auto-expires links after 30 days. There are no servers to manage, it scales to zero cost when idle, and to thousands of concurrent requests under load.

**Q2: Why DynamoDB instead of RDS for this use case?**
> Every operation is a simple key lookup — no joins needed. DynamoDB's per-request pricing means zero cost at zero traffic, matching Lambda's cost model. RDS requires a minimum running cost and connection management that conflicts with Lambda's stateless model under load.

**Q3: How does the click counter increment without slowing down the redirect?**
> The redirect function fires the DynamoDB UpdateItem call without awaiting it — a fire-and-forget pattern. The 301 redirect is returned immediately while the counter update happens asynchronously in the background, keeping redirect latency minimal.

**Q4: What is DynamoDB TTL and how did you use it here?**
> TTL is a DynamoDB feature that automatically deletes items when a Unix timestamp attribute expires, at no cost. Short URLs store an `expiresAt` timestamp 30 days from creation — DynamoDB silently removes them after that with no Lambda or cron job needed.

**Q5: What happens if two requests try to create the same short code simultaneously?**
> The `PutItemCommand` uses a `ConditionExpression: attribute_not_exists(shortCode)` — if the code already exists, DynamoDB rejects the write with a `ConditionalCheckFailedException`. The function should handle this by retrying with a newly generated code.

---

## 12. Hands-On Assignment for Today

1. **Create the DynamoDB table** (`url-shortener`, partition key `shortCode`, on-demand mode) and **enable TTL** on `expiresAt`.

2. **Create the IAM role** with DynamoDB + CloudWatch permissions.

3. **Create and deploy all four Lambda functions** — zip with dependencies, upload, set environment variables.

4. **Create the HTTP API** in API Gateway with all four routes — copy the URL into `BASE_URL` env var.

5. **Run the full test sequence** from Section 8 — verify all 8 curl tests pass.

6. **Open the short URL in a browser** — confirm it actually redirects.

7. Check **DynamoDB → url-shortener** in the console — see your items, including the `clicks` counter incrementing.

8. Write a short note in your own words covering:
   - Why this architecture costs nearly zero when there's no traffic
   - How DynamoDB TTL eliminated the need for a cleanup Lambda
   - The fire-and-forget click counter pattern and why it improves UX

---

## 13. Day 25 Goal — Can You Teach This?

By the end of today, you should be able to explain, without notes:

> *"My URL Shortener is fully serverless — API Gateway routes four HTTP endpoints to four Lambda functions, all reading from and writing to a DynamoDB table keyed on the short code. DynamoDB TTL auto-expires links after 30 days with zero extra code. Click counting uses a fire-and-forget update so redirects stay fast. The whole system costs near zero at rest, scales instantly under load, and has zero infrastructure to manage — exactly what makes serverless the right choice for this workload."*

---

## Portfolio Project 3 Complete! 🎉

**Serverless URL Shortener** — API Gateway + Lambda + DynamoDB — fully built, tested, and deployable. This project demonstrates:
- Serverless architecture design
- DynamoDB data modeling
- Lambda function composition
- API Gateway route configuration
- Production patterns (TTL, fire-and-forget, conditional writes)

---

**Next up — Day 26: Docker Basics**
