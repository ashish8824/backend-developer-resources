# Kafka Basics

## What is Kafka?

Apache Kafka is a distributed event streaming platform. At its core, it's a high-throughput, fault-tolerant, durable message log. Producers write events to Kafka; consumers read them — independently, at their own pace, and potentially replaying old events.

```
Traditional MQ (RabbitMQ):
Producer -> [Queue] -> Consumer (message deleted after consumption)

Kafka:
Producer -> [Topic Log] -> Consumer Group A (reads at its own offset)
                        -> Consumer Group B (reads at its own offset, independently)
                        -> Consumer Group C (can replay from the beginning)
```

## Why Kafka?

### 1. High Throughput

Kafka can handle millions of messages per second per broker. LinkedIn (who built it) uses it to process trillions of messages per day.

### 2. Durability

Messages are persisted to disk and replicated across brokers. Unlike in-memory queues, data survives broker restarts.

### 3. Replay

Unlike traditional MQs where messages are deleted after consumption, Kafka retains messages for a configurable period (days, weeks). Any consumer can re-read old messages — useful for rebuilding state, debugging, or adding a new service that needs historical data.

### 4. Decoupling

Producers and consumers are completely independent. Producers don't know who's consuming or how many consumers exist. New consumers can be added without changing the producer.

### 5. Horizontal Scalability

Both producers and consumers scale horizontally. Partitions allow parallel processing across consumer instances.

## Core Concepts

### Event (Message)

The unit of data in Kafka. Has a key, value, timestamp, and optional headers.

```json
{
  "key": "user-123",
  "value": {
    "eventType": "ORDER_PLACED",
    "orderId": "ORD-456",
    "total": 299.99
  },
  "timestamp": 1705312200000,
  "headers": { "source": "order-service" }
}
```

### Topic

A named channel to which events are published. Think of it like a database table, but for a stream of events.

```
Topics:
- order-events
- payment-events
- user-events
- inventory-updates
```

### Partition

Each topic is split into one or more partitions. A partition is an ordered, immutable log of events. Events in a partition are guaranteed to be ordered; across partitions, ordering is not guaranteed.

```
Topic: order-events (3 partitions)
Partition 0: [event1, event4, event7, ...]
Partition 1: [event2, event5, event8, ...]
Partition 2: [event3, event6, event9, ...]
```

**Why partitions?**
- Parallelism: multiple consumer instances can read different partitions simultaneously
- Scalability: more partitions = more throughput (up to a point)
- Ordering: all events with the same key go to the same partition (hash of key % numPartitions), guaranteeing order per key

### Offset

A sequential, monotonically increasing integer that identifies each event's position within a partition.

```
Partition 0: offset 0, offset 1, offset 2, offset 3, ...
```

Consumers track their current position using an offset. They commit their offset to Kafka so they know where to resume after a restart.

### Broker

A Kafka server. A Kafka cluster has multiple brokers for fault tolerance. Each broker holds some partitions.

### ZooKeeper / KRaft

Older Kafka used ZooKeeper to manage cluster metadata (leader election, broker registration). Kafka 2.8+ ships **KRaft** mode — Kafka manages its own metadata without ZooKeeper, simplifying deployment.

### Producer

A client that publishes events to a topic.

### Consumer

A client that reads events from a topic. Consumers track their offset and commit it to Kafka.

### Consumer Group

A group of consumers that collectively consume a topic. Each partition is assigned to exactly one consumer in the group at a time — enabling parallel processing.

```
Topic: order-events (4 partitions)
Consumer Group: order-processor (3 instances)

Instance 1: reads Partition 0, Partition 1
Instance 2: reads Partition 2
Instance 3: reads Partition 3
```

If you have more consumers than partitions, some consumers are idle. Partitions are the unit of parallelism — max useful consumers = number of partitions.

### Replication

Each partition has one leader and N-1 followers (replicas on other brokers). Producers and consumers always talk to the leader. If the leader fails, a follower is elected as the new leader.

```
Partition 0:
  Leader: Broker 1
  Replica: Broker 2
  Replica: Broker 3
```

`replication.factor = 3` means 3 copies of each partition exist. You can lose up to 2 brokers and still read/write data.

## Kafka Guarantees

### Message Ordering

Within a partition: guaranteed. Across partitions: not guaranteed.

**Design implication:** if you need all events for a user to be processed in order, use the user's ID as the message key — all events for that user go to the same partition.

### Delivery Semantics

**At-most-once:** producer fires and forgets (no ack). Fastest, but messages can be lost.

**At-least-once (default):** producer waits for leader ack (`acks=1`) or all replicas ack (`acks=all`). On timeout, retries. Consumer commits offset after processing. If consumer crashes before committing, it reprocesses — duplicates possible.

**Exactly-once (Kafka EOS):** using idempotent producers + transactional APIs. Each message is produced and consumed exactly once. Requires `enable.idempotence=true` and transactional producers. Highest overhead.

## Spring Boot + Kafka (Java)

### Dependencies (pom.xml)

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

### Configuration (application.yml)

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all                     # wait for all replicas to ack
      retries: 3
      properties:
        enable.idempotence: true    # idempotent producer (no duplicate sends on retry)
    consumer:
      group-id: order-processor
      auto-offset-reset: earliest   # read from beginning if no committed offset
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      enable-auto-commit: false     # manual offset commit for reliability
```

### Producer

```java
@Service
public class OrderProducer {

    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void publishOrderEvent(Order order) {
        OrderEvent event = new OrderEvent(
                order.getId(),
                "ORDER_PLACED",
                order.getUserId(),
                order.getTotal()
        );

        // key = userId -> all events for same user go to same partition (ordered)
        kafkaTemplate.send("order-events", order.getUserId().toString(), event)
                .whenComplete((result, ex) -> {
                    if (ex != null) {
                        log.error("Failed to send event for order {}: {}", order.getId(), ex.getMessage());
                    } else {
                        log.info("Event sent: partition={}, offset={}",
                                result.getRecordMetadata().partition(),
                                result.getRecordMetadata().offset());
                    }
                });
    }
}
```

### Consumer

```java
@Service
public class OrderConsumer {

    @KafkaListener(
            topics = "order-events",
            groupId = "order-processor",
            concurrency = "3"   // 3 consumer threads, one per partition (if 3 partitions)
    )
    public void handleOrderEvent(
            ConsumerRecord<String, OrderEvent> record,
            Acknowledgment ack) {

        try {
            OrderEvent event = record.value();
            log.info("Processing event: partition={}, offset={}, orderId={}",
                    record.partition(), record.offset(), event.getOrderId());

            processOrder(event);

            ack.acknowledge();  // manual commit — only after successful processing

        } catch (Exception e) {
            log.error("Failed to process event at offset {}: {}", record.offset(), e.getMessage());
            // don't ack -> will be redelivered (at-least-once)
            // or send to DLT (Dead Letter Topic)
        }
    }
}
```

### Dead Letter Topic (DLT) with Spring Kafka

```java
@Bean
public DefaultErrorHandler errorHandler(KafkaTemplate<String, Object> template) {
    // After 3 retries, send to dead letter topic
    DeadLetterPublishingRecoverer recoverer =
            new DeadLetterPublishingRecoverer(template,
                    (r, e) -> new TopicPartition(r.topic() + ".DLT", r.partition()));

    FixedBackOff backOff = new FixedBackOff(1000L, 3L); // 3 retries, 1s apart

    return new DefaultErrorHandler(recoverer, backOff);
}
```

Failed messages go to `order-events.DLT` — a separate topic for investigation and replay.

## Node.js + Kafka (kafkajs)

```javascript
const { Kafka } = require('kafkajs');

const kafka = new Kafka({
    clientId: 'order-service',
    brokers: ['localhost:9092']
});

// Producer
const producer = kafka.producer({ idempotent: true });

async function publishOrderEvent(order) {
    await producer.connect();
    await producer.send({
        topic: 'order-events',
        messages: [{
            key: String(order.userId),  // partition by userId for ordering
            value: JSON.stringify({
                eventType: 'ORDER_PLACED',
                orderId: order.id,
                total: order.total
            })
        }]
    });
}

// Consumer
const consumer = kafka.consumer({ groupId: 'order-processor' });

async function startConsumer() {
    await consumer.connect();
    await consumer.subscribe({ topic: 'order-events', fromBeginning: false });

    await consumer.run({
        autoCommit: false,  // manual commit
        eachMessage: async ({ topic, partition, message, heartbeat }) => {
            try {
                const event = JSON.parse(message.value.toString());
                await processOrder(event);

                // commit only after successful processing
                await consumer.commitOffsets([{
                    topic,
                    partition,
                    offset: String(Number(message.offset) + 1)
                }]);
            } catch (err) {
                console.error('Processing failed:', err);
                // don't commit -> message will be redelivered
            }
        }
    });
}
```

## Important Kafka Configuration Parameters

### Producer

| Parameter | Effect |
|---|---|
| `acks=0` | No ack — fastest, messages can be lost |
| `acks=1` | Leader ack — fast, messages lost if leader fails before replication |
| `acks=all` (`-1`) | All in-sync replicas ack — safest, slower |
| `enable.idempotence=true` | Prevents duplicate sends on retry |
| `linger.ms` | Batching delay — wait N ms to batch more messages before sending (increases throughput, adds small latency) |
| `batch.size` | Max bytes per batch — larger batches = higher throughput |
| `compression.type` | `gzip`, `snappy`, `lz4` — reduce network/disk I/O |

### Consumer

| Parameter | Effect |
|---|---|
| `auto.offset.reset=earliest` | Start from beginning if no offset found |
| `auto.offset.reset=latest` | Start from now (miss historical messages) |
| `enable.auto.commit=false` | Manual offset commit — recommended for reliability |
| `max.poll.records` | Max messages per poll — tune for batch processing |
| `session.timeout.ms` | If consumer doesn't poll in this time, it's considered dead and partitions reassigned |

## Kafka vs RabbitMQ (Quick Reference)

| | Kafka | RabbitMQ |
|---|---|---|
| Retention | Configurable (days/weeks) | Deleted after ACK |
| Replay | Yes | No (need DLQ) |
| Throughput | Millions/sec | Tens of thousands/sec |
| Ordering | Per partition | Per queue |
| Consumer groups | Multiple independent groups | Competing consumers |
| Routing | Simple (topic/key) | Rich (exchanges, bindings) |
| Best for | Event streaming, audit logs, high throughput | Task queues, RPC, complex routing |

## Production Architecture

```
Order Service (Producer)
  |
  | (order-events topic, 6 partitions, replication=3)
  |
Kafka Cluster (3 brokers)
  |
  +----> Consumer Group: email-service (3 instances, each reads 2 partitions)
  +----> Consumer Group: inventory-service (3 instances)
  +----> Consumer Group: analytics-service (1 instance, reads all 6 partitions)
  +----> Consumer Group: fraud-detection (3 instances)
```

## Interview Questions

**Q1: What is a Kafka partition and why does it matter?**

A partition is an ordered, immutable log within a topic. Partitions enable parallelism — each consumer in a group reads a different partition simultaneously. The number of partitions determines the maximum parallelism for a consumer group (max useful consumers = number of partitions). Events with the same key always go to the same partition, guaranteeing order per key.

**Q2: What is a consumer group and how does it enable parallel processing?**

A consumer group is a set of consumers that together consume a topic. Kafka assigns each partition to exactly one consumer in the group. If a consumer group has 3 consumers and the topic has 6 partitions, each consumer reads 2 partitions in parallel. Multiple independent consumer groups can each consume the entire topic independently (each has its own offset tracking).

**Q3: How does Kafka guarantee message ordering?**

Within a partition, messages are strictly ordered by offset. If you need all events for a particular entity (e.g. a user) to be processed in order, use that entity's ID as the message key — Kafka routes all messages with the same key to the same partition, guaranteeing in-order processing for that entity.

**Q4: What is the difference between Kafka and a traditional message queue like RabbitMQ?**

Kafka retains messages for a configurable period (unlike RabbitMQ which deletes after ACK), supports multiple independent consumer groups each reading the full topic, and handles millions of messages/sec. RabbitMQ excels at complex routing (exchange types), RPC patterns, and lower-latency task processing at smaller scale. Kafka is for event streaming and audit logs; RabbitMQ is for task queues and traditional messaging.

**Q5: What does `enable.auto.commit=false` mean and why use it?**

By default, Kafka consumers automatically commit their offset at regular intervals regardless of processing success. With `auto.commit=false`, you only commit the offset after successfully processing the message. This ensures that if the consumer crashes mid-processing, the message is redelivered rather than silently skipped — enabling at-least-once delivery.

**Q6: What is a Dead Letter Topic and how does it differ from a DLQ in RabbitMQ?**

A Dead Letter Topic (DLT) is a regular Kafka topic where messages that failed processing after all retries are sent. Because Kafka retains messages on the DLT like any other topic, you can replay them later, analyze them, or route them to an alert. In RabbitMQ, a DLQ is a queue — same concept, different storage model.
