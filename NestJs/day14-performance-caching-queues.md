# Day 14 — Performance, Caching & Queues
### NestJS Interview Prep | 3-Year Backend Engineer Level

> **Goal for today:** Master performance optimization in NestJS — Redis caching with CacheManager, Bull job queues for background processing, Bull dashboard, lazy loading modules, and clustering with PM2. A 3-year backend engineer must know how to make APIs fast, offload heavy work to background jobs, and scale to handle production traffic.

---

## 🧭 First-Timer Guidance — Read This First

**The three pillars of NestJS performance:**

```
1. CACHING — Don't compute the same thing twice
   Problem:  GET /products hits DB every time → 100ms per request
   Solution: Cache result in Redis → 1ms on subsequent requests
   When:     Read-heavy endpoints, expensive computations, external API calls

2. QUEUES — Don't make users wait for heavy work
   Problem:  POST /orders triggers: save to DB + send email + update inventory
             + process payment + generate PDF invoice → user waits 8 seconds
   Solution: Save to DB immediately → return 201 → background queue handles rest
             User gets response in 50ms, everything else happens behind the scenes
   When:     Email sending, image processing, report generation, webhooks

3. CLUSTERING — Use all CPU cores
   Problem:  Node.js is single-threaded → one process uses 1 of your 8 CPU cores
   Solution: PM2 cluster mode → 8 Node.js processes → all cores utilized
   When:     Production deployment on multi-core servers
```

**Setup — install packages:**
```bash
# Caching
npm install @nestjs/cache-manager cache-manager
npm install cache-manager-ioredis-yet ioredis  # Redis store

# Bull queues
npm install @nestjs/bull bull
npm install @types/bull --save-dev
npm install bull-board  # Dashboard UI

# PM2 (install globally)
npm install -g pm2
```

---

## Table of Contents

1. [Caching Fundamentals — Why and When](#1-caching-fundamentals--why-and-when)
2. [In-Memory Caching — Simple Setup](#2-in-memory-caching--simple-setup)
3. [Redis Caching — Production Setup](#3-redis-caching--production-setup)
4. [Cache Interceptor — Automatic Caching](#4-cache-interceptor--automatic-caching)
5. [Manual Caching with CacheManager](#5-manual-caching-with-cachemanager)
6. [Cache Invalidation Strategies](#6-cache-invalidation-strategies)
7. [Bull Queues — Background Jobs](#7-bull-queues--background-jobs)
8. [Job Processors — Handling Queue Work](#8-job-processors--handling-queue-work)
9. [Advanced Queue Patterns](#9-advanced-queue-patterns)
10. [Bull Dashboard — Monitoring Queues](#10-bull-dashboard--monitoring-queues)
11. [Lazy Loading Modules — Startup Performance](#11-lazy-loading-modules--startup-performance)
12. [PM2 Clustering — Multi-Core Scaling](#12-pm2-clustering--multi-core-scaling)
13. [Interview Questions & Answers](#13-interview-questions--answers)

---

## 1. Caching Fundamentals — Why and When

### The cost of not caching

```
WITHOUT caching — GET /products?category=electronics:
  Request 1: Query PostgreSQL → 120ms
  Request 2: Query PostgreSQL → 118ms  (same query, same result)
  Request 3: Query PostgreSQL → 125ms
  ...
  Request 1000: Query PostgreSQL → 119ms
  Total: 1000 × ~120ms = 120 SECONDS of DB time for same data
  DB CPU: maxed out, slow for other queries

WITH Redis caching:
  Request 1: Cache MISS → Query PostgreSQL (120ms) → Store in Redis
  Request 2: Cache HIT → Read from Redis (1ms) ✓
  Request 3: Cache HIT → Read from Redis (1ms) ✓
  ...
  Request 1000: Cache HIT → Read from Redis (1ms) ✓
  DB load: 1 query instead of 1000 — 99.9% reduction
```

### Cache types and trade-offs

```
In-Memory Cache (default NestJS):
  Storage: RAM of your Node.js process
  Speed:   ~0.1ms (no network call)
  Shared:  ❌ Each server instance has its own cache
  Persistent: ❌ Lost when process restarts
  Size:    Limited by Node.js process memory
  Use:     Development, single-instance apps, tiny datasets

Redis Cache:
  Storage: Redis server (separate process/machine)
  Speed:   ~1-2ms (network + Redis lookup)
  Shared:  ✓ All server instances share one cache
  Persistent: ✓ Optional persistence to disk
  Size:    Limited by Redis server RAM (easily GBs)
  Use:     Production, multi-instance, all serious apps

CDN Cache (outside NestJS scope):
  Storage: Edge servers worldwide
  Speed:   ~10-50ms (nearest edge server)
  Use:     Static assets, public API responses
```

### Cache-aside pattern (most common)

```
READ:
  1. Check cache for key
  2. If HIT → return cached value (fast path)
  3. If MISS → query database → store in cache → return value

WRITE:
  1. Write to database
  2. Invalidate (delete) the related cache key
     OR update the cache with new value

This is the pattern NestJS's CacheInterceptor implements automatically.
```

---

## 2. In-Memory Caching — Simple Setup

```typescript
// src/app.module.ts — in-memory cache setup

import { Module } from '@nestjs/common';
import { CacheModule } from '@nestjs/cache-manager';

@Module({
  imports: [
    CacheModule.register({
      isGlobal: true,
      // isGlobal: true → available in all modules without importing

      ttl: 60,
      // Time-to-live in SECONDS — how long each cached item lives
      // After TTL expires, next request fetches fresh data
      // Default: 5 seconds (very low — usually want higher)

      max: 100,
      // Maximum number of items in cache
      // When exceeded: LRU (Least Recently Used) items are evicted
      // In-memory: be careful with large objects consuming RAM

      // store defaults to 'memory' (in-process cache)
    }),
  ],
})
export class AppModule {}
```

---

## 3. Redis Caching — Production Setup

```typescript
// src/app.module.ts — production Redis cache

import { Module } from '@nestjs/common';
import { CacheModule } from '@nestjs/cache-manager';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { redisStore } from 'cache-manager-ioredis-yet';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),

    CacheModule.registerAsync({
      isGlobal: true,
      inject: [ConfigService],
      useFactory: async (configService: ConfigService) => ({
        store: await redisStore({
          // ── Redis connection ───────────────────────────────────────────
          socket: {
            host: configService.get('REDIS_HOST', 'localhost'),
            port: configService.get<number>('REDIS_PORT', 6379),
            tls: configService.get('NODE_ENV') === 'production',
            // TLS for production Redis (AWS ElastiCache, Upstash, etc.)
          },
          password: configService.get('REDIS_PASSWORD'),
          // Optional: no password for local dev

          database: 0,
          // Redis database index (0-15)
          // Use different databases for different apps or purposes:
          // 0: cache, 1: sessions, 2: queue metadata

          // ── Key prefix ─────────────────────────────────────────────────
          keyPrefix: `${configService.get('APP_NAME', 'myapp')}:cache:`,
          // All cache keys are prefixed: 'myapp:cache:products:all'
          // Prevents key collisions if multiple apps use same Redis
          // Makes cache inspection easier (filter by prefix in Redis CLI)
        }),

        // ── Cache settings ────────────────────────────────────────────────
        ttl: parseInt(configService.get('CACHE_TTL', '300')),
        // 300 seconds = 5 minutes default TTL

        max: parseInt(configService.get('CACHE_MAX', '1000')),
        // Max items in Redis (Redis has its own memory limits)
        // Redis's maxmemory-policy handles eviction when full
      }),
    }),
  ],
})
export class AppModule {}
```

### Feature module registration

```typescript
// src/products/products.module.ts
// When CacheModule is global, no import needed in feature modules
// But if you need module-specific TTL:

import { CacheModule } from '@nestjs/cache-manager';

@Module({
  imports: [
    CacheModule.register({
      ttl: 3600,
      // This module's cache uses 1 hour TTL
      // Overrides the global default for this module only
    }),
  ],
  // ...
})
export class ProductsModule {}
```

---

## 4. Cache Interceptor — Automatic Caching

### How CacheInterceptor works

```typescript
// CacheInterceptor automatically:
// 1. Generates a cache key from the request URL + query params
// 2. Checks Redis for that key
// 3. If HIT: returns cached value, skips controller + service
// 4. If MISS: calls controller → service → caches response → returns it

// Only caches GET requests by default
// POST/PUT/PATCH/DELETE are NOT cached (would be wrong — they change data)
```

### Applying CacheInterceptor

```typescript
// src/products/products.controller.ts

import {
  Controller, Get, Param, Query,
  UseInterceptors, CacheInterceptor,
  CacheKey, CacheTTL,
} from '@nestjs/common';
import { CacheInterceptor } from '@nestjs/cache-manager';

// ── Apply to entire controller ─────────────────────────────────────────────
@Controller('products')
@UseInterceptors(CacheInterceptor)
// All GET endpoints in this controller are cached
// Cache key = URL path + query string
// Cache TTL = global default (from CacheModule.register)
export class ProductsController {

  @Get()
  // Cache key: '/products' or '/products?category=electronics&page=1'
  findAll(@Query() query: FilterProductDto) {
    return this.productsService.findAll(query);
  }

  @Get('featured')
  @CacheTTL(3600)
  // Override TTL for this specific endpoint: 1 hour
  // Featured products change rarely — cache longer
  getFeatured() {
    return this.productsService.findFeatured();
  }

  @Get(':id')
  @CacheKey('product-detail')
  // Custom cache key base — appended with the id param
  // CacheKey: manually set the key instead of using the URL
  findOne(@Param('id') id: string) {
    return this.productsService.findOne(+id);
  }
}

// ── Apply globally (cache ALL GET requests) ────────────────────────────────
// src/app.module.ts
import { APP_INTERCEPTOR } from '@nestjs/core';
import { CacheInterceptor } from '@nestjs/cache-manager';

@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: CacheInterceptor,
      // Every GET request is cached automatically
      // Override with @CacheTTL(0) on specific routes to disable
    },
  ],
})
export class AppModule {}
```

### Customizing cache key generation

```typescript
// src/common/interceptors/custom-cache.interceptor.ts
// The default cache key is just the URL — but you might need to include
// auth context (user-specific cache) or other factors

import { CacheInterceptor, CACHE_KEY_METADATA } from '@nestjs/cache-manager';
import { ExecutionContext, Injectable } from '@nestjs/common';

@Injectable()
export class HttpCacheInterceptor extends CacheInterceptor {
  // Override trackBy() to customize the cache key
  trackBy(context: ExecutionContext): string | undefined {
    const request = context.switchToHttp().getRequest();
    const { httpAdapter } = this.httpAdapterHost;

    // Don't cache non-GET requests
    if (request.method !== 'GET') {
      return undefined;
      // undefined: don't cache this request
    }

    // Check for custom @CacheKey() decorator
    const cacheKey = this.reflector.get(
      CACHE_KEY_METADATA,
      context.getHandler(),
    );
    if (cacheKey) {
      return `${cacheKey}:${httpAdapter.getRequestUrl(request)}`;
    }

    // For authenticated routes: include user ID in cache key
    // This creates USER-SPECIFIC cache (each user gets their own cached data)
    const userId = request.user?.id;
    const baseUrl = httpAdapter.getRequestUrl(request);

    if (userId) {
      return `user:${userId}:${baseUrl}`;
      // Cache key: 'user:123:/api/v1/orders?page=1'
      // User 1's orders cache is separate from User 2's orders cache
    }

    // Public routes: cache by URL only (shared across all users)
    return baseUrl;
  }
}
```

---

## 5. Manual Caching with CacheManager

### When to use manual caching

```typescript
// Automatic CacheInterceptor is great for simple GET endpoints
// But sometimes you need manual control:
// - Cache database query results (not HTTP responses)
// - Cache external API responses
// - Cache expensive computations
// - Custom invalidation logic
// - Caching in services, not just controllers
```

### CacheManager in services

```typescript
// src/products/products.service.ts

import { Injectable, Inject } from '@nestjs/common';
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Cache } from 'cache-manager';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Product } from './entities/product.entity';

@Injectable()
export class ProductsService {
  constructor(
    @Inject(CACHE_MANAGER)
    // CACHE_MANAGER: the injection token for the cache instance
    private readonly cacheManager: Cache,

    @InjectRepository(Product)
    private readonly productRepository: Repository<Product>,
  ) {}

  // ── Basic get/set ──────────────────────────────────────────────────────
  async findOne(id: number): Promise<Product> {
    const cacheKey = `product:${id}`;

    // 1. Try cache first
    const cached = await this.cacheManager.get<Product>(cacheKey);
    // get<Product>(): returns Product or undefined if not found

    if (cached) {
      return cached;
      // Cache HIT — return immediately, no DB query
    }

    // 2. Cache MISS — fetch from database
    const product = await this.productRepository.findOne({
      where: { id },
      relations: { category: true },
    });

    if (!product) {
      throw new NotFoundException(`Product #${id} not found`);
    }

    // 3. Store in cache
    await this.cacheManager.set(
      cacheKey,
      product,
      300,
      // TTL: 300 seconds (5 minutes)
      // Undefined = use global default TTL from CacheModule.register
    );

    return product;
  }

  // ── Caching complex queries ────────────────────────────────────────────
  async findAll(filter: FilterProductDto): Promise<PaginatedResult<Product>> {
    // Build a cache key that includes ALL filter parameters
    // Same filter params → same cache key → same cached result
    const cacheKey = `products:${JSON.stringify({
      category: filter.category,
      minPrice: filter.minPrice,
      maxPrice: filter.maxPrice,
      page: filter.page,
      limit: filter.limit,
      sortBy: filter.sortBy,
    })}`;
    // Note: JSON.stringify keeps key stable regardless of property order

    const cached = await this.cacheManager.get<PaginatedResult<Product>>(cacheKey);
    if (cached) return cached;

    // Expensive database query
    const result = await this.executeProductQuery(filter);

    await this.cacheManager.set(cacheKey, result, 60);
    // 60 seconds TTL for lists (changes more frequently than single items)

    return result;
  }

  // ── Caching external API calls ─────────────────────────────────────────
  async getExchangeRates(currency: string): Promise<ExchangeRates> {
    const cacheKey = `exchange:${currency}`;

    let rates = await this.cacheManager.get<ExchangeRates>(cacheKey);

    if (!rates) {
      // External API call — slow and potentially rate-limited
      rates = await this.currencyApiService.getRates(currency);

      await this.cacheManager.set(
        cacheKey,
        rates,
        3600,  // Cache for 1 hour — exchange rates don't change that often
      );
    }

    return rates;
  }

  // ── Manual cache invalidation ──────────────────────────────────────────
  async update(id: number, dto: UpdateProductDto): Promise<Product> {
    const product = await this.productRepository.findOne({ where: { id } });
    if (!product) throw new NotFoundException();

    Object.assign(product, dto);
    const updated = await this.productRepository.save(product);

    // Invalidate the specific product cache
    await this.cacheManager.del(`product:${id}`);
    // del(): delete a specific cache key

    // Also invalidate product list caches (they're now stale)
    // Problem: we don't know all the list cache keys that might include this product
    // Solution: store keys by pattern OR use a tag-based approach
    // Simple solution: clear all products caches
    await this.invalidateProductLists();

    return updated;
  }

  async invalidateProductLists(): Promise<void> {
    // Strategy 1: Key pattern deletion (Redis-specific)
    // Using ioredis directly for pattern-based deletion
    const client = this.cacheManager.store.getClient();
    // Get the underlying Redis client

    const keys = await client.keys('myapp:cache:products:*');
    // Find all keys matching the product list pattern
    if (keys.length > 0) {
      await client.del(...keys);
      // Delete all matching keys at once
    }

    // Strategy 2: Simple TTL approach (works with any store)
    // Don't delete manually — let the short TTL handle expiry
    // Set list cache TTL to 60 seconds — stale for max 60s after update
    // Often acceptable: "eventual consistency" for list views
  }

  // ── Caching with tags (advanced) ──────────────────────────────────────
  // Tag-based invalidation: group cache entries by tag
  // Invalidating a tag deletes all entries with that tag
  // Not built into cache-manager — requires custom implementation

  async setWithTag(key: string, value: any, ttl: number, tags: string[]) {
    const client = this.cacheManager.store.getClient();

    // Store the value
    await this.cacheManager.set(key, value, ttl);

    // Register this key under each tag
    for (const tag of tags) {
      const tagKey = `tag:${tag}`;
      await client.sadd(tagKey, key);
      // SADD: add to Redis Set (automatically deduplicates)
      await client.expire(tagKey, ttl + 60);
      // Tag set should live slightly longer than the cached values
    }
  }

  async invalidateTag(tag: string): Promise<void> {
    const client = this.cacheManager.store.getClient();
    const tagKey = `tag:${tag}`;

    // Get all keys associated with this tag
    const keys = await client.smembers(tagKey);
    // SMEMBERS: get all members of a Redis Set

    if (keys.length > 0) {
      await client.del(...keys);  // Delete all tagged cache entries
    }
    await client.del(tagKey);    // Delete the tag set itself
  }

  // Usage example with tags:
  // Store product #1 under tag 'products' and 'category:electronics'
  // await setWithTag('product:1', product, 300, ['products', 'category:electronics'])
  //
  // When electronics category is updated: invalidate 'category:electronics'
  // → Deletes all cached products in electronics category
  //
  // When ANY product is updated: invalidate 'products'
  // → Deletes all product caches
}
```

---

## 6. Cache Invalidation Strategies

### The hardest problem in computer science

```typescript
// "There are only two hard things in Computer Science:
//  cache invalidation and naming things." — Phil Karlton

// Strategy 1: TTL-based (simplest, most common)
// Set appropriate TTL — stale data for at most TTL seconds
// Pros: simple, automatic, no invalidation code
// Cons: data can be stale for up to TTL seconds

await cacheManager.set('products:featured', data, 300);
// Users might see stale featured products for up to 5 minutes
// Acceptable for non-critical data

// Strategy 2: Delete on write (cache-aside)
async updateProduct(id: number, dto: UpdateProductDto) {
  const updated = await this.repo.save(dto);
  await cacheManager.del(`product:${id}`);  // Immediately invalidate
  return updated;
}

// Strategy 3: Update cache on write (write-through)
async updateProduct(id: number, dto: UpdateProductDto) {
  const updated = await this.repo.save(dto);
  await cacheManager.set(`product:${id}`, updated, 300); // Update cache immediately
  return updated;
}
// Pros: cache always has latest data
// Cons: cache is updated even if nobody reads this product

// Strategy 4: Version-based cache keys
// Append a version number to cache keys
// "Invalidate" by incrementing the version (old key becomes orphaned)
async getProductCacheKey(id: number): Promise<string> {
  const version = await this.cacheManager.get<number>(`version:product:${id}`) || 1;
  return `product:${id}:v${version}`;
}

async invalidateProduct(id: number): Promise<void> {
  const currentVersion = await this.cacheManager.get<number>(`version:product:${id}`) || 1;
  await this.cacheManager.set(`version:product:${id}`, currentVersion + 1);
  // Old cache key (v1) will expire naturally via TTL
  // New requests use v2 key → cache miss → fresh data fetched
}
```

---

## 7. Bull Queues — Background Jobs

### What is Bull?

Bull is a Redis-backed job queue for Node.js. It lets you:
- Add jobs to a queue and process them in the background
- Retry failed jobs automatically
- Schedule delayed jobs
- Process jobs in parallel across multiple workers
- Monitor job status and history

### Core Bull concepts

```
Job:      A unit of work with a name and data payload
Queue:    A named list of jobs waiting to be processed
Worker:   A function that processes jobs from a queue
Producer: Code that adds jobs to the queue
Consumer: The worker that processes jobs

Job States:
  waiting  → job is in queue, waiting to be picked up
  active   → job is currently being processed by a worker
  completed → job finished successfully
  failed    → job threw an error (may be retried)
  delayed   → job scheduled for future execution
  paused    → queue is paused, job waiting
```

### Setting up Bull module

```typescript
// src/app.module.ts — register queues globally

import { BullModule } from '@nestjs/bull';
import { ConfigModule, ConfigService } from '@nestjs/config';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),

    BullModule.forRootAsync({
      // forRootAsync: configure Bull with ConfigService
      inject: [ConfigService],
      useFactory: (configService: ConfigService) => ({
        redis: {
          host: configService.get('REDIS_HOST', 'localhost'),
          port: configService.get<number>('REDIS_PORT', 6379),
          password: configService.get('REDIS_PASSWORD'),
          db: 1,
          // Use Redis DB index 1 for queues (0 for cache)
          // Keeps queue data separate from cache data
          maxRetriesPerRequest: null,
          // null: don't limit Redis command retries
          // Required for Bull to work correctly
          enableReadyCheck: false,
          // Disable "ready" check for faster startup
        },

        defaultJobOptions: {
          // These options apply to ALL jobs unless overridden per-job
          removeOnComplete: 100,
          // Keep last 100 completed jobs in Redis (for monitoring)
          // false: keep all (uses more Redis memory)
          // true: remove immediately on completion

          removeOnFail: 200,
          // Keep last 200 failed jobs (for debugging)

          attempts: 3,
          // Retry failed jobs up to 3 times total

          backoff: {
            type: 'exponential',
            // Wait longer between each retry:
            // Retry 1: wait 2s, Retry 2: wait 4s, Retry 3: wait 8s
            delay: 2000,
            // Initial delay in milliseconds
          },
        },
      }),
    }),
  ],
})
export class AppModule {}
```

### Registering specific queues in feature modules

```typescript
// src/email/email.module.ts

import { BullModule } from '@nestjs/bull';

@Module({
  imports: [
    BullModule.registerQueue({
      name: 'email',
      // Queue name — must match in processor and producer

      // Optional: override defaults for this queue
      defaultJobOptions: {
        attempts: 5,
        // Emails are important — retry more times
        backoff: { type: 'exponential', delay: 1000 },
        removeOnComplete: 50,
      },
    }),

    BullModule.registerQueue({
      name: 'image-processing',
      // Image processing queue — different settings
      defaultJobOptions: {
        attempts: 2,
        timeout: 60000,
        // Job timeout: 60 seconds (image processing can be slow)
        removeOnComplete: 10,
      },
    }),
  ],
  providers: [EmailService, EmailProcessor],
  // EmailProcessor: the worker that processes email jobs
})
export class EmailModule {}
```

### Adding jobs to queues (Producer)

```typescript
// src/users/users.service.ts — adding jobs to the email queue

import { InjectQueue } from '@nestjs/bull';
import { Queue } from 'bull';

@Injectable()
export class UsersService {
  constructor(
    @InjectQueue('email')
    // @InjectQueue('email'): injects the 'email' queue
    // Must match the name in BullModule.registerQueue({ name: 'email' })
    private readonly emailQueue: Queue,

    @InjectQueue('image-processing')
    private readonly imageQueue: Queue,
  ) {}

  async register(dto: RegisterDto): Promise<User> {
    // Save user to database
    const user = await this.createUser(dto);

    // ── Add job to email queue (fire and forget) ───────────────────────
    await this.emailQueue.add(
      'send-welcome',
      // Job name: identifies WHICH type of job this is
      // Your processor uses this name to route to the right handler
      {
        userId: user.id,
        email: user.email,
        name: `${user.firstName} ${user.lastName}`,
      },
      // Job data: the payload your processor receives
      {
        // ── Per-job options ──────────────────────────────────────────────
        priority: 1,
        // Job priority: 1 = highest, higher numbers = lower priority
        // Welcome emails are high priority

        delay: 0,
        // Delay before processing (ms): 0 = process immediately

        attempts: 3,
        // Retry up to 3 times if processing fails
        // Overrides the queue's default

        backoff: {
          type: 'exponential',
          delay: 2000,
          // Wait 2s, 4s, 8s between retries
        },

        removeOnComplete: true,
        // Remove from queue immediately when done
        // Don't need to monitor this specific job
      },
    );

    // ── Delayed job ────────────────────────────────────────────────────
    // Send a follow-up email 24 hours after registration
    await this.emailQueue.add(
      'send-activation-reminder',
      { userId: user.id, email: user.email },
      {
        delay: 24 * 60 * 60 * 1000,  // 24 hours in milliseconds
        // Job won't be picked up for 24 hours
      },
    );

    // ── Repeatable/scheduled job ───────────────────────────────────────
    // Schedule weekly digest email (runs every Monday at 9am)
    await this.emailQueue.add(
      'weekly-digest',
      { userId: user.id },
      {
        repeat: {
          cron: '0 9 * * 1',
          // Cron format: minute hour dayOfMonth month dayOfWeek
          // '0 9 * * 1' = Every Monday at 9:00 AM
          tz: 'Asia/Kolkata',  // User's timezone
        },
        jobId: `weekly-digest-${user.id}`,
        // Unique jobId prevents duplicate scheduled jobs
        // If a job with same jobId exists, it won't be added again
      },
    );

    return user;
  }

  async updateProfilePicture(
    userId: number,
    imagePath: string,
  ): Promise<void> {
    // Don't process image synchronously — add to queue
    await this.imageQueue.add(
      'resize-avatar',
      {
        userId,
        imagePath,
        sizes: [32, 64, 128, 256],  // Generate multiple sizes
      },
      {
        priority: 5,
        // Lower priority than emails — image processing can wait
        attempts: 2,
        timeout: 30000,  // 30 second timeout
      },
    );
  }

  async bulkImportUsers(csvData: string): Promise<{ jobId: string }> {
    // Add a single bulk job
    const job = await this.emailQueue.add(
      'bulk-import',
      { csvData, importedBy: 'admin' },
      { removeOnComplete: false },  // Keep for status checking
    );

    return { jobId: job.id.toString() };
    // Return job ID so client can poll for status
  }

  // Check job status
  async getJobStatus(jobId: string): Promise<string> {
    const job = await this.emailQueue.getJob(jobId);
    if (!job) throw new NotFoundException('Job not found');

    const state = await job.getState();
    // Returns: 'waiting' | 'active' | 'completed' | 'failed' | 'delayed'
    return state;
  }
}
```

---

## 8. Job Processors — Handling Queue Work

### Creating a processor

```typescript
// src/email/email.processor.ts

import {
  Processor,
  Process,
  OnQueueActive,
  OnQueueCompleted,
  OnQueueFailed,
  OnQueueStalled,
} from '@nestjs/bull';
import { Job } from 'bull';
import { Logger } from '@nestjs/common';

// @Processor('email'): this class processes jobs from the 'email' queue
// Must match the queue name from BullModule.registerQueue({ name: 'email' })
@Processor('email')
export class EmailProcessor {
  private readonly logger = new Logger(EmailProcessor.name);

  constructor(
    private readonly emailService: EmailService,
    // Can inject any NestJS provider — full DI support
  ) {}

  // ── Job handlers ──────────────────────────────────────────────────────
  @Process('send-welcome')
  // @Process('send-welcome'): handles jobs with name 'send-welcome'
  async handleWelcomeEmail(job: Job<WelcomeEmailData>): Promise<void> {
    const { userId, email, name } = job.data;
    // job.data: the payload you passed when adding the job

    this.logger.log(`Processing welcome email for ${email} (job ${job.id})`);

    try {
      // Update progress — visible in Bull dashboard
      await job.progress(10);
      // job.progress(n): set job progress 0-100%

      await this.emailService.sendWelcome({
        to: email,
        name,
        userId,
      });

      await job.progress(100);
      // 100% = done

    } catch (error) {
      this.logger.error(
        `Failed to send welcome email to ${email}: ${error.message}`,
        error.stack,
      );
      throw error;
      // Throwing causes Bull to:
      // 1. Mark job as FAILED
      // 2. Retry if attempts remaining (based on job options)
      // 3. Move to failed queue after all attempts exhausted
    }
  }

  @Process('send-activation-reminder')
  async handleActivationReminder(job: Job): Promise<void> {
    const { userId, email } = job.data;

    // Check if user already activated — no need to send reminder
    const user = await this.usersService.findOne(userId);
    if (user.isEmailVerified) {
      this.logger.log(`User ${userId} already verified, skipping reminder`);
      return;
      // Return without throwing — job marked as COMPLETED (not failed)
    }

    await this.emailService.sendActivationReminder({ to: email, userId });
  }

  @Process('weekly-digest')
  async handleWeeklyDigest(job: Job): Promise<void> {
    const { userId } = job.data;

    await job.progress(20);
    const digest = await this.digestService.generateForUser(userId);

    await job.progress(60);
    await this.emailService.sendDigest({
      to: digest.user.email,
      content: digest,
    });

    await job.progress(100);
  }

  // ── Default processor — handles jobs without specific @Process() ──────
  @Process()
  async handleDefault(job: Job): Promise<void> {
    this.logger.warn(
      `Received unknown job type: ${job.name} with data:`,
      job.data,
    );
    // Log and discard unknown job types gracefully
  }

  // ── Concurrency control ────────────────────────────────────────────────
  @Process({ name: 'bulk-import', concurrency: 1 })
  // concurrency: 1 → only 1 bulk import runs at a time
  // (default: 1 per @Process — configure at class level for queue)
  async handleBulkImport(job: Job<{ csvData: string }>): Promise<void> {
    const rows = parseCsv(job.data.csvData);
    const total = rows.length;

    for (let i = 0; i < rows.length; i++) {
      await this.usersService.createFromCsvRow(rows[i]);
      await job.progress(Math.round((i + 1) / total * 100));
      // Update progress as we process each row
    }
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // QUEUE EVENT LISTENERS
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  @OnQueueActive()
  // Fires when a job starts processing
  onActive(job: Job): void {
    this.logger.log(
      `Processing job ${job.id} of type ${job.name} with data:`,
      JSON.stringify(job.data),
    );
  }

  @OnQueueCompleted()
  // Fires when a job completes successfully
  onCompleted(job: Job, result: any): void {
    this.logger.log(
      `Job ${job.id} (${job.name}) completed in ${job.processedOn - job.timestamp}ms`,
    );
  }

  @OnQueueFailed()
  // Fires when a job fails (all retry attempts exhausted)
  onFailed(job: Job, error: Error): void {
    this.logger.error(
      `Job ${job.id} (${job.name}) failed after ${job.attemptsMade} attempts:`,
      error.message,
      error.stack,
    );

    // Alert on critical failures
    if (job.attemptsMade >= job.opts.attempts) {
      // All retries exhausted — this is a real problem
      this.alertService.sendAlert({
        type: 'queue-job-failed',
        queue: 'email',
        jobName: job.name,
        jobData: job.data,
        error: error.message,
      });
    }
  }

  @OnQueueStalled()
  // Fires when a job gets "stuck" — started processing but never finished
  // Happens if worker process crashed during job execution
  onStalled(job: Job): void {
    this.logger.warn(`Job ${job.id} stalled and will be reprocessed`);
    // Bull automatically re-queues stalled jobs
  }
}
```

---

## 9. Advanced Queue Patterns

### Image processing queue with multiple job types

```typescript
// src/media/media.processor.ts

import { Processor, Process } from '@nestjs/bull';
import { Job } from 'bull';
import * as sharp from 'sharp';  // Image processing library

@Processor({
  name: 'image-processing',
  concurrency: 4,
  // concurrency: 4 → process up to 4 images simultaneously
  // Good for I/O-bound work (image resizing waits for file reads)
  // For CPU-bound: keep lower (1-2) to avoid blocking event loop
})
export class MediaProcessor {

  @Process('resize-avatar')
  async resizeAvatar(job: Job<{
    userId: number;
    imagePath: string;
    sizes: number[];
  }>): Promise<void> {
    const { userId, imagePath, sizes } = job.data;
    const results: string[] = [];
    const totalSteps = sizes.length;

    for (let i = 0; i < sizes.length; i++) {
      const size = sizes[i];
      const outputPath = `uploads/avatars/${userId}_${size}x${size}.webp`;

      await sharp(imagePath)
        .resize(size, size, { fit: 'cover' })
        .webp({ quality: 80 })
        .toFile(outputPath);

      results.push(outputPath);
      await job.progress(Math.round((i + 1) / totalSteps * 100));
    }

    // Update user record with new avatar paths
    await this.usersService.updateAvatarPaths(userId, results);
  }

  @Process('generate-thumbnail')
  async generateThumbnail(job: Job): Promise<{ thumbnailPath: string }> {
    // Jobs can RETURN values — stored in job.returnvalue
    // Accessible via dashboard or job.getState()
    const thumbnail = await this.createThumbnail(job.data);
    return { thumbnailPath: thumbnail };
    // Returned value stored in Redis — access via job.returnvalue
  }
}

// Separate processor for report generation:
@Processor({ name: 'reports', concurrency: 1 })
export class ReportsProcessor {

  @Process('generate-monthly-report')
  async generateMonthlyReport(job: Job): Promise<void> {
    const { month, year, adminEmail } = job.data;

    await job.progress(10);
    const reportData = await this.analyticsService.getMonthlyData(month, year);

    await job.progress(50);
    const pdfBuffer = await this.pdfService.generateReport(reportData);

    await job.progress(80);
    await this.storageService.uploadReport(pdfBuffer, `${year}-${month}.pdf`);

    await job.progress(90);
    await this.emailService.sendReport({
      to: adminEmail,
      month,
      year,
      attachmentUrl: reportUrl,
    });

    await job.progress(100);
  }
}
```

### Order processing saga with queues

```typescript
// src/orders/orders.processor.ts
// Demonstrates queue chaining: job → triggers another job

@Processor('orders')
export class OrdersProcessor {
  constructor(
    @InjectQueue('payments') private paymentsQueue: Queue,
    @InjectQueue('inventory') private inventoryQueue: Queue,
    @InjectQueue('email') private emailQueue: Queue,
    @InjectQueue('notifications') private notificationsQueue: Queue,
  ) {}

  @Process('process-order')
  async processOrder(job: Job<{ orderId: number }>): Promise<void> {
    const { orderId } = job.data;
    const order = await this.ordersService.findOne(orderId);

    // Step 1: Reserve inventory (add to inventory queue)
    await this.inventoryQueue.add(
      'reserve',
      { orderId, items: order.items },
      { priority: 1 },
    );
    await job.progress(25);

    // Step 2: Process payment
    await this.paymentsQueue.add(
      'charge',
      { orderId, amount: order.total, userId: order.userId },
      { priority: 1, attempts: 5 },
    );
    await job.progress(50);

    // Step 3: Send confirmation email
    await this.emailQueue.add(
      'order-confirmation',
      { orderId, userId: order.userId },
    );
    await job.progress(75);

    // Step 4: Push notification
    await this.notificationsQueue.add(
      'order-update',
      { orderId, userId: order.userId, status: 'confirmed' },
    );
    await job.progress(100);
  }
}
```

---

## 10. Bull Dashboard — Monitoring Queues

```typescript
// src/main.ts — set up Bull dashboard

import { NestFactory } from '@nestjs/core';
import { NestExpressApplication } from '@nestjs/platform-express';
import { createBullBoard } from '@bull-board/api';
import { BullAdapter } from '@bull-board/api/bullAdapter';
import { ExpressAdapter } from '@bull-board/express';
import { Queue } from 'bull';

async function bootstrap() {
  const app = await NestFactory.create<NestExpressApplication>(AppModule);

  // ── Bull Board setup ─────────────────────────────────────────────────
  const serverAdapter = new ExpressAdapter();
  serverAdapter.setBasePath('/admin/queues');
  // Dashboard accessible at: http://localhost:3000/admin/queues

  // Get all queue instances from DI container
  const emailQueue = app.get<Queue>('BullQueue_email');
  const imageQueue = app.get<Queue>('BullQueue_image-processing');
  const ordersQueue = app.get<Queue>('BullQueue_orders');
  const reportsQueue = app.get<Queue>('BullQueue_reports');
  // Bull queue providers are named 'BullQueue_<queueName>'

  createBullBoard({
    queues: [
      new BullAdapter(emailQueue),
      new BullAdapter(imageQueue),
      new BullAdapter(ordersQueue),
      new BullAdapter(reportsQueue),
    ],
    serverAdapter,
    options: {
      uiConfig: {
        boardTitle: 'MyApp Job Queues',
        boardLogo: { path: '/logo.png' },
      },
    },
  });

  // Mount the dashboard on the Express app
  app.use('/admin/queues', serverAdapter.getRouter());
  // Dashboard at: http://localhost:3000/admin/queues

  // !! IMPORTANT: Protect the dashboard in production !!
  // Add authentication middleware before the bull-board routes:
  app.use('/admin/queues', (req, res, next) => {
    const apiKey = req.headers['x-admin-key'];
    if (apiKey !== process.env.ADMIN_API_KEY) {
      res.status(401).json({ message: 'Unauthorized' });
      return;
    }
    next();
  });

  await app.listen(3000);
  console.log('Bull Dashboard: http://localhost:3000/admin/queues');
}
bootstrap();

// Bull Dashboard features:
// ✓ View all queues and their stats (waiting/active/completed/failed)
// ✓ Inspect individual jobs (data, error, retry count, timing)
// ✓ Retry failed jobs manually
// ✓ Clean old jobs
// ✓ Pause/resume queues
// ✓ Real-time job updates
```

---

## 11. Lazy Loading Modules — Startup Performance

### What is lazy loading?

```typescript
// EAGER loading (default):
// ALL modules load when app starts
// Start time grows as you add more features
// App might use Module X only for /admin routes
// But Module X loads at startup even if no admin requests come in

// LAZY loading:
// Load modules only when first needed
// Reduces startup time for large apps
// Great for infrequently used features (admin, reports, dev tools)
```

```typescript
// src/admin/admin.controller.ts — lazy loaded heavy module

import { Controller, Get, OnModuleInit } from '@nestjs/common';
import { LazyModuleLoader } from '@nestjs/core';

@Controller('admin')
export class AdminController {
  constructor(
    private readonly lazyModuleLoader: LazyModuleLoader,
    // LazyModuleLoader: NestJS utility for lazy module loading
  ) {}

  @Get('reports')
  async generateReport() {
    // ReportsModule is NOT loaded at app startup
    // It loads the FIRST time this endpoint is called
    const { ReportsModule } = await import('./reports/reports.module');
    const moduleRef = await this.lazyModuleLoader.load(() => ReportsModule);
    // load(): returns a module reference — loads the module if not already loaded
    // Subsequent calls return the SAME module (loaded once, cached)

    // Get the service from the lazy-loaded module
    const { ReportsService } = await import('./reports/reports.service');
    const reportsService = moduleRef.get(ReportsService);

    return reportsService.generateMonthlyReport();
  }
}

// ── When to use lazy loading ────────────────────────────────────────────────
// ✓ Admin modules used rarely
// ✓ Heavy modules with many imports (PDF generation, AI, ML)
// ✓ Feature-flagged modules (only enable in specific environments)
// ✓ Webhook handlers for specific integrations
// ✓ Developer tools modules (only in development)
//
// Don't use for:
// ✗ Core business modules (users, orders, products)
// ✗ Auth module (needed on every request)
// ✗ Config module (needed at startup)
```

---

## 12. PM2 Clustering — Multi-Core Scaling

### The problem PM2 solves

```
Node.js is single-threaded:
  1 process = 1 CPU core used
  8-core server = 7 cores sitting idle

PM2 cluster mode:
  Spawns N worker processes (one per CPU core)
  8-core server = 8 Node.js processes
  OS load balances requests between them
  Memory: 8 × ~200MB = 1.6GB (each process has own memory)
```

### PM2 ecosystem file

```javascript
// ecosystem.config.js — PM2 configuration

module.exports = {
  apps: [
    {
      name: 'myapp-api',
      script: 'dist/main.js',
      // The compiled NestJS app

      // ── Cluster mode ─────────────────────────────────────────────────
      instances: 'max',
      // 'max': create one instance per CPU core
      // Or: instances: 4 (specific number)

      exec_mode: 'cluster',
      // 'cluster': PM2 cluster mode — all instances share the same port
      // 'fork': separate processes (not load balanced by PM2)

      // ── Memory management ────────────────────────────────────────────
      max_memory_restart: '1G',
      // Restart a worker if it uses more than 1GB RAM
      // Prevents memory leaks from taking down the server

      // ── Environment ──────────────────────────────────────────────────
      env: {
        NODE_ENV: 'development',
        PORT: 3000,
      },
      env_production: {
        NODE_ENV: 'production',
        PORT: 3000,
      },
      // pm2 start ecosystem.config.js --env production

      // ── Logging ──────────────────────────────────────────────────────
      log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
      error_file: '/var/log/myapp/error.log',
      out_file: '/var/log/myapp/out.log',
      merge_logs: true,
      // Merge logs from all instances into one file

      // ── Zero-downtime restarts ────────────────────────────────────────
      wait_ready: true,
      // Wait for 'ready' signal from app before routing traffic
      // App sends: process.send('ready')
      // Ensures new version is up before killing old one

      listen_timeout: 10000,
      // Max time to wait for app to be ready (ms)

      kill_timeout: 5000,
      // Max time to wait for graceful shutdown (ms)

      // ── Auto restart ──────────────────────────────────────────────────
      watch: false,
      // false in production (watch = restart on file change = dev feature)

      autorestart: true,
      // Restart automatically on crash

      restart_delay: 5000,
      // Wait 5 seconds before restarting crashed process

      max_restarts: 10,
      // Max restart attempts before giving up
      // Prevents infinite restart loops
    },
  ],
};
```

### NestJS app with PM2 ready signal

```typescript
// src/main.ts — signal PM2 when app is ready

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // ... configure app ...

  await app.listen(3000);

  // Signal PM2 that this worker is ready to receive traffic
  if (process.send) {
    process.send('ready');
    // process.send exists when running under PM2 (IPC channel)
    // This enables PM2's wait_ready: true for zero-downtime deploys
  }

  // Graceful shutdown: handle SIGTERM from PM2
  process.on('SIGTERM', async () => {
    console.log('SIGTERM received: gracefully shutting down...');

    // Stop accepting new requests
    await app.close();
    // NestJS closes HTTP server and calls onModuleDestroy() hooks
    // Allows in-flight requests to complete

    // Disconnect from queue workers
    // (Bull workers stop accepting new jobs but finish current ones)

    process.exit(0);
    // Exit cleanly — PM2 starts a new worker to replace this one
  });
}

bootstrap();
```

### PM2 commands

```bash
# Start application
pm2 start ecosystem.config.js --env production

# Check status of all processes
pm2 status
# Output: shows each worker, CPU%, memory, restarts, uptime

# View logs (all workers merged)
pm2 logs myapp-api
pm2 logs myapp-api --lines 100  # last 100 lines

# Real-time monitoring
pm2 monit
# Interactive CPU, memory, log viewer

# Zero-downtime reload (restart workers one by one)
pm2 reload myapp-api
# New worker starts → gets 'ready' signal → old worker gracefully exits
# Traffic never drops

# Hard restart (all workers restart simultaneously)
pm2 restart myapp-api  # Brief downtime

# Stop all workers
pm2 stop myapp-api

# Delete from PM2 (stops and removes)
pm2 delete myapp-api

# Save PM2 config for startup on server reboot
pm2 save
pm2 startup  # Generate startup script for this OS

# Scale workers (while running)
pm2 scale myapp-api 4   # Scale to exactly 4 workers
pm2 scale myapp-api +2  # Add 2 more workers
```

### Important: shared state with clustering

```typescript
// PROBLEM with clustering:
// Each PM2 worker is a separate Node.js process
// In-memory state is NOT shared between workers

// ❌ This breaks with multiple workers:
@Injectable()
export class SessionService {
  private sessions = new Map<string, Session>();
  // Worker 1 has its sessions
  // Worker 2 has different sessions
  // Client might hit Worker 1 for login, Worker 2 for next request
  // → Session not found on Worker 2 → logged out!

// ✅ Solutions for shared state:

// 1. Use Redis for sessions (not in-memory)
// @nestjs/cache-manager with Redis store (covered earlier)

// 2. Use Redis for rate limiting
// ThrottlerModule with Redis storage (covered Day 10)

// 3. Use Redis adapter for WebSockets
// Socket.io Redis adapter (covered Day 11)

// 4. Use Redis/Bull for queues
// Bull already uses Redis → works across all workers ✓

// Rule: ANYTHING that needs to be shared across requests
// or workers must go through Redis, not in-memory variables
```

---

## 13. Interview Questions & Answers

**Q1: What is the difference between in-memory caching and Redis caching in NestJS?**

> "In-memory caching stores data in the Node.js process RAM using a simple JavaScript Map. It's extremely fast (~0.1ms) since there's no network call, but it has three critical limitations: it's not shared across multiple server instances (each process has its own isolated cache), it's lost when the process restarts or crashes, and it's limited by Node.js process memory. Redis caching stores data in a separate Redis server accessible by all instances. It's slightly slower (~1-2ms for network round trip) but shared across all app instances, persistent across restarts, and can store GBs of data. For single-instance development apps, in-memory works fine. For any production deployment with multiple instances or zero-downtime requirements, Redis is mandatory."

---

**Q2: How does Bull guarantee that a job is processed even if the worker crashes?**

> "Bull uses Redis as its persistence layer. When a job is added to the queue, it's stored in Redis. When a worker picks up the job, Bull moves it to an 'active' state in Redis but keeps a heartbeat timer. If the worker crashes without completing the job, the heartbeat stops. Bull's 'stalled job' detection periodically checks for jobs that have been active too long without a heartbeat. When it detects a stalled job, it moves it back to the waiting queue for reprocessing. With `noAck: false` (manual acknowledgement mode), a job is only removed from Redis after the processor explicitly completes it — throwing an error triggers the retry logic instead. This is why `attempts` and `backoff` options are important: they define how many times and how long Bull waits before giving up and marking a job as permanently failed."

---

**Q3: What is cache invalidation and what strategies exist?**

> "Cache invalidation is determining when cached data is stale and needs to be removed or updated. It's notoriously difficult because you must balance freshness vs performance. TTL-based invalidation sets a time limit on cached data — it expires automatically after N seconds. Simple but data can be stale for up to TTL seconds. Write-through invalidation deletes or updates the cache immediately when the underlying data changes — ensures consistency but requires finding all affected cache keys, which can be complex for list/aggregate caches. Tag-based invalidation groups cache entries under tags (e.g., all product caches tagged with 'products') — invalidating the tag deletes all related entries atomically. Event-driven invalidation uses message queues — when a service updates data, it emits a cache-invalidation event that all cache layers listen to. For most applications, TTL + delete-on-write for specific keys is sufficient."

---

**Q4: When would you use a job queue instead of processing work synchronously?**

> "I use queues for work that doesn't need to be completed before returning the HTTP response, OR work that might fail and needs retries, OR work that should be rate-limited. Concretely: email sending (user doesn't need to wait for the email server), image/video processing (can take minutes), PDF generation for reports, payment webhooks (must be processed reliably with retries), batch data imports, sending push notifications to thousands of users, and any external API calls that might be rate-limited. The key question is: does the user need this result RIGHT NOW in this HTTP response? If yes, keep it synchronous. If no, put it in a queue. Background queues also provide resilience — if the email service is temporarily down, emails queue up and are sent when it recovers, rather than losing them or blocking the API."

---

**Q5: How does PM2 cluster mode work and what problems does it solve?**

> "PM2 cluster mode spawns N worker processes — typically one per CPU core — all listening on the same port. The OS kernel balances incoming TCP connections across these workers. Since each worker is an independent Node.js process, they truly run in parallel using different CPU cores. This solves the CPU underutilization problem of single-process Node.js: an 8-core server runs 8 processes, utilizing all cores. PM2's `reload` command enables zero-downtime deployments: it starts new workers one at a time, waits for the `ready` signal from each before routing traffic to it, then gracefully shuts down the old workers. The critical caveat is that workers don't share memory — any in-memory state (sessions, rate limit counters, WebSocket connections) must be stored in Redis. PM2 also handles automatic restarts on crashes, memory limit enforcement, and centralized logging from all workers."

---

**Q6: How do you prevent Bull queue jobs from running in duplicate when you have multiple workers?**

> "Bull uses Redis's atomic operations (specifically `RPOPLPUSH` and Lua scripts) to ensure each job is picked up by exactly one worker — this is built into Bull's architecture. When multiple workers poll the queue simultaneously, Redis's single-threaded nature ensures only one gets each job. For scheduled/repeatable jobs, Bull uses a `jobId` to deduplicate — if you add a repeatable job with `jobId: 'weekly-report-user-1'`, and it already exists in the queue, Bull won't create a duplicate. For idempotent processing, I also implement idempotency at the business logic level: check if the operation was already performed before doing it (e.g., check if welcome email was already sent before sending again). This handles the edge case where a job completes but the acknowledgement to Redis fails, causing it to be reprocessed."

---

## Quick Reference — Day 14 Cheat Sheet

```
Setup:
  npm install @nestjs/cache-manager cache-manager cache-manager-ioredis-yet ioredis
  npm install @nestjs/bull bull

CacheModule setup:
  CacheModule.registerAsync({ isGlobal: true, useFactory: async (config) => ({
    store: await redisStore({ socket: { host, port }, password }),
    ttl: 300,
  }) })

Cache methods (CACHE_MANAGER):
  await cacheManager.get<T>(key)         → T | undefined
  await cacheManager.set(key, value, ttl) → void
  await cacheManager.del(key)            → void
  await cacheManager.reset()             → void (clear all)

CacheInterceptor:
  @UseInterceptors(CacheInterceptor)  → cache GET endpoints
  @CacheTTL(3600)                     → override TTL per route
  @CacheKey('custom-key')             → override cache key

Bull setup:
  BullModule.forRootAsync({ useFactory: (config) => ({ redis: {...} }) })
  BullModule.registerQueue({ name: 'email', defaultJobOptions: {...} })

Producer (add jobs):
  @InjectQueue('email') private queue: Queue
  await queue.add('job-name', data, { priority, delay, attempts })
  await queue.add('cron-job', data, { repeat: { cron: '0 9 * * 1' } })

Consumer (process jobs):
  @Processor('email')
  @Process('job-name')
  async handle(job: Job<DataType>): Promise<void>
  await job.progress(50)  → update progress 0-100

Job options:
  priority: 1           → 1 = highest, higher = lower priority
  delay: 5000           → 5 second delay before processing
  attempts: 3           → retry up to 3 times
  backoff: { type: 'exponential', delay: 2000 }
  removeOnComplete: 100 → keep last 100 completed jobs
  repeat: { cron: '...' } → scheduled recurring job
  jobId: 'unique-id'    → prevent duplicate jobs

Bull Dashboard:
  createBullBoard({ queues: [new BullAdapter(queue)], serverAdapter })
  app.use('/admin/queues', serverAdapter.getRouter())
  Protect with auth middleware in production

PM2:
  exec_mode: 'cluster', instances: 'max'   → use all CPU cores
  pm2 start ecosystem.config.js --env production
  pm2 reload myapp-api                      → zero-downtime restart
  pm2 logs / pm2 monit / pm2 status

PM2 + shared state:
  In-memory state → NOT shared across workers
  Sessions, rate limits, WebSocket state → MUST use Redis
  Bull queues → already use Redis ✓
  CacheManager → use Redis store ✓

Cache invalidation strategies:
  TTL: let it expire (simplest, data stale for up to TTL)
  Delete-on-write: del(key) after update (immediate consistency)
  Write-through: set(key, newValue) after update
  Tag-based: group keys by tag, delete tag to invalidate all
```

---

*Day 14 complete. Tomorrow — Day 15: Deployment, Logging & Monitoring — Docker, Dockerfile for NestJS, environment-based config, Winston/Pino logging, health checks with @nestjs/terminus, Swagger docs, and CI/CD basics.*
