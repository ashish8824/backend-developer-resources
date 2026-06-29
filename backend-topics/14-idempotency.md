# Idempotency

## What is Idempotency?

An operation is idempotent if performing it multiple times produces the same result as performing it once. No matter how many times you repeat the operation, the outcome is identical to the first execution.

**Math analogy:**

```
f(x) = f(f(x))   <- idempotent function

Example: absolute value
||-5|| = |-5| = 5   <- same result on repeated application

Non-example: increment
++(++(5)) = 7 ≠ 6   <- different result each time
```

**In API terms:**

```
Idempotent:
POST /payments { orderId: "ORD-123", amount: 100 }
POST /payments { orderId: "ORD-123", amount: 100 }  <- same request again
Result: charged once, $100 — even if called 10 times

Non-idempotent (broken):
POST /payments { orderId: "ORD-123", amount: 100 }
POST /payments { orderId: "ORD-123", amount: 100 }
Result: charged twice, $200 — double charge bug
```

## Why is Idempotency Critical?

### The Retry Problem

Networks are unreliable. What happens when a request is sent but no response comes back?

```
Client -> POST /payments -> Server (processes payment, charges $100)
                         -> Response: 200 OK
                         -> Response lost in network
Client: did it work? No response received...
Client retries -> POST /payments -> Server processes AGAIN -> charges $200!
```

The client has no way to know if the server processed the request or if the response was lost. Without idempotency, retrying causes double execution.

### This Happens More Than You Think

- Mobile apps on flaky connections
- Load balancer retries on timeout
- Message queue at-least-once delivery (duplicate messages)
- Client-side retry logic (exponential backoff)
- HTTP/2 connection resets
- Kubernetes pod restarts mid-request

## HTTP Methods and Idempotency

| Method | Idempotent | Safe (no side effects) | Example |
|---|---|---|---|
| GET | ✅ Yes | ✅ Yes | Fetch user profile |
| HEAD | ✅ Yes | ✅ Yes | Check if resource exists |
| OPTIONS | ✅ Yes | ✅ Yes | CORS preflight |
| PUT | ✅ Yes | ❌ No | Replace entire resource |
| DELETE | ✅ Yes | ❌ No | Delete resource |
| POST | ❌ No (by default) | ❌ No | Create resource, payment |
| PATCH | ❌ No (by default) | ❌ No | Partial update |

**Why is DELETE idempotent?**

```
DELETE /users/123  -> 200 OK (user deleted)
DELETE /users/123  -> 404 Not Found (already deleted)
```

The end state is the same — the user doesn't exist. The second call returns a different status code, but the state of the system is identical.

**Why is PUT idempotent but PATCH not?**

```
PUT /users/123 { name: "Alice", email: "alice@example.com" }
PUT /users/123 { name: "Alice", email: "alice@example.com" }
Result: same user state — idempotent ✅

PATCH /users/123 { loginCount: +1 }
PATCH /users/123 { loginCount: +1 }
Result: loginCount incremented twice — NOT idempotent ❌

PATCH /users/123 { name: "Alice" }  <- set to value, not increment
PATCH /users/123 { name: "Alice" }
Result: same state — idempotent ✅ (if it's a SET, not an increment)
```

## Idempotency Keys

The standard mechanism for making non-idempotent operations (POST, PATCH) safe to retry.

**How it works:**

1. Client generates a unique key (UUID) for the request
2. Client sends the key in a header: `Idempotency-Key: uuid-abc-123`
3. Server checks if it has already processed a request with this key
4. If YES: return the stored result without re-executing
5. If NO: execute the operation, store the result mapped to the key, return result

```
Client generates: key = UUID.randomUUID() = "550e8400-e29b-41d4-a716-446655440000"

First call:
Client -> POST /payments
          Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
          { amount: 100, orderId: "ORD-123" }

Server:
  -> Check Redis/DB: key "550e8400..." not found
  -> Process payment: charge $100
  -> Store: "550e8400..." -> { status: 200, body: { paymentId: "PAY-789" } }
  -> Return: 200 OK { paymentId: "PAY-789" }

Network drops response. Client retries:

Second call (retry):
Client -> POST /payments
          Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
          { amount: 100, orderId: "ORD-123" }

Server:
  -> Check Redis/DB: key "550e8400..." FOUND
  -> Return stored result: 200 OK { paymentId: "PAY-789" }  (no re-execution)
```

Payment charged exactly once, even though the request was sent twice.

## Java Implementation (Spring Boot)

### Idempotency Key Table (DB Schema)

```sql
CREATE TABLE idempotency_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    idempotency_key VARCHAR(255) UNIQUE NOT NULL,
    request_hash    VARCHAR(255),        -- optional: detect same key, different body
    response_status INTEGER NOT NULL,
    response_body   JSONB NOT NULL,
    created_at      TIMESTAMP DEFAULT NOW(),
    expires_at      TIMESTAMP            -- TTL: auto-cleanup old keys
);

CREATE INDEX idx_idempotency_key ON idempotency_keys(idempotency_key);
```

### Idempotency Filter / Interceptor

```java
@Component
public class IdempotencyInterceptor implements HandlerInterceptor {

    @Autowired
    private IdempotencyKeyRepository idempotencyRepo;

    @Autowired
    private ObjectMapper objectMapper;

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) throws Exception {

        // Only apply to POST and PATCH
        String method = request.getMethod();
        if (!method.equals("POST") && !method.equals("PATCH")) {
            return true;
        }

        String idempotencyKey = request.getHeader("Idempotency-Key");
        if (idempotencyKey == null) {
            return true;  // key not required for all endpoints; enforce per-endpoint
        }

        // Check if already processed
        Optional<IdempotencyRecord> existing =
                idempotencyRepo.findByKey(idempotencyKey);

        if (existing.isPresent()) {
            IdempotencyRecord record = existing.get();
            response.setStatus(record.getResponseStatus());
            response.setContentType("application/json");
            response.getWriter().write(record.getResponseBody());
            response.getWriter().flush();
            return false;  // stop further processing — return cached response
        }

        // Store key in request attribute for postHandle to save
        request.setAttribute("idempotencyKey", idempotencyKey);
        return true;
    }
}
```

### Payment Service with Idempotency

```java
@Service
public class PaymentService {

    @Autowired
    private PaymentRepository paymentRepo;

    @Autowired
    private IdempotencyKeyRepository idempotencyRepo;

    @Transactional
    public PaymentResult processPayment(PaymentRequest request, String idempotencyKey) {

        // Double-check within transaction (race condition protection)
        Optional<IdempotencyRecord> existing =
                idempotencyRepo.findByKey(idempotencyKey);

        if (existing.isPresent()) {
            // Already processed — return stored result
            return objectMapper.readValue(
                    existing.get().getResponseBody(), PaymentResult.class);
        }

        // Process payment
        Payment payment = new Payment();
        payment.setOrderId(request.getOrderId());
        payment.setAmount(request.getAmount());
        payment.setStatus(PaymentStatus.COMPLETED);
        paymentRepo.save(payment);

        PaymentResult result = new PaymentResult(payment.getId(), "SUCCESS");

        // Store idempotency record in SAME transaction
        IdempotencyRecord record = new IdempotencyRecord();
        record.setIdempotencyKey(idempotencyKey);
        record.setResponseStatus(200);
        record.setResponseBody(objectMapper.writeValueAsString(result));
        record.setExpiresAt(LocalDateTime.now().plusDays(7));
        idempotencyRepo.save(record);

        return result;
    }
}
```

**Critical:** the payment and the idempotency record must be saved in the **same transaction**. If you save the payment and crash before saving the idempotency record, the next retry would process the payment again.

### Redis-Based Idempotency (Simpler, with TTL)

```java
@Service
public class RedisIdempotencyService {

    @Autowired
    private StringRedisTemplate redis;

    private static final long TTL_SECONDS = 86400; // 24 hours

    public Optional<String> getStoredResponse(String key) {
        String value = redis.opsForValue().get("idem:" + key);
        return Optional.ofNullable(value);
    }

    public boolean tryLock(String key) {
        // SET NX — only set if not exists (atomic)
        Boolean set = redis.opsForValue().setIfAbsent(
                "idem:lock:" + key,
                "processing",
                Duration.ofSeconds(30)
        );
        return Boolean.TRUE.equals(set);
    }

    public void storeResponse(String key, String responseJson) {
        redis.opsForValue().set(
                "idem:" + key,
                responseJson,
                Duration.ofSeconds(TTL_SECONDS)
        );
        redis.delete("idem:lock:" + key);
    }
}
```

## Node.js Implementation

```javascript
const { v4: uuidv4 } = require('uuid');
const redis = require('ioredis');

const client = new redis();

async function idempotentMiddleware(req, res, next) {
    const idempotencyKey = req.headers['idempotency-key'];

    if (!idempotencyKey) {
        return next();
    }

    const cached = await client.get(`idem:${idempotencyKey}`);

    if (cached) {
        const stored = JSON.parse(cached);
        return res.status(stored.status).json(stored.body);
    }

    // Intercept response to cache it
    const originalJson = res.json.bind(res);
    res.json = async (body) => {
        if (res.statusCode < 500) {
            await client.setex(
                `idem:${idempotencyKey}`,
                86400,  // 24 hour TTL
                JSON.stringify({ status: res.statusCode, body })
            );
        }
        return originalJson(body);
    };

    next();
}

// Client-side: generate key per logical operation
async function placeOrder(orderData) {
    const idempotencyKey = uuidv4(); // generated once per order attempt

    for (let attempt = 0; attempt < 3; attempt++) {
        try {
            const response = await fetch('/api/payments', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                    'Idempotency-Key': idempotencyKey  // same key on every retry
                },
                body: JSON.stringify(orderData)
            });
            return await response.json();
        } catch (err) {
            if (attempt === 2) throw err;
            await sleep(1000 * Math.pow(2, attempt));  // exponential backoff
        }
    }
}
```

## Idempotency in Message Queues

Message queues guarantee at-least-once delivery — consumers must handle duplicate messages.

**Pattern: track processed message IDs**

```java
@KafkaListener(topics = "payment-events")
public void processPaymentEvent(PaymentEvent event, Acknowledgment ack) {

    String messageId = event.getMessageId();

    // Check if already processed
    if (processedEventRepo.existsById(messageId)) {
        log.info("Duplicate event {}, skipping", messageId);
        ack.acknowledge();
        return;
    }

    // Process
    processPayment(event);

    // Mark as processed
    processedEventRepo.save(new ProcessedEvent(messageId, LocalDateTime.now()));
    ack.acknowledge();
}
```

**Deduplication window:** you don't need to store every message ID forever. Store them for a window longer than your maximum expected duplicate delay (e.g. 24 hours for most systems, 7 days for financial systems).

## Idempotency vs Exactly-Once

| | Idempotency | Exactly-Once |
|---|---|---|
| Where logic lives | Application layer | Infrastructure layer (Kafka EOS, DB transactions) |
| Mechanism | Deduplication by key | Transactional guarantees |
| Handles | Duplicate requests/messages | Duplicate at the messaging level |
| Cost | Medium (key storage + lookup) | High (distributed coordination) |
| Use case | APIs, message consumers | High-throughput Kafka pipelines |

Idempotency at the application layer is usually the right answer — you don't depend on infrastructure guarantees and it works across any transport.

## Real-World Examples

| System | How They Implement Idempotency |
|---|---|
| Stripe | `Idempotency-Key` header; stores results for 24 hours per key |
| PayPal | `PayPal-Request-Id` header; deduplicates payment requests |
| AWS S3 PUT | Inherently idempotent — PUT replaces the object, same result always |
| Kafka consumers | Consumer tracks processed message IDs in a DB/Redis |
| Webhook delivery | Receiver stores event ID and deduplicates on re-delivery |

## Key Design Decisions

### 1. TTL for Idempotency Keys

Don't store forever — use a TTL:

```
Financial APIs:   7 days (to cover delayed retries, reconciliation windows)
Payment APIs:     24 hours
Standard APIs:    1–2 hours
Message queues:   Duration of max replay window (hours to days)
```

### 2. What to Store as the Response

Store the full response (status code + body). On duplicate, replay the exact same response — the client should receive identical responses on retry.

### 3. Race Condition Between Duplicate Requests

If two identical requests arrive simultaneously (before the first finishes processing):

```
T1: Check key -> not found -> start processing
T2: Check key -> not found (T1 hasn't saved yet) -> start processing too!
```

Fix: use a database unique constraint on the idempotency key + handle the unique constraint violation gracefully, OR use Redis `SET NX` to acquire a "processing lock" before checking.

## Interview Questions

**Q1: What is idempotency and why does it matter in distributed systems?**

An operation is idempotent if repeated execution produces the same result as a single execution. In distributed systems, network failures force clients to retry operations without knowing if the server processed them. Without idempotency, retries cause duplicate operations (double charges, duplicate orders). Idempotency makes retries safe — a fundamental requirement for reliable APIs.

**Q2: Which HTTP methods are idempotent by specification and which aren't?**

GET, PUT, DELETE, HEAD, OPTIONS are idempotent. POST and PATCH are not (by default). PUT is idempotent because it fully replaces a resource to the same state every time. DELETE is idempotent because the end state (resource gone) is the same regardless of how many times you call it. POST typically creates a new resource each time.

**Q3: How do you make a POST endpoint idempotent?**

Use an idempotency key — the client sends a unique UUID in an `Idempotency-Key` header. The server checks if a request with that key has already been processed. If yes, return the stored result without re-executing. If no, execute and store the result mapped to that key. Both the business operation and the idempotency record must be saved in the same DB transaction.

**Q4: Where should idempotency keys be stored?**

Options: a dedicated DB table (durable, supports complex queries, harder to expire), or Redis (fast lookup, built-in TTL for auto-expiry, less durable). For financial systems, a DB table is safer — you want the record to survive Redis restarts. For general APIs, Redis with an appropriate TTL is simpler and faster.

**Q5: How do you handle duplicate messages in a Kafka consumer?**

Store the message ID (or a unique business identifier) in a DB or Redis after successful processing. On each incoming message, check if the ID has already been processed — if yes, skip. Use a TTL or expiry window to avoid storing IDs forever. This pattern is required because Kafka guarantees at-least-once delivery, not exactly-once.

**Q6: What is the race condition in idempotency checks and how do you fix it?**

If two identical requests arrive simultaneously, both may find the key absent and proceed to process simultaneously — defeating idempotency. Fix with: a database unique constraint on the idempotency key (the second insert fails with a constraint violation, which you handle by returning the first result), or a Redis `SET NX` lock acquired before processing (only one acquires the lock, the other waits).
