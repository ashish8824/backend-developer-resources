# Day 10 — REST API Best Practices
### NestJS Interview Prep | 3-Year Backend Engineer Level

> **Goal for today:** Master production-grade REST API design in NestJS — URI versioning, pagination strategies, filtering and sorting patterns, rate limiting, compression, security headers with Helmet, HATEOAS, and API documentation with Swagger. A 3-year backend engineer doesn't just make APIs that work — they make APIs that are consistent, secure, performant, and developer-friendly.

---

## 🧭 First-Timer Guidance — Read This First

**Why "Best Practices" matter in interviews:**

When an interviewer asks "how would you design a REST API for a product listing?", they're not just checking if you can write `@Get()`. They want to know:
- How you handle 10 million products (pagination)
- How clients filter by category, price range (filtering)
- How you prevent abuse (rate limiting)
- How you evolve the API without breaking clients (versioning)
- How you secure the API (headers, HTTPS, CORS)

**The REST API lifecycle of a 3-year engineer:**

```
Junior dev:   POST /createUser   GET /getUserById/1    DELETE /deleteUser
Mid-level:    POST /users        GET /users/1          DELETE /users/1
Senior/3yr:   POST /users        GET /users/1          DELETE /users/1
              + versioning       + pagination          + rate limiting
              + validation       + filtering           + security headers
              + proper errors    + sorting             + documentation
              + HTTPS            + HATEOAS links       + compression
```

**Setup — install packages for today:**
```bash
npm install @nestjs/throttler
npm install compression @types/compression
npm install helmet
npm install @nestjs/swagger swagger-ui-express
```

---

## Table of Contents

1. [REST Principles — The Foundation](#1-rest-principles--the-foundation)
2. [API Versioning — Evolving Without Breaking](#2-api-versioning--evolving-without-breaking)
3. [Pagination — Handling Large Datasets](#3-pagination--handling-large-datasets)
4. [Filtering — Query Parameter Patterns](#4-filtering--query-parameter-patterns)
5. [Sorting — Flexible Ordering](#5-sorting--flexible-ordering)
6. [Consistent Response Format — API Envelope](#6-consistent-response-format--api-envelope)
7. [Error Handling — Consistent Error Responses](#7-error-handling--consistent-error-responses)
8. [Rate Limiting — Throttler](#8-rate-limiting--throttler)
9. [Compression — Reducing Response Size](#9-compression--reducing-response-size)
10. [Security Headers — Helmet](#10-security-headers--helmet)
11. [CORS — Cross-Origin Resource Sharing](#11-cors--cross-origin-resource-sharing)
12. [HATEOAS — Hypermedia Links](#12-hateoas--hypermedia-links)
13. [Swagger — API Documentation](#13-swagger--api-documentation)
14. [Interview Questions & Answers](#14-interview-questions--answers)

---

## 1. REST Principles — The Foundation

### What makes an API truly RESTful?

```
REST = Representational State Transfer
Defined by Roy Fielding in 2000 — 6 constraints:

1. CLIENT-SERVER: Client and server are separate, communicate via HTTP
2. STATELESS: Each request contains ALL info needed — no server-side sessions
3. CACHEABLE: Responses must define if they can be cached
4. UNIFORM INTERFACE: Consistent URLs, HTTP methods, response formats
5. LAYERED SYSTEM: Client doesn't know if it's talking to final server or proxy
6. CODE ON DEMAND (optional): Server can send executable code (JavaScript)
```

### Resource naming — the most common interview topic

```
Resources are NOUNS, not verbs — HTTP methods ARE the verbs

❌ Wrong (RPC-style):
  GET    /getUsers
  POST   /createUser
  POST   /updateUser/1
  POST   /deleteUser/1
  GET    /getUserOrders/1
  POST   /sendWelcomeEmail/1

✅ Correct (REST-style):
  GET    /users              → list all users
  POST   /users              → create a user
  GET    /users/1            → get user #1
  PUT    /users/1            → replace user #1 (full update)
  PATCH  /users/1            → update user #1 (partial update)
  DELETE /users/1            → delete user #1

  GET    /users/1/orders     → get orders for user #1 (nested resource)
  POST   /users/1/orders     → create order for user #1

  POST   /users/1/emails/welcome  → trigger action (exception: verb for actions)
  POST   /auth/login              → actions that don't map to resources

HTTP Status Codes — use them correctly:
  200 OK              → successful GET, PUT, PATCH
  201 Created         → successful POST (resource created)
  204 No Content      → successful DELETE (or PUT/PATCH with no body)
  400 Bad Request     → invalid request data (validation failed)
  401 Unauthorized    → not authenticated (no/invalid token)
  403 Forbidden       → authenticated but not authorized
  404 Not Found       → resource doesn't exist
  409 Conflict        → duplicate resource (email already exists)
  422 Unprocessable   → semantically invalid (insufficient funds)
  429 Too Many Reqs   → rate limit exceeded
  500 Internal Error  → unexpected server error
```

---

## 2. API Versioning — Evolving Without Breaking

### Why versioning is critical

```
Without versioning:
  v1 API: GET /users returns { id, name, email }
  You want to change: { id, firstName, lastName, email }
  → All existing clients BREAK (they expect 'name', now get 'firstName')
  → You can't deploy the change without coordinating with ALL clients first

With versioning:
  GET /v1/users → { id, name, email }              (old clients still work)
  GET /v2/users → { id, firstName, lastName, email } (new clients use v2)
  → Both work simultaneously
  → Deprecate v1 after clients migrate (send Deprecation header)
```

### Three versioning strategies — NestJS supports all

```typescript
// src/main.ts

import { VersioningType } from '@nestjs/common';

const app = await NestFactory.create(AppModule);

// ── Strategy 1: URI Versioning (MOST COMMON) ──────────────────────────────
app.enableVersioning({
  type: VersioningType.URI,
  // URLs: /v1/users, /v2/users
  // Pros: Simple, visible, bookmarkable, easy to test in browser
  // Cons: URL "pollution", breaks REST principle (version is not a resource)
  // Used by: Twitter, GitHub, Stripe, most public APIs
  defaultVersion: '1',
  // If no version specified in @Controller → use v1
  prefix: 'v',
  // Prefix before the version number: /v1/, /v2/
});

// ── Strategy 2: Header Versioning ──────────────────────────────────────────
// app.enableVersioning({
//   type: VersioningType.HEADER,
//   header: 'X-API-Version',
//   // Client sends: X-API-Version: 2
//   // Pros: Clean URLs, truly RESTful
//   // Cons: Can't test easily in browser, not visible in URL
//   // Used by: Azure, some enterprise APIs
// });

// ── Strategy 3: Media Type Versioning ──────────────────────────────────────
// app.enableVersioning({
//   type: VersioningType.MEDIA_TYPE,
//   key: 'v=',
//   // Client sends: Accept: application/json;v=2
//   // Most RESTful but most complex — rarely used in practice
// });
```

### Controller-level versioning

```typescript
// src/users/users-v1.controller.ts

import { Controller, Get, Version } from '@nestjs/common';

// Version on entire controller
@Controller({
  path: 'users',
  version: '1',
  // All routes: GET /v1/users, POST /v1/users, etc.
})
export class UsersV1Controller {

  @Get()
  findAll() {
    // V1 response format — legacy
    return { users: [], total: 0 };
    // Note: 'users' key (old format)
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return { id, name: 'John Doe', email: 'john@test.com' };
    // V1: 'name' field (old format)
  }
}

// src/users/users-v2.controller.ts

@Controller({
  path: 'users',
  version: '2',
  // All routes: GET /v2/users, POST /v2/users, etc.
})
export class UsersV2Controller {

  @Get()
  findAll() {
    // V2 response format — new standard
    return {
      data: [],
      meta: { total: 0, page: 1, limit: 20 },
    };
    // Note: 'data' key + pagination meta (new format)
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return {
      id,
      firstName: 'John',
      lastName: 'Doe',
      email: 'john@test.com',
    };
    // V2: 'firstName' + 'lastName' (new format)
  }
}
```

### Method-level versioning

```typescript
// src/users/users.controller.ts
// Mix versions in the same controller file

@Controller('users')
export class UsersController {

  @Version('1')
  @Get()
  findAllV1() {
    return { users: [], total: 0 };
  }

  @Version('2')
  @Get()
  findAllV2() {
    return { data: [], meta: {} };
  }

  @Version(['1', '2'])
  // Same method handles BOTH versions
  @Get('health')
  health() {
    return { status: 'ok' };
  }

  // VERSION_NEUTRAL: accessible regardless of version
  @Version(VERSION_NEUTRAL)
  @Get('ping')
  ping() {
    return 'pong';
  }
}
```

### Deprecation strategy

```typescript
// src/common/interceptors/deprecation.interceptor.ts
// Add deprecation headers to old API versions

@Injectable()
export class DeprecationInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const response = context.switchToHttp().getResponse();
    const request = context.switchToHttp().getRequest();

    // Check if this is a v1 route
    if (request.path.startsWith('/v1/')) {
      response.setHeader('Deprecation', 'true');
      response.setHeader(
        'Sunset',
        'Sat, 01 Jun 2025 00:00:00 GMT',
        // When v1 will be SHUT DOWN — gives clients time to migrate
      );
      response.setHeader(
        'Link',
        '</v2/users>; rel="successor-version"',
        // Points to the new version
      );
    }

    return next.handle();
  }
}
```

---

## 3. Pagination — Handling Large Datasets

### Why pagination is non-negotiable

```
GET /users → 10 million rows
  - Kills your database (full table scan)
  - Kills your server (JSON serializing 10M objects)
  - Kills the client (parsing 10M objects)
  - Kills the network (gigabytes of data transfer)

GET /users?page=1&limit=20 → 20 rows
  - Fast database query (LIMIT 20 OFFSET 0)
  - Small response payload
  - Network efficient
  - Client can render incrementally
```

### Strategy 1: Offset Pagination (most common)

```typescript
// src/common/dto/pagination.dto.ts

import { IsOptional, IsInt, Min, Max } from 'class-validator';
import { Type } from 'class-transformer';

export class PaginationDto {
  @IsOptional()
  @Type(() => Number)
  // @Type(() => Number): convert query string '1' → number 1
  @IsInt()
  @Min(1)
  page?: number = 1;
  // Default: page 1

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  @Max(100)
  // Cap at 100 — prevent "give me ALL 10M records" requests
  limit?: number = 20;
  // Default: 20 per page
}

// src/common/interfaces/paginated-result.interface.ts

export interface PaginatedResult<T> {
  data: T[];
  meta: {
    total: number;        // Total records matching the query
    page: number;         // Current page
    limit: number;        // Records per page
    totalPages: number;   // Math.ceil(total / limit)
    hasPrevPage: boolean; // page > 1
    hasNextPage: boolean; // page < totalPages
    prevPage: number | null;  // page - 1 or null
    nextPage: number | null;  // page + 1 or null
  };
}

// src/common/utils/paginate.ts

export function paginateResponse<T>(
  data: T[],
  total: number,
  page: number,
  limit: number,
): PaginatedResult<T> {
  const totalPages = Math.ceil(total / limit);

  return {
    data,
    meta: {
      total,
      page,
      limit,
      totalPages,
      hasPrevPage: page > 1,
      hasNextPage: page < totalPages,
      prevPage: page > 1 ? page - 1 : null,
      nextPage: page < totalPages ? page + 1 : null,
    },
  };
}

// Usage in service:
async findAll(paginationDto: PaginationDto): Promise<PaginatedResult<User>> {
  const { page = 1, limit = 20 } = paginationDto;

  const [data, total] = await this.userRepository.findAndCount({
    skip: (page - 1) * limit,  // OFFSET: skip (page-1)*limit records
    take: limit,                // LIMIT: return 'limit' records
    order: { createdAt: 'DESC' },
  });

  return paginateResponse(data, total, page, limit);
}
```

### Strategy 2: Cursor Pagination (for real-time feeds)

```typescript
// Offset pagination problem:
// If new records are inserted between page 1 and page 2 requests:
//   Page 1 (before insert): records 1-20
//   Insert record 0 (becomes first)
//   Page 2 (after insert): records 21-40 → but record 20 appears AGAIN
//   (it shifted to position 21 due to new first record)
// Result: DUPLICATE records across pages — bad UX

// Cursor pagination solution:
// Instead of "skip N records", use "get records AFTER this specific record"
// cursor = the unique identifier of the LAST record on current page

// GET /posts?limit=20&cursor=eyJpZCI6MTAwfQ==
// GET /posts?limit=20&cursor=<base64 of {id: 100}>
// Returns: posts where id > 100, limit 20

// src/common/dto/cursor-pagination.dto.ts

import { IsOptional, IsString, IsInt, Min, Max } from 'class-validator';
import { Type } from 'class-transformer';

export class CursorPaginationDto {
  @IsOptional()
  @IsString()
  cursor?: string;
  // Base64 encoded cursor (last item's identifier)
  // Null for first page

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  @Max(100)
  limit?: number = 20;
}

// In service:
async findWithCursor(dto: CursorPaginationDto) {
  const { cursor, limit = 20 } = dto;

  let decodedCursor: { id: number; createdAt: string } | null = null;

  if (cursor) {
    // Decode the base64 cursor
    const decoded = Buffer.from(cursor, 'base64').toString('utf-8');
    decodedCursor = JSON.parse(decoded);
  }

  const queryBuilder = this.postRepository
    .createQueryBuilder('post')
    .where('post.isPublished = true');

  if (decodedCursor) {
    // Get records that come AFTER the cursor
    // Using createdAt + id for stable sorting (id breaks ties)
    queryBuilder.andWhere(
      '(post.createdAt < :createdAt OR (post.createdAt = :createdAt AND post.id < :id))',
      {
        createdAt: decodedCursor.createdAt,
        id: decodedCursor.id,
      },
    );
  }

  const items = await queryBuilder
    .orderBy('post.createdAt', 'DESC')
    .addOrderBy('post.id', 'DESC')
    .take(limit + 1)
    // Take one extra to determine if there's a next page
    .getMany();

  const hasNextPage = items.length > limit;
  const data = hasNextPage ? items.slice(0, -1) : items;
  // Remove the extra item we fetched

  // Create cursor from last item
  const lastItem = data[data.length - 1];
  const nextCursor = lastItem
    ? Buffer.from(
        JSON.stringify({
          id: lastItem.id,
          createdAt: lastItem.createdAt.toISOString(),
        }),
      ).toString('base64')
    : null;

  return {
    data,
    meta: {
      hasNextPage,
      nextCursor,
      // Client sends this cursor in next request
      limit,
    },
  };
}

// Client usage:
// First request:  GET /posts?limit=20
// Response:       { data: [...20 posts], meta: { nextCursor: 'eyJ...' } }
// Second request: GET /posts?limit=20&cursor=eyJ...
// Stable across inserts/deletes — no duplicate or missing records
```

### Offset vs Cursor — when to use each

```
Offset Pagination:
  ✓ When users need to jump to specific pages ("Go to page 50")
  ✓ Admin dashboards with stable data
  ✓ Simple to implement and understand
  ✗ Inconsistent with real-time data (inserts shift pages)
  ✗ Slow for large offsets (DB must count and skip rows)
  Examples: admin lists, reports, search results

Cursor Pagination:
  ✓ Real-time feeds (Twitter, Instagram, Facebook)
  ✓ Large datasets (billions of rows)
  ✓ Consistent results despite concurrent inserts
  ✗ Can't jump to specific pages
  ✗ More complex to implement
  Examples: social media feeds, notification lists, activity logs
```

---

## 4. Filtering — Query Parameter Patterns

```typescript
// src/products/dto/filter-product.dto.ts

import {
  IsOptional, IsString, IsNumber, Min, Max,
  IsEnum, IsBoolean, IsArray, IsInt,
} from 'class-validator';
import { Type, Transform } from 'class-transformer';

export class FilterProductDto extends PaginationDto {
  // ── Search ─────────────────────────────────────────────────────────────
  @IsOptional()
  @IsString()
  @Transform(({ value }) => value?.trim())
  search?: string;
  // GET /products?search=laptop
  // Searches name, description, SKU

  // ── Exact match filters ────────────────────────────────────────────────
  @IsOptional()
  @IsString()
  category?: string;
  // GET /products?category=electronics

  @IsOptional()
  @IsEnum(ProductStatus)
  status?: ProductStatus;
  // GET /products?status=active

  @IsOptional()
  @Type(() => Boolean)
  @IsBoolean()
  isFeatured?: boolean;
  // GET /products?isFeatured=true

  // ── Range filters ──────────────────────────────────────────────────────
  @IsOptional()
  @Type(() => Number)
  @IsNumber()
  @Min(0)
  minPrice?: number;
  // GET /products?minPrice=100

  @IsOptional()
  @Type(() => Number)
  @IsNumber()
  @Min(0)
  maxPrice?: number;
  // GET /products?maxPrice=1000

  @IsOptional()
  @Type(() => Number)
  @IsNumber()
  @Min(0)
  @Max(5)
  minRating?: number;
  // GET /products?minRating=4 → products rated 4.0 and above

  // ── Array filters ──────────────────────────────────────────────────────
  @IsOptional()
  @IsArray()
  @IsString({ each: true })
  @Transform(({ value }) => {
    // Handle both ?tags=a,b,c and ?tags[]=a&tags[]=b
    if (typeof value === 'string') return value.split(',').map(v => v.trim());
    return value;
  })
  tags?: string[];
  // GET /products?tags=sale,new,featured

  @IsOptional()
  @IsArray()
  @IsInt({ each: true })
  @Type(() => Number)
  @Transform(({ value }) => {
    if (typeof value === 'string') return value.split(',').map(Number);
    if (Array.isArray(value)) return value.map(Number);
    return [Number(value)];
  })
  categoryIds?: number[];
  // GET /products?categoryIds=1,2,3

  // ── Date range filters ─────────────────────────────────────────────────
  @IsOptional()
  @Type(() => Date)
  @IsDate()
  createdAfter?: Date;
  // GET /products?createdAfter=2024-01-01

  @IsOptional()
  @Type(() => Date)
  @IsDate()
  createdBefore?: Date;
  // GET /products?createdBefore=2024-12-31
}
```

### Building dynamic queries with TypeORM

```typescript
// src/products/products.service.ts

@Injectable()
export class ProductsService {
  async findAll(filterDto: FilterProductDto): Promise<PaginatedResult<Product>> {
    const {
      search,
      category,
      status,
      isFeatured,
      minPrice,
      maxPrice,
      minRating,
      tags,
      categoryIds,
      createdAfter,
      createdBefore,
      page = 1,
      limit = 20,
      sortBy = 'createdAt',
      sortOrder = 'DESC',
    } = filterDto;

    const queryBuilder = this.productRepository
      .createQueryBuilder('product')
      .leftJoinAndSelect('product.category', 'category');

    // ── Search ──────────────────────────────────────────────────────────
    if (search) {
      queryBuilder.andWhere(
        '(product.name ILIKE :search OR product.description ILIKE :search OR product.sku ILIKE :search)',
        { search: `%${search}%` },
      );
    }

    // ── Exact match filters ────────────────────────────────────────────
    if (category) {
      queryBuilder.andWhere('category.slug = :category', { category });
    }

    if (status) {
      queryBuilder.andWhere('product.status = :status', { status });
    }

    if (isFeatured !== undefined) {
      queryBuilder.andWhere('product.isFeatured = :isFeatured', { isFeatured });
    }

    // ── Range filters ──────────────────────────────────────────────────
    if (minPrice !== undefined) {
      queryBuilder.andWhere('product.price >= :minPrice', { minPrice });
    }

    if (maxPrice !== undefined) {
      queryBuilder.andWhere('product.price <= :maxPrice', { maxPrice });
    }

    if (minRating !== undefined) {
      queryBuilder.andWhere('product.avgRating >= :minRating', { minRating });
    }

    // ── Array filters ──────────────────────────────────────────────────
    if (tags?.length) {
      queryBuilder.andWhere('product.tags && :tags', { tags });
      // PostgreSQL array overlap operator &&
      // Returns products that have ANY of the given tags
    }

    if (categoryIds?.length) {
      queryBuilder.andWhere('product.categoryId IN (:...categoryIds)', {
        categoryIds,
        // :...categoryIds = spread array into IN (...) clause
      });
    }

    // ── Date filters ────────────────────────────────────────────────────
    if (createdAfter) {
      queryBuilder.andWhere('product.createdAt >= :createdAfter', {
        createdAfter,
      });
    }

    if (createdBefore) {
      queryBuilder.andWhere('product.createdAt <= :createdBefore', {
        createdBefore,
      });
    }

    // ── Sorting ────────────────────────────────────────────────────────
    const allowedSortFields = [
      'name', 'price', 'createdAt', 'avgRating', 'salesCount'
    ];
    const safeSortField = allowedSortFields.includes(sortBy)
      ? `product.${sortBy}`
      : 'product.createdAt';
    // Whitelist sort fields to prevent SQL injection via sort field names

    queryBuilder.orderBy(safeSortField, sortOrder);

    // ── Pagination ─────────────────────────────────────────────────────
    queryBuilder.skip((page - 1) * limit).take(limit);

    const [data, total] = await queryBuilder.getManyAndCount();

    return paginateResponse(data, total, page, limit);
  }
}
```

---

## 5. Sorting — Flexible Ordering

```typescript
// src/common/dto/sort.dto.ts

import { IsOptional, IsEnum, IsString } from 'class-validator';

export enum SortOrder {
  ASC = 'ASC',
  DESC = 'DESC',
}

export class SortDto {
  @IsOptional()
  @IsString()
  sortBy?: string = 'createdAt';
  // Which field to sort by

  @IsOptional()
  @IsEnum(SortOrder)
  sortOrder?: SortOrder = SortOrder.DESC;
  // ASC or DESC
}

// Multi-field sorting
// GET /products?sort=price:asc,rating:desc,name:asc
// → ORDER BY price ASC, rating DESC, name ASC

export class MultiSortDto {
  @IsOptional()
  @IsString()
  @Transform(({ value }) => {
    if (!value) return [];
    // Parse "price:asc,rating:desc" → [{ field: 'price', order: 'ASC' }]
    return value.split(',').map((part: string) => {
      const [field, order = 'asc'] = part.trim().split(':');
      return {
        field,
        order: order.toUpperCase() as SortOrder,
      };
    });
  })
  sort?: Array<{ field: string; order: SortOrder }>;
}

// Using multi-sort in service:
const SORTABLE_FIELDS = ['price', 'rating', 'name', 'createdAt', 'salesCount'];

if (sort?.length) {
  sort.forEach(({ field, order }, index) => {
    if (!SORTABLE_FIELDS.includes(field)) return;
    // Whitelist check — only allow known fields

    if (index === 0) {
      queryBuilder.orderBy(`product.${field}`, order);
    } else {
      queryBuilder.addOrderBy(`product.${field}`, order);
    }
  });
}
```

---

## 6. Consistent Response Format — API Envelope

### Why a consistent envelope matters

```typescript
// Without envelope — inconsistent across endpoints:
// GET /users → [{ id: 1, name: 'Alice' }]
// GET /users/1 → { id: 1, name: 'Alice' }
// POST /users → { user: { id: 1 }, message: 'Created' }
// DELETE /users/1 → 'User deleted'
// GET /users (error) → { error: 'Not found' }
// Client must handle EVERY response format differently — nightmare!

// With envelope — consistent across ALL endpoints:
// GET /users →
// {
//   "success": true,
//   "data": [{ "id": 1, "name": "Alice" }],
//   "meta": { "total": 100, "page": 1, "totalPages": 5 },
//   "timestamp": "2024-01-01T10:00:00.000Z"
// }
// Client handles ONE format — much simpler
```

### TransformInterceptor for consistent responses

```typescript
// src/common/interceptors/transform.interceptor.ts

import {
  Injectable, NestInterceptor, ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';
import { Request, Response } from 'express';

export interface ApiResponse<T> {
  success: boolean;
  data: T;
  message?: string;
  meta?: Record<string, any>;
  timestamp: string;
  path: string;
}

@Injectable()
export class TransformInterceptor<T>
  implements NestInterceptor<T, ApiResponse<T>>
{
  intercept(
    context: ExecutionContext,
    next: CallHandler,
  ): Observable<ApiResponse<T>> {
    const request = context.switchToHttp().getRequest<Request>();
    const response = context.switchToHttp().getResponse<Response>();

    return next.handle().pipe(
      map((data) => {
        // Handle paginated responses (have data + meta)
        if (data && typeof data === 'object' && 'meta' in data && 'data' in data) {
          return {
            success: true,
            data: data.data,
            meta: data.meta,
            timestamp: new Date().toISOString(),
            path: request.url,
          };
        }

        // Handle regular responses
        return {
          success: true,
          data,
          timestamp: new Date().toISOString(),
          path: request.url,
        };
      }),
    );
  }
}

// Register globally in AppModule:
// { provide: APP_INTERCEPTOR, useClass: TransformInterceptor }

// Now ALL endpoints return:
// {
//   "success": true,
//   "data": { ... },     ← the actual data
//   "timestamp": "...",
//   "path": "/api/v1/users"
// }
```

---

## 7. Error Handling — Consistent Error Responses

```typescript
// src/common/filters/all-exceptions.filter.ts

import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
  Logger,
} from '@nestjs/common';
import { QueryFailedError } from 'typeorm';

@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  private readonly logger = new Logger(AllExceptionsFilter.name);

  catch(exception: unknown, host: ArgumentsHost): void {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    const request = ctx.getRequest();

    let statusCode: number;
    let message: string | string[];
    let error: string;

    if (exception instanceof HttpException) {
      // NestJS HTTP exceptions (NotFoundException, BadRequestException, etc.)
      statusCode = exception.getStatus();
      const exceptionResponse = exception.getResponse();

      if (typeof exceptionResponse === 'string') {
        message = exceptionResponse;
        error = HttpStatus[statusCode] || 'Error';
      } else {
        message = (exceptionResponse as any).message || 'An error occurred';
        error = (exceptionResponse as any).error || HttpStatus[statusCode];
      }

    } else if (exception instanceof QueryFailedError) {
      // TypeORM database errors
      statusCode = HttpStatus.CONFLICT;

      // Handle specific PostgreSQL error codes
      const pgError = exception as any;
      if (pgError.code === '23505') {
        // Unique constraint violation
        message = 'A record with this value already exists';
        error = 'Conflict';
        statusCode = HttpStatus.CONFLICT; // 409
      } else if (pgError.code === '23503') {
        // Foreign key constraint violation
        message = 'Referenced record does not exist';
        error = 'Conflict';
      } else {
        message = 'Database error occurred';
        error = 'Database Error';
        statusCode = HttpStatus.INTERNAL_SERVER_ERROR;
      }

      // Log full error internally (not sent to client)
      this.logger.error('Database error:', exception.message, exception.stack);

    } else {
      // Unexpected errors (bugs, network errors, etc.)
      statusCode = HttpStatus.INTERNAL_SERVER_ERROR;
      message = process.env.NODE_ENV === 'production'
        ? 'An internal error occurred'
        : (exception as Error)?.message || 'Unknown error';
      error = 'Internal Server Error';

      this.logger.error(
        'Unexpected error:',
        exception instanceof Error ? exception.stack : String(exception),
      );
    }

    // Send consistent error response
    response.status(statusCode).json({
      success: false,
      error,
      message,
      statusCode,
      timestamp: new Date().toISOString(),
      path: request.url,
    });
  }
}

// All errors now look like:
// {
//   "success": false,
//   "error": "Not Found",
//   "message": "User #999 not found",
//   "statusCode": 404,
//   "timestamp": "2024-01-01T10:00:00.000Z",
//   "path": "/api/v1/users/999"
// }

// Validation errors:
// {
//   "success": false,
//   "error": "Bad Request",
//   "message": ["email must be an email", "password must be longer than 8 characters"],
//   "statusCode": 400,
//   "timestamp": "...",
//   "path": "/api/v1/auth/register"
// }
```

---

## 8. Rate Limiting — Throttler

### Setup and configuration

```typescript
// src/app.module.ts

import { ThrottlerModule, ThrottlerGuard } from '@nestjs/throttler';
import { ThrottlerStorageRedisService } from 'nestjs-throttler-storage-redis';
// npm install nestjs-throttler-storage-redis ioredis (for distributed throttling)

@Module({
  imports: [
    ThrottlerModule.forRootAsync({
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        throttlers: [
          {
            name: 'short',
            ttl: 1000,   // 1 second window
            limit: 10,   // 10 requests per second per IP (burst protection)
          },
          {
            name: 'medium',
            ttl: 60000,  // 1 minute window
            limit: 100,  // 100 requests per minute per IP
          },
          {
            name: 'long',
            ttl: 3600000, // 1 hour window
            limit: 1000,  // 1000 requests per hour per IP
          },
        ],

        // For distributed systems (multiple servers):
        // storage: new ThrottlerStorageRedisService({
        //   host: config.get('REDIS_HOST'),
        //   port: config.get('REDIS_PORT'),
        // }),
        // Without Redis: each server has its own counter (not shared)
        // With Redis: all servers share the same counter (correct limiting)

        // Custom key generator
        generateKey: (context, suffix) => {
          const req = context.switchToHttp().getRequest();
          // Default: throttle by IP address
          // Custom: throttle by user ID (if authenticated)
          const userId = req.user?.id;
          const ip = req.ip;
          return `${userId || ip}-${suffix}`;
          // Authenticated users: throttled by user ID (consistent across IPs)
          // Anonymous users: throttled by IP
        },

        errorMessage: 'Too many requests. Please wait before trying again.',
        // Custom error message for 429 responses
      }),
    }),
  ],
  providers: [
    {
      provide: APP_GUARD,
      useClass: ThrottlerGuard,
      // Apply rate limiting to ALL routes globally
    },
  ],
})
export class AppModule {}
```

### Per-route and per-controller throttling

```typescript
// src/auth/auth.controller.ts

import { Throttle, SkipThrottle } from '@nestjs/throttler';

@Controller('auth')
export class AuthController {

  // Strict rate limiting for login (brute force protection)
  @Throttle({ short: { ttl: 300000, limit: 5 } })
  // 5 attempts per 5 minutes — STRICT
  // Overrides global rate limit for this specific route
  @Post('login')
  login() {}

  // Stricter for password reset (avoid email flooding)
  @Throttle({ medium: { ttl: 3600000, limit: 3 } })
  // 3 requests per hour — very strict
  @Post('forgot-password')
  forgotPassword() {}

  // Skip throttling for this route entirely
  @SkipThrottle()
  @Get('me')
  getProfile() {}

  // Skip specific throttlers only
  @SkipThrottle({ short: true })
  // Skip 'short' throttler but keep 'medium' and 'long'
  @Get('health')
  health() {}
}

// Global skip for specific controllers:
@SkipThrottle()
@Controller('webhooks')
export class WebhooksController {
  // Webhooks from Stripe/GitHub — don't rate limit external services
}
```

### Custom ThrottlerGuard with better errors

```typescript
// src/common/guards/custom-throttler.guard.ts

import { ThrottlerGuard, ThrottlerException } from '@nestjs/throttler';
import { ExecutionContext, Injectable } from '@nestjs/common';
import { TooManyRequestsException } from '../exceptions/too-many-requests.exception';

@Injectable()
export class CustomThrottlerGuard extends ThrottlerGuard {
  // Override to customize the 429 error response
  protected async throwThrottlingException(
    context: ExecutionContext,
    throttlerLimitDetail: ThrottlerLimitDetail,
  ): Promise<void> {
    const response = context.switchToHttp().getResponse();

    // Add Retry-After header — tells client when to retry
    response.setHeader(
      'Retry-After',
      Math.ceil(throttlerLimitDetail.ttl / 1000),
      // TTL in seconds
    );

    // Add rate limit headers (like GitHub API)
    response.setHeader('X-RateLimit-Limit', throttlerLimitDetail.limit);
    response.setHeader('X-RateLimit-Remaining', 0);
    response.setHeader(
      'X-RateLimit-Reset',
      Math.floor(Date.now() / 1000) + Math.ceil(throttlerLimitDetail.ttl / 1000),
    );

    throw new TooManyRequestsException(
      `Rate limit exceeded. Try again in ${Math.ceil(throttlerLimitDetail.ttl / 1000)} seconds.`,
    );
  }
}
```

---

## 9. Compression — Reducing Response Size

```typescript
// src/main.ts

import * as compression from 'compression';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Enable gzip/deflate compression
  app.use(
    compression({
      level: 6,
      // Compression level: 1 (fastest, least compression) to 9 (slowest, most)
      // Level 6: good balance (Node.js default)

      threshold: 1024,
      // Only compress responses larger than 1KB
      // Compressing tiny responses wastes CPU

      filter: (req, res) => {
        // Custom filter: don't compress certain responses
        if (req.headers['x-no-compression']) {
          return false;
        }
        // Use default compression filter for everything else
        return compression.filter(req, res);
      },
    }),
  );

  // Compression reduces JSON response size by 60-80%
  // 100KB JSON → ~20KB compressed
  // Critical for mobile clients and high-traffic APIs

  await app.listen(3000);
}

// How compression works:
// 1. Client sends: Accept-Encoding: gzip, deflate, br
// 2. Server compresses response body with gzip
// 3. Server sends: Content-Encoding: gzip
// 4. Client decompresses automatically
// All modern browsers and HTTP clients handle this transparently
```

---

## 10. Security Headers — Helmet

```typescript
// src/main.ts

import helmet from 'helmet';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Helmet adds security-related HTTP headers
  app.use(helmet({
    // ── Content Security Policy ────────────────────────────────────────
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        // Only allow resources from same origin by default

        scriptSrc: ["'self'", "'unsafe-inline'"],
        // Allow scripts from same origin + inline (for Swagger UI)

        styleSrc: ["'self'", "'unsafe-inline'"],
        // Allow styles from same origin + inline

        imgSrc: ["'self'", 'data:', 'https:'],
        // Allow images from same origin, data: URIs, and HTTPS

        connectSrc: ["'self'"],
        // Allow fetch/XHR to same origin only

        fontSrc: ["'self'", 'https:'],
        // Allow fonts from same origin and HTTPS
      },
    },
    // Prevents XSS attacks by controlling what resources can load

    // ── X-Frame-Options ────────────────────────────────────────────────
    frameguard: { action: 'deny' },
    // Prevents your page from being embedded in an iframe
    // Protects against clickjacking attacks
    // Header: X-Frame-Options: DENY

    // ── HTTP Strict Transport Security ─────────────────────────────────
    hsts: {
      maxAge: 31536000,     // 1 year in seconds
      includeSubDomains: true,
      preload: true,
    },
    // Tells browsers: "Always use HTTPS for this domain for 1 year"
    // Even if user types http://, browser redirects to https:// automatically
    // Header: Strict-Transport-Security: max-age=31536000; includeSubDomains; preload

    // ── X-Content-Type-Options ─────────────────────────────────────────
    noSniff: true,
    // Prevents MIME type sniffing — browser must use declared Content-Type
    // Header: X-Content-Type-Options: nosniff

    // ── Referrer Policy ────────────────────────────────────────────────
    referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
    // Controls how much referrer info is sent
    // 'strict-origin-when-cross-origin': send origin only for cross-origin HTTPS

    // ── Permissions Policy ─────────────────────────────────────────────
    // Disable features you don't use to reduce attack surface
    permittedCrossDomainPolicies: false,

    // ── Remove X-Powered-By ────────────────────────────────────────────
    hidePoweredBy: true,
    // Removes 'X-Powered-By: Express' header
    // Don't reveal what technology you're using to attackers
  }));

  await app.listen(3000);
}

// Headers Helmet adds:
// X-DNS-Prefetch-Control: off
// X-Frame-Options: DENY
// X-Content-Type-Options: nosniff
// Strict-Transport-Security: max-age=31536000
// X-Download-Options: noopen
// X-Permitted-Cross-Domain-Policies: none
// Referrer-Policy: strict-origin-when-cross-origin
// Content-Security-Policy: default-src 'self'; ...
```

---

## 11. CORS — Cross-Origin Resource Sharing

```typescript
// src/main.ts

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  const config = app.get(ConfigService);

  // CORS: controls which domains can call your API
  app.enableCors({
    // ── Which origins can access the API ──────────────────────────────
    origin: (origin, callback) => {
      // Dynamic origin validation
      const allowedOrigins = config
        .get<string>('ALLOWED_ORIGINS', '')
        .split(',')
        .map(o => o.trim())
        .filter(Boolean);

      // Allow requests with no origin (mobile apps, Postman, curl)
      if (!origin) return callback(null, true);

      // Allow listed origins
      if (allowedOrigins.includes(origin)) return callback(null, true);

      // Allow all subdomains of your domain in production
      const mainDomain = config.get('MAIN_DOMAIN', 'myapp.com');
      if (origin.endsWith(`.${mainDomain}`)) return callback(null, true);

      // Block everything else
      callback(new Error(`CORS: Origin ${origin} not allowed`), false);
    },

    // ── Allowed HTTP methods ───────────────────────────────────────────
    methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
    // OPTIONS: required for CORS preflight requests

    // ── Allowed headers ────────────────────────────────────────────────
    allowedHeaders: [
      'Content-Type',
      'Authorization',
      'X-Request-Id',
      'X-API-Key',
    ],

    // ── Headers client can read ────────────────────────────────────────
    exposedHeaders: [
      'X-Total-Count',     // Total records (for pagination)
      'X-Request-Id',      // Request tracking
      'X-RateLimit-Limit',
      'X-RateLimit-Remaining',
      'Retry-After',
    ],

    // ── Credentials (cookies, authorization headers) ───────────────────
    credentials: true,
    // Allow cookies and Authorization header to be sent cross-origin
    // Required if using httpOnly cookies for tokens
    // When true: origin cannot be '*' (must be specific origins)

    // ── Preflight cache ────────────────────────────────────────────────
    maxAge: 86400,
    // Cache preflight response for 24 hours
    // Browser won't send OPTIONS preflight on every request
    // Significant performance improvement for CORS-heavy apps
  });

  await app.listen(3000);
}
```

---

## 12. HATEOAS — Hypermedia Links

### What is HATEOAS?

HATEOAS (Hypermedia As The Engine Of Application State) = responses include links to related actions.

```json
// Without HATEOAS — client must know all possible URLs:
{
  "id": 1,
  "name": "Alice",
  "role": "user"
}

// With HATEOAS — response tells client what it can do next:
{
  "id": 1,
  "name": "Alice",
  "role": "user",
  "_links": {
    "self": { "href": "/api/v1/users/1", "method": "GET" },
    "update": { "href": "/api/v1/users/1", "method": "PATCH" },
    "delete": { "href": "/api/v1/users/1", "method": "DELETE" },
    "orders": { "href": "/api/v1/users/1/orders", "method": "GET" },
    "collection": { "href": "/api/v1/users", "method": "GET" }
  }
}
```

```typescript
// src/common/utils/hateoas.util.ts
// Utility to add HATEOAS links to responses

export interface HateoasLink {
  href: string;
  method: string;
  rel?: string;
}

export interface WithLinks<T> {
  data: T;
  _links: Record<string, HateoasLink>;
}

export function addLinks<T extends { id: number | string }>(
  data: T,
  resourcePath: string,
  options?: {
    additionalLinks?: Record<string, HateoasLink>;
    includeDelete?: boolean;
  },
): WithLinks<T> {
  const links: Record<string, HateoasLink> = {
    self: {
      href: `${resourcePath}/${data.id}`,
      method: 'GET',
    },
    update: {
      href: `${resourcePath}/${data.id}`,
      method: 'PATCH',
    },
  };

  if (options?.includeDelete !== false) {
    links.delete = {
      href: `${resourcePath}/${data.id}`,
      method: 'DELETE',
    };
  }

  links.collection = {
    href: resourcePath,
    method: 'GET',
  };

  return {
    data,
    _links: { ...links, ...options?.additionalLinks },
  };
}

// Usage in controller:
@Get(':id')
async findOne(@Param('id', ParseIntPipe) id: number) {
  const user = await this.usersService.findOne(id);

  return addLinks(user, '/api/v1/users', {
    additionalLinks: {
      orders: { href: `/api/v1/users/${id}/orders`, method: 'GET' },
      profile: { href: `/api/v1/users/${id}/profile`, method: 'GET' },
    },
  });
}
```

---

## 13. Swagger — API Documentation

```typescript
// src/main.ts — Swagger setup

import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // ── Swagger Document Configuration ────────────────────────────────────
  const swaggerConfig = new DocumentBuilder()
    .setTitle('MyApp API')
    .setDescription(`
      ## MyApp REST API Documentation
      
      ### Authentication
      All protected endpoints require a Bearer JWT token.
      Get your token from POST /auth/login
      
      ### Rate Limiting
      - 100 requests per minute per IP
      - 5 login attempts per 5 minutes
    `)
    .setVersion('1.0')
    .setContact('API Team', 'https://myapp.com', 'api@myapp.com')
    .setLicense('MIT', 'https://opensource.org/licenses/MIT')

    .addBearerAuth(
      {
        type: 'http',
        scheme: 'bearer',
        bearerFormat: 'JWT',
        name: 'Authorization',
        description: 'Enter JWT token',
        in: 'header',
      },
      'JWT-auth',
      // This name matches what you put in @ApiBearerAuth('JWT-auth')
    )

    .addServer('http://localhost:3000', 'Local Development')
    .addServer('https://api.staging.myapp.com', 'Staging')
    .addServer('https://api.myapp.com', 'Production')

    .addTag('Authentication', 'Login, register, refresh tokens')
    .addTag('Users', 'User management')
    .addTag('Products', 'Product catalog')
    .addTag('Orders', 'Order management')

    .build();

  const document = SwaggerModule.createDocument(app, swaggerConfig);

  SwaggerModule.setup('docs', app, document, {
    // UI available at: http://localhost:3000/docs
    swaggerOptions: {
      persistAuthorization: true,
      // Keep auth token between page refreshes
      docExpansion: 'none',
      // Collapse all endpoints by default (cleaner UI for large APIs)
      filter: true,
      // Enable search box
      showRequestDuration: true,
      // Show how long each request took
    },
    customSiteTitle: 'MyApp API Docs',
    customCss: '.swagger-ui .topbar { display: none }',
    // Hide the top bar (green Swagger logo)
  });

  await app.listen(3000);
}
```

### Decorating controllers with Swagger

```typescript
// src/users/users.controller.ts

import {
  ApiTags, ApiOperation, ApiResponse, ApiBearerAuth,
  ApiParam, ApiQuery, ApiBody, ApiProperty,
  getSchemaPath,
} from '@nestjs/swagger';

// ── Response DTO for Swagger ──────────────────────────────────────────────
export class UserResponseDto {
  @ApiProperty({ example: 1, description: 'User ID' })
  id: number;

  @ApiProperty({ example: 'Alice', description: 'First name' })
  firstName: string;

  @ApiProperty({ example: 'Smith', description: 'Last name' })
  lastName: string;

  @ApiProperty({ example: 'alice@example.com', description: 'Email address' })
  email: string;

  @ApiProperty({ enum: Role, example: Role.USER, description: 'User role' })
  role: Role;

  @ApiProperty({ example: '2024-01-01T10:00:00.000Z' })
  createdAt: Date;
}

// ── Paginated Response Schema ─────────────────────────────────────────────
export class PaginatedUsersDto {
  @ApiProperty({ type: [UserResponseDto] })
  data: UserResponseDto[];

  @ApiProperty({
    example: {
      total: 100,
      page: 1,
      limit: 20,
      totalPages: 5,
    },
  })
  meta: object;
}

@ApiTags('Users')
// Groups this controller's endpoints under "Users" tag in Swagger UI
@ApiBearerAuth('JWT-auth')
// All endpoints in this controller require Bearer JWT
// Shows a lock icon in Swagger UI
@Controller({ path: 'users', version: '1' })
export class UsersController {

  @ApiOperation({
    summary: 'Get all users',
    description: 'Returns a paginated list of users. Requires ADMIN role.',
  })
  @ApiQuery({ name: 'page', required: false, type: Number, example: 1 })
  @ApiQuery({ name: 'limit', required: false, type: Number, example: 20 })
  @ApiQuery({ name: 'search', required: false, type: String, example: 'alice' })
  @ApiResponse({
    status: 200,
    description: 'Users retrieved successfully',
    type: PaginatedUsersDto,
  })
  @ApiResponse({ status: 401, description: 'Unauthorized — invalid or missing token' })
  @ApiResponse({ status: 403, description: 'Forbidden — insufficient permissions' })
  @Get()
  findAll(@Query() query: FilterUserDto) {}

  @ApiOperation({ summary: 'Get user by ID' })
  @ApiParam({ name: 'id', type: Number, example: 1, description: 'User ID' })
  @ApiResponse({
    status: 200,
    description: 'User found',
    type: UserResponseDto,
  })
  @ApiResponse({ status: 404, description: 'User not found' })
  @Get(':id')
  findOne(@Param('id', ParseIntPipe) id: number) {}

  @ApiOperation({ summary: 'Create a new user' })
  @ApiBody({ type: CreateUserDto })
  @ApiResponse({
    status: 201,
    description: 'User created successfully',
    type: UserResponseDto,
  })
  @ApiResponse({ status: 409, description: 'Email already exists' })
  @Post()
  create(@Body() dto: CreateUserDto) {}
}
```

---

## 14. Interview Questions & Answers

**Q1: What versioning strategy do you prefer and why?**

> "I prefer URI versioning — `/v1/users`, `/v2/users` — for most public APIs because it's the most visible and practical approach. It's easy to test in a browser, shows up clearly in logs, and clients can see which version they're using without needing to inspect headers. The theoretical argument against it is that version isn't a REST resource, but in practice this is the dominant industry approach used by Stripe, GitHub, Twitter, and most successful APIs. For internal microservice communication where you control all clients, header versioning is cleaner. For very sophisticated clients following the JSON:API spec, media type versioning can work. The most important thing is picking one strategy and being consistent — mixed versioning strategies across your API create confusion."

---

**Q2: What is the difference between offset pagination and cursor pagination?**

> "Offset pagination uses `SKIP n` and `LIMIT m` — 'skip the first 20 records and return the next 20'. It's simple to implement and supports jumping to arbitrary pages, but it has two problems: for very large offsets the database still has to count through all skipped rows (slow), and if records are inserted or deleted between requests, pages shift — you get duplicate or missing records. Cursor pagination instead asks 'give me records that come after this specific record' using a cursor — typically a base64-encoded combination of the last record's ID and timestamp. This is stable regardless of inserts and deletions, and the query is efficient since it uses an index. The trade-off is you can't jump to page 50 — you can only go forward and backward. I use offset pagination for admin dashboards with stable data, and cursor pagination for social feeds and real-time data."

---

**Q3: How do you handle rate limiting in a distributed system with multiple server instances?**

> "In a single-server setup, `@nestjs/throttler`'s in-memory storage works fine. But with multiple servers (horizontal scaling, Kubernetes pods), each server maintains its own counter — a user could hit 100 requests on each of 5 servers, getting 500 total requests while each server thinks they're within the 100-request limit. The solution is to use a shared storage backend, specifically Redis via `nestjs-throttler-storage-redis`. All server instances connect to the same Redis instance and increment a shared counter with an expiry TTL. This ensures rate limits are enforced globally across the entire cluster. The counter key typically includes the IP address or user ID, and Redis's atomic `INCR` command prevents race conditions. In production with auto-scaling, always use Redis-backed throttling."

---

**Q4: What security headers does Helmet add and why do they matter?**

> "Helmet adds several HTTP response headers that each prevent specific attack categories. `X-Frame-Options: DENY` prevents clickjacking — attackers embedding your page in an invisible iframe to trick users into clicking things. `X-Content-Type-Options: nosniff` prevents MIME sniffing — browsers trying to 'guess' the content type of a response and treating JavaScript as HTML. `Strict-Transport-Security` enforces HTTPS — once a browser sees this header, it won't connect over HTTP for the specified duration. `Content-Security-Policy` is the most powerful — it tells browsers exactly which sources scripts, styles, images, and other resources can load from, preventing XSS attacks by blocking unauthorized script execution. `X-Powered-By` removal stops attackers from knowing you're running Express and looking for Express-specific vulnerabilities."

---

**Q5: How do you design a flexible filtering system for a REST API?**

> "I use query parameters following consistent naming conventions. For exact matches: `?status=active&category=electronics`. For ranges: `?minPrice=100&maxPrice=1000`. For array includes: `?tags=sale,new` (comma-separated) or `?categoryIds=1,2,3`. For text search: `?search=laptop`. For dates: `?createdAfter=2024-01-01&createdBefore=2024-12-31`. In the DTO, I use `@Type(() => Number)` for number conversions, `@Transform` for array parsing, and `@IsOptional()` on all filter fields. In the service, I build the query conditionally — only adding WHERE clauses for filters that were actually provided. Crucially, I whitelist allowed sort fields to prevent SQL injection through the sort parameter, and I cap the limit parameter at a maximum (100) to prevent 'give me all 10M records' requests."

---

**Q6: What is HATEOAS and is it worth implementing?**

> "HATEOAS (Hypermedia As The Engine Of Application State) means including links in your API responses that tell clients what they can do next — similar to how a web page includes links to other pages. A user response would include links to update, delete, get their orders, etc. The benefit is that clients don't need to hardcode URL patterns — they discover them from responses, making the API more discoverable and clients less brittle to URL changes. However, HATEOAS has fallen out of favor for most REST APIs because it adds significant response payload size, most modern clients are built with the full API spec (Swagger) and don't benefit from runtime discovery, and implementing it consistently is complex. I implement a lightweight version of HATEOAS with `_links` on single-resource responses for discoverability, but I don't follow the full Richardson Maturity Model Level 3 in most projects. It's more valuable for public APIs with diverse clients."

---

**Q7: How do you handle API backward compatibility when you need to make breaking changes?**

> "The strategy has several layers. First, I use versioning from day one — even if I start with v1, having the infrastructure means I don't scramble when breaking changes are needed. Second, I try to avoid breaking changes: adding optional fields is non-breaking, adding new endpoints is non-breaking, deprecating vs removing is the key distinction. When I must make breaking changes, I release v2 alongside v1. I add `Deprecation: true` and `Sunset: <date>` response headers to v1 endpoints to alert clients programmatically. I document the migration path clearly in changelogs and Swagger descriptions. I monitor v1 usage metrics — when traffic drops below a threshold (or after the sunset date), I retire v1. The golden rule: never remove a version until you've communicated the sunset date, given clients adequate migration time (typically 6-12 months for public APIs), and confirmed major clients have migrated."

---

## Quick Reference — Day 10 Cheat Sheet

```
Versioning:
  app.enableVersioning({ type: VersioningType.URI, defaultVersion: '1' })
  @Controller({ path: 'users', version: '1' })
  @Version('2') on method
  @Version(VERSION_NEUTRAL) — version-independent
  Sunset headers for deprecation

Pagination:
  Offset: skip=(page-1)*limit, take=limit, findAndCount()
  Cursor: WHERE id < cursor ORDER BY id DESC LIMIT n+1
  Always: max limit cap (100), return total + meta in response

Filtering query params:
  Exact: ?status=active
  Range: ?minPrice=100&maxPrice=1000
  Array: ?tags=a,b,c → Transform split(',')
  Search: ?search=laptop → ILIKE '%laptop%'
  Date:  ?createdAfter=2024-01-01

Rate limiting (ThrottlerModule):
  Multiple tiers: short(1s/10), medium(60s/100), long(1hr/1000)
  @Throttle() — override per route
  @SkipThrottle() — disable per route
  Redis storage for distributed systems

Security (main.ts):
  app.use(helmet())         — security headers
  app.use(compression())    — gzip responses
  app.enableCors({...})     — CORS control
  credentials: true + specific origins (not *)

Response format:
  Success: { success: true, data, meta?, timestamp, path }
  Error:   { success: false, error, message, statusCode, timestamp, path }
  Use TransformInterceptor (global) + AllExceptionsFilter (global)

Swagger:
  DocumentBuilder: title, version, bearerAuth, tags, servers
  @ApiTags('Users') on controller
  @ApiBearerAuth() on protected controllers
  @ApiOperation({ summary, description }) on methods
  @ApiResponse({ status, type }) on methods
  @ApiProperty({ example, description }) on DTO fields
  SwaggerModule.setup('docs', app, document)

Whitelist sort fields:
  const allowed = ['price', 'name', 'createdAt'];
  if (!allowed.includes(sortBy)) sortBy = 'createdAt';
  // Prevents SQL injection via sort field names
```

---

*Day 10 complete. Tomorrow — Day 11: WebSockets & Real-time — WebSocket gateways, Socket.io adapter, namespaces, rooms, broadcasting, and authentication in WebSockets.*
