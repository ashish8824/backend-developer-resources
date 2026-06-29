# Event-Driven Architecture (EDA)

## What is Event-Driven Architecture?

Event-Driven Architecture (EDA) is a software design pattern where services communicate by producing and consuming events rather than making direct API calls to each other. An event represents something that happened — a fact in the past.

```
Traditional (Request-Driven):
Order Service -> HTTP POST -> Payment Service (waits for response)
             -> HTTP POST -> Inventory Service (waits for response)
             -> HTTP POST -> Notification Service (waits for response)
Total time: sum of all calls. All services must be available.

Event-Driven:
Order Service -> publishes "ORDER_PLACED" event -> Event Bus
              -> returns response immediately

Payment Service    <- subscribes to "ORDER_PLACED" -> processes independently
Inventory Service  <- subscribes to "ORDER_PLACED" -> processes independently
Notification Service <- subscribes to "ORDER_PLACED" -> processes independently
```

## Why Event-Driven Architecture?

### 1. Loose Coupling

The Order Service doesn't know Payment, Inventory, or Notification Services exist. It just publishes an event. New services can subscribe without modifying Order Service.

```
New requirement: add a loyalty points service
Traditional: modify OrderService to call LoyaltyService
EDA: LoyaltyService subscribes to ORDER_PLACED — OrderService unchanged
```

### 2. Scalability

Each consumer scales independently based on its own load.

```
ORDER_PLACED events: 1,000/sec
Notification Service: slow (sends email) -> needs 10 instances to keep up
Payment Service: fast -> 2 instances sufficient
Each scales separately
```

### 3. Resilience

If a consumer fails, events queue up and are processed when it recovers. The producer is unaffected.

```
Email Service goes down:
  ORDER_PLACED events accumulate in Kafka/RabbitMQ
  Email Service restarts after 5 minutes
  Processes all backlogged events in order
  Zero events lost, zero impact on Order Service
```

### 4. Async Processing

Long-running operations (sending email, generating PDF, updating analytics) don't block the user's request.

```
User places order -> immediate response "Order confirmed"
                 -> Email sent asynchronously (user doesn't wait for this)
```

### 5. Audit Trail

Every event is a record of what happened. Store all events and you have a complete history.

## Core Concepts

### Event

An immutable fact representing something that happened. Always in past tense.

```json
{
  "eventId": "evt-789",
  "eventType": "ORDER_PLACED",
  "version": "1.0",
  "timestamp": "2025-01-15T10:30:00Z",
  "source": "order-service",
  "data": {
    "orderId": "ORD-123",
    "userId": "USR-456",
    "items": [
      { "productId": "PROD-789", "quantity": 2, "price": 49.99 }
    ],
    "total": 99.98
  }
}
```

Key properties:
- **Immutable:** events never change once published
- **Past tense:** `ORDER_PLACED` not `PLACE_ORDER`
- **Self-contained:** enough data for consumers to act without extra API calls
- **Has version:** schema can evolve; consumers handle multiple versions

### Event Producer

The service that detects a state change and publishes an event.

### Event Consumer

A service that subscribes to specific event types and reacts to them.

### Event Broker / Bus

The middleware that receives events from producers and delivers them to consumers. Examples: Kafka, RabbitMQ, AWS SNS/SQS, Google Pub/Sub, Azure Service Bus.

### Event Stream vs Event Queue

| | Event Stream (Kafka) | Event Queue (RabbitMQ) |
|---|---|---|
| Message retention | Retained (days/weeks) — replay possible | Deleted after consumption |
| Multiple consumers | Each consumer group reads independently | Competing consumers share messages |
| Ordering | Per partition | Per queue |
| Best for | Event sourcing, audit log, high throughput | Task distribution, RPC |

## Event-Driven Patterns

### 1. Event Notification

Producer notifies consumers that something happened. Consumers fetch details if needed.

```json
{ "eventType": "USER_REGISTERED", "userId": "USR-123" }
```

Consumers call back to User Service to get full user details. Lightweight events, but consumers need extra API calls.

### 2. Event-Carried State Transfer

Event contains all the data consumers need — no callback required.

```json
{
  "eventType": "ORDER_PLACED",
  "orderId": "ORD-123",
  "userId": "USR-456",
  "userEmail": "alice@example.com",  // email included so Notification Service doesn't need to fetch it
  "items": [...],
  "total": 99.98
}
```

Heavier events but fully decoupled — consumers don't need to call back.

### 3. Event Sourcing

Instead of storing current state, store the sequence of events that led to it. Current state is derived by replaying events.

```
Traditional (state):
orders table: { id: 123, status: SHIPPED, total: 99.98 }

Event Sourcing (event log):
event 1: ORDER_PLACED    { orderId: 123, total: 99.98 }
event 2: PAYMENT_RECEIVED { orderId: 123, amount: 99.98 }
event 3: ORDER_SHIPPED   { orderId: 123, trackingNo: "TRK-001" }

Current state = replay all events for order 123
```

Benefits:
- Complete audit trail
- Time travel — reconstruct state at any past point
- Easy to add new projections (read models) from historical events
- Debug by replaying events

Drawbacks:
- More complex querying (can't just `SELECT * FROM orders WHERE status = 'SHIPPED'`)
- Need to manage snapshots for performance (don't replay 1M events every time)
- Schema evolution is hard (old events with old schemas must still be playable)

### 4. CQRS (Command Query Responsibility Segregation)

Separate the write model (Commands) from the read model (Queries). Often used with Event Sourcing.

```
Command Side (writes):
User sends PlaceOrderCommand
  -> Order aggregate validates and applies
  -> Saves events: ORDER_PLACED, PAYMENT_REQUESTED
  -> Events published to Kafka

Query Side (reads):
Event consumers update denormalized read models (projections):
  OrderSummaryProjection: { orderId, status, total, userEmail }
  UserOrdersProjection:   { userId, orders: [...] }

Read API queries these projections directly — fast, simple SQL
```

Benefits:
- Read side optimized for queries (denormalized, indexed perfectly)
- Write side optimized for domain logic
- Scale read and write sides independently

## Java Implementation (Spring Boot + Kafka)

### Event Definition

```java
// Base event class
@Data
public abstract class DomainEvent {
    private String eventId = UUID.randomUUID().toString();
    private String eventType;
    private String version = "1.0";
    private LocalDateTime timestamp = LocalDateTime.now();
    private String source;
}

// Specific event
@Data
@EqualsAndHashCode(callSuper = true)
public class OrderPlacedEvent extends DomainEvent {
    private String orderId;
    private String userId;
    private String userEmail;
    private List<OrderItem> items;
    private BigDecimal total;

    public OrderPlacedEvent() {
        setEventType("ORDER_PLACED");
        setSource("order-service");
    }
}
```

### Producer (Order Service)

```java
@Service
public class OrderService {

    @Autowired
    private OrderRepository orderRepo;

    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;

    @Transactional
    public Order placeOrder(PlaceOrderRequest request) {
        // 1. Business logic
        Order order = new Order();
        order.setUserId(request.getUserId());
        order.setItems(request.getItems());
        order.setTotal(calculateTotal(request.getItems()));
        order.setStatus(OrderStatus.PLACED);

        orderRepo.save(order);

        // 2. Publish event (use Transactional Outbox in production)
        OrderPlacedEvent event = new OrderPlacedEvent();
        event.setOrderId(order.getId().toString());
        event.setUserId(request.getUserId());
        event.setUserEmail(request.getUserEmail());
        event.setItems(request.getItems());
        event.setTotal(order.getTotal());

        kafkaTemplate.send("order-events", order.getId().toString(), event);

        return order;
    }
}
```

### Consumers (Independent Services)

```java
// Payment Service consumer
@Service
public class PaymentEventConsumer {

    @KafkaListener(topics = "order-events", groupId = "payment-service")
    public void handleOrderPlaced(ConsumerRecord<String, OrderPlacedEvent> record,
                                  Acknowledgment ack) {
        if (!"ORDER_PLACED".equals(record.value().getEventType())) {
            ack.acknowledge();
            return;  // only handle ORDER_PLACED events
        }

        OrderPlacedEvent event = record.value();
        try {
            paymentService.initiatePayment(event.getOrderId(), event.getTotal());
            ack.acknowledge();
        } catch (Exception e) {
            log.error("Payment processing failed for order {}", event.getOrderId(), e);
            // don't ack -> retry or DLT
        }
    }
}

// Notification Service consumer — completely separate, different group-id
@Service
public class NotificationEventConsumer {

    @KafkaListener(topics = "order-events", groupId = "notification-service")
    public void handleOrderPlaced(ConsumerRecord<String, OrderPlacedEvent> record,
                                  Acknowledgment ack) {
        OrderPlacedEvent event = record.value();
        if (!"ORDER_PLACED".equals(event.getEventType())) {
            ack.acknowledge();
            return;
        }

        emailService.sendOrderConfirmation(event.getUserEmail(), event.getOrderId());
        ack.acknowledge();
    }
}
```

Both consumers receive every ORDER_PLACED event independently — different `groupId` means each gets its own copy.

## Node.js Implementation

```javascript
const { Kafka } = require('kafkajs');

const kafka = new Kafka({ brokers: ['localhost:9092'] });

// Producer
async function placeOrder(orderData) {
    const order = await saveOrderToDb(orderData);

    const producer = kafka.producer();
    await producer.connect();
    await producer.send({
        topic: 'order-events',
        messages: [{
            key: order.id,
            value: JSON.stringify({
                eventId: uuidv4(),
                eventType: 'ORDER_PLACED',
                timestamp: new Date().toISOString(),
                data: {
                    orderId: order.id,
                    userId: order.userId,
                    userEmail: order.userEmail,
                    total: order.total
                }
            })
        }]
    });

    return order;
}

// Consumer (Notification Service)
async function startNotificationConsumer() {
    const consumer = kafka.consumer({ groupId: 'notification-service' });
    await consumer.connect();
    await consumer.subscribe({ topic: 'order-events' });

    await consumer.run({
        eachMessage: async ({ message }) => {
            const event = JSON.parse(message.value.toString());

            if (event.eventType === 'ORDER_PLACED') {
                await sendOrderConfirmationEmail(event.data.userEmail, event.data.orderId);
            }
        }
    });
}
```

## Saga Pattern (Distributed Transactions via Events)

When a business process spans multiple services (e.g. order → payment → inventory → shipping), each step is a local transaction that publishes an event. Failure triggers compensating events.

### Choreography Saga (Event-Driven)

Services react to each other's events — no central coordinator.

```
Order Service   -> ORDER_PLACED event
Payment Service -> handles ORDER_PLACED -> PAYMENT_COMPLETED event
                                        -> (or PAYMENT_FAILED event)
Inventory Service -> handles PAYMENT_COMPLETED -> INVENTORY_RESERVED event
Shipping Service  -> handles INVENTORY_RESERVED -> SHIPMENT_CREATED event

Failure flow:
Payment fails -> PAYMENT_FAILED event
Order Service handles PAYMENT_FAILED -> cancels order -> ORDER_CANCELLED event
Inventory Service doesn't reserve (never got PAYMENT_COMPLETED)
```

**Pros:** simple, no coordinator bottleneck.
**Cons:** hard to track the overall state of a saga, complex failure scenarios.

### Orchestration Saga

A central "saga orchestrator" tells each service what to do and handles failures.

```
Saga Orchestrator:
  Step 1: command PaymentService to charge
  Step 2: command InventoryService to reserve
  Step 3: command ShippingService to create shipment

  If Step 2 fails:
    compensate: command PaymentService to refund (Step 1 undo)
    saga ends as FAILED
```

**Pros:** clear saga state, easier to reason about failures, centralized monitoring.
**Cons:** orchestrator can become a bottleneck; it knows too much about all services.

## Interview Questions

**Q1: What is the difference between event-driven and request-driven architecture?**

In request-driven (synchronous), Service A calls Service B directly and waits for a response — they're tightly coupled and both must be available simultaneously. In event-driven (async), Service A publishes an event; Service B consumes it independently — they're decoupled in time and space. New consumers can be added without modifying the producer.

**Q2: What is event sourcing?**

Instead of storing only the current state, store every event that led to that state. Current state is derived by replaying events. Provides a complete audit trail, time-travel debugging, and easy new read-model creation from historical events. Trade-off: complex queries require projections, and replaying millions of events requires snapshots for performance.

**Q3: What is CQRS?**

Command Query Responsibility Segregation separates the write model (commands modify state, emit events) from the read model (queries read from denormalized projections updated by event consumers). Read side is optimized for queries; write side for domain logic. Often paired with event sourcing — events from the write side update read-side projections.

**Q4: What is the Saga pattern and why is it needed?**

When a business transaction spans multiple services (order → payment → inventory), traditional ACID transactions don't work across service boundaries. The Saga pattern breaks it into local transactions, each publishing events. If one step fails, compensating transactions (refund, unreserve) undo previous steps. Choreography has services react to each other's events; Orchestration has a central coordinator directing each step.

**Q5: What is the Transactional Outbox pattern and why does it matter in EDA?**

Without the Outbox pattern, you risk: saving to DB succeeds but publishing the event fails → downstream services never know the order was placed. The Outbox pattern saves the event to the DB in the same transaction as the business data, then a separate poller reads unpublished events and sends them to the broker. Guarantees at-least-once event delivery without distributed transactions.
