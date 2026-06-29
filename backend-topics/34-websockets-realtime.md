# WebSockets & Real-Time Communication

## What is a WebSocket?

A WebSocket is a full-duplex, persistent communication channel between a client and server over a single TCP connection. Unlike HTTP (request-response), WebSockets allow both client AND server to send messages to each other at any time.

```
HTTP (Request-Response — one-way each time):
Client: GET /notifications  ──────────────────► Server
Server: 200 OK (latest data) ◄────────────────── Server
        (connection closed)

Client must poll every N seconds for updates — wasteful

WebSocket (Persistent — bidirectional):
Client: WS Handshake ─────────────────────────► Server
Server: 101 Switching Protocols ◄──────────────── Server
        (connection stays open)

Server ──────► "New order placed" ─────────────► Client (anytime!)
Client ──────► "Mark read" ────────────────────► Server (anytime!)
```

## HTTP Polling vs WebSocket vs SSE

| Method | Direction | Connection | Use Case |
|---|---|---|---|
| Short Polling | Client → Server (repeated) | New HTTP per poll | Simple status checks |
| Long Polling | Client → Server (holds open) | One HTTP, server delays response | Basic real-time |
| Server-Sent Events (SSE) | Server → Client only | Persistent HTTP | Live feeds, dashboards |
| WebSocket | Bidirectional | Persistent TCP | Chat, gaming, collaboration |

### Short Polling

```javascript
// Client polls every 3 seconds — very inefficient
setInterval(async () => {
    const resp = await fetch('/api/notifications');
    const data = await resp.json();
    updateUI(data);
}, 3000);
// Problem: 99% of requests return "no new data" — waste of resources
```

### Server-Sent Events (SSE)

One-way stream from server to client. Good for dashboards, live feeds, progress bars.

```java
// Spring Boot SSE
@GetMapping(value = "/api/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter streamEvents(Authentication auth) {
    SseEmitter emitter = new SseEmitter(Long.MAX_VALUE);
    String userId = auth.getName();

    sseService.addEmitter(userId, emitter);

    emitter.onCompletion(() -> sseService.removeEmitter(userId));
    emitter.onTimeout(() -> sseService.removeEmitter(userId));

    return emitter;
}

// Send event from anywhere in the app
public void sendToUser(String userId, Object data) {
    SseEmitter emitter = emitters.get(userId);
    if (emitter != null) {
        try {
            emitter.send(SseEmitter.event()
                    .name("notification")
                    .data(data, MediaType.APPLICATION_JSON));
        } catch (IOException e) {
            emitters.remove(userId);
        }
    }
}
```

```javascript
// Client — native EventSource API
const eventSource = new EventSource('/api/events', {
    headers: { 'Authorization': 'Bearer ' + token }
});

eventSource.addEventListener('notification', (event) => {
    const data = JSON.parse(event.data);
    showNotification(data);
});

eventSource.onerror = () => {
    // Browser auto-reconnects on connection loss
    console.log('SSE connection lost, reconnecting...');
};
```

## WebSocket — Full Duplex

### WebSocket Handshake

WebSocket starts as an HTTP request and upgrades to a WebSocket connection:

```
Client → Server:
GET /ws HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

Server → Client:
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

After the 101 response, the connection is a TCP socket — no more HTTP overhead.

## Spring Boot WebSocket (STOMP over WebSocket)

STOMP (Simple Text Oriented Messaging Protocol) is a messaging protocol that runs on top of WebSocket. Spring Boot has first-class support for it.

### Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

### WebSocket Configuration

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        // In-memory broker for topic and queue subscriptions
        config.enableSimpleBroker("/topic", "/queue");

        // Prefix for messages from client TO server
        config.setApplicationDestinationPrefixes("/app");

        // Prefix for user-specific messages
        config.setUserDestinationPrefix("/user");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")           // WebSocket endpoint
                .setAllowedOriginPatterns("*")
                .withSockJS();                // SockJS fallback for browsers without WS support
    }
}
```

### Controller — Handling Messages

```java
@Controller
public class ChatController {

    @Autowired
    private SimpMessagingTemplate messagingTemplate;

    // Handle message from client at /app/chat.send
    @MessageMapping("/chat.send")
    @SendTo("/topic/chatroom.general")  // broadcast to all subscribers of this topic
    public ChatMessage sendMessage(ChatMessage message, Principal principal) {
        message.setSender(principal.getName());
        message.setTimestamp(LocalDateTime.now());
        return message;
    }

    // Handle private message (user-to-user)
    @MessageMapping("/chat.private")
    public void sendPrivateMessage(PrivateMessage message, Principal principal) {
        message.setSender(principal.getName());

        // Send to specific user — /user/{username}/queue/private
        messagingTemplate.convertAndSendToUser(
                message.getRecipient(),
                "/queue/private",
                message
        );
    }

    // Join notification
    @MessageMapping("/chat.join")
    @SendTo("/topic/chatroom.general")
    public ChatMessage joinRoom(ChatMessage message, Principal principal) {
        message.setType("JOIN");
        message.setSender(principal.getName());
        return message;
    }
}
```

### Sending Messages from Services (Server Push)

```java
@Service
public class NotificationService {

    @Autowired
    private SimpMessagingTemplate messagingTemplate;

    // Broadcast to all subscribers of a topic
    public void broadcastOrderUpdate(String orderId, OrderStatus status) {
        OrderStatusUpdate update = new OrderStatusUpdate(orderId, status);
        messagingTemplate.convertAndSend("/topic/orders." + orderId, update);
    }

    // Send to specific user
    public void notifyUser(String userId, Notification notification) {
        messagingTemplate.convertAndSendToUser(
                userId,
                "/queue/notifications",
                notification
        );
    }

    // Broadcast to all connected users
    public void broadcastSystemAlert(String message) {
        messagingTemplate.convertAndSend("/topic/alerts",
                new SystemAlert(message, LocalDateTime.now()));
    }
}

// Trigger from anywhere — e.g. after order status changes
@Service
public class OrderService {

    @Autowired
    private NotificationService notificationService;

    public void updateOrderStatus(Long orderId, OrderStatus newStatus) {
        Order order = orderRepo.findById(orderId).orElseThrow();
        order.setStatus(newStatus);
        orderRepo.save(order);

        // Push real-time update to connected clients
        notificationService.broadcastOrderUpdate(orderId.toString(), newStatus);
        notificationService.notifyUser(order.getUserId().toString(),
                new Notification("Your order status changed to: " + newStatus));
    }
}
```

### WebSocket Security (JWT Authentication)

```java
@Component
public class WebSocketAuthInterceptor implements ChannelInterceptor {

    @Autowired
    private JwtService jwtService;

    @Override
    public Message<?> preSend(Message<?> message, MessageChannel channel) {
        StompHeaderAccessor accessor =
                MessageHeaderAccessor.getAccessor(message, StompHeaderAccessor.class);

        if (StompCommand.CONNECT.equals(accessor.getCommand())) {
            // Extract JWT from connection header
            String token = accessor.getFirstNativeHeader("Authorization");

            if (token != null && token.startsWith("Bearer ")) {
                token = token.substring(7);
                try {
                    String username = jwtService.extractUsername(token);
                    UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                    UsernamePasswordAuthenticationToken auth =
                            new UsernamePasswordAuthenticationToken(
                                    userDetails, null, userDetails.getAuthorities());

                    accessor.setUser(auth);  // sets Principal for the WS session
                } catch (JwtException e) {
                    throw new MessageDeliveryException("Invalid JWT token");
                }
            }
        }
        return message;
    }
}

// Register in WebSocket config
@Override
public void configureClientInboundChannel(ChannelRegistration registration) {
    registration.interceptors(webSocketAuthInterceptor);
}
```

## Node.js WebSocket (Socket.IO)

Socket.IO is the standard for WebSocket in Node.js — it handles reconnection, rooms, namespaces, and falls back to polling if WebSocket isn't available.

### Server Setup

```javascript
const express = require('express');
const http = require('http');
const { Server } = require('socket.io');
const jwt = require('jsonwebtoken');

const app = express();
const httpServer = http.createServer(app);

const io = new Server(httpServer, {
    cors: {
        origin: 'https://myapp.com',
        methods: ['GET', 'POST']
    }
});

// JWT Authentication middleware
io.use((socket, next) => {
    const token = socket.handshake.auth.token;
    if (!token) return next(new Error('Authentication required'));

    try {
        const decoded = jwt.verify(token, process.env.JWT_SECRET);
        socket.userId = decoded.sub;
        socket.userEmail = decoded.email;
        next();
    } catch (err) {
        next(new Error('Invalid token'));
    }
});

// Connection handler
io.on('connection', (socket) => {
    console.log(`User ${socket.userId} connected: ${socket.id}`);

    // Join user-specific room for private messages
    socket.join(`user:${socket.userId}`);

    // Join a chat room
    socket.on('join:room', (roomId) => {
        socket.join(`room:${roomId}`);
        io.to(`room:${roomId}`).emit('user:joined', {
            userId: socket.userId,
            roomId
        });
    });

    // Handle chat message
    socket.on('message:send', ({ roomId, content }) => {
        const message = {
            id: Date.now(),
            roomId,
            senderId: socket.userId,
            content,
            timestamp: new Date().toISOString()
        };

        // Broadcast to everyone in the room
        io.to(`room:${roomId}`).emit('message:new', message);

        // Persist to DB asynchronously
        messageRepo.save(message);
    });

    // Typing indicator
    socket.on('typing:start', ({ roomId }) => {
        socket.to(`room:${roomId}`).emit('typing:start', { userId: socket.userId });
    });

    socket.on('typing:stop', ({ roomId }) => {
        socket.to(`room:${roomId}`).emit('typing:stop', { userId: socket.userId });
    });

    // Disconnect
    socket.on('disconnect', (reason) => {
        console.log(`User ${socket.userId} disconnected: ${reason}`);
        // Notify room members
        socket.rooms.forEach(room => {
            if (room !== socket.id) {
                io.to(room).emit('user:left', { userId: socket.userId });
            }
        });
    });
});

// Send from server to specific user (from REST endpoint, job, etc.)
function notifyUser(userId, event, data) {
    io.to(`user:${userId}`).emit(event, data);
}

// Broadcast order update
function broadcastOrderUpdate(orderId, status) {
    io.to(`order:${orderId}`).emit('order:updated', { orderId, status });
}
```

### Client

```javascript
import { io } from 'socket.io-client';

const socket = io('https://api.myapp.com', {
    auth: { token: localStorage.getItem('token') },
    transports: ['websocket'],  // prefer WebSocket over polling
    reconnectionAttempts: 5,
    reconnectionDelay: 1000
});

socket.on('connect', () => {
    console.log('Connected:', socket.id);

    // Join order tracking room
    socket.emit('join:room', `order-${orderId}`);
});

socket.on('order:updated', ({ orderId, status }) => {
    updateOrderStatusUI(orderId, status);
    showToast(`Order ${orderId} is now ${status}`);
});

socket.on('message:new', (message) => {
    appendMessageToChat(message);
});

socket.on('typing:start', ({ userId }) => {
    showTypingIndicator(userId);
});

socket.on('disconnect', () => {
    showReconnectingIndicator();
});

socket.on('connect_error', (err) => {
    console.error('Connection failed:', err.message);
});
```

## Scaling WebSockets

### The Scaling Problem

```
Load Balancer
     |
App Server 1 (User A connected here)
App Server 2 (User B connected here)

User B sends message to User A:
  Message arrives at Server 2
  Server 2 tries to send to User A — but User A is on Server 1!
  Message lost!
```

### Solution: Redis Pub/Sub Adapter

All servers publish messages to Redis. All servers subscribe and forward to their connected clients.

```javascript
// Socket.IO with Redis Adapter
const { createAdapter } = require('@socket.io/redis-adapter');
const { createClient } = require('redis');

const pubClient = createClient({ url: 'redis://localhost:6379' });
const subClient = pubClient.duplicate();

await Promise.all([pubClient.connect(), subClient.connect()]);

io.adapter(createAdapter(pubClient, subClient));
```

```
User B → Server 2 → publishes to Redis → Server 1 subscribes → forwards to User A
```

```java
// Spring Boot: use Spring Session or a message broker (RabbitMQ/Redis)
// for multi-node WebSocket scaling

// In WebSocketConfig — use a real broker instead of simple in-memory broker
@Override
public void configureMessageBroker(MessageBrokerRegistry config) {
    config.enableStompBrokerRelay("/topic", "/queue")  // RabbitMQ or ActiveMQ
            .setRelayHost("localhost")
            .setRelayPort(61613);
    config.setApplicationDestinationPrefixes("/app");
}
```

## Connection Management Best Practices

### Heartbeat / Ping-Pong

```javascript
// Keep connection alive + detect stale connections
const io = new Server(httpServer, {
    pingTimeout: 20000,   // disconnect if no pong in 20s
    pingInterval: 10000   // send ping every 10s
});
```

### Reconnection Logic

```javascript
// Client-side reconnection (Socket.IO handles automatically)
const socket = io('https://api.myapp.com', {
    reconnection: true,
    reconnectionAttempts: Infinity,
    reconnectionDelay: 1000,        // 1s initial
    reconnectionDelayMax: 30000,    // max 30s between retries
    randomizationFactor: 0.5        // add jitter
});

// On reconnect, re-join rooms and resync state
socket.on('reconnect', () => {
    socket.emit('resync', { lastEventId: lastSeenEventId });
});
```

## Interview Questions

**Q1: What is the difference between HTTP long polling and WebSockets?**

Long polling: client sends a request, server holds it open until data is available (or timeout), returns data, client immediately sends another request — simulates real-time but with HTTP overhead on every cycle. WebSocket: a single persistent TCP connection is established; both sides can send messages at any time with minimal overhead (no headers per message). WebSockets are more efficient for high-frequency, low-latency bidirectional communication.

**Q2: When would you use SSE vs WebSocket?**

SSE for one-way server-to-client streaming (live dashboards, progress bars, news feeds) — simpler, works over standard HTTP, auto-reconnects, good browser support. WebSocket for bidirectional communication (chat, collaborative editing, live gaming, trading terminals) where the client also sends frequent messages. SSE is simpler and sufficient for most notification/feed use cases.

**Q3: How do you scale WebSockets across multiple servers?**

A client connects to one server; messages to other users may need to reach a different server. Solution: use a shared pub/sub layer. When Server A receives a message for User B (connected to Server B), Server A publishes to Redis. Server B subscribes and forwards to User B's socket. Socket.IO Redis Adapter handles this automatically; Spring Boot uses STOMP broker relay (RabbitMQ) for the same purpose.

**Q4: How do you authenticate WebSocket connections?**

During the STOMP CONNECT frame or Socket.IO handshake, pass a JWT in the headers. A server-side interceptor/middleware validates the token and sets the user context for the session. All subsequent messages from that socket are treated as authenticated with that user. Don't rely on cookies for WebSocket auth in APIs — use explicit token passing in connection headers.

**Q5: What is a Socket.IO Room and how does it work?**

A Room is a named group of sockets. A socket can join/leave rooms; messages can be broadcast to all sockets in a room. Rooms are used for chat rooms (`room:general`), per-user channels (`user:123`), resource subscriptions (`order:456`). `socket.join('room-id')` and `io.to('room-id').emit(...)` are the core operations. Rooms are server-side only — clients don't know who else is in a room.
