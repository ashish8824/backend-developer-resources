# Module 12: Pub/Sub (Publish/Subscribe)

**Level:** Intermediate

Pub/Sub is Redis's built-in messaging system — a completely different use case from caching. Instead of storing data, it's about **broadcasting real-time messages** between different parts of your system.

---

## 1. The Core Idea

**Pub/Sub** stands for **Publish/Subscribe**. It works like a radio station:

- A **publisher** broadcasts a message on a named **channel**.
- Any number of **subscribers** listening to that channel instantly receive the message.
- The publisher doesn't know or care who's listening — it just broadcasts.

```
                     ┌──────────────┐
   Publisher ───────▶│   Channel:    │───────▶ Subscriber A
   (sends msg)        │   "chat-room" │───────▶ Subscriber B
                     └──────────────┘───────▶ Subscriber C
```

This is exactly like a radio: the station (publisher) broadcasts on a frequency (channel), and anyone tuned in (subscriber) hears it. If nobody's tuned in, the message just... disappears. There's no "replay" — Pub/Sub messages are **not stored anywhere**.

---

## 2. Trying It in `redis-cli` (Two Terminal Windows)

**Terminal 1 — Subscribe to a channel:**

```bash
redis-cli
127.0.0.1:6379> SUBSCRIBE notifications
Reading messages... (press Ctrl-C to quit)
```

This terminal is now "listening" and will hang here, waiting for messages.

**Terminal 2 — Publish a message:**

```bash
redis-cli
127.0.0.1:6379> PUBLISH notifications "Hello, subscribers!"
(integer) 1   # this means: 1 subscriber received the message
```

Back in **Terminal 1**, you'll instantly see:

```
1) "message"
2) "notifications"
3) "Hello, subscribers!"
```

---

## 3. Pub/Sub in Node.js

**Important:** Once a Redis client calls `.subscribe()`, that connection is dedicated to *listening* — it can't be used for normal commands like `GET`/`SET` anymore. This is why you should create a **separate Redis connection** specifically for subscribing.

```js
// publisher.js
const Redis = require("ioredis");
const redis = new Redis(); // normal connection, used for publishing + other commands

async function sendNotification(message) {
  // PUBLISH returns the number of subscribers that received the message
  const receivedBy = await redis.publish("notifications", message);
  console.log(`Message delivered to ${receivedBy} subscriber(s)`);
}

sendNotification("New order placed! 🎉");
```

```js
// subscriber.js
const Redis = require("ioredis");

// A DEDICATED connection just for subscribing — never reuse this one for normal commands
const subscriber = new Redis();

// Tell Redis we want to listen to the "notifications" channel
subscriber.subscribe("notifications", (err, count) => {
  if (err) {
    console.error("Failed to subscribe:", err);
  } else {
    console.log(`Subscribed successfully! Listening on ${count} channel(s).`);
  }
});

// This event fires every time ANY subscribed channel receives a message
subscriber.on("message", (channel, message) => {
  console.log(`📩 Received on [${channel}]: ${message}`);
});
```

Run `subscriber.js` in one terminal, then run `publisher.js` in another — you'll see the message appear live.

---

## 4. Real-World Use Case: Real-Time Chat Notifications

Imagine a chat app running on **multiple server instances** (common in production, for scalability). If User A (connected to Server 1) sends a message to User B (connected to Server 2), Server 1 has no direct way to reach Server 2's WebSocket connection. Redis Pub/Sub solves this perfectly:

```
User A ──▶ Server 1 ──▶ PUBLISH "chat:room:42" message
                              │
                     Redis broadcasts to ALL subscribed servers
                              │
Server 2 (SUBSCRIBED) ◀──────┘
    │
    ▼
Sends message down User B's WebSocket connection
```

```js
// Simplified example combining Socket.IO + Redis Pub/Sub across servers
const io = require("socket.io")(server);
const redis = require("./config/redisClient");
const subscriber = redis.duplicate(); // create a second connection just for subscribing

subscriber.subscribe("chat:room:42");

subscriber.on("message", (channel, message) => {
  // Whenever ANY server publishes to this channel, broadcast it to
  // all locally-connected WebSocket clients in that chat room
  io.to("room:42").emit("newMessage", JSON.parse(message));
});

io.on("connection", (socket) => {
  socket.on("sendMessage", async (data) => {
    // Instead of directly emitting only to THIS server's connected clients,
    // publish to Redis so EVERY server instance hears about it
    await redis.publish("chat:room:42", JSON.stringify(data));
  });
});
```

This is exactly how many real-time apps scale chat/notifications across multiple server instances without needing sticky sessions or complex coordination.

---

## 5. Pattern Subscriptions (`PSUBSCRIBE`)

You can subscribe to multiple channels at once using wildcard patterns:

```bash
# Subscribe to ALL channels starting with "chat:room:"
PSUBSCRIBE chat:room:*
```

```js
subscriber.psubscribe("chat:room:*");

subscriber.on("pmessage", (pattern, channel, message) => {
  console.log(`Pattern [${pattern}] matched channel [${channel}]: ${message}`);
});
```

Useful when you have many dynamically-named channels (e.g., one per chat room) and want to listen to all of them without subscribing individually.

---

## 6. Important Limitation: Pub/Sub is "Fire and Forget"

If a subscriber isn't connected **at the exact moment** a message is published, that subscriber **will never see that message** — there's no history, no replay, no storage.

```
Publisher sends message at 10:00:00
Subscriber connects at    10:00:01
                                   → subscriber missed it completely!
```

**If you need guaranteed delivery / message history / replay**, Pub/Sub is the wrong tool — you want:

- **Redis Streams** (an append-only log, similar to Kafka) for guaranteed, replayable event delivery.
- **A proper queue** (Module 15, using Lists or a library like BullMQ) if you need "process this job exactly once, even if a worker is temporarily offline."

**Rule of thumb:** Use Pub/Sub for *live, ephemeral, best-effort* notifications (chat messages while online, live dashboards, "someone is typing" indicators). Use Streams or Queues when *delivery must be guaranteed*.

---

## 7. Interview Question: "What's the difference between Redis Pub/Sub and a message queue?"

**Answer:** Pub/Sub is a broadcast mechanism — a message is delivered live to whoever happens to be subscribed at that exact moment, and is lost forever if nobody's listening; there's no persistence or acknowledgment. A message queue (e.g., built with Redis Lists, or a dedicated system like BullMQ/RabbitMQ) persists messages/jobs until a worker explicitly processes them, supports retries, and guarantees each job is eventually handled — even if all workers are temporarily offline when the job was created.

---

## 8. Summary

- Pub/Sub lets you broadcast real-time messages to any number of live subscribers on a named channel.
- Messages are **not stored** — if nobody's listening at that instant, the message is lost.
- Always use a **separate, dedicated Redis connection** for subscribing.
- Great for real-time features (chat, notifications, live updates) especially across multiple server instances.
- Not suitable when you need guaranteed, replayable delivery — use Streams or a proper queue for that.

---

## 9. Homework

1. Run the `publisher.js`/`subscriber.js` example and watch a message flow between two terminals.
2. Try publishing a message *before* starting your subscriber — confirm it's never received.
3. Explain, in your own words, why Pub/Sub alone would be a poor choice for a "process this payment" background job system.

---

**Next up:** [Module 13 — Rate Limiting](./13-rate-limiting.md)
