# Circuit Breaker

## What is a Circuit Breaker?

A Circuit Breaker is a design pattern that prevents an application from repeatedly trying to execute an operation that is likely to fail. Like a physical electrical circuit breaker — when there's a fault, it "trips" and stops current flow to prevent damage. When the fault is fixed, it resets and allows current flow again.

```
Without Circuit Breaker:
Payment Service -> Fraud Detection Service (down)
  -> Timeout after 5 sec
  -> Retry (another 5 sec)
  -> Timeout
  -> ... every request waits 10+ seconds before failing
  -> Payment Service threads exhausted
  -> Payment Service goes down too (cascading failure)

With Circuit Breaker:
Payment Service -> Fraud Detection Service (down)
  -> Circuit OPEN -> fail immediately (no waiting)
  -> Return fallback response
  -> Payment Service stays healthy
  -> Fraud Detection gets time to recover
```

## Why do we need it?

### 1. Prevent Cascading Failures

In a microservices system, services depend on each other. If Service B fails:

```
Service A -> Service B (down) -> A's threads wait for timeout
-> A runs out of threads -> A stops responding
-> Service C calling A also fails
-> Cascade continues until system is completely down
```

A circuit breaker stops the cascade at Service A — it fails fast instead of waiting.

### 2. Give Failing Services Time to Recover

When a circuit is open (tripped), requests stop flowing to the failing service. This removes load, giving it a chance to recover (restart, clear queues, etc.).

### 3. Fail Fast

Instead of waiting for a timeout (5–30 seconds), the circuit breaker returns an error or fallback response in milliseconds. This keeps latency predictable and threads/resources free.

### 4. Graceful Degradation

Return a fallback response (cached data, default value, friendly error) when the downstream service is unavailable — instead of crashing or hanging.

## Circuit Breaker States

The circuit breaker is a state machine with three states:

### CLOSED (Normal Operation)

Requests flow through normally. The circuit breaker monitors success/failure rate.

```
Requests -> [Circuit Breaker: CLOSED] -> Downstream Service
                    |
            counts failures
            if failure rate > threshold -> transition to OPEN
```

### OPEN (Tripped)

The downstream service has exceeded the failure threshold. Requests are immediately rejected without even trying to reach the downstream service.

```
Requests -> [Circuit Breaker: OPEN] -> Fail immediately (no downstream call)
                    |
            wait for timeout (e.g. 30 seconds)
            then transition to HALF-OPEN
```

### HALF-OPEN (Testing Recovery)

After the timeout, a small number of probe requests are allowed through to test if the downstream service has recovered.

```
Limited requests -> [Circuit Breaker: HALF-OPEN] -> Downstream Service
  if probe succeeds -> transition to CLOSED
  if probe fails    -> transition back to OPEN
```

### State Diagram

```
      failures > threshold
CLOSED ─────────────────────> OPEN
  ^                              |
  |    probe succeeds            | timeout expires
  |                              v
  └──────────────────────── HALF-OPEN
       probe fails -> OPEN
```

## Key Configuration Parameters

| Parameter | Description | Typical Value |
|---|---|---|
| `failureRateThreshold` | % of failures to trip the circuit | 50% |
| `slowCallRateThreshold` | % of slow calls (exceeding duration) to trip | 80% |
| `slowCallDurationThreshold` | Duration that counts as "slow" | 2 seconds |
| `minimumNumberOfCalls` | Min calls before calculating failure rate | 10 |
| `waitDurationInOpenState` | How long to stay OPEN before trying HALF-OPEN | 30 seconds |
| `permittedCallsInHalfOpenState` | How many probe calls in HALF-OPEN | 3 |
| `slidingWindowSize` | Number of recent calls to measure failure rate over | 10 or 100 |

## Resilience4j (Java — Modern Standard)

Resilience4j is the recommended circuit breaker library for Spring Boot (replacing Hystrix which is no longer maintained).

### Dependencies (pom.xml)

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

### Configuration (application.yml)

```yaml
resilience4j:
  circuitbreaker:
    instances:
      fraudService:
        failureRateThreshold: 50           # trip if 50%+ of calls fail
        slowCallRateThreshold: 80          # trip if 80%+ of calls are slow
        slowCallDurationThreshold: 2s      # calls > 2s count as slow
        minimumNumberOfCalls: 10           # need at least 10 calls to calculate
        waitDurationInOpenState: 30s       # stay OPEN for 30s before HALF-OPEN
        permittedCallsInHalfOpenState: 3   # allow 3 probe calls in HALF-OPEN
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 10

  retry:
    instances:
      fraudService:
        maxAttempts: 3
        waitDuration: 500ms
        retryExceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
```

### Usage in Service

```java
@Service
public class PaymentService {

    @Autowired
    private FraudDetectionClient fraudClient;

    @CircuitBreaker(name = "fraudService", fallbackMethod = "fraudCheckFallback")
    @Retry(name = "fraudService")
    @TimeLimiter(name = "fraudService")
    public CompletableFuture<FraudResult> checkFraud(Payment payment) {
        return CompletableFuture.supplyAsync(() ->
                fraudClient.check(payment)
        );
    }

    // Fallback method — called when circuit is OPEN or all retries exhausted
    public CompletableFuture<FraudResult> fraudCheckFallback(
            Payment payment, Exception e) {
        log.warn("Fraud service unavailable, using fallback. Error: {}", e.getMessage());
        // Return a safe default — allow payment with low risk score for now
        return CompletableFuture.completedFuture(
                new FraudResult(FraudStatus.UNKNOWN, 0.0)
        );
    }
}
```

### Monitoring Circuit Breaker State

```java
@Autowired
private CircuitBreakerRegistry circuitBreakerRegistry;

public CircuitBreakerStatus getStatus() {
    CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker("fraudService");
    return new CircuitBreakerStatus(
            cb.getState().name(),
            cb.getMetrics().getFailureRate(),
            cb.getMetrics().getNumberOfSuccessfulCalls(),
            cb.getMetrics().getNumberOfFailedCalls()
    );
}
```

Resilience4j also exposes Actuator endpoints at `/actuator/circuitbreakers` and metrics for Prometheus/Grafana out of the box.

## Node.js Implementation (opossum)

```javascript
const CircuitBreaker = require('opossum');
const axios = require('axios');

// The function to protect
async function callFraudService(payment) {
    const response = await axios.post('http://fraud-service/check', payment, {
        timeout: 2000
    });
    return response.data;
}

// Wrap it in a circuit breaker
const breaker = new CircuitBreaker(callFraudService, {
    timeout: 2000,           // if function takes longer than 2s, it fails
    errorThresholdPercentage: 50,  // trip if 50%+ fail
    resetTimeout: 30000,     // try again after 30 seconds
    volumeThreshold: 10      // minimum calls before tripping
});

// Fallback
breaker.fallback((payment) => ({
    status: 'UNKNOWN',
    riskScore: 0.0
}));

// Event listeners for monitoring
breaker.on('open',     () => console.log('Circuit OPEN  - fraud service unhealthy'));
breaker.on('halfOpen', () => console.log('Circuit HALF-OPEN - testing fraud service'));
breaker.on('close',    () => console.log('Circuit CLOSED - fraud service recovered'));

// Usage
async function processPayment(payment) {
    try {
        const fraudResult = await breaker.fire(payment);
        // proceed with payment
    } catch (err) {
        // circuit is open, fallback already applied
    }
}
```

## Circuit Breaker + Retry Interaction

Using retry and circuit breaker together needs the right order:

```
Retry wraps -> Circuit Breaker wraps -> Actual Call

Flow:
Retry attempts Call
  -> Circuit Breaker checks state
     -> CLOSED: makes call, fails -> Retry tries again
     -> After too many failures: Circuit OPENS
     -> OPEN: Retry tries again -> Circuit Breaker rejects immediately
     -> After 3 retries all rejected: Retry gives up -> Fallback
```

**Important:** if the circuit is OPEN, retries are pointless — the circuit will reject them immediately. Configure the retry to NOT retry on `CallNotPermittedException` (Resilience4j's exception for OPEN circuit):

```yaml
resilience4j:
  retry:
    instances:
      fraudService:
        ignoreExceptions:
          - io.github.resilience4j.circuitbreaker.CallNotPermittedException
```

## Bulkhead Pattern (Used Alongside Circuit Breaker)

A bulkhead limits the number of concurrent calls to a service — like ship bulkheads that prevent one flooded compartment from sinking the whole ship.

```java
@Bulkhead(name = "fraudService", type = Bulkhead.Type.THREADPOOL)
public CompletableFuture<FraudResult> checkFraud(Payment payment) {
    return fraudClient.check(payment);
}
```

```yaml
resilience4j:
  bulkhead:
    instances:
      fraudService:
        maxConcurrentCalls: 10   # max 10 concurrent calls to fraud service
        maxWaitDuration: 100ms   # wait up to 100ms for a slot before rejecting
```

Without a bulkhead, a slow downstream service can consume all available threads — a circuit breaker alone doesn't prevent thread exhaustion.

## Fallback Strategies

When the circuit is open or the call fails, what do you return?

| Strategy | Description | Example |
|---|---|---|
| Cached response | Return last known good value from cache | Show last known product price |
| Default value | Return a safe/neutral default | Fraud score = 0 (assume low risk) |
| Static response | Return a hardcoded response | "Service temporarily unavailable" message |
| Graceful degradation | Return partial data | Show order history without real-time tracking |
| Queue for later | Accept the request, process when service recovers | Queue payment for retry |

## Production Architecture

```
Client
  |
API Gateway (first layer rate limiting)
  |
Payment Service
  |
  +-- [Circuit Breaker] --> Fraud Detection Service
  |         |
  |     (OPEN: fallback)
  |
  +-- [Circuit Breaker] --> Inventory Service
  |         |
  |     (OPEN: fallback)
  |
  +-- [Circuit Breaker] --> Notification Service
            |
        (OPEN: queue for later)
```

## Interview Questions

**Q1: What problem does a circuit breaker solve?**

It prevents cascading failures in microservices. When a downstream service fails, without a circuit breaker, every request to the upstream service ties up a thread waiting for a timeout — eventually exhausting all threads and bringing the upstream service down too. A circuit breaker fails fast (in milliseconds) when the downstream is unhealthy, freeing resources and isolating the failure.

**Q2: Explain the three states of a circuit breaker.**

CLOSED: normal operation, requests flow through, failure rate is monitored. OPEN: failure threshold exceeded, all requests are immediately rejected with no downstream call — service gets time to recover. HALF-OPEN: after a timeout, a small number of probe requests test if the service has recovered; success closes the circuit, failure re-opens it.

**Q3: What is the difference between a Circuit Breaker and a Retry?**

Retry is for transient errors — temporary blips that will probably succeed if tried again. Circuit Breaker is for sustained failures — the downstream service is genuinely down. Using retry alone against a down service wastes resources and adds latency per attempt. Circuit breaker + retry together: retry for transient failures, and once the circuit opens, stop retrying entirely until the service recovers.

**Q4: What is a fallback and when should you use it?**

A fallback is an alternative response returned when the circuit is open or the call fails. Use it when you can meaningfully serve the client despite the downstream failure — e.g. return cached data, a default value, or degrade functionality gracefully. Don't use a fallback that silently hides errors when the user needs accurate information (e.g. silently approving a payment when fraud detection is down may not be acceptable).

**Q5: What is the Bulkhead pattern and how does it complement Circuit Breaker?**

Bulkhead limits concurrent calls to a service (or allocates a fixed thread pool per dependency). Circuit breaker trips when the failure rate is too high. Bulkhead prevents any one dependency from consuming all threads even before the circuit trips — it contains resource exhaustion. Together: bulkhead limits thread usage, circuit breaker stops calling failing services.

**Q6: How do you monitor a circuit breaker in production?**

Expose circuit breaker state and metrics via health endpoints (Spring Actuator `/actuator/circuitbreakers`), emit metrics to Prometheus, and visualize on Grafana. Alert when a circuit transitions to OPEN (a service is down). Track failure rate, slow call rate, and state transitions over time. Resilience4j integrates with Micrometer for all of this out of the box.
