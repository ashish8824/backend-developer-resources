# Day 3: IAM — Identity and Access Management
### *(Users, Groups, Roles & Policies — The Single Most Important AWS Service to Master)*

> **Roadmap reference:** Week 1, Day 3 — "IAM Users, Groups, Roles, Policies"

---

## Why This Matters

If you only deeply understand ONE AWS service before walking into a backend developer interview, make it **IAM**.

Here's why: every single AWS service you'll touch from here on — EC2, S3, RDS, Lambda — relies on IAM to decide *who is allowed to do what*. Misconfigured IAM is also the #1 cause of real-world AWS security incidents (leaked keys, over-permissioned roles, public S3 buckets). Get this right, and you've already cleared a huge chunk of both the security and interview bar.

---

## 1. What is IAM?

**IAM (Identity and Access Management) is the AWS service that controls who can access your AWS account and what they're allowed to do once inside.**

Think of IAM as the **security desk and ID-badge system** of a company building:
- It decides who gets a badge (Users)
- It decides which badges open which doors (Policies)
- It groups employees by department for easier badge management (Groups)
- It issues temporary visitor badges for contractors (Roles)

```
              ┌─────────────────────────┐
              │          IAM             │
              │   (Security & Access)    │
              └─────────────────────────┘
                          │
        ┌─────────┬───────┴───────┬────────────┐
        │         │               │            │
     USERS     GROUPS          ROLES        POLICIES
   (people)  (teams of      (temporary,    (the actual
              users)         assumable      permission
                              identities)     rules)
```

---

## 2. IAM Users

**An IAM User represents a single person (or application) that needs to access your AWS account.**

```
Root Account (the account owner — you, on signup)
        │
        ├── IAM User: "ashish-dev"     → for daily development work
        ├── IAM User: "ci-cd-bot"      → for GitHub Actions deployments
        └── IAM User: "readonly-audit" → for someone who only needs to view
```

### Critical Rule #1: Never Use the Root Account Day-to-Day

When you first create an AWS account, you get a **Root User** with unrestricted access to everything — billing, deleting resources, all of it. The very first thing every AWS course (and every real company) tells you:

```
DO:
  Root account → Used ONLY to create your first IAM user, then locked away
  IAM User    → Used for all daily work, with only the permissions you need

DON'T:
  Use Root account credentials for daily development
  Share Root account credentials with a team
  Embed Root account keys in code
```

> **Real-world consequence:** If your Root credentials leak (e.g., accidentally committed to a public GitHub repo), an attacker has *complete* control of your AWS account — billing, deletion, everything. If an IAM User's limited credentials leak instead, the damage is contained to whatever that user was permitted to do.

---

## 3. IAM Groups

**A Group is a collection of IAM Users that share the same permissions — letting you manage access for many people at once instead of one by one.**

```
Group: "Backend-Developers"
   ├── User: ashish
   ├── User: rahul
   └── User: priya

   → All three automatically get the same EC2/S3/RDS permissions
     attached to this group, in one place.
```

### Why Groups Matter (Real Example)

Without groups:
```
New developer joins the team
        ↓
You manually attach 6 different policies to their individual user
        ↓
Repeat this process for every new hire
        ↓
Easy to forget one, easy to create inconsistent permissions
```

With groups:
```
New developer joins the team
        ↓
You add them to the "Backend-Developers" group
        ↓
They instantly inherit all the correct permissions
        ↓
One consistent, auditable source of truth
```

---

## 4. IAM Roles

**A Role is a temporary identity that can be "assumed" by a user, application, or AWS service — without permanent long-term credentials.**

This is the part beginners find most confusing, so let's be very precise:

```
IAM USER  → Has permanent credentials, tied to a specific person/app
IAM ROLE  → Has NO permanent credentials, temporarily "assumed" by
            whoever/whatever needs it, then expires
```

### Why Roles Exist — The Problem They Solve

Imagine your EC2 instance needs to read files from an S3 bucket.

```
BAD APPROACH (don't do this):
  Create an IAM User
  Generate Access Key + Secret Key
  Hardcode those keys into your EC2 application code
  → If your code or repo ever leaks, those keys are exposed FOREVER
    (until you manually rotate them)

GOOD APPROACH (the AWS-recommended way):
  Create an IAM Role with S3 read permissions
  Attach that Role directly to the EC2 instance
  → AWS automatically provides temporary, auto-rotating credentials
  → Nothing is hardcoded, nothing to leak
```

```
EC2 Instance
     │
     │  "I need to read from S3"
     ▼
IAM Role (attached to instance)
     │
     │  AWS automatically issues short-lived temporary credentials
     ▼
S3 Bucket → access granted, no hardcoded secrets anywhere
```

### Common Real-World Use Cases for Roles

| Scenario | Role Used For |
|---|---|
| EC2 needs to read/write S3 | EC2 Instance Role |
| Lambda needs to write to DynamoDB | Lambda Execution Role |
| ECS Fargate task needs Secrets Manager access | ECS Task Role |
| Another AWS account needs temporary access to yours | Cross-Account Role |

> **Direct connection to your own work:** If you've deployed PulseBloom on ECS Fargate, the task that pulls secrets from Secrets Manager isn't using a hardcoded key — it's almost certainly assuming an **IAM Task Role** with exactly the permissions it needs, nothing more.

---

## 5. IAM Policies

**A Policy is a JSON document that defines exactly what actions are allowed or denied, on which resources.**

This is the actual "rulebook" — Users, Groups, and Roles don't have permissions by themselves; permissions only exist because a **Policy** is attached to them.

### Anatomy of a Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::pulsebloom-uploads/*"
    }
  ]
}
```

Breaking this down:

| Field | Meaning |
|---|---|
| `Effect` | `Allow` or `Deny` |
| `Action` | The specific operation (e.g., `s3:GetObject` = read a file from S3) |
| `Resource` | The exact AWS resource this applies to (here, one specific S3 bucket) |

```
Plain English translation:
"Allow this identity to read (GetObject) files from the
 pulsebloom-uploads S3 bucket — and nothing else."
```

### The Golden Rule: Principle of Least Privilege

**Always grant the minimum permissions needed to do the job — nothing more.**

```
BAD POLICY (way too broad):
  "Action": "*"
  "Resource": "*"
  → This identity can do ANYTHING to ANY resource in your account

GOOD POLICY (scoped tightly):
  "Action": "s3:GetObject"
  "Resource": "arn:aws:s3:::pulsebloom-uploads/*"
  → This identity can ONLY read files from ONE specific bucket
```

> **Why this matters in practice:** If credentials with `"*"`/`"*"` permissions ever leak, an attacker can delete your entire AWS account's resources. If tightly-scoped credentials leak, the damage is contained to exactly what that policy allowed — often nothing serious.

---

## 6. How It All Connects

```
        POLICY (the rulebook — what's allowed)
            │
            │ attached to
            ▼
   ┌────────┴────────┬─────────────┐
   │                  │             │
  USER              GROUP         ROLE
(a person)      (team of users) (temporary identity
                                  for apps/services)
```

A policy can be attached directly to a User, attached to a Group (so all members inherit it), or attached to a Role (so anything that assumes the role gets it).

---

## 7. Connecting This to Real Backend Development

Here's how a typical real deployment uses all four IAM concepts together:

```
1. You (developer) → IAM User "ashish-dev"
                       with a Policy allowing EC2/S3/RDS access
                       for development

2. Your CI/CD pipeline (GitHub Actions) → IAM User "ci-cd-bot"
                       with a Policy scoped ONLY to deploying
                       to ECS — nothing else

3. Your EC2/ECS application → IAM Role
                       with a Policy allowing it to read
                       Secrets Manager + write to S3

4. Your whole dev team → IAM Group "Backend-Developers"
                       so permissions stay consistent as
                       people join/leave
```

---

## 8. Interview Questions (Practice Explaining These Out Loud)

**Q1: What is IAM?**
> The AWS service that manages identities (users, roles) and controls what actions they're permitted to take on which resources, via policies.

**Q2: What's the difference between an IAM User and an IAM Role?**
> A User has permanent, long-term credentials tied to a specific person or application. A Role has no permanent credentials — it's temporarily assumed by a user, application, or AWS service, and the credentials auto-expire.

**Q3: Why would you use a Role instead of hardcoding access keys in your application?**
> Roles provide temporary, auto-rotating credentials with no risk of a permanently leaked secret — far more secure than embedding long-lived access keys directly in code.

**Q4: What is the Principle of Least Privilege?**
> Granting an identity only the exact permissions it needs to perform its job — nothing broader — so that if credentials are ever compromised, the damage is limited.

**Q5: What's the purpose of IAM Groups?**
> To manage permissions for multiple users at once, by attaching policies to the group rather than to each user individually — keeping access consistent and easier to audit.

**Q6: Why shouldn't you use the AWS Root account for daily work?**
> The Root account has unrestricted access to everything, including billing and account deletion. If its credentials leak, the entire account is compromised. IAM Users with scoped permissions limit that blast radius.

---

## 9. Hands-On Assignment for Today

1. Go to **AWS Console → IAM**.
2. **Create a new IAM User** for yourself (e.g., `ashish-dev`) — don't use Root for anything going forward.
3. **Create a Group** called `Backend-Developers` and attach an existing AWS-managed policy like `AmazonS3ReadOnlyAccess` to it.
4. **Add your new user to that group.**
5. Browse to **IAM → Roles** and look at any AWS-suggested role templates (e.g., "EC2 role for S3 access") — just read through one, don't create it yet (you'll do this hands-on when you reach EC2 on Day 4).
6. Write a short note in your own words covering:
   - The difference between a User and a Role
   - Why you'd use a Group instead of attaching policies one-by-one
   - One real example of a Policy and what it allows

---

## 10. Day 3 Goal — Can You Teach This?

By the end of today, you should be able to explain, without notes:

> *"IAM controls who can access AWS and what they're allowed to do. Users are for people, Groups make managing many users easier, Roles are temporary identities assumed by apps or services instead of hardcoding credentials, and Policies are the actual JSON rules that define exact permissions — always scoped as tightly as possible."*

---

**Next up — Day 4: EC2 Introduction, Launch an Ubuntu Instance, Connect via SSH**
