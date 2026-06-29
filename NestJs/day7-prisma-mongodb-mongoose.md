# Day 7 — Database with Prisma & MongoDB
### NestJS Interview Prep | 3-Year Backend Engineer Level

> **Goal for today:** Master two alternatives to TypeORM — Prisma (modern SQL ORM) and Mongoose (MongoDB ODM). Understand how each works, their advantages and limitations, when to choose each over TypeORM, and how to integrate them into NestJS professionally. A 3-year engineer must know all three database tools and explain the trade-offs clearly in interviews.

---

## 🧭 First-Timer Guidance — Read This First

**Why learn THREE database tools?**

In interviews, you'll be asked: *"Have you used Prisma? What about MongoDB?"* Companies use different tools. Knowing only TypeORM limits you.

**Quick mental map:**

```
TypeORM  → Traditional ORM, TypeScript decorators on entities
           Best for: teams coming from Java/Spring background,
           complex legacy schemas, full control over SQL

Prisma   → Modern ORM, schema-first approach, auto-generated client
           Best for: rapid development, type safety, clean APIs,
           greenfield projects, teams who love DX (developer experience)

Mongoose → MongoDB ODM (Object-Document Mapper), schema + model pattern
           Best for: document databases, flexible/dynamic data structures,
           real-time apps, horizontal scaling, unstructured data
```

**The core difference in philosophy:**

```
TypeORM:  Code First → Write classes → Generate schema
Prisma:   Schema First → Write schema → Generate code
Mongoose: Schema + Model → Documents (not tables)
```

**Setup for each:**
```bash
# Prisma
npm install prisma @prisma/client
npx prisma init

# Mongoose  
npm install @nestjs/mongoose mongoose
```

---

## Table of Contents

1. [Prisma — Modern ORM for NestJS](#1-prisma--modern-orm-for-nestjs)
2. [Prisma Schema — Defining Your Data Model](#2-prisma-schema--defining-your-data-model)
3. [PrismaService — Connecting to NestJS DI](#3-prismaservice--connecting-to-nestjs-di)
4. [Prisma CRUD — The Prisma Client API](#4-prisma-crud--the-prisma-client-api)
5. [Prisma Migrations](#5-prisma-migrations)
6. [Prisma vs TypeORM — Head-to-Head](#6-prisma-vs-typeorm--head-to-head)
7. [MongoDB & Mongoose — NoSQL in NestJS](#7-mongodb--mongoose--nosql-in-nestjs)
8. [Mongoose Schemas & Models](#8-mongoose-schemas--models)
9. [Mongoose CRUD Operations](#9-mongoose-crud-operations)
10. [Mongoose Advanced — Virtuals, Middleware, Population](#10-mongoose-advanced--virtuals-middleware-population)
11. [When to Choose SQL vs NoSQL](#11-when-to-choose-sql-vs-nosql)
12. [Interview Questions & Answers](#12-interview-questions--answers)

---

## 1. Prisma — Modern ORM for NestJS

### What is Prisma?

Prisma is a **next-generation ORM** that takes a fundamentally different approach from TypeORM:

- You write a `schema.prisma` file (the schema definition language)
- Prisma generates a fully-typed `PrismaClient` from your schema
- You use that generated client in your services
- No decorators on classes — the schema IS the source of truth

### Why developers love Prisma

```
TypeORM pain points Prisma solves:

1. Type safety:
   TypeORM: findOne() returns User | undefined (you decide the type)
   Prisma: every method returns EXACTLY what the schema says, down to
           which relations are included — perfect TypeScript inference

2. N+1 detection:
   TypeORM: accidentally make 100 queries in a loop — no warning
   Prisma: Prisma's dataloader detects N+1 and warns you

3. Migrations:
   TypeORM: migrations can be confusing, need separate data-source.ts
   Prisma: prisma migrate dev is a joy to use

4. Query results:
   TypeORM: query results are class instances with methods
   Prisma: query results are plain typed objects — simpler, predictable

5. Schema visualization:
   TypeORM: have to read entities to understand schema
   Prisma: one schema.prisma file shows EVERYTHING at a glance
```

### Project structure with Prisma

```
my-nestjs-app/
├── prisma/
│   ├── schema.prisma        ← Your entire data model
│   ├── migrations/          ← Generated migration files
│   │   └── 20240101_init/
│   │       └── migration.sql
│   └── seed.ts              ← Database seeding script
├── src/
│   ├── prisma/
│   │   ├── prisma.service.ts   ← PrismaClient wrapper
│   │   └── prisma.module.ts    ← Global Prisma module
│   ├── users/
│   │   ├── users.service.ts
│   │   └── users.controller.ts
│   └── app.module.ts
└── .env                     ← DATABASE_URL connection string
```

---

## 2. Prisma Schema — Defining Your Data Model

### The schema.prisma file — complete example

```prisma
// prisma/schema.prisma
// This single file defines: database connection, generator, ALL your models

// ── Generator ─────────────────────────────────────────────────────────────
generator client {
  provider = "prisma-client-js"
  // Tells Prisma: generate a JavaScript/TypeScript client
  // After 'npx prisma generate', a fully-typed client appears in node_modules
  // You import it: import { PrismaClient } from '@prisma/client'
}

// ── Data Source ───────────────────────────────────────────────────────────
datasource db {
  provider = "postgresql"
  // Options: "postgresql" | "mysql" | "sqlite" | "mongodb" | "sqlserver"
  
  url = env("DATABASE_URL")
  // DATABASE_URL=postgresql://user:password@localhost:5432/mydb
  // Prisma reads this from .env using the env() function
  
  // shadowDatabaseUrl: required for some providers during migration
  // shadowDatabaseUrl = env("SHADOW_DATABASE_URL")
}

// ── Models (Tables) ───────────────────────────────────────────────────────
// Each model maps to a database table
// Field name → camelCase in Prisma, snake_case in database (by convention)

model User {
  // ── Primary Key ──────────────────────────────────────────────────────
  id        Int       @id @default(autoincrement())
  // @id: marks as primary key
  // @default(autoincrement()): auto-increment integer
  
  // Alternative: UUID primary key
  // id        String    @id @default(uuid())
  
  // ── Required fields ───────────────────────────────────────────────────
  email     String    @unique
  // @unique: creates UNIQUE constraint
  
  firstName String    @map("first_name")
  // @map("first_name"): maps to 'first_name' column in DB
  // Prisma field is camelCase, DB column is snake_case
  
  lastName  String    @map("last_name")
  
  password  String
  
  // ── Optional fields ───────────────────────────────────────────────────
  phone     String?
  // ? makes the field optional (nullable in database)
  
  bio       String?   @db.Text
  // @db.Text: maps to TEXT type instead of VARCHAR(191)
  // @db prefix: database-specific type annotations
  
  // ── Enum field ────────────────────────────────────────────────────────
  role      Role      @default(USER)
  // References the Role enum defined below
  
  // ── Boolean ───────────────────────────────────────────────────────────
  isActive  Boolean   @default(true)
  
  // ── Decimal (for money) ───────────────────────────────────────────────
  balance   Decimal   @default(0) @db.Decimal(10, 2)
  // Decimal: Prisma's type for exact decimal numbers
  // @db.Decimal(10, 2): PostgreSQL DECIMAL(10,2)
  
  // ── JSON ──────────────────────────────────────────────────────────────
  metadata  Json?
  // Json: maps to jsonb in PostgreSQL, json in MySQL
  
  // ── Timestamps ────────────────────────────────────────────────────────
  createdAt DateTime  @default(now()) @map("created_at")
  updatedAt DateTime  @updatedAt @map("updated_at")
  // @default(now()): set to current timestamp on CREATE
  // @updatedAt: automatically updated on every UPDATE
  deletedAt DateTime? @map("deleted_at")
  // Soft delete: nullable timestamp
  
  // ── Relations ─────────────────────────────────────────────────────────
  profile   Profile?
  // One-to-One: a user has at most one profile
  
  orders    Order[]
  // One-to-Many: a user has many orders (array)
  
  roles     UserRole[]
  // Many-to-Many through explicit join table
  
  posts     Post[]

  // ── Indexes and constraints ───────────────────────────────────────────
  @@map("users")
  // @@map: database table name (users vs user)
  
  @@index([firstName, lastName])
  // @@index: composite index on first + last name
  
  @@index([email, isActive])
  // Index for common query: WHERE email = ? AND is_active = true
}

// ── Enum ──────────────────────────────────────────────────────────────────
enum Role {
  USER
  ADMIN
  MODERATOR
}

// ── Profile (One-to-One with User) ───────────────────────────────────────
model Profile {
  id        Int     @id @default(autoincrement())
  avatar    String?
  website   String?
  
  // Foreign key — the owning side of the relation
  userId    Int     @unique @map("user_id")
  // @unique on FK: enforces one-to-one (one profile per user)
  
  user      User    @relation(fields: [userId], references: [id], onDelete: Cascade)
  // fields: [userId] → the FK in THIS model
  // references: [id] → the PK in the related model (User)
  // onDelete: Cascade → delete profile if user is deleted
  
  @@map("profiles")
}

// ── Order (Many-to-One with User, One-to-Many with OrderItem) ────────────
model Order {
  id        Int         @id @default(autoincrement())
  total     Decimal     @db.Decimal(10, 2)
  status    OrderStatus @default(PENDING)
  createdAt DateTime    @default(now()) @map("created_at")
  
  userId    Int         @map("user_id")
  user      User        @relation(fields: [userId], references: [id])
  // Many orders belong to one user
  // No @unique here — multiple orders per user allowed
  
  items     OrderItem[]
  // One order has many items
  
  @@map("orders")
  @@index([userId])         // Index for "get all orders for user X"
  @@index([status, createdAt]) // Index for "get pending orders sorted by date"
}

enum OrderStatus {
  PENDING
  CONFIRMED
  SHIPPED
  DELIVERED
  CANCELLED
}

// ── OrderItem (Many-to-One with Order AND Product) ────────────────────────
model OrderItem {
  id        Int     @id @default(autoincrement())
  quantity  Int
  price     Decimal @db.Decimal(10, 2)
  
  orderId   Int     @map("order_id")
  order     Order   @relation(fields: [orderId], references: [id], onDelete: Cascade)
  
  productId Int     @map("product_id")
  product   Product @relation(fields: [productId], references: [id])
  
  @@map("order_items")
  @@unique([orderId, productId])
  // Composite unique: can't have same product twice in one order
}

// ── Many-to-Many through explicit join table ──────────────────────────────
// Prisma supports implicit M2M (no join model needed)
// but explicit join table gives you extra fields on the relation

model UserRole {
  userId    Int      @map("user_id")
  roleId    Int      @map("role_id")
  assignedAt DateTime @default(now()) @map("assigned_at")
  assignedBy Int?    @map("assigned_by")
  // Extra fields on the join table — not possible with implicit M2M
  
  user      User     @relation(fields: [userId], references: [id])
  role      AppRole  @relation(fields: [roleId], references: [id])
  
  @@id([userId, roleId])    // Composite primary key
  @@map("user_roles")
}

model AppRole {
  id          Int        @id @default(autoincrement())
  name        String     @unique
  description String?
  permissions String[]   // Array type (PostgreSQL array)
  userRoles   UserRole[]
  
  @@map("app_roles")
}

// ── Post (with full-text search field) ───────────────────────────────────
model Post {
  id          Int      @id @default(autoincrement())
  title       String   @db.VarChar(255)
  content     String   @db.Text
  published   Boolean  @default(false)
  viewCount   Int      @default(0) @map("view_count")
  publishedAt DateTime? @map("published_at")
  createdAt   DateTime @default(now()) @map("created_at")
  updatedAt   DateTime @updatedAt @map("updated_at")
  
  authorId    Int      @map("author_id")
  author      User     @relation(fields: [authorId], references: [id])
  
  tags        Tag[]    @relation("PostTags")
  // Implicit many-to-many: Prisma creates _PostTags join table automatically
  // No explicit model needed for simple M2M without extra fields
  
  @@map("posts")
  @@index([authorId, published])
}

model Tag {
  id    Int    @id @default(autoincrement())
  name  String @unique
  posts Post[] @relation("PostTags")
  
  @@map("tags")
}

model Product {
  id          Int         @id @default(autoincrement())
  name        String
  description String?     @db.Text
  price       Decimal     @db.Decimal(10, 2)
  stock       Int         @default(0)
  sku         String      @unique
  isActive    Boolean     @default(true) @map("is_active")
  createdAt   DateTime    @default(now()) @map("created_at")
  updatedAt   DateTime    @updatedAt @map("updated_at")
  
  orderItems  OrderItem[]
  
  @@map("products")
}
```

---

## 3. PrismaService — Connecting to NestJS DI

### Why create a PrismaService wrapper?

Prisma's `PrismaClient` is not a NestJS provider by default. You need to wrap it in a service so:
1. It participates in NestJS's dependency injection
2. You can manage the connection lifecycle (connect on startup, disconnect on shutdown)
3. You can add logging, middleware, and soft-delete filtering globally

```typescript
// src/prisma/prisma.service.ts

import {
  Injectable,
  OnModuleInit,
  OnModuleDestroy,
  Logger,
} from '@nestjs/common';
import { PrismaClient } from '@prisma/client';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class PrismaService
  extends PrismaClient            // Extend PrismaClient to inherit all methods
  implements OnModuleInit, OnModuleDestroy
{
  private readonly logger = new Logger(PrismaService.name);

  constructor(private readonly configService: ConfigService) {
    super({
      // PrismaClient constructor options
      datasources: {
        db: {
          url: configService.getOrThrow('DATABASE_URL'),
        },
      },

      log: [
        // Configure what Prisma logs
        { emit: 'event', level: 'query' },
        // 'event': fires an event you can listen to
        // 'stdout': prints to console directly
        { emit: 'stdout', level: 'error' },
        { emit: 'stdout', level: 'warn' },
        { emit: 'stdout', level: 'info' },
      ],

      errorFormat: 'pretty',
      // 'pretty': colored, readable errors
      // 'colorless': for production logging
      // 'minimal': just the error message
    });
  }

  async onModuleInit(): Promise<void> {
    // Connect to database when NestJS module initializes
    // If connection fails, app startup fails — good!
    await this.$connect();
    this.logger.log('Database connection established');

    // ── Soft delete middleware ─────────────────────────────────────────────
    // Globally filter out soft-deleted records
    // This means ALL queries automatically exclude deleted records
    this.$use(async (params, next) => {
      // params: query details (model, action, args)
      // next: function to call next middleware or execute query

      // Apply soft delete filter to read operations
      if (params.action === 'findFirst' || params.action === 'findMany') {
        // Add WHERE deletedAt = null to every find query
        params.args = params.args || {};
        params.args.where = {
          ...params.args.where,
          deletedAt: null,
          // If query already has a where, merge it with our filter
        };
      }

      // Convert 'delete' to soft delete (set deletedAt instead of DELETE)
      if (params.action === 'delete') {
        params.action = 'update';
        params.args.data = { deletedAt: new Date() };
        // Instead of DELETE FROM table WHERE id = x
        // Run: UPDATE table SET deleted_at = NOW() WHERE id = x
      }

      if (params.action === 'deleteMany') {
        params.action = 'updateMany';
        params.args.data = { deletedAt: new Date() };
      }

      return next(params);
    });

    // ── Query logging middleware ───────────────────────────────────────────
    // Log slow queries for performance monitoring
    if (configService.get('NODE_ENV') === 'development') {
      this.$on('query', (event: any) => {
        if (event.duration > 100) {
          // Log queries that take more than 100ms
          this.logger.warn(
            `Slow query (${event.duration}ms): ${event.query}`,
          );
        }
      });
    }
  }

  async onModuleDestroy(): Promise<void> {
    // Disconnect when NestJS is shutting down
    // Prevents connection leaks during hot reloads in development
    await this.$disconnect();
    this.logger.log('Database connection closed');
  }

  // ── Helper: enable shutdown hooks ─────────────────────────────────────
  // Call this in main.ts for graceful shutdown
  async enableShutdownHooks(app: any) {
    process.on('beforeExit', async () => {
      await app.close();
    });
  }
}
```

### PrismaModule — making PrismaService globally available

```typescript
// src/prisma/prisma.module.ts

import { Global, Module } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Global()
// @Global(): PrismaService is available everywhere without importing PrismaModule
// Like ConfigModule.forRoot({ isGlobal: true })
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
  // Must export so other modules can inject PrismaService
})
export class PrismaModule {}

// Register in AppModule:
// @Module({ imports: [PrismaModule, ...] })
// export class AppModule {}
```

---

## 4. Prisma CRUD — The Prisma Client API

### Using PrismaService in a service

```typescript
// src/users/users.service.ts

import { Injectable, NotFoundException, ConflictException } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { Prisma, User } from '@prisma/client';
// Import types generated by Prisma from your schema

@Injectable()
export class UsersService {
  constructor(private readonly prisma: PrismaService) {}
  // Inject PrismaService — it extends PrismaClient, so you get all methods

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // FIND OPERATIONS
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  async findAll(): Promise<User[]> {
    return this.prisma.user.findMany();
    // prisma.user: auto-generated accessor for the User model
    // findMany(): SELECT * FROM users
    // Returns: User[] — fully typed from schema
  }

  async findAllWithOptions(): Promise<User[]> {
    return this.prisma.user.findMany({
      // ── where: filtering ──────────────────────────────────────────────
      where: {
        isActive: true,
        role: 'ADMIN',
        // Simple equality: field: value
        
        email: {
          contains: '@gmail.com',
          // String operators: contains, startsWith, endsWith
          mode: 'insensitive',
          // Case-insensitive matching (PostgreSQL: ILIKE)
        },
        
        createdAt: {
          gte: new Date('2024-01-01'),
          // gte: greater than or equal (>=)
          // lte: less than or equal (<=)
          // gt: greater than (>)
          // lt: less than (<)
        },
        
        // OR conditions
        OR: [
          { firstName: { contains: 'alice' } },
          { firstName: { contains: 'bob' } },
        ],
        
        // NOT condition
        NOT: {
          role: 'MODERATOR',
        },
      },

      // ── select: specific fields ────────────────────────────────────────
      select: {
        id: true,
        firstName: true,
        lastName: true,
        email: true,
        role: true,
        // password: false — omit sensitive field (false = don't include)
        createdAt: true,
      },
      // When select is used, the return type changes to match selected fields!
      // TypeScript knows exactly which fields are in the result
      // This is Prisma's killer feature — perfect type narrowing

      // ── include: load relations ────────────────────────────────────────
      include: {
        profile: true,
        // Include the profile relation (LEFT JOIN + map to Profile object)
        
        orders: {
          include: {
            items: {
              include: {
                product: true,
                // Three levels deep: user → orders → items → product
              },
            },
          },
          where: {
            status: { not: 'CANCELLED' },
            // Filter which orders to include
          },
          orderBy: { createdAt: 'desc' },
          take: 5,
          // Only include last 5 orders
        },
      },
      // Note: can't use BOTH select AND include on the same level

      // ── orderBy: sorting ───────────────────────────────────────────────
      orderBy: [
        { createdAt: 'desc' },  // Primary sort
        { firstName: 'asc' },   // Secondary sort (when createdAt is equal)
      ],

      // ── pagination ─────────────────────────────────────────────────────
      skip: 0,    // OFFSET
      take: 20,   // LIMIT
    });
  }

  async findOne(id: number): Promise<User> {
    const user = await this.prisma.user.findUnique({
      where: { id },
      // findUnique: finds by unique field (id, email, etc.)
      // Returns null if not found (not undefined — Prisma consistency)
    });

    if (!user) {
      throw new NotFoundException(`User #${id} not found`);
    }
    return user;
  }

  async findByEmail(email: string): Promise<User | null> {
    return this.prisma.user.findUnique({
      where: { email },
      // email is @unique in schema → can use findUnique
    });
    // Returns null if not found
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // CREATE OPERATIONS
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  async create(data: Prisma.UserCreateInput): Promise<User> {
    // Prisma.UserCreateInput: auto-generated type for create operations
    // TypeScript knows exactly which fields are required vs optional
    // Based on your schema: required = no ?, optional = has ?

    return this.prisma.user.create({
      data,
      // SQL: INSERT INTO users (...) VALUES (...) RETURNING *
    });
  }

  async createWithRelations(): Promise<User> {
    return this.prisma.user.create({
      data: {
        firstName: 'Alice',
        lastName: 'Smith',
        email: 'alice@example.com',
        password: 'hashed_password',

        // Create related records in the same operation!
        profile: {
          create: {
            avatar: 'https://example.com/avatar.jpg',
            website: 'https://alice.dev',
          },
        },
        // INSERT INTO users + INSERT INTO profiles — in one round trip

        orders: {
          create: [
            {
              total: 99.99,
              items: {
                create: [
                  { productId: 1, quantity: 2, price: 49.99 },
                ],
              },
            },
          ],
        },
      },
      include: {
        profile: true,
        orders: true,
      },
      // Returns the created user WITH profile and orders loaded
    });
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // UPDATE OPERATIONS
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  async update(id: number, data: Prisma.UserUpdateInput): Promise<User> {
    // Prisma.UserUpdateInput: all fields optional (for partial update)
    try {
      return await this.prisma.user.update({
        where: { id },
        data,
        // SQL: UPDATE users SET ... WHERE id = $1 RETURNING *
        // If id doesn't exist → throws PrismaClientKnownRequestError (P2025)
      });
    } catch (error) {
      if (error instanceof Prisma.PrismaClientKnownRequestError) {
        if (error.code === 'P2025') {
          // P2025: "Record to update not found"
          throw new NotFoundException(`User #${id} not found`);
        }
      }
      throw error;
    }
  }

  async upsert(email: string, createData: any, updateData: any): Promise<User> {
    return this.prisma.user.upsert({
      where: { email },
      create: createData,
      update: updateData,
      // If email exists → UPDATE, if not → INSERT
      // SQL: INSERT ... ON CONFLICT (email) DO UPDATE SET ...
      // Perfect for: sync operations, idempotent imports
    });
  }

  async incrementLoginCount(id: number): Promise<User> {
    return this.prisma.user.update({
      where: { id },
      data: {
        loginCount: { increment: 1 },
        // Atomic increment — no race condition
        // SQL: UPDATE users SET login_count = login_count + 1 WHERE id = $1
        // Also: decrement, multiply, divide, set
        
        balance: { decrement: 10 },
        // Deduct 10 from balance atomically
      },
    });
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // DELETE OPERATIONS
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  async remove(id: number): Promise<User> {
    return this.prisma.user.delete({
      where: { id },
      // Hard delete — permanently removes from database
      // Throws P2025 if not found
    });
  }

  // Soft delete — using the middleware in PrismaService
  // When you call delete(), middleware converts it to update({ deletedAt: new Date() })
  async softDelete(id: number): Promise<User> {
    return this.prisma.user.delete({ where: { id } });
    // Middleware intercepts this and runs update instead
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // COUNT & AGGREGATE
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  async count(where?: Prisma.UserWhereInput): Promise<number> {
    return this.prisma.user.count({ where });
    // SELECT COUNT(*) FROM users WHERE ...
  }

  async getStats() {
    const [totalUsers, activeUsers, stats] = await this.prisma.$transaction([
      this.prisma.user.count(),
      this.prisma.user.count({ where: { isActive: true } }),
      this.prisma.user.aggregate({
        _avg: { balance: true },    // AVG(balance)
        _sum: { balance: true },    // SUM(balance)
        _max: { balance: true },    // MAX(balance)
        _min: { balance: true },    // MIN(balance)
        _count: { _all: true },     // COUNT(*)
      }),
    ]);
    // $transaction([]) — batch multiple queries in a single round trip
    // All execute concurrently, results returned as array

    return {
      totalUsers,
      activeUsers,
      averageBalance: stats._avg.balance,
      totalBalance: stats._sum.balance,
    };
  }

  async groupByRole() {
    return this.prisma.user.groupBy({
      by: ['role'],
      // GROUP BY role
      _count: { _all: true },
      // COUNT(*) per group
      _avg: { balance: true },
      // AVG(balance) per group
      having: {
        balance: { _avg: { gt: 100 } },
        // HAVING AVG(balance) > 100
      },
      orderBy: { _count: { id: 'desc' } },
    });
    // Returns: [{ role: 'USER', _count: { _all: 100 }, _avg: { balance: 50.5 } }]
  }
}
```

### Prisma transactions

```typescript
// Prisma transactions — two approaches

// ── Approach 1: $transaction([]) — batch queries ───────────────────────
async batchQueries() {
  const [users, count] = await this.prisma.$transaction([
    this.prisma.user.findMany({ take: 10 }),
    this.prisma.user.count(),
  ]);
  // Both queries run in a single transaction
  // Array of operations, array of results
}

// ── Approach 2: $transaction(callback) — interactive transaction ────────
async transferFunds(fromId: number, toId: number, amount: number) {
  return this.prisma.$transaction(async (tx) => {
    // tx: Prisma transaction client — use INSTEAD of this.prisma inside
    // All operations through tx are in the same transaction

    const sender = await tx.user.findUniqueOrThrow({
      where: { id: fromId },
      // findUniqueOrThrow: throws PrismaClientKnownRequestError if not found
      // Cleaner than findUnique() + manual null check
    });

    if (sender.balance.toNumber() < amount) {
      throw new Error('Insufficient funds');
      // Throwing inside transaction callback → automatic ROLLBACK
    }

    const [updatedSender, updatedReceiver] = await Promise.all([
      tx.user.update({
        where: { id: fromId },
        data: { balance: { decrement: amount } },
        // Atomic decrement — no race conditions
      }),
      tx.user.update({
        where: { id: toId },
        data: { balance: { increment: amount } },
      }),
    ]);

    await tx.transaction.create({
      data: { fromId, toId, amount, status: 'COMPLETED' },
    });

    return { updatedSender, updatedReceiver };
    // If we reach here without throwing → automatic COMMIT
  }, {
    maxWait: 5000,     // Max time to wait for a transaction slot (ms)
    timeout: 10000,    // Max time the transaction can run (ms)
    isolationLevel: Prisma.TransactionIsolationLevel.Serializable,
    // Isolation level: ReadUncommitted | ReadCommitted | RepeatableRead | Serializable
  });
}
```

---

## 5. Prisma Migrations

### The Prisma migration workflow

```bash
# Development workflow — always use migrate dev

# Step 1: Edit your schema.prisma (add a field, model, relation)
# Example: add 'phone' field to User model

# Step 2: Generate and apply migration
npx prisma migrate dev --name add_phone_to_user
# This command:
# 1. Diffs schema.prisma against database
# 2. Generates SQL migration file in prisma/migrations/
# 3. Applies the migration to your development database
# 4. Runs prisma generate (updates the PrismaClient types)
# 5. Records migration in _prisma_migrations table

# ── Generated migration file ───────────────────────────────────────────────
# prisma/migrations/20240101120000_add_phone_to_user/migration.sql
# -- AlterTable
# ALTER TABLE "users" ADD COLUMN "phone" VARCHAR(20);

# Step 3: The PrismaClient is automatically regenerated
# Your IDE now autocompletes user.phone everywhere

# ────────────────────────────────────────────────────────────────────────────
# Production workflow

# Apply pending migrations (no auto-generation in production)
npx prisma migrate deploy
# Only runs pending migrations — never generates new ones
# Safe to run in CI/CD pipeline before app starts

# ────────────────────────────────────────────────────────────────────────────
# Other useful commands

npx prisma generate
# Regenerate PrismaClient after manual schema changes (no migration)

npx prisma db push
# Push schema to database WITHOUT creating migration files
# Good for prototyping, NOT for production

npx prisma studio
# Opens a beautiful web-based database browser
# View and edit your data at http://localhost:5555

npx prisma db seed
# Run your seed script (prisma/seed.ts)
# Add to package.json: "prisma": { "seed": "ts-node prisma/seed.ts" }
```

### Prisma seed file

```typescript
// prisma/seed.ts — populate database with initial data

import { PrismaClient } from '@prisma/client';
import * as bcrypt from 'bcrypt';

const prisma = new PrismaClient();

async function main() {
  // upsert: insert if not exists, update if exists
  // Makes seeding idempotent — safe to run multiple times
  
  const adminUser = await prisma.user.upsert({
    where: { email: 'admin@myapp.com' },
    update: {},  // Don't change if already exists
    create: {
      firstName: 'Admin',
      lastName: 'User',
      email: 'admin@myapp.com',
      password: await bcrypt.hash('Admin@123456', 12),
      role: 'ADMIN',
      isActive: true,
    },
  });

  const categories = await Promise.all([
    prisma.category.upsert({
      where: { name: 'Electronics' },
      update: {},
      create: { name: 'Electronics' },
    }),
    prisma.category.upsert({
      where: { name: 'Books' },
      update: {},
      create: { name: 'Books' },
    }),
  ]);

  console.log('Seed complete:', { adminUser, categories });
}

main()
  .catch(console.error)
  .finally(() => prisma.$disconnect());
```

---

## 6. Prisma vs TypeORM — Head-to-Head

```
Feature                   TypeORM                     Prisma
────────────────────────────────────────────────────────────────────────
Approach                  Code-first (decorators)     Schema-first (schema.prisma)
Type safety               Good                        Excellent (auto-generated types)
Migration experience      Complex                     Simple (migrate dev is great)
Query builder             Powerful but verbose        Elegant, readable
Raw SQL                   queryRunner.query()         prisma.$queryRaw``
Relation loading          find({ relations })         include: {}
Select specific fields    find({ select })            select: {}
Soft delete               @DeleteDateColumn()         Middleware (manual)
Active Record             Yes (entity methods)        No — always Data Mapper
Transactions              queryRunner or callback     $transaction([]) or callback
Performance               Good                        Similar, better batching
Learning curve            Steep                       Gentler
Database support          Wide                        PostgreSQL, MySQL, SQLite,
                                                      MongoDB, SQL Server
Schema visualization      Read entity files           One schema.prisma file
Generated client          No                          Yes — regenerated on schema change
Studio (data browser)     No                          Yes (prisma studio)
Community                 Large                       Growing rapidly
NestJS integration        @nestjs/typeorm             Manual PrismaService (simple)

CHOOSE TypeORM when:
  ✓ Team is familiar with Active Record / Hibernate patterns
  ✓ You need more database support (Oracle, SAP HANA)
  ✓ Existing codebase uses TypeORM
  ✓ You want entity methods (user.save(), user.remove())

CHOOSE Prisma when:
  ✓ Starting a new project
  ✓ TypeScript type safety is top priority
  ✓ Team prefers schema-first development
  ✓ You want better DX (developer experience)
  ✓ You love 'prisma studio' for database browsing
```

---

## 7. MongoDB & Mongoose — NoSQL in NestJS

### What is MongoDB?

MongoDB is a **document database** — it stores data as JSON-like documents (called BSON) instead of rows in tables. There are no fixed columns — each document can have different fields.

### SQL vs MongoDB conceptual mapping

```
SQL (PostgreSQL)          MongoDB
────────────────────────────────────────
Database                  Database
Table                     Collection
Row                       Document
Column                    Field
JOIN                      $lookup (aggregation) or embed
Primary key               _id (ObjectId by default)
Foreign key               Reference (stored as ObjectId)
Schema                    Flexible (optional validation)

SQL:
┌──────┬──────────┬───────────┐
│ id   │ name     │ email     │
├──────┼──────────┼───────────┤
│ 1    │ Alice    │ a@test.com│
└──────┴──────────┴───────────┘

MongoDB:
{
  "_id": ObjectId("65a1b2c3d4e5f6a7b8c9d0e1"),
  "name": "Alice",
  "email": "a@test.com",
  "address": {
    "city": "Mumbai",
    "country": "India"
  },
  "tags": ["developer", "nodejs"],
  "orders": [
    { "product": "Laptop", "price": 50000 }
  ]
  // No fixed schema — can add any fields at any time
}
```

### When to use MongoDB

```
Use MongoDB when:
  ✓ Data structure varies per record (user profiles, product catalogs)
  ✓ You need to store embedded arrays (comments, tags, line items)
  ✓ Horizontal scaling is critical (sharding is native to MongoDB)
  ✓ Rapidly changing schemas (startups, prototyping)
  ✓ Real-time applications (change streams for live updates)
  ✓ Content management systems
  ✓ Event logs, time-series data, IoT sensor data
  ✓ Catalogs with varying attributes per product type

Don't use MongoDB when:
  ✗ You need complex multi-table transactions
  ✗ Your data is highly relational (lots of JOINs needed)
  ✗ Financial data requiring ACID guarantees (use PostgreSQL)
  ✗ Reporting that needs complex aggregations across many collections
```

### NestJS + Mongoose setup

```typescript
// src/app.module.ts

import { MongooseModule } from '@nestjs/mongoose';
import { ConfigModule, ConfigService } from '@nestjs/config';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),

    MongooseModule.forRootAsync({
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        uri: config.getOrThrow('MONGODB_URI'),
        // MONGODB_URI=mongodb://localhost:27017/myapp
        // OR Atlas: mongodb+srv://user:pass@cluster.mongodb.net/myapp

        // ── Connection options ───────────────────────────────────────────
        dbName: config.get('MONGODB_DB', 'myapp'),
        // Explicitly set database name (can also be in URI)

        // Connection pool
        maxPoolSize: 10,
        // Max connections in pool (similar to PostgreSQL pool)

        serverSelectionTimeoutMS: 5000,
        // Fail fast if can't connect to MongoDB in 5 seconds

        socketTimeoutMS: 45000,
        // How long to wait for a response from MongoDB

        // ── Auto-reconnect ────────────────────────────────────────────────
        autoIndex: config.get('NODE_ENV') !== 'production',
        // Auto-build indexes in development
        // In production: build indexes manually or with ensureIndexes()
        // Automatic indexing on startup can cause performance issues in prod
      }),
    }),
  ],
})
export class AppModule {}
```

---

## 8. Mongoose Schemas & Models

### Creating a Schema with NestJS decorators

```typescript
// src/users/schemas/user.schema.ts

import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { Document, Types } from 'mongoose';
import { Exclude } from 'class-transformer';

// @Schema() decorator: marks this class as a Mongoose schema
@Schema({
  // Collection name in MongoDB (default: lowercase plural of class name → 'users')
  collection: 'users',

  // Timestamps: automatically adds createdAt and updatedAt fields
  timestamps: true,
  // MongoDB adds: createdAt: ISODate, updatedAt: ISODate

  // Versioning: MongoDB's own __v field
  versionKey: false,
  // false: don't add __v (version key) to documents — cleaner output

  // JSON transformation
  toJSON: {
    // Transform document when converting to JSON
    transform: (doc, ret) => {
      ret.id = ret._id.toString();  // Rename _id to id
      delete ret._id;               // Remove _id
      delete ret.__v;               // Remove version key
      delete ret.password;          // Remove password from JSON output
      return ret;
    },
    virtuals: true,  // Include virtual properties in JSON output
  },

  toObject: {
    virtuals: true,  // Include virtuals in object output
  },
})
export class User {
  // ── Basic fields ──────────────────────────────────────────────────────
  @Prop({
    required: true,
    // required: true — field must be present in document
    // Mongoose validation (not the same as class-validator)
    trim: true,
    // trim: true — automatically remove whitespace
    maxlength: 50,
    // maxlength: max string length (Mongoose validation)
    minlength: 2,
  })
  firstName: string;

  @Prop({ required: true, trim: true, maxlength: 50 })
  lastName: string;

  @Prop({
    required: true,
    unique: true,
    // unique: true — creates a unique index in MongoDB
    // MongoDB creates this index automatically
    lowercase: true,
    // lowercase: true — automatically convert to lowercase before saving
    trim: true,
    match: [/^\S+@\S+\.\S+$/, 'Please provide a valid email'],
    // match: regex validation with custom error message
  })
  email: string;

  @Exclude()
  // class-transformer: exclude from JSON serialization
  @Prop({
    required: true,
    select: false,
    // select: false — excluded from queries by default
    // Just like TypeORM's @Column({ select: false })
    minlength: 8,
  })
  password: string;

  // ── Enum field ────────────────────────────────────────────────────────
  @Prop({
    type: String,
    enum: ['user', 'admin', 'moderator'],
    // enum: only these values are allowed
    default: 'user',
  })
  role: string;

  // ── Boolean ───────────────────────────────────────────────────────────
  @Prop({ default: true })
  isActive: boolean;

  // ── Number ────────────────────────────────────────────────────────────
  @Prop({
    type: Number,
    default: 0,
    min: [0, 'Balance cannot be negative'],
    // min/max: range validation with error message
  })
  balance: number;

  // ── Optional field ────────────────────────────────────────────────────
  @Prop({ type: String, default: null })
  phone: string | null;

  // ── Array ─────────────────────────────────────────────────────────────
  @Prop([String])
  // Array of strings — MongoDB stores as native array
  tags: string[];

  // ── Embedded object (subdocument) ─────────────────────────────────────
  @Prop({
    type: {
      street: String,
      city: String,
      state: String,
      country: String,
      pinCode: String,
    },
    // No _id for subdocuments by default
    _id: false,
  })
  address: {
    street: string;
    city: string;
    state: string;
    country: string;
    pinCode: string;
  };

  // ── Reference to another collection ───────────────────────────────────
  @Prop({
    type: Types.ObjectId,
    // ObjectId: MongoDB's unique document identifier
    ref: 'Profile',
    // ref: name of the referenced model
    // Enables .populate('profile') to join documents
  })
  profile: Types.ObjectId;

  // Array of references
  @Prop([{ type: Types.ObjectId, ref: 'Order' }])
  orders: Types.ObjectId[];

  // ── Date ──────────────────────────────────────────────────────────────
  @Prop({ type: Date })
  lastLoginAt: Date;

  @Prop({ type: Date, default: null })
  deletedAt: Date | null;

  // ── Metadata (flexible schema) ────────────────────────────────────────
  @Prop({ type: Object })
  // type: Object — allows any object structure
  // This is MongoDB's document flexibility — no fixed schema
  metadata: Record<string, any>;
}

// Create the Mongoose Schema from the class
export const UserSchema = SchemaFactory.createForClass(User);
// SchemaFactory reads the @Prop() decorators and creates a Mongoose schema

// ── Indexes ───────────────────────────────────────────────────────────────
// Add indexes after schema creation for more control
UserSchema.index({ email: 1 }, { unique: true });
// 1 = ascending, -1 = descending

UserSchema.index({ firstName: 'text', lastName: 'text' });
// Text index for full-text search: db.users.find({ $text: { $search: 'alice' } })

UserSchema.index({ createdAt: -1 });
// Index for sorting by newest first

UserSchema.index({ role: 1, isActive: 1 });
// Compound index for common query: WHERE role = ? AND isActive = ?

// ── Virtual property ──────────────────────────────────────────────────────
UserSchema.virtual('fullName').get(function() {
  return `${this.firstName} ${this.lastName}`;
  // Computed property — not stored in database
  // Available when toJSON({ virtuals: true }) or toObject({ virtuals: true })
});

// ── Type alias ────────────────────────────────────────────────────────────
// Mongoose documents extend the Document class
// This gives you TypeScript types for document methods (.save(), .remove(), etc.)
export type UserDocument = User & Document;
// Usage: User class + Mongoose Document methods = UserDocument
```

### Registering schemas in a module

```typescript
// src/users/users.module.ts

import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { User, UserSchema } from './schemas/user.schema';
import { Profile, ProfileSchema } from './schemas/profile.schema';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';

@Module({
  imports: [
    MongooseModule.forFeature([
      // Register schemas in this module
      { name: User.name, schema: UserSchema },
      // User.name = 'User' (class name as string)
      // MongooseModule creates a 'User' model from UserSchema
      // @InjectModel('User') in service injects this model
      
      { name: Profile.name, schema: ProfileSchema },
      // Multiple schemas in the same module
      
      // With custom collection name:
      // { name: User.name, schema: UserSchema, collection: 'app_users' }
    ]),
  ],
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService, MongooseModule],
})
export class UsersModule {}
```

---

## 9. Mongoose CRUD Operations

```typescript
// src/users/users.service.ts — Mongoose version

import { Injectable, NotFoundException } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model, Types, FilterQuery, UpdateQuery } from 'mongoose';
import { User, UserDocument } from './schemas/user.schema';
import { CreateUserDto } from './dto/create-user.dto';
import { UpdateUserDto } from './dto/update-user.dto';

@Injectable()
export class UsersService {
  constructor(
    @InjectModel(User.name)
    // @InjectModel(User.name) = @InjectModel('User')
    // Injects the Mongoose Model for the User collection
    // Model<UserDocument>: fully typed Mongoose model
    private readonly userModel: Model<UserDocument>,
  ) {}

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // FIND OPERATIONS
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  async findAll(): Promise<UserDocument[]> {
    return this.userModel
      .find()
      // find(): SELECT * FROM users (returns all matching documents)
      // find({ isActive: true }): filter condition
      .select('-password -__v')
      // Projection: exclude fields with - prefix
      // '-password': don't include password
      // '+password': include password (when select: false on schema)
      .sort({ createdAt: -1 })
      // sort: -1 descending, 1 ascending
      .exec();
      // exec(): returns a proper Promise (Mongoose queries are thenable but not Promises)
      // Always call .exec() for consistency
  }

  async findWithFilter(
    filter: FilterQuery<UserDocument>,
    // FilterQuery: Mongoose type for query conditions
    // { isActive: true, role: 'admin' } → WHERE is_active = true AND role = 'admin'
  ): Promise<UserDocument[]> {
    return this.userModel
      .find(filter)
      .select('firstName lastName email role createdAt')
      // Positive projection: ONLY include these fields
      .skip(0)
      .limit(20)
      .lean()
      // lean(): return plain JavaScript objects instead of Mongoose Documents
      // 5-10x faster for read-only operations (no mongoose overhead)
      // Downside: no .save(), no virtuals, no document methods
      .exec();
  }

  async findById(id: string): Promise<UserDocument> {
    // Validate ObjectId format before querying
    if (!Types.ObjectId.isValid(id)) {
      throw new NotFoundException(`Invalid ID format: ${id}`);
    }

    const user = await this.userModel
      .findById(id)
      // findById: shorthand for findOne({ _id: id })
      // Automatically converts string to ObjectId
      .populate('profile')
      // populate: fetches the referenced document and replaces ObjectId with it
      // Like a LEFT JOIN in SQL
      // Without populate: { profile: ObjectId('65a1...') }
      // With populate: { profile: { _id: ObjectId('65a1...'), avatar: '...', ... } }
      .exec();

    if (!user) {
      throw new NotFoundException(`User ${id} not found`);
    }

    return user;
  }

  async search(query: string): Promise<UserDocument[]> {
    return this.userModel
      .find({
        $or: [
          // $or: MongoDB OR operator
          { firstName: { $regex: query, $options: 'i' } },
          // $regex: pattern matching, $options: 'i' = case insensitive
          { lastName: { $regex: query, $options: 'i' } },
          { email: { $regex: query, $options: 'i' } },
        ],
      })
      .exec();

    // Alternatively, use text index for better performance:
    // this.userModel.find({ $text: { $search: query } })
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // CREATE OPERATIONS
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  async create(createUserDto: CreateUserDto): Promise<UserDocument> {
    const user = new this.userModel(createUserDto);
    // Creates a new Mongoose document instance
    // Runs schema validation, default values, transforms

    return user.save();
    // save(): INSERT the document into MongoDB
    // Runs pre-save middleware (@BeforeInsert equivalent)
    // Returns the saved document with _id and timestamps

    // Alternative (shorter):
    // return this.userModel.create(createUserDto);
    // create() = new Model() + save() in one call
  }

  async createMany(dtos: CreateUserDto[]): Promise<UserDocument[]> {
    return this.userModel.insertMany(dtos, {
      ordered: false,
      // ordered: false — insert all documents, don't stop on first error
      // ordered: true (default) — stop on first error
    }) as unknown as UserDocument[];
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // UPDATE OPERATIONS
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  async update(id: string, updateUserDto: UpdateUserDto): Promise<UserDocument> {
    const user = await this.userModel
      .findByIdAndUpdate(
        id,
        { $set: updateUserDto },
        // $set: only update specified fields — partial update
        // Without $set: REPLACES the entire document!
        {
          new: true,
          // new: true — return the UPDATED document (not the original)
          // Default: false — returns the document BEFORE update
          runValidators: true,
          // runValidators: run schema validation on update
          // Default: false (validators only run on .save())
        },
      )
      .exec();

    if (!user) throw new NotFoundException(`User ${id} not found`);
    return user;
  }

  async pushToArray(userId: string, tag: string): Promise<UserDocument> {
    return this.userModel
      .findByIdAndUpdate(
        userId,
        { $push: { tags: tag } },
        // $push: add element to array
        // $addToSet: add element only if not already in array (unique)
        // $pull: remove element from array: { $pull: { tags: tag } }
        // $pop: remove first (-1) or last (1) element: { $pop: { tags: 1 } }
        { new: true },
      )
      .exec();
  }

  async incrementBalance(userId: string, amount: number): Promise<UserDocument> {
    return this.userModel
      .findByIdAndUpdate(
        userId,
        { $inc: { balance: amount } },
        // $inc: increment by value (negative = decrement)
        // Atomic: no race conditions
        { new: true },
      )
      .exec();
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // DELETE OPERATIONS
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  async remove(id: string): Promise<UserDocument> {
    const user = await this.userModel.findByIdAndDelete(id).exec();
    if (!user) throw new NotFoundException(`User ${id} not found`);
    return user;
  }

  async softDelete(id: string): Promise<UserDocument> {
    return this.userModel
      .findByIdAndUpdate(id, { $set: { deletedAt: new Date() } }, { new: true })
      .exec();
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // AGGREGATION PIPELINE — MongoDB's GROUP BY equivalent
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  async getOrderStats() {
    return this.userModel.aggregate([
      // Aggregation pipeline: array of stages, each transforms the data

      // Stage 1: $match — filter documents (like WHERE)
      { $match: { isActive: true, deletedAt: null } },

      // Stage 2: $lookup — join another collection (like LEFT JOIN)
      {
        $lookup: {
          from: 'orders',           // Collection to join
          localField: '_id',        // Field in users collection
          foreignField: 'userId',   // Field in orders collection
          as: 'orders',             // Name of the joined array field
        },
      },

      // Stage 3: $unwind — flatten array (one doc per array element)
      // { $unwind: '$orders' },

      // Stage 4: $group — GROUP BY equivalent
      {
        $group: {
          _id: '$role',             // GROUP BY role
          userCount: { $sum: 1 },   // COUNT(*)
          totalBalance: { $sum: '$balance' },   // SUM(balance)
          avgBalance: { $avg: '$balance' },     // AVG(balance)
          orderCount: { $sum: { $size: '$orders' } }, // orders array length
        },
      },

      // Stage 5: $sort — ORDER BY
      { $sort: { userCount: -1 } },

      // Stage 6: $project — SELECT specific fields
      {
        $project: {
          role: '$_id',
          userCount: 1,
          totalBalance: { $round: ['$totalBalance', 2] },
          avgBalance: { $round: ['$avgBalance', 2] },
          _id: 0,  // Exclude _id from output
        },
      },
    ]);
  }
}
```

---

## 10. Mongoose Advanced — Virtuals, Middleware, Population

### Middleware (pre/post hooks)

```typescript
// Add to schema BEFORE SchemaFactory.createForClass()
// OR add directly to the created schema

// Pre-save hook: hash password before saving
UserSchema.pre('save', async function(next) {
  // 'this' refers to the document being saved
  if (!this.isModified('password')) {
    // isModified(): only hash if password field actually changed
    // Without this: password gets re-hashed on every save (even name changes)
    return next();
  }

  const salt = await bcrypt.genSalt(12);
  this.password = await bcrypt.hash(this.password, salt);
  next();
});

// Pre-find hook: automatically exclude soft-deleted documents
UserSchema.pre(/^find/, function(next) {
  // /^find/: regex matches all find operations (find, findOne, findById, etc.)
  // 'this' is the Query object
  this.where({ deletedAt: null });
  // Adds WHERE deleted_at IS NULL to ALL find queries automatically
  next();
});

// Post-save hook: send welcome email after user created
UserSchema.post('save', async function(doc) {
  if (this.isNew) {
    // isNew: true only for newly created documents (first save)
    // Send welcome email
    console.log(`Welcome email sent to ${doc.email}`);
  }
});

// Instance method: custom method on document
UserSchema.methods.comparePassword = async function(candidatePassword: string): Promise<boolean> {
  return bcrypt.compare(candidatePassword, this.password);
};
// Usage: const isValid = await user.comparePassword('plaintext');

// Static method: custom method on Model
UserSchema.statics.findActiveByEmail = function(email: string) {
  return this.findOne({ email, isActive: true });
};
// Usage: const user = await UserModel.findActiveByEmail('alice@test.com');
```

### Advanced populate — joining collections

```typescript
// Simple populate
await this.userModel.findById(id).populate('profile').exec();

// Populate with field selection
await this.userModel.findById(id)
  .populate('profile', 'avatar website -_id')
  // 'profile': field to populate
  // 'avatar website -_id': fields to include/exclude
  .exec();

// Multiple populations
await this.userModel.findById(id)
  .populate('profile')
  .populate({
    path: 'orders',
    match: { status: { $ne: 'CANCELLED' } },
    // Only populate non-cancelled orders
    options: { sort: { createdAt: -1 }, limit: 5 },
    // Sort and limit populated documents
    populate: {
      path: 'items',
      populate: { path: 'product' },
      // Nested populate: orders → items → product
    },
  })
  .exec();

// populate() with virtual (requires schema virtualsWith option)
UserSchema.virtual('postCount', {
  ref: 'Post',
  localField: '_id',
  foreignField: 'authorId',
  count: true,
  // count: true — returns count instead of actual documents
});
// Usage: user.postCount → number of posts by this user
```

---

## 11. When to Choose SQL vs NoSQL

### Decision framework

```
Choose PostgreSQL + TypeORM/Prisma when:
  ✓ Financial data: transactions, accounting, balances
    → ACID compliance critical, no approximations
  ✓ Complex relations: users → orders → items → products → categories
    → JOINs are natural, relations are well-defined
  ✓ Reporting and analytics: aggregations across multiple tables
    → SQL is more expressive for complex analytical queries
  ✓ Data consistency: strict schema enforcement
    → All rows must have the same columns
  ✓ Compliance: GDPR, financial audits
    → Need for complex queries across structured data
  Examples: banking, e-commerce, ERP, CRM

Choose MongoDB + Mongoose when:
  ✓ Variable structure: different fields per document
    → Product catalog where Electronics has voltage, Clothing has size
  ✓ Hierarchical data: naturally nested
    → Blog posts with embedded comments, tags, metadata
  ✓ Horizontal scaling: need to shard across servers
    → MongoDB was built for distributed systems
  ✓ Real-time: need MongoDB change streams
    → Listen to database changes and push to clients via WebSockets
  ✓ Rapid prototyping: schema changes frequently
    → No migrations needed, just update your code
  Examples: CMS, user profiles, product catalogs, IoT, analytics events

Choose BOTH in the same application (polyglot persistence):
  ✓ Store user accounts in PostgreSQL (needs ACID)
  ✓ Store user activity logs in MongoDB (high write, flexible schema)
  ✓ Cache frequently-read data in Redis
  → Different problems need different tools
```

---

## 12. Interview Questions & Answers

**Q1: What is the key philosophical difference between TypeORM and Prisma?**

> "TypeORM is code-first: you write TypeScript classes with decorators, and TypeORM maps them to database tables. The entities ARE the schema definition. Prisma is schema-first: you write a `schema.prisma` file that defines your data models, and Prisma generates a fully-typed `PrismaClient` from it. With TypeORM, the generated SQL can sometimes be surprising. With Prisma, the schema file is the single source of truth, the migration history is clean, and the generated client has perfect TypeScript inference — including knowing exactly which fields are present based on the `select` and `include` options you use. Prisma trades flexibility for better type safety and developer experience."

---

**Q2: What is the difference between findOne() in TypeORM vs findUnique() in Prisma?**

> "Both retrieve a single record matching a condition, but with different constraints. TypeORM's `findOne()` accepts any condition — it's basically `SELECT ... LIMIT 1`. It returns `null` if not found. Prisma's `findUnique()` can only be used with fields marked as `@unique` or `@id` in the schema — Prisma uses this constraint to guarantee at the type level that at most one record matches. For non-unique conditions, Prisma has `findFirst()` which is equivalent to TypeORM's `findOne()`. Prisma also has `findUniqueOrThrow()` and `findFirstOrThrow()` that throw a `PrismaClientKnownRequestError` if not found, eliminating the need for manual null checks."

---

**Q3: How does Mongoose's populate() work and what is its limitation?**

> "Mongoose's `populate()` resolves document references — when a field stores an ObjectId referencing another collection, `populate()` fetches those documents and replaces the ObjectId with the actual data. Under the hood, it runs a second query to the referenced collection and maps the results. The key limitation is that `populate()` is NOT a MongoDB JOIN — MongoDB doesn't have native JOINs in the same sense as SQL. Populate runs N+1 queries: first to fetch the parent documents, then separate queries for each referenced collection. For complex scenarios, the MongoDB aggregation pipeline's `$lookup` stage is more efficient as it does the joining server-side. However, `populate()` is simpler to use for most cases."

---

**Q4: When would you choose MongoDB over PostgreSQL for a Node.js backend?**

> "I'd choose MongoDB when the data model benefits from document structure. The classic example is a product catalog where different product types have completely different attributes — a phone has storage and battery specs, a t-shirt has size and color. In PostgreSQL, you'd need separate tables or a jsonb column. In MongoDB, each product document naturally has its own fields. I'd also choose MongoDB for high-write scenarios like activity logs, IoT sensor data, or analytics events where you're writing millions of records and need horizontal sharding. And for content where data is naturally hierarchical — a blog post with embedded comments and tags maps perfectly to a document. I'd avoid MongoDB for financial systems, complex multi-collection transactions, or data with many-to-many relations that need frequent querying from multiple directions."

---

**Q5: What is a Prisma middleware and how do you use it for soft deletes?**

> "Prisma middleware is a function that intercepts queries before they execute — similar to NestJS interceptors but at the database level. You register it with `prisma.$use(async (params, next) => { ... })`. For soft deletes, the middleware intercepts delete operations and converts them to updates that set a `deletedAt` timestamp instead of running a real DELETE. It also intercepts find operations and adds a `WHERE deletedAt = null` condition automatically. This way, the application code can call `prisma.user.delete()` and `prisma.user.findMany()` normally without worrying about soft delete logic — the middleware handles it transparently. The limitation is that Prisma's middleware API was deprecated in Prisma 5 in favor of the newer Prisma extension system, so in newer projects you'd use `prisma.$extends()` instead."

---

**Q6: What is the _id vs id difference in MongoDB documents?**

> "MongoDB uses `_id` as the primary key field internally — it's always ObjectId type by default, a 12-byte unique identifier encoded as a 24-character hex string. It's automatically created by MongoDB on insert. In Mongoose, `_id` is the actual database field. Many applications expose a cleaner `id` field to API clients by using Mongoose's virtual `id` which maps `_id.toString()` to a string. In the `toJSON` transform, you can rename `_id` to `id` and delete `_id` from the output. With Prisma and MongoDB, Prisma maps `_id` to an `id` field in your schema automatically, so your application code always uses `id` while the database stores `_id`."

---

**Q7: How do Mongoose pre and post middleware hooks differ from TypeORM lifecycle hooks?**

> "Both let you run code before/after database operations, but with different scope. TypeORM lifecycle hooks like `@BeforeInsert()` and `@AfterLoad()` are decorators on entity class methods — they only work with the entity the decorator is on, and they require using `repository.save()` to trigger (direct `update()` queries bypass them). Mongoose hooks are registered on the Schema object with `.pre()` and `.post()`. You can use regex patterns like `/^find/` to match multiple operations at once. Mongoose hooks have access to `this` — in query middleware it's the Query object, in document middleware it's the Document. TypeORM hooks are simpler and more discoverable since they're decorators right on the entity class. Mongoose hooks are more powerful and flexible, especially the ability to intercept all find operations with a single regex hook."

---

## Quick Reference — Day 7 Cheat Sheet

```
PRISMA:
  Setup:  npm install prisma @prisma/client + npx prisma init
  Schema: schema.prisma → @id, @unique, @default, @map, @relation
  Service: extends PrismaClient, $connect() in onModuleInit
  CRUD:   prisma.user.findMany/findUnique/create/update/delete/upsert
  Types:  Prisma.UserCreateInput, Prisma.UserUpdateInput, Prisma.UserWhereInput
  TX:     prisma.$transaction([]) or prisma.$transaction(async tx => {})
  Mig:    npx prisma migrate dev (dev) | npx prisma migrate deploy (prod)

MONGOOSE:
  Setup:  npm install @nestjs/mongoose mongoose
  Schema: @Schema() class + @Prop() decorators + SchemaFactory.createForClass()
  Module: MongooseModule.forRootAsync() + MongooseModule.forFeature([{ name, schema }])
  Model:  @InjectModel(User.name) private userModel: Model<UserDocument>
  CRUD:   find/findById/findOne/create/save/findByIdAndUpdate/findByIdAndDelete
  Operators: $set $inc $push $pull $addToSet $or $and $regex $in $gt $lt
  Populate: .populate('field') — replaces ObjectId with document
  Aggregation: Model.aggregate([{$match}, {$lookup}, {$group}, {$sort}, {$project}])
  Hooks:  Schema.pre('save', fn) | Schema.post('find', fn)
  Lean:   .lean() — plain objects, faster for read-only

SQL vs NoSQL choice:
  PostgreSQL → ACID, relations, financial, reporting, compliance
  MongoDB    → flexible schema, embedded docs, horizontal scale, real-time

TypeORM vs Prisma:
  TypeORM  → code-first, decorators, entity methods, more DB support
  Prisma   → schema-first, generated client, perfect types, better DX
```

---

*Day 7 complete — Week 1 Foundation is done! Tomorrow — Day 8: Authentication with JWT & Passport — Local strategy, JWT strategy, token signing, refresh tokens, bcrypt, and the complete auth flow from login to protected route.*
