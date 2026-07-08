# Module 20 — Security in Distributed Systems

> **Masterclass:** System Design Masterclass (30 Modules)
> **Level:** Advanced
> **Audience:** Node.js backend developers, SDE‑2 / Senior Backend interview candidates, engineers transitioning into architecture roles
> **Prerequisite:** Modules 1–19 (System Design Intro through Observability)

---

## 1. Introduction

Nineteen modules have each deferred security with a brief callout: Module 3's "never expose the database port," Module 6's "never make a bucket public," Module 9's "strip client-supplied trust headers," Module 18's "fallbacks must not leak internal details," Module 19's "never log sensitive data in plaintext." Each of these was correct and specific — and none of them, individually, constitutes a security *architecture*. This module finally assembles them into one: **OAuth2** and **JWT** for authentication and authorization, **mTLS** for service-to-service trust, the **WAF** for perimeter defense, **secrets management** for credentials, and **IAM** for the access-control discipline tying all of it together.

The organizing principle for this entire module, stated upfront: security is not a layer you add at the end. Every module in this course has already made a security-relevant decision — Module 3's network isolation, Module 9's gateway trust boundary, Module 16's Database-per-Service — and this module's job is to name the *complete* discipline those decisions are all instances of: **defense in depth**, applied consistently and deliberately, not as an afterthought bolted onto a finished system.

---

## 2. Learning Objectives

By the end of this module, you will be able to:

1. Explain **authentication** versus **authorization** precisely, and where each decision belongs architecturally.
2. Explain **OAuth2** and its core flows, and why it exists as a *delegation* protocol, not merely a login mechanism.
3. Explain **JWT** structure and validation mechanically, completing Module 9's gateway-level JWT handling with full rigor.
4. Explain **mTLS** and why service-to-service authentication matters even inside a "trusted" private network (Module 3).
5. Explain the role of a **WAF (Web Application Firewall)** and what class of attacks it defends against that a network firewall (Module 3) does not.
6. Design a **secrets management** strategy that avoids the most common, damaging real-world credential-handling mistakes.
7. Apply **IAM (Identity and Access Management)** principles — least privilege, role-based access — consistently across a distributed system's many components.

---

## 3. Why This Concept Exists

Every module in this course has built a system that gets progressively more capable and more distributed — more services (Module 16), more databases (Module 15), more asynchronous communication paths (Module 11, 17), more caches (Module 7), more observability data (Module 19). Each new component is also a new potential entry point, a new place credentials might leak, a new boundary where "who is allowed to do this" must be decided correctly. Module 12 taught us that distributed systems fail in ambiguous, partial ways — security failures inherit this same characteristic, except the "failure" is often an adversary deliberately exploiting exactly that ambiguity, not a random fault.

Security in distributed systems exists as its own discipline because **a single strong lock on the front door means nothing if there are ten other doors, and the "trusted internal network" behind the front door is itself made of many independent, imperfect components** (Module 16's many microservices, each written and deployed by different teams). Authentication and authorization protocols, service-to-service trust mechanisms, and centralized secrets management all exist to make "who is this, and what are they allowed to do" a consistently, correctly answerable question — at every one of those many doors, not just the first one.

---

## 4. Problem Statement

> Our blog platform, now decomposed into Core Content, Search, Notification, and Recommendation services (Module 16), behind an API Gateway (Module 9), needs: (1) users to log in via their existing Google account rather than a new password (delegated authentication), (2) a way for the Recommendation Service to verify that a request genuinely came from the API Gateway and not an attacker who somehow gained network access to the private subnet, (3) protection against a newly-discovered SQL injection attempt targeting the search query parameter, and (4) a way to store the third-party email provider's API key (Module 18) without it ever appearing in source code or plaintext configuration files. Design the security architecture addressing all four requirements, naming the specific pattern for each.

---

## 5. Real-World Analogy

**Authentication is a bouncer checking your ID at the door — establishing who you are.** **Authorization is a separate decision, made by a different person entirely, about which rooms in the building your specific membership tier allows you into** — VIP members get the rooftop lounge, general members don't, even though both were equally, successfully authenticated at the door. Conflating these two decisions — assuming "the bouncer let them in" automatically means "they can go anywhere" — is a common, serious architectural mistake this module corrects precisely.

**OAuth2 is you letting a valet park your car by handing over a valet key, not your master key.** The valet key (an OAuth2 access token) lets the valet do exactly one thing — drive and park the car — without giving them access to your car's other master-key capabilities (opening the trunk with your golf clubs, glovebox documents). This is precisely why Google-login (Section 4's requirement 1) uses OAuth2: our blog platform gets a limited, scoped "valet key" proving the user authenticated with Google and granting exactly the specific permissions the user consented to (e.g., "read their email address"), never Google's actual master credentials.

**mTLS is every employee in a secure building wearing a badge that must be scanned and verified — by *both* parties — every time they enter a room, even a room deep inside the building that's already behind the main entrance's security check.** Section 4's requirement 2 — the Recommendation Service verifying a request genuinely came from the Gateway — exists precisely because "you're already inside the building" (the private subnet, Module 3) is not the same guarantee as "I've specifically verified your badge for *this* room" — an attacker who compromises *any* single machine inside the private network can otherwise impersonate any service, badge-free.

**A WAF is a security guard specifically trained to recognize forged documents and social-engineering scripts, standing at the door in addition to the regular ID-checking bouncer.** A regular bouncer (Module 3's network firewall) checks whether you're allowed through the door at all (IP/port level); a WAF (Section 4's requirement 3) inspects the *actual content* of what you're carrying in, looking for known attack patterns (like a SQL injection payload) that a simple "are you on the guest list" check would never catch.

---

## 6. Technical Definition

**Authentication:** The process of verifying that an entity (a user, service, or device) is who or what it claims to be.

**Authorization:** The process of determining whether an already-authenticated entity is permitted to perform a specific action or access a specific resource.

**OAuth2:** An authorization delegation framework allowing a user to grant a third-party application limited access to their resources on another service, without sharing their credentials directly.

**JWT (JSON Web Token):** A compact, self-contained, cryptographically-signed token format used to represent claims (such as identity and permissions) that can be verified without a database lookup, by validating the signature alone.

**mTLS (Mutual TLS):** An extension of standard TLS (Module 4) in which *both* the client and server present and verify certificates, establishing two-way authenticated trust rather than the client alone verifying the server's identity.

**WAF (Web Application Firewall):** A security layer that inspects HTTP request content (not just network-level headers) for known malicious patterns — SQL injection, cross-site scripting, and similar application-layer attacks — and blocks matching requests before they reach application code.

---

## 7. Core Terminology

| Term | Precise Definition | One-line Intuition |
|---|---|---|
| **Access Token** | A short-lived credential (often a JWT) proving an authenticated session's identity and granted scope | "The valet key — limited, temporary" |
| **Refresh Token** | A longer-lived credential used to obtain a new access token without re-authenticating | "A backstage pass that lets you request a new valet key later" |
| **Scope (OAuth2)** | The specific, limited set of permissions an access token grants | "Exactly what the valet key can and can't do" |
| **Claims (JWT)** | The actual data payload inside a JWT — identity, permissions, expiry — that the signature certifies as unaltered | "What's written and sealed inside the token" |
| **Least Privilege** | The principle that any entity should have the minimum access necessary to perform its function, no more | "Give the valet ONLY the car key, nothing else" |
| **Role-Based Access Control (RBAC)** | An authorization model granting permissions based on a user's assigned role, rather than per-individual configuration | "VIP members vs. general members, not one custom rule per person" |
| **Secrets Management** | The dedicated, secure storage, rotation, and access control of sensitive credentials (API keys, database passwords, signing keys) | "A vault, not a sticky note on the monitor" |

---

## 8. Internal Working

### Why authentication and authorization are genuinely separate decisions, precisely

Recall Module 9's gateway — it validates a JWT (authentication: "this really is user 123") and forwards the request with a trusted `X-User-Id` header. But **the gateway generally doesn't decide** whether user 123 is allowed to delete post 456 — that's an authorization decision, and it depends on business logic the gateway doesn't and shouldn't need to know (is user 123 the post's author? an admin? neither?). **Authorization decisions belong close to the resource they protect** — typically inside the owning service (Core Content, per Module 16's data-ownership principle), not centralized at the gateway, precisely because the gateway would need intimate knowledge of every service's business rules to make these decisions correctly, re-coupling services the gateway was meant to keep decoupled.

```javascript
// Core Content Service — authorization lives HERE, not at the gateway
app.delete('/posts/:id', async (req, res) => {
  const post = await postRepository.findById(req.params.id);
  const userId = req.headers['x-user-id']; // authenticated identity, trusted per Module 9's gateway boundary

  // AUTHORIZATION decision — made here, using domain knowledge the gateway doesn't have
  if (post.authorId !== userId && !isAdmin(userId)) {
    return res.status(403).json({ error: 'Forbidden — not your post' });
  }
  await postRepository.delete(req.params.id);
  res.status(204).send();
});
```

### How OAuth2's Authorization Code flow resolves Section 4's Google-login requirement, mechanically

1. Our blog platform redirects the user to Google's login page, with a request specifying exactly which scopes it wants (e.g., "view your email address").
2. The user logs in **directly with Google**, never entering their Google password into our blog platform at all — this is the core security property: our platform never sees, and can never leak, the user's actual Google credentials.
3. Google redirects back to our platform with a short-lived **authorization code**.
4. Our backend exchanges this code, plus our platform's own secret credentials (Section 30's secrets management), directly with Google's servers (never through the user's browser) for an **access token**.
5. Our platform uses this access token to fetch the user's email from Google's API, then creates or logs into the corresponding local account.

**Why step 4's "directly with Google's servers, never through the browser" detail matters, precisely:** the authorization code alone is not sufficient to obtain an access token — it must be combined with our platform's own secret (never exposed to the browser or the user) — this is what prevents an attacker who somehow intercepts the authorization code in transit (e.g., via a malicious browser extension) from being able to complete the exchange themselves; they'd also need our platform's server-side secret, which never left our backend.

### How mTLS resolves Section 4's requirement 2, and why it's necessary even inside a private subnet

Module 3 established the API Gateway as the sole entry point into the private subnet, and Module 9 established that backend services should be unreachable except through it. But this is a **network-topology** guarantee — it says nothing about what happens if an attacker, through some other means (a compromised container, a misconfigured security group, a supply-chain attack on a dependency), gains the ability to send traffic *from inside* the private subnet. Without mTLS, the Recommendation Service has **no way to cryptographically verify** that a request claiming to be from the Gateway actually is — it can only trust that "this arrived on the private network," which Module 12's own lesson (never trust an unverified assumption during ambiguous conditions) suggests is an incomplete guarantee.

```javascript
const https = require('https');
const fs = require('fs');

const server = https.createServer({
  key: fs.readFileSync('recommendation-service-key.pem'),
  cert: fs.readFileSync('recommendation-service-cert.pem'),
  ca: fs.readFileSync('internal-ca.pem'),          // trust only certs signed by our internal CA
  requestCert: true,                                // REQUEST the client's certificate too (the "mutual" in mTLS)
  rejectUnauthorized: true,                          // REJECT any client that doesn't present a valid cert
}, app);
```

**Why `requestCert: true` and `rejectUnauthorized: true` together implement genuine mutual authentication:** standard TLS (Module 4) only has the *client* verify the *server's* certificate; these two options make the *server* also demand and verify the *client's* certificate — meaning the Recommendation Service will only accept a connection from something possessing a valid certificate signed by the platform's own internal Certificate Authority — precisely the cryptographic proof "unreachable except through the Gateway" (Module 9's network-level guarantee) lacked on its own.

---

## 9. Request Lifecycle

### Mermaid Sequence Diagram — OAuth2 Authorization Code Flow (Resolving Section 4's Google-Login Requirement)

```mermaid
sequenceDiagram
    participant User
    participant Browser
    participant OurPlatform as Our Blog Platform (backend)
    participant Google as Google OAuth Server

    Browser->>OurPlatform: "Log in with Google" clicked
    OurPlatform-->>Browser: Redirect to Google, with requested scopes
    Browser->>Google: User logs in DIRECTLY with Google
    Google-->>Browser: Redirect back with authorization code
    Browser->>OurPlatform: Deliver authorization code
    OurPlatform->>Google: Exchange code + OUR secret for access token (server-to-server, NEVER via browser)
    Google-->>OurPlatform: Access token
    OurPlatform->>Google: GET /userinfo with access token
    Google-->>OurPlatform: { email: "user@example.com" }
    OurPlatform->>OurPlatform: Find or create local account, issue OUR OWN session JWT
    OurPlatform-->>Browser: 200 OK + our platform's session JWT
```

**Step-by-step explanation:** notice our platform's **own** password/credential system is never involved at all — the user's Google password never touches our servers, and our platform issues its **own**, separate session JWT at the very end, completely decoupling "how the user proved their identity" (Google's job) from "how our platform recognizes them on subsequent requests" (our own JWT, exactly Module 9's existing gateway-auth mechanism, unchanged).

### Mermaid Sequence Diagram — mTLS-Verified Service-to-Service Call (Resolving Section 4's Requirement 2)

```mermaid
sequenceDiagram
    participant Gateway as API Gateway
    participant Recommend as Recommendation Service

    Gateway->>Recommend: TLS handshake, presents Gateway's client certificate
    Recommend->>Recommend: Verify cert signed by internal CA, not expired, not revoked
    Recommend->>Gateway: Presents Recommendation Service's own server certificate
    Gateway->>Gateway: Verify Recommendation Service's cert too (mutual!)
    Note over Gateway,Recommend: BOTH sides cryptographically verified —\nnot just "we're on the same private network"
    Gateway->>Recommend: Forward actual request (now over the mutually-authenticated channel)
    Recommend-->>Gateway: Response
```

---

## 10. Architecture Overview

```mermaid
flowchart TB
    User --> WAF[WAF - inspects request CONTENT]
    WAF --> LB[Load Balancer]
    LB --> Gateway[API Gateway - JWT auth, Module 9]
    Gateway -->|mTLS| CoreContent[Core Content Service]
    Gateway -->|mTLS| Search[Search Service]
    Gateway -->|mTLS| Recommend[Recommendation Service]
    CoreContent -->|authorization decisions live HERE| CoreDB[(Core DB)]
    Gateway -.OAuth2 delegation.-> Google[Google OAuth]
    CoreContent -.fetches from.-> Vault[(Secrets Manager)]
```

**HLD-level insight, resolving all four of Section 4's requirements in one diagram:** notice security is applied at **every layer**, each addressing a genuinely distinct threat — WAF at the perimeter (requirement 3), OAuth2 at the identity-delegation layer (requirement 1), mTLS between internal services (requirement 2), and a dedicated secrets manager for credentials (requirement 4) — this is **defense in depth** (Section 1's organizing principle) made concrete: no single layer is assumed sufficient on its own.

---

## 11. Capacity Estimation

Security doesn't have a traditional throughput-style capacity estimation the way earlier modules did, but JWT validation cost is a genuine, measurable performance consideration directly connecting to Module 9, Section 25's caching recommendation:

**Scenario:** Estimating the cryptographic overhead of JWT signature verification at our established 5,000 req/s peak.

**Given:** RSA signature verification (a common JWT signing algorithm) takes approximately 0.1–0.5ms per verification on typical hardware.

**Step 1 — Total verification overhead:**
```
5,000 req/s × 0.3ms (average) = 1,500ms of CPU time consumed per second, across the fleet
```

**Step 2 — Comparing to a single core's available time budget:**
```
1,500ms / 1,000ms per second ≈ 1.5 CPU-cores' worth of continuous work,
distributed across your gateway fleet — a real, non-trivial, but manageable cost
```

**Conclusion, directly validating Module 9, Section 25's caching recommendation:** this confirms that JWT verification is cheap **per call** but adds up to a real, measurable fleet-wide cost at scale — reinforcing why caching the JWKS (public key set used for verification) rather than re-fetching it per request is valuable, and why choosing a faster signing algorithm (e.g., HMAC for scenarios where a shared secret is acceptable, versus RSA where public-key distribution is needed) is a legitimate, quantifiable performance decision, not a premature optimization.

---

## 12. High-Level Design (HLD)

```mermaid
flowchart TD
    A[New request/component needing\na security decision] --> B{Is this about WHO\nis making the request?}
    B -->|Yes| C[Authentication - OAuth2 for\ndelegated login, JWT for session]
    B -->|No, about WHAT they're\nallowed to do| D[Authorization - decided CLOSE\nto the resource, per Section 8]
    A --> E{Is this a service-to-service\ncall within the private network?}
    E -->|Yes| F[mTLS - don't rely on network\nisolation alone, Section 8]
    A --> G{Is this inspecting request\nCONTENT for known attack patterns?}
    G -->|Yes| H[WAF - perimeter, content-aware defense]
    A --> I{Does this involve a credential\nor API key?}
    I -->|Yes| J[Secrets Manager - never in code,\nconfig files, or plaintext, Section 30]
```

**HLD-level insight:** this decision flow operationalizes Section 6's precise definitions into an actionable routing of "which security pattern applies to which specific concern" — a genuinely comprehensive security architecture, per this module's opening principle, is the deliberate application of all four branches together, not a single pattern treated as sufficient on its own.

---

## 13. Low-Level Design (LLD)

### JWT structure and validation, mechanically (completing Module 9's gateway preview with full rigor)

```javascript
const jwt = require('jsonwebtoken');
const jwksClient = require('jwks-rsa');

const client = jwksClient({ jwksUri: 'https://accounts.google.com/.well-known/jwks.json' });

function getSigningKey(header, callback) {
  client.getSigningKey(header.kid, (err, key) => {
    callback(err, key?.getPublicKey());
  });
}

function verifyToken(token) {
  return new Promise((resolve, reject) => {
    jwt.verify(token, getSigningKey, { algorithms: ['RS256'] }, (err, decoded) => {
      if (err) return reject(err);
      resolve(decoded); // { sub: "user123", exp: ..., iat: ..., scope: "..." }
    });
  });
}
```

**LLD-level design note, directly connecting to Section 11's caching motivation:** `jwksClient` internally caches the fetched public keys (Module 9, Section 25's recommendation, now shown as a real, standard library behavior) — the signature check itself (`jwt.verify`) requires **no database lookup at all**; it's purely a cryptographic computation against the cached public key, which is precisely why JWTs scale so well for high-throughput authentication compared to a session-lookup-per-request model (Module 2's stateless-session lesson, now visible at the cryptographic level).

### Role-based authorization middleware (Section 7's RBAC, implemented)

```javascript
function requireRole(...allowedRoles) {
  return (req, res, next) => {
    const userRoles = req.user.roles || []; // populated after JWT verification, from its claims
    const hasPermission = allowedRoles.some(role => userRoles.includes(role));
    if (!hasPermission) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
  };
}

app.delete('/posts/:id/force-delete', requireRole('admin'), async (req, res) => {
  // Only reachable by users whose JWT claims include the "admin" role
});
```

---

## 14. ASCII Diagrams

```
AUTHENTICATION vs AUTHORIZATION — different questions, different owners

  AUTHENTICATION (WHO are you?)         AUTHORIZATION (WHAT can you do?)
  ┌─────────────────────┐               ┌─────────────────────┐
  │  API Gateway         │               │  Core Content Service│
  │  validates JWT        │               │  checks: is this     │
  │  signature/expiry      │               │  YOUR post? admin?   │
  └─────────────────────┘               └─────────────────────┘
  Centralized (Module 9)                Decentralized, close to
                                          the resource (Section 8)
```

```
JWT STRUCTURE (three base64url-encoded parts, dot-separated)

  eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ1c2VyMTIzIn0.SIGNATURE_BYTES
  └──────┬──────┘      └─────────┬─────────┘   └──────┬──────┘
       HEADER                CLAIMS                 SIGNATURE
   {alg, typ}          {sub, exp, roles, ...}   cryptographic proof
                                                 claims weren't altered
```

---

## 15. Mermaid Flowcharts

*(Section 12 covers the canonical "which security pattern applies" decision flow.)*

### Decision Flow: Where Should This Authorization Check Live?

```mermaid
flowchart TD
    A[New authorization rule needed] --> B{Does this rule apply\nUNIFORMLY across ALL services\ne.g. 'must be authenticated at all'?}
    B -->|Yes| C[Gateway level - Module 9's centralized pattern]
    B -->|No, depends on THIS service's\nspecific business/domain logic| D[Owning service level -\nSection 8's decentralized pattern]
    D --> E{Does the rule require data\nonly the owning service has,\ne.g. 'is this YOUR post'?}
    E -->|Yes| F[MUST live in the owning service -\nno other choice preserves\nModule 16's data ownership]
```

---

## 16. Mermaid Sequence Diagrams

*(Section 9 covers this module's two canonical sequence diagrams — OAuth2 flow and mTLS verification. Additional diagram below.)*

### WAF Blocking a SQL Injection Attempt (Resolving Section 4's Requirement 3)

```mermaid
sequenceDiagram
    participant Attacker
    participant WAF
    participant Gateway
    participant Search as Search Service

    Attacker->>WAF: GET /search?q=' OR '1'='1'; DROP TABLE posts;--
    WAF->>WAF: Inspect query content — matches known SQL injection pattern
    WAF-->>Attacker: 403 Forbidden (blocked BEFORE reaching any application code)
    Note over Gateway,Search: Gateway and Search Service never even\nsee this malicious request — blocked at the perimeter

    Attacker->>WAF: GET /search?q=nodejs (legitimate request)
    WAF->>WAF: No malicious pattern detected
    WAF->>Gateway: Forward normally
    Gateway->>Search: Forward normally
```

**Why this matters, precisely, beyond Module 5's parameterized-query defense:** Module 5, Section 24 already taught parameterized queries as the *correct, definitive* fix for SQL injection at the code level — the WAF is **not a substitute** for that fix, but an **additional, defense-in-depth layer** that can catch and block obviously malicious patterns even before they reach application code, providing protection against, e.g., a newly-discovered vulnerability in a dependency, or a code path that a developer forgot to parameterize — exactly the "don't rely on a single layer" principle this entire module is built around.

---

## 17. Component Diagrams

```mermaid
flowchart LR
    subgraph Perimeter
        WAF[WAF]
        LB[Load Balancer]
    end
    subgraph Identity Layer
        OAuthHandler[OAuth2 Handler]
        JWTValidator[JWT Validator]
    end
    subgraph Service Mesh - mTLS enforced
        Gateway[API Gateway]
        Services[Backend Services]
    end
    subgraph Credential Layer
        SecretsManager[Secrets Manager]
    end
    WAF --> LB --> Gateway
    Gateway --> OAuthHandler
    Gateway --> JWTValidator
    Gateway <-->|mTLS| Services
    Services --> SecretsManager
```

**Why these four concerns are modeled as distinct, isolated components:** exactly this module's Section 1 organizing principle — defense in depth means each layer has **one specific, named job**, and no single component's compromise or misconfiguration cascades into total system compromise, directly mirroring Module 9's per-backend circuit-breaking isolation, now applied to security boundaries specifically.

---

## 18. Deployment Diagrams

```mermaid
flowchart TB
    subgraph Public Edge
        WAF[WAF - managed service]
        LB[Load Balancer]
    end
    subgraph Private Subnet - mTLS required for ALL internal traffic
        Gateway[API Gateway]
        CoreContent[Core Content]
        Search[Search]
        Recommend[Recommendation]
    end
    subgraph Isolated Secrets Subnet
        Vault[(Secrets Manager)]
    end
    Internet --> WAF --> LB --> Gateway
    Gateway <-->|mTLS| CoreContent
    Gateway <-->|mTLS| Search
    Gateway <-->|mTLS| Recommend
    CoreContent & Search & Recommend -->|authenticated, least-privilege| Vault
```

**Deployment-level note:** the Secrets Manager sits in its **own, further-isolated subnet** — a deliberate, additional layer of separation beyond even Module 16's per-service database isolation, reflecting that credentials are frequently the single highest-value target for an attacker (a compromised credential can often be used to bypass every other security layer this module describes).

---

## 19. Network Diagrams

This module's mTLS requirement directly extends, rather than replaces, Module 3 and Module 9's network isolation principles:

```
  Module 3/9's guarantee:        THIS MODULE'S ADDITION:
  "No network PATH exists        "AND, even if a path somehow existed,
   from outside to backend        every connection must present a
   services except via Gateway"   valid, verifiable certificate (mTLS)"

  (Network isolation = the FIRST layer)
  (mTLS = the SECOND, cryptographic layer,
   protecting against the case where the
   first layer's assumption is somehow violated)
```

---

## 20. Database Design

Security directly informs database credential and access design, extending Module 16, Section 24's least-privilege observation with full concreteness:

```sql
-- Least-privilege database roles, per service (Module 16's Database-per-Service, security-enforced)
CREATE ROLE core_content_service WITH LOGIN PASSWORD 'fetched-from-secrets-manager';
GRANT SELECT, INSERT, UPDATE, DELETE ON posts, comments TO core_content_service;
-- Explicitly NO grant on any OTHER service's tables — even within the same database instance,
-- if multiple services temporarily share one during a migration (Module 16's transitional case)

CREATE ROLE search_service WITH LOGIN PASSWORD 'fetched-from-secrets-manager';
GRANT SELECT ON search_index TO search_service; -- read-only where write access isn't needed
```

**Why `search_service`'s grant is `SELECT`-only where applicable:** least privilege (Section 7) applies at the database-role level too — if a specific service's function never requires write access to a specific table, granting it anyway is an unnecessary, avoidable expansion of what a compromised instance of that service could do.

---

## 21. API Design

Security-relevant API design should make the authentication/authorization requirement of every endpoint **explicit and discoverable**, extending Module 9, Section 21's per-endpoint labeling discipline:

```
POST /posts                  → requires: authenticated (any role)
DELETE /posts/:id/force-delete → requires: authenticated + role=admin (Section 13)
GET  /posts/:id               → requires: none (public read)
POST /auth/google/callback    → OAuth2 callback endpoint — NOT for direct client use,
                                  validated via Section 9's exact flow
```

**Why documenting this explicitly at the API level matters:** it turns "does this endpoint need auth" from tribal knowledge into a discoverable, verifiable contract — directly supporting Module 19's observability principle (you should be able to *ask* your system this question, not just hope someone remembers correctly) applied to security requirements specifically.

---

## 22. Scalability Considerations

| Consideration | Impact |
|---|---|
| JWT verification | Stateless, scales horizontally with zero shared state (Section 11's cost calculation) — a direct security-layer benefit of Module 2's statelessness principle |
| mTLS certificate management | Certificate issuance/rotation must scale with service count (Module 16) — manual certificate management becomes untenable beyond a handful of services, motivating automated tools (e.g., a service mesh's built-in certificate authority) |
| WAF rule evaluation | Must scale with request volume without becoming a bottleneck itself — most managed WAF services are designed for this, but a self-hosted WAF's own capacity must be planned for like any other component (Module 2) |
| Secrets Manager availability | Every service's ability to start up or refresh credentials depends on it — Section 23 addresses this as a real, load-bearing dependency requiring its own reliability treatment |

---

## 23. Reliability & Fault Tolerance

- **The Secrets Manager becomes a genuine, load-bearing dependency** (Section 22) — if it's unavailable, services that need to fetch or rotate credentials at startup could fail to boot; this deserves the same reliability rigor (Module 18's patterns: caching recently-fetched secrets locally with a fallback, appropriate timeouts) as any other critical dependency.
- **mTLS certificate expiry must be actively monitored and automated for renewal** — an expired internal certificate causing a service-to-service outage is a real, entirely preventable failure mode, and manual certificate renewal processes are a common source of exactly this kind of avoidable incident.
- **Defense in depth (Section 1) is itself a reliability property**, not just a security one — if any single layer (say, the WAF) has a false-negative or is briefly misconfigured, the other layers (parameterized queries, Module 5; mTLS; least-privilege database roles) continue providing protection, rather than the whole system's security collapsing to zero.

---

## 24. Security Considerations

*(This module's entire content is the Security Considerations treatment other modules have deferred — this section instead names the meta-principle unifying it all.)*

- **Defense in depth means no single control is ever assumed sufficient alone** — every layer in Section 10's architecture diagram exists because the layer before it might, for some unanticipated reason, fail or be bypassed.
- **The principle of least privilege should be applied at every layer simultaneously** — database roles (Section 20), OAuth2 scopes (Section 8), RBAC roles (Section 13), and secrets access (Section 30) should all independently enforce "the minimum necessary," rather than any one of them being relied upon exclusively.
- **Security is every prior module's concern, made explicit** — Module 3's network isolation, Module 6's private buckets, Module 9's header-stripping, Module 16's Database-per-Service, and Module 19's no-plaintext-secrets-in-logs are all, retroactively, instances of this module's principles — this module didn't introduce a new concern so much as name and unify one that was present throughout.

---

## 25. Performance Optimization

- **Cache JWKS public keys** (Section 13's `jwksClient` behavior) rather than fetching them per request — a direct, quantified performance win validated by Section 11's overhead calculation.
- **Prefer stateless JWT verification over session-database lookups** for authentication — Module 2's statelessness principle, applied specifically to auth, avoids adding a database round trip to every single authenticated request.
- **Use session resumption for mTLS connections** between frequently-communicating services, avoiding a full handshake (Module 4's TLS-handshake-cost lesson) on every single request between the same two services.

---

## 26. Monitoring & Observability

Directly extending Module 19's framework to security-specific signals:

- **Authentication failure rate** — a sudden spike can indicate a credential-stuffing attack or a client-side bug; directly echoing Module 9, Section 26's original observation, now given full context.
- **Authorization denial rate, per endpoint** — a rising 403 rate on a specific endpoint may reveal either a legitimate access-control bug or a reconnaissance attempt probing for weaknesses.
- **WAF block rate and blocked-pattern breakdown** — visibility into what's actually being blocked (and, periodically, verifying it's not also blocking legitimate traffic — a false-positive risk worth monitoring specifically).
- **Secrets access audit log** — who/what accessed which secret and when, a critical forensic capability if a credential compromise is ever suspected.

---

## 27. Common Bottlenecks

| Bottleneck | Symptom | Root Cause |
|---|---|---|
| JWKS fetched on every request | High auth-layer latency, elevated load on the identity provider | JWKS caching not implemented (Section 13/25) |
| Certificate expiry outage | Sudden, fleet-wide service-to-service failures | No automated certificate renewal/monitoring (Section 23) |
| Overly broad database grants | Larger-than-necessary blast radius if a service is compromised | Least privilege not enforced at the database-role level (Section 20) |
| WAF false positives blocking legitimate traffic | Users report inexplicable request failures for specific, valid inputs | WAF rules too aggressive/broad, not tuned against real traffic patterns |
| Secrets Manager unavailability at startup | Services fail to boot during a Secrets Manager outage | No local caching/fallback for previously-fetched secrets (Section 23) |

---

## 28. Trade-off Analysis

> "I chose to implement **mTLS for all internal service-to-service traffic**, optimizing for **cryptographic verification of service identity, not just network-path trust**, at the cost of **certificate management overhead and a small per-connection handshake cost**, which is acceptable because a compromised instance inside the private subnet impersonating another service is a realistic, serious enough risk to justify this additional layer, beyond Module 3/9's network isolation alone."

> "I chose **OAuth2 delegation for Google login** rather than building our own password-based authentication for this use case, optimizing for **never handling or storing the user's actual Google credentials, reducing our platform's liability and attack surface**, at the cost of **a dependency on Google's OAuth infrastructure's availability for this login path**, which is acceptable because the security benefit of never touching third-party credentials directly far outweighs this dependency risk, and we retain our own separate session JWT (Section 9) for ongoing authentication regardless."

---

## 29. Anti-patterns & Common Mistakes

1. **Conflating authentication and authorization**, assuming "they're logged in" automatically means "they can do this specific action" (Section 8) — a serious, common access-control bug source.
2. **Trusting network isolation alone for service-to-service authentication**, without mTLS — leaves the system vulnerable if that network-level assumption is ever violated (Section 8/23).
3. **Storing secrets in source code, environment files committed to version control, or plaintext configuration** — one of the most common, damaging real-world security incidents, directly motivating Section 30's dedicated treatment.
4. **Relying on a WAF as a substitute for parameterized queries** (Module 5) rather than as an additional, defense-in-depth layer — a WAF misconfiguration or novel bypass technique would leave the system fully exposed if it were the *only* SQL-injection defense.
5. **Overly broad database or IAM grants** "to avoid permission errors during development," never revisited before production (Section 20/24).
6. **No monitoring of authentication/authorization failure rates** (Section 26), missing early signals of an active attack or credential-stuffing attempt.

---

## 30. Production Best Practices

- **Use a dedicated secrets manager** (e.g., AWS Secrets Manager, HashiCorp Vault) for every credential — never source code, never plaintext config files, never environment variables set via insecure means.
- **Enforce mTLS for all internal service-to-service communication**, not just relying on network isolation alone.
- **Apply least privilege consistently** — database roles, OAuth2 scopes, RBAC permissions, and secrets access should all independently grant only the minimum necessary.
- **Place authorization decisions close to the resource they protect**, in the owning service, per Module 16's data-ownership principle — never centralize business-logic-dependent authorization at the gateway.
- **Layer a WAF in addition to, never instead of, proper input validation and parameterized queries.**
- **Automate certificate and secret rotation**, with monitoring for expiry, rather than relying on manual processes.
- **Monitor authentication and authorization signals** (Section 26) as first-class security metrics, not an afterthought.

---

## 31. Real-World Examples

- **The 2017 Equifax breach**, one of the most consequential real-world security incidents in recent history, was significantly attributable to an unpatched, known vulnerability combined with insufficient network segmentation and monitoring — a real-world, high-stakes illustration of exactly why this module's defense-in-depth principle (Section 1, 24) matters: a single missed patch became catastrophic specifically because subsequent layers of defense weren't robust enough to contain it.
- **Uber's 2016 data breach**, involving credentials found in a public GitHub repository, is one of the most frequently cited real-world examples of Section 29's "secrets in source code" anti-pattern — directly, concretely validating this module's Section 30 secrets-management guidance as a response to a genuine, repeatedly-occurring class of real incident, not a theoretical concern.
- **OAuth2's widespread industry adoption** (Google, Facebook, GitHub, and virtually every major platform offering "log in with X") reflects the same Section 8 delegation principle at massive, real-world scale — the near-universal preference for OAuth2 over platforms building and storing their own third-party credentials directly validates this module's specific recommendation for Section 4's Google-login requirement.

---

## 32. Node.js Implementation Examples

### A secrets-manager client with local caching and fallback (extending Section 23's reliability treatment)

```javascript
class SecretsClient {
  constructor(secretsManagerSDK) {
    this.sdk = secretsManagerSDK;
    this.cache = new Map();
  }

  async getSecret(secretName) {
    try {
      const value = await this.sdk.getSecretValue({ SecretId: secretName }).promise();
      this.cache.set(secretName, value.SecretString); // cache on success (Section 23's resilience pattern)
      return value.SecretString;
    } catch (err) {
      const cached = this.cache.get(secretName);
      if (cached) {
        console.warn(`Secrets Manager unavailable, using cached value for ${secretName}`);
        return cached; // graceful degradation — directly applying Module 18's fallback principle
      }
      throw new Error(`Secret ${secretName} unavailable, and no cached fallback exists`);
    }
  }
}

// Usage
const secrets = new SecretsClient(awsSecretsManagerSDK);
const emailApiKey = await secrets.getSecret('email-provider-api-key'); // Section 4's requirement 4, resolved
```

**Why this directly resolves Section 4's requirement 4, and connects to Module 18:** the API key is never in source code, never in a plaintext config file — it's fetched from a dedicated secrets manager at runtime, with a Module 18-style graceful fallback (use the last successfully-cached value) if the Secrets Manager is briefly unavailable, directly applying this module's reliability lesson (Section 23) rather than treating secrets fetching as an unprotected, single point of failure.

---

## 33. Interview Questions

### Easy
1. What is the difference between authentication and authorization?
2. What is OAuth2, and why is it described as a delegation framework rather than simply "login"?
3. What is a JWT, and how can its signature be verified without a database lookup?
4. What is mTLS, and how does it differ from standard TLS?
5. What does a WAF protect against that a network firewall (Module 3) does not?
6. Why should secrets never be stored in source code or plaintext configuration files?

### Medium
7. Explain why authorization decisions should generally live close to the resource they protect, rather than centralized at the API Gateway.
8. Walk through the OAuth2 Authorization Code flow, explaining why the authorization code alone is insufficient to obtain an access token.
9. Why is mTLS still valuable even when backend services are already isolated in a private subnet unreachable from the internet?
10. Design a least-privilege database role strategy for a 3-service microservices system, following Module 16's Database-per-Service boundaries.
11. Explain why a WAF should never be relied upon as a substitute for parameterized queries.
12. Design a graceful degradation strategy for a service that needs to fetch a secret from a Secrets Manager that is temporarily unavailable.

### Hard
13. Design a complete security architecture for a 5-service microservices platform, addressing authentication (OAuth2), service-to-service trust (mTLS), perimeter defense (WAF), and secrets management, explaining how each layer provides defense in depth.
14. Explain, precisely, why JWT-based stateless authentication scales more efficiently than session-based authentication requiring a database lookup per request, connecting this to Module 2's statelessness principle.
15. A security review discovers that the API Gateway is making authorization decisions that require detailed business logic about post ownership. Diagnose why this violates Module 16's data-ownership principle and propose the correct redesign.
16. Design a certificate rotation strategy for an mTLS-secured microservices platform that avoids both manual, error-prone renewal processes and the risk of an expired-certificate outage.
17. Discuss the real-world Uber 2016 breach (Section 31) in terms of this module's specific anti-patterns and best practices, explaining exactly which recommended practice would have prevented it.

---

## 34. Scenario-Based Design Questions

1. **Scenario:** A junior engineer commits a database password directly into a configuration file that gets pushed to a public GitHub repository. Walk through the immediate incident response and the long-term process fix.
2. **Scenario:** Your API Gateway is currently making authorization decisions like "is this user the author of this post," requiring it to query Core Content's database directly. Diagnose the architectural problem and redesign it correctly.
3. **Scenario:** A newly discovered vulnerability allows an attacker to craft a malicious search query that bypasses your application's input validation. Your WAF successfully blocks it. Discuss why this doesn't mean the underlying vulnerability can be ignored.
4. **Scenario:** An internal service's mTLS certificate expires unexpectedly, causing a fleet-wide service-to-service outage. Propose both an immediate fix and a long-term prevention strategy.
5. **Scenario:** Your platform wants to let users "log in with GitHub" in addition to Google. Design the OAuth2 flow, explaining what changes and what stays the same compared to the Google flow.
6. **Scenario:** A compromised container inside your private subnet attempts to make requests directly to your Recommendation Service, bypassing the API Gateway. Explain how mTLS would (or wouldn't) prevent this from succeeding.
7. **Scenario:** Your team debates whether a specific database role should have `DELETE` permission "just in case it's needed later." Evaluate this against the least-privilege principle.
8. **Scenario:** A security audit finds that your Secrets Manager has no access logging, and you cannot determine whether a suspected leaked credential was ever actually accessed by an unauthorized party. Propose the remediation.
9. **Scenario:** Your platform's JWT verification is adding noticeable latency under high load, and investigation reveals the JWKS is being fetched fresh on every single request. Diagnose and fix.
10. **Scenario:** An interviewer asks you to design authentication and authorization for a healthcare records platform with strict regulatory compliance requirements. Discuss how your approach would differ from, or extend, this module's blog-platform-focused examples.

---

## 35. Hands-on Exercises

1. Implement JWT generation and verification (Section 13) locally, including a deliberately expired token and a deliberately tampered payload, verifying both are correctly rejected.
2. Implement the RBAC middleware from Section 13, and write tests verifying a non-admin user is correctly denied access to an admin-only endpoint, while an admin user succeeds.
3. Set up a local mTLS connection between two simple Node.js HTTPS servers using self-signed certificates and a local CA, and verify that a connection attempt without a valid client certificate is correctly rejected.
4. Implement the `SecretsClient` with local caching and fallback (Section 32), simulate a Secrets Manager outage, and verify the cached fallback value is correctly used instead of the request failing outright.
5. Research and document, in your own words, the specific difference between the OAuth2 Authorization Code flow and the (now largely deprecated for security reasons) Implicit flow, explaining why the Authorization Code flow is considered more secure.

---

## 36. Mini Project

**Build:** A complete authentication and authorization layer for the blog platform, directly resolving Module 20's Section 4 requirements 1, 3, and 4.

**Requirements:**
- Implement OAuth2 "Log in with Google" (or a simulated equivalent OAuth2 provider for testing) issuing your platform's own session JWT upon successful login (Section 9).
- Implement JWT verification middleware at the API Gateway, and RBAC-based authorization middleware within Core Content Service for at least one admin-only action.
- Integrate a secrets manager (a real cloud service, or a local simulation) for storing and retrieving the email provider's API key (Module 18), with local caching and fallback (Section 32).
- Configure a basic WAF rule (using a real managed WAF service, or a simple regex-based middleware simulation) blocking an obvious SQL-injection-pattern query string.

**Success criteria:** A user can log in via the OAuth2 flow and receive a valid session JWT; an admin-only endpoint correctly rejects non-admin users; your secrets client correctly falls back to a cached value during a simulated Secrets Manager outage; and a deliberately malicious query string is blocked before reaching your application code.

---

## 37. Advanced Project

**Build:** Extend the Mini Project with mTLS between services and a full security-monitoring dashboard.

1. Implement mTLS (Section 8/13) between your API Gateway and at least one backend service, using a local CA, and write a test verifying a connection attempt without a valid client certificate is correctly rejected.
2. Implement authentication-failure-rate and authorization-denial-rate monitoring (Section 26), directly reusing Module 19's structured logging and metrics patterns, and simulate a burst of failed login attempts to verify your monitoring correctly detects and surfaces the spike.
3. Implement least-privilege database roles (Section 20) for at least two simulated services sharing infrastructure, and write a test proving one service's credentials cannot access the other's tables.
4. Write a complete security architecture document for the full, 20-module blog platform built throughout this masterclass, mapping every layer (network isolation from Module 3, Database-per-Service from Module 16, and this module's OAuth2/JWT/mTLS/WAF/secrets management) into one unified defense-in-depth diagram, explicitly identifying which specific threat each layer defends against.

**Success criteria:** You have working, tested mTLS enforcement with a passing rejection test, functioning security-specific monitoring that correctly detects a simulated attack pattern, verified least-privilege database isolation between services, and a complete, unified security architecture document synthesizing every prior module's security-relevant decisions into this module's defense-in-depth framework — setting up Module 21 (Rate Limiting), which formalizes the specific algorithms (token bucket, leaky bucket, sliding window) underlying the abuse-prevention concerns this module's WAF and authentication-failure-monitoring have referenced but not yet fully specified.

---

## 38. Summary

- **Authentication and authorization are genuinely separate decisions** — the former establishes identity (often centralized, at the gateway); the latter determines permitted actions (best decided close to the resource, using domain-specific knowledge the gateway shouldn't need).
- **OAuth2 is a delegation framework**, not a login mechanism — its core security property is that a user's actual credentials for a third-party service (Google) are never seen by the platform requesting delegated access.
- **JWTs enable stateless, database-lookup-free authentication verification**, directly extending Module 2's statelessness principle to the authentication layer, at a real but manageable computational cost (Section 11).
- **mTLS provides cryptographic service-to-service trust that network isolation alone cannot** — it protects against exactly the scenario where a network-topology assumption is somehow violated.
- **A WAF is a defense-in-depth layer, never a substitute** for proper input validation and parameterized queries (Module 5) — it catches what other layers might miss, not the other way around.
- **Secrets must live in a dedicated secrets manager**, never in source code or plaintext configuration — one of the most common and damaging real-world security failures, and one of the most straightforward to prevent.
- **Defense in depth** — this module's unifying principle — means every layer assumes the layers around it might, someday, fail or be bypassed, and is designed to still provide protection when that happens.

---

## 39. Revision Notes

- Authentication = WHO (centralized, often at gateway); Authorization = WHAT they can do (decentralized, close to the resource)
- OAuth2 = delegation — user's real credentials for the third party (Google) never touch your platform
- JWT = self-contained, signature-verifiable, no DB lookup needed — cache the JWKS public keys
- mTLS = BOTH sides verify certs — protects against a compromised node inside an otherwise-isolated private network
- WAF = content-inspecting perimeter defense — additional layer, NEVER a substitute for parameterized queries
- Secrets Manager = the only correct place for credentials — never source code, never plaintext config
- Defense in depth = no single layer assumed sufficient — every layer backs up every other layer

---

## 40. One-Page Cheat Sheet

```
SYSTEM DESIGN — MODULE 20 CHEAT SHEET
─────────────────────────────────────
AUTHENTICATION (WHO)              AUTHORIZATION (WHAT)
  Centralized (Gateway, Module 9)   Decentralized (owning service)
  OAuth2 for delegated login        RBAC / least privilege
  JWT for stateless session proof   Close to the resource — needs domain data

OAUTH2 CORE PROPERTY
  User's REAL credentials (Google, etc.) NEVER touch your platform
  Your platform gets a limited, scoped access token — a "valet key"

JWT — stateless, no DB lookup needed to verify
  Cache the JWKS public keys — don't refetch per request

mTLS — BOTH sides verify certs
  Protects against: compromised node INSIDE an "isolated" private network
  Network isolation (Module 3/9) = layer 1; mTLS = layer 2 (cryptographic)

WAF — perimeter, content-aware defense
  Additional layer, NEVER a substitute for parameterized queries (Module 5)

SECRETS MANAGEMENT
  NEVER in source code, NEVER in plaintext config
  Dedicated secrets manager + local cache/fallback (Module 18's pattern)

GOLDEN RULE — DEFENSE IN DEPTH
  No single security layer is ever assumed sufficient on its own.
```

---

## Key Takeaways

- Security in a distributed system is not one control but the deliberate composition of many, each assuming the others might someday fail — this module's defense-in-depth principle retroactively unifies every brief security callout from the previous nineteen modules into one coherent discipline.
- Authentication and authorization are frequently, damagingly conflated — recognizing that "who you are" and "what you can do" are answered by different components, using different data, at different points in the request lifecycle, is the single highest-value distinction in this module.
- The most common, most damaging real-world security incidents (Uber's 2016 breach, countless others) trace back to remarkably simple, well-understood, entirely preventable mistakes — secrets in source code, missing least privilege — making disciplined adherence to this module's practices disproportionately valuable relative to their implementation cost.

## 20 Practice Questions
*(See Section 33 — 6 Easy, 6 Medium, 5 Hard — plus 3 rapid-fire additions:)*
18. Why does OAuth2's Authorization Code flow require a server-to-server exchange step, rather than delivering the access token directly to the browser?
19. What's the precise difference between a network firewall (Module 3) and a WAF, in terms of what layer of the request each inspects?
20. Why should database role permissions be scoped per-service (Module 16) even when multiple services temporarily share one database instance during a migration?

## 10 Scenario-Based Questions
*(See Section 34 in full.)*

## 5 Design Assignments
*(See Sections 36–37 — Mini Project and Advanced Project — plus:)*
1. Design a complete authentication and authorization architecture for a multi-tenant SaaS platform, addressing tenant isolation as an additional authorization dimension beyond individual user roles.
2. Write a one-page incident response plan for a suspected leaked API key, including immediate containment, rotation, and audit steps.
3. Propose an mTLS certificate lifecycle management strategy (issuance, rotation, revocation, monitoring) for a 10-service microservices platform.

## Suggested Next Module

**→ Module 21: Rate Limiting** — with authentication, authorization, and perimeter defenses now in place, we formalize the specific algorithms — token bucket, leaky bucket, fixed and sliding window — that determine exactly how many requests any single authenticated (or unauthenticated) client is allowed to make, completing the abuse-prevention picture this module's WAF and monitoring sections have referenced throughout.
