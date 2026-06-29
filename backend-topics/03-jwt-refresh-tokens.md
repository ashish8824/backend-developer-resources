# JWT + Refresh Tokens

## What is JWT?

JWT (JSON Web Token) is a compact, URL-safe token format that represents claims (statements about a user) which can be cryptographically verified and trusted — because it's digitally signed.

After login, instead of creating a server-side session, the server issues a JWT that the client holds and presents on every request.

## Why JWT? (Stateless Authentication)

**Traditional session-based auth:**

```
Login -> Server creates session -> Session ID stored server-side (e.g. in memory/Redis)
Every request -> Server looks up session ID -> Validates
```

Problem: the server must store session state somewhere, and every request needs a lookup. Doesn't scale cleanly across multiple servers without a shared session store.

**JWT-based auth:**

```
Login -> Server creates JWT containing user info + signs it -> Sends token to client
Every request -> Client sends JWT -> Server verifies signature (no DB lookup needed)
```

The server doesn't need to store anything — the token itself carries verified claims. This is what "stateless" means.

## Structure of a JWT

Three parts separated by dots: `header.payload.signature`

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

### Header (base64url-encoded JSON)

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

`alg` = signing algorithm, `typ` = token type.

### Payload / Claims (base64url-encoded JSON)

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "iat": 1516239022,
  "exp": 1516242622
}
```

**Standard claims:**

| Claim | Meaning |
|---|---|
| `iss` | Issuer |
| `sub` | Subject — usually the user ID |
| `aud` | Audience — intended recipient |
| `exp` | Expiration time |
| `iat` | Issued at |
| `nbf` | Not before |

Custom claims (role, permissions, etc.) can be added on top of these.

### Signature

```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret)
```

The signature is what makes the token tamper-proof — if anyone modifies the payload, the signature no longer matches.

**Important:** the payload is *encoded*, not *encrypted*. Base64 is trivially reversible by anyone — never put sensitive data (passwords, SSNs) in a JWT payload. Anyone can decode and read it; they just can't forge a *valid* signature without the secret/private key.

## Signing Algorithms

### HS256 (HMAC + SHA-256) — Symmetric

The same secret key signs and verifies. Both sides need the same shared secret.

```
Server (sign) --- shared secret ---> Server (verify)
```

Good for: a single backend service that both issues and validates its own tokens.

### RS256 (RSA + SHA-256) — Asymmetric

A private key signs; the corresponding public key verifies.

```
Auth Server (private key, signs) -----> Other Services (public key, verify only)
```

Good for: microservices architectures where many services need to *verify* tokens but shouldn't be able to *issue* them. The public key can be distributed freely; the private key never leaves the auth server.

## Access Token vs Refresh Token

**Why two tokens instead of one?**

**Access Token**
- Short-lived (e.g. 15 minutes)
- Sent with every API request
- Contains user claims
- If stolen, the damage window is limited because it expires quickly

**Refresh Token**
- Long-lived (e.g. 7–30 days)
- Used only to obtain a new access token when the access token expires
- Stored more securely, sent rarely
- Tracked server-side (DB/Redis) so it **can** be revoked

**Flow:**

```
1. Login -> Server issues Access Token (15 min) + Refresh Token (7 days)
2. Client uses Access Token for API calls
3. Access Token expires after 15 min
4. Client sends Refresh Token to /refresh endpoint
5. Server validates Refresh Token -> issues new Access Token (+ optionally new Refresh Token)
6. Client continues using new Access Token
```

**Why not just one long-lived token?** If a long-lived token is stolen, the attacker has access for the entire duration, with no way to revoke it — since validation is stateless by design. Splitting the responsibility lets the access token stay fast/stateless while the refresh token (checked server-side) provides a revocation point.

## Where to Store Tokens on the Client

| Storage | XSS risk | CSRF risk | Notes |
|---|---|---|---|
| `localStorage` | High — readable by any JS, including injected scripts | Low | If an attacker achieves XSS, they can read the token directly |
| `httpOnly` Cookie | Low — JS can't read it | Higher — needs CSRF protection (CSRF tokens, `SameSite`) | Browser sends it automatically; malicious JS can't read it |
| In-memory (JS variable) | Lower — not persisted anywhere | Low | Most secure, but lost on page refresh, hurting UX |

**Common recommendation:** keep the access token in memory only, and store the refresh token in an `httpOnly`, `Secure`, `SameSite=Strict` cookie.

## Refresh Token Rotation

Security best practice: every time a refresh token is used, issue a brand-new refresh token and invalidate the old one.

```
Client sends Refresh Token A -> Server validates -> issues Access Token + new Refresh Token B -> invalidates Token A
```

**Why?** If a refresh token is stolen and used by an attacker, and the legitimate user later tries to use the same (now-invalidated) token, the server detects reuse of an already-rotated token — a strong signal of compromise — and can revoke the *entire token family* (all descendant tokens), forcing re-login. This is called **Refresh Token Reuse Detection**.

## Revocation — The Stateless Tradeoff

JWTs are stateless by design — that's the whole point. But that also means you can't simply "delete" an already-issued access token before it naturally expires; there's no central record to delete it from.

**Common solutions:**

1. Keep access tokens very short-lived, so the exposure window from an unrevoked token stays small, and put the real revocation power in the refresh token check (which **is** checked against a server-side store on every refresh call).
2. Maintain a small blocklist (e.g. a Redis set of revoked token IDs via the `jti` claim) for emergencies — "user reported account compromise" — and check it on each request just for that (typically small) revoked population. This reintroduces a bit of statefulness, but only as an exception path.

## Database / Redis Schema for Refresh Tokens

```
RefreshToken {
  token_id (jti)
  user_id
  token_hash       -- never store the raw token, store a hash
  expires_at
  revoked (boolean)
  replaced_by       -- tracks the rotation chain
  created_at
}
```

Storing a hash instead of the raw token means even a DB leak doesn't directly expose usable tokens.

## Java Implementation (using the `jjwt` library)

```java
import io.jsonwebtoken.*;
import io.jsonwebtoken.security.Keys;
import javax.crypto.SecretKey;
import java.util.Date;

public class JwtService {

    private final SecretKey secretKey = Keys.secretKeyFor(SignatureAlgorithm.HS256);
    private final long accessTokenValidityMs = 15 * 60 * 1000;            // 15 min
    private final long refreshTokenValidityMs = 7L * 24 * 60 * 60 * 1000; // 7 days

    public String generateAccessToken(String userId) {
        return Jwts.builder()
                .setSubject(userId)
                .claim("type", "access")
                .setIssuedAt(new Date())
                .setExpiration(new Date(System.currentTimeMillis() + accessTokenValidityMs))
                .signWith(secretKey)
                .compact();
    }

    public String generateRefreshToken(String userId) {
        return Jwts.builder()
                .setSubject(userId)
                .claim("type", "refresh")
                .setIssuedAt(new Date())
                .setExpiration(new Date(System.currentTimeMillis() + refreshTokenValidityMs))
                .signWith(secretKey)
                .compact();
    }

    public Claims validateToken(String token) {
        try {
            return Jwts.parserBuilder()
                    .setSigningKey(secretKey)
                    .build()
                    .parseClaimsJws(token)
                    .getBody();
        } catch (ExpiredJwtException e) {
            throw new RuntimeException("Token expired");
        } catch (JwtException e) {
            throw new RuntimeException("Invalid token");
        }
    }
}
```

**Refresh endpoint logic:**

```java
public TokenPair refresh(String refreshToken) {

    Claims claims = jwtService.validateToken(refreshToken);

    // Check against DB/Redis: is this refresh token revoked or already rotated?
    RefreshTokenRecord record = refreshTokenRepo.findByTokenHash(hash(refreshToken));

    if (record == null || record.isRevoked()) {
        throw new SecurityException("Refresh token reuse detected — revoking all sessions");
    }

    String userId = claims.getSubject();

    // Rotate: invalidate old, issue new
    record.setRevoked(true);
    refreshTokenRepo.save(record);

    String newAccessToken = jwtService.generateAccessToken(userId);
    String newRefreshToken = jwtService.generateRefreshToken(userId);

    refreshTokenRepo.save(new RefreshTokenRecord(newRefreshToken, userId /* ... */));

    return new TokenPair(newAccessToken, newRefreshToken);
}
```

## Production Architecture

```
Client
  |
Login -> Auth Service --(RS256 private key signs)--> Access Token + Refresh Token
  |
API Gateway / Services --(RS256 public key verifies)--> validate Access Token, no DB call needed
  |
Refresh Endpoint -> checks Refresh Token against Redis/DB -> issues new pair
```

## Common Vulnerabilities

### 1. `"alg": "none"` attack

Some libraries historically accepted `alg: none` in the header, meaning "no signature verification needed" — letting attackers forge tokens trivially. **Fix:** always explicitly whitelist allowed algorithms server-side; never let the token's own header dictate how it gets verified.

### 2. Algorithm confusion (RS256 → HS256)

If a server accepts both RS256 and HS256, an attacker can take the (public, freely available) RSA public key and use it as an HMAC *secret* to forge an HS256-signed token that the server mistakenly accepts as valid. **Fix:** pin the expected algorithm explicitly per key; don't let the token's header choose the verification algorithm.

### 3. Weak or leaked secret (HS256)

If the HMAC secret is weak or leaked, anyone can forge tokens. **Fix:** use long random secrets, rotate them periodically, or use RS256 so the signing key never has to leave the auth server at all.

### 4. Storing sensitive data in the payload

The payload is base64-encoded, not encrypted — never put passwords, SSNs, or other sensitive fields in claims.

### 5. Long-lived access tokens with no revocation path

If access tokens live for days/weeks with no blocklist, a stolen token is unrevokable until natural expiry. **Fix:** short-lived access tokens + revocable refresh tokens.

## Interview Questions

**Q1: Why use JWT instead of traditional sessions?**

Stateless — no server-side session store needed, and tokens are verifiable across multiple services without a shared session-DB lookup on every request. Tradeoff: harder to revoke before natural expiry.

**Q2: Why have both Access and Refresh tokens instead of one?**

Access token: short-lived, used frequently, limited blast radius if stolen. Refresh token: long-lived but checked against a server-side store (Redis/DB), so it *can* be revoked — it's the bridge between a fully stateless approach (access token) and a fully revocable one (session-based auth).

**Q3: How do you handle logout with JWT, since you can't "delete" a token?**

- Refresh token: mark it revoked in the DB — this prevents minting new access tokens going forward.
- Access token: either let it expire naturally (short TTL), or maintain a small blocklist (by `jti`) for cases needing immediate revocation.

**Q4: What is Refresh Token Rotation and why does it matter?**

Issue a new refresh token every time the old one is used, invalidating the old one in the process. If an already-invalidated token is presented again, that's a strong signal of token theft (someone else already used it) — the server can revoke the entire token family and force re-login.

**Q5: HS256 vs RS256 — when would you choose each?**

HS256 (symmetric, one shared secret): simpler, fine when a single service both issues and verifies tokens. RS256 (asymmetric, private key signs / public key verifies): needed in microservices where many services must verify tokens but only the auth service should be able to issue them — the public key can be distributed freely without risk.

**Q6: Where should you store the JWT on the frontend?**

Avoid `localStorage` if possible — it's vulnerable to XSS, since any injected script can read it. Prefer an `httpOnly`, `Secure`, `SameSite` cookie for the refresh token (JS can't read it, mitigating XSS) combined with CSRF protections, and keep the access token in memory only.
