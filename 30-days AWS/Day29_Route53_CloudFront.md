# Day 29: Route 53 & CloudFront
### *(Custom Domains + Global Content Delivery — The Final Layer of a Production App)*

> **Roadmap reference:** Week 4, Day 29 — "Route 53, CloudFront"

---

## Why This Matters

Your app right now is accessible at an ugly AWS-generated URL:
```
http://my-production-alb-xxxxxxxxxx.ap-south-1.elb.amazonaws.com
```

No real product ships with that. Today you replace it with:
```
https://api.pulsebloom.com
```

And for static content (images, frontend assets, videos), instead of serving them from a single server in Mumbai, you deliver them from **the AWS Edge Location nearest to each user — worldwide** — making your app feel instant regardless of where users are.

These two services — **Route 53** (DNS) and **CloudFront** (CDN) — are the last mile of a production-grade AWS setup.

---

## PART 1: Route 53

---

## 1. What is Route 53?

**Amazon Route 53 is AWS's highly available, scalable DNS (Domain Name System) service — it translates human-readable domain names into IP addresses that computers use to connect.**

```
User types: https://api.pulsebloom.com
        ↓
Route 53 DNS lookup:
  "What is the IP/address of api.pulsebloom.com?"
        ↓
Route 53 answers:
  "It's this ALB DNS: my-alb-xxxx.elb.amazonaws.com"
        ↓
Browser connects to the ALB
        ↓
Your app responds
```

> **Why "Route 53"?** DNS runs on port 53. Classic AWS naming.

---

## 2. Core Route 53 Concepts

```
HOSTED ZONE
  → A container for DNS records for a specific domain
  → e.g., "pulsebloom.com" hosted zone
  → Contains all the records that define routing for that domain
  → Two types:
      Public hosted zone  → routes internet traffic (what you'll use)
      Private hosted zone → routes traffic only within a VPC

DNS RECORD
  → An individual routing rule inside a hosted zone
  → Tells DNS resolvers where to send traffic for a subdomain

RECORD TYPES (most important):
  A Record     → maps a domain to an IPv4 address
  AAAA Record  → maps a domain to an IPv6 address
  CNAME Record → maps a domain to another domain name (alias)
  ALIAS Record → AWS-specific, like CNAME but for root domains
                 and works with AWS resources (ALB, CloudFront, etc.)
  MX Record    → mail server routing
  TXT Record   → text data (domain verification, SPF records)
```

---

## 3. Registering a Domain with Route 53

```
Route 53 → Registered domains → Register domain

  Search for: pulsebloom.com (or any domain)
  → Select available domain
  → Enter contact details
  → Purchase (prices vary: .com ≈ $12/year, .in ≈ $3/year)

After registration (takes a few minutes):
  → Route 53 automatically creates a Hosted Zone for your domain
  → AWS generates 4 nameservers for your domain
  → If registering elsewhere (GoDaddy, Namecheap), update their
    nameservers to point to these Route 53 nameservers
```

> If you already own a domain elsewhere, just create a Hosted Zone in Route 53 and update your registrar's nameservers — you don't need to transfer the domain.

---

## 4. Creating DNS Records — Pointing Your Domain to AWS Resources

### Point api.pulsebloom.com → ALB

```
Route 53 → Hosted zones → pulsebloom.com → Create record

  Record name: api
  Record type: A
  Alias: ON  ← use Alias for AWS resources, not CNAME
  Route traffic to:
    → Alias to Application and Classic Load Balancer
    → Region: ap-south-1
    → Select your ALB: my-production-alb
  Routing policy: Simple routing
  → Create records
```

```
Result:
  api.pulsebloom.com → my-production-alb-xxxx.elb.amazonaws.com
                     → ALB → EC2/ECS → Your app
```

### Why ALIAS Instead of CNAME for AWS Resources?

```
CNAME:
  → Can't be used for root domain (pulsebloom.com can't be a CNAME)
  → Each DNS lookup charges a tiny fee
  → Standard DNS, works anywhere

ALIAS (AWS-specific):
  → Works for root domain (pulsebloom.com itself)
  → Free DNS queries to AWS resources
  → Automatically updates if ALB IP changes
  → Only works when pointing to AWS resources (ALB, CloudFront, S3, etc.)
  → ALWAYS use Alias when pointing to AWS resources
```

### Multiple Records — A Complete Domain Setup

```
Route 53 → pulsebloom.com Hosted Zone

Records:
  pulsebloom.com       A → ALIAS → CloudFront distribution (frontend)
  www.pulsebloom.com   A → ALIAS → CloudFront distribution (frontend)
  api.pulsebloom.com   A → ALIAS → ALB (Node.js backend)
  assets.pulsebloom.com CNAME → CloudFront distribution (S3 assets)
```

---

## 5. Route 53 Routing Policies — Beyond Simple DNS

Route 53 supports advanced routing that goes far beyond simple domain → IP mapping:

```
SIMPLE ROUTING
  → One record, one destination
  → Basic use case

WEIGHTED ROUTING
  → Split traffic by percentage
  → Use case: A/B testing, gradual rollouts
  → api.pulsebloom.com → 90% to prod, 10% to new version

LATENCY-BASED ROUTING
  → Route users to the region with lowest latency
  → Use case: multi-region deployments
  → Indian users → ap-south-1
  → US users     → us-east-1
  → EU users     → eu-west-1
  → All at the same domain — Route 53 picks the fastest region

FAILOVER ROUTING
  → Primary/Secondary setup
  → If primary health check fails → Route 53 automatically
    switches traffic to secondary
  → Use case: disaster recovery

GEOLOCATION ROUTING
  → Route based on user's geographic location
  → Use case: compliance (EU users must hit EU servers),
    language-specific versions
```

> **Latency-based routing** is the most powerful for global apps — users always hit the nearest region without any code changes. This is how apps like Zomato, Swiggy, and global SaaS products serve users from the closest data center automatically.

---

## 6. Health Checks in Route 53

Route 53 can monitor your endpoints and only route traffic to healthy ones:

```
Route 53 → Health checks → Create health check

  Monitor: Endpoint
  Protocol: HTTPS
  Domain name: api.pulsebloom.com
  Path: /health
  Interval: 30 seconds
  Failure threshold: 3

Now combine with Failover Routing:
  Primary record: api.pulsebloom.com → Mumbai ALB
                  health check: ap-south-1-health-check
  Secondary record: api.pulsebloom.com → Singapore ALB
                    failover: secondary

If Mumbai goes down:
  Health check fails
       ↓
  Route 53 detects within ~90 seconds
       ↓
  Automatically routes traffic to Singapore
       ↓
  Users experience minimal disruption
```

---

## PART 2: CloudFront

---

## 7. What is CloudFront?

**Amazon CloudFront is AWS's Content Delivery Network (CDN) — a global network of 600+ Edge Locations that caches and delivers your content from the location nearest to each user.**

```
WITHOUT CloudFront:
  User in London requests an image
        ↓
  Request travels to your S3 bucket in Mumbai
  Distance: ~7,000 km / ~150ms round trip
  For every single user request
        ↓
  Slow for global users

WITH CloudFront:
  User in London requests an image
        ↓
  CloudFront Edge Location in London checks cache
  Cache HIT → serves immediately from ~20km away → ~5ms
        ↓
  Cache MISS (first request) → fetches from Mumbai, caches it
  → Next user in London gets it from cache → ~5ms
```

---

## 8. CloudFront Core Concepts

```
DISTRIBUTION
  → A CloudFront deployment — your CDN configuration
  → Gets a domain: d1abc123.cloudfront.net
  → You point your custom domain at this

ORIGIN
  → Where CloudFront fetches content FROM (when cache is empty)
  → e.g., S3 bucket, ALB, EC2, API Gateway, custom HTTP server

BEHAVIOR
  → Rules defining HOW CloudFront handles requests
  → Path patterns, cache settings, HTTP methods allowed
  → e.g., /api/* → forward to ALB (no caching, dynamic)
         /assets/* → cache from S3 for 24 hours (static)

CACHE
  → Content stored at Edge Locations
  → TTL (Time To Live) controls how long it's cached
  → CloudFront serves cached content without hitting your origin

INVALIDATION
  → Manually clear cached content when you update files
  → e.g., after deploying new frontend build
  → Cost: first 1,000 invalidations/month free
```

---

## 9. CloudFront for S3 Static Assets

The most common CloudFront use case — serving images, CSS, JS faster:

### Step 1: Create a CloudFront Distribution for S3

```
CloudFront → Distributions → Create distribution

  Origin domain:
    → Select your S3 bucket (pulsebloom-uploads)

  Origin access:
    → Origin access control settings (recommended)
    → Create new OAC (Origin Access Control)
    → This lets CloudFront access private S3 without making bucket public

  Default cache behavior:
    Viewer protocol policy: Redirect HTTP to HTTPS
    Allowed HTTP methods: GET, HEAD (static files only)
    Cache policy: CachingOptimized (AWS managed — good default)

  Settings:
    Price class: Use all edge locations (best performance)
                 OR Use only North America, Europe (cheaper)
    Alternate domain name (CNAME): assets.pulsebloom.com
    Custom SSL certificate:
      → Request certificate from ACM (Certificate Manager)
         in us-east-1 ← MUST be us-east-1 for CloudFront SSL
      → Validate via DNS (Route 53 auto-validates if same account)
      → Select validated cert

  → Create distribution (takes 5-10 minutes to deploy globally)
```

### Step 2: Update S3 Bucket Policy for CloudFront OAC

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::pulsebloom-uploads/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::ACCOUNT:distribution/DIST_ID"
        }
      }
    }
  ]
}
```

### Step 3: Point Custom Domain to CloudFront

```
Route 53 → pulsebloom.com hosted zone → Create record

  Record name: assets
  Record type: A
  Alias: ON
  Route traffic to:
    → Alias to CloudFront distribution
    → Select your distribution
  → Create records
```

```
Result:
  assets.pulsebloom.com → CloudFront → S3 (private, secure)

  User in Mumbai: served from Mumbai Edge Location (~5ms)
  User in London: served from London Edge Location (~5ms)
  User in NYC:    served from NYC Edge Location (~5ms)
  All hitting the same CloudFront distribution
```

---

## 10. CloudFront for Your API (Dynamic Content)

CloudFront can also sit in front of your ALB — useful for HTTPS termination, DDoS protection, and global acceleration of API calls:

```
CloudFront → Create distribution

  Origins:
    Origin 1 (Static assets):
      Domain: pulsebloom-uploads.s3.amazonaws.com
      Path pattern: /assets/*
      Cache: 24 hours

    Origin 2 (API backend):
      Domain: my-production-alb.elb.amazonaws.com
      Path pattern: /api/*
      Cache: DISABLED (dynamic content, never cache)
      Forward headers: ALL (for auth tokens, etc.)
      Forward query strings: ALL

  Single CloudFront URL serves both static and dynamic:
    https://d1abc.cloudfront.net/assets/logo.png → S3 (cached)
    https://d1abc.cloudfront.net/api/users        → ALB (not cached)
```

---

## 11. Cache Invalidation — After Deploying New Assets

```bash
# After deploying a new frontend build, old assets may be cached
# Invalidate specific files or patterns:

aws cloudfront create-invalidation \
  --distribution-id YOUR_DISTRIBUTION_ID \
  --paths "/assets/*"

# Or invalidate everything (use sparingly — causes cache miss storm):
aws cloudfront create-invalidation \
  --distribution-id YOUR_DISTRIBUTION_ID \
  --paths "/*"
```

```
Best practice: Use versioned filenames instead of invalidations
  OLD: /assets/app.js          (invalidate on every deploy)
  NEW: /assets/app.abc123.js   (new hash = new URL = no invalidation needed)

Modern frontend build tools (Vite, Webpack) do this automatically.
```

---

## 12. The Complete Production Architecture — All 29 Days Combined

```
User: "I want to visit pulsebloom.com"
        ↓
ROUTE 53: "pulsebloom.com → CloudFront distribution"
        ↓
CLOUDFRONT EDGE LOCATION (nearest to user):
  /assets/* → S3 bucket (cached, fast, global)
  /api/*    → ALB (dynamic, not cached)
        ↓ (for API requests)
ALB (Day 18):
  Health-checked, load-balanced across instances
        ↓
ECS FARGATE TASKS (Day 27):
  Private subnets, containerized Node.js app
        ↓
RDS PostgreSQL (Day 12):
  Private subnet, Multi-AZ, automated backups
  ← S3 for file uploads (Day 10)
  ← Secrets Manager for credentials (Day 3/22)
  ← CloudWatch for monitoring (Day 20/21)
        ↓
Route 53 health check monitors the whole thing
If Mumbai ALB fails → failover to Singapore ALB automatically

DOMAIN: pulsebloom.com (Route 53)
SSL:    *.pulsebloom.com cert (ACM)
CDN:    assets.pulsebloom.com (CloudFront → S3)
API:    api.pulsebloom.com (Route 53 → ALB → ECS → RDS)
```

---

## 13. Interview Questions (Practice Explaining These Out Loud)

**Q1: What is Route 53 and what does it do?**
> AWS's DNS service — it translates domain names to IP addresses or AWS resource endpoints, with advanced routing capabilities like latency-based, weighted, failover, and geolocation routing for intelligent traffic management.

**Q2: What is the difference between an A record, CNAME, and ALIAS record?**
> An A record maps a domain directly to an IPv4 address. A CNAME maps a domain to another domain name — but can't be used on root domains. An AWS ALIAS record is like a CNAME but works for root domains and AWS resources (ALB, CloudFront), with free DNS queries and automatic IP tracking.

**Q3: What is CloudFront and how does it improve performance?**
> CloudFront is AWS's CDN — a global network of 600+ Edge Locations that cache content close to users. Instead of every request traveling to your origin server, CloudFront serves cached content from the nearest Edge Location, dramatically reducing latency for global users.

**Q4: What is the difference between an Origin and a Behavior in CloudFront?**
> An Origin is where CloudFront fetches content from when the cache is empty (S3, ALB, API Gateway). A Behavior is a rule defining how CloudFront handles requests matching a path pattern — which origin to use, whether to cache, what headers to forward, and for how long.

**Q5: Why must an ACM SSL certificate for CloudFront be created in us-east-1?**
> CloudFront is a global service processed from AWS's us-east-1 infrastructure — it only recognizes SSL certificates from the us-east-1 region's ACM, regardless of where your actual resources are deployed.

**Q6: What is latency-based routing in Route 53?**
> A routing policy that sends each user's request to the AWS region with the lowest latency for that user — not necessarily the geographically closest, but the fastest responding. Used in multi-region deployments so users worldwide automatically hit the best-performing region.

---

## 14. Hands-On Assignment for Today

1. **Create a Route 53 Hosted Zone** (or register a domain if you don't have one — `.in` domains are often ₹250/year).

2. **Create an A record** pointing `api.yourdomain.com` → your ALB (Alias record, not CNAME).

3. **Test DNS propagation:**
   ```bash
   nslookup api.yourdomain.com
   curl http://api.yourdomain.com/health
   ```

4. **Create a CloudFront distribution** for your S3 bucket from Day 8:
   - Origin: your S3 bucket with OAC
   - Update S3 bucket policy to allow CloudFront OAC

5. **Create a Route 53 record** pointing `assets.yourdomain.com` → your CloudFront distribution.

6. **Upload a test image** to S3 → access it via `assets.yourdomain.com/filename` → confirm it loads.

7. **Test cache behavior:**
   ```bash
   curl -I https://assets.yourdomain.com/test-image.jpg
   # Look for: X-Cache: Hit from cloudfront (after first request)
   ```

8. Write a short note covering:
   - Why you'd use an ALIAS record instead of CNAME for an ALB
   - Why CloudFront SSL certs must be in us-east-1
   - One scenario where you'd use latency-based routing over simple routing

---

## 15. Day 29 Goal — Can You Teach This?

By the end of today, you should be able to explain, without notes:

> *"Route 53 handles DNS — translating api.pulsebloom.com to my ALB using an ALIAS record (not CNAME), and enabling advanced routing like latency-based to serve global users from the nearest region. CloudFront is my CDN — static assets cached at 600+ Edge Locations worldwide so users in London get the same millisecond response as users in Mumbai. Together, they're the public face of my infrastructure: Route 53 directs traffic intelligently, CloudFront delivers content fast globally, and my ALB + ECS + RDS handle the dynamic backend."*

---

**Next up — Day 30: Final Revision, Mock Interview & Resume Update**
