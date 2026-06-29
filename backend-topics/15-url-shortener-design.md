# URL Shortener Design

## What is a URL Shortener?

A URL shortener takes a long URL and generates a short alias for it. When a user visits the short URL, they are redirected to the original long URL.

```
Input:  https://www.example.com/blog/posts/how-to-design-url-shortener-system-2025
Output: https://short.ly/abc123

User visits: https://short.ly/abc123
Redirect:    -> https://www.example.com/blog/posts/how-to-design-url-shortener-system-2025
```

Real examples: bit.ly, tinyurl.com, t.co (Twitter), goo.gl

## Requirements Clarification (Interview Step 1)

Always clarify requirements before designing. Ask:

### Functional Requirements

- Shorten a URL: given a long URL, generate a short unique alias
- Redirect: given the short URL, redirect to the original long URL
- Custom aliases: allow users to specify their own short code (optional)
- Expiry: URLs can expire after a configured time (optional)
- Analytics: track click count, geolocation, device type (optional)

### Non-Functional Requirements

- High availability: redirect must always work (99.99% uptime)
- Low latency: redirect should be fast — < 10ms
- Scalability: handle high read traffic (redirects >> writes)
- Durability: short URLs should not be lost

### Scale Estimation

```
Write (new URLs):    100 million URLs/month
                   = 100M / (30 * 24 * 3600)
                   = ~40 URLs/second

Read (redirects):    read:write ratio = 100:1
                   = 40 * 100
                   = 4,000 redirects/second (peak: 10x = 40,000/sec)

Storage (10 years):
  100M URLs/month * 12 months * 10 years = 12 billion URLs
  Each record ~500 bytes
  Total: 12B * 500 = ~6TB

Cache:
  80% of traffic hits 20% of URLs (Pareto principle)
  Cache the hot 20%: 4,000 redirects/sec * 0.2 = cache ~800 hot URLs per second
```

## Short Code Generation

This is the most important design decision. The short code must be:

- Globally unique (no two URLs get the same code)
- Short (6–8 characters)
- URL-safe (no special characters)

### Option 1: Random Base62 Encoding

Base62 uses characters: `[A-Z] [a-z] [0-9]` = 62 characters

```
6 characters of base62: 62^6 = 56 billion combinations
7 characters of base62: 62^7 = 3.5 trillion combinations
```

6 characters is more than enough for billions of URLs.

**Algorithm:**

```java
public String generateShortCode() {
    String chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789";
    SecureRandom random = new SecureRandom();
    StringBuilder code = new StringBuilder(7);

    for (int i = 0; i < 7; i++) {
        code.append(chars.charAt(random.nextInt(chars.length())));
    }
    return code.toString();  // e.g. "aB3xZ7q"
}
```

**Collision handling:** check if the generated code already exists in the DB. If yes, regenerate. At low fill rate (we have 3.5 trillion slots for 12 billion URLs), collision probability is very low.

**Problem at scale:** as the DB fills up, collision probability rises and may require multiple retries.

### Option 2: MD5/SHA256 Hash of the Long URL

```java
public String hashUrl(String longUrl) {
    MessageDigest md = MessageDigest.getInstance("MD5");
    byte[] hash = md.digest(longUrl.getBytes(StandardCharsets.UTF_8));

    // Take first 43 bits -> 7 base62 characters
    // (or encode full hash and take first 7 chars)
    String base64 = Base64.getUrlEncoder().encodeToString(hash);
    return base64.substring(0, 7);  // take first 7 chars
}
```

**Problem:** same URL always generates same code (good for deduplication, but hash collisions across different URLs are possible). Two different long URLs could produce the same 7-char prefix — need collision handling.

### Option 3: Auto-Increment ID + Base62 Encoding (Recommended)

Use a DB auto-increment ID, then encode it to base62. This is deterministic, unique, and requires no collision check.

```
ID 1        -> base62 -> "000001"
ID 12345678 -> base62 -> "FpPkn"
ID 56800235584 -> base62 -> "aaaaaaa"
```

**Base62 encoding:**

```java
public class Base62Encoder {

    private static final String CHARS =
            "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789";

    public String encode(long id) {
        StringBuilder result = new StringBuilder();

        while (id > 0) {
            result.insert(0, CHARS.charAt((int)(id % 62)));
            id /= 62;
        }

        // Pad to 7 characters
        while (result.length() < 7) {
            result.insert(0, 'A');
        }

        return result.toString();
    }

    public long decode(String code) {
        long result = 0;
        for (char c : code.toCharArray()) {
            result = result * 62 + CHARS.indexOf(c);
        }
        return result;
    }
}
```

**Distributed ID generation:** auto-increment on a single DB is a bottleneck at scale. Use a distributed ID generator:

- **Snowflake ID** — 64-bit time-sortable unique ID (see Database Sharding notes)
- **DB range pre-allocation** — a dedicated ID service pre-allocates ranges (e.g. server 1 gets IDs 1–10000, server 2 gets 10001–20000) — each server hands out IDs from its range without hitting the DB every time

## Database Schema

```sql
-- URL table
CREATE TABLE urls (
    id              BIGSERIAL PRIMARY KEY,
    short_code      VARCHAR(10) UNIQUE NOT NULL,
    long_url        TEXT NOT NULL,
    user_id         BIGINT,                      -- nullable (anonymous)
    created_at      TIMESTAMP DEFAULT NOW(),
    expires_at      TIMESTAMP,                   -- NULL = never expires
    click_count     BIGINT DEFAULT 0
);

CREATE UNIQUE INDEX idx_short_code ON urls(short_code);
CREATE INDEX idx_user_id ON urls(user_id);       -- "my URLs" queries
CREATE INDEX idx_expires_at ON urls(expires_at); -- cleanup expired URLs

-- Analytics table (optional, separate service)
CREATE TABLE url_clicks (
    id          BIGSERIAL PRIMARY KEY,
    short_code  VARCHAR(10) NOT NULL,
    clicked_at  TIMESTAMP DEFAULT NOW(),
    ip_address  VARCHAR(45),
    user_agent  TEXT,
    country     VARCHAR(2),
    referrer    TEXT
);
```

## Redirect Implementation

### HTTP Redirect Types

```
301 Moved Permanently:
  - Browser caches the redirect
  - Future visits go directly to long URL (bypass our server)
  - Pro: reduces server load
  - Con: can't track clicks accurately; can't update the redirect

302 Found (Temporary Redirect):
  - Browser does NOT cache the redirect
  - Every visit goes through our server
  - Pro: accurate click tracking; can update/expire URLs
  - Con: more server load (every click hits us)
```

For analytics and ability to update/expire links: use **302**.
For performance (if analytics not needed): use **301**.

### Spring Boot Implementation

```java
@RestController
public class RedirectController {

    @Autowired
    private UrlService urlService;

    @Autowired
    private RedisTemplate<String, String> redis;

    @GetMapping("/{shortCode}")
    public ResponseEntity<Void> redirect(
            @PathVariable String shortCode,
            HttpServletRequest request) {

        // 1. Check Redis cache first
        String longUrl = redis.opsForValue().get("url:" + shortCode);

        if (longUrl == null) {
            // 2. Cache miss — query DB
            UrlRecord record = urlService.findByShortCode(shortCode)
                    .orElseThrow(() -> new UrlNotFoundException(shortCode));

            // Check expiry
            if (record.getExpiresAt() != null &&
                    record.getExpiresAt().isBefore(LocalDateTime.now())) {
                throw new UrlExpiredException(shortCode);
            }

            longUrl = record.getLongUrl();

            // 3. Cache for 24 hours
            redis.opsForValue().set("url:" + shortCode, longUrl, Duration.ofHours(24));
        }

        // 4. Async click tracking (don't block the redirect)
        String ip = request.getRemoteAddr();
        String userAgent = request.getHeader("User-Agent");
        CompletableFuture.runAsync(() ->
                urlService.trackClick(shortCode, ip, userAgent));

        // 5. Redirect
        return ResponseEntity.status(HttpStatus.FOUND)
                .location(URI.create(longUrl))
                .build();
    }
}
```

### URL Shortening

```java
@Service
public class UrlService {

    @Autowired
    private UrlRepository urlRepo;

    @Autowired
    private Base62Encoder encoder;

    @Autowired
    private RedisTemplate<String, String> redis;

    public String shortenUrl(String longUrl, String customCode, Long userId) {

        // Validate URL
        if (!isValidUrl(longUrl)) {
            throw new InvalidUrlException(longUrl);
        }

        // Check for duplicate long URL (optional deduplication)
        Optional<UrlRecord> existing = urlRepo.findByLongUrl(longUrl);
        if (existing.isPresent() && customCode == null) {
            return existing.get().getShortCode();
        }

        String shortCode;

        if (customCode != null) {
            // Custom alias
            if (urlRepo.existsByShortCode(customCode)) {
                throw new CodeAlreadyExistsException(customCode);
            }
            shortCode = customCode;
        } else {
            // Auto-generate: insert first to get auto-increment ID, then encode
            UrlRecord record = new UrlRecord();
            record.setLongUrl(longUrl);
            record.setUserId(userId);
            record = urlRepo.save(record);

            shortCode = encoder.encode(record.getId());
            record.setShortCode(shortCode);
            urlRepo.save(record);
            return shortCode;
        }

        UrlRecord record = new UrlRecord(shortCode, longUrl, userId);
        urlRepo.save(record);

        // Pre-warm cache
        redis.opsForValue().set("url:" + shortCode, longUrl, Duration.ofHours(24));

        return shortCode;
    }

    public void trackClick(String shortCode, String ip, String userAgent) {
        // Increment counter in Redis (fast, async)
        redis.opsForValue().increment("clicks:" + shortCode);

        // Async write to analytics DB
        UrlClick click = new UrlClick(shortCode, ip, userAgent, LocalDateTime.now());
        clickRepo.save(click);
    }
}
```

## Complete System Design

### Architecture Diagram

```
                    [DNS: short.ly]
                         |
                   [CDN/Edge Cache]     <- cache hot redirects at edge (lowest latency)
                         |
                  [Load Balancer]
                  /      |      \
           [App 1]  [App 2]  [App 3]   <- stateless API servers
                         |
              [Redis Cluster]           <- hot URL cache, click counters
                         |
              [Database Cluster]
              Primary (writes)
              Replicas (reads)
                         |
            [Analytics Service]         <- async click processing
                         |
            [Analytics DB (ClickHouse / Redshift)]
```

### Read Path (Redirect)

```
1. User visits short.ly/abc123
2. DNS resolves to load balancer
3. Load balancer routes to nearest app server
4. App checks Redis cache: "url:abc123"
   HIT  -> 302 redirect to long URL (< 5ms)
   MISS -> query DB read replica -> cache in Redis -> 302 redirect (< 30ms)
5. Async: increment click counter in Redis
6. Background: flush click counters to analytics DB periodically
```

### Write Path (Shorten URL)

```
1. POST /shorten { longUrl: "...", customCode: "..." }
2. Validate URL (format check, optionally check if accessible)
3. Generate short code (base62 of auto-increment ID)
4. Insert into DB primary
5. Pre-warm Redis cache
6. Return short URL
```

## Handling Scale Challenges

### High Read Traffic

- **Redis cache** in front of DB — most redirects served from memory
- **CDN edge caching** — 301 redirects cached at CDN level (closest to user)
- **Read replicas** — redirect queries go to DB replicas, not the primary
- **Horizontal scaling** — stateless app servers scale infinitely behind the load balancer

### High Write Traffic

- **Pre-allocate ID ranges** — each server gets a batch of IDs (e.g. 1000 IDs), uses them without hitting the DB each time
- **Async analytics** — click tracking never blocks the redirect — written asynchronously to Redis and batched to analytics DB

### URL Expiry and Cleanup

```java
// Scheduled job — run nightly
@Scheduled(cron = "0 0 2 * * *")  // 2 AM every day
public void cleanupExpiredUrls() {
    LocalDateTime now = LocalDateTime.now();
    int deleted = urlRepo.deleteByExpiresAtBefore(now);
    log.info("Cleaned up {} expired URLs", deleted);
}
```

Also remove expired URLs from Redis:

```java
// When serving a redirect, check expiry — if expired, delete from cache too
if (record.getExpiresAt() != null && record.getExpiresAt().isBefore(LocalDateTime.now())) {
    redis.delete("url:" + shortCode);
    throw new UrlExpiredException(shortCode);
}
```

## API Design

```
POST   /api/v1/urls
Body:  { "longUrl": "https://...", "customCode": "mylink", "expiresIn": 86400 }
Response: { "shortUrl": "https://short.ly/mylink", "shortCode": "mylink" }

GET    /{shortCode}
Response: 302 Redirect to long URL (or 404 if not found, 410 if expired)

GET    /api/v1/urls/{shortCode}
Response: { "shortCode": "abc123", "longUrl": "...", "clickCount": 1500, "createdAt": "..." }

DELETE /api/v1/urls/{shortCode}
Response: 204 No Content

GET    /api/v1/urls/{shortCode}/analytics
Response: { "totalClicks": 1500, "clicksByCountry": {...}, "clicksByDay": [...] }
```

## Security Considerations

### Malicious URL Protection

Block known malicious URLs using Google Safe Browsing API or similar:

```java
public boolean isMaliciousUrl(String url) {
    // Check against Google Safe Browsing API
    return safeBrowsingService.check(url);
}
```

### Rate Limiting on URL Creation

Prevent spam/abuse — limit anonymous users to 10 URLs/hour, registered users to 1000/hour.

### Custom Code Squatting

Reserve common words (`admin`, `api`, `health`, `login`) to prevent them from being used as custom short codes.

## Interview Follow-Up Questions

**Q1: How would you design the URL shortening to be collision-free at scale?**

Use auto-increment IDs encoded to base62. Each URL gets a guaranteed unique integer ID from the database; encoding to base62 produces a compact, unique short code. No collision checking needed. For distributed scale, use a pre-allocated ID range service (each server gets a batch of IDs) or Snowflake IDs to avoid single-DB bottleneck.

**Q2: Should you use 301 or 302 for redirects?**

302 for a real URL shortener — it lets the server count clicks accurately and allows URLs to be updated or expired. 301 is cached by the browser, so you lose click visibility and can't update the destination. Use 301 only if analytics are unnecessary and you want to minimize server load permanently.

**Q3: How does caching work in the redirect path?**

Hot URLs (top 20% serving 80% of traffic) are cached in Redis with a 24-hour TTL. The app checks Redis before hitting the DB — most redirects are served in < 5ms from memory. For the coldest URLs, the DB read replica is queried and the result is cached. A CDN layer can further cache 302 responses at the edge for globally distributed users.

**Q4: How do you handle analytics without slowing down redirects?**

Track clicks asynchronously — the redirect returns immediately, and click tracking happens in a background thread or via an async message queue (Kafka). Click counters are incremented in Redis (very fast, atomic `INCR`) and periodically flushed to an analytics database (ClickHouse, Redshift) in batch. Never do synchronous analytics writes in the redirect path.

**Q5: How do you prevent the same long URL from generating multiple short codes?**

Optionally, add a unique index on `long_url` and check for existing entries before creating a new short code. If the URL already exists, return the existing short code. Tradeoff: if the same long URL should produce different short codes per user (for attribution), don't deduplicate — make it user-scoped (`UNIQUE (long_url, user_id)`).

**Q6: How would you scale this to handle 100,000 redirects per second?**

Multiple layers: stateless app servers (scale horizontally behind load balancer), Redis cluster caching hot URLs (cache hit rate 95%+), DB read replicas for cache misses, CDN edge caching for globally popular URLs (301 or short-TTL cached 302), async click tracking via Kafka/Redis so analytics never touch the redirect path, and pre-allocated ID ranges to eliminate DB write contention on URL creation.
