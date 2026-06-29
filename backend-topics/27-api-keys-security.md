# API Keys & API Security

## What is an API Key?

An API Key is a unique identifier (token) issued to a client application to authenticate and authorize API access. Unlike user authentication (JWT/Session), API keys identify the application or developer calling your API — not the end user.

```
Without API Key:
Anyone can call your API -> abuse, overload, no accountability

With API Key:
Client sends: GET /api/products
              X-API-Key: sk_live_abc123xyz789...

Server:
  -> Looks up key in DB/Redis
  -> Identifies: key belongs to "Acme Corp", plan: PREMIUM, limit: 10,000/day
  -> Checks rate limit for this key
  -> Serves request and logs usage
```

## API Key vs JWT vs OAuth

| Feature      | API Key                         | JWT                 | OAuth 2.0                      |
| ------------ | ------------------------------- | ------------------- | ------------------------------ |
| Identifies   | Application / developer         | User (end user)     | User (delegated)               |
| Best for     | B2B APIs, machine-to-machine    | User sessions, SPAs | Third-party delegated access   |
| Statefulness | Stateful (server must look up)  | Stateless           | Stateful (server checks token) |
| Revocation   | Instant (delete from store)     | Hard (until expiry) | Instant (revoke refresh token) |
| Expiry       | Usually long-lived or no expiry | Short-lived (15min) | Short-lived access token       |
| User context | None                            | Full user context   | Scoped user context            |

## API Key Design

### Key Format

```
YOUR_STRIPE_LIVE_SECRET_KEY <- production key
YOUR_STRIPE_TEST_SECRET_KEY  <- test/sandbox key

Anatomy:
sk_     = prefix identifies type (sk = secret key, pk = public key)
live_   = environment (live vs test)
<random> = 32+ cryptographically random characters
```

API keys are typically 32–64 character random strings using base62 or hex characters.

### Key Generation

```java
@Service
public class ApiKeyService {

    @Autowired
    private ApiKeyRepository apiKeyRepo;

    public ApiKeyResponse generateApiKey(Long userId, String name, String plan) {

        // Generate cryptographically secure random key
        byte[] randomBytes = new byte[32];
        new SecureRandom().nextBytes(randomBytes);
        String rawKey = "sk_live_" + Base64.getUrlEncoder()
                .withoutPadding()
                .encodeToString(randomBytes);

        // Store HASH of key (never store raw key — treat like a password)
        String keyHash = hashKey(rawKey);
        String keyPrefix = rawKey.substring(0, 12);  // store prefix for identification

        ApiKey apiKey = new ApiKey();
        apiKey.setUserId(userId);
        apiKey.setName(name);
        apiKey.setKeyHash(keyHash);
        apiKey.setKeyPrefix(keyPrefix);  // "sk_live_aB3x" — shown in UI for identification
        apiKey.setPlan(plan);
        apiKey.setCreatedAt(LocalDateTime.now());
        apiKeyRepo.save(apiKey);

        // Return raw key ONCE — never retrievable again
        return new ApiKeyResponse(rawKey, keyPrefix, "Key will only be shown once");
    }

    private String hashKey(String rawKey) {
        // Use SHA-256 for API keys (bcrypt is too slow for per-request lookup)
        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            byte[] hash = digest.digest(rawKey.getBytes(StandardCharsets.UTF_8));
            return HexFormat.of().formatHex(hash);
        } catch (NoSuchAlgorithmException e) {
            throw new RuntimeException(e);
        }
    }
}
```

**Critical:** never store raw API keys — store a SHA-256 hash. If your DB is compromised, raw keys can't be extracted. Users are shown the key only once on creation (like GitHub personal access tokens).

### Database Schema

```sql
CREATE TABLE api_keys (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT REFERENCES users(id),
    name        VARCHAR(100),           -- "Production Key", "Mobile App"
    key_hash    VARCHAR(64) UNIQUE,     -- SHA-256 hash of the raw key
    key_prefix  VARCHAR(12),            -- first 12 chars: "sk_live_aB3x" (for UI display)
    plan        VARCHAR(20),            -- FREE, STARTER, PREMIUM, ENTERPRISE
    rate_limit  INT DEFAULT 1000,       -- requests per hour
    scopes      TEXT[],                 -- ["read:products", "write:orders"]
    last_used_at TIMESTAMP,
    expires_at  TIMESTAMP,             -- NULL = never expires
    revoked     BOOLEAN DEFAULT FALSE,
    created_at  TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_api_keys_hash ON api_keys(key_hash);
CREATE INDEX idx_api_keys_user ON api_keys(user_id);
```

## API Key Authentication Filter

### Spring Boot

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class ApiKeyAuthFilter extends OncePerRequestFilter {

    @Autowired
    private ApiKeyRepository apiKeyRepo;

    @Autowired
    private StringRedisTemplate redis;

    @Autowired
    private RateLimiterService rateLimiter;

    private static final String API_KEY_HEADER = "X-API-Key";
    private static final List<String> PUBLIC_PATHS =
            List.of("/api/public", "/health", "/actuator");

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws IOException, ServletException {

        String path = request.getRequestURI();
        if (PUBLIC_PATHS.stream().anyMatch(path::startsWith)) {
            chain.doFilter(request, response);
            return;
        }

        String rawKey = request.getHeader(API_KEY_HEADER);
        if (rawKey == null || rawKey.isBlank()) {
            sendError(response, 401, "API key required. Include X-API-Key header.");
            return;
        }

        // 1. Hash the incoming key
        String keyHash = hashKey(rawKey);

        // 2. Look up in Redis cache first (avoid DB hit on every request)
        ApiKey apiKey = getApiKeyFromCache(keyHash);
        if (apiKey == null) {
            apiKey = apiKeyRepo.findByKeyHashAndRevokedFalse(keyHash).orElse(null);
            if (apiKey != null) {
                cacheApiKey(keyHash, apiKey);
            }
        }

        if (apiKey == null) {
            sendError(response, 401, "Invalid API key");
            return;
        }

        // 3. Check expiry
        if (apiKey.getExpiresAt() != null &&
                apiKey.getExpiresAt().isBefore(LocalDateTime.now())) {
            sendError(response, 401, "API key expired");
            return;
        }

        // 4. Check rate limit
        if (!rateLimiter.allowRequest("apikey:" + apiKey.getId(), apiKey.getRateLimit())) {
            response.setHeader("X-RateLimit-Limit", String.valueOf(apiKey.getRateLimit()));
            response.setHeader("Retry-After", "3600");
            sendError(response, 429, "Rate limit exceeded");
            return;
        }

        // 5. Check scope for this endpoint
        String requiredScope = resolveRequiredScope(request);
        if (requiredScope != null && !apiKey.getScopes().contains(requiredScope)) {
            sendError(response, 403, "Insufficient scope. Required: " + requiredScope);
            return;
        }

        // 6. Inject API key context into request for downstream use
        request.setAttribute("apiKeyId", apiKey.getId());
        request.setAttribute("userId", apiKey.getUserId());
        request.setAttribute("plan", apiKey.getPlan());

        // 7. Update last_used_at asynchronously (don't block request)
        CompletableFuture.runAsync(() ->
                apiKeyRepo.updateLastUsed(apiKey.getId(), LocalDateTime.now()));

        chain.doFilter(request, response);
    }

    private ApiKey getApiKeyFromCache(String keyHash) {
        String cached = redis.opsForValue().get("apikey:" + keyHash);
        return cached != null ? deserialize(cached, ApiKey.class) : null;
    }

    private void cacheApiKey(String keyHash, ApiKey apiKey) {
        redis.opsForValue().set(
                "apikey:" + keyHash,
                serialize(apiKey),
                Duration.ofMinutes(5)
        );
    }

    private void sendError(HttpServletResponse response, int status, String message)
            throws IOException {
        response.setStatus(status);
        response.setContentType("application/json");
        response.getWriter().write(
                "{\"error\": \"" + message + "\", \"status\": " + status + "}"
        );
    }
}
```

### Node.js Middleware

```javascript
const crypto = require("crypto");

function hashKey(rawKey) {
  return crypto.createHash("sha256").update(rawKey).digest("hex");
}

async function apiKeyMiddleware(req, res, next) {
  const rawKey = req.headers["x-api-key"];

  if (!rawKey) {
    return res.status(401).json({ error: "API key required" });
  }

  const keyHash = hashKey(rawKey);

  // Check Redis cache first
  let apiKey = null;
  const cached = await redis.get(`apikey:${keyHash}`);
  if (cached) {
    apiKey = JSON.parse(cached);
  } else {
    apiKey = await ApiKey.findOne({ keyHash, revoked: false });
    if (apiKey) {
      await redis.setex(`apikey:${keyHash}`, 300, JSON.stringify(apiKey));
    }
  }

  if (!apiKey) {
    return res.status(401).json({ error: "Invalid API key" });
  }

  if (apiKey.expiresAt && new Date(apiKey.expiresAt) < new Date()) {
    return res.status(401).json({ error: "API key expired" });
  }

  // Rate limiting
  const allowed = await rateLimiter.check(
    `apikey:${apiKey.id}`,
    apiKey.rateLimit,
  );
  if (!allowed) {
    return res.status(429).json({ error: "Rate limit exceeded" });
  }

  req.apiKey = apiKey;
  req.userId = apiKey.userId;
  next();
}
```

## API Key Best Practices

### 1. Key Rotation

Allow users to rotate (regenerate) keys without downtime:

```java
@Transactional
public ApiKeyResponse rotateApiKey(Long apiKeyId, Long userId) {
    ApiKey existing = apiKeyRepo.findByIdAndUserId(apiKeyId, userId)
            .orElseThrow(() -> new NotFoundException("API key not found"));

    // Revoke old key
    existing.setRevoked(true);
    existing.setRevokedAt(LocalDateTime.now());
    apiKeyRepo.save(existing);

    // Invalidate cache
    redis.delete("apikey:" + existing.getKeyHash());

    // Generate new key with same settings
    return generateApiKey(userId, existing.getName() + " (rotated)", existing.getPlan());
}
```

### 2. Key Scopes / Permissions

```
Scopes:
  read:products   -> can list and get products
  write:products  -> can create and update products
  read:orders     -> can list and get orders
  admin:all       -> full access

Key A (read-only): scopes = ["read:products", "read:orders"]
Key B (full):      scopes = ["read:products", "write:products", "read:orders", "write:orders"]
```

### 3. Key Expiry

```java
// Keys for CI/CD pipelines: no expiry
// Keys for partners: 1-year expiry, renewable
// Keys for temporary integrations: 30-day expiry
apiKey.setExpiresAt(LocalDateTime.now().plusDays(365));
```

### 4. Usage Analytics

```sql
CREATE TABLE api_key_usage (
    id          BIGSERIAL PRIMARY KEY,
    api_key_id  BIGINT REFERENCES api_keys(id),
    endpoint    VARCHAR(200),
    method      VARCHAR(10),
    status_code INT,
    response_ms INT,
    called_at   TIMESTAMP DEFAULT NOW()
);

-- Usage dashboard query
SELECT endpoint, COUNT(*) as calls, AVG(response_ms) as avg_ms,
       SUM(CASE WHEN status_code >= 400 THEN 1 ELSE 0 END) as errors
FROM api_key_usage
WHERE api_key_id = 42 AND called_at > NOW() - INTERVAL '24 hours'
GROUP BY endpoint
ORDER BY calls DESC;
```

### 5. Environment Separation

```
Production keys:   sk_live_...   -> live DB, real payments, real emails
Test/Sandbox keys: sk_test_...   -> test DB, simulated payments, no real emails
```

Both key types share the same API but route to different backends internally.

## General API Security Checklist

### Authentication & Authorization

- [ ] All endpoints require authentication (except explicitly public ones)
- [ ] Authorization checked at method level (not just route level)
- [ ] API keys hashed in storage (never raw)
- [ ] JWT secrets are long, random, rotated periodically
- [ ] OAuth scopes enforced per endpoint

### Transport Security

- [ ] HTTPS enforced everywhere (redirect HTTP to HTTPS)
- [ ] HSTS header: `Strict-Transport-Security: max-age=31536000; includeSubDomains`
- [ ] TLS 1.2+ only (disable older TLS versions)
- [ ] Certificates auto-renewed (Let's Encrypt / AWS ACM)

### Input Validation & Output

- [ ] All inputs validated and sanitized
- [ ] Parameterized queries (no SQL injection risk)
- [ ] Output encoded (no XSS risk)
- [ ] Error messages don't leak internals (stack traces, DB errors)

### Rate Limiting & DDoS

- [ ] Rate limiting on all public endpoints
- [ ] Stricter limits on auth endpoints (login, register, OTP)
- [ ] Request size limits (`Content-Length` max)
- [ ] WAF in front of API

### Security Headers (Helmet / Spring Security)

```javascript
// Node.js (helmet)
const helmet = require("helmet");
app.use(helmet());
// Sets: X-Content-Type-Options, X-Frame-Options, X-XSS-Protection,
//       Strict-Transport-Security, Content-Security-Policy, etc.
```

```java
// Spring Security
http.headers(headers -> headers
    .frameOptions(frame -> frame.deny())
    .xssProtection(xss -> xss.enable())
    .contentTypeOptions(Customizer.withDefaults())
    .httpStrictTransportSecurity(hsts -> hsts
        .maxAgeInSeconds(31536000)
        .includeSubDomains(true))
);
```

### Sensitive Data Exposure

- [ ] Never log API keys, passwords, or tokens
- [ ] Mask sensitive fields in responses (card last 4 only, not full number)
- [ ] Encrypt sensitive DB columns at rest
- [ ] No secrets in code/version control (use env vars / secrets manager)

## Interview Questions

**Q1: How should API keys be stored in a database?**

Never store raw API keys — store a SHA-256 hash (unlike passwords, bcrypt is too slow for per-request lookups). Show the raw key to the user only once on creation. Store a short prefix (first 8–12 chars) for display purposes in the UI ("Key ending in ...aB3x"). If the DB is compromised, attackers get hashes they can't reverse to usable keys.

**Q2: Why use SHA-256 for API key hashing instead of bcrypt?**

bcrypt is intentionally slow (100ms+) — great for passwords where you only verify once per login. API keys are verified on every request (potentially thousands per second). SHA-256 is fast (microseconds) and sufficient for API keys because they're already long random strings (high entropy), not dictionary words that need slow hashing to resist brute force.

**Q3: What is the difference between an API key and a JWT?**

An API key identifies an application/developer and is looked up server-side (stateful). A JWT identifies an end user, is self-contained and verified locally (stateless). API keys are long-lived and for B2B/machine-to-machine use; JWTs are short-lived and for user sessions. API keys are instantly revocable; JWTs are hard to revoke before expiry.

**Q4: What are API key scopes and why are they important?**

Scopes define what a specific API key is allowed to do (`read:products`, `write:orders`). Different keys can have different permission sets. A read-only integration key can't accidentally (or maliciously) modify data. Following the principle of least privilege — grant only what's needed for each specific integration.

**Q5: How do you handle key rotation without downtime for the API consumer?**

Support a grace period where both old and new keys are valid simultaneously (e.g. 24–48 hours). The flow: user generates new key, updates their integration, verifies it works, then revokes the old key. Some systems automatically support dual-key validation during the rotation window.
