# Redis Pub/Sub & Redis Streams

## PART 1 — REDIS PUB/SUB

## What is Redis Pub/Sub?

Redis Pub/Sub is a messaging pattern where publishers send messages to channels, and subscribers receive messages from channels they're subscribed to. It's fire-and-forget — messages are delivered to currently connected subscribers only. No persistence, no acknowledgement.

```
Publisher -> Channel: "notifications" -> Subscriber 1
                                      -> Subscriber 2
                                      -> Subscriber 3

If Subscriber 4 connects AFTER the message is sent:
  -> Subscriber 4 misses the message (no persistence)
```

## Redis Pub/Sub vs Kafka/RabbitMQ

| Feature | Redis Pub/Sub | Kafka | RabbitMQ |
|---|---|---|---|
| Persistence | None — fire and forget | Yes (configurable days) | Until ACKed |
| Offline consumers | Messages lost | Replay from offset | Queue holds messages |
| Delivery guarantee | At-most-once | At-least-once / exactly-once | At-least-once |
| Throughput | Very high (in-memory) | Very high | High |
| Best for | Real-time broadcasting, live notifications | Event streaming, audit log | Task queues |

## Redis Pub/Sub Commands

```bash
# Terminal 1: Subscribe to channel
SUBSCRIBE notifications
# Waiting for messages...

# Terminal 2: Subscribe to pattern (wildcard)
PSUBSCRIBE user:*     # receives: user:123, user:456, etc.
PSUBSCRIBE order:*

# Terminal 3: Publish a message
PUBLISH notifications "New order placed: ORD-123"
# -> Terminal 1 receives the message immediately
PUBLISH user:456 "Profile updated"
# -> Terminal 2 receives (matches user:*)
```

## Java Implementation (Spring Data Redis)

### Configuration

```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        return new LettuceConnectionFactory("localhost", 6379);
    }

    @Bean
    public RedisTemplate<String, String> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, String> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        template.setDefaultSerializer(new StringRedisSerializer());
        return template;
    }

    // Pub/Sub listener container
    @Bean
    public RedisMessageListenerContainer redisContainer(
            RedisConnectionFactory factory,
            NotificationListener notificationListener) {

        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(factory);

        // Subscribe to specific channel
        container.addMessageListener(notificationListener,
                new ChannelTopic("notifications"));

        // Subscribe to pattern
        container.addMessageListener(notificationListener,
                new PatternTopic("order:*"));

        return container;
    }
}
```

### Publisher

```java
@Service
public class NotificationPublisher {

    @Autowired
    private StringRedisTemplate redis;

    public void publishNotification(String message) {
        redis.convertAndSend("notifications", message);
    }

    public void publishOrderEvent(String orderId, String event) {
        String payload = String.format("{\"orderId\":\"%s\",\"event\":\"%s\",\"ts\":%d}",
                orderId, event, System.currentTimeMillis());
        redis.convertAndSend("order:" + orderId, payload);
    }
}
```

### Subscriber (Listener)

```java
@Component
public class NotificationListener implements MessageListener {

    @Override
    public void onMessage(Message message, byte[] pattern) {
        String channel = new String(message.getChannel());
        String body = new String(message.getBody());

        log.info("Received on channel [{}]: {}", channel, body);

        // Route to appropriate handler based on channel
        if (channel.equals("notifications")) {
            handleGlobalNotification(body);
        } else if (channel.startsWith("order:")) {
            String orderId = channel.substring(6);
            handleOrderEvent(orderId, body);
        }
    }

    private void handleGlobalNotification(String message) {
        // Push to connected WebSocket clients, etc.
        webSocketService.broadcastToAll(message);
    }

    private void handleOrderEvent(String orderId, String payload) {
        webSocketService.broadcastToOrderSubscribers(orderId, payload);
    }
}
```

## Node.js Pub/Sub

```javascript
const { createClient } = require('redis');

// Need SEPARATE clients for pub and sub
// A client in subscribe mode can ONLY subscribe, not execute other commands
const publisher  = createClient({ url: 'redis://localhost:6379' });
const subscriber = createClient({ url: 'redis://localhost:6379' });

await publisher.connect();
await subscriber.connect();

// Subscriber
await subscriber.subscribe('notifications', (message, channel) => {
    console.log(`Received on ${channel}: ${message}`);
    // broadcast to WebSocket clients
    io.emit('notification', JSON.parse(message));
});

// Pattern subscribe
await subscriber.pSubscribe('order:*', (message, channel) => {
    const orderId = channel.split(':')[1];
    console.log(`Order ${orderId} event: ${message}`);
});

// Publisher
function publishNotification(data) {
    publisher.publish('notifications', JSON.stringify(data));
}

function publishOrderEvent(orderId, event) {
    publisher.publish(`order:${orderId}`, JSON.stringify({ orderId, event }));
}
```

## Real-World Use Cases for Redis Pub/Sub

| Use Case | Channel Pattern | Description |
|---|---|---|
| Live notifications | `user:{userId}` | Notify a user of new messages/alerts |
| Order status updates | `order:{orderId}` | Push order status to tracking page |
| Chat rooms | `room:{roomId}` | Broadcast chat messages to room members |
| Live dashboard | `metrics` | Push real-time metrics to admin dashboard |
| Cache invalidation | `cache:invalidate` | Notify all servers to evict a cache key |
| WebSocket fan-out | `broadcast` | Push to all connected clients |

### Cache Invalidation Across Servers

```java
// When cache is invalidated on Server 1, notify all servers:
public void invalidateCache(String key) {
    redis.delete(key);                           // local cache delete
    redis.convertAndSend("cache:invalidate", key); // broadcast to all servers
}

// All servers listen and delete from their local caches:
@Component
public class CacheInvalidationListener implements MessageListener {
    @Autowired
    private CacheManager cacheManager;

    @Override
    public void onMessage(Message message, byte[] pattern) {
        String key = new String(message.getBody());
        cacheManager.getCache("products").evict(key);
    }
}
```

---

## PART 2 — REDIS STREAMS

## What are Redis Streams?

Redis Streams (added in Redis 5.0) is a persistent, append-only log data structure — like a more feature-rich version of Pub/Sub. Unlike Pub/Sub, messages are stored and can be consumed by late subscribers, acknowledged, and replayed.

```
Redis Pub/Sub:       fire-and-forget, no persistence
Redis Streams:       persistent log, consumer groups, acknowledgement, replay
                     → Mini Kafka inside Redis
```

## Stream Data Structure

A Stream is an ordered log of entries. Each entry has:
- An auto-generated ID: `<millisecondsTimestamp>-<sequenceNumber>` e.g. `1705312200000-0`
- A set of field-value pairs (like a hash)

```bash
# Add to stream
XADD order-events * eventType ORDER_PLACED orderId ORD-123 userId USR-456 total 99.99

# Read from stream
XRANGE order-events - +          # read all entries
XRANGE order-events 1705312200000-0 +  # read from specific ID onwards
XREVRANGE order-events + -       # read in reverse

# Stream length
XLEN order-events

# Read new messages (blocking, like a tail -f)
XREAD COUNT 10 BLOCK 5000 STREAMS order-events $  # $ = only new messages
```

## Consumer Groups

Consumer groups allow multiple consumers to process messages from the same stream, each consumer seeing different messages (like Kafka consumer groups). The stream tracks which messages have been acknowledged.

```bash
# Create consumer group (reads from beginning)
XGROUP CREATE order-events order-processor $ MKSTREAM
# $ = start from new messages only; 0 = start from beginning

# Read as a consumer in a group
XREADGROUP GROUP order-processor consumer-1
    COUNT 10 BLOCK 5000
    STREAMS order-events >
# > = only undelivered messages

# Acknowledge processed message
XACK order-events order-processor 1705312200000-0

# Read pending (unacknowledged) messages
XPENDING order-events order-processor - + 10
# Shows messages delivered but not yet ACKed (possibly failed)
```

## Java Implementation (Spring Data Redis Streams)

### Producer

```java
@Service
public class OrderStreamProducer {

    @Autowired
    private StringRedisTemplate redis;

    private static final String STREAM_KEY = "order-events";

    public void publishOrderEvent(String orderId, String eventType, BigDecimal total) {
        Map<String, String> fields = new HashMap<>();
        fields.put("eventType", eventType);
        fields.put("orderId", orderId);
        fields.put("total", total.toString());
        fields.put("timestamp", Instant.now().toString());

        RecordId id = redis.opsForStream().add(STREAM_KEY, fields);
        log.info("Published event {} with ID {}", eventType, id);
    }
}
```

### Consumer (with Consumer Group)

```java
@Service
public class OrderStreamConsumer {

    @Autowired
    private StringRedisTemplate redis;

    private static final String STREAM_KEY = "order-events";
    private static final String GROUP_NAME = "order-processor";
    private static final String CONSUMER_NAME = "consumer-" + UUID.randomUUID();

    @PostConstruct
    public void init() {
        // Create consumer group (idempotent)
        try {
            redis.opsForStream().createGroup(STREAM_KEY, ReadOffset.latest(), GROUP_NAME);
        } catch (RedisSystemException e) {
            // Group already exists — OK
        }

        // Start consuming in background thread
        CompletableFuture.runAsync(this::consumeMessages);
    }

    private void consumeMessages() {
        Consumer consumer = Consumer.from(GROUP_NAME, CONSUMER_NAME);
        StreamReadOptions options = StreamReadOptions.empty().count(10).block(Duration.ofSeconds(5));

        while (true) {
            try {
                List<MapRecord<String, String, String>> records = redis.opsForStream()
                        .read(consumer,
                              options,
                              StreamOffset.create(STREAM_KEY, ReadOffset.lastConsumed()));

                for (MapRecord<String, String, String> record : records) {
                    processRecord(record);
                }

            } catch (Exception e) {
                log.error("Error reading stream", e);
                Thread.sleep(1000);
            }
        }
    }

    private void processRecord(MapRecord<String, String, String> record) {
        try {
            Map<String, String> fields = record.getValue();
            String eventType = fields.get("eventType");
            String orderId = fields.get("orderId");

            log.info("Processing {} for order {}", eventType, orderId);

            switch (eventType) {
                case "ORDER_PLACED"    -> handleOrderPlaced(orderId, fields);
                case "ORDER_CANCELLED" -> handleOrderCancelled(orderId);
            }

            // Acknowledge — removes from pending list
            redis.opsForStream().acknowledge(STREAM_KEY, GROUP_NAME, record.getId());

        } catch (Exception e) {
            log.error("Failed to process record {}: {}", record.getId(), e.getMessage());
            // Don't ACK — will remain in pending list for retry
        }
    }
}
```

### Handling Pending Messages (Failed/Stuck Messages)

```java
@Scheduled(fixedDelay = 60000)  // every minute
public void processPendingMessages() {
    // Get messages pending for more than 30 seconds (probably failed)
    PendingMessagesSummary pending = redis.opsForStream()
            .pending(STREAM_KEY, GROUP_NAME);

    if (pending.getTotalPendingMessages() == 0) return;

    PendingMessages messages = redis.opsForStream().pending(
            STREAM_KEY,
            GROUP_NAME,
            Range.unbounded(),
            10L
    );

    for (PendingMessage msg : messages) {
        if (msg.getTotalDeliveryCount() > 3) {
            // Too many retries -> move to dead letter stream
            redis.opsForStream().add("order-events-dlq", msg.getIdAsString());
            redis.opsForStream().acknowledge(STREAM_KEY, GROUP_NAME, msg.getId());
        } else {
            // Reclaim and retry
            redis.opsForStream().claim(
                    STREAM_KEY, GROUP_NAME, CONSUMER_NAME,
                    Duration.ofSeconds(30), msg.getId()
            );
        }
    }
}
```

## Node.js Streams

```javascript
const { createClient } = require('redis');
const client = createClient();
await client.connect();

// Producer
async function publishOrderEvent(orderId, eventType, total) {
    const id = await client.xAdd('order-events', '*', {
        eventType,
        orderId,
        total: String(total),
        timestamp: new Date().toISOString()
    });
    console.log(`Published event with ID: ${id}`);
}

// Create consumer group
try {
    await client.xGroupCreate('order-events', 'order-processor', '$', { MKSTREAM: true });
} catch (e) {
    if (!e.message.includes('BUSYGROUP')) throw e;
}

// Consumer
async function startConsumer(consumerName) {
    while (true) {
        const results = await client.xReadGroup(
            'order-processor',
            consumerName,
            [{ key: 'order-events', id: '>' }],
            { COUNT: 10, BLOCK: 5000 }
        );

        if (!results) continue;

        for (const { messages } of results) {
            for (const { id, message } of messages) {
                try {
                    await processEvent(message);
                    await client.xAck('order-events', 'order-processor', id);
                } catch (err) {
                    console.error(`Failed to process ${id}:`, err);
                    // No ACK = stays in pending
                }
            }
        }
    }
}

startConsumer('consumer-1');
```

## Pub/Sub vs Streams — When to Use Which

| Scenario | Use Pub/Sub | Use Streams |
|---|---|---|
| Live chat messages | ✅ Instant broadcast, loss OK | ❌ Overkill |
| Real-time notifications | ✅ Fire-and-forget | ✅ If you need delivery guarantee |
| Task/job queue | ❌ No persistence | ✅ At-least-once with ACK |
| Event log / audit | ❌ No history | ✅ Persistent log |
| Multiple consumers, each need all messages | ❌ One sub per group | ✅ Multiple consumer groups |
| Cache invalidation broadcast | ✅ Fast, loss is fine | ❌ Overkill |
| Order processing pipeline | ❌ Events can be lost | ✅ Guaranteed delivery |

## Interview Questions

**Q1: What is the key limitation of Redis Pub/Sub?**

Messages are only delivered to currently connected subscribers — there's no persistence. If a subscriber is offline, messages published while it was down are permanently lost. It's pure fire-and-forget. For guaranteed delivery, use Redis Streams (or Kafka/RabbitMQ).

**Q2: How do Redis Streams differ from Redis Pub/Sub?**

Streams are persistent — messages are stored in an append-only log and remain available for new/late consumers. Streams support consumer groups (like Kafka) where multiple consumers split the work. Messages must be explicitly acknowledged; unacknowledged messages remain in a pending list for retry. Pub/Sub is ephemeral broadcast; Streams are persistent reliable messaging.

**Q3: What is a Consumer Group in Redis Streams?**

A named group of consumers that collectively process a stream. Each message is delivered to exactly one consumer in the group (load-balanced). Each group independently tracks its position in the stream — multiple groups each get all messages. Messages must be ACKed to remove them from the pending list; unACKed messages can be reclaimed and retried.

**Q4: How do you handle failed messages in Redis Streams?**

Unacknowledged messages remain in the Pending Entries List (PEL). A periodic job queries XPENDING to find messages pending for too long (indicating consumer failure). Messages can be reclaimed with XCLAIM for retry, or moved to a dead-letter stream after N failed attempts. This gives Redis Streams at-least-once delivery semantics.

**Q5: When would you use Redis Pub/Sub over Kafka for real-time features?**

Redis Pub/Sub for pure real-time broadcasting where message loss is acceptable and simplicity matters: live chat, real-time dashboards, cache invalidation, WebSocket fan-out. Kafka when you need persistence, replay, guaranteed delivery, or high throughput at scale. Redis Pub/Sub has lower latency and zero setup overhead for simple use cases.
