# Day 24 — Caching Strategies at Scale: CDNs, Cache-Control, Multi-Layer Architecture

> **Why this day matters:** Day 17 covered Redis caching specifically. Today zooms out to the FULL picture — every layer a request passes through before it even reaches your Redis cache, and the HTTP-level caching mechanisms (headers, CDNs) that can make your application-level cache irrelevant for a huge share of requests because they never even reach your server at all. This system-wide view is exactly what separates "I used Redis to cache a query" from "I understand caching as an architectural concern."

---

## 1. Background: the full journey of a request, and where caching can intercept it

```
Browser ──> CDN ──> Load Balancer ──> Node App ──> Redis Cache ──> Database
   │           │                          │              │
   │           │                          │              └─ Day 17's topic
   │           │                          └─ Application-level caching (in-memory, etc.)
   │           └─ CDN/edge caching -- can answer WITHOUT ever reaching your servers at all
   └─ Browser caching -- can answer WITHOUT ever leaving the user's device
```

**The critical insight that reframes everything today:** the BEST possible cache hit is one that **never reaches your server at all**. Every layer that can intercept and answer a request earlier in this chain saves more total work (network round trips, server CPU, database load) than the layer after it. This is why a well-designed system thinks about caching at EVERY layer, not just "add Redis in front of the database."

---

## 2. Browser Caching — the closest possible cache to the user

### Background: how does a browser know whether to reuse a cached response, or fetch fresh?

This is entirely controlled by HTTP response **headers** your server sends. Get these right, and repeat visitors don't even make a network request for unchanged resources at all.

```javascript
// Cache-Control header -- the primary mechanism for controlling
// browser (and CDN/proxy) caching behavior
app.get('/api/static-config', (req, res) => {
  // 'public' -- can be cached by ANY cache (browser, CDN, shared proxies)
  // 'max-age=3600' -- consider this fresh for 3600 seconds (1 hour);
  //   after that, the browser must revalidate or refetch
  res.setHeader('Cache-Control', 'public, max-age=3600');
  res.json({ featureFlags: { newCheckout: true } });
});

app.get('/api/user/profile', (req, res) => {
  // 'private' -- can only be cached by the END USER'S browser, NOT by
  //   any shared/intermediate cache (CDN, corporate proxy) -- critical
  //   for any response containing user-specific or sensitive data
  // 'no-store' -- don't cache this AT ALL, anywhere, ever -- appropriate
  //   for highly sensitive data (e.g., a one-time payment confirmation
  //   page, or anything where even briefly stale cached data is unacceptable)
  res.setHeader('Cache-Control', 'private, no-store');
  res.json({ name: 'Alice', balance: 542.10 });
});
```

### ETags — a smarter alternative to blind expiration times

**Background: what problem does this solve that `max-age` alone doesn't?** `max-age` is a TIME-based guess — "this is probably still fresh for an hour." But what if the data DIDN'T change at all in that hour? You'd still want to revalidate eventually, but ideally WITHOUT re-downloading the entire response if nothing actually changed. An **ETag** is a hash/fingerprint of the response content — the browser can ask "has anything changed since I last saw ETag X?" and the server can answer with a tiny "no, nothing changed" (HTTP 304 Not Modified) instead of resending the full response body.

```javascript
const crypto = require('crypto');

app.get('/api/articles/:id', async (req, res) => {
  const article = await Article.findByPk(req.params.id);

  // Generate a hash of the content -- this becomes the ETag
  const etag = crypto.createHash('md5').update(JSON.stringify(article)).digest('hex');
  res.setHeader('ETag', etag);

  // Check if the CLIENT already has this exact version (sent via the
  // 'If-None-Match' request header, automatically managed by browsers)
  if (req.headers['if-none-match'] === etag) {
    return res.status(304).end(); // "Not Modified" -- tiny response, no body needed at all
  }

  res.json(article); // content actually changed (or first request) -- send the full response
});
```

**Interview tip:** if asked "what's the difference between `Cache-Control` max-age and ETags," the precise distinction is: `max-age` avoids a network request ENTIRELY until it expires (fastest, but can serve stale data within that window); ETags still make a network request, but that request is tiny if nothing changed (a 304 with no body) — a good real system often uses BOTH together, accepting a short `max-age` for speed, with ETag-based revalidation kicking in once that expires.

---

## 3. CDNs (Content Delivery Networks)

### Background: what problem does a CDN solve that browser caching alone doesn't?

Browser caching only helps a RETURNING visitor (same browser, same device). It does nothing for the FIRST request from any given user, and does nothing to reduce the distance data has to travel — a user in Australia hitting a server in the US still has real network latency on every uncached request. A **CDN** solves both: it's a globally distributed network of servers ("edge locations" or "points of presence") that cache content GEOGRAPHICALLY CLOSE to users, so even a user's FIRST request can potentially be served from a nearby edge server instead of traveling all the way to your origin server.

```
User in Australia ──> Nearest CDN edge server (in Australia) ──> 
   (if cached) returns immediately, NEVER reaching your origin server at all
   (if not cached) fetches from your origin server ONCE, caches it at
   that edge location, serves it -- and ALL SUBSEQUENT Australian users
   benefit from that cached copy too, not just the original requester
```

**What CDNs are traditionally best at caching:** static assets — images, CSS, JS bundles, videos — content that's the SAME for every user and changes infrequently. **Modern CDNs increasingly also cache API responses** for cacheable, non-personalized data (e.g., a public product catalog endpoint), using the same `Cache-Control` headers discussed above to decide what and how long to cache.

```javascript
// A CDN-friendly API endpoint -- public data, same for everyone,
// genuinely safe to cache at the EDGE, not just in the browser
app.get('/api/public/products', async (req, res) => {
  const products = await Product.findAll();
  res.setHeader('Cache-Control', 'public, max-age=60, s-maxage=300');
  // 's-maxage' specifically targets SHARED caches (CDNs, proxies) --
  // letting you set a DIFFERENT cache duration for the CDN (5 minutes)
  // versus the end user's own browser (1 minute), useful when you want
  // the CDN to absorb most repeat traffic for longer while still
  // letting individual browsers revalidate a bit more frequently
  res.json(products);
});
```

**Interview tip:** if asked "why use a CDN instead of just relying on your application's Redis cache," the key distinction is GEOGRAPHIC distribution and request INTERCEPTION before it reaches your infrastructure at all — Redis still requires a request to travel all the way to your server/region and your app to actually run (even if the DB query itself is skipped); a CDN cache hit can be served from a location near the user, without your servers being involved in any way.

---

## 4. Multi-Layer Caching Architecture — putting it all together

```
1. Browser cache       -- fastest, zero network request, single-user only
2. CDN / edge cache    -- fast, geographically close, shared across ALL users near that edge
3. Reverse proxy cache -- e.g., Nginx can cache responses itself, in front of your app servers
4. Application cache   -- in-memory (per-process, Day 15's limitation applies) or Redis (shared, Day 17)
5. Database query cache -- some databases have their own internal query caching
6. The actual database -- the "source of truth," the most expensive layer to hit
```

**The framing that ties this whole day together:** each layer exists to ABSORB load before it reaches the next, more expensive layer. A well-cached, well-designed system might serve 95%+ of requests for popular, cacheable content WITHOUT EVER reaching the actual database, let alone running expensive business logic — and a huge share of THOSE without even reaching your application servers at all (browser/CDN layers).

```javascript
// A realistic combined approach for one endpoint, using multiple layers
// deliberately and explicitly
app.get('/api/articles/:id', async (req, res) => {
  const articleId = req.params.id;

  // LAYER: application-level Redis cache (Day 17) -- check first
  const cached = await redisClient.get(`article:${articleId}`);
  if (cached) {
    const article = JSON.parse(cached);
    res.setHeader('Cache-Control', 'public, max-age=60'); // also let the BROWSER/CDN cache this
    return res.json(article);
  }

  // Cache miss -- hit the actual database
  const article = await Article.findByPk(articleId);
  if (!article) return res.status(404).json({ error: 'Not found' });

  await redisClient.set(`article:${articleId}`, JSON.stringify(article), { EX: 300 });

  res.setHeader('Cache-Control', 'public, max-age=60');
  res.json(article);
});
```

---

## 5. Cache Invalidation at Scale — beyond Day 17's single-Redis-instance view

### Background: what's different about invalidation once a CDN is involved?

Day 17 covered deleting a Redis key on update (`client.del()`). But if a CDN has ALSO cached that same content at dozens of geographically distributed edge locations, deleting your Redis key does nothing to those CDN-cached copies — they'll keep serving the OLD version until their own TTL expires, unless you explicitly tell the CDN to invalidate/purge that specific cached content.

```javascript
// Conceptual example -- most CDN providers (Cloudflare, CloudFront,
// Fastly) offer an API specifically for this purpose
async function invalidateCDNCache(path) {
  await fetch('https://api.cdn-provider.com/purge', {
    method: 'POST',
    headers: { Authorization: `Bearer ${process.env.CDN_API_KEY}` },
    body: JSON.stringify({ paths: [path] }),
  });
}

async function updateArticle(articleId, updates) {
  const article = await Article.findByPk(articleId);
  await article.update(updates);

  await redisClient.del(`article:${articleId}`);        // Day 17's app-level invalidation
  await invalidateCDNCache(`/api/articles/${articleId}`); // ALSO purge the CDN's cached copy
}
```

**Why this matters as a real interview-worthy nuance:** invalidation gets HARDER, not easier, as you add more caching layers — each layer is one more place stale data can hide. A common, pragmatic real-world compromise: accept a SHORT TTL at the CDN/browser level (e.g., 60 seconds) for content that changes occasionally, rather than building a fully synchronized, instant multi-layer invalidation system for every piece of cacheable content — explicit purging is reserved for content where even a brief staleness window is genuinely unacceptable.

---

## 6. How this connects to real backend work (3 YOE framing)

- **"Your API serves the same public data to millions of users worldwide, and your server costs are high — what would you do?"** → Push that data through a CDN with appropriate `Cache-Control`/`s-maxage` headers, since identical, non-personalized responses are ideal CDN candidates — this can eliminate the vast majority of requests from ever reaching your origin servers at all.
- **"A user reports seeing their old profile picture even after updating it — what are the likely causes?"** → Browser caching the old image (check `Cache-Control` on that asset), and/or a CDN caching it at the edge without being purged on update — would need either a versioned/hashed filename (forcing a new URL on change, sidestepping invalidation entirely) or an explicit CDN purge call.
- **"Why might `private, no-store` be the right `Cache-Control` choice for a banking API's account balance endpoint, even though caching elsewhere has made your app fast?"** → Sensitive, user-specific, rapidly-changing data where ANY staleness (even briefly) could show an incorrect balance — the cost of an extra round trip is far outweighed by the requirement for absolute correctness on this specific kind of data.
- **"How would you decide what TTL to use at each caching layer for a given piece of data?"** → Based on how frequently the underlying data changes, how tolerable staleness is for that specific use case, and the cost/complexity of active invalidation versus just accepting a bounded staleness window via a shorter TTL.

---

## 7. Hands-on practice for today

```javascript
// practice-day24.js
const express = require('express');
const crypto = require('crypto');

const app = express();

const articles = {
  1: { id: 1, title: 'Intro to Caching', content: 'Caching is great.' },
};

app.get('/api/articles/:id', (req, res) => {
  const article = articles[req.params.id];
  if (!article) return res.status(404).json({ error: 'Not found' });

  const etag = crypto.createHash('md5').update(JSON.stringify(article)).digest('hex');
  res.setHeader('ETag', etag);
  res.setHeader('Cache-Control', 'public, max-age=30');

  if (req.headers['if-none-match'] === etag) {
    return res.status(304).end();
  }

  res.json(article);
});

app.listen(3000, () => console.log('Running on http://localhost:3000'));
```

Test the ETag flow manually:
```bash
# First request -- note the ETag value in the response headers
curl -i http://localhost:3000/api/articles/1

# Second request, passing that ETag back -- should get a 304 with NO body
curl -i http://localhost:3000/api/articles/1 -H 'If-None-Match: "PASTE_THE_ETAG_VALUE_HERE"'
```

Confirm the second request returns `304 Not Modified` with an empty body, while the first returns the full JSON — direct, hands-on proof of how ETag-based revalidation actually saves bandwidth on unchanged content.

---

## 8. Interview Q&A Recap (say these out loud)

1. **Q: Why is a CDN cache hit considered "better" than even a fast Redis cache hit?**
   A: A CDN can serve a request from an edge location geographically close to the user, without the request ever reaching your origin infrastructure at all — a Redis hit still requires the request to travel to your server's region and your application to run, even though it skips the database.

2. **Q: What's the difference between `Cache-Control: private` and `Cache-Control: public`?**
   A: `private` means only the end user's own browser may cache the response — appropriate for user-specific data. `public` means any cache, including shared CDNs/proxies, may cache and serve it to other users — only appropriate for non-personalized, identical-for-everyone content.

3. **Q: How do ETags improve on simple time-based (`max-age`) caching?**
   A: They let the client revalidate cheaply — if content hasn't actually changed, the server responds with a tiny 304 Not Modified instead of resending the full response body, avoiding both unnecessary bandwidth usage and the staleness risk of relying purely on a fixed expiration time.

4. **Q: What's `s-maxage`, and why would you set it differently from `max-age`?**
   A: `s-maxage` specifically controls how long SHARED caches (CDNs, proxies) consider a response fresh, separately from `max-age`, which governs the end user's own browser — letting you, for example, have a CDN cache something longer than an individual browser does.

5. **Q: Why does adding more caching layers make invalidation harder, not easier?**
   A: Each additional layer (browser, CDN, application cache) is one more place stale data can persist after an update — invalidating your application-level Redis cache does nothing to already-cached copies sitting at CDN edge locations or in users' browsers, which require their own explicit purge mechanisms or short TTLs to stay reasonably fresh.

6. **Q: When would `Cache-Control: no-store` be the right choice, despite the performance cost of never caching?**
   A: For highly sensitive or rapidly-changing data where any staleness is unacceptable (e.g., real-time account balances, one-time payment confirmations) — correctness outweighs the performance benefit of caching for that specific kind of response.

---

## Tomorrow (Day 25) Preview
We cover **microservices fundamentals**: what actually qualifies as a microservices architecture (vs. a monolith), service-to-service communication patterns (REST, gRPC conceptually, message queues), the API Gateway pattern, and the real tradeoffs (not just benefits) of breaking a system into microservices — a topic where interviewers specifically probe for balanced, honest judgment rather than buzzword enthusiasm.
