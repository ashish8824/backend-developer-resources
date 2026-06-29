# Day 12 — Microservices & Message Brokers
### NestJS Interview Prep | 3-Year Backend Engineer Level

> **Goal for today:** Master microservices architecture in NestJS — transport layers, message patterns, RabbitMQ, Kafka, client proxy, and hybrid apps. A 3-year backend engineer must understand WHEN to use microservices, HOW to communicate between them, and HOW to handle failures. This is one of the most asked topics in senior-level interviews.

---

## 🧭 First-Timer Guidance — Read This First

**What are Microservices? Why do they exist?**

```
Monolith (everything in one app):
  ┌─────────────────────────────────┐
  │  Users │ Orders │ Products │ Email│
  │  Auth  │ Payment│ Inventory│ SMS  │
  └─────────────────────────────────┘
  One codebase, one deployment, one database

  Problems at scale:
  - Deploy a small email change → redeploy EVERYTHING
  - Email service crashes → ENTIRE app is down
  - Products need 10 servers, Email needs 1 → can't scale independently
  - One team's bug breaks another team's feature

Microservices (split into separate apps):
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │  Users   │  │  Orders  │  │ Products │
  │  Service │  │  Service │  │  Service │
  └────┬─────┘  └────┬─────┘  └────┬─────┘
       │              │              │
  ┌────┴─────┐  ┌─────┴────┐  ┌─────┴────┐
  │  Email   │  │ Payment  │  │ Inventory│
  │  Service │  │  Service │  │  Service │
  └──────────┘  └──────────┘  └──────────┘

  Benefits:
  - Deploy services independently (change email → redeploy only email)
  - Failure isolation (email crashes → orders still work)
  - Scale independently (products needs 10 instances, email needs 1)
  - Teams own services independently
```

**Two communication styles between microservices:**

```
1. SYNCHRONOUS (Request-Response):
   Service A → "Process this payment" → Service B → "Done, here's receipt"
   Service A WAITS for response
   Like a function call — blocks until answer comes back
   Transport: TCP, HTTP/REST, gRPC

2. ASYNCHRONOUS (Message-Based / Event-Driven):
   Service A → "Order was placed" → Message Broker
   Service B, C, D pick up the message independently
   Service A does NOT wait — fire and forget
   Transport: RabbitMQ, Kafka, Redis Pub/Sub
   
   Benefits of async:
   - Service A continues working even if B/C/D are down
   - Messages queue up and are processed when service recovers
   - Easy to add new consumers without changing producer
```

**Setup — install packages:**
```bash
# Core microservices
npm install @nestjs/microservices

# RabbitMQ
npm install amqplib amqp-connection-manager
npm install @types/amqplib --save-dev

# Kafka
npm install kafkajs

# Redis transport
npm install ioredis
```

---

## Table of Contents

1. [Microservice Architecture — Core Concepts](#1-microservice-architecture--core-concepts)
2. [NestJS Transport Layers — Overview](#2-nestjs-transport-layers--overview)
3. [TCP Transport — Simple Synchronous](#3-tcp-transport--simple-synchronous)
4. [Redis Transport — Pub/Sub Pattern](#4-redis-transport--pubsub-pattern)
5. [RabbitMQ Transport — Production Messaging](#5-rabbitmq-transport--production-messaging)
6. [Kafka Transport — High Throughput Events](#6-kafka-transport--high-throughput-events)
7. [Message Patterns — Request-Response vs Events](#7-message-patterns--request-response-vs-events)
8. [Client Proxy — Calling Microservices](#8-client-proxy--calling-microservices)
9. [Error Handling in Microservices](#9-error-handling-in-microservices)
10. [Hybrid Apps — HTTP + Microservice](#10-hybrid-apps--http--microservice)
11. [Event-Driven Architecture Patterns](#11-event-driven-architecture-patterns)
12. [Interview Questions & Answers](#12-interview-questions--answers)

---

## 1. Microservice Architecture — Core Concepts

### The two roles in NestJS microservices

```
Every service can play TWO roles simultaneously:

PRODUCER (sends messages):
  - The HTTP API Gateway receives a REST request
  - It calls the Orders microservice: "Create this order"
  - It calls the Email microservice: "Send confirmation email"

CONSUMER (receives and handles messages):
  - The Orders microservice LISTENS for "createOrder" messages
  - The Email microservice LISTENS for "sendEmail" messages

Most services are BOTH:
  Orders service receives "createOrder" from API Gateway (CONSUMER)
  Orders service sends "orderCreated" to Email service (PRODUCER)
```

### The Message Broker's role

```
Without broker (direct connection):
  Service A → TCP → Service B
  Problem: A must know B's address
  Problem: If B is down, A fails immediately

With broker (indirect):
  Service A → RabbitMQ → Service B
  A just knows the broker address (stable)
  If B is down: message queues in RabbitMQ, delivered when B recovers
  If traffic spikes: messages queue up, B processes at its pace
  Multiple instances of B: broker load-balances messages between them

Message Broker responsibilities:
  - Store messages until consumers are ready
  - Route messages to correct consumers
  - Guarantee delivery (at-least-once, exactly-once)
  - Load balance across multiple consumers
  - Dead letter queues for failed messages
  - Replay events (Kafka only)
```

### When to use microservices vs monolith

```
Use Monolith when:
  ✓ Small team (< 10 developers)
  ✓ Simple domain with few business boundaries
  ✓ Early startup — move fast, unclear requirements
  ✓ Simple deployment requirements
  ✗ Microservices add complexity without benefit at this stage

Use Microservices when:
  ✓ Large team — multiple teams need independent deployments
  ✓ Different scaling requirements per service
  ✓ Need fault isolation (one failure shouldn't kill everything)
  ✓ Services have different technology requirements
  ✓ Well-understood domain boundaries
  ✗ Don't start here — start monolith, extract services when needed
```

---

## 2. NestJS Transport Layers — Overview

```
NestJS @nestjs/microservices supports these transports:

Transport    Use Case                         Protocol
─────────────────────────────────────────────────────────
TCP          Simple internal sync calls       Custom binary
Redis        Simple pub/sub, caching          Redis protocol
RabbitMQ     Enterprise messaging, queues     AMQP 0-9-1
Kafka        High-throughput event streaming  Kafka protocol
gRPC         Typed RPC, streaming             HTTP/2 + Protobuf
MQTT         IoT, lightweight pub/sub         MQTT
NATS         Cloud-native messaging           NATS protocol

Most common in NestJS projects:
1. RabbitMQ — best for most microservice use cases
2. Kafka     — high throughput event streaming, event sourcing
3. TCP       — simple internal communication (same network)
4. Redis     — simple pub/sub, not for production messaging
```

---

## 3. TCP Transport — Simple Synchronous

### Setting up a TCP microservice

```typescript
// ── MICROSERVICE APP (consumer) ────────────────────────────────────────────
// src/main.ts of the Orders microservice

import { NestFactory } from '@nestjs/core';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';
import { AppModule } from './app.module';

async function bootstrap() {
  // Create a microservice (NOT an HTTP app)
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.TCP,
      options: {
        host: '0.0.0.0',   // Listen on all network interfaces
        port: 3001,        // Port for TCP connections
        // Different from HTTP port (3000)
      },
    },
  );

  await app.listen();
  // Note: .listen() not .listen(port) — port is in transport options
  console.log('Orders microservice listening on port 3001');
}
bootstrap();
```

### Message Controller (Consumer side)

```typescript
// src/orders/orders.controller.ts — IN THE MICROSERVICE

import {
  Controller,
  Logger,
} from '@nestjs/common';
import {
  MessagePattern,
  EventPattern,
  Payload,
  Ctx,
  TcpContext,
} from '@nestjs/microservices';

@Controller()
// Note: no route path — microservice controllers don't have HTTP routes
// They respond to MESSAGE PATTERNS instead
export class OrdersController {
  private readonly logger = new Logger(OrdersController.name);

  constructor(private readonly ordersService: OrdersService) {}

  // ── @MessagePattern — Request-Response pattern ─────────────────────────
  @MessagePattern({ cmd: 'create_order' })
  // Matches when someone sends a message with pattern { cmd: 'create_order' }
  // This is a REQUEST-RESPONSE: caller waits for this method to return
  async createOrder(
    @Payload() data: CreateOrderDto,
    // @Payload(): extracts the message data (like @Body() for HTTP)
  ): Promise<Order> {
    this.logger.log(`Creating order for user ${data.userId}`);

    const order = await this.ordersService.create(data);
    return order;
    // The returned value is sent BACK to the caller as the response
  }

  @MessagePattern({ cmd: 'get_order' })
  async getOrder(@Payload() data: { orderId: number }): Promise<Order> {
    return this.ordersService.findOne(data.orderId);
  }

  @MessagePattern({ cmd: 'get_user_orders' })
  async getUserOrders(
    @Payload() data: { userId: number; page: number; limit: number },
  ): Promise<PaginatedResult<Order>> {
    return this.ordersService.findByUser(data);
  }

  // ── @EventPattern — Fire-and-Forget pattern ────────────────────────────
  @EventPattern('order_paid')
  // Matches when someone emits an EVENT with name 'order_paid'
  // No response — caller doesn't wait
  // Fire and forget: producer emits, consumer handles, no ack to producer
  async handleOrderPaid(
    @Payload() data: { orderId: number; amount: number },
  ): Promise<void> {
    this.logger.log(`Processing payment for order ${data.orderId}`);
    await this.ordersService.markAsPaid(data.orderId, data.amount);
    // No return value — void — no response sent back
  }
}
```

---

## 4. Redis Transport — Pub/Sub Pattern

```typescript
// Redis transport uses Redis pub/sub for message delivery
// Simpler than RabbitMQ but less reliable (no persistence, no queuing)
// Messages lost if no consumer is running

// ── MICROSERVICE with Redis transport ─────────────────────────────────────

// src/main.ts (email-service)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(
  AppModule,
  {
    transport: Transport.REDIS,
    options: {
      host: process.env.REDIS_HOST || 'localhost',
      port: parseInt(process.env.REDIS_PORT || '6379'),
      password: process.env.REDIS_PASSWORD,
      db: 0,                          // Redis database index
      retryAttempts: 5,               // Reconnection attempts
      retryDelay: 1000,               // Delay between retries (ms)
    },
  },
);

await app.listen();

// ── Controller (consumer) ──────────────────────────────────────────────────
@Controller()
export class EmailController {

  @MessagePattern('send_welcome_email')
  // Redis transport: pattern is typically a string (not object)
  async sendWelcome(@Payload() data: { email: string; name: string }) {
    await this.emailService.sendWelcome(data.email, data.name);
    return { success: true };
  }

  @EventPattern('user_registered')
  // EventPattern for fire-and-forget via Redis pub/sub
  async onUserRegistered(@Payload() data: { userId: number; email: string }) {
    await this.emailService.sendWelcome(data.email, 'New User');
  }
}
```

---

## 5. RabbitMQ Transport — Production Messaging

### Why RabbitMQ over simple TCP/Redis?

```
TCP Transport problems:
  - Synchronous: caller waits for response
  - If service is down: request fails immediately
  - No message persistence
  - No load balancing across instances

RabbitMQ advantages:
  ✓ Messages persist if consumer is down (queue stores them)
  ✓ Dead Letter Queues (DLQ): failed messages go to separate queue for inspection
  ✓ Load balancing: multiple consumers compete for messages
  ✓ Acknowledgements: message only removed after successful processing
  ✓ Routing: flexible exchange → queue routing rules
  ✓ Priority queues, delayed messages, TTL
```

### RabbitMQ key concepts

```
Exchange:
  Receives messages from producers
  Routes them to queues based on ROUTING RULES

Queue:
  Stores messages until consumers process them
  Durable: survives RabbitMQ restart
  Exclusive: only one consumer at a time

Binding:
  Connection between exchange and queue
  Defines WHICH messages go to WHICH queue

Exchange Types:
  direct:  Route by exact routing key ("order.created" → orders queue)
  topic:   Route by pattern ("order.*" → all order events)
  fanout:  Broadcast to ALL bound queues (no routing key)
  headers: Route by message headers

ACK (Acknowledgement):
  Consumer tells RabbitMQ "I processed this message successfully"
  Only then does RabbitMQ remove it from queue
  If consumer crashes before ACK: message re-queued automatically
```

### Setting up RabbitMQ microservice

```typescript
// ── MICROSERVICE (consumer) ────────────────────────────────────────────────
// src/main.ts of notifications-service

import { NestFactory } from '@nestjs/core';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.RMQ,  // RMQ = RabbitMQ
      options: {
        urls: [process.env.RABBITMQ_URL || 'amqp://localhost:5672'],
        // amqp://username:password@host:port/vhost
        // amqp://guest:guest@localhost:5672/    ← default local

        queue: 'notifications_queue',
        // The queue this microservice CONSUMES from
        // NestJS auto-creates this queue if it doesn't exist

        queueOptions: {
          durable: true,
          // durable: true → queue survives RabbitMQ restart
          // Messages persisted to disk
          // ALWAYS true in production

          // arguments: {
          //   'x-dead-letter-exchange': 'dlx',
          //   'x-dead-letter-routing-key': 'notifications.dead',
          //   // Failed messages go to dead letter exchange
          //   'x-message-ttl': 86400000, // Messages expire after 24 hours
          // },
        },

        noAck: false,
        // noAck: false → MANUAL acknowledgement mode (RECOMMENDED)
        // NestJS ACKs message only after handler returns successfully
        // If handler throws → message is NACKED and re-queued
        // noAck: true → auto-ack (dangerous: message removed even if processing fails)

        prefetchCount: 10,
        // How many unacknowledged messages this consumer can hold at once
        // Prevents one slow consumer from blocking all messages
        // Set based on your processing capacity

        isGlobalPrefetchCount: false,
        // false: prefetch per consumer channel
        // true: prefetch per entire connection
      },
    },
  );

  await app.listen();
  console.log('Notifications microservice listening on RabbitMQ');
}
bootstrap();
```

### RabbitMQ Controller (consumer)

```typescript
// src/notifications/notifications.controller.ts

import { Controller, Logger } from '@nestjs/common';
import {
  MessagePattern,
  EventPattern,
  Payload,
  Ctx,
  RmqContext,  // RabbitMQ-specific context
} from '@nestjs/microservices';

@Controller()
export class NotificationsController {
  private readonly logger = new Logger(NotificationsController.name);

  // ── Message Pattern (Request-Response) ────────────────────────────────
  @MessagePattern('send_notification')
  async sendNotification(
    @Payload() data: SendNotificationDto,
    @Ctx() context: RmqContext,
    // @Ctx(): injects the transport context
    // RmqContext: gives access to RabbitMQ-specific message properties
  ): Promise<{ success: boolean }> {
    const channel = context.getChannelRef();
    // channel: the AMQP channel object
    const originalMsg = context.getMessage();
    // originalMsg: the raw AMQP message (has headers, properties, etc.)

    try {
      await this.notificationService.send(data);

      // Manual acknowledgement: tell RabbitMQ "I processed this successfully"
      channel.ack(originalMsg);
      // Message REMOVED from queue — won't be redelivered

      return { success: true };

    } catch (error) {
      this.logger.error(`Failed to send notification: ${error.message}`);

      const shouldRequeue = this.shouldRequeue(error);
      // Decide: retry this message? or send to dead letter queue?

      channel.nack(
        originalMsg,
        false,       // allUpTo: false → only nack this message, not all
        shouldRequeue, // requeue: true → put back in queue for retry
                       //          false → send to dead letter queue
      );

      throw error;
      // Throwing propagates the error back to the producer (if sync pattern)
    }
  }

  private shouldRequeue(error: any): boolean {
    // Don't retry on business logic errors (e.g., invalid data)
    if (error.name === 'ValidationError') return false;
    // Don't retry if we've tried too many times
    // (check x-death header for retry count)
    // Retry on transient errors (network, timeout)
    return true;
  }

  // ── Event Pattern (Fire-and-Forget) ──────────────────────────────────
  @EventPattern('user.registered')
  // Dot notation is common for event names: domain.action
  async onUserRegistered(
    @Payload() data: { userId: number; email: string; name: string },
    @Ctx() context: RmqContext,
  ): Promise<void> {
    const channel = context.getChannelRef();
    const originalMsg = context.getMessage();

    try {
      await this.notificationService.sendWelcomeEmail(data);
      channel.ack(originalMsg);

    } catch (error) {
      this.logger.error(`Failed to send welcome email: ${error.message}`);
      channel.nack(originalMsg, false, false);
      // Don't requeue — send to dead letter queue for manual inspection
    }
  }

  @EventPattern('order.shipped')
  async onOrderShipped(
    @Payload() data: { orderId: number; userId: number; trackingNumber: string },
    @Ctx() context: RmqContext,
  ): Promise<void> {
    const channel = context.getChannelRef();
    const msg = context.getMessage();

    try {
      await this.notificationService.sendShippingNotification(data);
      channel.ack(msg);

    } catch (error) {
      channel.nack(msg, false, false);
    }
  }
}
```

### RabbitMQ with multiple queues (fanout/topic exchanges)

```typescript
// For event broadcasting to MULTIPLE services
// Use RabbitMQ exchanges instead of direct queue routing

// In a separate exchange setup service or RabbitMQ management:
// Exchange: 'events' (type: topic)
// Routing patterns:
//   order.*    → orders_queue (Orders service)
//   user.*     → users_queue (Users service)
//   order.*    → analytics_queue (Analytics service) — same pattern, different queue

// Multiple consumers can bind to the same exchange with different routing keys
// This is the power of RabbitMQ over simple TCP

// Service A publishes to exchange 'events' with routing key 'order.placed'
// Service B (Orders) receives via 'order.placed' pattern
// Service C (Analytics) receives via 'order.*' pattern (wildcard)
// Service D (Email) receives via 'order.placed' (exact match)
```

---

## 6. Kafka Transport — High Throughput Events

### Why Kafka over RabbitMQ?

```
RabbitMQ:             Kafka:
Message Queue         Event Log / Stream
Delete after ACK      Retain messages for days/weeks/forever
One consumer gets msg Multiple consumer GROUPS each get ALL messages
Up to ~50k msg/sec    Millions of messages per second
Good for work queues  Good for event sourcing, analytics, audit logs
Hard to replay        Easy to replay from any point in history

Choose Kafka when:
  ✓ Need to replay events (event sourcing)
  ✓ Multiple independent consumers of same events
  ✓ Very high throughput (millions/sec)
  ✓ Long message retention for audit/analytics
  ✓ Building an event-driven data pipeline

Choose RabbitMQ when:
  ✓ Complex routing rules
  ✓ Priority queues
  ✓ Simple work queue (one consumer processes each message)
  ✓ Team is familiar with AMQP concepts
  ✓ Easier to set up and manage
```

### Kafka key concepts

```
Topic:      Category of messages (like a table in a database)
            'orders', 'payments', 'user-events'

Partition:  A topic is split into partitions for parallelism
            Partition 0: [msg1, msg4, msg7]
            Partition 1: [msg2, msg5, msg8]
            Partition 2: [msg3, msg6, msg9]
            Messages within a partition are ORDERED
            Messages across partitions are NOT ordered

Consumer Group:
            Group of consumers that together consume a topic
            Each partition consumed by ONE consumer in the group
            Adding consumers = more parallelism (up to # partitions)
            
            Group A (Orders service): all consume 'orders' topic
            Group B (Analytics): also consume 'orders' topic independently
            → Both get ALL messages (unlike RabbitMQ where only one gets each)

Offset:     Position in a partition — "I've read up to message 1500"
            Consumers commit their offset
            On restart: resume from committed offset (no message loss)
```

### Kafka microservice setup

```typescript
// ── Kafka Consumer (microservice) ─────────────────────────────────────────
// src/main.ts of analytics-service

const app = await NestFactory.createMicroservice<MicroserviceOptions>(
  AppModule,
  {
    transport: Transport.KAFKA,
    options: {
      client: {
        clientId: 'analytics-service',
        // Unique ID for this Kafka client

        brokers: ['kafka1:9092', 'kafka2:9092', 'kafka3:9092'],
        // Kafka broker addresses (cluster)
        // In development: ['localhost:9092']

        ssl: process.env.NODE_ENV === 'production',
        // SSL for production Kafka clusters

        sasl: process.env.NODE_ENV === 'production' ? {
          mechanism: 'plain',
          username: process.env.KAFKA_USERNAME,
          password: process.env.KAFKA_PASSWORD,
        } : undefined,
        // Authentication for production clusters

        retry: {
          retries: 8,           // Number of retries
          initialRetryTime: 300, // Initial retry delay (ms)
          maxRetryTime: 30000,   // Max retry delay (ms)
          factor: 0.2,           // Randomization factor
        },
      },

      consumer: {
        groupId: 'analytics-consumer-group',
        // Consumer group ID — all instances with same groupId
        // share the topic partitions (load balancing)
        // Different services MUST have different groupIds to each get all messages

        allowAutoTopicCreation: true,
        // Auto-create topics if they don't exist
        // false in production — create topics with proper config

        sessionTimeout: 30000,
        // Time after which consumer is considered dead if no heartbeat

        heartbeatInterval: 3000,
        // How often to send heartbeat to Kafka (must be < sessionTimeout/3)
      },
    },
  },
);

await app.listen();
```

### Kafka Controller (consumer)

```typescript
// src/analytics/analytics.controller.ts

import {
  MessagePattern,
  EventPattern,
  Payload,
  Ctx,
  KafkaContext,  // Kafka-specific context
} from '@nestjs/microservices';

@Controller()
export class AnalyticsController {

  // ── Kafka event handler ────────────────────────────────────────────────
  @EventPattern('order.placed')
  // Subscribes to 'order.placed' Kafka topic
  async handleOrderPlaced(
    @Payload() message: KafkaMessage<OrderPlacedEvent>,
    // KafkaMessage wraps the Kafka message with metadata
    @Ctx() context: KafkaContext,
  ): Promise<void> {
    const { value, offset, partition, timestamp } = message;
    // value: the actual message payload
    // offset: position in partition (for exactly-once processing)
    // partition: which partition this came from
    // timestamp: when message was produced

    const originalMessage = context.getMessage();
    // Raw Kafka message with all metadata

    const heartbeat = context.getHeartbeat();
    // heartbeat(): call this periodically for long-running handlers
    // Prevents Kafka from thinking consumer is dead during processing

    try {
      // Long-running analytics processing
      await heartbeat(); // Tell Kafka we're still alive

      await this.analyticsService.processOrderEvent(value);

      await heartbeat(); // Call again after processing
      // Kafka commits offset automatically after handler returns

    } catch (error) {
      // With Kafka, throwing here will:
      // - NOT commit the offset
      // - Kafka will redeliver the message to another consumer
      this.logger.error(`Failed to process event at offset ${offset}: ${error.message}`);
      throw error; // Redeliver for retry
    }
  }

  // ── Kafka consumer with custom deserializer ────────────────────────────
  @EventPattern('user.activity')
  async trackUserActivity(
    @Payload() data: UserActivityEvent,
  ): Promise<void> {
    // Kafka delivers messages in batches for efficiency
    // NestJS handles each message individually

    await this.analyticsService.trackActivity({
      userId: data.userId,
      action: data.action,
      metadata: data.metadata,
      timestamp: new Date(data.timestamp),
    });
  }
}
```

---

## 7. Message Patterns — Request-Response vs Events

### @MessagePattern vs @EventPattern — deep comparison

```typescript
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// @MessagePattern — Request-Response
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

// CONSUMER side:
@MessagePattern({ cmd: 'find_user' })
async findUser(@Payload() data: { userId: number }): Promise<User> {
  return this.usersService.findOne(data.userId);
  // Return value is sent BACK to the caller
}

// PRODUCER side (covered in section 8):
const user = await this.usersClient
  .send({ cmd: 'find_user' }, { userId: 1 })
  .toPromise();
// Caller WAITS (blocks) until response arrives
// Default timeout: 30 seconds — configure in ClientProxy options

// When to use @MessagePattern:
// ✓ You need data back from the other service
// ✓ Sequential operations (create order → get confirmation)
// ✓ Internal API calls between services
// ✗ Avoid for events that many services need
// ✗ Creates tight coupling (caller depends on callee)

// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// @EventPattern — Publish-Subscribe (Fire and Forget)
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

// CONSUMER side:
@EventPattern('order.placed')
async onOrderPlaced(@Payload() data: OrderPlacedEvent): Promise<void> {
  await this.inventoryService.reserveItems(data.items);
  // No return value — void
}

// PRODUCER side:
this.ordersClient.emit('order.placed', {
  orderId: order.id,
  userId: order.userId,
  items: order.items,
  total: order.total,
});
// Does NOT wait — continues immediately
// Returns void

// MULTIPLE CONSUMERS of the same event:
// Analytics service: @EventPattern('order.placed') → track revenue
// Email service:     @EventPattern('order.placed') → send confirmation
// Inventory service: @EventPattern('order.placed') → reserve stock
// All receive the same event independently

// When to use @EventPattern:
// ✓ Notifying multiple services about something that happened
// ✓ Decoupled, async workflows
// ✓ You don't need a response
// ✓ Building event-driven architecture
```

---

## 8. Client Proxy — Calling Microservices

### Setting up ClientProxy

```typescript
// src/orders/orders.module.ts — HTTP service that calls microservices

import { Module } from '@nestjs/common';
import {
  ClientsModule,
  Transport,
  ClientProxy,
} from '@nestjs/microservices';

@Module({
  imports: [
    ClientsModule.registerAsync([
      // ── Register Users microservice client ────────────────────────────
      {
        name: 'USERS_SERVICE',
        // Injection token — use this to inject the client
        inject: [ConfigService],
        useFactory: (config: ConfigService): ClientProvider => ({
          transport: Transport.RMQ,
          options: {
            urls: [config.getOrThrow('RABBITMQ_URL')],
            queue: 'users_queue',
            // Must match the queue name in the Users microservice
            queueOptions: { durable: true },
          },
        }),
      },

      // ── Register Orders microservice client ───────────────────────────
      {
        name: 'ORDERS_SERVICE',
        inject: [ConfigService],
        useFactory: (config: ConfigService): ClientProvider => ({
          transport: Transport.RMQ,
          options: {
            urls: [config.getOrThrow('RABBITMQ_URL')],
            queue: 'orders_queue',
            queueOptions: { durable: true },
          },
        }),
      },

      // ── Register Kafka client ─────────────────────────────────────────
      {
        name: 'KAFKA_SERVICE',
        inject: [ConfigService],
        useFactory: (config: ConfigService): ClientProvider => ({
          transport: Transport.KAFKA,
          options: {
            client: {
              clientId: 'api-gateway',
              brokers: [config.getOrThrow('KAFKA_BROKER')],
            },
            producer: {
              allowAutoTopicCreation: true,
            },
          },
        }),
      },
    ]),
  ],
  controllers: [OrdersController],
  providers: [OrdersService],
})
export class OrdersModule {}
```

### Using ClientProxy in services

```typescript
// src/orders/orders.service.ts — calling other microservices

import { Injectable, Inject, Logger, NotFoundException } from '@nestjs/common';
import { ClientProxy } from '@nestjs/microservices';
import { firstValueFrom, timeout, catchError, throwError } from 'rxjs';

@Injectable()
export class OrdersService {
  private readonly logger = new Logger(OrdersService.name);

  constructor(
    @Inject('USERS_SERVICE')
    private readonly usersClient: ClientProxy,
    // Inject by the NAME registered in ClientsModule

    @Inject('ORDERS_SERVICE')
    private readonly ordersClient: ClientProxy,

    @Inject('KAFKA_SERVICE')
    private readonly kafkaClient: ClientProxy,

    @InjectRepository(Order)
    private readonly orderRepository: Repository<Order>,
  ) {}

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // SEND — Request-Response (returns Observable)
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  async createOrder(dto: CreateOrderDto): Promise<Order> {
    // Step 1: Validate user exists (synchronous call to Users service)
    const user = await firstValueFrom(
      this.usersClient
        .send<User>({ cmd: 'find_user' }, { userId: dto.userId })
        // .send(): Request-Response pattern
        // Returns Observable<User>
        // firstValueFrom: converts Observable → Promise
        .pipe(
          timeout(5000),
          // Throw TimeoutError if no response in 5 seconds
          // Prevents hanging forever if Users service is down

          catchError((error) => {
            if (error.name === 'TimeoutError') {
              return throwError(() => new ServiceUnavailableException(
                'Users service is unavailable. Please try again.',
              ));
            }
            if (error.message?.includes('User not found')) {
              return throwError(() => new NotFoundException(
                `User ${dto.userId} not found`,
              ));
            }
            return throwError(() => error);
          }),
        ),
    );

    this.logger.log(`Creating order for user ${user.email}`);

    // Step 2: Check inventory (synchronous call)
    const inventoryStatus = await firstValueFrom(
      this.ordersClient
        .send<InventoryCheckResult>(
          { cmd: 'check_inventory' },
          { items: dto.items },
        )
        .pipe(timeout(3000)),
    );

    if (!inventoryStatus.available) {
      throw new BadRequestException(
        `Items not available: ${inventoryStatus.unavailableItems.join(', ')}`,
      );
    }

    // Step 3: Create the order in database
    const order = await this.orderRepository.save(
      this.orderRepository.create({
        ...dto,
        status: OrderStatus.PENDING,
      }),
    );

    // Step 4: Publish event — fire and forget (async notifications)
    this.usersClient.emit('order.placed', {
      orderId: order.id,
      userId: order.userId,
      items: order.items,
      total: order.total,
      timestamp: new Date().toISOString(),
    });
    // .emit(): Fire-and-forget
    // Does NOT return a meaningful Observable (returns void Observable)
    // Don't await — continue immediately

    // Step 5: Also publish to Kafka for analytics and audit
    this.kafkaClient.emit('order.placed', {
      key: order.id.toString(),  // Kafka message key (for partition routing)
      value: {
        orderId: order.id,
        userId: order.userId,
        amount: order.total,
        timestamp: new Date().toISOString(),
      },
    });

    return order;
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // Parallel microservice calls
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  async getDashboardData(userId: number) {
    // Call multiple services IN PARALLEL
    const [user, orders, balance] = await Promise.all([
      firstValueFrom(
        this.usersClient
          .send({ cmd: 'find_user' }, { userId })
          .pipe(timeout(3000)),
      ),
      firstValueFrom(
        this.ordersClient
          .send({ cmd: 'get_user_orders' }, { userId, limit: 5 })
          .pipe(timeout(3000)),
      ),
      firstValueFrom(
        this.usersClient
          .send({ cmd: 'get_balance' }, { userId })
          .pipe(timeout(3000)),
      ),
    ]);
    // All three requests fire simultaneously — total time = slowest response
    // Much faster than sequential await

    return { user, recentOrders: orders, balance };
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // Retry pattern
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  async callWithRetry<T>(
    observable: Observable<T>,
    maxRetries = 3,
  ): Promise<T> {
    import { retry, delay } from 'rxjs/operators';

    return firstValueFrom(
      observable.pipe(
        retry({
          count: maxRetries,
          delay: (error, retryCount) => {
            // Exponential backoff: 1s, 2s, 4s
            const delayMs = Math.pow(2, retryCount - 1) * 1000;
            this.logger.warn(
              `Retry ${retryCount}/${maxRetries} after ${delayMs}ms: ${error.message}`,
            );
            return timer(delayMs);
          },
        }),
      ),
    );
  }
}
```

---

## 9. Error Handling in Microservices

### RPC Exception handling

```typescript
// src/users/users.controller.ts (microservice consumer)

import { RpcException } from '@nestjs/microservices';

@MessagePattern({ cmd: 'find_user' })
async findUser(@Payload() data: { userId: number }): Promise<User> {
  const user = await this.usersService.findOne(data.userId);

  if (!user) {
    // Use RpcException instead of HttpException in microservices
    // RpcException is serialized and sent back to the caller
    throw new RpcException({
      statusCode: 404,
      message: `User ${data.userId} not found`,
      error: 'Not Found',
    });
    // The calling service receives this as an error
    // They can catch and handle it
  }

  return user;
}

// On the caller (producer) side:
try {
  const user = await firstValueFrom(
    this.usersClient
      .send({ cmd: 'find_user' }, { userId: 999 })
      .pipe(
        catchError((error) => {
          // error.error: the RpcException object thrown by the consumer
          // { statusCode: 404, message: 'User 999 not found', error: 'Not Found' }
          const status = error.error?.statusCode || 500;
          const message = error.error?.message || 'Service error';

          // Convert RpcException to HTTP exception for the API response
          throw new HttpException(message, status);
          // Now the HTTP client gets a proper 404 response
        }),
      ),
  );
} catch (error) {
  // HttpException thrown above — will be caught by NestJS exception filters
  throw error;
}
```

### Exception filter for microservices

```typescript
// src/common/filters/rpc-exception.filter.ts

import {
  ArgumentsHost,
  Catch,
  ExceptionFilter,
  Logger,
} from '@nestjs/common';
import { RpcException } from '@nestjs/microservices';
import { Observable, throwError } from 'rxjs';

@Catch(RpcException)
// @Catch(RpcException) — catches RPC exceptions specifically
export class RpcExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(RpcExceptionFilter.name);

  catch(exception: RpcException, host: ArgumentsHost): Observable<any> {
    const error = exception.getError();
    this.logger.error('RPC Exception:', error);

    // Return the error as an Observable (required for microservice filters)
    return throwError(() => error);
  }
}

// Register in microservice main.ts:
// app.useGlobalFilters(new RpcExceptionFilter());
```

### Circuit Breaker Pattern

```typescript
// Pattern to prevent cascading failures
// If a service keeps failing, stop calling it temporarily

// Using opossum library:
// npm install opossum @types/opossum

import * as CircuitBreaker from 'opossum';

@Injectable()
export class CircuitBreakerService {
  private breakers = new Map<string, CircuitBreaker>();

  getBreaker(name: string, fn: Function): CircuitBreaker {
    if (!this.breakers.has(name)) {
      const breaker = new CircuitBreaker(fn, {
        timeout: 3000,
        // Consider failure if takes > 3 seconds

        errorThresholdPercentage: 50,
        // Open circuit if 50% of requests fail

        resetTimeout: 30000,
        // Try again after 30 seconds (half-open state)

        volumeThreshold: 5,
        // At least 5 requests before evaluating
      });

      breaker.on('open', () =>
        this.logger.warn(`Circuit OPEN: ${name} is unavailable`),
      );
      breaker.on('halfOpen', () =>
        this.logger.log(`Circuit HALF-OPEN: testing ${name}`),
      );
      breaker.on('close', () =>
        this.logger.log(`Circuit CLOSED: ${name} recovered`),
      );

      this.breakers.set(name, breaker);
    }

    return this.breakers.get(name)!;
  }

  async callService<T>(
    serviceName: string,
    fn: () => Promise<T>,
    fallback?: () => T,
  ): Promise<T> {
    const breaker = this.getBreaker(serviceName, fn);

    if (fallback) {
      breaker.fallback(fallback);
      // If circuit is open → return fallback instead of throwing
    }

    return breaker.fire() as Promise<T>;
  }
}

// Usage in service:
async getUserProfile(userId: number) {
  return this.circuitBreaker.callService(
    'users-service',
    () => firstValueFrom(
      this.usersClient.send({ cmd: 'find_user' }, { userId }).pipe(timeout(3000)),
    ),
    () => ({ id: userId, name: 'Unknown', email: '' }),
    // Fallback: return a default user if service is unavailable
  );
}
```

---

## 10. Hybrid Apps — HTTP + Microservice

### What is a Hybrid App?

A hybrid app serves BOTH HTTP requests (REST API) AND listens for microservice messages. The most common pattern is the **API Gateway** — a service that receives HTTP requests from clients and communicates with backend microservices.

```typescript
// src/main.ts — Hybrid App (HTTP + RabbitMQ consumer)

import { NestFactory } from '@nestjs/core';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';
import { AppModule } from './app.module';
import { ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  // Step 1: Create the HTTP application
  const app = await NestFactory.create(AppModule);
  // This creates a full HTTP server

  // Step 2: Connect a microservice transport
  app.connectMicroservice<MicroserviceOptions>({
    transport: Transport.RMQ,
    options: {
      urls: [process.env.RABBITMQ_URL],
      queue: 'api-gateway-queue',
      queueOptions: { durable: true },
    },
  });
  // Now this app ALSO listens on RabbitMQ

  // Step 3: Configure HTTP
  app.setGlobalPrefix('api/v1');
  app.useGlobalPipes(new ValidationPipe({ transform: true, whitelist: true }));

  // Step 4: Start BOTH
  await app.startAllMicroservices();
  // Start all connected microservices first

  await app.listen(3000);
  // Then start the HTTP server

  console.log('API Gateway: HTTP on :3000 + RabbitMQ consumer active');
}
bootstrap();

// Now the same app:
// - Handles REST API requests at http://localhost:3000/api/v1/*
// - Consumes RabbitMQ messages from 'api-gateway-queue'
// - Can call other microservices via ClientProxy
// - Any controller can have BOTH @Get() and @MessagePattern() handlers
```

### API Gateway pattern

```typescript
// src/app.module.ts — API Gateway

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    TypeOrmModule.forRootAsync({...}),

    // Register clients for ALL backend microservices
    ClientsModule.registerAsync([
      {
        name: 'USERS_SERVICE',
        inject: [ConfigService],
        useFactory: (config: ConfigService) => ({
          transport: Transport.RMQ,
          options: {
            urls: [config.getOrThrow('RABBITMQ_URL')],
            queue: 'users_queue',
            queueOptions: { durable: true },
          },
        }),
      },
      {
        name: 'ORDERS_SERVICE',
        inject: [ConfigService],
        useFactory: (config: ConfigService) => ({
          transport: Transport.RMQ,
          options: {
            urls: [config.getOrThrow('RABBITMQ_URL')],
            queue: 'orders_queue',
            queueOptions: { durable: true },
          },
        }),
      },
      {
        name: 'PRODUCTS_SERVICE',
        inject: [ConfigService],
        useFactory: (config: ConfigService) => ({
          transport: Transport.RMQ,
          options: {
            urls: [config.getOrThrow('RABBITMQ_URL')],
            queue: 'products_queue',
            queueOptions: { durable: true },
          },
        }),
      },
    ]),

    AuthModule,
    UsersModule,
    OrdersModule,
    ProductsModule,
  ],
})
export class AppModule {}

// ── API Gateway Controller ─────────────────────────────────────────────────
// Receives HTTP → calls microservice → returns response

@Controller('orders')
@UseGuards(JwtAuthGuard)
export class OrdersController {
  constructor(
    @Inject('ORDERS_SERVICE') private ordersClient: ClientProxy,
    @Inject('PRODUCTS_SERVICE') private productsClient: ClientProxy,
  ) {}

  @Post()
  async createOrder(
    @Body() dto: CreateOrderDto,
    @CurrentUser() user: User,
  ) {
    // HTTP request → forward to Orders microservice
    return firstValueFrom(
      this.ordersClient
        .send({ cmd: 'create_order' }, { ...dto, userId: user.id })
        .pipe(
          timeout(10000),
          catchError(err => throwError(() =>
            new HttpException(
              err.error?.message || 'Order creation failed',
              err.error?.statusCode || 500,
            ),
          )),
        ),
    );
  }

  @Get()
  async getMyOrders(
    @CurrentUser() user: User,
    @Query() query: PaginationDto,
  ) {
    return firstValueFrom(
      this.ordersClient
        .send({ cmd: 'get_user_orders' }, { userId: user.id, ...query })
        .pipe(timeout(5000)),
    );
  }

  // This same controller ALSO handles microservice events:
  @EventPattern('payment.completed')
  // When payment service emits this event, this controller receives it
  async onPaymentCompleted(
    @Payload() data: { orderId: number; paymentId: string },
  ) {
    // Forward to orders service to update status
    this.ordersClient.emit('order.update_status', {
      orderId: data.orderId,
      status: 'PAID',
    });
  }
}
```

---

## 11. Event-Driven Architecture Patterns

### Saga Pattern — distributed transactions

```typescript
// Problem: Multi-step operation across services
// "Place order" needs: reserve inventory → charge payment → update order status
// If payment fails: must undo inventory reservation (compensating transaction)

// Saga orchestration (one coordinator service):

@Injectable()
export class OrderSagaService {
  async placeOrder(dto: CreateOrderDto): Promise<Order> {
    const sagaId = uuid();
    // Unique ID to correlate all saga steps

    // Step 1: Reserve inventory
    const reservation = await firstValueFrom(
      this.inventoryClient
        .send('inventory.reserve', {
          sagaId,
          items: dto.items,
        })
        .pipe(timeout(5000)),
    );

    if (!reservation.success) {
      throw new BadRequestException('Items not available');
    }

    try {
      // Step 2: Process payment
      const payment = await firstValueFrom(
        this.paymentsClient
          .send('payment.process', {
            sagaId,
            userId: dto.userId,
            amount: dto.total,
            currency: dto.currency,
          })
          .pipe(timeout(10000)),
      );

      if (!payment.success) {
        // Payment failed → compensate: release inventory
        await firstValueFrom(
          this.inventoryClient.send('inventory.release', {
            sagaId,
            reservationId: reservation.reservationId,
          }),
        );
        throw new BadRequestException('Payment failed');
      }

      // Step 3: Create order record
      const order = await firstValueFrom(
        this.ordersClient.send('order.create', {
          sagaId,
          ...dto,
          paymentId: payment.paymentId,
          status: 'CONFIRMED',
        }),
      );

      // Step 4: Publish success event (for other services)
      this.eventBus.emit('order.placed', {
        sagaId,
        orderId: order.id,
        userId: dto.userId,
        items: dto.items,
        total: dto.total,
      });

      return order;

    } catch (error) {
      // Compensate: release inventory if we reserved it
      if (reservation?.reservationId) {
        await firstValueFrom(
          this.inventoryClient.send('inventory.release', {
            sagaId,
            reservationId: reservation.reservationId,
          }).pipe(timeout(5000), catchError(() => of(null))),
          // Don't throw if compensation also fails — log and alert
        );
      }

      throw error;
    }
  }
}
```

### Outbox Pattern — guaranteed event publishing

```typescript
// Problem: Service saves to DB + publishes event
// If app crashes BETWEEN save and publish → event is lost
// "At-least-once" delivery requires the outbox pattern

// Solution: Save event to DB in SAME transaction as business data
// A separate process reads the outbox and publishes

// src/orders/entities/outbox-event.entity.ts
@Entity('outbox_events')
export class OutboxEvent {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  aggregateId: string;     // e.g., order ID

  @Column()
  aggregateType: string;   // e.g., 'Order'

  @Column()
  eventType: string;       // e.g., 'order.placed'

  @Column({ type: 'jsonb' })
  payload: Record<string, any>;

  @Column({ default: false })
  processed: boolean;

  @Column({ nullable: true })
  processedAt: Date;

  @CreateDateColumn()
  createdAt: Date;
}

// In the order creation service:
async createOrder(dto: CreateOrderDto): Promise<Order> {
  return this.dataSource.transaction(async (manager) => {
    // Step 1: Create order
    const order = manager.create(Order, dto);
    await manager.save(order);

    // Step 2: Create outbox event IN SAME TRANSACTION
    const outboxEvent = manager.create(OutboxEvent, {
      aggregateId: order.id.toString(),
      aggregateType: 'Order',
      eventType: 'order.placed',
      payload: {
        orderId: order.id,
        userId: order.userId,
        items: order.items,
        total: order.total,
      },
    });
    await manager.save(outboxEvent);

    // Both saved atomically — if transaction fails, NEITHER is saved
    return order;
  });
}

// Separate outbox processor (runs every few seconds):
@Injectable()
export class OutboxProcessor {
  @Cron('*/5 * * * * *')  // Every 5 seconds
  async processOutbox(): Promise<void> {
    const events = await this.outboxRepository.find({
      where: { processed: false },
      order: { createdAt: 'ASC' },
      take: 100,
    });

    for (const event of events) {
      try {
        // Publish to message broker
        await firstValueFrom(
          this.eventBus.emit(event.eventType, event.payload),
        );

        // Mark as processed
        await this.outboxRepository.update(event.id, {
          processed: true,
          processedAt: new Date(),
        });

      } catch (error) {
        this.logger.error(
          `Failed to publish event ${event.id}: ${error.message}`,
        );
        // Will retry on next run
      }
    }
  }
}
```

---

## 12. Interview Questions & Answers

**Q1: What is the difference between synchronous and asynchronous communication in microservices?**

> "Synchronous communication (Request-Response) means the calling service sends a request and WAITS for a response before continuing — like a function call. In NestJS, this uses `clientProxy.send()` which returns an Observable that resolves when the response arrives. TCP, HTTP, and gRPC are synchronous transports. Asynchronous communication (Event-Driven) means the service publishes a message and continues immediately without waiting — the consumer processes it independently. This uses `clientProxy.emit()`. The key trade-offs: synchronous is simpler and gives immediate feedback but creates tight coupling and fails if the downstream service is unavailable. Async is more resilient (messages queue if consumer is down), supports multiple consumers of the same event, and scales better, but makes it harder to track the outcome of an operation."

---

**Q2: What is the difference between RabbitMQ and Kafka?**

> "They solve different problems despite both being message brokers. RabbitMQ is a traditional message queue — messages are delivered to one consumer and deleted after acknowledgement. It excels at complex routing rules (exchange types: direct, topic, fanout, headers), priority queues, dead letter queues, and work distribution where each task should be processed by exactly one worker. Kafka is a distributed event log — messages are retained for configurable periods (days or forever) and multiple consumer groups can each independently read ALL messages. Kafka is designed for very high throughput (millions of messages per second) and supports replaying events from any point in history, making it ideal for event sourcing and analytics pipelines. I'd choose RabbitMQ for most microservice communication (order processing, notifications) and Kafka when I need event replay, audit logs, or very high throughput analytics pipelines."

---

**Q3: How does the ClientProxy work? What is firstValueFrom()?**

> "ClientProxy is NestJS's abstraction for sending messages to a microservice. `client.send()` returns a cold Observable that hasn't started yet. The observable only executes (sends the message and waits for response) when you subscribe to it. `firstValueFrom()` from RxJS subscribes and converts the Observable to a Promise, resolving with the first emitted value. I use `firstValueFrom()` because async/await is cleaner than manual subscribe/unsubscribe. The key difference from `.toPromise()` (deprecated) is that `firstValueFrom()` throws if the Observable completes without emitting, which is the correct behavior for a request-response pattern. I always add `.pipe(timeout(5000))` before firstValueFrom to prevent hanging indefinitely if the microservice doesn't respond."

---

**Q4: What happens when a microservice is down? How do you handle failures?**

> "The behavior depends on the transport. With TCP/Redis, the call fails immediately with a connection error. With RabbitMQ and Kafka, messages queue up while the consumer is down and are delivered when it recovers — this is the key resilience advantage of message brokers. For the caller, I implement several patterns: First, timeout with `.pipe(timeout(5000))` to fail fast rather than hanging. Second, fallback logic in `catchError()` — return cached data, a default response, or re-throw an appropriate HTTP exception. Third, circuit breakers (using opossum) — if a service has 50% failure rate, stop calling it for 30 seconds to let it recover and prevent overwhelming a struggling service. For critical workflows, I use the saga pattern for distributed transactions with compensating transactions on failure."

---

**Q5: What is the difference between @MessagePattern and @EventPattern?**

> "`@MessagePattern` implements Request-Response — the consumer handles the message and RETURNS a value that's sent back to the producer. The producer awaits this response via `client.send()`. It's like an RPC call. Use it when you need data back: finding a user, validating an order, checking inventory. `@EventPattern` implements fire-and-forget — the consumer handles the event but sends NO response back. The producer doesn't wait, using `client.emit()`. Multiple consumers can independently handle the same event. Use it for side effects: send an email when an order is placed, update analytics when payment completes. The naming convention I use: MessagePattern uses command names like `{ cmd: 'create_order' }`, EventPattern uses domain events like `'order.placed'` — past tense because they describe something that already happened."

---

**Q6: What is the Outbox Pattern and why is it important?**

> "The outbox pattern solves the dual-write problem — when you need to both save data to a database AND publish an event to a message broker atomically. If you save to DB then publish: the app might crash between these two steps, leaving the event unpublished and downstream services unaware. If you publish then save: the event fires but the save might fail, causing inconsistent state. The outbox pattern works by saving the event to an `outbox_events` table in the SAME database transaction as your business data. If the transaction succeeds, both the data and the outbox record are saved atomically. A separate background process (or scheduled job) reads unpublished outbox records and publishes them to the message broker, marking them processed. This guarantees at-least-once delivery with no dual-write inconsistency. The trade-off is slight latency (a few seconds for the outbox processor to run) and added complexity."

---

**Q7: How does the API Gateway pattern work in NestJS microservices?**

> "The API Gateway is a hybrid NestJS app that acts as the entry point for all client requests. It serves HTTP (REST API) for external clients and communicates with backend microservices internally. In NestJS, you create it with `NestFactory.create()` for HTTP and then call `app.connectMicroservice()` to also consume messages, then `app.startAllMicroservices()` followed by `app.listen()`. The gateway uses `ClientsModule` to register clients for each backend service, injects them with `@Inject('SERVICE_NAME')`, and forwards requests using `client.send()` or `client.emit()`. This gives you a single external API surface while keeping internal services decoupled. Concerns handled at the gateway: authentication (JWT verification), rate limiting, request logging, SSL termination, and API versioning — so individual microservices don't each need to implement these."

---

## Quick Reference — Day 12 Cheat Sheet

```
Install:
  npm install @nestjs/microservices amqplib amqp-connection-manager kafkajs

Microservice app:
  NestFactory.createMicroservice(AppModule, { transport, options })
  await app.listen()  ← no port argument

Hybrid app (HTTP + microservice):
  app = NestFactory.create(AppModule)           ← HTTP
  app.connectMicroservice({ transport, options }) ← add MS transport
  await app.startAllMicroservices()             ← start MS consumers
  await app.listen(3000)                        ← start HTTP server

Consumer decorators:
  @MessagePattern({ cmd: 'name' })  → Request-Response (return value sent back)
  @EventPattern('event.name')       → Fire-and-Forget (void, no response)
  @Payload()                        → message data (like @Body())
  @Ctx()                            → transport context (RmqContext, KafkaContext)

Producer (ClientProxy):
  ClientsModule.registerAsync([{ name: 'TOKEN', useFactory: () => config }])
  @Inject('TOKEN') private client: ClientProxy
  client.send(pattern, data)   → Observable<Response> (Request-Response)
  client.emit(event, data)     → Observable<void> (Fire-and-Forget)
  firstValueFrom(observable.pipe(timeout(5000)))  → Promise

RabbitMQ config:
  transport: Transport.RMQ
  urls: ['amqp://user:pass@host:5672']
  queue: 'queue_name'
  queueOptions: { durable: true }
  noAck: false    ← manual ACK (recommended)
  prefetchCount: 10

RabbitMQ context:
  channel.ack(originalMsg)           ← success: remove from queue
  channel.nack(msg, false, true)     ← fail + requeue
  channel.nack(msg, false, false)    ← fail + dead letter queue

Kafka config:
  transport: Transport.KAFKA
  client: { clientId, brokers: ['host:9092'] }
  consumer: { groupId: 'my-group' }
  ← Same groupId = load balanced (one consumer gets each message)
  ← Different groupId = each gets ALL messages

Error handling:
  throw new RpcException({ statusCode, message, error })  ← in consumer
  catchError(err => throwError(() => new HttpException(...))) ← in producer
  .pipe(timeout(5000))  ← always add timeout to send()

Patterns:
  Saga: coordinator orchestrates multi-step distributed transaction
        + compensating transactions on failure
  Outbox: save event to DB in same transaction, processor publishes later
          → guarantees at-least-once delivery, no dual-write problem
  Circuit Breaker: stop calling failing service temporarily
                   → open → half-open (test) → closed (recovered)

Transport comparison:
  TCP     → simple sync, same network, no persistence
  Redis   → simple pub/sub, no persistence, no queuing
  RabbitMQ → complex routing, persistence, DLQ, production messaging
  Kafka   → high throughput, event replay, multiple consumer groups
```

---

*Day 12 complete — Week 2 done! Tomorrow — Day 13: Testing — Unit tests with Jest, mocking providers, testing controllers and services, e2e tests with supertest, TestingModule, and code coverage.*
