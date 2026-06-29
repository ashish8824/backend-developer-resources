# Day 11 — WebSockets & Real-time Communication
### NestJS Interview Prep | 3-Year Backend Engineer Level

> **Goal for today:** Master real-time communication in NestJS using WebSockets and Socket.io. Understand gateways, namespaces, rooms, broadcasting patterns, authentication in WebSockets, and when to use WebSockets vs other real-time approaches. A 3-year backend engineer must know how to build scalable real-time features like live chat, notifications, dashboards, and collaborative tools.

---

## 🧭 First-Timer Guidance — Read This First

**Why WebSockets? The problem with HTTP polling:**

```
HTTP (Request-Response) — how REST APIs work:
  Client: "Hey, any new messages?"   → Server: "No"
  Client: "Hey, any new messages?"   → Server: "No"
  Client: "Hey, any new messages?"   → Server: "Yes! Here's 1 message"
  
  Client must ASK every second → wasteful, slow, server overloaded
  This is called POLLING — terrible for real-time

WebSocket — persistent bidirectional connection:
  Client connects once → Server can PUSH at any time
  Server: "You have a new message!" (no asking needed)
  Server: "User Alice is typing..."
  Server: "Your order status changed to shipped"
  
  ONE connection, BOTH sides can send ANYTIME → true real-time
```

**WebSocket vs HTTP comparison:**

```
HTTP:
  ✓ Stateless — easy to scale horizontally
  ✓ Cacheable — CDN can cache responses
  ✓ Works everywhere — no special setup
  ✗ Client must poll for updates — inefficient
  ✗ High latency for real-time (polling interval)

WebSocket:
  ✓ True real-time — server pushes instantly
  ✓ Low latency — persistent connection, no handshake per message
  ✓ Bidirectional — both client and server send freely
  ✗ Stateful — server must track connections (scaling harder)
  ✗ Not cacheable
  ✗ Firewall/proxy issues (some block WS)

Server-Sent Events (SSE) — middle ground:
  ✓ Server → Client only (one direction)
  ✓ Works over HTTP (no firewall issues)
  ✓ Auto-reconnect built in
  ✗ Client cannot send messages (only HTTP requests)
  Good for: notifications, live feeds, progress updates
```

**Real-world use cases:**

```
WebSockets:   Live chat, multiplayer games, collaborative editing,
              trading platforms, real-time auctions, pair programming

SSE:          Notifications, live scores, progress bars,
              news feeds, dashboard metrics, order tracking

Long Polling: Fallback when WebSockets unavailable (Socket.io handles this)
```

**Setup — install packages:**
```bash
npm install @nestjs/websockets @nestjs/platform-socket.io
npm install socket.io
npm install @types/socket.io --save-dev
```

---

## Table of Contents

1. [WebSocket Fundamentals — How It Works](#1-websocket-fundamentals--how-it-works)
2. [WebSocket Gateway — The NestJS Way](#2-websocket-gateway--the-nestjs-way)
3. [Socket.io Adapter — Advanced Configuration](#3-socketio-adapter--advanced-configuration)
4. [Events — Sending and Receiving Messages](#4-events--sending-and-receiving-messages)
5. [Namespaces — Logical Separation](#5-namespaces--logical-separation)
6. [Rooms — Group Communication](#6-rooms--group-communication)
7. [Broadcasting — Sending to Multiple Clients](#7-broadcasting--sending-to-multiple-clients)
8. [Authentication in WebSockets](#8-authentication-in-websockets)
9. [Real-World Example — Chat Application](#9-real-world-example--chat-application)
10. [Real-World Example — Live Notifications](#10-real-world-example--live-notifications)
11. [Scaling WebSockets — Redis Adapter](#11-scaling-websockets--redis-adapter)
12. [Integration with HTTP Controllers](#12-integration-with-http-controllers)
13. [Interview Questions & Answers](#13-interview-questions--answers)

---

## 1. WebSocket Fundamentals — How It Works

### The WebSocket handshake

```
1. Client sends HTTP Upgrade request:
   GET /socket.io/?EIO=4&transport=websocket HTTP/1.1
   Host: api.myapp.com
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
   Sec-WebSocket-Version: 13

2. Server responds with 101 Switching Protocols:
   HTTP/1.1 101 Switching Protocols
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

3. HTTP connection is UPGRADED to WebSocket
   Now both sides can send frames (not HTTP) at any time
   Connection stays open until explicitly closed

Socket.io adds on top of WebSocket:
  - Auto-reconnection on disconnect
  - Fallback to HTTP long-polling if WebSocket unavailable
  - Rooms and namespaces
  - Acknowledgements (guaranteed delivery)
  - Event-based API (emit/on)
  - Binary support
```

### Events vs HTTP

```
HTTP mindset:     Request → Response (one cycle)
WebSocket mindset: Events (fire and forget OR with ack)

Server emits events → Client listens
Client emits events → Server listens
Both can emit at ANY time, to ANYONE

Event names are just strings:
  'message'            → chat message
  'user:typing'        → typing indicator
  'notification'       → push notification
  'order:updated'      → order status change
  'error'              → error event (special Socket.io event)
  'connect'/'disconnect' → connection lifecycle (built-in)
```

---

## 2. WebSocket Gateway — The NestJS Way

### What is a Gateway?

A Gateway is NestJS's equivalent of a Controller — but for WebSocket events instead of HTTP requests. It's a class decorated with `@WebSocketGateway()` that listens for WebSocket events and handles them.

```typescript
// src/chat/chat.gateway.ts

import {
  WebSocketGateway,
  WebSocketServer,
  SubscribeMessage,
  MessageBody,
  ConnectedSocket,
  OnGatewayInit,
  OnGatewayConnection,
  OnGatewayDisconnect,
} from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';
import { Logger, UseGuards } from '@nestjs/common';

// @WebSocketGateway() — marks this class as a WebSocket gateway
@WebSocketGateway({
  // ── Port ──────────────────────────────────────────────────────────────
  port: 3001,
  // Optional: run WebSocket on a DIFFERENT port from HTTP
  // Default: same port as HTTP (usually 3000)
  // Same port is simpler and works with most proxies

  // ── Namespace ────────────────────────────────────────────────────────
  namespace: '/chat',
  // Optional: scope this gateway to a namespace
  // Clients connect to: ws://localhost:3000/chat
  // Covered in depth in section 5

  // ── CORS ─────────────────────────────────────────────────────────────
  cors: {
    origin: ['http://localhost:3001', 'https://myapp.com'],
    // Which origins can connect via WebSocket
    credentials: true,
    // Allow cookies to be sent (for session-based auth)
  },

  // ── Transport ────────────────────────────────────────────────────────
  transports: ['websocket', 'polling'],
  // WebSocket first, fall back to HTTP long-polling if needed
  // 'polling': works through strict firewalls
  // 'websocket' only: faster but may not work everywhere
})
export class ChatGateway
  implements OnGatewayInit, OnGatewayConnection, OnGatewayDisconnect
{
  // ── Lifecycle interfaces ───────────────────────────────────────────────
  // OnGatewayInit: called when gateway initializes
  // OnGatewayConnection: called when a client connects
  // OnGatewayDisconnect: called when a client disconnects
  // These are OPTIONAL — implement only what you need

  private readonly logger = new Logger(ChatGateway.name);

  // ── @WebSocketServer() — injects the Socket.io server instance ─────────
  @WebSocketServer()
  server: Server;
  // 'server' is the Socket.io Server instance
  // Use it to: emit to all clients, get connected sockets, join rooms
  // Available AFTER gateway initializes (not in constructor)

  // ── Lifecycle: Gateway initialized ────────────────────────────────────
  afterInit(server: Server): void {
    // Called once when the gateway is fully initialized
    // Good for: setting up server-level middleware, logging
    this.logger.log('WebSocket Gateway initialized');

    // Add Socket.io server-level middleware
    server.use((socket, next) => {
      // Runs for EVERY connection before OnGatewayConnection
      // Good for: authentication, logging, rate limiting
      this.logger.debug(`New connection attempt from ${socket.handshake.address}`);
      next();
    });
  }

  // ── Lifecycle: Client connected ────────────────────────────────────────
  async handleConnection(client: Socket): Promise<void> {
    // Called when a client successfully connects
    // client.id: unique Socket.io-generated ID for this connection
    // client.handshake: contains headers, auth, query, address

    this.logger.log(`Client connected: ${client.id}`);

    // You can access query parameters from the connection URL:
    // ws://localhost:3000/chat?token=xxx
    const token = client.handshake.auth.token || client.handshake.query.token;

    if (!token) {
      // Reject connection if no auth token
      client.emit('error', { message: 'Authentication required' });
      client.disconnect(true);
      // disconnect(true): force disconnect immediately
      return;
    }

    // Validate token and attach user to socket
    try {
      const user = await this.authService.validateToken(token as string);
      client.data.user = user;
      // client.data: Socket.io's way to attach custom data to a socket
      // Persists for the lifetime of the connection
      // All handlers can access client.data.user

      // Notify the client they're authenticated
      client.emit('authenticated', {
        userId: user.id,
        message: 'Connected successfully',
      });

      // Add client to their personal room (for direct messages)
      // Covered in section 6
      client.join(`user:${user.id}`);

    } catch (error) {
      client.emit('error', { message: 'Invalid authentication token' });
      client.disconnect(true);
    }
  }

  // ── Lifecycle: Client disconnected ────────────────────────────────────
  handleDisconnect(client: Socket): void {
    const user = client.data.user;
    this.logger.log(
      `Client disconnected: ${client.id} (User: ${user?.id || 'unknown'})`,
    );

    // Cleanup: notify others that user went offline
    if (user) {
      // Broadcast to all clients in the user's rooms
      client.broadcast.emit('user:offline', { userId: user.id });
      // client.broadcast: emit to EVERYONE except this client
    }
  }
}
```

### Registering the Gateway

```typescript
// src/chat/chat.module.ts

import { Module } from '@nestjs/common';
import { ChatGateway } from './chat.gateway';
import { ChatService } from './chat.service';
import { AuthModule } from '../auth/auth.module';

@Module({
  imports: [AuthModule],
  // Import AuthModule so ChatGateway can inject AuthService
  providers: [
    ChatGateway,
    // Gateways are PROVIDERS — registered in providers array
    // NOT in controllers array (that's for HTTP)
    ChatService,
  ],
})
export class ChatModule {}

// In AppModule:
// @Module({ imports: [ChatModule] })
// export class AppModule {}
```

---

## 3. Socket.io Adapter — Advanced Configuration

### Default setup (works out of the box)

```typescript
// src/main.ts
// NestJS uses Socket.io adapter by default — no extra config needed for basic use

import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Default: NestJS automatically sets up Socket.io
  // Your @WebSocketGateway() decorators are picked up automatically

  await app.listen(3000);
  // Both HTTP and WebSocket run on port 3000
}
bootstrap();
```

### Custom Socket.io adapter for advanced configuration

```typescript
// src/main.ts — when you need to customize Socket.io

import { NestFactory } from '@nestjs/core';
import { IoAdapter } from '@nestjs/platform-socket.io';
import { AppModule } from './app.module';
import { ServerOptions } from 'socket.io';

// Custom adapter to configure Socket.io options
export class CustomSocketIoAdapter extends IoAdapter {
  createIOServer(port: number, options?: ServerOptions) {
    const server = super.createIOServer(port, {
      ...options,

      // ── CORS ────────────────────────────────────────────────────────
      cors: {
        origin: process.env.ALLOWED_ORIGINS?.split(',') || '*',
        credentials: true,
        methods: ['GET', 'POST'],
      },

      // ── Connection management ────────────────────────────────────────
      pingTimeout: 60000,
      // How long to wait for pong before declaring connection dead (ms)
      // Default: 20000 (20s). Increase for slow networks.

      pingInterval: 25000,
      // How often to send ping to check connection health (ms)
      // Default: 25000 (25s)

      // ── Transport ────────────────────────────────────────────────────
      transports: ['websocket', 'polling'],
      // Try WebSocket first, fall back to polling

      allowUpgrades: true,
      // Allow upgrading from polling to WebSocket after initial connection

      // ── Message size limits ──────────────────────────────────────────
      maxHttpBufferSize: 1e6,
      // Max message size: 1MB
      // Prevent large payload attacks
      // Default: 1MB

      // ── Compression ──────────────────────────────────────────────────
      httpCompression: {
        threshold: 1024,
        // Only compress messages larger than 1KB
      },

      // ── Path ─────────────────────────────────────────────────────────
      path: '/socket.io/',
      // Default Socket.io path — change if needed for proxy routing
    });

    return server;
  }
}

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Use custom adapter
  app.useWebSocketAdapter(new CustomSocketIoAdapter(app));

  await app.listen(3000);
}
```

---

## 4. Events — Sending and Receiving Messages

### @SubscribeMessage() — handling events from clients

```typescript
// src/chat/chat.gateway.ts

import {
  SubscribeMessage,
  MessageBody,
  ConnectedSocket,
  WsException,
  WsResponse,
} from '@nestjs/websockets';
import { UseGuards, UsePipes, ValidationPipe } from '@nestjs/common';
import { Socket, Server } from 'socket.io';

@WebSocketGateway({ namespace: '/chat', cors: { origin: '*' } })
export class ChatGateway {
  @WebSocketServer()
  server: Server;

  // ── Basic event handler ──────────────────────────────────────────────────
  @SubscribeMessage('message')
  // 'message': the event name — client emits this event name
  // Client: socket.emit('message', { content: 'Hello!' })

  handleMessage(
    @MessageBody() data: { content: string; roomId: string },
    // @MessageBody(): extracts the data payload from the event
    // Like @Body() but for WebSocket events

    @ConnectedSocket() client: Socket,
    // @ConnectedSocket(): injects the Socket instance of the sender
    // Like @Req() but for WebSocket events
  ): WsResponse<any> {
    // WsResponse<T>: the return type for synchronous event responses
    // { event: string, data: T }

    const user = client.data.user;

    // Return a response directly to the sender:
    return {
      event: 'message',    // Event name to emit back to client
      data: {
        id: Date.now(),
        content: data.content,
        sender: { id: user.id, name: user.name },
        timestamp: new Date().toISOString(),
        roomId: data.roomId,
      },
    };
    // Client receives: socket.on('message', (data) => { ... })
  }

  // ── Async event handler ──────────────────────────────────────────────────
  @SubscribeMessage('sendMessage')
  async handleSendMessage(
    @MessageBody() dto: CreateMessageDto,
    @ConnectedSocket() client: Socket,
  ): Promise<void> {
    // Async: when you need to await database operations

    const user = client.data.user;
    if (!user) {
      throw new WsException('Unauthorized');
      // WsException: WebSocket equivalent of HttpException
      // Emitted to client as 'error' event
    }

    try {
      // Save message to database
      const message = await this.chatService.createMessage({
        content: dto.content,
        roomId: dto.roomId,
        senderId: user.id,
      });

      // Broadcast to everyone in the room (including sender)
      this.server.to(dto.roomId).emit('newMessage', {
        id: message.id,
        content: message.content,
        sender: { id: user.id, name: user.name },
        timestamp: message.createdAt,
      });
      // Note: void return — no direct response to sender
      // They receive the 'newMessage' event as part of the room broadcast

    } catch (error) {
      throw new WsException('Failed to send message');
    }
  }

  // ── Event with acknowledgement ───────────────────────────────────────────
  @SubscribeMessage('joinRoom')
  async handleJoinRoom(
    @MessageBody() data: { roomId: string },
    @ConnectedSocket() client: Socket,
  ): Promise<{ success: boolean; roomId: string; members: number }> {
    // When you return a value from an async handler,
    // Socket.io sends it as an ACKNOWLEDGEMENT to the sender
    // Client: socket.emit('joinRoom', { roomId: 'abc' }, (response) => {
    //           console.log(response) → { success: true, roomId: 'abc', members: 5 }
    //         })

    const { roomId } = data;
    const user = client.data.user;

    // Join the Socket.io room
    await client.join(roomId);
    // client.join(): adds this socket to the named room
    // Now this socket receives all events emitted to this room

    // Get current member count
    const sockets = await this.server.in(roomId).fetchSockets();
    const memberCount = sockets.length;

    // Notify others in the room (not the joiner themselves)
    client.to(roomId).emit('userJoined', {
      userId: user.id,
      username: user.name,
      timestamp: new Date().toISOString(),
    });
    // client.to(roomId): sends to everyone in room EXCEPT sender

    return {
      success: true,
      roomId,
      members: memberCount,
    };
    // This return value is the acknowledgement sent back to the joining client
  }

  // ── Typing indicator ──────────────────────────────────────────────────────
  @SubscribeMessage('typing')
  handleTyping(
    @MessageBody() data: { roomId: string; isTyping: boolean },
    @ConnectedSocket() client: Socket,
  ): void {
    const user = client.data.user;

    // Broadcast typing status to everyone in room EXCEPT the typer
    client.to(data.roomId).emit('userTyping', {
      userId: user.id,
      username: user.name,
      isTyping: data.isTyping,
    });
    // No return — typing indicator is fire-and-forget
  }

  // ── Leave room ─────────────────────────────────────────────────────────
  @SubscribeMessage('leaveRoom')
  async handleLeaveRoom(
    @MessageBody() data: { roomId: string },
    @ConnectedSocket() client: Socket,
  ): Promise<void> {
    const user = client.data.user;
    await client.leave(data.roomId);
    // client.leave(): removes this socket from the room

    // Notify others in the room
    client.to(data.roomId).emit('userLeft', {
      userId: user.id,
      username: user.name,
    });
  }
}
```

### Validation in WebSocket events

```typescript
// src/chat/dto/create-message.dto.ts

import { IsString, IsNotEmpty, MaxLength, IsUUID } from 'class-validator';

export class CreateMessageDto {
  @IsString()
  @IsNotEmpty()
  @MaxLength(2000)
  content: string;

  @IsUUID()
  roomId: string;
}

// Apply ValidationPipe to WebSocket handlers:
@WebSocketGateway({ namespace: '/chat' })
@UsePipes(new ValidationPipe({ transform: true, whitelist: true }))
// Apply globally to ALL event handlers in this gateway
export class ChatGateway {

  @SubscribeMessage('sendMessage')
  async handleMessage(
    @MessageBody() dto: CreateMessageDto,
    // ValidationPipe validates dto before this method runs
    // If invalid: WsException is thrown automatically
    @ConnectedSocket() client: Socket,
  ) {
    // dto is guaranteed valid here
  }
}
```

---

## 5. Namespaces — Logical Separation

### What are namespaces?

Namespaces divide your WebSocket server into separate communication channels. Clients connect to specific namespaces. Events in one namespace don't bleed into another.

```
Default namespace: /
  All clients connect to ws://localhost:3000
  Events visible to all clients in default namespace

Chat namespace: /chat
  Only chat clients connect to ws://localhost:3000/chat
  Chat events only visible to chat clients

Notifications namespace: /notifications
  Only notification clients connect to ws://localhost:3000/notifications
  Notification events separate from chat

Admin namespace: /admin
  Only admin dashboard connects to ws://localhost:3000/admin
  Separate namespace for administrative features
```

```typescript
// Multiple gateways = multiple namespaces

// src/chat/chat.gateway.ts
@WebSocketGateway({ namespace: '/chat' })
export class ChatGateway {
  @WebSocketServer() server: Server;

  @SubscribeMessage('message')
  handleMessage(@MessageBody() data: any) {
    // This event only reaches clients connected to /chat
  }
}

// src/notifications/notifications.gateway.ts
@WebSocketGateway({ namespace: '/notifications' })
export class NotificationsGateway {
  @WebSocketServer() server: Server;

  @SubscribeMessage('subscribe')
  handleSubscribe(@ConnectedSocket() client: Socket) {
    // This event only reaches clients connected to /notifications
  }

  // Method to push notification from outside (called by HTTP services)
  sendNotification(userId: number, notification: any) {
    this.server.to(`user:${userId}`).emit('notification', notification);
  }
}

// src/admin/admin.gateway.ts
@WebSocketGateway({ namespace: '/admin' })
@UseGuards(WsAdminGuard)  // Only admins can connect
export class AdminGateway {
  @WebSocketServer() server: Server;

  broadcastMetrics(metrics: SystemMetrics) {
    this.server.emit('metrics', metrics);
    // Sends to ALL admin dashboard connections
  }
}
```

### Client-side connection (for context)

```javascript
// Frontend JavaScript — connecting to specific namespaces
import { io } from 'socket.io-client';

// Connect to chat namespace
const chatSocket = io('http://localhost:3000/chat', {
  auth: { token: 'jwt-token-here' },
  // auth: sent in handshake.auth on server side
});

chatSocket.on('connect', () => console.log('Chat connected'));
chatSocket.on('newMessage', (msg) => console.log('New message:', msg));
chatSocket.emit('sendMessage', { content: 'Hello!', roomId: 'room-1' });

// Connect to notifications namespace separately
const notifSocket = io('http://localhost:3000/notifications', {
  auth: { token: 'jwt-token-here' },
});

notifSocket.on('notification', (notif) => showNotification(notif));
```

---

## 6. Rooms — Group Communication

### What are rooms?

Rooms are server-side groupings of sockets. Unlike namespaces (which clients explicitly connect to), rooms are managed by the SERVER and clients are added/removed by the server's logic.

```
Namespace /chat:
  Room 'general':    [socket1, socket2, socket3]
  Room 'team-alpha': [socket1, socket4, socket5]
  Room 'dm-1-2':     [socket1, socket2]           ← Direct messages

  socket1 is in: general, team-alpha, dm-1-2
  socket2 is in: general, dm-1-2
  socket3 is in: general

  Emit to 'general' → socket1, socket2, socket3 all receive
  Emit to 'team-alpha' → socket1, socket4, socket5 receive
  socket3 does NOT receive team-alpha messages
```

```typescript
// src/chat/chat.gateway.ts — comprehensive room management

@WebSocketGateway({ namespace: '/chat' })
export class ChatGateway {
  @WebSocketServer()
  server: Server;

  async handleConnection(client: Socket) {
    const user = client.data.user;
    if (!user) return;

    // ── Personal room — for direct messages and notifications ────────────
    await client.join(`user:${user.id}`);
    // Every user gets their own room: 'user:1', 'user:2', etc.
    // Use to send direct messages: server.to('user:5').emit(...)

    // ── Auto-join rooms based on user membership ──────────────────────────
    const userRooms = await this.chatService.getUserRooms(user.id);
    // Get all rooms this user is a member of from database

    for (const room of userRooms) {
      await client.join(`room:${room.id}`);
      // Join all rooms the user belongs to on connection
    }

    // ── Announce online status ────────────────────────────────────────────
    await this.updateUserStatus(user.id, 'online');
    this.server.emit('user:online', { userId: user.id });
  }

  // ── Join a room ────────────────────────────────────────────────────────
  @SubscribeMessage('room:join')
  async joinRoom(
    @MessageBody() data: { roomId: string },
    @ConnectedSocket() client: Socket,
  ) {
    const user = client.data.user;
    const roomKey = `room:${data.roomId}`;

    // Verify user has permission to join this room
    const canJoin = await this.chatService.canUserJoinRoom(
      user.id,
      data.roomId,
    );

    if (!canJoin) {
      throw new WsException('Not authorized to join this room');
    }

    // Add user to room in database
    await this.chatService.addUserToRoom(user.id, data.roomId);

    // Join Socket.io room
    await client.join(roomKey);

    // Get room history (last 50 messages)
    const history = await this.chatService.getRoomHistory(data.roomId, 50);

    // Send room history to the joining user only
    client.emit('room:history', { roomId: data.roomId, messages: history });

    // Notify everyone in the room (except the joiner)
    client.to(roomKey).emit('room:userJoined', {
      roomId: data.roomId,
      user: { id: user.id, name: user.name, avatar: user.avatar },
      timestamp: new Date().toISOString(),
    });

    // Send member list to the joining user
    const sockets = await this.server.in(roomKey).fetchSockets();
    const memberIds = sockets.map(s => (s as any).data.user?.id).filter(Boolean);

    return { success: true, memberIds };
  }

  // ── Leave a room ───────────────────────────────────────────────────────
  @SubscribeMessage('room:leave')
  async leaveRoom(
    @MessageBody() data: { roomId: string },
    @ConnectedSocket() client: Socket,
  ) {
    const user = client.data.user;
    const roomKey = `room:${data.roomId}`;

    await client.leave(roomKey);

    client.to(roomKey).emit('room:userLeft', {
      roomId: data.roomId,
      userId: user.id,
    });

    return { success: true };
  }

  // ── Create a new room ──────────────────────────────────────────────────
  @SubscribeMessage('room:create')
  async createRoom(
    @MessageBody() data: { name: string; members: number[] },
    @ConnectedSocket() client: Socket,
  ) {
    const user = client.data.user;

    // Create room in database
    const room = await this.chatService.createRoom({
      name: data.name,
      creatorId: user.id,
      members: [user.id, ...data.members],
    });

    // Join the room creator
    await client.join(`room:${room.id}`);

    // Find and join all invited members (if they're currently connected)
    for (const memberId of data.members) {
      // Emit to member's personal room (they might be offline)
      this.server.to(`user:${memberId}`).emit('room:invited', {
        roomId: room.id,
        name: room.name,
        invitedBy: { id: user.id, name: user.name },
      });

      // If the member is online, auto-join them to the room
      // We can't force join a socket by user ID directly
      // They'll join when they receive 'room:invited' event
    }

    return { success: true, room };
  }

  // ── Get room members ───────────────────────────────────────────────────
  @SubscribeMessage('room:members')
  async getRoomMembers(
    @MessageBody() data: { roomId: string },
  ) {
    const roomKey = `room:${data.roomId}`;

    // Get all sockets currently in the room
    const sockets = await this.server.in(roomKey).fetchSockets();

    const onlineMembers = sockets.map(socket => ({
      socketId: socket.id,
      userId: (socket.data as any).user?.id,
      username: (socket.data as any).user?.name,
    }));

    // Get all room members from database (including offline)
    const allMembers = await this.chatService.getRoomMembers(data.roomId);

    return {
      onlineMembers,
      totalMembers: allMembers.length,
    };
  }
}
```

---

## 7. Broadcasting — Sending to Multiple Clients

### All broadcasting patterns

```typescript
// Comprehensive broadcasting guide in a gateway:

@WebSocketGateway({ namespace: '/chat' })
export class ChatGateway {
  @WebSocketServer() server: Server;

  demonstrateBroadcasting(client: Socket) {
    const data = { message: 'Hello!' };

    // ── 1. Send to specific socket (one client) ────────────────────────
    client.emit('event', data);
    // or
    this.server.to(client.id).emit('event', data);
    // Sends to ONE specific client only

    // ── 2. Send to everyone EXCEPT sender ─────────────────────────────
    client.broadcast.emit('event', data);
    // Sends to ALL connected clients in namespace EXCEPT this client
    // Use for: "User X is now online" — everyone sees it except X

    // ── 3. Send to ALL clients in namespace ───────────────────────────
    this.server.emit('event', data);
    // Sends to EVERYONE connected to this namespace
    // Use for: system announcements, server-wide events

    // ── 4. Send to a room ─────────────────────────────────────────────
    this.server.to('room:123').emit('event', data);
    // Sends to ALL sockets in the room 'room:123'
    // Use for: chat messages, room-level events

    // ── 5. Send to room EXCEPT sender ─────────────────────────────────
    client.to('room:123').emit('event', data);
    // Sends to all in room EXCEPT the current client
    // Use for: "User X sent a message" — X already knows they sent it

    // ── 6. Send to multiple rooms ──────────────────────────────────────
    this.server.to('room:123').to('room:456').emit('event', data);
    // Sends to all sockets in EITHER room
    // Deduplicates: if socket is in both rooms, receives only ONCE

    // ── 7. Send to specific user by user ID ────────────────────────────
    this.server.to(`user:${userId}`).emit('event', data);
    // Uses the personal room we join on connection
    // Works even if user has multiple connections (multiple tabs)
    // All connections of the user receive the event

    // ── 8. Exclude specific sockets ────────────────────────────────────
    this.server.except(client.id).emit('event', data);
    // Sends to everyone EXCEPT the specified socket IDs
    // Can chain: .except('id1').except('id2')

    // ── 9. Send to room and exclude sockets ───────────────────────────
    this.server.to('room:123').except(client.id).emit('event', data);
    // In room 123, everyone EXCEPT this client

    // ── 10. Volatile emit — skip if client is not ready ────────────────
    client.volatile.emit('liveMetrics', data);
    // If client's connection is slow or buffer is full → skip this message
    // Use for: live metrics, typing indicators — if dropped, no big deal
    // Don't use for: important messages that must be delivered

    // ── 11. With timeout — detect if client received it ────────────────
    this.server
      .timeout(5000)
      .to('room:123')
      .emit('importantEvent', data, (err, responses) => {
        if (err) {
          console.log('Some clients did not acknowledge within 5 seconds');
        }
        console.log('Responses from clients:', responses);
      });
  }
}
```

---

## 8. Authentication in WebSockets

### The challenge

```
HTTP auth is easy: check Authorization header on each request
WebSocket: ONE connection that lasts hours/days
  → Auth token might EXPIRE during the connection
  → Can't check auth on every message (overhead)
  → Must authenticate AT CONNECTION TIME
  → And handle token expiry DURING connection
```

### Method 1: Auth at connection time (recommended)

```typescript
// src/auth/ws-jwt.guard.ts

import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { WsException } from '@nestjs/websockets';
import { Socket } from 'socket.io';

@Injectable()
export class WsJwtGuard implements CanActivate {
  constructor(private jwtService: JwtService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    // For WebSocket: switch to WS context
    const client: Socket = context.switchToWs().getClient();
    // context.switchToWs() — like switchToHttp() but for WebSockets

    const token =
      client.handshake.auth.token ||         // socket.io auth: { token: '...' }
      client.handshake.headers.authorization  // Authorization: Bearer xxx
        ?.replace('Bearer ', '');

    if (!token) {
      throw new WsException('No authentication token provided');
    }

    try {
      const payload = await this.jwtService.verifyAsync(token, {
        secret: process.env.JWT_SECRET,
      });

      // Attach user to client.data (persists for connection lifetime)
      client.data.user = payload;
      return true;

    } catch {
      throw new WsException('Invalid or expired token');
    }
  }
}

// Usage on individual event handlers:
@SubscribeMessage('sendMessage')
@UseGuards(WsJwtGuard)
async handleMessage(
  @MessageBody() data: any,
  @ConnectedSocket() client: Socket,
) {
  const user = client.data.user;
  // user is guaranteed to be valid here
}
```

### Method 2: Middleware-based auth (auth on connection, not per-event)

```typescript
// src/chat/chat.gateway.ts
// Authenticate ONCE during connection — not on every event

@WebSocketGateway({ namespace: '/chat' })
export class ChatGateway implements OnGatewayInit {
  @WebSocketServer() server: Server;

  constructor(
    private jwtService: JwtService,
    private usersService: UsersService,
  ) {}

  afterInit(server: Server) {
    // Socket.io middleware: runs for EVERY connection attempt
    server.use(async (socket: Socket, next) => {
      try {
        const token =
          socket.handshake.auth.token ||
          socket.handshake.headers.authorization?.replace('Bearer ', '');

        if (!token) {
          return next(new Error('Authentication required'));
          // Passing an Error to next() rejects the connection
        }

        const payload = await this.jwtService.verifyAsync(token);
        const user = await this.usersService.findOne(parseInt(payload.sub));

        if (!user || !user.isActive) {
          return next(new Error('User not found or inactive'));
        }

        // Attach user to socket — available in ALL handlers
        socket.data.user = user;
        next();
        // next() with no arguments = accept connection

      } catch (error) {
        next(new Error('Invalid token'));
      }
    });
  }

  async handleConnection(client: Socket) {
    // By the time handleConnection runs, middleware already validated the token
    // client.data.user is already set and verified
    const user = client.data.user;
    console.log(`User ${user.email} connected`);
  }
}
```

### Handling token expiry during active connections

```typescript
// Problem: User connects with 15-minute access token
// After 15 minutes: they're still connected but token is expired
// If they emit an event after expiry → what happens?

// Solution: Client-side token refresh + server-side re-auth event

// Gateway handler for token refresh:
@SubscribeMessage('auth:refresh')
async handleTokenRefresh(
  @MessageBody() data: { refreshToken: string },
  @ConnectedSocket() client: Socket,
): Promise<{ accessToken: string }> {
  try {
    // Validate refresh token
    const payload = await this.jwtService.verifyAsync(data.refreshToken, {
      secret: process.env.JWT_REFRESH_SECRET,
    });

    // Get fresh user from DB
    const user = await this.usersService.findOne(parseInt(payload.sub));
    if (!user || !user.isActive) throw new Error('User not found');

    // Update socket's user data with fresh data
    client.data.user = user;

    // Generate new access token
    const accessToken = this.jwtService.sign(
      { sub: user.id.toString(), email: user.email, role: user.role },
      { expiresIn: '15m' },
    );

    return { accessToken };
    // Client stores new access token for future HTTP requests

  } catch {
    // Refresh failed → disconnect
    client.emit('auth:expired', { message: 'Session expired. Please log in again.' });
    client.disconnect(true);
    throw new WsException('Token refresh failed');
  }
}
```

---

## 9. Real-World Example — Chat Application

### Complete chat gateway

```typescript
// src/chat/chat.gateway.ts — production-quality chat

import {
  WebSocketGateway, WebSocketServer, SubscribeMessage,
  MessageBody, ConnectedSocket, OnGatewayConnection,
  OnGatewayDisconnect, WsException,
} from '@nestjs/websockets';
import { UseGuards, UsePipes, ValidationPipe, Logger } from '@nestjs/common';
import { Server, Socket } from 'socket.io';
import { ChatService } from './chat.service';
import { JwtService } from '@nestjs/jwt';
import { UsersService } from '../users/users.service';

@WebSocketGateway({
  namespace: '/chat',
  cors: {
    origin: process.env.ALLOWED_ORIGINS?.split(',') || '*',
    credentials: true,
  },
})
@UsePipes(new ValidationPipe({ transform: true, whitelist: true }))
export class ChatGateway
  implements OnGatewayConnection, OnGatewayDisconnect
{
  @WebSocketServer() server: Server;
  private readonly logger = new Logger(ChatGateway.name);

  // Track online users: userId → Set of socket IDs
  // Handles users with multiple tabs open
  private onlineUsers = new Map<number, Set<string>>();

  constructor(
    private chatService: ChatService,
    private jwtService: JwtService,
    private usersService: UsersService,
  ) {}

  // ── Connection handling ────────────────────────────────────────────────
  async handleConnection(client: Socket) {
    try {
      const token =
        client.handshake.auth?.token ||
        client.handshake.headers.authorization?.replace('Bearer ', '');

      if (!token) throw new Error('No token');

      const payload = await this.jwtService.verifyAsync(token);
      const user = await this.usersService.findOne(parseInt(payload.sub));

      if (!user) throw new Error('User not found');

      client.data.user = user;

      // Track online status
      if (!this.onlineUsers.has(user.id)) {
        this.onlineUsers.set(user.id, new Set());
      }
      this.onlineUsers.get(user.id)!.add(client.id);

      // Join personal room
      client.join(`user:${user.id}`);

      // Rejoin all rooms the user is a member of
      const rooms = await this.chatService.getUserRooms(user.id);
      for (const room of rooms) {
        client.join(`room:${room.id}`);
      }

      // Only announce online if this is their FIRST connection
      // (might be opening a second tab)
      if (this.onlineUsers.get(user.id)!.size === 1) {
        this.server.emit('user:online', {
          userId: user.id,
          name: user.firstName,
        });
      }

      this.logger.log(`User ${user.email} connected (${client.id})`);

    } catch (error) {
      this.logger.warn(`Connection rejected: ${error.message}`);
      client.emit('error', { message: 'Authentication failed' });
      client.disconnect(true);
    }
  }

  async handleDisconnect(client: Socket) {
    const user = client.data?.user;
    if (!user) return;

    // Remove this socket from online tracking
    const userSockets = this.onlineUsers.get(user.id);
    if (userSockets) {
      userSockets.delete(client.id);

      // Only announce offline if ALL connections closed
      if (userSockets.size === 0) {
        this.onlineUsers.delete(user.id);
        this.server.emit('user:offline', { userId: user.id });
      }
    }

    this.logger.log(`User ${user.email} disconnected (${client.id})`);
  }

  // ── Send message ───────────────────────────────────────────────────────
  @SubscribeMessage('message:send')
  async sendMessage(
    @MessageBody() dto: SendMessageDto,
    @ConnectedSocket() client: Socket,
  ) {
    const user = client.data.user;

    // Verify user is member of the room
    const isMember = await this.chatService.isUserInRoom(user.id, dto.roomId);
    if (!isMember) throw new WsException('Not a member of this room');

    // Save to database
    const message = await this.chatService.saveMessage({
      content: dto.content,
      roomId: dto.roomId,
      senderId: user.id,
      messageType: dto.type || 'text',
    });

    // Broadcast to everyone in the room
    this.server.to(`room:${dto.roomId}`).emit('message:new', {
      id: message.id,
      content: message.content,
      type: message.messageType,
      sender: {
        id: user.id,
        name: `${user.firstName} ${user.lastName}`,
        avatar: user.avatar,
      },
      roomId: dto.roomId,
      timestamp: message.createdAt.toISOString(),
      readBy: [user.id],  // Sender already "read" it
    });
  }

  // ── Message read receipt ───────────────────────────────────────────────
  @SubscribeMessage('message:read')
  async markAsRead(
    @MessageBody() dto: { messageId: number; roomId: string },
    @ConnectedSocket() client: Socket,
  ) {
    const user = client.data.user;

    await this.chatService.markMessageRead(dto.messageId, user.id);

    // Notify message sender that their message was read
    const message = await this.chatService.getMessage(dto.messageId);
    this.server.to(`user:${message.senderId}`).emit('message:readBy', {
      messageId: dto.messageId,
      readBy: user.id,
      timestamp: new Date().toISOString(),
    });
  }

  // ── Typing indicator ────────────────────────────────────────────────────
  @SubscribeMessage('typing:start')
  handleTypingStart(
    @MessageBody() dto: { roomId: string },
    @ConnectedSocket() client: Socket,
  ) {
    const user = client.data.user;
    client.to(`room:${dto.roomId}`).emit('typing:update', {
      userId: user.id,
      name: user.firstName,
      roomId: dto.roomId,
      isTyping: true,
    });
  }

  @SubscribeMessage('typing:stop')
  handleTypingStop(
    @MessageBody() dto: { roomId: string },
    @ConnectedSocket() client: Socket,
  ) {
    const user = client.data.user;
    client.to(`room:${dto.roomId}`).emit('typing:update', {
      userId: user.id,
      name: user.firstName,
      roomId: dto.roomId,
      isTyping: false,
    });
  }

  // ── Method to emit from outside (called by HTTP endpoints) ─────────────
  emitToRoom(roomId: string, event: string, data: any) {
    this.server.to(`room:${roomId}`).emit(event, data);
  }

  emitToUser(userId: number, event: string, data: any) {
    this.server.to(`user:${userId}`).emit(event, data);
  }

  isUserOnline(userId: number): boolean {
    return this.onlineUsers.has(userId) &&
      this.onlineUsers.get(userId)!.size > 0;
  }
}
```

---

## 10. Real-World Example — Live Notifications

```typescript
// src/notifications/notifications.gateway.ts

@WebSocketGateway({ namespace: '/notifications' })
export class NotificationsGateway implements OnGatewayConnection {
  @WebSocketServer() server: Server;

  async handleConnection(client: Socket) {
    const token = client.handshake.auth?.token;
    if (!token) {
      client.disconnect(true);
      return;
    }

    try {
      const payload = await this.jwtService.verifyAsync(token);
      client.data.userId = parseInt(payload.sub);
      client.join(`user:${payload.sub}`);
      // Personal room for this user's notifications

      // Send any pending/unread notifications on connect
      const unread = await this.notificationService.getUnread(
        parseInt(payload.sub),
      );
      if (unread.length > 0) {
        client.emit('notifications:unread', { notifications: unread });
      }

    } catch {
      client.disconnect(true);
    }
  }

  // Called by OTHER services (not by client events)
  async sendNotification(userId: number, notification: {
    type: 'ORDER_UPDATE' | 'MESSAGE' | 'PAYMENT' | 'SYSTEM';
    title: string;
    body: string;
    data?: any;
  }) {
    // Save to database first
    const saved = await this.notificationService.create({
      userId,
      ...notification,
    });

    // Emit to user (works even if user has multiple tabs)
    this.server.to(`user:${userId}`).emit('notification:new', {
      id: saved.id,
      type: saved.type,
      title: saved.title,
      body: saved.body,
      data: saved.data,
      isRead: false,
      createdAt: saved.createdAt.toISOString(),
    });

    // Update unread count
    const unreadCount = await this.notificationService.getUnreadCount(userId);
    this.server.to(`user:${userId}`).emit('notification:count', {
      count: unreadCount,
    });
  }

  @SubscribeMessage('notification:markRead')
  async markRead(
    @MessageBody() data: { notificationId: number },
    @ConnectedSocket() client: Socket,
  ) {
    const userId = client.data.userId;
    await this.notificationService.markRead(data.notificationId, userId);

    const unreadCount = await this.notificationService.getUnreadCount(userId);
    client.emit('notification:count', { count: unreadCount });
  }
}

// src/orders/orders.service.ts — emit notification from HTTP service
@Injectable()
export class OrdersService {
  constructor(
    private notificationsGateway: NotificationsGateway,
  ) {}

  async updateOrderStatus(orderId: number, status: OrderStatus) {
    const order = await this.orderRepository.findOne({
      where: { id: orderId },
      relations: { user: true },
    });

    await this.orderRepository.update(orderId, { status });

    // Push real-time notification to user
    await this.notificationsGateway.sendNotification(order.userId, {
      type: 'ORDER_UPDATE',
      title: 'Order Status Updated',
      body: `Your order #${orderId} is now ${status}`,
      data: { orderId, status },
    });
  }
}
```

---

## 11. Scaling WebSockets — Redis Adapter

### The scaling problem

```
Single server (no problem):
  Server 1: [socket1, socket2, socket3]
  server.to('room:1').emit('msg') → socket1 and socket2 receive ✓

Multiple servers (problem!):
  Server 1: [socket1, socket2]    ← User A's socket
  Server 2: [socket3, socket4]    ← User B's socket
  
  Server 1: server.to('user:B').emit('msg')
  → Only looks at Server 1's sockets
  → socket3, socket4 on Server 2 NEVER receive it! ✗

Solution: Redis Pub/Sub adapter
  All servers share a Redis channel
  Server 1 emits → Redis → all servers receive → forward to their sockets
```

```typescript
// npm install @socket.io/redis-adapter ioredis

// src/main.ts

import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';
import { RedisIoAdapter } from './adapters/redis-io.adapter';

// src/adapters/redis-io.adapter.ts

import { IoAdapter } from '@nestjs/platform-socket.io';
import { ServerOptions } from 'socket.io';
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';
import { ConfigService } from '@nestjs/config';

export class RedisIoAdapter extends IoAdapter {
  private adapterConstructor: ReturnType<typeof createAdapter>;

  constructor(
    app: any,
    private configService: ConfigService,
  ) {
    super(app);
  }

  async connectToRedis(): Promise<void> {
    const redisUrl = this.configService.getOrThrow('REDIS_URL');
    // REDIS_URL=redis://localhost:6379

    // Create two Redis clients (pub and sub — both required)
    const pubClient = createClient({ url: redisUrl });
    const subClient = pubClient.duplicate();
    // Redis requires separate connections for pub and sub

    await Promise.all([pubClient.connect(), subClient.connect()]);

    this.adapterConstructor = createAdapter(pubClient, subClient);
  }

  createIOServer(port: number, options?: ServerOptions): any {
    const server = super.createIOServer(port, options);
    server.adapter(this.adapterConstructor);
    // Plug in the Redis adapter
    // Now ALL emit() calls are synchronized across server instances
    return server;
  }
}

// In main.ts:
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const configService = app.get(ConfigService);
  const redisIoAdapter = new RedisIoAdapter(app, configService);
  await redisIoAdapter.connectToRedis();

  app.useWebSocketAdapter(redisIoAdapter);
  // Replace default adapter with Redis adapter

  await app.listen(3000);
}

// Now with 10 servers:
// User A connects to Server 1, User B connects to Server 5
// Server 1: server.to('user:B').emit('hello')
// → Redis pub/sub → Server 5 receives → forwards to User B's socket ✓
```

---

## 12. Integration with HTTP Controllers

### Injecting Gateway into HTTP services

```typescript
// Use case: HTTP endpoint triggers WebSocket event
// e.g., Admin changes order status → user's browser updates in real-time

// src/orders/orders.controller.ts

@Controller('orders')
export class OrdersController {
  constructor(
    private ordersService: OrdersService,
    private notificationsGateway: NotificationsGateway,
    // Inject the WebSocket gateway directly
    // Works because gateways are injectable providers
  ) {}

  @Patch(':id/status')
  @AdminRoute()
  async updateStatus(
    @Param('id', ParseIntPipe) id: number,
    @Body() dto: UpdateOrderStatusDto,
  ) {
    const order = await this.ordersService.updateStatus(id, dto.status);

    // Push real-time update to the customer
    this.notificationsGateway.emitToUser(order.userId, 'order:statusUpdated', {
      orderId: id,
      newStatus: dto.status,
      message: `Order #${id} is now ${dto.status}`,
    });

    return order;
  }
}

// The module needs both:
@Module({
  imports: [NotificationsModule],
  // Import the module that provides NotificationsGateway
  controllers: [OrdersController],
  providers: [OrdersService],
})
export class OrdersModule {}
```

---

## 13. Interview Questions & Answers

**Q1: What is the difference between WebSockets, HTTP polling, and Server-Sent Events?**

> "HTTP polling is when the client repeatedly makes HTTP requests to check for new data — simple but inefficient, wasting bandwidth even when there's nothing new. Long polling is an improvement where the server holds the request open until data is available, then the client immediately makes another request. Server-Sent Events (SSE) is a one-directional stream from server to client over HTTP — the server can push updates whenever they happen, but the client can't send messages back (only make new HTTP requests). WebSockets create a persistent bidirectional TCP connection — after an HTTP upgrade handshake, both client and server can send messages at any time with minimal overhead. SSE is simpler and works better through proxies; WebSockets are needed for true two-way real-time communication like chat, gaming, or collaborative editing."

---

**Q2: What is a Socket.io namespace and how is it different from a room?**

> "A namespace is a connection-level separation — clients explicitly connect to a specific namespace URL like `/chat` or `/notifications`. Each namespace has its own event handlers, middleware, and connection pools. Rooms are server-side groupings within a namespace — the server adds sockets to rooms, and clients don't see or choose rooms directly. You use rooms to organize sockets for targeted broadcasting: a chat room, a user's personal room for direct messages, or a team room. The key distinction: namespaces separate concerns at the application level (chat vs notifications vs admin), while rooms are used for dynamic grouping within those concerns (which chat rooms a user has joined, which team they're in)."

---

**Q3: How do you authenticate WebSocket connections in NestJS?**

> "I authenticate WebSocket connections at connection time, not per-message. When the client connects, they pass the JWT in the Socket.io auth handshake: `io.connect(url, { auth: { token: 'Bearer ...' } })`. In the `handleConnection` lifecycle method or a Socket.io server-level middleware registered in `afterInit`, I extract and verify the JWT using JwtService. If valid, I attach the decoded user to `socket.data.user` — this persists for the lifetime of the connection. If invalid, I call `client.disconnect(true)` to reject. For per-event authorization, I create a `WsJwtGuard` that reads from `socket.data.user` set during connection. The challenge is token expiry — I handle this by having the client emit a `auth:refresh` event before the access token expires, and the server validates the refresh token and updates `socket.data.user`."

---

**Q4: How do you scale WebSockets across multiple server instances?**

> "The fundamental problem is that each server only knows about its own connected sockets. When Server 1 emits to a room, sockets on Server 2 in that room never receive it. The solution is the Redis Pub/Sub adapter from `@socket.io/redis-adapter`. Instead of emitting directly to local sockets, the adapter publishes the event to a Redis channel. All server instances subscribe to that channel and forward events to their local sockets matching the target. This is transparent — you still call `server.to('room:X').emit()` and it works across all instances. I create the adapter with two Redis clients (one for publishing, one for subscribing) and plug it in via `app.useWebSocketAdapter(new RedisIoAdapter(app))`. For very high scale, Socket.io also supports horizontal scaling with sticky sessions, but the Redis adapter is the cleaner solution."

---

**Q5: What is the difference between `socket.emit`, `socket.broadcast.emit`, and `server.emit`?**

> "`socket.emit('event', data)` sends the event only to that specific socket — the client who owns this socket receives it. `socket.broadcast.emit('event', data)` sends to ALL connected clients in the namespace EXCEPT the socket who triggered it — useful for 'user X sent a message' where X already knows they sent it. `server.emit('event', data)` sends to absolutely ALL connected clients in the namespace including the triggering socket — for system-wide announcements. Adding `.to('room:X')` before emit scopes it to that room: `server.to('room:X').emit` → all in room X including sender; `socket.to('room:X').emit` → all in room X except sender. For user-specific events, I use the personal room pattern: each user joins `user:{id}` on connection, so `server.to('user:5').emit` reaches all of user 5's connections including multiple tabs."

---

**Q6: How would you implement a typing indicator?**

> "The typing indicator is a good example of real-time that needs careful design. When a user starts typing, the client emits `typing:start` with the room ID. The gateway receives this and broadcasts to everyone else in the room (using `client.to(roomId).emit`) that user X is typing. When they stop or send the message, they emit `typing:stop` and the gateway broadcasts accordingly. Client-side, each user tracked as 'typing' should have a timeout — if no `typing:start` comes within 3 seconds, auto-clear their typing indicator (in case `typing:stop` was missed due to a disconnect). I use `volatile.emit` for typing indicators because they're not critical — if the network is slow and the message is dropped, the indicator will just stop showing when the next one arrives. This avoids overwhelming a slow connection with typing events."

---

## Quick Reference — Day 11 Cheat Sheet

```
Setup:
  npm install @nestjs/websockets @nestjs/platform-socket.io socket.io

Gateway anatomy:
  @WebSocketGateway({ namespace: '/chat', cors: {...} })
  @WebSocketServer() server: Server;    ← inject Socket.io server
  implements OnGatewayInit              → afterInit(server)
  implements OnGatewayConnection        → handleConnection(client)
  implements OnGatewayDisconnect        → handleDisconnect(client)
  @SubscribeMessage('eventName')        → handle client event
  @MessageBody() data                   → event payload (like @Body())
  @ConnectedSocket() client: Socket     → the socket (like @Req())

Return values from @SubscribeMessage:
  void              → no direct response (use server.emit for broadcasting)
  WsResponse<T>     → { event: string, data: T } sent back to client
  Promise<T>        → acknowledgement sent back to client

Broadcasting cheat sheet:
  client.emit()                    → ONLY to this socket
  client.broadcast.emit()          → ALL sockets EXCEPT this one
  server.emit()                    → ALL sockets in namespace
  server.to('room').emit()         → ALL in room (including sender)
  client.to('room').emit()         → ALL in room (EXCLUDING sender)
  server.to('user:5').emit()       → user 5's personal room (all tabs)
  client.volatile.emit()           → skip if not ready (typing indicators)

Rooms:
  client.join('room:123')          → add socket to room
  client.leave('room:123')         → remove socket from room
  server.in('room:123').fetchSockets() → get all sockets in room
  Personal room: client.join('user:' + userId) on connection

Auth in WebSockets:
  Authenticate in afterInit() middleware OR handleConnection()
  token = socket.handshake.auth.token || socket.handshake.headers.authorization
  On failure: client.disconnect(true)
  On success: client.data.user = user  ← persists for connection lifetime
  WsJwtGuard → reads from client.data.user for per-event auth

Scaling (Redis adapter):
  npm install @socket.io/redis-adapter ioredis
  createAdapter(pubClient, subClient)
  app.useWebSocketAdapter(new RedisIoAdapter(app))
  → All servers share event bus via Redis pub/sub

WebSocket vs SSE vs HTTP:
  HTTP polling   → simple, inefficient, high latency
  SSE            → server→client only, HTTP-based, auto-reconnect
  WebSocket      → bidirectional, persistent, true real-time

Inject gateway into HTTP service:
  constructor(private chatGateway: ChatGateway) {}
  this.chatGateway.emitToUser(userId, 'event', data)
  Works because gateways are @Injectable() providers
```

---

*Day 11 complete. Tomorrow — Day 12: Microservices & Message Brokers — Transport layers, TCP & Redis transport, RabbitMQ, Kafka, message patterns, client proxy, and hybrid apps.*
