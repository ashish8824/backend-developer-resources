# Message Queue

## What is a Message Queue?

A Message Queue is a form of asynchronous communication between services. Instead of Service A calling Service B directly and waiting for a response, Service A puts a message into a queue. Service B picks it up and processes it independently — at its own pace.

```
Without MQ (Synchronous):
User -> Payment Service -> Email Service (wait...) -> SMS Service (wait...) -> Response
Total time: 3 seconds

With MQ (Asynchronous):
User -> Payment Service -> Queue -> Response (instant)
                                 -> Email Service (processes independently)
                                 -> SMS Service (processes independently)
```

## Why do we need it?

### 1. Decoupling

Services don't need to know about each other. The payment service doesn't call the email service directly — it just publishes an event. The email service subscribes to it. You can add/remove consumers without touching the producer.

### 2. Absorb Traffic Spikes (Load Leveling)

```
Black Friday: 100,000 orders/minute
Order processing capacity: 10,000 orders/minute

Without MQ: system crashes — requests pile up, timeouts everywhere
With MQ:    100,000 messages queue up, processed at 10,000/min steadily over 10 min
```

The queue acts as a buffer — producers can spike without killing consumers.

### 3. Reliability

If the consumer crashes mid-processing, the message stays in the queue (or is returned to it) and gets redelivered when the consumer recovers. No data lost.

### 4. Parallel Processing

Multiple consumers can process messages from the same queue simultaneously, scaling throughput horizontally.

### 5. Retry & Error Handling

Failed messages can be retried automatically, moved to a Dead Letter Queue (DLQ) after too many failures, and investigated separately — without losing the original message.

## Core Concepts

### Message

The unit of data passed between services. Usually JSON or binary (Avro, Protobuf).

```json
{
  "eventType": "ORDER_PLACED",
  "orderId": "ORD-12345",
  "userId": "USR-67890",
  "total": 499.99,
  "timestamp": "2025-01-15T10:30:00Z"
}
```

### Producer (Publisher)

The service that creates and sends messages to the queue.

### Consumer (Subscriber)

The service that reads and processes messages from the queue.

### Queue

A buffer that holds messages until a consumer picks them up. FIFO (First In, First Out) by default.

### Topic

In pub/sub systems (Kafka, SNS), messages are published to a topic. Multiple consumer groups can each independently consume from the same topic — each group gets all messages.

### Exchange (RabbitMQ-specific)

In RabbitMQ, producers don't send directly to queues — they send to an exchange, which routes messages to queues based on routing rules.

## Messaging Patterns

### 1. Point-to-Point (Queue)

One producer, one consumer group. Each message is processed by exactly one consumer.

```
Producer -> [Queue] -> Consumer A
                    -> Consumer B  (one of them gets each message, not both)
```

Use case: task distribution — e.g. resize this image, send this email.

### 2. Publish-Subscribe (Topic)

One producer, multiple independent consumer groups. Each group gets a copy of every message.

```
Producer -> [Topic] -> Consumer Group 1 (Email Service) — gets all messages
                    -> Consumer Group 2 (SMS Service)   — gets all messages
                    -> Consumer Group 3 (Analytics)     — gets all messages
```

Use case: event broadcasting — "order placed" needs to trigger email, SMS, inventory update, analytics simultaneously.

### 3. Request-Reply

Asynchronous RPC over a queue — producer sends a message to a request queue and includes a reply-to queue address. Consumer processes and sends the result to the reply queue.

```
Producer -> [Request Queue] -> Consumer
Consumer -> [Reply Queue]   -> Producer
```

Use case: long-running tasks where the caller wants a result eventually (e.g. report generation).

### 4. Fan-Out

A single message is broadcast to multiple queues simultaneously. RabbitMQ fanout exchange does this.

```
Producer -> Exchange (fanout) -> Queue 1 -> Consumer 1
                              -> Queue 2 -> Consumer 2
                              -> Queue 3 -> Consumer 3
```

## Dead Letter Queue (DLQ)

A special queue where messages go when they can't be processed successfully after a configured number of retries.

```
Message -> Consumer -> Fails -> Retry 1 -> Fails -> Retry 2 -> Fails -> DLQ
```

Why DLQ matters:
- Prevents a "poison message" (one that always fails) from clogging the main queue
- Failed messages are preserved for investigation and manual replay
- The main queue keeps flowing even when some messages have errors

**Common DLQ patterns:**

```
DLQ -> Alert engineers (PagerDuty/Slack notification)
DLQ -> Admin UI to inspect and replay manually
DLQ -> Scheduled job that retries DLQ messages at off-peak hours
```

## Message Delivery Guarantees

### At-Most-Once

Message is delivered zero or one times. If the consumer crashes before acknowledging, the message is lost (not redelivered).

```
Producer -> Queue -> Consumer (processes message, crashes before ACK)
Message is NOT redelivered — it's lost.
```

Use when: loss is acceptable — analytics, telemetry, metrics where occasional data loss is fine.

### At-Least-Once

Message is delivered one or more times. If the consumer crashes before acknowledging, the message is redelivered — possibly processed multiple times.

```
Producer -> Queue -> Consumer (processes message, crashes before ACK)
Message IS redelivered -> Consumer processes it again (duplicate!)
```

Use when: you can't afford to lose messages, and your processing is idempotent (safe to run twice).

### Exactly-Once

Message is delivered and processed exactly once, even under failure. The hardest to implement — requires coordination between queue and consumer storage (e.g. transactional outbox, idempotency keys).

Use when: financial transactions, inventory updates — operations where duplicates cause real damage.

## RabbitMQ

RabbitMQ is a traditional message broker implementing the AMQP protocol. Good for complex routing, task queues, and RPC patterns.

### Key Components

```
Producer -> Exchange -> Binding -> Queue -> Consumer
```

- **Exchange types:**
  - `direct` — routes by exact routing key match
  - `topic` — routes by pattern matching (e.g. `order.*` matches `order.placed`, `order.cancelled`)
  - `fanout` — sends to all bound queues
  - `headers` — routes by message header attributes

### RabbitMQ Flow

```
1. Producer publishes message to Exchange with routing key "order.placed"
2. Exchange matches routing key to bound queues
3. Queue holds message
4. Consumer picks up message, processes it
5. Consumer sends ACK -> Queue deletes message
   (or NACK -> Queue requeues or DLQs message)
```

### Java (Spring AMQP)

```java
// Config
@Configuration
public class RabbitConfig {

    @Bean
    public Queue orderQueue() {
        return QueueBuilder.durable("order.queue")
                .withArgument("x-dead-letter-exchange", "dlx")
                .withArgument("x-dead-letter-routing-key", "order.dlq")
                .build();
    }

    @Bean
    public TopicExchange orderExchange() {
        return new TopicExchange("order.exchange");
    }

    @Bean
    public Binding binding(Queue orderQueue, TopicExchange orderExchange) {
        return BindingBuilder.bind(orderQueue)
                .to(orderExchange)
                .with("order.*");
    }
}

// Producer
@Service
public class OrderProducer {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    public void publishOrder(Order order) {
        rabbitTemplate.convertAndSend(
                "order.exchange",
                "order.placed",
                order
        );
    }
}

// Consumer
@Service
public class OrderConsumer {

    @RabbitListener(queues = "order.queue")
    public void handleOrder(Order order) {
        // process the order
        processOrder(order);
        // ACK is sent automatically on successful return
        // if exception thrown -> NACK -> retry or DLQ
    }
}
```

## Kafka vs RabbitMQ

| Feature | RabbitMQ | Kafka |
|---|---|---|
| Model | Message broker (push-based) | Distributed log (pull-based) |
| Message retention | Deleted after ACK | Retained for configurable period (days/weeks) |
| Consumer groups | Competing consumers on a queue | Each group independently reads from offset |
| Message ordering | Per-queue FIFO | Per-partition ordering |
| Throughput | High (tens of thousands/sec) | Very high (millions/sec) |
| Replay messages | Not natively (need DLQ) | Yes — rewind consumer offset |
| Routing | Rich (exchange types, bindings) | Simple (topic + partition key) |
| Use case | Task queues, RPC, complex routing | Event streaming, audit logs, high-throughput pipelines |

## Node.js Implementation (with `amqplib`)

```javascript
const amqp = require('amqplib');

// Producer
async function publishOrder(order) {
    const conn = await amqp.connect('amqp://localhost');
    const channel = await conn.createChannel();

    await channel.assertExchange('order.exchange', 'topic', { durable: true });

    channel.publish(
        'order.exchange',
        'order.placed',
        Buffer.from(JSON.stringify(order)),
        { persistent: true }   // survives broker restart
    );

    await channel.close();
    await conn.close();
}

// Consumer
async function startConsumer() {
    const conn = await amqp.connect('amqp://localhost');
    const channel = await conn.createChannel();

    await channel.assertQueue('order.queue', { durable: true });
    channel.prefetch(10);  // process max 10 messages at once

    channel.consume('order.queue', async (msg) => {
        if (!msg) return;
        try {
            const order = JSON.parse(msg.content.toString());
            await processOrder(order);
            channel.ack(msg);    // success -> remove from queue
        } catch (err) {
            console.error('Processing failed:', err);
            channel.nack(msg, false, false);  // failure -> send to DLQ
        }
    });
}
```

## Transactional Outbox Pattern

The hardest problem in async messaging: how do you atomically save to your DB AND publish a message to a queue — without risk of one succeeding and the other failing?

```
// Wrong approach — not atomic:
db.save(order);           // succeeds
queue.publish(orderEvent); // CRASHES -> order saved but event never published
// downstream services never process this order
```

**Solution — Outbox Pattern:**

```
1. In the same DB transaction:
   - Save the order to orders table
   - Save the event to outbox table (same transaction, same DB)
   COMMIT

2. A separate "outbox poller" process:
   - Reads unpublished events from outbox table
   - Publishes them to the message queue
   - Marks them published in the outbox table
```

```sql
-- outbox table
CREATE TABLE outbox (
    id UUID PRIMARY KEY,
    event_type VARCHAR(100),
    payload JSONB,
    published BOOLEAN DEFAULT false,
    created_at TIMESTAMP
);
```

```java
@Transactional
public void placeOrder(Order order) {
    orderRepo.save(order);          // save order

    OutboxEvent event = new OutboxEvent(
            UUID.randomUUID(),
            "ORDER_PLACED",
            toJson(order)
    );
    outboxRepo.save(event);         // save event in same transaction
}

// Scheduled poller (separate process or @Scheduled)
@Scheduled(fixedDelay = 1000)
public void pollOutbox() {
    List<OutboxEvent> pending = outboxRepo.findByPublished(false);
    for (OutboxEvent event : pending) {
        messageQueue.publish(event);
        event.setPublished(true);
        outboxRepo.save(event);
    }
}
```

Either both the order and the event are saved, or neither is — the outbox guarantees at-least-once delivery to the queue without distributed transactions.

## Production Architecture

```
Order Service
  |
  +-- DB (orders + outbox) -- Outbox Poller
                                   |
                            Message Broker (RabbitMQ / Kafka)
                                   |
                    +--------------+-------------+
                    |              |             |
               Email Service  SMS Service  Inventory Service
                    |
                  (fail)
                    |
                   DLQ --> Alert / Manual Replay
```

## Interview Questions

**Q1: What is the difference between a Message Queue and a Topic?**

A queue delivers each message to one consumer (competing consumers share the work). A topic delivers each message to every subscriber independently — different consumer groups each get their own copy of all messages. Use queues for task distribution; use topics for event broadcasting.

**Q2: What is a Dead Letter Queue and why is it important?**

A DLQ holds messages that repeatedly failed processing. It prevents a poison message from blocking the main queue indefinitely, preserves failed messages for investigation, and keeps the main processing flow unblocked. Without it, one bad message could cause the consumer to retry forever and process nothing else.

**Q3: What is the difference between At-Least-Once and Exactly-Once delivery?**

At-least-once: messages are never lost but may be delivered multiple times on failure/retry — consumers must be idempotent. Exactly-once: each message is processed precisely once even under failure — much harder, requires coordination (transactional producers, idempotent consumers with deduplication). Most real-world systems use at-least-once with idempotent consumers.

**Q4: How do you atomically save to the DB and publish a message?**

Use the Transactional Outbox Pattern: write both the business data and the event to the DB in a single transaction. A separate outbox poller reads unpublished events and sends them to the queue, then marks them published. This guarantees both succeed or neither does, without a distributed transaction.

**Q5: What does `prefetch` / `maxConcurrency` control in a consumer?**

It limits how many messages a consumer fetches from the broker at once before acknowledging. Setting `prefetch = 10` means the broker delivers up to 10 messages to a consumer; it won't send more until some are acknowledged. This prevents a slow consumer from being overwhelmed and controls back-pressure.

**Q6: When would you choose RabbitMQ over Kafka?**

RabbitMQ for: complex routing needs (topic/header exchanges), traditional task queues, RPC patterns, when messages should be deleted after processing. Kafka for: high-throughput event streams, audit logs, event replay/rewind, fan-out to many consumer groups, when you need to retain messages for days/weeks for re-processing.
