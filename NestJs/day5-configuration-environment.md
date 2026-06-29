# Day 5 — Configuration & Environment Management
### NestJS Interview Prep | 3-Year Backend Engineer Level

> **Goal for today:** Master how to manage configuration and environment variables in NestJS professionally. Understand ConfigModule, ConfigService, environment-specific configs, Joi schema validation, typed configuration with `registerAs()`, and secrets management. A 3-year engineer must know how to build apps that are configurable, secure, and deployable across multiple environments without code changes.

---

## 🧭 First-Timer Guidance — Read This First

**The core problem this day solves:**

Every real application runs in multiple environments:
- **Local** — your laptop, connects to a local database
- **Development/Staging** — shared server, connects to a staging database
- **Production** — live servers, connects to production database

The code is identical in all environments. Only the **configuration** changes — database URLs, API keys, ports, feature flags, JWT secrets.

**The wrong way (hardcoding):**
```typescript
// ❌ NEVER do this — hardcoded, insecure, impossible to change without deploying
const dbHost = 'localhost';
const jwtSecret = 'my-super-secret-key-12345';
const stripeKey = 'sk_live_abc123verysecretkey';
```

If you push this to GitHub, your API keys are exposed. If you need to change the database, you must redeploy. This is how production incidents happen.

**The right way (environment variables):**
```bash
# .env file — never commit to git
DB_HOST=localhost
JWT_SECRET=my-super-secret-key-12345
STRIPE_KEY=sk_live_abc123verysecretkey
```

```typescript
// ✅ Read from environment at runtime
const dbHost = process.env.DB_HOST;
```

**What NestJS's `@nestjs/config` adds on top:**
- Loads `.env` files automatically
- Validates that required variables exist at startup (fail fast)
- Provides type-safe access via `ConfigService`
- Namespaces config into logical groups (`database`, `jwt`, `stripe`)
- Works across all environments (Docker, Kubernetes, CI/CD)

**Setup — install first:**
```bash
npm install @nestjs/config
npm install joi  # For schema validation
```

---

## Table of Contents

1. [Environment Variables — The Foundation](#1-environment-variables--the-foundation)
2. [ConfigModule — Loading Configuration](#2-configmodule--loading-configuration)
3. [ConfigService — Reading Configuration](#3-configservice--reading-configuration)
4. [Joi Validation — Fail Fast at Startup](#4-joi-validation--fail-fast-at-startup)
5. [registerAs() — Namespaced Typed Config](#5-registeras--namespaced-typed-config)
6. [Multiple Environment Files](#6-multiple-environment-files)
7. [Custom Config Files — No .env](#7-custom-config-files--no-env)
8. [Using Config in main.ts (Before App Starts)](#8-using-config-in-maints-before-app-starts)
9. [Secrets Management — Production Patterns](#9-secrets-management--production-patterns)
10. [Complete Real-World Configuration Setup](#10-complete-real-world-configuration-setup)
11. [Interview Questions & Answers](#11-interview-questions--answers)

---

## 1. Environment Variables — The Foundation

### What are environment variables?
Environment variables are key-value pairs stored **outside your code** in the operating system or a `.env` file. They are injected into your Node.js process at runtime via `process.env`.

### The .env file

```bash
# .env — root of your project
# This file is loaded by @nestjs/config
# NEVER commit this to git — add to .gitignore

# ── Application ────────────────────────────────────────────────────────
NODE_ENV=development
PORT=3000
API_VERSION=v1
APP_NAME=MyNestApp
GLOBAL_PREFIX=api

# ── Database ────────────────────────────────────────────────────────────
DB_TYPE=postgres
DB_HOST=localhost
DB_PORT=5432
DB_USERNAME=postgres
DB_PASSWORD=your_password_here
DB_DATABASE=myapp_dev
DB_SYNCHRONIZE=false
DB_LOGGING=true
DB_POOL_SIZE=10

# ── JWT Authentication ──────────────────────────────────────────────────
JWT_SECRET=your-256-bit-secret-minimum-32-characters-long
JWT_EXPIRES_IN=15m
JWT_REFRESH_SECRET=another-256-bit-secret-for-refresh-tokens
JWT_REFRESH_EXPIRES_IN=7d

# ── Redis (Caching & Sessions) ──────────────────────────────────────────
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_TTL=300

# ── Email Service ────────────────────────────────────────────────────────
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USER=your-email@gmail.com
MAIL_PASSWORD=your-app-password
MAIL_FROM=noreply@myapp.com

# ── External APIs ────────────────────────────────────────────────────────
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
AWS_S3_BUCKET=my-app-bucket
AWS_REGION=ap-south-1

# ── Throttling / Rate Limiting ───────────────────────────────────────────
THROTTLE_TTL=60
THROTTLE_LIMIT=100

# ── CORS ─────────────────────────────────────────────────────────────────
ALLOWED_ORIGINS=http://localhost:3001,https://myapp.com
```

### .env.example — the file you DO commit to git

```bash
# .env.example — ALWAYS commit this
# Shows teammates what variables are needed, but with NO real values

NODE_ENV=development
PORT=3000
API_VERSION=v1

DB_HOST=localhost
DB_PORT=5432
DB_USERNAME=postgres
DB_PASSWORD=
DB_DATABASE=myapp_dev

JWT_SECRET=
JWT_EXPIRES_IN=15m
JWT_REFRESH_SECRET=
JWT_REFRESH_EXPIRES_IN=7d

REDIS_HOST=localhost
REDIS_PORT=6379

MAIL_HOST=
MAIL_PORT=587
MAIL_USER=
MAIL_PASSWORD=

STRIPE_SECRET_KEY=
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_S3_BUCKET=
AWS_REGION=ap-south-1
```

### .gitignore — protect your secrets

```bash
# .gitignore — add these
.env
.env.local
.env.development.local
.env.test.local
.env.production.local

# BUT commit these:
# .env.example      ← template with empty values
# .env.test         ← if it contains no real secrets
```

### process.env — how Node.js reads env vars

```typescript
// Without @nestjs/config, you'd read like this:
const port = process.env.PORT;
// Problem: process.env.PORT is ALWAYS a string (or undefined)
// process.env.DB_PORT === '5432'  (string, not number)
// process.env.DB_SYNCHRONIZE === 'false'  (string, not boolean)

// You'd need to manually convert:
const port = parseInt(process.env.PORT ?? '3000', 10);
const dbSync = process.env.DB_SYNCHRONIZE === 'true';

// And handle missing variables:
if (!process.env.JWT_SECRET) {
  throw new Error('JWT_SECRET is required');
}

// @nestjs/config handles ALL of this for you cleanly
```

---

## 2. ConfigModule — Loading Configuration

### Basic setup

```typescript
// src/app.module.ts

import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [
    ConfigModule.forRoot({
      // ── isGlobal ──────────────────────────────────────────────────────
      isGlobal: true,
      // Makes ConfigModule available EVERYWHERE without importing it
      // in each feature module.
      // Without isGlobal: you'd need to import ConfigModule in every
      // module that uses ConfigService → very tedious
      // With isGlobal: import once in AppModule, use everywhere

      // ── envFilePath ───────────────────────────────────────────────────
      envFilePath: '.env',
      // Which file to load. Default is '.env' in root directory.
      // Can be string or array: ['.env.local', '.env']
      // Array: files on the LEFT take precedence over files on the RIGHT

      // ── ignoreEnvFile ─────────────────────────────────────────────────
      ignoreEnvFile: process.env.NODE_ENV === 'production',
      // In production, DON'T load .env files
      // Production env vars are injected by the deployment system
      // (Docker env, Kubernetes secrets, AWS Parameter Store)
      // .env files in production = security risk (file on disk)

      // ── expandVariables ───────────────────────────────────────────────
      expandVariables: true,
      // Allows referencing other env vars in values:
      // DB_URL=${DB_TYPE}://${DB_USERNAME}:${DB_PASSWORD}@${DB_HOST}:${DB_PORT}/${DB_DATABASE}
      // Without expandVariables, the ${} would be treated as literal text
    }),
  ],
})
export class AppModule {}
```

### How ConfigModule.forRoot() works internally

```typescript
// When ConfigModule.forRoot() runs, it does this:

// 1. Reads process.env (system environment variables)
// 2. Parses the .env file using 'dotenv' library
// 3. Merges them: process.env takes precedence over .env file
//    (System env vars always win — important for Docker/Kubernetes)
// 4. Stores the merged config object
// 5. Makes ConfigService available for injection everywhere

// Priority order (highest to lowest):
// 1. Actual system environment variables (export PORT=4000 in terminal)
// 2. .env.local file
// 3. .env file
// 4. Default values in ConfigService.get()

// This means: same .env file, but system env overrides specific values
// Production system sets DB_HOST=prod-db.aws.com
// .env file has DB_HOST=localhost
// → ConfigService.get('DB_HOST') returns 'prod-db.aws.com' ✅
```

---

## 3. ConfigService — Reading Configuration

### Injecting and using ConfigService

```typescript
// src/users/users.service.ts

import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class UsersService {
  constructor(
    private readonly configService: ConfigService,
    // NestJS injects ConfigService automatically
    // Available because ConfigModule.forRoot({ isGlobal: true })
  ) {}

  someMethod() {
    // ── Basic get ──────────────────────────────────────────────────────
    const port = this.configService.get('PORT');
    // Returns: string | undefined
    // Returns undefined if PORT is not set — dangerous!

    // ── Get with default value ─────────────────────────────────────────
    const port2 = this.configService.get('PORT', 3000);
    // Returns: PORT from env, OR 3000 if PORT is not set
    // Second argument = default value (fallback)

    // ── Get with TypeScript type parameter ─────────────────────────────
    const port3 = this.configService.get<number>('PORT', 3000);
    // TypeScript knows this is a number
    // But: this is just a TYPE ASSERTION, not actual conversion
    // process.env values are STILL strings unless you use registerAs()

    // ── getOrThrow — strict mode (RECOMMENDED for required vars) ────────
    const jwtSecret = this.configService.getOrThrow<string>('JWT_SECRET');
    // If JWT_SECRET is not set → throws MissingEnvVarException at runtime
    // Fail loud and early rather than silently using undefined
    // Use getOrThrow for ALL required configuration
  }
}
```

### The type problem — and how to solve it

```typescript
// THE PROBLEM:
// process.env values are ALWAYS strings
// this.configService.get<number>('PORT') is just a TypeScript lie
// At runtime, PORT is still the string '3000', not number 3000

@Injectable()
export class DatabaseService {
  private dbPort: number;

  constructor(private readonly configService: ConfigService) {
    // ❌ Wrong — type assertion but still a string at runtime
    this.dbPort = this.configService.get<number>('DB_PORT');
    // this.dbPort is '5432' (string) at runtime, even though TypeScript thinks it's number
    // console.log(typeof this.dbPort) → 'string' 😱

    // ✅ Correct — manually parse
    this.dbPort = parseInt(this.configService.get<string>('DB_PORT', '5432'), 10);
    // Now it's actually a number

    // ✅ Even better — use registerAs() for automatic typing (covered in section 5)
  }
}

// Real-world usage — always parse primitive types manually:
getConfig() {
  return {
    port: parseInt(this.configService.getOrThrow('PORT'), 10),
    // INT: always parseInt

    dbSync: this.configService.get('DB_SYNCHRONIZE') === 'true',
    // BOOLEAN: compare string to 'true'

    poolSize: parseInt(this.configService.get('DB_POOL_SIZE', '10'), 10),
    // INT with default

    debug: this.configService.get('DEBUG', 'false') === 'true',
    // BOOLEAN with default

    allowedOrigins: this.configService.get('ALLOWED_ORIGINS', '').split(','),
    // ARRAY: split comma-separated string
  };
}
```

### Using ConfigService in TypeORM setup

```typescript
// src/app.module.ts
// Database configuration that reads from environment

import { TypeOrmModule } from '@nestjs/typeorm';
import { ConfigModule, ConfigService } from '@nestjs/config';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),

    // TypeOrmModule.forRootAsync — async factory that can inject ConfigService
    TypeOrmModule.forRootAsync({
      // inject tells NestJS what to inject into the factory function
      inject: [ConfigService],

      // useFactory receives the injected services as arguments
      useFactory: (configService: ConfigService) => ({
        type: configService.getOrThrow<'postgres'>('DB_TYPE'),
        // 'postgres' | 'mysql' | 'sqlite' etc.

        host: configService.getOrThrow<string>('DB_HOST'),
        port: parseInt(configService.getOrThrow<string>('DB_PORT'), 10),
        username: configService.getOrThrow<string>('DB_USERNAME'),
        password: configService.getOrThrow<string>('DB_PASSWORD'),
        database: configService.getOrThrow<string>('DB_DATABASE'),

        entities: [__dirname + '/**/*.entity{.ts,.js}'],
        // Glob pattern: auto-discover all .entity.ts files

        synchronize: configService.get('DB_SYNCHRONIZE') === 'true',
        // NEVER true in production — destroys/recreates tables
        // true only for rapid local development
        // Production: use migrations (covered Day 6)

        logging: configService.get('NODE_ENV') !== 'production',
        // Log SQL queries in development, not in production (too noisy)

        ssl: configService.get('NODE_ENV') === 'production'
          ? { rejectUnauthorized: false }  // Required by AWS RDS, Heroku Postgres
          : false,
        // SSL required by most cloud database providers

        extra: {
          max: parseInt(configService.get('DB_POOL_SIZE', '10'), 10),
          // Connection pool size
          // Development: 5-10, Production: 20-50 depending on traffic
        },
      }),
    }),
  ],
})
export class AppModule {}
```

---

## 4. Joi Validation — Fail Fast at Startup

### Why validate environment variables?

Without validation, your app starts even with missing critical variables. The error happens later — when a request hits the code that needs the missing variable. In production, this could mean:
- App starts successfully
- First user tries to log in
- JWT_SECRET is missing → crash → 500 error
- Users can't log in for minutes until you notice

With Joi validation, the app **refuses to start** if any required variable is missing or invalid. You catch the problem immediately on deployment, not when users are already affected.

```bash
# With Joi validation — app refuses to start:
[ConfigModule] Error: Config validation error: "JWT_SECRET" is required
               at Object.<anonymous> (/app/src/main.ts:10:1)

# You immediately know what's wrong — fix before any user is affected
```

### Setting up Joi validation

```typescript
// src/app.module.ts

import * as Joi from 'joi';
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      envFilePath: '.env',

      validationSchema: Joi.object({
        // ── Application ──────────────────────────────────────────────────

        NODE_ENV: Joi.string()
          .valid('development', 'staging', 'production', 'test')
          // Only allow these specific values
          .default('development'),
          // Default: if not set, use 'development'

        PORT: Joi.number()
          .integer()
          .min(1024)    // Port must be >= 1024 (below = system ports)
          .max(65535)   // Port must be <= 65535
          .default(3000),
          // Joi handles the string→number conversion for you here!

        // ── Database ────────────────────────────────────────────────────

        DB_HOST: Joi.string()
          .hostname()   // Must be a valid hostname
          .required(),  // REQUIRED — no default, must be set

        DB_PORT: Joi.number()
          .integer()
          .default(5432),

        DB_USERNAME: Joi.string()
          .min(1)
          .required(),

        DB_PASSWORD: Joi.string()
          .min(1)
          .required()
          .when('NODE_ENV', {
            is: 'production',
            then: Joi.string().min(12),
            // In production, password must be at least 12 characters
            otherwise: Joi.string().min(1),
            // Development allows any password
          }),

        DB_DATABASE: Joi.string().required(),

        DB_SYNCHRONIZE: Joi.boolean()
          .default(false),
          // Joi automatically converts 'true'/'false' strings to booleans

        // ── JWT ──────────────────────────────────────────────────────────

        JWT_SECRET: Joi.string()
          .min(32)   // Minimum 32 characters for security
          .required(),

        JWT_EXPIRES_IN: Joi.string()
          .pattern(/^\d+[smhd]$/)
          // Regex: digit(s) followed by s/m/h/d (seconds/minutes/hours/days)
          // Valid: '15m', '1h', '7d', '3600s'
          .default('15m'),

        JWT_REFRESH_SECRET: Joi.string()
          .min(32)
          .required(),

        JWT_REFRESH_EXPIRES_IN: Joi.string()
          .default('7d'),

        // ── Redis ────────────────────────────────────────────────────────

        REDIS_HOST: Joi.string()
          .hostname()
          .default('localhost'),

        REDIS_PORT: Joi.number()
          .integer()
          .default(6379),

        REDIS_PASSWORD: Joi.string()
          .allow('')    // Empty string is allowed (no auth in development)
          .optional(),

        // ── Throttling ───────────────────────────────────────────────────

        THROTTLE_TTL: Joi.number()
          .integer()
          .positive()
          .default(60),   // 60 seconds window

        THROTTLE_LIMIT: Joi.number()
          .integer()
          .positive()
          .default(100),  // 100 requests per window

        // ── Email ────────────────────────────────────────────────────────

        MAIL_HOST: Joi.string()
          .hostname()
          .optional(),

        MAIL_PORT: Joi.number()
          .integer()
          .valid(25, 465, 587, 2525)  // Only allow standard SMTP ports
          .default(587),

        MAIL_USER: Joi.string()
          .email()   // Must be a valid email
          .optional(),

        MAIL_PASSWORD: Joi.string()
          .optional(),

        // ── AWS ──────────────────────────────────────────────────────────

        AWS_ACCESS_KEY_ID: Joi.string()
          .pattern(/^AKIA[A-Z0-9]{16}$/)
          // AWS access keys always start with AKIA and are 20 chars total
          .optional(),

        AWS_SECRET_ACCESS_KEY: Joi.string()
          .length(40)    // AWS secret keys are always 40 characters
          .optional(),

        AWS_S3_BUCKET: Joi.string().optional(),

        AWS_REGION: Joi.string()
          .valid(
            'us-east-1', 'us-west-2', 'eu-west-1', 'ap-south-1',
            // Add other regions your app might use
          )
          .default('ap-south-1'),
      }),

      validationOptions: {
        allowUnknown: true,
        // Allow environment variables not defined in the schema
        // Without this: ANY extra env var causes validation to fail
        // Example: TERM, USER, PATH, HOME are set by the OS → would fail
        // Always set to true

        abortEarly: false,
        // Validate ALL variables and report ALL errors at once
        // Without this: stops at first error, you fix it, then find the next one
        // With this: shows ALL missing/invalid vars in one error message
      },
    }),
  ],
})
export class AppModule {}
```

---

## 5. registerAs() — Namespaced Typed Config

### The problem with flat config

```typescript
// Without namespacing — all config is flat
this.configService.get('DB_HOST')
this.configService.get('DB_PORT')
this.configService.get('DB_USERNAME')
this.configService.get('DB_PASSWORD')
// Problems:
// 1. Typos in string keys ('DB_HOST' vs 'DB-HOST') — no type checking
// 2. No grouping — database, JWT, email all mixed together
// 3. You have to manually parse types (parseInt, === 'true')
// 4. No autocomplete in IDE
```

### registerAs() — the solution

`registerAs()` creates a **namespace** for related config values. You read the whole namespace at once as a typed object. This is the MOST IMPORTANT pattern for professional NestJS config.

```typescript
// src/config/database.config.ts

import { registerAs } from '@nestjs/config';

// registerAs('namespace', factory) creates a configuration namespace
// The factory function runs ONCE at startup, reads env vars, returns typed config
export default registerAs('database', () => ({
  // The factory returns a plain object
  // Every value here is properly typed from the start
  
  type: process.env.DB_TYPE as 'postgres' | 'mysql' | 'sqlite',
  // Explicitly typed — TypeScript knows this is a database type

  host: process.env.DB_HOST || 'localhost',
  // String — with fallback

  port: parseInt(process.env.DB_PORT ?? '5432', 10),
  // parseInt HERE in the factory — so consumers get a number, not string

  username: process.env.DB_USERNAME || 'postgres',
  password: process.env.DB_PASSWORD || '',
  database: process.env.DB_DATABASE || 'myapp',

  synchronize: process.env.DB_SYNCHRONIZE === 'true',
  // Boolean — properly converted

  logging: process.env.NODE_ENV !== 'production',
  // Derived boolean — computed from another env var

  ssl: process.env.NODE_ENV === 'production'
    ? { rejectUnauthorized: false }
    : false,

  poolSize: parseInt(process.env.DB_POOL_SIZE ?? '10', 10),
}));

// The TypeScript type is inferred automatically from what the factory returns
// TypeScript knows: { type: string; host: string; port: number; synchronize: boolean; ... }
```

```typescript
// src/config/jwt.config.ts

import { registerAs } from '@nestjs/config';

export default registerAs('jwt', () => ({
  secret: process.env.JWT_SECRET,
  // In production: throw if missing (Joi validation catches this at startup)

  expiresIn: process.env.JWT_EXPIRES_IN || '15m',

  refreshSecret: process.env.JWT_REFRESH_SECRET,

  refreshExpiresIn: process.env.JWT_REFRESH_EXPIRES_IN || '7d',
}));
```

```typescript
// src/config/app.config.ts

import { registerAs } from '@nestjs/config';

export default registerAs('app', () => ({
  name: process.env.APP_NAME || 'MyNestApp',
  env: process.env.NODE_ENV || 'development',
  port: parseInt(process.env.PORT ?? '3000', 10),
  globalPrefix: process.env.GLOBAL_PREFIX || 'api',
  apiVersion: process.env.API_VERSION || 'v1',
  
  allowedOrigins: process.env.ALLOWED_ORIGINS
    ? process.env.ALLOWED_ORIGINS.split(',').map(origin => origin.trim())
    : ['http://localhost:3001'],

  throttle: {
    ttl: parseInt(process.env.THROTTLE_TTL ?? '60', 10),
    limit: parseInt(process.env.THROTTLE_LIMIT ?? '100', 10),
  },

  isDevelopment: process.env.NODE_ENV === 'development',
  isProduction: process.env.NODE_ENV === 'production',
  isTest: process.env.NODE_ENV === 'test',
}));
```

```typescript
// src/config/mail.config.ts

import { registerAs } from '@nestjs/config';

export default registerAs('mail', () => ({
  host: process.env.MAIL_HOST || 'smtp.gmail.com',
  port: parseInt(process.env.MAIL_PORT ?? '587', 10),
  secure: process.env.MAIL_PORT === '465',
  // SMTP port 465 = SSL/TLS (secure: true), 587 = STARTTLS (secure: false)
  auth: {
    user: process.env.MAIL_USER,
    pass: process.env.MAIL_PASSWORD,
  },
  from: process.env.MAIL_FROM || 'noreply@myapp.com',
  fromName: process.env.MAIL_FROM_NAME || 'MyApp',
}));
```

### Loading registered configs in AppModule

```typescript
// src/app.module.ts

import { ConfigModule } from '@nestjs/config';
import databaseConfig from './config/database.config';
import jwtConfig from './config/jwt.config';
import appConfig from './config/app.config';
import mailConfig from './config/mail.config';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      load: [
        databaseConfig,   // Registers 'database' namespace
        jwtConfig,        // Registers 'jwt' namespace
        appConfig,        // Registers 'app' namespace
        mailConfig,       // Registers 'mail' namespace
      ],
      // Joi validation schema still works alongside load[]
      validationSchema: Joi.object({ /* ... */ }),
    }),
  ],
})
export class AppModule {}
```

### Reading namespaced config — two approaches

```typescript
// ── Approach 1: Using ConfigService.get() with namespace path ──────────

@Injectable()
export class AuthService {
  constructor(private readonly configService: ConfigService) {}

  getJwtSecret(): string {
    // Dot notation: 'namespace.property'
    return this.configService.get<string>('jwt.secret');
    // Returns the 'secret' property from the 'jwt' namespace

    // Get the whole namespace object:
    const jwtConfig = this.configService.get('jwt');
    // { secret: '...', expiresIn: '15m', refreshSecret: '...', ... }

    // Nested:
    const throttleLimit = this.configService.get<number>('app.throttle.limit');
  }
}


// ── Approach 2: @InjectConfig() — the BEST approach for typed config ───

// First, create a TypeScript interface for your config
export interface DatabaseConfig {
  type: 'postgres' | 'mysql' | 'sqlite';
  host: string;
  port: number;
  username: string;
  password: string;
  database: string;
  synchronize: boolean;
  logging: boolean;
  poolSize: number;
}

// In your service, inject the config directly using the namespace token
@Injectable()
export class DatabaseService {
  constructor(
    @Inject(databaseConfig.KEY)
    // databaseConfig.KEY is a unique symbol created by registerAs()
    // This injects the entire 'database' namespace config as a typed object
    private readonly dbConfig: DatabaseConfig,
  ) {}

  getConnectionString(): string {
    // Full TypeScript autocomplete and type safety!
    const { host, port, username, password, database } = this.dbConfig;
    // host → string ✅
    // port → number ✅ (already parsed in the config factory)
    // synchronize → boolean ✅

    return `postgresql://${username}:${password}@${host}:${port}/${database}`;
  }
}
```

### TypeORM with namespaced config

```typescript
// src/app.module.ts

import { TypeOrmModule } from '@nestjs/typeorm';
import { ConfigModule, ConfigService } from '@nestjs/config';
import databaseConfig from './config/database.config';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true, load: [databaseConfig] }),

    TypeOrmModule.forRootAsync({
      inject: [ConfigService],
      useFactory: (configService: ConfigService) => {
        const db = configService.get('database');
        // Gets the ENTIRE database namespace as an object — all properly typed
        
        return {
          type: db.type,
          host: db.host,
          port: db.port,         // Already a number from the factory
          username: db.username,
          password: db.password,
          database: db.database,
          synchronize: db.synchronize,  // Already a boolean
          logging: db.logging,
          ssl: db.ssl,
          extra: { max: db.poolSize },
          autoLoadEntities: true,
        };
      },
    }),
  ],
})
export class AppModule {}
```

---

## 6. Multiple Environment Files

### Environment-specific .env files

```bash
# File naming convention:
.env                    # Base defaults (always loaded)
.env.development        # Development overrides
.env.staging            # Staging overrides
.env.production         # Production overrides (usually empty — use system env)
.env.test               # Test environment (loaded during 'npm run test')
```

```typescript
// src/app.module.ts

ConfigModule.forRoot({
  isGlobal: true,

  // Load environment-specific file + base file
  // Files listed FIRST have HIGHER priority
  envFilePath: [
    `.env.${process.env.NODE_ENV || 'development'}`,
    // e.g., .env.development or .env.production
    // This file overrides the base .env below

    '.env',
    // Base file with defaults
    // Values here are ONLY used if not in the environment-specific file
  ],

  // In production, DON'T read .env files — use system environment
  ignoreEnvFile: process.env.NODE_ENV === 'production',
})
```

```bash
# .env (base defaults — committed to git with safe defaults)
NODE_ENV=development
PORT=3000
DB_PORT=5432
DB_HOST=localhost
DB_SYNCHRONIZE=false
THROTTLE_TTL=60
THROTTLE_LIMIT=100
```

```bash
# .env.development (local development — NOT committed)
DB_USERNAME=postgres
DB_PASSWORD=localpassword
DB_DATABASE=myapp_dev
DB_SYNCHRONIZE=true
DB_LOGGING=true
JWT_SECRET=development-only-secret-not-for-production
JWT_REFRESH_SECRET=development-refresh-secret
```

```bash
# .env.test (test environment — can be committed if no real secrets)
DB_DATABASE=myapp_test
DB_SYNCHRONIZE=true
JWT_SECRET=test-secret-key-not-for-production-use
JWT_REFRESH_SECRET=test-refresh-secret
THROTTLE_LIMIT=1000
# High throttle limit for tests — don't want tests to be rate-limited
```

```bash
# .env.staging (staging server — NOT committed, stored in CI/CD secrets)
DB_HOST=staging-db.internal
DB_USERNAME=staging_user
DB_PASSWORD=staging_password_very_long
DB_DATABASE=myapp_staging
JWT_SECRET=staging-256-bit-random-secret-here
```

### Loading test config automatically

```typescript
// jest.config.js — tell Jest to set NODE_ENV before running tests

module.exports = {
  moduleFileExtensions: ['js', 'json', 'ts'],
  rootDir: 'src',
  testRegex: '.*\\.spec\\.ts$',
  transform: { '^.+\\.(t|j)s$': 'ts-jest' },
  testEnvironment: 'node',
  
  // This is how .env.test gets loaded automatically in tests
  setupFiles: ['<rootDir>/../test/setup.ts'],
};

// test/setup.ts
import * as dotenv from 'dotenv';
dotenv.config({ path: '.env.test' });
// Loads .env.test before any test file runs
```

---

## 7. Custom Config Files — No .env

### When you might NOT use .env files

In some teams and environments, configuration comes from:
- JSON config files checked into git (non-sensitive configs)
- YAML files
- Remote configuration services (AWS AppConfig, Consul, etc.)
- Hard-coded objects (feature flags)

```typescript
// src/config/features.config.ts
// Feature flags — can be committed to git (no secrets)

import { registerAs } from '@nestjs/config';

export default registerAs('features', () => ({
  // Feature flags for gradual rollouts
  enableNewDashboard: process.env.FEATURE_NEW_DASHBOARD === 'true' || false,
  enableBetaApi: process.env.FEATURE_BETA_API === 'true' || false,
  maxUploadSize: 10 * 1024 * 1024,  // 10MB — hardcoded is fine for this
  
  supportedLanguages: ['en', 'hi', 'ta', 'te', 'mr'],
  defaultLanguage: 'en',
  
  pagination: {
    defaultLimit: 20,
    maxLimit: 100,
  },
  
  cache: {
    userProfileTtl: 300,       // 5 minutes
    productListTtl: 60,        // 1 minute
    staticDataTtl: 3600,       // 1 hour
  },
}));

// src/config/cors.config.ts
// CORS configuration — can be more complex than a single env var

export default registerAs('cors', () => {
  const allowedOrigins = process.env.ALLOWED_ORIGINS
    ? process.env.ALLOWED_ORIGINS.split(',').map(o => o.trim())
    : [];

  // Always allow localhost in development
  if (process.env.NODE_ENV === 'development') {
    allowedOrigins.push('http://localhost:3001', 'http://localhost:3002');
  }

  return {
    origins: allowedOrigins,
    methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
    allowedHeaders: ['Content-Type', 'Authorization', 'X-Request-Id'],
    exposedHeaders: ['X-Total-Count', 'X-Request-Id'],
    credentials: true,
    maxAge: 86400,  // Preflight cache: 24 hours
  };
});
```

---

## 8. Using Config in main.ts (Before App Starts)

### The challenge: main.ts runs BEFORE NestJS DI is ready

```typescript
// Problem: main.ts runs before the NestJS container is built
// You can't inject ConfigService here — the DI container doesn't exist yet

// ❌ This doesn't work:
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  const configService = new ConfigService();  // Wrong! Can't instantiate like this
  const port = configService.get('PORT');
}

// ✅ Solution 1: app.get() — retrieve from already-built container
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  // At this point, NestJS has built the container
  // app.get() retrieves providers from the container

  const configService = app.get(ConfigService);
  // Gets the ConfigService that was configured with all your env vars

  const port = configService.getOrThrow<number>('PORT');
  const globalPrefix = configService.get('GLOBAL_PREFIX', 'api');

  app.setGlobalPrefix(globalPrefix);
  
  await app.listen(port);
  console.log(`Application running on port ${port}`);
}

// ✅ Solution 2: process.env directly for startup config (simpler)
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // For simple startup configuration, just read process.env directly
  // By this point, ConfigModule has already loaded .env into process.env
  const port = parseInt(process.env.PORT ?? '3000', 10);
  const prefix = process.env.GLOBAL_PREFIX || 'api';

  app.setGlobalPrefix(prefix);
  await app.listen(port);
}
```

### Full professional main.ts

```typescript
// src/main.ts

import { NestFactory } from '@nestjs/core';
import { ValidationPipe, Logger, VersioningType } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { AppModule } from './app.module';
import * as helmet from 'helmet';
import * as compression from 'compression';

async function bootstrap() {
  // Create a Logger before the app (works without DI)
  const logger = new Logger('Bootstrap');

  // Create the NestJS application
  // ConfigModule.forRoot() runs immediately — .env is loaded here
  const app = await NestFactory.create(AppModule, {
    logger: ['error', 'warn', 'log'],
    // Only show these log levels. In development add 'debug', 'verbose'
    bufferLogs: true,
    // Buffer logs during startup — use custom logger once app is ready
  });

  // Retrieve ConfigService from the built container
  const configService = app.get(ConfigService);

  // ── Security Middleware ────────────────────────────────────────────────
  app.use(helmet());
  // Adds security headers: CSP, HSTS, X-Frame-Options, etc.

  // ── Compression ───────────────────────────────────────────────────────
  app.use(compression());
  // Gzip compress all responses > 1KB

  // ── CORS ──────────────────────────────────────────────────────────────
  const allowedOrigins = configService.get<string>('ALLOWED_ORIGINS', '')
    .split(',')
    .map(o => o.trim())
    .filter(Boolean);

  app.enableCors({
    origin: allowedOrigins.length ? allowedOrigins : '*',
    credentials: true,
    methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
    allowedHeaders: ['Content-Type', 'Authorization'],
  });

  // ── API Versioning ────────────────────────────────────────────────────
  app.enableVersioning({
    type: VersioningType.URI,
    // URI versioning: /v1/users, /v2/users
    // Alternative: HEADER (X-API-Version: 1) or MEDIA_TYPE
    defaultVersion: '1',
  });

  // ── Global Prefix ─────────────────────────────────────────────────────
  const globalPrefix = configService.get('GLOBAL_PREFIX', 'api');
  app.setGlobalPrefix(globalPrefix);
  // All routes: /api/v1/users, /api/v1/products, etc.

  // ── Global Validation ─────────────────────────────────────────────────
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      forbidNonWhitelisted: true,
      transform: true,
      transformOptions: { enableImplicitConversion: true },
    }),
  );

  // ── Start Server ──────────────────────────────────────────────────────
  const port = configService.get<number>('PORT', 3000);
  const nodeEnv = configService.get<string>('NODE_ENV', 'development');

  await app.listen(port);

  logger.log(`━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━`);
  logger.log(`🚀 Application is running!`);
  logger.log(`📡 Environment: ${nodeEnv}`);
  logger.log(`🌐 URL: http://localhost:${port}/${globalPrefix}`);
  logger.log(`━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━`);
}

bootstrap().catch((err) => {
  // If bootstrap() throws (e.g., Joi validation fails, DB connection fails)
  // This catches it and logs properly instead of unhandled rejection
  console.error('Application failed to start:', err);
  process.exit(1);  // Exit with error code — signals to Docker/systemd that startup failed
});
```

---

## 9. Secrets Management — Production Patterns

### Why .env files are not enough for production

```
Problems with .env files in production:
1. File on disk — anyone with server access can read it
2. Not rotatable easily — changing a secret requires redeploying
3. Not auditable — no log of who accessed the secrets
4. Not scalable — with 50 microservices, managing 50 .env files is chaos

Production solutions:
- AWS Secrets Manager / Parameter Store
- HashiCorp Vault
- Azure Key Vault
- Google Secret Manager
- Kubernetes Secrets
- Docker Swarm secrets
```

### AWS Parameter Store / Secrets Manager integration

```typescript
// src/config/aws-secrets.config.ts
// Fetches secrets from AWS Secrets Manager at startup

import { registerAs } from '@nestjs/config';

async function fetchFromAWS(secretName: string): Promise<Record<string, string>> {
  // In production, you'd use the AWS SDK to fetch secrets
  // npm install @aws-sdk/client-secrets-manager

  const {
    SecretsManagerClient,
    GetSecretValueCommand,
  } = await import('@aws-sdk/client-secrets-manager');

  const client = new SecretsManagerClient({
    region: process.env.AWS_REGION || 'ap-south-1',
  });

  const response = await client.send(
    new GetSecretValueCommand({ SecretId: secretName }),
  );

  // Secret is stored as JSON string in AWS
  return JSON.parse(response.SecretString || '{}');
}

// Note: registerAs factory can be async
export default registerAs('secrets', async () => {
  // Only fetch from AWS in production
  // In development, read from .env as normal
  if (process.env.NODE_ENV === 'production') {
    try {
      const secrets = await fetchFromAWS(
        `${process.env.APP_NAME}/${process.env.NODE_ENV}/secrets`
      );
      // Secret stored in AWS like: {"JWT_SECRET":"xxx","DB_PASSWORD":"yyy"}

      return {
        jwtSecret: secrets.JWT_SECRET,
        jwtRefreshSecret: secrets.JWT_REFRESH_SECRET,
        dbPassword: secrets.DB_PASSWORD,
        stripeKey: secrets.STRIPE_SECRET_KEY,
      };
    } catch (error) {
      console.error('Failed to fetch secrets from AWS:', error);
      throw error;  // Fail startup if secrets can't be fetched
    }
  }

  // Development: use .env file values
  return {
    jwtSecret: process.env.JWT_SECRET,
    jwtRefreshSecret: process.env.JWT_REFRESH_SECRET,
    dbPassword: process.env.DB_PASSWORD,
    stripeKey: process.env.STRIPE_SECRET_KEY,
  };
});
```

### Kubernetes Secrets pattern

```yaml
# k8s/secrets.yaml (never commit actual values)
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
type: Opaque
data:
  # Values are base64 encoded: echo -n 'mysecret' | base64
  JWT_SECRET: bXlzZWNyZXQ=
  DB_PASSWORD: cGFzc3dvcmQ=
```

```yaml
# k8s/deployment.yaml
spec:
  containers:
    - name: myapp
      image: myapp:latest
      env:
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: JWT_SECRET
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: DB_PASSWORD
      # These get injected as process.env.JWT_SECRET etc.
      # Your NestJS app reads them exactly as it would from .env
```

### Secret rotation pattern

```typescript
// src/config/rotating-secrets.service.ts
// Pattern for hot-reloading secrets without app restart

@Injectable()
export class RotatingSecretsService implements OnModuleInit {
  private secrets: Record<string, string> = {};
  private refreshInterval: NodeJS.Timeout;

  async onModuleInit() {
    // Load secrets initially
    await this.refreshSecrets();

    // Refresh every 5 minutes — picks up rotated secrets
    this.refreshInterval = setInterval(
      () => this.refreshSecrets(),
      5 * 60 * 1000,  // 5 minutes
    );
  }

  private async refreshSecrets() {
    try {
      // Fetch latest secrets from AWS/Vault
      this.secrets = await fetchFromAWS('myapp/secrets');
      console.log('Secrets refreshed successfully');
    } catch (error) {
      // Don't throw — keep using existing secrets
      console.error('Failed to refresh secrets:', error);
    }
  }

  getSecret(key: string): string {
    return this.secrets[key] || process.env[key];
  }
}
```

---

## 10. Complete Real-World Configuration Setup

### Full project config structure

```
src/
├── config/
│   ├── app.config.ts          ← Port, env, prefix, CORS
│   ├── database.config.ts     ← DB connection settings
│   ├── jwt.config.ts          ← JWT secrets and expiry
│   ├── mail.config.ts         ← SMTP settings
│   ├── redis.config.ts        ← Redis connection
│   ├── aws.config.ts          ← AWS credentials and settings
│   ├── features.config.ts     ← Feature flags
│   └── index.ts               ← Barrel export of all configs
├── app.module.ts
└── main.ts
```

```typescript
// src/config/index.ts — barrel export

export { default as appConfig } from './app.config';
export { default as databaseConfig } from './database.config';
export { default as jwtConfig } from './jwt.config';
export { default as mailConfig } from './mail.config';
export { default as redisConfig } from './redis.config';
export { default as awsConfig } from './aws.config';
export { default as featuresConfig } from './features.config';
```

```typescript
// src/config/redis.config.ts

import { registerAs } from '@nestjs/config';

export default registerAs('redis', () => ({
  host: process.env.REDIS_HOST || 'localhost',
  port: parseInt(process.env.REDIS_PORT ?? '6379', 10),
  password: process.env.REDIS_PASSWORD || undefined,
  // undefined means no auth — empty string is different from no auth
  ttl: parseInt(process.env.REDIS_TTL ?? '300', 10),  // Default 5 min
  db: parseInt(process.env.REDIS_DB ?? '0', 10),
  // Redis database index: 0-15 (use different DBs for cache vs sessions vs queue)
}));
```

```typescript
// src/app.module.ts — the complete setup

import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import * as Joi from 'joi';
import {
  appConfig,
  databaseConfig,
  jwtConfig,
  mailConfig,
  redisConfig,
  awsConfig,
  featuresConfig,
} from './config';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,

      // Load environment-specific file first, then base .env
      envFilePath: [
        `.env.${process.env.NODE_ENV || 'development'}`,
        '.env',
      ],

      // Don't read .env files in production
      ignoreEnvFile: process.env.NODE_ENV === 'production',

      // registerAs configs — properly typed namespaces
      load: [
        appConfig,
        databaseConfig,
        jwtConfig,
        mailConfig,
        redisConfig,
        awsConfig,
        featuresConfig,
      ],

      // Joi validation — fail fast if required vars are missing
      validationSchema: Joi.object({
        NODE_ENV: Joi.string()
          .valid('development', 'staging', 'production', 'test')
          .default('development'),
        PORT: Joi.number().default(3000),
        DB_HOST: Joi.string().required(),
        DB_PORT: Joi.number().default(5432),
        DB_USERNAME: Joi.string().required(),
        DB_PASSWORD: Joi.string().required(),
        DB_DATABASE: Joi.string().required(),
        JWT_SECRET: Joi.string().min(32).required(),
        JWT_EXPIRES_IN: Joi.string().default('15m'),
        JWT_REFRESH_SECRET: Joi.string().min(32).required(),
        JWT_REFRESH_EXPIRES_IN: Joi.string().default('7d'),
        REDIS_HOST: Joi.string().default('localhost'),
        REDIS_PORT: Joi.number().default(6379),
      }),

      validationOptions: {
        allowUnknown: true,   // Don't fail on OS env vars (PATH, HOME, etc.)
        abortEarly: false,    // Show ALL errors at once
      },

      expandVariables: true,  // Allow ${VAR} references in .env values
    }),
  ],
})
export class AppModule {}
```

### Using config in feature modules

```typescript
// src/auth/auth.module.ts

import { Module } from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';
import { ConfigModule, ConfigService } from '@nestjs/config';

@Module({
  imports: [
    // JwtModule with async config — reads from ConfigService
    JwtModule.registerAsync({
      inject: [ConfigService],
      useFactory: (configService: ConfigService) => ({
        // Use namespaced config — properly typed
        secret: configService.get<string>('jwt.secret'),
        signOptions: {
          expiresIn: configService.get<string>('jwt.expiresIn'),
        },
      }),
    }),
  ],
})
export class AuthModule {}
```

```typescript
// src/auth/auth.service.ts

import { Injectable, Inject } from '@nestjs/common';
import { ConfigType } from '@nestjs/config';
import jwtConfig from '../config/jwt.config';

@Injectable()
export class AuthService {
  constructor(
    // ConfigType<typeof jwtConfig> gives you the EXACT TypeScript type
    // of what the jwtConfig factory returns — full autocomplete!
    @Inject(jwtConfig.KEY)
    private readonly jwt: ConfigType<typeof jwtConfig>,
  ) {}

  getJwtOptions() {
    return {
      secret: this.jwt.secret,         // TypeScript knows this is string
      expiresIn: this.jwt.expiresIn,   // TypeScript knows this is string
      // Full IDE autocomplete with zero chance of typos
    };
  }
}
```

---

## 11. Interview Questions & Answers

**Q1: Why should you never hardcode configuration values in your NestJS app?**

> "Hardcoded values create three major problems. First, security: sensitive values like API keys, JWT secrets, and database passwords end up in your source code repository — once pushed to git, they're potentially exposed forever even if you delete them later. Second, portability: the same codebase needs to run in development, staging, and production with different databases, different API keys, and different settings. Hardcoded values require code changes to deploy to different environments. Third, maintainability: rotating a compromised secret means a code change, review, and full deployment instead of just updating a configuration value. Environment variables solve all three: they're separate from code, easily changed per environment, and can be rotated without touching source code."

---

**Q2: How does ConfigModule work and why use it instead of just dotenv directly?**

> "ConfigModule from `@nestjs/config` is a wrapper around dotenv that integrates with NestJS's dependency injection system. When you call `ConfigModule.forRoot()`, it reads your `.env` file and loads it into `process.env`, then creates a `ConfigService` provider that's registered in the DI container. `ConfigService` provides a typed `get()` method, a `getOrThrow()` for required values, and integrates with `registerAs()` for namespaced configs. It also adds Joi schema validation, environment-specific file loading, and variable expansion. You could use dotenv directly, but then you lose DI integration — you can't inject ConfigService, you have to pass config through function arguments or use module-level singletons, which is harder to test and maintain."

---

**Q3: What is registerAs() and why is it better than plain ConfigService.get()?**

> "`registerAs('namespace', factory)` creates a configuration namespace with a unique injection token. The factory function runs once at startup and returns a typed object where all type conversions happen centrally. The advantages over plain `ConfigService.get()` are: first, type safety — the factory returns a real typed object so TypeScript knows `port` is a `number`, not a `string | undefined`. Second, DRY type conversion — you call `parseInt()` once in the factory, not in every service that reads the port. Third, namespacing — related config groups together logically, and you inject the whole namespace with `@Inject(config.KEY)` instead of reading individual keys scattered across services. Fourth, testability — you can inject a mock config object in tests instead of setting environment variables."

---

**Q4: What happens when Joi validation fails? Why is this a good thing?**

> "When Joi validation fails, NestJS throws an error during `ConfigModule.forRoot()` initialization, which happens before the HTTP server even starts listening. The error message clearly lists every missing or invalid environment variable. This 'fail fast' behavior is valuable because the alternative is worse: the app starts, accepts traffic, and then crashes with a cryptic error when the first request hits code that uses the missing variable. With Joi validation, deployment pipelines catch the problem immediately — the app refuses to start in staging before it ever reaches production. The `abortEarly: false` option is important: it collects ALL validation errors and reports them in one message rather than stopping at the first error, so you can fix everything in one pass."

---

**Q5: How do you access ConfigService before the NestJS app is fully started in main.ts?**

> "After `NestFactory.create(AppModule)` completes, the DI container is fully built even though the HTTP server hasn't started yet. You can use `app.get(ConfigService)` to retrieve ConfigService from the container at this point. This is the standard pattern for reading config in main.ts — for example, to get the PORT before calling `app.listen(port)`. An alternative is to read directly from `process.env` in main.ts since `ConfigModule.forRoot()` runs during `NestFactory.create()` and populates `process.env` from the `.env` file. Either approach works, but `app.get(ConfigService)` is preferred because it respects the same Joi validation and config transformation that the rest of your app uses."

---

**Q6: How do you handle configuration in production where you don't want .env files?**

> "In production, I use `ignoreEnvFile: true` in `ConfigModule.forRoot()` so NestJS doesn't look for .env files. Environment variables are instead injected by the deployment infrastructure — Docker Compose via environment fields, Kubernetes via ConfigMaps and Secrets, or AWS ECS via task definition environment variables. For sensitive secrets specifically, I integrate with a secrets manager like AWS Secrets Manager or HashiCorp Vault: a custom factory runs at startup, fetches secrets via the API, and populates process.env or returns a config object. This is more secure than files on disk because secrets are stored in audited, access-controlled systems rather than plain files. The app code doesn't change — it still reads from ConfigService the same way."

---

**Q7: What is the difference between ConfigService.get() and ConfigService.getOrThrow()?**

> "`get(key)` returns the value or `undefined` if the key doesn't exist. This can lead to silent failures where your code uses `undefined` without realizing it — a JWT secret of `undefined` would cause all tokens to fail at runtime, but not at startup. `getOrThrow(key)` throws a `MissingEnvVarException` immediately if the key isn't set. This is the 'fail fast' principle applied at the service level. In practice: use `getOrThrow()` for values that your code absolutely cannot function without — JWT secret, database password, API keys. Use `get(key, defaultValue)` for optional configuration with sensible defaults — port number, log level, pagination defaults. Having Joi validation at module level AND `getOrThrow()` at usage points gives you two layers of protection."

---

## Quick Reference — Day 5 Cheat Sheet

```
Setup:
  npm install @nestjs/config joi

ConfigModule.forRoot() options:
  isGlobal: true              → available everywhere without importing
  envFilePath: ['.env.dev', '.env']  → priority: left wins
  ignoreEnvFile: true         → production: don't read files, use system env
  load: [dbConfig, jwtConfig] → namespaced configs via registerAs()
  validationSchema: Joi.object({})   → fail-fast if required vars missing
  validationOptions: { allowUnknown: true, abortEarly: false }
  expandVariables: true       → allow ${OTHER_VAR} in .env values

ConfigService methods:
  .get('KEY')                 → string | undefined (use with caution)
  .get('KEY', 'default')      → string | 'default' (with fallback)
  .get<number>('KEY')         → TypeScript says number, still string at runtime!
  .getOrThrow('KEY')          → throws if not set (use for required vars)
  .get('namespace.property')  → read from registerAs() namespace

registerAs('namespace', () => ({ ... })):
  - Parse types IN the factory (parseInt, === 'true')
  - Inject with @Inject(config.KEY) private config: ConfigType<typeof config>
  - Access as typed object with full autocomplete

Type parsing (always do this explicitly):
  number: parseInt(process.env.PORT ?? '3000', 10)
  boolean: process.env.FLAG === 'true'
  array: process.env.ORIGINS?.split(',').map(o => o.trim())

.env file rules:
  .env             → base defaults
  .env.development → dev overrides (NOT committed if has secrets)
  .env.test        → test config (can be committed if no real secrets)
  .env.example     → template with empty values (ALWAYS commit this)
  .gitignore: .env, .env.*.local, .env.development, .env.production

Production:
  ignoreEnvFile: true         → use system/Docker/K8s env vars
  AWS Secrets Manager         → for rotating/audited secrets
  getOrThrow() on all required vars
  Joi validation on all vars at startup

In main.ts (after NestFactory.create()):
  const config = app.get(ConfigService);
  const port = config.get<number>('PORT', 3000);
  await app.listen(port);
```

---

*Day 5 complete. Tomorrow — Day 6: Database with TypeORM — Entities, Repositories, Relations, QueryBuilder, Migrations, and Transaction handling. The most code-heavy day.*
