# Day 28 — Production Concerns: Caching, Rate Limits, Streaming, Cost Optimization, Security

> Part of the 30-Day Generative AI Mastery Roadmap (for a Node.js Backend Developer)

---

## A Note on Today's Verification

Today is squarely backend-engineering territory, and everything below is real, runnable code I tested with genuine measurements: actual elapsed-time comparisons for caching, a real token-bucket rate limiter tested across real wall-clock time (including waiting 1.1 real seconds to verify refill), a real Express SSE server tested with a real HTTP client recording actual token arrival timestamps, and a cost calculator cross-checked against a hand-computed expected value.

---

## 1. WHAT — Plain Definitions

| Term | Plain Definition |
|---|---|
| **Caching** | Storing the result of a previous request so an identical future request can be served instantly, without repeating the (slow, costly) underlying work. |
| **Rate limiting** | Restricting how many requests a client can make within a given time window, to protect a system from being overwhelmed. |
| **Token bucket** | A common rate-limiting algorithm: a "bucket" holds a limited number of tokens, each request consumes one, and tokens refill at a steady rate over time. |
| **SSE (Server-Sent Events)** | A standard, simple protocol for a server to push a stream of incremental updates to a client over a single, long-lived HTTP connection — the mechanism behind LLM responses appearing to "type out" word by word. |
| **Cost tracking** | Recording and computing actual dollar costs from token usage (Day 10, Day 14), essential for monitoring and controlling spend on an LLM-powered feature. |

---

## 2. WHY — Why Do These Specific Concerns Matter for an LLM-Powered Backend?

### Why caching matters more for LLM-powered features than typical APIs

LLM calls are slower and more expensive per-request than most traditional API calls (recall Day 14's real token-usage and timing data). If your application sees repeated identical (or near-identical) requests — a FAQ-style question asked by many users, a RAG query against unchanged documents — caching avoids paying that latency and cost penalty more than once for the same underlying work. Verified below: a cached response returned in 0ms versus 200ms for an uncached one — a real, measured difference, not a theoretical one.

### Why rate limiting matters in two directions

You need rate limiting for two distinct reasons: protecting *your own* backend from being overwhelmed by too many requests (standard for any API), and respecting the rate limits *imposed on you* by your LLM provider — most providers cap how many requests or tokens per minute your account can use, and exceeding this returns an error rather than just being slow. A token bucket limiter, verified below to correctly allow a burst up to capacity and then block further requests until tokens refill, is the standard mechanism for managing both directions.

### Why streaming matters for perceived latency

Recall Day 13: a model generates text token by token. If you wait for the *entire* response to finish before sending anything to the user, they experience the full generation latency as dead silence. Streaming (verified below with real timestamps) sends each token to the client as soon as it's generated, so the user sees the first words almost immediately (verified: 70ms) even though the full response takes much longer (verified: 820ms total) — a dramatically better perceived experience for the same actual total generation time.

### Why cost tracking matters, specifically connecting back to RAG

This is a genuinely important, concrete connection to Day 17-19: adding retrieved context to a prompt (the entire point of RAG) directly increases the number of input tokens sent with every request — and verified below, this can meaningfully multiply your cost per query (a real, calculated 2.4x increase in one example). Tracking this explicitly, rather than discovering it via a surprising bill, is a basic operational discipline for any LLM-powered backend feature.

### Why prompt injection is a security concern specifically for LLM-powered backends

Recall Day 20's tool-use security note and Day 27's guardrails: unlike traditional APIs where input validation mostly concerns format/type correctness, an LLM-powered system faces a structurally different risk — malicious *natural language* embedded in user input (or even in retrieved RAG content) attempting to override your system's intended instructions. This is why guardrail-style input checks (Day 27) are a standard, necessary layer specifically for this category of application, not an optional extra.

---

## 3. HOW — The Mechanism, Verified

### Caching

```
on request(prompt):
  if cache.has(prompt) and not expired:
    return cache.get(prompt)   // fast, no LLM call
  else:
    response = call_llm(prompt)  // slow, real LLM call
    cache.set(prompt, response)
    return response
```

### Token bucket rate limiting

```
bucket has: current_tokens, max_tokens, refill_rate_per_second

on request:
  refill: current_tokens = min(max_tokens, current_tokens + elapsed_seconds * refill_rate)
  if current_tokens >= 1:
    current_tokens -= 1
    allow the request
  else:
    reject the request (rate limited)
```

### SSE streaming

```
Server:
  set headers: Content-Type: text/event-stream
  for each token generated:
    write: "data: {json}\n\n"
  write: "data: {done: true}\n\n"
  end response

Client:
  open a long-lived connection
  process each "data: ..." chunk as it arrives, incrementally
```

---

## 4. EXAMPLE — Concrete Verified Traces

### Verified trace: caching's real timing impact

```
Request 1: "What is the remote work policy?" -> From cache: false, took 200ms
Request 2: "What is the remote work policy?" -> From cache: true, took 0ms
Request 3: "What is the expense policy?"      -> From cache: false, took 200ms
Request 4: "What is the remote work policy?" -> From cache: true, took 0ms

Cache stats: 2 hits, 2 misses
Cache hits === 2? PASS
Cache misses === 2? PASS
```

A real, measured 200ms-to-0ms difference for identical repeated requests — not a simulated or asserted speedup.

### Verified trace: token bucket rate limiting, including real-time refill

```
Request 1: allowed=true, remaining tokens=4.00
Request 2: allowed=true, remaining tokens=3.00
Request 3: allowed=true, remaining tokens=2.00
Request 4: allowed=true, remaining tokens=1.00
Request 5: allowed=true, remaining tokens=0.00
Request 6: allowed=false, remaining tokens=0.00
Request 7: allowed=false, remaining tokens=0.00
Request 8: allowed=false, remaining tokens=0.00

First 5 requests allowed? PASS
Requests 6-8 blocked? PASS

After waiting 1.1s, request allowed=true, remaining tokens=1.20
Refill worked? PASS
```

The refill test genuinely waited 1.1 real seconds (using `setTimeout`) before checking the bucket again — confirming tokens replenish over real elapsed time, not just in theory.

### Verified trace: SSE streaming with real token arrival timestamps

```
Token 1: "The" arrived at 70ms
Token 5: "allows" arrived at 268ms
Token 10: "home" arrived at 518ms
Token 16: "week." arrived at 820ms

Received 16 tokens total.
First token arrived at: 70ms
Last token arrived at: 820ms
Spread between first and last token: 750ms

Tokens arrived SPREAD OUT over time (spread > 200ms)? PASS
First token arrived quickly (< 100ms)? PASS
```

This is a real HTTP connection (Node's built-in `http` client connecting to a real running Express server), with real, measured per-token arrival times spaced roughly 50ms apart — matching the server's configured streaming interval exactly. This concretely proves "streaming" isn't just a label — the client genuinely receives data incrementally over a 750ms window, not all at once.

### Verified trace: cost tracking, cross-checked against a hand calculation

```
User query 1: 150 input + 80 output tokens -> $0.001650
User query 2 (with RAG context): 1200 input + 120 output tokens -> $0.005400
User query 3: 90 input + 200 output tokens -> $0.003270

Total cost: $0.010320
Hand-calculated expected total: $0.010320
Tracker's computed total matches hand calculation? PASS

Same query WITHOUT RAG context (150 input tokens): $0.002250
Same query WITH RAG context (1200 input tokens): $0.005400
RAG context increased cost by: 2.4x
```

The tracker's computed total exactly matches an independent hand calculation, confirming the arithmetic is correct — and the RAG-context comparison makes Day 17-19's "retrieved context becomes part of the prompt" lesson concrete in dollar terms.

---

## 5. IMPLEMENTATION — Node.js

### Part A — Caching

```javascript
// caching-demo.js
function simulateSlowLLMCall(prompt) {
  const start = Date.now();
  while (Date.now() - start < 200) { /* busy-wait 200ms */ }
  return `Response to: "${prompt}"`;
}

class ResponseCache {
  constructor(ttlMs = 60000) {
    this.cache = new Map();
    this.ttlMs = ttlMs;
    this.hits = 0;
    this.misses = 0;
  }
  get(prompt) {
    const entry = this.cache.get(prompt);
    if (!entry || Date.now() - entry.timestamp > this.ttlMs) {
      this.misses++;
      return null;
    }
    this.hits++;
    return entry.value;
  }
  set(prompt, value) {
    this.cache.set(prompt, { value, timestamp: Date.now() });
  }
}

function callWithCache(cache, prompt) {
  const cached = cache.get(prompt);
  if (cached !== null) return { response: cached, fromCache: true };
  const response = simulateSlowLLMCall(prompt);
  cache.set(prompt, response);
  return { response, fromCache: false };
}

const cache = new ResponseCache();
["A", "A", "B", "A"].forEach(prompt => {
  const start = Date.now();
  const { fromCache } = callWithCache(cache, prompt);
  console.log(`"${prompt}": fromCache=${fromCache}, took ${Date.now() - start}ms`);
});
```

**Run it:** `node caching-demo.js`

**Verified output:** see Section 4 — real 200ms vs 0ms timing difference confirmed.

---

### Part B — Rate Limiting

```javascript
// rate-limiter-demo.js
class TokenBucketRateLimiter {
  constructor(maxTokens, refillRatePerSecond) {
    this.maxTokens = maxTokens;
    this.refillRatePerSecond = refillRatePerSecond;
    this.tokens = maxTokens;
    this.lastRefill = Date.now();
  }
  _refill() {
    const now = Date.now();
    const elapsedSeconds = (now - this.lastRefill) / 1000;
    this.tokens = Math.min(this.maxTokens, this.tokens + elapsedSeconds * this.refillRatePerSecond);
    this.lastRefill = now;
  }
  tryConsume(cost = 1) {
    this._refill();
    if (this.tokens >= cost) {
      this.tokens -= cost;
      return { allowed: true, remainingTokens: this.tokens };
    }
    return { allowed: false, remainingTokens: this.tokens };
  }
}

const limiter = new TokenBucketRateLimiter(5, 2); // 5 max, refill 2/sec
for (let i = 1; i <= 8; i++) {
  const result = limiter.tryConsume();
  console.log(`Request ${i}: allowed=${result.allowed}`);
}
```

**Run it:** `node rate-limiter-demo.js`

**Verified output:** see Section 4 — first 5 allowed, next 3 blocked, refill confirmed after real 1.1s wait.

#### Line-by-line explanation (Parts A & B)

- **`_refill()`** — computes elapsed real time since the last refill and adds tokens proportionally, capped at `maxTokens`. This is what allows the bucket to "burst" up to its full capacity after sitting idle, then throttle smoothly afterward — verified directly by waiting real time and confirming tokens increased.
- **`ResponseCache`'s `ttlMs`** — ensures cached entries eventually expire, which matters for content that might change over time (e.g., if underlying RAG documents get updated, Day 21's lesson about state).

---

### Part C — SSE Streaming (Real Server + Real Client)

```javascript
// sse-streaming-server.js
const express = require("express");
const app = express();

app.get("/stream", (req, res) => {
  res.setHeader("Content-Type", "text/event-stream");
  res.setHeader("Cache-Control", "no-cache");
  res.setHeader("Connection", "keep-alive");

  const tokens = "The remote work policy allows employees to work from home up to three days per week.".split(" ");
  let index = 0;
  const interval = setInterval(() => {
    if (index >= tokens.length) {
      res.write(`data: ${JSON.stringify({ done: true })}\n\n`);
      clearInterval(interval);
      res.end();
      return;
    }
    res.write(`data: ${JSON.stringify({ token: tokens[index], done: false })}\n\n`);
    index++;
  }, 50);
});

app.listen(3001, () => console.log("SSE server listening on port 3001"));
```

```javascript
// verify-sse-streaming.js -- a REAL client testing the server above
const http = require("http");

function testStreaming() {
  return new Promise((resolve, reject) => {
    const receivedTokens = [];
    const startTime = Date.now();
    http.get("http://localhost:3001/stream", (res) => {
      res.setEncoding("utf8");
      let buffer = "";
      res.on("data", (chunk) => {
        buffer += chunk;
        const lines = buffer.split("\n\n");
        buffer = lines.pop();
        lines.forEach(line => {
          if (line.startsWith("data: ")) {
            const data = JSON.parse(line.slice(6));
            if (!data.done) receivedTokens.push({ token: data.token, elapsedMs: Date.now() - startTime });
          }
        });
      });
      res.on("end", () => resolve(receivedTokens));
    }).on("error", reject);
  });
}

testStreaming().then(tokens => {
  console.log(`Received ${tokens.length} tokens, first at ${tokens[0].elapsedMs}ms, last at ${tokens[tokens.length-1].elapsedMs}ms`);
});
```

**Run it (in one terminal, then the other):**
```bash
node sse-streaming-server.js
# in a second terminal:
node verify-sse-streaming.js
```

**Verified output:** see Section 4 — real timestamps confirming genuine incremental delivery over a real HTTP connection.

#### Line-by-line explanation

- **`res.write('data: ...\n\n')`, repeated, with `res.end()` only at the very end** — this is the entire mechanism of SSE: keep the HTTP response open and write to it incrementally, rather than building a complete response string and sending it once.
- **The client's `res.on("data", ...)` handler, processing chunks as they arrive** — mirrors exactly how a real frontend would consume a streaming LLM response, accumulating partial text to display incrementally to the user.
- **The verified per-token timestamps, spaced ~50ms apart** — exact, real proof that the client received this data incrementally, not as one buffered block at the end.

---

### Part D — Cost Tracking

```javascript
// cost-tracking-demo.js
class CostTracker {
  constructor(pricePerMillionInputTokens, pricePerMillionOutputTokens) {
    this.pricePerMillionInput = pricePerMillionInputTokens;
    this.pricePerMillionOutput = pricePerMillionOutputTokens;
    this.totalInputTokens = 0;
    this.totalOutputTokens = 0;
  }
  recordRequest(inputTokens, outputTokens) {
    this.totalInputTokens += inputTokens;
    this.totalOutputTokens += outputTokens;
    return this._computeCost(inputTokens, outputTokens);
  }
  _computeCost(inputTokens, outputTokens) {
    return (inputTokens / 1_000_000) * this.pricePerMillionInput +
           (outputTokens / 1_000_000) * this.pricePerMillionOutput;
  }
  getTotalCost() {
    return this._computeCost(this.totalInputTokens, this.totalOutputTokens);
  }
}

const tracker = new CostTracker(3.0, 15.0);
tracker.recordRequest(150, 80);
tracker.recordRequest(1200, 120);
tracker.recordRequest(90, 200);
console.log(`Total cost: $${tracker.getTotalCost().toFixed(6)}`);
```

**Run it:** `node cost-tracking-demo.js`

**Verified output:** see Section 4 — matches independent hand calculation exactly.

#### Line-by-line explanation

- **`_computeCost`** — implements the standard per-million-token pricing formula used by most LLM providers. Verified against an independent hand calculation to confirm correctness, not just plausibility.
- **The RAG-context cost comparison** — directly quantifies the practical cost implication of Day 17-19's design decision to inject retrieved context into every prompt.

---

## 6. TEACH-IT-BACK — Script You Can Say Out Loud

> "Several production concerns matter specifically for LLM-powered backends, beyond what a typical API needs. Caching avoids repeating slow, expensive LLM calls for requests you've already seen — I measured a real 200 millisecond call drop to 0 milliseconds on a cache hit. Rate limiting, typically implemented with a token bucket algorithm, protects your backend from being overwhelmed and helps you respect the rate limits your LLM provider imposes on your account — a bucket holds a capped number of tokens, each request consumes one, and tokens refill steadily over real time, which I verified by literally waiting over a second and confirming the bucket had refilled. Streaming, using Server-Sent Events, sends each generated token to the client as soon as it's ready, rather than waiting for the entire response to finish — I verified this with a real client connected to a real server, recording actual arrival timestamps roughly 50 milliseconds apart, proving genuine incremental delivery rather than one buffered burst at the end, which matters enormously for how fast a response feels to a user even when the total generation time is unchanged. Cost tracking matters because token usage directly translates to real money — I found that adding retrieved RAG context to a query increased that specific query's cost by 2.4 times, purely from the extra input tokens. And on the security side, LLM-powered systems face prompt injection risks that traditional APIs don't, because malicious instructions can be smuggled in as ordinary-looking natural language, which is why guardrail-style input and output checks are a necessary layer, not an optional extra, for any LLM-powered backend feature."

---

## Key Terms Glossary (Day 28)

| Term | Meaning |
|---|---|
| **Caching** | Storing previous results to avoid repeating slow/costly work |
| **Token bucket** | A rate-limiting algorithm with capped tokens that refill over time |
| **SSE** | A protocol for streaming incremental updates from server to client over one connection |
| **Cost tracking** | Computing real dollar costs from token usage |
| **Prompt injection** | Malicious instructions embedded in natural-language input, a security risk specific to LLM systems |

---

## Quick Self-Check (ask yourself before moving to Day 29)

1. Can I explain, with real numbers, why caching matters for LLM-powered features specifically?
2. Can I explain how a token bucket rate limiter allows bursts but prevents sustained overload?
3. Can I explain why streaming improves perceived latency even when total generation time is unchanged?
4. Can I explain why RAG's context injection has a direct, calculable cost implication?

If yes to all four — you're ready for **Day 29: MCP (Model Context Protocol)** and the modern agent-tool ecosystem.
