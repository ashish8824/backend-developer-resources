# Day 9: S3 Permissions & Bucket Policies
### *(The Single Most Common Cause of Real-World AWS Data Breaches — and How to Avoid It)*

> **Roadmap reference:** Week 2, Day 9 — "S3 Permissions, Bucket Policies"

---

## Why This Matters

Search "S3 data breach" and you'll find a long, recurring history of companies — including large ones — accidentally exposing millions of user records because of one thing: **a misconfigured S3 bucket made public by mistake.**

Yesterday you deliberately left "Block Public Access" turned ON. Today you learn *exactly* how S3 permissions work, so that any time you DO need to open access, you do it precisely and intentionally — never by accident.

---

## 1. The Three Layers of S3 Access Control

This is the part that confuses almost everyone at first — S3 permissions aren't decided by just one setting, but by **multiple layers stacked together**, and the *most restrictive* one always wins.

```
┌──────────────────────────────────────────────┐
│  1. Block Public Access (account/bucket level) │  ← master override switch
├──────────────────────────────────────────────┤
│  2. IAM Policies (attached to users/roles)      │  ← "who" can do "what"
├──────────────────────────────────────────────┤
│  3. Bucket Policies (attached to the bucket)    │  ← bucket-level rules
├──────────────────────────────────────────────┤
│  4. ACLs (Access Control Lists — legacy)        │  ← older, mostly avoided now
└──────────────────────────────────────────────┘

   If ANY layer denies access → access is denied.
   ALL relevant layers must agree to ALLOW for access to succeed.
```

> **Key mental model:** Think of these layers like multiple locked doors in sequence. It doesn't matter if 3 out of 4 doors are unlocked — if even one is locked, you don't get through.

---

## 2. Layer 1: Block Public Access — The Master Switch

**Block Public Access is an account-level and bucket-level safety setting that, when enabled, overrides and blocks ANY other policy that would otherwise make content public — even if you accidentally write one.**

```
4 sub-settings under "Block Public Access":
  1. Block public access via new ACLs
  2. Block public access via any ACLs
  3. Block public access via new public bucket policies
  4. Block public access via any public bucket policies
```

```
Scenario:
  You accidentally write a Bucket Policy that makes
  everything public...
        ↓
  ...but Block Public Access is still ON
        ↓
  Result: Still blocked. AWS protects you from your own mistake.
```

> This is exactly why AWS made Block Public Access **ON by default** for new buckets (as you saw yesterday) — it's a deliberate guardrail against the #1 historical cause of S3 breaches.

---

## 3. Layer 2: IAM Policies vs Bucket Policies — What's the Difference?

This distinction is one of the most commonly asked S3 interview questions.

```
IAM POLICY                          BUCKET POLICY
────────────                        ──────────────
Attached to: a User, Group, or Role  Attached to: the bucket itself
Answers: "What can THIS IDENTITY     Answers: "What can ANYONE do
          do, across AWS?"                     to THIS BUCKET?"
Good for: your own team/apps          Good for: granting access to
                                       external accounts, or making
                                       specific content public
```

```
         IAM POLICY                        BUCKET POLICY
  (attached to a User/Role)            (attached to the Bucket)
            │                                    │
            └────────────────┬───────────────────┘
                              ▼
                    Both are evaluated —
                  access only succeeds if
                  nothing denies it
```

> **Practical rule of thumb:** Use IAM Policies when managing access for your own users/applications (e.g., "my EC2 Role can read this bucket"). Use Bucket Policies when you need rules tied to the bucket regardless of identity (e.g., "make this one folder of marketing images public-readable").

---

## 4. Anatomy of a Bucket Policy

Bucket Policies use the same JSON format you learned with IAM on Day 3 — same `Effect`/`Action`/`Resource` structure, just attached to a bucket instead of a user.

### Example: Making a Specific Folder Publicly Readable

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadForMarketingAssets",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::pulsebloom-assets/public/*"
    }
  ]
}
```

```
Breaking this down:
  Principal: "*"          → applies to EVERYONE (the public internet)
  Action: s3:GetObject     → allows only READING objects, not deleting/writing
  Resource: ".../public/*" → ONLY the "public/" folder — nothing else
                              in the bucket is affected
```

> Notice the scoping discipline here — same Principle of Least Privilege from Day 3. This policy intentionally opens up only one specific subfolder, not the entire bucket.

### Example: Restricting Access to a Specific VPC (More Advanced, Good to Recognize)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": "arn:aws:s3:::pulsebloom-private/*",
      "Condition": {
        "StringNotEquals": {
          "aws:sourceVpc": "vpc-0abc1234"
        }
      }
    }
  ]
}
```

```
Plain English: "Deny ALL access to this bucket, UNLESS the
request is coming from this specific VPC."
```

This pattern is common for backend-only data that should never be reachable from the public internet at all — only from your own application's network.

---

## 5. Public vs Private — Choosing the Right Approach Per Use Case

```
PUBLIC content (safe to open broadly):
  - Marketing images, public blog assets
  - Static website files (HTML/CSS/JS for a frontend build)
  → Use a Bucket Policy scoped to a specific public/ prefix

PRIVATE content (must stay restricted):
  - User health/mood data (PulseBloom)
  - Patient records (QueueCare)
  - Any personally identifiable information (PII)
  → Keep Block Public Access ON, use IAM Roles + pre-signed URLs instead
```

### Pre-Signed URLs — The Safe Way to Share Private Files Temporarily

**A pre-signed URL is a temporary, time-limited link that grants access to a specific private S3 object, without making the bucket public.**

```
Use case: A user wants to download their own private uploaded
document, but the bucket itself must stay fully private.

Your backend generates a pre-signed URL (valid for, say, 5 minutes)
        ↓
Frontend uses that URL directly to download the file
        ↓
After 5 minutes, the URL stops working — no permanent public access
```

```
Normal private object:
  https://pulsebloom-private.s3.amazonaws.com/users/42/report.pdf
  → Access Denied (bucket is private)

Pre-signed URL (generated by your backend):
  https://pulsebloom-private.s3.amazonaws.com/users/42/report.pdf
    ?X-Amz-Signature=...&X-Amz-Expires=300&...
  → Works for exactly 5 minutes, then expires
```

> This is the production-correct way to let users access their own private files — never by making the whole bucket public, even "just for one user's files."

---

## 6. A Real Breach Scenario, Walked Through (So You Internalize Why This Matters)

```
1. Developer creates an S3 bucket for "internal backups"
2. Developer disables Block Public Access "just to test something quickly"
3. Developer writes a Bucket Policy intended for one specific use case,
   but accidentally scopes it with Resource: "arn:aws:s3:::backups/*"
   (the wildcard * at the END means "everything in this bucket")
4. Developer forgets to revert the change
5. Months later, the bucket — containing real customer data — is
   discovered fully public by a security researcher or attacker
   scanning for open buckets (this is automated and constant on
   the internet — it's not a matter of "if," it's "when")
```

```
The fix, applied at every layer:
  Block Public Access: stayed ON (would have stopped this entirely)
  Bucket Policy: scoped Resource to a specific narrow prefix, not "/*"
  Regular audits: periodically reviewing bucket policies for drift
```

---

## 7. Interview Questions (Practice Explaining These Out Loud)

**Q1: What's the difference between an IAM Policy and a Bucket Policy?**
> An IAM Policy is attached to a user, group, or role and defines what that identity can do across AWS. A Bucket Policy is attached directly to a bucket and defines what any principal (including the public, if specified) can do to that specific bucket.

**Q2: What does Block Public Access do, and why is it enabled by default?**
> It's an account/bucket-level safety override that blocks any policy or ACL from making content public, even accidentally. It's on by default because misconfigured public buckets are one of the most common real-world causes of data breaches.

**Q3: How would you safely let a user download their own private file from a private S3 bucket?**
> Generate a pre-signed URL from the backend — a temporary, time-limited link granting access to that specific object, without making the bucket or object publicly accessible.

**Q4: If a Bucket Policy allows public access, but Block Public Access is enabled, what happens?**
> Access remains blocked. Block Public Access overrides any conflicting public-granting policy — it's specifically designed to protect against this kind of misconfiguration.

**Q5: Why would you scope a Bucket Policy's Resource to `bucket-name/public/*` instead of `bucket-name/*`?**
> To follow the Principle of Least Privilege — opening only the intended subset of content (e.g., one public folder) instead of accidentally exposing the entire bucket, including private data that might also live there.

---

## 8. Hands-On Assignment for Today

1. Go back to your Day 8 bucket.
2. Create a new "folder" (key prefix) called `public/` and upload one test image into it.
3. Go to **Bucket → Permissions → Bucket Policy**, and write a policy (using the example in Section 4) that allows public `GetObject` access **only** to the `public/*` prefix.
4. You'll need to **temporarily adjust Block Public Access** to allow this one specific policy — do this deliberately, understanding exactly what you're opening, and reconfirm everything else in the bucket stays private.
5. Test it: open the object's URL directly in an incognito browser window — it should load without AWS credentials.
6. Try accessing a file you uploaded yesterday (outside the `public/` folder) the same way — confirm it's still denied.
7. Write a short note in your own words covering:
   - The difference between IAM Policies and Bucket Policies
   - What Block Public Access protects against
   - When you'd use a pre-signed URL instead of a public Bucket Policy

---

## 9. Day 9 Goal — Can You Teach This?

By the end of today, you should be able to explain, without notes:

> *"S3 access is controlled by multiple layers — Block Public Access acts as a master safety switch, IAM Policies control what my own users/roles can do, and Bucket Policies control access to the bucket itself, including the public if explicitly allowed. For private user data, I'd never make a bucket public — I'd use pre-signed URLs to grant temporary, scoped access instead. Most real-world S3 breaches come from accidentally opening an entire bucket instead of narrowly scoping a policy."*

---

**Next up — Day 10: Upload Files from Node.js to S3**
