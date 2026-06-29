# Day 3 — Request Lifecycle & Middleware
### NestJS Interview Prep | 3-Year Backend Engineer Level

> **Goal for today:** Understand EVERY layer a request passes through in NestJS — Middleware, Guards, Interceptors, Pipes, and Exception Filters. These are what make NestJS powerful. A 3-year engineer must be able to implement each one from scratch and explain exactly WHEN and WHY to use each.

---

## 🧭 First-Timer Guidance — Read This Before Starting

If you're using NestJS for the first time, here is the mindset you need:

**Think of an HTTP request like a person entering a building:**

```
Person (HTTP Request) enters a building (your NestJS app)

1. Security at the main gate checks ID      → Middleware
2. Receptionist checks if they're allowed   → Guard
3. Assistant logs their arrival time        → Interceptor (before)
4. Metal detector scans their bag           → Pipe (validation)
5. They meet the person they came to see    → Controller + Service
6. Assistant logs their departure time      → Interceptor (after)
7. If they cause trouble, security removes  → Exception Filter
```

**Why does this layered architecture matter?**

Before NestJS, in plain Express, you'd put ALL of this inside the route handler:

```javascript
// Express — everything mixed together (ugly, unscalable)
app.post('/users', async (req, res) => {
  // Authentication check
  const token = req.headers.authorization;
  if (!token) return res.status(401).json({ error: 'Unauthorized' });
  
  // Validation
  if (!req.body.email) return res.status(400).json({ error: 'Email required' });
  
  // Logging
  console.log(`[${Date.now()}] Creating user`);
  
  // Business logic
  try {
    const user = await db.create(req.body);
    res.status(201).json(user);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});
```

NestJS separates each concern into its own layer. Each route handler becomes clean and readable. The cross-cutting concerns (auth, logging, validation) are written ONCE and applied everywhere.

**How to practice today:**
1. Read each section carefully — understand the WHY
2. Create the files in your NestJS project as you read
3. Test each one with a tool like Postman or Thunder Client (VS Code extension)
4. The goal is NOT to memorize code — it's to understand the pattern

---

## Table of Contents

1. [The Complete Request Lifecycle — Visual Map](#1-the-complete-request-lifecycle--visual-map)
2. [Middleware — The First Gate](#2-middleware--the-first-gate)
3. [Guards — Who Is Allowed In?](#3-guards--who-is-allowed-in)
4. [Interceptors — Wrap Everything](#4-interceptors--wrap-everything)
5. [Pipes — Validate & Transform Data](#5-pipes--validate--transform-data)
6. [Exception Filters — Catch All Errors](#6-exception-filters--catch-all-errors)
7. [Custom Decorators — Clean Up Your Controllers](#7-custom-decorators--clean-up-your-controllers)
8. [Applying Everything — Scope Levels](#8-applying-everything--scope-levels)
9. [Putting It All Together — A Real Auth Flow](#9-putting-it-all-together--a-real-auth-flow)
10. [Interview Questions & Answers](#10-interview-questions--answers)

---

## 1. The Complete Request Lifecycle — Visual Map

```
  Incoming HTTP Request
  POST /api/v1/users  { "name": "Alice", "email": "alice@test.com" }
  │
  ▼
  ┌────────────────────────────────────────────────┐
  │  1. MIDDLEWARE                                  │
  │     - Runs before NestJS routing               │
  │     - Has access to raw req/res objects        │
  │     - Must call next() to proceed              │
  │     - Example: Morgan logger, CORS, body parser│
  └─────────────────────┬──────────────────────────┘
                        │ next()
  ▼
  ┌────────────────────────────────────────────────┐
  │  2. GUARDS                                      │
  │     - Can BLOCK the request (return false)     │
  │     - Authentication: Is this user logged in?  │
  │     - Authorization: Can this user do this?    │
  │     - Returns true (continue) or throws        │
  └─────────────────────┬──────────────────────────┘
                        │ canActivate() = true
  ▼
  ┌────────────────────────────────────────────────┐
  │  3. INTERCEPTORS (before handler)              │
  │     - Wraps the handler execution              │
  │     - Runs code BEFORE handler                 │
  │     - Example: start timer, add request ID     │
  └─────────────────────┬──────────────────────────┘
                        │ next.handle()
  ▼
  ┌────────────────────────────────────────────────┐
  │  4. PIPES                                       │
  │     - Transform: string '1' → number 1         │
  │     - Validate: throw 400 if DTO is invalid    │
  │     - Runs just before the controller method   │
  └─────────────────────┬──────────────────────────┘
                        │ validated & transformed data
  ▼
  ┌────────────────────────────────────────────────┐
  │  5. CONTROLLER METHOD                          │
  │     @Post() create(@Body() dto: CreateUserDto) │
  │     Calls → UsersService.create(dto)           │
  └─────────────────────┬──────────────────────────┘
                        │ returns data or throws
  ▼
  ┌────────────────────────────────────────────────┐
  │  6. INTERCEPTORS (after handler)               │
  │     - Transforms the RESPONSE                  │
  │     - Example: wrap in { data, statusCode }    │
  │     - Log response time                        │
  └─────────────────────┬──────────────────────────┘
                        │ or if error was thrown ↓
  ▼
  ┌────────────────────────────────────────────────┐
  │  7. EXCEPTION FILTERS                          │
  │     - Catches ANY unhandled exception          │
  │     - Formats error into consistent JSON       │
  │     - Example: { statusCode, message, error }  │
  └─────────────────────┬──────────────────────────┘
                        │
  ▼
  Outgoing HTTP Response
  HTTP 201 { "id": 1, "name": "Alice", "email": "alice@test.com" }
```

**The key insight:** Each layer has ONE responsibility. You write them ONCE and apply them globally, per-controller, or per-route. This is what makes NestJS so powerful.

---

## 2. Middleware — The First Gate

### What is Middleware?
Middleware is a function that runs **before the route handler is determined**. It has access to the raw Express `Request` and `Response` objects and a `next()` function.

### Simple explanation
Middleware doesn't know WHICH controller will handle the request — it just processes the raw HTTP request. It's the same as Express middleware because NestJS uses Express underneath.

### When to use Middleware (vs Guards/Interceptors)
```
Use Middleware when:
  ✓ You need to work with raw req/res objects
  ✓ Logging ALL requests (before routing)
  ✓ Adding request IDs
  ✓ Parsing request bodies (body-parser)
  ✓ CORS headers
  ✓ Rate limiting at HTTP level

Don't use Middleware when:
  ✗ You need NestJS execution context (Guards/Interceptors have this)
  ✗ You need to know which controller will handle the request
  ✗ You need access to DI container in a complex way
```

### Creating a functional middleware (simple)

```typescript
// src/common/middleware/logger.middleware.ts

import { Request, Response, NextFunction } from 'express';

// Functional middleware — a plain function, no class needed
// This is the simplest form — use when you don't need DI
export function loggerMiddleware(
  req: Request,
  res: Response,
  next: NextFunction,  // Must call next() or the request hangs forever
): void {
  // Log the incoming request details
  const { method, originalUrl, ip } = req;
  const userAgent = req.get('user-agent') || 'unknown';
  const startTime = Date.now();

  console.log(
    `→ ${method} ${originalUrl} | IP: ${ip} | Agent: ${userAgent}`,
  );

  // Listen for when the response finishes to log response info
  res.on('finish', () => {
    // res.statusCode is available after response is sent
    const responseTime = Date.now() - startTime;
    console.log(
      `← ${method} ${originalUrl} | ${res.statusCode} | ${responseTime}ms`,
    );
  });

  // CRITICAL: Always call next() unless you're ending the request here
  // Not calling next() = request hangs, browser waits forever
  next();
}
```

### Creating a class-based middleware (can use Dependency Injection)

```typescript
// src/common/middleware/request-id.middleware.ts

import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import { v4 as uuidv4 } from 'uuid';  // npm install uuid @types/uuid

// Class middleware — can inject NestJS providers
// Use when you need DI (e.g., inject a service to log to database)
@Injectable()
export class RequestIdMiddleware implements NestMiddleware {
  // You can inject services here
  // constructor(private readonly logService: LogService) {}

  use(req: Request, res: Response, next: NextFunction): void {
    // Generate a unique ID for this request
    // This lets you trace a request across multiple log lines and services
    const requestId = uuidv4();

    // Attach to request object — available in controllers via @Req()
    (req as any).requestId = requestId;

    // Send back in response header — useful for debugging
    // Client can include this ID when reporting bugs
    res.setHeader('X-Request-Id', requestId);

    next();
  }
}
```

### Applying middleware in AppModule

```typescript
// src/app.module.ts

import { Module, NestModule, MiddlewareConsumer, RequestMethod } from '@nestjs/common';
import { loggerMiddleware } from './common/middleware/logger.middleware';
import { RequestIdMiddleware } from './common/middleware/request-id.middleware';

@Module({
  // ...imports, controllers, providers
})
// NestModule interface requires configure() method
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    // ── Apply to ALL routes ───────────────────────────────────────────
    consumer
      .apply(loggerMiddleware)  // Can apply multiple: .apply(mw1, mw2, mw3)
      .forRoutes('*');          // '*' = all routes

    // ── Apply to specific routes ──────────────────────────────────────
    consumer
      .apply(RequestIdMiddleware)
      .forRoutes(
        { path: 'users', method: RequestMethod.GET },
        // Only GET /users — not POST /users, not /products
        { path: 'users/:id', method: RequestMethod.ALL },
        // ALL methods for /users/:id
      );

    // ── Apply to a specific controller ────────────────────────────────
    consumer
      .apply(RequestIdMiddleware)
      .forRoutes(UsersController);
      // Applies to ALL routes defined in UsersController

    // ── Exclude certain routes ────────────────────────────────────────
    consumer
      .apply(loggerMiddleware)
      .exclude(
        { path: 'health', method: RequestMethod.GET },
        // Don't log health check requests (too noisy)
      )
      .forRoutes('*');
  }
}
```

### Global middleware (applied in main.ts)

```typescript
// src/main.ts

import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import * as helmet from 'helmet';
import * as compression from 'compression';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Third-party Express middleware — applied globally in main.ts
  // These are npm packages, not NestJS-specific

  app.use(helmet());
  // Helmet adds security-related HTTP headers:
  // - X-XSS-Protection: Prevents cross-site scripting
  // - X-Frame-Options: Prevents clickjacking
  // - X-Content-Type-Options: Prevents MIME sniffing
  // npm install helmet

  app.use(compression());
  // Compresses HTTP responses with gzip/deflate
  // Reduces response size by 70-80% for JSON responses
  // npm install compression @types/compression

  await app.listen(3000);
}
bootstrap();
```

---

## 3. Guards — Who Is Allowed In?

### What is a Guard?
A Guard is a class that decides whether a request should be **allowed to proceed** to the controller. It's the security checkpoint.

### Simple explanation
A Guard answers ONE question: **"Can this user do this?"**
- If yes: `return true` → request continues
- If no: `throw UnauthorizedException()` or `return false` → request blocked with 403

### The difference between Middleware and Guards

```
Middleware:
  - Runs before routing
  - No NestJS context (doesn't know which handler will run)
  - Used for raw HTTP processing
  - Cannot easily access route metadata/decorators

Guard:
  - Runs AFTER middleware, BEFORE the handler
  - Has ExecutionContext (knows which controller/method will handle request)
  - Can read custom metadata from decorators (e.g., @Roles('admin'))
  - Perfect for auth and authorization
```

### Creating an Auth Guard from scratch

```typescript
// src/common/guards/jwt-auth.guard.ts

import {
  Injectable,
  CanActivate,
  ExecutionContext,
  UnauthorizedException,
} from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { Request } from 'express';

// CanActivate interface requires ONE method: canActivate()
@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(private readonly jwtService: JwtService) {}
  // Guards CAN use dependency injection — here we inject JwtService

  canActivate(
    context: ExecutionContext,
    // ExecutionContext is NestJS's abstraction for the current request
    // Works for HTTP, WebSockets, and Microservices — same interface
  ): boolean | Promise<boolean> {
    // Switch to HTTP context to get the Express Request object
    const request = context.switchToHttp().getRequest<Request>();

    // Extract the JWT from the Authorization header
    // Standard format: "Authorization: Bearer <token>"
    const token = this.extractTokenFromHeader(request);

    if (!token) {
      // Throwing here is cleaner than returning false
      // false → 403 Forbidden, throw UnauthorizedException → 401 Unauthorized
      throw new UnauthorizedException('No authentication token provided');
    }

    try {
      // Verify and decode the JWT
      // jwtService.verify() throws if token is invalid or expired
      const payload = this.jwtService.verify(token);

      // Attach decoded payload to request — accessible in controllers
      // Controllers can use @Req() to access request.user
      // Or use a custom @CurrentUser() decorator (shown in section 7)
      request['user'] = payload;

      return true;  // Allow the request to proceed

    } catch (error) {
      // Token is invalid, expired, or tampered with
      throw new UnauthorizedException('Invalid or expired token');
    }
  }

  // Helper method — extracts Bearer token from Authorization header
  private extractTokenFromHeader(request: Request): string | null {
    const authHeader = request.headers.authorization;
    // authHeader: "Bearer eyJhbGciOiJIUzI1NiJ9.eyJ1c2VySWQiOjF9.abc"

    if (!authHeader) return null;

    const [type, token] = authHeader.split(' ');
    // type: "Bearer", token: "eyJhbGciOiJIUzI1NiJ9..."

    // Only accept Bearer tokens, not Basic or other schemes
    return type === 'Bearer' ? token : null;
  }
}
```

### Creating a Role-Based Access Control (RBAC) Guard

```typescript
// src/common/guards/roles.guard.ts

import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
// Reflector is NestJS's utility to READ metadata set by decorators

// Step 1: Create the @Roles() decorator that sets metadata
// src/common/decorators/roles.decorator.ts
import { SetMetadata } from '@nestjs/common';
export const ROLES_KEY = 'roles';  // Key to store/retrieve roles metadata

// @Roles('admin', 'moderator') sets metadata: { roles: ['admin', 'moderator'] }
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);

// Step 2: Create the guard that READS that metadata
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}
  // Reflector is a special NestJS service for reading decorator metadata

  canActivate(context: ExecutionContext): boolean {
    // Read the roles required by the route handler or controller class
    // getAllAndOverride: checks method first, then class — method wins
    const requiredRoles = this.reflector.getAllAndOverride<string[]>(
      ROLES_KEY,
      [
        context.getHandler(),  // The specific method being called
        context.getClass(),    // The controller class
      ],
    );

    // If no @Roles() decorator used, allow everyone through
    // This means routes without @Roles() are accessible to all authenticated users
    if (!requiredRoles || requiredRoles.length === 0) {
      return true;
    }

    // Get the user from request (set by JwtAuthGuard)
    const { user } = context.switchToHttp().getRequest();

    // Check if the user's role is in the required roles list
    return requiredRoles.some((role) => user?.role === role);
    // Returns true if user has at least one of the required roles
    // Returns false → NestJS throws 403 Forbidden automatically
  }
}
```

### Using Guards — applying them at different levels

```typescript
// src/users/users.controller.ts

import { Controller, Get, Post, Delete, Param, Body, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from '../common/guards/jwt-auth.guard';
import { RolesGuard } from '../common/guards/roles.guard';
import { Roles } from '../common/decorators/roles.decorator';

// ── Apply guard to ENTIRE controller ─────────────────────────────────
@Controller('users')
@UseGuards(JwtAuthGuard)  // ALL routes in this controller require auth
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  // ── Route with no guard override → inherits controller guard ────────
  @Get()
  findAll() {
    // Protected by JwtAuthGuard from controller level
    return this.usersService.findAll();
  }

  // ── Route with additional guard ──────────────────────────────────────
  @Delete(':id')
  @UseGuards(RolesGuard)   // Both JwtAuthGuard AND RolesGuard run
  @Roles('admin')          // Only admins can delete users
  remove(@Param('id') id: string) {
    return this.usersService.remove(+id);
  }

  // ── Public route — override controller-level guard ───────────────────
  // Problem: the controller has @UseGuards(JwtAuthGuard)
  // But /users/register should be public (no token needed to register!)
  // Solution: custom @Public() decorator + guard that checks for it
}

// PATTERN: Making routes public when controller has global auth guard

// Step 1: Create @Public() decorator
import { SetMetadata } from '@nestjs/common';
export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);

// Step 2: Modify JwtAuthGuard to skip if route is marked @Public()
@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(
    private readonly jwtService: JwtService,
    private readonly reflector: Reflector,  // Add Reflector
  ) {}

  canActivate(context: ExecutionContext): boolean {
    // Check if the route is marked as public
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);

    // Skip auth check for public routes
    if (isPublic) return true;

    // ... rest of auth logic from before
    const request = context.switchToHttp().getRequest();
    const token = this.extractTokenFromHeader(request);
    // ...
  }
}

// Step 3: Use @Public() on routes that should be accessible without auth
@Controller('users')
@UseGuards(JwtAuthGuard)  // Controller-level auth
export class UsersController {
  @Post('register')
  @Public()  // ← Skips the JwtAuthGuard check
  register(@Body() dto: CreateUserDto) {
    return this.usersService.create(dto);
  }
}
```

### Applying Guards Globally

```typescript
// src/main.ts

// Option 1: In main.ts (cannot use DI easily)
app.useGlobalGuards(new JwtAuthGuard());
// Problem: new JwtAuthGuard() — you're creating it manually
// Cannot inject dependencies this way!

// Option 2: In AppModule (RECOMMENDED — works with DI)
// src/app.module.ts
import { APP_GUARD } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_GUARD,  // Special NestJS token for global guards
      useClass: JwtAuthGuard,
      // NestJS creates this guard using DI — can inject services!
    },
    {
      provide: APP_GUARD,
      useClass: RolesGuard,
      // Multiple APP_GUARD entries = multiple global guards
      // They run in order: JwtAuthGuard first, then RolesGuard
    },
  ],
})
export class AppModule {}

// Now EVERY route requires authentication
// Use @Public() to mark routes that should be accessible without auth
```

---

## 4. Interceptors — Wrap Everything

### What is an Interceptor?
An interceptor **wraps around the handler** — it runs code BEFORE the handler and also AFTER the handler. It uses RxJS Observables to do this.

### Simple explanation
An interceptor is like a sandwich: your handler execution is the filling, the interceptor provides the bread on both sides.

```
Interceptor (before) → Handler → Interceptor (after)
        ↕                              ↕
   Start timer                    Log duration
   Add request ID                 Transform response
   Cache check                    Format error
```

### Why RxJS Observables?

This is a common confusion for beginners. NestJS uses RxJS here because Observables can:
- Wrap async operations that may or may not complete
- Transform streams of values (important for streaming responses)
- Add timeout, retry, error handling to any async operation

You don't need to be an RxJS expert. You just need to know: `next.handle()` returns an Observable, and you use `.pipe()` to transform it.

```typescript
// src/common/interceptors/logging.interceptor.ts

import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  Logger,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap, map, catchError } from 'rxjs/operators';

// ── Interceptor 1: Performance Logger ─────────────────────────────────
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  private readonly logger = new Logger(LoggingInterceptor.name);
  // NestJS built-in Logger — shows colored output in console

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    // ── BEFORE the handler ──────────────────────────────────────────
    const request = context.switchToHttp().getRequest();
    const { method, url } = request;
    const startTime = Date.now();

    this.logger.log(`→ ${method} ${url}`);

    // next.handle() calls the route handler and returns an Observable
    // Everything BEFORE this line is "before" logic
    // Everything in .pipe() is "after" logic
    return next.handle().pipe(
      // ── AFTER the handler successfully returns ────────────────────
      tap((data) => {
        // tap: "look at" the value without changing it
        const duration = Date.now() - startTime;
        this.logger.log(`← ${method} ${url} | ${duration}ms`);
      }),
    );
  }
}
```

```typescript
// src/common/interceptors/transform.interceptor.ts

// ── Interceptor 2: Response Transformation ─────────────────────────────
// Problem: Your services return raw data. But the client expects a standard envelope:
// { "data": {...}, "statusCode": 200, "message": "success", "timestamp": "..." }
// You could add this in every controller — OR do it once in an interceptor

interface StandardResponse<T> {
  data: T;
  statusCode: number;
  message: string;
  timestamp: string;
}

@Injectable()
export class TransformInterceptor<T>
  implements NestInterceptor<T, StandardResponse<T>>
{
  intercept(
    context: ExecutionContext,
    next: CallHandler,
  ): Observable<StandardResponse<T>> {
    const response = context.switchToHttp().getResponse();

    return next.handle().pipe(
      // map: transforms the value the Observable emits
      // Whatever your service returned is in 'data' parameter
      map((data) => ({
        data,                                       // The actual response data
        statusCode: response.statusCode,            // HTTP status code (200, 201, etc.)
        message: 'success',
        timestamp: new Date().toISOString(),
      })),
    );
  }
}

// Result: every response is now wrapped:
// GET /users returns:
// {
//   "data": [{ "id": 1, "name": "Alice" }],
//   "statusCode": 200,
//   "message": "success",
//   "timestamp": "2024-01-01T10:00:00.000Z"
// }
```

```typescript
// src/common/interceptors/cache.interceptor.ts

// ── Interceptor 3: Simple In-Memory Cache ──────────────────────────────
// For GET requests, return cached response if available
// Real apps use Redis (covered on Day 14) — this shows the pattern

import { of } from 'rxjs';  // 'of' creates an Observable that emits a value immediately

@Injectable()
export class CacheInterceptor implements NestInterceptor {
  private readonly cache = new Map<string, any>();
  // In-memory cache — cleared on app restart
  // Production: use Redis via @nestjs/cache-manager

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();

    // Only cache GET requests
    if (request.method !== 'GET') {
      return next.handle();  // Skip caching for mutations
    }

    const cacheKey = request.url;  // Use URL as cache key

    // Check if we have a cached response
    if (this.cache.has(cacheKey)) {
      // Return cached value IMMEDIATELY — handler never runs
      return of(this.cache.get(cacheKey));
      // 'of(value)' creates an Observable that immediately emits 'value'
    }

    // No cache — call the handler and cache the result
    return next.handle().pipe(
      tap((response) => {
        this.cache.set(cacheKey, response);
        // Simple: cache indefinitely. Real: set TTL with setTimeout
      }),
    );
  }
}
```

```typescript
// src/common/interceptors/timeout.interceptor.ts

// ── Interceptor 4: Request Timeout ────────────────────────────────────
// If a request takes too long, cancel it and return 408

import { timeout, catchError } from 'rxjs/operators';
import { TimeoutError, throwError } from 'rxjs';
import { RequestTimeoutException } from '@nestjs/common';

@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      timeout(5000),  // Cancel if handler doesn't complete in 5 seconds
      catchError((err) => {
        if (err instanceof TimeoutError) {
          // Convert RxJS TimeoutError to NestJS HTTP exception
          return throwError(() => new RequestTimeoutException());
        }
        // Other errors — rethrow as-is, let exception filter handle them
        return throwError(() => err);
      }),
    );
  }
}
```

### Applying Interceptors

```typescript
// ── On a single route ──────────────────────────────────────────────────
@Get()
@UseInterceptors(LoggingInterceptor)
findAll() { return []; }

// ── On entire controller ───────────────────────────────────────────────
@Controller('users')
@UseInterceptors(LoggingInterceptor, TransformInterceptor)
export class UsersController {}

// ── Globally (in AppModule — RECOMMENDED for transform + logging) ──────
@Module({
  providers: [
    { provide: APP_INTERCEPTOR, useClass: LoggingInterceptor },
    { provide: APP_INTERCEPTOR, useClass: TransformInterceptor },
    { provide: APP_INTERCEPTOR, useClass: TimeoutInterceptor },
    // Run in order: Logging → Transform → Timeout (innermost first on return)
  ],
})
export class AppModule {}
```

---

## 5. Pipes — Validate & Transform Data

### What is a Pipe?
A Pipe is a class that processes **incoming data** just before it reaches the controller method. It does two things:
1. **Transform**: convert data from one type to another (string → number, string → Date)
2. **Validate**: check if data is valid, throw 400 if not

### Simple explanation
A Pipe is the "security scanner" that checks the bag (request body/params) before letting it through.

### Built-in Pipes NestJS provides

```typescript
import {
  ParseIntPipe,         // '123' → 123, throws if not a number
  ParseFloatPipe,       // '1.5' → 1.5
  ParseBoolPipe,        // 'true' → true, 'false' → false
  ParseArrayPipe,       // '[1,2,3]' or '1,2,3' → [1, 2, 3]
  ParseUUIDPipe,        // Validates UUID format
  ParseEnumPipe,        // Validates value is in enum
  DefaultValuePipe,     // Provides default value if undefined
  ValidationPipe,       // Validates class with class-validator decorators
} from '@nestjs/common';

// Using built-in pipes:
@Get(':id')
findOne(
  @Param('id', ParseIntPipe) id: number,
  // ParseIntPipe: '5' → 5, '/users/abc' → 400 Bad Request
) {}

@Get()
findAll(
  @Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number,
  // DefaultValuePipe(1): if ?page is not provided, default to 1
  // ParseIntPipe: then convert string '1' to number 1
) {}

@Get(':status')
findByStatus(
  @Param('status', new ParseEnumPipe(UserStatus)) status: UserStatus,
  // Only allows values in the UserStatus enum
  // 'active' → UserStatus.ACTIVE, 'invalid' → 400 Bad Request
) {}
```

### ValidationPipe — the most important pipe

```typescript
// The ValidationPipe is the heart of NestJS validation
// It uses class-validator and class-transformer under the hood

// Setup (npm install class-validator class-transformer)

// Step 1: Create a DTO class with validation decorators
// src/users/dto/create-user.dto.ts

import {
  IsString,
  IsEmail,
  IsNotEmpty,
  MinLength,
  MaxLength,
  IsOptional,
  IsEnum,
  IsInt,
  Min,
  Max,
  IsArray,
  ArrayNotEmpty,
  Matches,
  IsUrl,
  IsDateString,
} from 'class-validator';

export enum UserRole {
  USER = 'user',
  ADMIN = 'admin',
  MODERATOR = 'moderator',
}

export class CreateUserDto {
  // ── String validations ──────────────────────────────────────────────
  @IsNotEmpty({ message: 'Name cannot be empty' })  // Custom error message
  @IsString({ message: 'Name must be a string' })
  @MinLength(2, { message: 'Name must be at least 2 characters' })
  @MaxLength(100)
  name: string;

  // ── Email validation ────────────────────────────────────────────────
  @IsEmail({}, { message: 'Please provide a valid email address' })
  email: string;

  // ── Password with regex ─────────────────────────────────────────────
  @IsString()
  @MinLength(8)
  @Matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/, {
    message: 'Password must contain uppercase, lowercase, and a number',
  })
  password: string;

  // ── Enum validation ─────────────────────────────────────────────────
  @IsEnum(UserRole, { message: 'Role must be user, admin, or moderator' })
  @IsOptional()  // Optional field — skip validation if not provided
  role?: UserRole;

  // ── Number validation ───────────────────────────────────────────────
  @IsInt()
  @Min(18, { message: 'Must be at least 18 years old' })
  @Max(120)
  @IsOptional()
  age?: number;

  // ── Array validation ────────────────────────────────────────────────
  @IsArray()
  @ArrayNotEmpty()
  @IsString({ each: true })  // each: true → validate each array element
  @IsOptional()
  tags?: string[];

  // ── URL validation ───────────────────────────────────────────────────
  @IsUrl({}, { message: 'Must be a valid URL' })
  @IsOptional()
  profilePicture?: string;
}

// Step 2: Configure ValidationPipe globally in main.ts
// (We covered this on Day 2, but here's the detailed version)

app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
    // CRITICAL: Strips properties NOT in the DTO
    // If client sends { name, email, password, isAdmin: true }
    // isAdmin is NOT in the DTO → automatically removed
    // Prevents property pollution attacks

    forbidNonWhitelisted: true,
    // Instead of silently stripping, THROW 400 Bad Request
    // Good for development — alerts API consumers of mistakes
    // Consider turning off in production for looser behavior

    transform: true,
    // Automatically transforms incoming data types
    // '5' → 5 for @IsInt() fields
    // 'true' → true for @IsBoolean() fields
    // Plain objects → DTO class instances (needed for class-validator)

    transformOptions: {
      enableImplicitConversion: true,
      // Even more aggressive type conversion
      // Query string '5' → number automatically based on TypeScript type
    },

    disableErrorMessages: process.env.NODE_ENV === 'production',
    // In production, don't leak internal field names in error messages
    // Development: detailed errors | Production: generic "Validation failed"
  }),
);
```

### Creating a custom Pipe

```typescript
// src/common/pipes/trim-strings.pipe.ts
// Problem: Users submit forms with accidental whitespace
// "  alice@test.com  " should become "alice@test.com"

import { Injectable, PipeTransform, ArgumentMetadata } from '@nestjs/common';

@Injectable()
export class TrimStringsPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata): any {
    // metadata.type: 'body' | 'query' | 'param' | 'custom'
    // metadata.metatype: the TypeScript class (CreateUserDto)
    // metadata.data: the @Body('name') field name if specified

    // Only process body data (objects)
    if (metadata.type !== 'body') return value;

    // Recursively trim all string values
    return this.trimObject(value);
  }

  private trimObject(obj: any): any {
    if (typeof obj === 'string') {
      return obj.trim();  // Trim the string
    }

    if (Array.isArray(obj)) {
      return obj.map((item) => this.trimObject(item));
    }

    if (obj !== null && typeof obj === 'object') {
      return Object.keys(obj).reduce((acc, key) => {
        acc[key] = this.trimObject(obj[key]);
        return acc;
      }, {} as any);
    }

    return obj;  // Numbers, booleans, null, undefined — return as-is
  }
}

// Usage:
app.useGlobalPipes(new TrimStringsPipe(), new ValidationPipe({...}));
// Order matters: Trim first, THEN validate the trimmed values
```

```typescript
// src/common/pipes/parse-sort.pipe.ts
// Custom pipe: converts '?sort=name:asc,email:desc' → [{ field: 'name', order: 'ASC' }]

import { PipeTransform, Injectable, BadRequestException } from '@nestjs/common';

interface SortOption {
  field: string;
  order: 'ASC' | 'DESC';
}

@Injectable()
export class ParseSortPipe implements PipeTransform<string, SortOption[]> {
  private readonly allowedFields = ['name', 'email', 'createdAt', 'updatedAt'];

  transform(value: string | undefined): SortOption[] {
    if (!value) return [{ field: 'createdAt', order: 'DESC' }];  // Default sort

    return value.split(',').map((part) => {
      const [field, order = 'asc'] = part.trim().split(':');

      // Validate field to prevent SQL injection
      if (!this.allowedFields.includes(field)) {
        throw new BadRequestException(
          `Invalid sort field: ${field}. Allowed: ${this.allowedFields.join(', ')}`,
        );
      }

      const orderUpper = order.toUpperCase();
      if (orderUpper !== 'ASC' && orderUpper !== 'DESC') {
        throw new BadRequestException(`Sort order must be 'asc' or 'desc'`);
      }

      return { field, order: orderUpper as 'ASC' | 'DESC' };
    });
  }
}

// Usage in controller:
@Get()
findAll(@Query('sort', ParseSortPipe) sort: SortOption[]) {
  // GET /users?sort=name:asc,createdAt:desc
  // sort = [{ field: 'name', order: 'ASC' }, { field: 'createdAt', order: 'DESC' }]
  return this.usersService.findAll({ sort });
}
```

---

## 6. Exception Filters — Catch All Errors

### What is an Exception Filter?
An Exception Filter catches **exceptions thrown anywhere** in the request pipeline and converts them into HTTP responses.

### Simple explanation
Without exception filters, if your code throws an unhandled error, Node.js crashes or sends an ugly error page. Exception filters catch EVERYTHING and format it into a clean JSON response.

### NestJS's default exception handling

```
Any exception thrown in your app → NestJS Global Exception Filter
  ↓
If it's an HttpException (NotFoundException, UnauthorizedException etc.)
  → Returns { statusCode, message, error } with correct HTTP status

If it's a generic Error (unexpected runtime error)
  → Returns { statusCode: 500, message: "Internal server error" }
  (NestJS hides the actual error in production)
```

### Creating a custom exception filter

```typescript
// src/common/filters/http-exception.filter.ts
// Replace NestJS's default error response with YOUR format

import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
  Logger,
} from '@nestjs/common';
import { Request, Response } from 'express';

// @Catch(HttpException) means: catch ONLY HttpExceptions
// @Catch() with no args means: catch EVERYTHING (all exceptions)
@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(HttpExceptionFilter.name);

  catch(exception: HttpException, host: ArgumentsHost): void {
    // ArgumentsHost is similar to ExecutionContext
    // Switch to HTTP context to get request and response
    const ctx = host.switchToHttp();
    const request = ctx.getRequest<Request>();
    const response = ctx.getResponse<Response>();

    // Get the HTTP status code from the exception
    const statusCode = exception.getStatus();
    // NotFoundException → 404, BadRequestException → 400, etc.

    // Get the exception response — can be string or object
    const exceptionResponse = exception.getResponse();
    // BadRequestException from ValidationPipe returns an object:
    // { message: ['email must be an email', 'name is too short'], error: 'Bad Request', statusCode: 400 }

    // Build a consistent error response structure
    const errorResponse = {
      success: false,
      statusCode,
      timestamp: new Date().toISOString(),
      path: request.url,           // Which endpoint caused the error
      method: request.method,      // GET, POST, etc.
      message:
        typeof exceptionResponse === 'string'
          ? exceptionResponse
          : (exceptionResponse as any).message,
          // Array of messages from ValidationPipe, or single message
      error:
        typeof exceptionResponse === 'object'
          ? (exceptionResponse as any).error
          : 'Error',
    };

    // Log the error (important for debugging production issues)
    this.logger.error(
      `${request.method} ${request.url} → ${statusCode}`,
      JSON.stringify(errorResponse),
    );

    // Send the formatted response
    response.status(statusCode).json(errorResponse);
  }
}

// Result for a 404 error:
// {
//   "success": false,
//   "statusCode": 404,
//   "timestamp": "2024-01-01T10:00:00.000Z",
//   "path": "/api/v1/users/999",
//   "method": "GET",
//   "message": "User with ID 999 not found",
//   "error": "Not Found"
// }
```

### Catching ALL exceptions (including unexpected errors)

```typescript
// src/common/filters/all-exceptions.filter.ts
// Catches HttpExceptions AND any other unexpected errors

import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
  Logger,
} from '@nestjs/common';

@Catch()  // No argument = catches EVERYTHING
export class AllExceptionsFilter implements ExceptionFilter {
  private readonly logger = new Logger(AllExceptionsFilter.name);

  catch(exception: unknown, host: ArgumentsHost): void {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    const request = ctx.getRequest();

    // Determine if this is a known HttpException or an unexpected error
    const isHttpException = exception instanceof HttpException;

    const statusCode = isHttpException
      ? exception.getStatus()
      : HttpStatus.INTERNAL_SERVER_ERROR;  // 500 for unexpected errors

    const message = isHttpException
      ? exception.getResponse()
      : 'An internal server error occurred';
      // NEVER expose raw error messages to clients in production
      // Raw errors may contain DB schema, file paths, env variables

    // Log the FULL error for internal debugging
    if (!isHttpException) {
      // Unexpected error — log the full stack trace
      this.logger.error(
        `Unexpected error on ${request.method} ${request.url}`,
        exception instanceof Error ? exception.stack : String(exception),
      );

      // In production: send to error tracking (Sentry, Datadog, etc.)
      // if (process.env.NODE_ENV === 'production') {
      //   Sentry.captureException(exception);
      // }
    }

    response.status(statusCode).json({
      success: false,
      statusCode,
      timestamp: new Date().toISOString(),
      path: request.url,
      message:
        process.env.NODE_ENV === 'production' && !isHttpException
          ? 'Internal server error'  // Hide details in production
          : message,                 // Show details in development
    });
  }
}
```

### Creating custom exceptions

```typescript
// src/common/exceptions/business.exceptions.ts
// Custom exceptions for business logic errors

import { HttpException, HttpStatus } from '@nestjs/common';

// Custom exception for business rules
export class InsufficientFundsException extends HttpException {
  constructor(required: number, available: number) {
    super(
      {
        statusCode: HttpStatus.UNPROCESSABLE_ENTITY,  // 422
        message: `Insufficient funds: required ${required}, available ${available}`,
        error: 'Insufficient Funds',
        // You can add any extra fields here
        details: { required, available },
      },
      HttpStatus.UNPROCESSABLE_ENTITY,
    );
  }
}

// Custom exception for external service failures
export class ExternalServiceException extends HttpException {
  constructor(serviceName: string, reason: string) {
    super(
      {
        statusCode: HttpStatus.BAD_GATEWAY,  // 502
        message: `External service '${serviceName}' failed: ${reason}`,
        error: 'External Service Error',
      },
      HttpStatus.BAD_GATEWAY,
    );
  }
}

// Usage in service:
async processPayment(userId: number, amount: number) {
  const user = await this.findOne(userId);

  if (user.balance < amount) {
    throw new InsufficientFundsException(amount, user.balance);
    // Returns:
    // { statusCode: 422, message: "Insufficient funds: required 100, available 50", details: {...} }
  }

  // process payment...
}
```

### Applying Exception Filters

```typescript
// ── On a route ────────────────────────────────────────────────────────
@Get(':id')
@UseFilters(HttpExceptionFilter)
findOne(@Param('id') id: string) {}

// ── On a controller ────────────────────────────────────────────────────
@Controller('users')
@UseFilters(AllExceptionsFilter)
export class UsersController {}

// ── Globally (RECOMMENDED) ─────────────────────────────────────────────
// In main.ts:
app.useGlobalFilters(new AllExceptionsFilter());

// OR in AppModule (with DI support):
@Module({
  providers: [
    { provide: APP_FILTER, useClass: AllExceptionsFilter },
  ],
})
export class AppModule {}
```

---

## 7. Custom Decorators — Clean Up Your Controllers

### What are custom decorators?
They let you create your OWN decorators to extract data from requests or apply combinations of decorators.

```typescript
// src/common/decorators/current-user.decorator.ts
// Problem: In every controller that needs the current user, you write:
//   @Req() req: Request
//   const user = req.user;  ← messy and repetitive

// Solution: @CurrentUser() decorator that does it automatically

import { createParamDecorator, ExecutionContext } from '@nestjs/common';
import { User } from '../../users/entities/user.entity';

export const CurrentUser = createParamDecorator(
  // data: value passed INTO the decorator, e.g., @CurrentUser('email') → data = 'email'
  // ctx: ExecutionContext — same as in guards and interceptors
  (data: keyof User | undefined, ctx: ExecutionContext): User | any => {
    const request = ctx.switchToHttp().getRequest();
    const user: User = request.user;  // Set by JwtAuthGuard

    // If a field name is passed, return just that field
    // @CurrentUser('email') → returns just the email string
    // @CurrentUser() → returns the whole user object
    return data ? user?.[data] : user;
  },
);

// Usage in controllers:
@Get('profile')
@UseGuards(JwtAuthGuard)
getProfile(@CurrentUser() user: User) {
  // No more @Req() + request.user — clean and readable
  return user;
}

@Get('my-email')
@UseGuards(JwtAuthGuard)
getEmail(@CurrentUser('email') email: string) {
  return { email };
}
```

```typescript
// src/common/decorators/api-paginated.decorator.ts
// Composite decorator — applies MULTIPLE decorators at once

import { applyDecorators, Get, UseGuards, UseInterceptors } from '@nestjs/common';
import { JwtAuthGuard } from '../guards/jwt-auth.guard';
import { TransformInterceptor } from '../interceptors/transform.interceptor';

// Convenience decorator for protected GET endpoints
// Instead of writing @Get() @UseGuards(JwtAuthGuard) @UseInterceptors(...) every time
export function AuthGet(path?: string) {
  return applyDecorators(
    Get(path),
    UseGuards(JwtAuthGuard),
    UseInterceptors(TransformInterceptor),
  );
}

// Usage:
@Controller('users')
export class UsersController {
  @AuthGet()          // Replaces @Get() + @UseGuards(JwtAuthGuard) + @UseInterceptors(...)
  findAll() {}

  @AuthGet(':id')     // Same, with a route param
  findOne(@Param('id', ParseIntPipe) id: number) {}
}
```

---

## 8. Applying Everything — Scope Levels

Understanding WHERE to apply each feature is critical.

```
               Global         Controller       Route
               ──────         ──────────       ─────
Middleware     app.use()      configure()      configure()
               NestModule     + forRoutes      + method

Guard          APP_GUARD      @UseGuards()     @UseGuards()
               in AppModule   on class         on method

Interceptor    APP_INTERCEPTOR @UseInterceptors @UseInterceptors
               in AppModule   on class         on method

Pipe           APP_PIPE       @UsePipes()      @UsePipes()
               in AppModule   on class         on method
                                               or @Param('x', Pipe)

Filter         APP_FILTER     @UseFilters()    @UseFilters()
               in AppModule   on class         on method
```

```typescript
// Priority: Method-level overrides Controller-level overrides Global
// More specific = higher priority

// Global guard (applied to everything)
{ provide: APP_GUARD, useClass: JwtAuthGuard }

// Controller guard (applied to this controller)
@UseGuards(RolesGuard)
class AdminController {}

// Method guard (applied to just this method)
@UseGuards(OwnershipGuard)
@Patch(':id')
update() {}

// Final result for update():
// Runs: JwtAuthGuard → RolesGuard → OwnershipGuard (all three)
// Each must pass for the request to reach the handler
```

---

## 9. Putting It All Together — A Real Auth Flow

Let's trace a real request through the entire pipeline:

```
Request: POST /api/v1/posts
Headers: Authorization: Bearer <jwt_token>
Body: { "title": "My Post", "content": "Hello world" }

─────────────────────────────────────────────────────────────────

1. MIDDLEWARE (LoggingInterceptor + RequestIdMiddleware)
   → Adds X-Request-Id: "abc-123" header
   → Logs: "→ POST /api/v1/posts"
   → Calls next()

2. GUARD (JwtAuthGuard — applied globally)
   → Extracts token from "Authorization: Bearer <jwt>"
   → Calls jwtService.verify(token)
   → Decodes: { userId: 1, email: 'alice@test.com', role: 'user' }
   → Attaches to request.user
   → Returns true → continues

3. GUARD (RolesGuard — applied to controller)
   → Reads @Roles('user', 'admin') from method metadata
   → Checks: request.user.role ('user') is in ['user', 'admin']
   → Returns true → continues

4. INTERCEPTOR — before (LoggingInterceptor)
   → Records startTime = Date.now()
   → Calls next.handle()

5. PIPE (ValidationPipe — global)
   → Validates CreatePostDto:
     - title: 'My Post' ✅ (IsString, MinLength 3)
     - content: 'Hello world' ✅ (IsString, MinLength 10)
   → Transforms: plain object → CreatePostDto instance
   → Passes validated DTO to controller

6. CONTROLLER (PostsController.create)
   → Calls postsService.create(dto, request.user)
   → Service creates post in database
   → Returns created Post entity

7. INTERCEPTOR — after (TransformInterceptor)
   → Wraps response:
     { data: { id: 1, title: 'My Post' }, statusCode: 201, message: 'success' }
   → LoggingInterceptor records: "← POST /api/v1/posts | 45ms"

8. RESPONSE: HTTP 201
   {
     "data": { "id": 1, "title": "My Post", "content": "Hello world" },
     "statusCode": 201,
     "message": "success",
     "timestamp": "2024-01-01T10:00:00.000Z"
   }

─────────────────────────────────────────────────────────────────

If Step 5 (validation) FAILED:
→ ValidationPipe throws BadRequestException
→ Skips controller and service
→ EXCEPTION FILTER catches it
→ Returns HTTP 400:
   {
     "success": false,
     "statusCode": 400,
     "message": ["title must be longer than 2 characters"],
     "error": "Bad Request"
   }
```

---

## 10. Interview Questions & Answers

**Q1: What is the difference between Middleware and Guards in NestJS?**

> "Both run before the route handler, but they serve different purposes and have different capabilities. **Middleware** is Express-level — it runs before NestJS even knows which controller will handle the request. It has no access to NestJS's execution context or route metadata. It's good for raw HTTP processing like adding headers, logging, and body parsing. **Guards** run after middleware, have the full NestJS execution context, and can read route metadata via the Reflector. Guards are specifically designed for authentication and authorization — they answer 'can this user access this resource?' and have access to the current route handler's decorators like `@Roles()`. Use middleware for infrastructure concerns, guards for security decisions."

---

**Q2: What is an ExecutionContext and why does NestJS need it?**

> "ExecutionContext is an abstraction that represents the current request, regardless of the underlying transport protocol. NestJS supports HTTP (Express/Fastify), WebSockets, and microservices (TCP, gRPC, RabbitMQ). An ExecutionContext wraps all of these in a unified interface. You call `context.switchToHttp()` for Express requests, `context.switchToWs()` for WebSocket messages, or `context.switchToRpc()` for microservice messages. This lets you write Guards and Interceptors that work across all transport types. `context.getHandler()` returns the actual method being called, and `context.getClass()` returns the controller class — both needed by the Reflector to read decorator metadata."

---

**Q3: What is the difference between Interceptors and Middleware?**

> "Middleware runs at the Express level and has no concept of NestJS handlers or response transformation. Interceptors run at the NestJS level and have the full execution context. Critically, Interceptors can transform the RESPONSE from a route handler using RxJS `.pipe(map())` — middleware cannot do this. Interceptors also run both before AND after the handler, while middleware only runs before. Interceptors use Observables, which allows powerful patterns like timeout (`timeout(5000)`), retry, and caching (`of(cachedValue)`). Use interceptors for: response transformation, logging with response times, caching, and timeouts."

---

**Q4: What is the order of execution in the NestJS pipeline and why does it matter?**

> "The order is: Middleware → Guards → Interceptors (before) → Pipes → Controller → Interceptors (after) → Exception Filters. The order matters for several reasons. Guards must run before pipes because there's no point validating a body if the user isn't authorized. Interceptors wrap the handler so their 'after' code runs after the response, allowing response transformation. Exception filters run last to catch anything that went wrong. If you put validation logic in middleware instead of pipes, it runs before route matching, and you won't have access to the DTO type information needed for class-validator."

---

**Q5: How do you make some routes public when using a global auth guard?**

> "The pattern is to create a `@Public()` decorator using `SetMetadata(IS_PUBLIC_KEY, true)`, then modify the global auth guard to check for this metadata using Reflector. At the start of `canActivate()`, you call `this.reflector.getAllAndOverride(IS_PUBLIC_KEY, [context.getHandler(), context.getClass()])`. If it returns `true`, you return `true` immediately, bypassing authentication. The guard is registered globally via `APP_GUARD` in the module providers so it has access to DI and can inject Reflector. This is better than applying `@UseGuards(JwtAuthGuard)` to every protected route because auth is the default and you only explicitly opt out."

---

**Q6: Why do Interceptors use RxJS Observables?**

> "NestJS route handlers can return either a plain value, a Promise, or an Observable. By having interceptors return Observables, NestJS creates a unified pipeline that can handle all three cases. The real power comes from RxJS operators: `timeout()` to cancel slow requests, `catchError()` to handle errors in the pipeline, `retry()` to automatically retry failed operations, and `map()` to transform responses. It also supports streaming responses — if a handler returns an Observable that emits multiple values, the interceptor pipeline handles all of them. In practice, most interceptors use just `tap()` and `map()`, so deep RxJS knowledge isn't required day-to-day."

---

**Q7: How would you implement rate limiting in NestJS?**

> "NestJS has `@nestjs/throttler` for built-in rate limiting. You install it, add `ThrottlerModule.forRoot()` to AppModule imports with rate and limit config, then add `ThrottlerGuard` as a global guard via `APP_GUARD`. You can then use `@SkipThrottle()` to exempt certain routes and `@Throttle()` to override limits per route. For distributed systems with multiple server instances, you configure the throttler storage to use Redis so rate limits are shared across instances. The guard tracks requests per IP by default, but you can customize the key by extending ThrottlerGuard and overriding `getTracker()` — for example, using the authenticated user's ID instead of IP."

---

## Quick Reference — Day 3 Cheat Sheet

```
Pipeline order:
  Middleware → Guard → Interceptor(before) → Pipe → Handler
  → Interceptor(after) → ExceptionFilter (if error)

Middleware:   Express level | no execution context | must call next()
Guard:        canActivate() → boolean | auth/authorization | has ExecutionContext
Interceptor:  wraps handler | before + after | uses RxJS | transforms response
Pipe:         transform + validate | just before handler | throw 400 on fail
ExceptionFilter: @Catch() | formats error response | runs last if error thrown

Applying globally (in AppModule providers):
  { provide: APP_GUARD, useClass: MyGuard }
  { provide: APP_INTERCEPTOR, useClass: MyInterceptor }
  { provide: APP_PIPE, useClass: ValidationPipe }
  { provide: APP_FILTER, useClass: AllExceptionsFilter }

Custom decorators:
  createParamDecorator() → extracts data from request (@CurrentUser)
  applyDecorators()      → combines multiple decorators (@AuthGet)
  SetMetadata()          → stores metadata read by guards via Reflector

RxJS in interceptors (just 4 operators to know):
  tap()        → side effects without changing value (logging)
  map()        → transform the response value
  catchError() → handle errors in the pipeline
  timeout()    → cancel if takes too long

Make route public with global auth guard:
  1. @Public() = SetMetadata('isPublic', true)
  2. In guard: check reflector.getAllAndOverride('isPublic', [...])
  3. If true → return true (skip auth)
```

---

*Day 3 complete. Tomorrow — Day 4: DTOs, Validation in depth, class-validator decorators, class-transformer, Response Serialization with Exclude/Expose, and whitelisting strategies.*
