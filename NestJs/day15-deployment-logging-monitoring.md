# Day 15 — Deployment, Logging & Monitoring
### NestJS Interview Prep | 3-Year Backend Engineer Level

> **Goal for today:** Master production deployment of NestJS applications — Docker containerization, environment-based configuration, structured logging with Winston/Pino, health checks with `@nestjs/terminus`, Swagger documentation, and CI/CD pipeline basics. This is the final day — a 3-year backend engineer must know how to take code from laptop to production reliably and safely.

---

## 🧭 First-Timer Guidance — Read This First

**The production deployment checklist mindset:**

A junior developer writes code that works on their laptop. A senior engineer writes code that runs safely in production for years. The difference:

```
Junior:  "It works on my machine"
Senior:  "It works in production, recovers from failures,
          logs everything important, alerts on critical errors,
          deploys without downtime, and can be monitored in real-time"
```

**What "production-ready" means for a NestJS app:**

```
1. CONTAINERIZED (Docker):
   Same environment everywhere — dev, staging, production
   "Works on my machine" → "Works in every container"

2. CONFIGURED SAFELY:
   No hardcoded secrets
   Environment-specific settings via env vars
   Validates config at startup (Joi) — fail fast

3. LOGGED PROPERLY:
   Structured JSON logs (not console.log strings)
   Every request logged with correlation ID
   Errors logged with full stack traces
   Logs ship to centralized system (Datadog, CloudWatch)

4. HEALTH MONITORED:
   /health endpoint that checks DB, Redis, external services
   Kubernetes/load balancer routes away from unhealthy instances
   Alerts when health degrades

5. DOCUMENTED:
   Swagger/OpenAPI for all endpoints
   Response schemas documented
   Auth requirements clear

6. DEPLOYABLE:
   CI/CD pipeline — tests pass → auto deploy
   Zero-downtime deployments
   Rollback capability
```

**Setup — install packages:**
```bash
npm install @nestjs/terminus
npm install winston nest-winston
# OR for better performance:
npm install pino pino-pretty nestjs-pino
npm install @nestjs/swagger swagger-ui-express
```

---

## Table of Contents

1. [Docker — Containerizing NestJS](#1-docker--containerizing-nestjs)
2. [Docker Compose — Local Development Stack](#2-docker-compose--local-development-stack)
3. [Environment Configuration — Production Patterns](#3-environment-configuration--production-patterns)
4. [Logging with Winston — Structured Logs](#4-logging-with-winston--structured-logs)
5. [Logging with Pino — High Performance Logs](#5-logging-with-pino--high-performance-logs)
6. [Request Logging — HTTP Access Logs](#6-request-logging--http-access-logs)
7. [Health Checks with @nestjs/terminus](#7-health-checks-with-nestjsterminus)
8. [Swagger Documentation — Complete Setup](#8-swagger-documentation--complete-setup)
9. [Graceful Shutdown](#9-graceful-shutdown)
10. [CI/CD Pipeline Basics](#10-cicd-pipeline-basics)
11. [Production Checklist](#11-production-checklist)
12. [Interview Questions & Answers](#12-interview-questions--answers)

---

## 1. Docker — Containerizing NestJS

### Why Docker?

```
Without Docker:
  "It works on my MacBook but fails on the Linux production server"
  Dev: Node 18, macOS, different ENV vars
  Prod: Node 16, Ubuntu, different path structure
  Result: bugs that only appear in production

With Docker:
  App + runtime + dependencies bundled into ONE image
  Same image runs on macOS, Linux, Windows, AWS, GCP, Azure
  "Build once, run anywhere"
  Consistent environment from dev → staging → production
```

### Multi-stage Dockerfile — production optimized

```dockerfile
# Dockerfile
# Multi-stage build: separate build stage from runtime stage
# This keeps the final image SMALL — no dev dependencies, no TypeScript compiler

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# STAGE 1: DEPENDENCIES
# Install all dependencies (including dev) for building
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FROM node:20-alpine AS deps
# node:20-alpine: Node 20 on Alpine Linux
# Alpine: minimal Linux distro (~5MB vs ~100MB for Ubuntu)
# Results in much smaller Docker images

WORKDIR /app
# Set working directory inside the container

# Copy package files first (Docker layer caching optimization)
COPY package.json package-lock.json ./
# Why copy these first?
# Docker caches each layer. If package.json doesn't change,
# npm install is cached and skipped on subsequent builds
# Only copying source files later doesn't invalidate this cache

RUN npm ci --only=production
# npm ci: clean install (faster than npm install, uses package-lock.json)
# --only=production: skip devDependencies (jest, eslint, ts-jest, etc.)
# This creates node_modules with ONLY runtime dependencies

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# STAGE 2: BUILD
# Compile TypeScript to JavaScript
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FROM node:20-alpine AS build

WORKDIR /app

COPY package.json package-lock.json ./

# Install ALL dependencies (including TypeScript, NestJS CLI)
RUN npm ci
# No --only=production: need TypeScript compiler and NestJS CLI for build

# Copy application source
COPY . .
# . . copies everything from current directory to /app in container
# .dockerignore excludes: node_modules, dist, .git, .env files

# Compile TypeScript → JavaScript
RUN npm run build
# Runs: nest build → compiles src/ → dist/

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# STAGE 3: PRODUCTION
# Minimal runtime image — only what's needed to RUN the app
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FROM node:20-alpine AS production

# Add curl for health checks in Docker Compose / K8s
RUN apk add --no-cache curl

# Create non-root user for security
# Running as root inside containers is a security vulnerability
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nestjs -u 1001 -G nodejs
# Creates group 'nodejs' and user 'nestjs'

WORKDIR /app

# Copy production node_modules from deps stage
COPY --from=deps /app/node_modules ./node_modules
# Only production dependencies — no TypeScript compiler

# Copy compiled JavaScript from build stage
COPY --from=build /app/dist ./dist

# Copy package.json (needed by Node.js for module resolution)
COPY package.json ./

# Set ownership to non-root user
RUN chown -R nestjs:nodejs /app
USER nestjs
# All subsequent commands and the running process use 'nestjs' user
# If the container is compromised, attacker has limited permissions

# Expose the port the app runs on
EXPOSE 3000
# EXPOSE is documentation — doesn't actually publish the port
# Publishing happens with docker run -p 3000:3000

# Health check built into the container
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
# Docker marks container as unhealthy if this fails 3 times in a row
# interval: check every 30 seconds
# timeout: fail if check takes > 10 seconds
# start-period: give app 30 seconds to start before checking

# Start the compiled JavaScript (not TypeScript)
# NEVER use 'nest start' in production — it recompiles on every start
CMD ["node", "dist/main"]
```

### .dockerignore — what to exclude from Docker context

```
# .dockerignore
# Prevents these from being sent to Docker daemon
# Keeps build context small → faster builds

node_modules
# Never copy local node_modules — install fresh in container
# Local node_modules might be compiled for wrong OS (macOS vs Linux binaries)

dist
# Old compiled output — fresh compile in container

.git
# Git history not needed in production container

.github
# CI/CD configs not needed

*.md
# Documentation not needed at runtime

.env
.env.*
# NEVER include .env files in Docker images
# Secrets would be baked into the image → visible in image layers

coverage
# Test coverage reports

test
__tests__
**/*.spec.ts
**/*.test.ts
# Test files not needed in production

.dockerignore
Dockerfile
docker-compose.yml
# Build files themselves

*.log
# Log files

.eslintrc.js
.prettierrc
# Dev tools config
```

### Building and running

```bash
# Build the Docker image
docker build -t myapp:latest .
# -t myapp:latest: name:tag for the image

# Build for specific environment
docker build -t myapp:1.2.3 --build-arg NODE_ENV=production .

# Run the container
docker run \
  -p 3000:3000 \
  # Map host port 3000 → container port 3000
  -e DATABASE_URL=postgresql://user:pass@localhost:5432/mydb \
  # Pass environment variables at runtime
  -e JWT_SECRET=production-secret \
  --name myapp-prod \
  # Container name
  --restart unless-stopped \
  # Restart policy: restart on crash, unless manually stopped
  myapp:latest

# Check if container is healthy
docker ps
# STATUS column shows: healthy / unhealthy / starting

# View container logs
docker logs myapp-prod -f
# -f: follow (like tail -f)

# Execute command inside running container
docker exec -it myapp-prod sh
# -it: interactive terminal
# sh: Alpine uses sh (not bash)

# Check image size
docker images myapp
# Multi-stage build should be ~200-300MB (vs 1GB+ without optimization)
```

---

## 2. Docker Compose — Local Development Stack

```yaml
# docker-compose.yml
# Defines your entire local development environment as code
# Run ALL services with: docker-compose up -d

version: '3.9'

services:
  # ── Application ──────────────────────────────────────────────────────────
  api:
    build:
      context: .
      target: production
      # Build only the 'production' stage of the Dockerfile
    container_name: myapp-api
    ports:
      - '3000:3000'
    environment:
      # Environment variables for the API container
      NODE_ENV: development
      PORT: 3000
      
      # Database (references the 'postgres' service below by name)
      DB_HOST: postgres
      # Docker Compose creates a network where service names are hostnames
      # 'postgres' = hostname of the postgres container
      DB_PORT: 5432
      DB_USERNAME: ${POSTGRES_USER:-myapp_user}
      # ${VAR:-default}: use .env var if set, otherwise use 'myapp_user'
      DB_PASSWORD: ${POSTGRES_PASSWORD:-localpassword}
      DB_DATABASE: ${POSTGRES_DB:-myapp_dev}
      DB_SYNCHRONIZE: 'true'
      # true for development — creates tables automatically

      # Redis (references the 'redis' service below)
      REDIS_HOST: redis
      REDIS_PORT: 6379

      # Auth
      JWT_SECRET: dev-only-secret-not-for-production-32chars
      JWT_REFRESH_SECRET: dev-only-refresh-secret-32chars
      JWT_EXPIRES_IN: 15m
      JWT_REFRESH_EXPIRES_IN: 7d

      # RabbitMQ
      RABBITMQ_URL: amqp://guest:guest@rabbitmq:5672

    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    # depends_on: wait for these services to be HEALTHY before starting API
    # Prevents "DB connection refused" on startup
    
    volumes:
      - ./src:/app/src:ro
      # Mount source code into container for hot-reload
      # :ro = read-only (container can't modify your local files)
      # Remove for production image
    
    restart: unless-stopped
    networks:
      - myapp-network

  # ── PostgreSQL Database ───────────────────────────────────────────────────
  postgres:
    image: postgres:16-alpine
    # Use official postgres image — Alpine for smaller size
    container_name: myapp-postgres
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-myapp_user}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-localpassword}
      POSTGRES_DB: ${POSTGRES_DB:-myapp_dev}
    ports:
      - '5432:5432'
      # Expose port so you can connect with pgAdmin, DBeaver, etc.
    volumes:
      - postgres-data:/var/lib/postgresql/data
      # Named volume: persists data between container restarts
      # Without volume: data lost when container is removed
      - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
      # Optional: run SQL script when DB is first created
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U ${POSTGRES_USER:-myapp_user} -d ${POSTGRES_DB:-myapp_dev}']
      # pg_isready: postgres utility that checks if DB is accepting connections
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
      # Give postgres 30s to start before checking
    networks:
      - myapp-network

  # ── Redis ─────────────────────────────────────────────────────────────────
  redis:
    image: redis:7-alpine
    container_name: myapp-redis
    command: redis-server --appendonly yes
    # appendonly yes: enable AOF persistence (data survives restarts)
    ports:
      - '6379:6379'
    volumes:
      - redis-data:/data
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - myapp-network

  # ── RabbitMQ ──────────────────────────────────────────────────────────────
  rabbitmq:
    image: rabbitmq:3.12-management-alpine
    # management: includes web UI at http://localhost:15672
    container_name: myapp-rabbitmq
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
      RABBITMQ_DEFAULT_VHOST: /
    ports:
      - '5672:5672'   # AMQP protocol port (for your app)
      - '15672:15672' # Management UI port (for your browser)
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
    healthcheck:
      test: ['CMD', 'rabbitmq-diagnostics', 'check_port_connectivity']
      interval: 30s
      timeout: 10s
      retries: 5
    networks:
      - myapp-network

  # ── Database administration UI (dev only) ─────────────────────────────────
  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: myapp-pgadmin
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@myapp.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - '5050:80'
      # pgAdmin at http://localhost:5050
    profiles:
      - tools
      # Only start with: docker-compose --profile tools up
      # Don't start by default — start explicitly when needed
    networks:
      - myapp-network

# ── Volumes (named persistent storage) ───────────────────────────────────────
volumes:
  postgres-data:
    driver: local
  redis-data:
    driver: local
  rabbitmq-data:
    driver: local

# ── Network (isolated virtual network for all services) ───────────────────────
networks:
  myapp-network:
    driver: bridge
    # bridge: isolated virtual network — services communicate by name
    # No port exposure needed between services (only host ports matter)
```

### Docker Compose commands

```bash
# Start all services in background
docker-compose up -d
# -d: detached mode (doesn't block terminal)

# Start with specific profiles
docker-compose --profile tools up -d

# View running containers and their status
docker-compose ps

# View logs from all services
docker-compose logs -f
# View logs from specific service
docker-compose logs -f api

# Rebuild and restart
docker-compose up -d --build

# Stop all services (keeps volumes)
docker-compose stop

# Stop and remove containers (keeps volumes)
docker-compose down

# Stop and remove containers AND volumes (wipes database!)
docker-compose down -v

# Run a command in a service
docker-compose exec api sh
docker-compose exec postgres psql -U myapp_user myapp_dev
```

---

## 3. Environment Configuration — Production Patterns

### Environment variable management

```typescript
// src/config/validate-env.ts
// Joi validation schema for all environment variables
// Run at startup — fail fast if anything is missing or wrong

import * as Joi from 'joi';

export const envValidationSchema = Joi.object({
  // Application
  NODE_ENV: Joi.string()
    .valid('development', 'staging', 'production', 'test')
    .required(),
  PORT: Joi.number().default(3000),

  // Database
  DB_HOST: Joi.string().required(),
  DB_PORT: Joi.number().default(5432),
  DB_USERNAME: Joi.string().required(),
  DB_PASSWORD: Joi.string().required(),
  DB_DATABASE: Joi.string().required(),
  DB_SYNCHRONIZE: Joi.boolean().default(false),

  // Redis
  REDIS_HOST: Joi.string().default('localhost'),
  REDIS_PORT: Joi.number().default(6379),

  // JWT
  JWT_SECRET: Joi.string().min(32).required(),
  JWT_EXPIRES_IN: Joi.string().default('15m'),
  JWT_REFRESH_SECRET: Joi.string().min(32).required(),
  JWT_REFRESH_EXPIRES_IN: Joi.string().default('7d'),
}).options({ allowUnknown: true });
// allowUnknown: true — don't fail on OS environment variables (PATH, HOME, etc.)
```

### Runtime config service

```typescript
// src/config/app.config.ts
// Typed configuration using registerAs()

import { registerAs } from '@nestjs/config';

export default registerAs('app', () => ({
  env: process.env.NODE_ENV || 'development',
  port: parseInt(process.env.PORT || '3000', 10),
  isDevelopment: process.env.NODE_ENV === 'development',
  isProduction: process.env.NODE_ENV === 'production',
  isTest: process.env.NODE_ENV === 'test',

  // Feature flags from environment
  enableSwagger: process.env.ENABLE_SWAGGER !== 'false',
  // Swagger enabled by default, disable in production with ENABLE_SWAGGER=false

  enableBullDashboard: process.env.NODE_ENV !== 'production',
  // Queue dashboard only in non-production environments
}));
```

---

## 4. Logging with Winston — Structured Logs

### Why structured logging matters

```
Bad logging (unstructured):
  console.log(`User ${userId} logged in at ${timestamp}`)
  → Output: "User 123 logged in at 2024-01-01T10:00:00Z"
  
  Problems:
  - Hard to search/filter in log aggregation systems
  - Can't easily extract userId for filtering
  - No log level metadata
  - No request correlation

Good logging (structured JSON):
  logger.info('User logged in', { userId: 123, timestamp, requestId, duration })
  → Output: {"level":"info","message":"User logged in","userId":123,"requestId":"abc","timestamp":"2024-01-01T10:00:00Z"}
  
  Benefits:
  - Search by userId: filter logs userId=123
  - Alert on level=error
  - Correlate by requestId across multiple log lines
  - Parse automatically by Datadog, CloudWatch, Elasticsearch
```

### Winston setup

```typescript
// src/logger/winston.config.ts

import * as winston from 'winston';
import { utilities as nestWinstonModuleUtilities } from 'nest-winston';

const { combine, timestamp, errors, json, colorize, printf } = winston.format;

// ── Development format: human-readable colored output ─────────────────────
const developmentFormat = combine(
  colorize({ all: true }),
  timestamp({ format: 'YYYY-MM-DD HH:mm:ss' }),
  errors({ stack: true }),  // Include stack trace on errors
  printf(({ level, message, timestamp, stack, ...meta }) => {
    const metaStr = Object.keys(meta).length
      ? `\n${JSON.stringify(meta, null, 2)}`
      : '';
    return `${timestamp} [${level}] ${message}${stack ? '\n' + stack : ''}${metaStr}`;
  }),
);

// ── Production format: structured JSON ───────────────────────────────────
const productionFormat = combine(
  timestamp(),
  errors({ stack: true }),
  json(),
  // JSON format: {"level":"info","message":"...","timestamp":"...","userId":123}
  // Automatically parsed by Datadog, CloudWatch, ELK stack
);

export const winstonConfig = {
  transports: [
    // ── Console transport (always enabled) ─────────────────────────────
    new winston.transports.Console({
      format: process.env.NODE_ENV === 'production'
        ? productionFormat
        : developmentFormat,
      // Different format per environment
    }),

    // ── File transport (production only) ──────────────────────────────
    ...(process.env.NODE_ENV === 'production'
      ? [
          // Error log file — only errors and above
          new winston.transports.File({
            filename: '/var/log/myapp/error.log',
            level: 'error',
            format: productionFormat,
            maxsize: 50 * 1024 * 1024,   // Rotate after 50MB
            maxFiles: 10,                  // Keep last 10 rotated files
            tailable: true,               // Tail-able for 'tail -f' command
          }),

          // Combined log file — all levels
          new winston.transports.File({
            filename: '/var/log/myapp/combined.log',
            format: productionFormat,
            maxsize: 100 * 1024 * 1024,  // 100MB per file
            maxFiles: 5,
          }),
        ]
      : []),

    // ── CloudWatch / external log service (uncomment for production) ────
    // new WinstonCloudWatch({...}) — npm install winston-cloudwatch
    // new DatadogWinston({...})    — npm install datadog-winston
  ],

  // Log levels: error(0) > warn(1) > info(2) > http(3) > debug(4) > silly(5)
  level: process.env.LOG_LEVEL || 
    (process.env.NODE_ENV === 'production' ? 'info' : 'debug'),
    // Production: log info and above (not debug — too verbose)
    // Development: log everything including debug

  // Silent mode for tests — don't pollute test output
  silent: process.env.NODE_ENV === 'test',
};
```

### Integrating Winston with NestJS

```typescript
// src/app.module.ts

import { Module, Logger } from '@nestjs/common';
import { WinstonModule } from 'nest-winston';
import { winstonConfig } from './logger/winston.config';

@Module({
  imports: [
    WinstonModule.forRoot(winstonConfig),
    // Replaces NestJS's built-in Logger with Winston
    // All NestJS internal logs now go through Winston too
    // (module initialization, route registration, etc.)
  ],
})
export class AppModule {}

// src/main.ts — use Winston as the NestJS logger
import { WinstonModule } from 'nest-winston';
import { winstonConfig } from './logger/winston.config';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    logger: WinstonModule.createLogger(winstonConfig),
    // Replace NestJS default logger with Winston
    // Now ALL NestJS framework logs use Winston
  });

  await app.listen(3000);
}
```

### Using the logger in services

```typescript
// src/users/users.service.ts

import { Injectable, Logger, NotFoundException } from '@nestjs/common';

@Injectable()
export class UsersService {
  // Create a logger scoped to this class
  // All logs from this logger include the class name automatically
  private readonly logger = new Logger(UsersService.name);
  // UsersService.name = 'UsersService'
  // Log output: [UsersService] User 123 logged in

  async findOne(id: number): Promise<User> {
    // Debug: detailed info for development debugging
    this.logger.debug(`Finding user with id: ${id}`);

    const user = await this.userRepository.findOne({ where: { id } });

    if (!user) {
      // Warn: unexpected but recoverable situation
      this.logger.warn(`User ${id} not found`, {
        userId: id,
        action: 'findOne',
        // Structured context — searchable in log aggregation
      });
      throw new NotFoundException(`User #${id} not found`);
    }

    // Verbose: trace-level (only in very detailed debugging)
    this.logger.verbose(`Found user: ${user.email}`);

    return user;
  }

  async create(dto: CreateUserDto): Promise<User> {
    try {
      const user = await this.userRepository.save(
        this.userRepository.create(dto),
      );

      // Info: significant business events
      this.logger.log('User created successfully', {
        userId: user.id,
        email: user.email,
        // Don't log: password, tokens, sensitive data
      });

      return user;

    } catch (error) {
      // Error: unexpected errors that need investigation
      this.logger.error('Failed to create user', {
        email: dto.email,
        error: error.message,
        // stack: error.stack  // Include for debugging
      });
      throw error;
    }
  }
}
```

---

## 5. Logging with Pino — High Performance Logs

```typescript
// Alternative to Winston: Pino is 5-10x faster
// Recommended for high-traffic applications
// npm install nestjs-pino pino-http pino-pretty

// src/app.module.ts

import { LoggerModule } from 'nestjs-pino';

@Module({
  imports: [
    LoggerModule.forRootAsync({
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        pinoHttp: {
          // ── Log level ─────────────────────────────────────────────────
          level: config.get('NODE_ENV') === 'production' ? 'info' : 'debug',

          // ── Format ────────────────────────────────────────────────────
          transport: config.get('NODE_ENV') !== 'production'
            ? {
                target: 'pino-pretty',
                // pino-pretty: human-readable colorized output for development
                options: {
                  colorize: true,
                  singleLine: true,  // Each log on one line (not multi-line)
                  translateTime: 'SYS:yyyy-mm-dd HH:MM:ss',
                },
              }
            : undefined,
          // Production: raw JSON (no pretty-printing overhead)

          // ── Request logging ──────────────────────────────────────────
          autoLogging: {
            ignore: (req) => req.url === '/health',
            // Don't log health check requests — too noisy (happens every 30s)
          },

          // ── Sensitive data redaction ──────────────────────────────────
          redact: {
            paths: [
              'req.headers.authorization',  // Don't log JWT tokens
              'req.body.password',          // Don't log passwords
              'req.body.creditCard',        // Don't log payment info
              '*.token',                    // Any 'token' field anywhere
              '*.secret',                   // Any 'secret' field
            ],
            censor: '[REDACTED]',
            // Replace sensitive values with '[REDACTED]' in logs
          },

          // ── Custom serializers ────────────────────────────────────────
          serializers: {
            req: (req) => ({
              method: req.method,
              url: req.url,
              id: req.id,         // Request ID for correlation
              // Don't include headers — might have auth tokens
            }),
            res: (res) => ({
              statusCode: res.statusCode,
            }),
          },

          // ── Custom request ID ─────────────────────────────────────────
          genReqId: (req) => {
            return req.headers['x-request-id'] ||
              `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
            // Use existing request ID from gateway/load balancer
            // Or generate a unique one for correlation tracking
          },
        },
      }),
    }),
  ],
})
export class AppModule {}

// src/main.ts — use Pino logger
import { Logger } from 'nestjs-pino';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, { bufferLogs: true });
  app.useLogger(app.get(Logger));
  // bufferLogs: true — buffer logs until Pino logger is ready
  await app.listen(3000);
}
```

---

## 6. Request Logging — HTTP Access Logs

### Request correlation and logging middleware

```typescript
// src/common/middleware/request-logger.middleware.ts
// Logs every HTTP request with timing and correlation ID

import { Injectable, NestMiddleware, Logger } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import { v4 as uuidv4 } from 'uuid';

@Injectable()
export class RequestLoggerMiddleware implements NestMiddleware {
  private readonly logger = new Logger('HTTP');

  use(req: Request, res: Response, next: NextFunction): void {
    const requestId = req.headers['x-request-id'] as string || uuidv4();
    const startTime = Date.now();

    // Attach request ID to request object (available in controllers)
    (req as any).requestId = requestId;

    // Add request ID to response headers (for debugging)
    res.setHeader('X-Request-Id', requestId);

    // Log when response finishes
    res.on('finish', () => {
      const duration = Date.now() - startTime;
      const { method, originalUrl, ip } = req;
      const { statusCode } = res;

      const logData = {
        requestId,
        method,
        url: originalUrl,
        statusCode,
        duration: `${duration}ms`,
        ip,
        userAgent: req.get('user-agent'),
        userId: (req as any).user?.id,
        // Include user ID if authenticated (set by JwtStrategy)
      };

      // Log level based on status code
      if (statusCode >= 500) {
        this.logger.error(`${method} ${originalUrl} ${statusCode} ${duration}ms`, logData);
      } else if (statusCode >= 400) {
        this.logger.warn(`${method} ${originalUrl} ${statusCode} ${duration}ms`, logData);
      } else {
        this.logger.log(`${method} ${originalUrl} ${statusCode} ${duration}ms`, logData);
      }

      // Alert on slow requests
      if (duration > 2000) {
        this.logger.warn('SLOW REQUEST detected', {
          ...logData,
          threshold: '2000ms',
        });
      }
    });

    next();
  }
}
```

---

## 7. Health Checks with @nestjs/terminus

### What are health checks?

```
Health check endpoint: GET /health

Returns:
  200 OK → App is healthy, load balancer routes traffic to it
  503 Service Unavailable → App is unhealthy, load balancer removes it

Kubernetes uses health checks for:
  Liveness probe:   Is the app alive? If not → restart the pod
  Readiness probe:  Is the app ready for traffic? If not → stop routing to it
  Startup probe:    Has the app finished starting? Give it more time

Without health checks:
  Pod starts, Kubernetes routes traffic immediately
  App is still connecting to DB → requests fail
  "Connection refused" errors for 30 seconds at deployment

With health checks:
  Pod starts, Kubernetes waits for readiness
  Only routes traffic when /health returns 200
  Zero failed requests at deployment
```

### Setting up health checks

```typescript
// src/health/health.module.ts

import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';
import { HttpModule } from '@nestjs/axios';
import { HealthController } from './health.controller';
import { DatabaseHealthIndicator } from './indicators/database.indicator';
import { RedisHealthIndicator } from './indicators/redis.indicator';

@Module({
  imports: [
    TerminusModule.forRoot({
      errorLogStyle: 'pretty',  // Pretty print health check errors in logs
    }),
    HttpModule,
    // HttpModule: for checking external HTTP endpoints
  ],
  controllers: [HealthController],
  providers: [DatabaseHealthIndicator, RedisHealthIndicator],
})
export class HealthModule {}
```

```typescript
// src/health/health.controller.ts

import { Controller, Get } from '@nestjs/common';
import {
  HealthCheck,
  HealthCheckService,
  TypeOrmHealthIndicator,
  MemoryHealthIndicator,
  DiskHealthIndicator,
  HttpHealthIndicator,
} from '@nestjs/terminus';
import { Public } from '../auth/decorators/public.decorator';
import { RedisHealthIndicator } from './indicators/redis.indicator';

@Controller('health')
@Public()
// Health endpoint must be public — no JWT required
// Load balancers and monitoring systems don't have auth tokens
export class HealthController {
  constructor(
    private readonly health: HealthCheckService,
    // HealthCheckService: runs all indicators and aggregates results

    private readonly db: TypeOrmHealthIndicator,
    // TypeOrmHealthIndicator: checks database connectivity

    private readonly memory: MemoryHealthIndicator,
    // MemoryHealthIndicator: checks Node.js process memory

    private readonly disk: DiskHealthIndicator,
    // DiskHealthIndicator: checks disk space

    private readonly http: HttpHealthIndicator,
    // HttpHealthIndicator: checks external HTTP services

    private readonly redis: RedisHealthIndicator,
    // Custom Redis indicator
  ) {}

  // ── Liveness probe — is the app alive? ────────────────────────────────
  // Kubernetes: if this returns unhealthy → restart the pod
  @Get('liveness')
  @HealthCheck()
  liveness() {
    return this.health.check([
      () => this.memory.checkHeap('memory_heap', 500 * 1024 * 1024),
      // Check heap usage < 500MB
      // If > 500MB: probably a memory leak → restart

      () => this.memory.checkRSS('memory_rss', 1024 * 1024 * 1024),
      // Check RSS (total memory) < 1GB
    ]);
    // Returns:
    // 200 OK: { status: 'ok', info: { memory_heap: { status: 'up' } } }
    // 503:    { status: 'error', error: { memory_heap: { status: 'down', message: '...' } } }
  }

  // ── Readiness probe — is the app ready for traffic? ─────────────────
  // Kubernetes: if this returns unhealthy → stop routing traffic to this pod
  @Get('readiness')
  @HealthCheck()
  readiness() {
    return this.health.check([
      // Check database connection
      () => this.db.pingCheck('database', { timeout: 3000 }),
      // pingCheck: runs SELECT 1 query, fails if no response in 3 seconds

      // Check Redis connection
      () => this.redis.isHealthy('redis'),
      // Custom indicator — see below

      // Check disk space
      () => this.disk.checkStorage('disk', {
        thresholdPercent: 0.9,  // Alert if > 90% full
        path: '/',
      }),
    ]);
  }

  // ── Full health check — all indicators ────────────────────────────────
  // Used by monitoring systems (Datadog, Prometheus) for full status
  @Get()
  @Get('status')  // Both /health and /health/status work
  @HealthCheck()
  check() {
    return this.health.check([
      // Database
      () => this.db.pingCheck('database', { timeout: 3000 }),

      // Redis
      () => this.redis.isHealthy('redis'),

      // Memory
      () => this.memory.checkHeap('memory_heap', 500 * 1024 * 1024),
      () => this.memory.checkRSS('memory_rss', 1024 * 1024 * 1024),

      // Disk
      () => this.disk.checkStorage('disk', {
        thresholdPercent: 0.9,
        path: '/',
      }),

      // External services (optional — only check if your app depends on them)
      () => this.http.pingCheck(
        'stripe-api',
        'https://api.stripe.com/v1/ping',
        { timeout: 3000 },
      ),
    ]);
  }
}
```

### Custom Redis health indicator

```typescript
// src/health/indicators/redis.indicator.ts

import { Injectable } from '@nestjs/common';
import { HealthIndicator, HealthIndicatorResult, HealthCheckError } from '@nestjs/terminus';
import { InjectRedis } from '@nestjs-modules/ioredis';  // or however you inject redis
import Redis from 'ioredis';

@Injectable()
export class RedisHealthIndicator extends HealthIndicator {
  constructor(@InjectRedis() private readonly redis: Redis) {
    super();
  }

  async isHealthy(key: string): Promise<HealthIndicatorResult> {
    try {
      const start = Date.now();
      const result = await this.redis.ping();
      // PING → PONG (Redis standard health check)
      const responseTime = Date.now() - start;

      if (result !== 'PONG') {
        throw new Error('Unexpected response from Redis');
      }

      return this.getStatus(key, true, { responseTime: `${responseTime}ms` });
      // getStatus(key, isHealthy, metadata):
      // true → { redis: { status: 'up', responseTime: '1ms' } }
      // false → { redis: { status: 'down', ... } }

    } catch (error) {
      throw new HealthCheckError(
        'Redis health check failed',
        this.getStatus(key, false, { error: error.message }),
      );
      // HealthCheckError: signals terminus that this check failed
    }
  }
}
```

### Health check response format

```json
// GET /health → 200 OK (all healthy)
{
  "status": "ok",
  "info": {
    "database": { "status": "up" },
    "redis": { "status": "up", "responseTime": "1ms" },
    "memory_heap": { "status": "up", "used": "245MB", "max": "500MB" },
    "disk": { "status": "up", "free": "45GB" }
  },
  "error": {},
  "details": {
    "database": { "status": "up" },
    "redis": { "status": "up", "responseTime": "1ms" }
  }
}

// GET /health → 503 Service Unavailable (database down)
{
  "status": "error",
  "info": {
    "redis": { "status": "up" },
    "memory_heap": { "status": "up" }
  },
  "error": {
    "database": {
      "status": "down",
      "message": "Connection refused"
    }
  },
  "details": {
    "database": { "status": "down", "message": "Connection refused" },
    "redis": { "status": "up" }
  }
}
```

---

## 8. Swagger Documentation — Complete Setup

```typescript
// src/main.ts — complete Swagger setup

import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { INestApplication } from '@nestjs/common';

async function setupSwagger(app: INestApplication, config: ConfigService) {
  const enableSwagger = config.get('ENABLE_SWAGGER', 'true') !== 'false';

  if (!enableSwagger) return;
  // Allow disabling Swagger in production via environment variable
  // Some companies disable API docs in production for security

  const swaggerConfig = new DocumentBuilder()
    .setTitle('MyApp API')
    .setDescription(`
## MyApp REST API

### Authentication
All protected endpoints require a JWT Bearer token.

**Get a token:** POST /api/v1/auth/login
**Use token:** Add header: \`Authorization: Bearer <your-token>\`

### Rate Limiting
- **General:** 100 requests per minute per IP
- **Login:** 5 attempts per 5 minutes per IP
- **Rate limit exceeded:** 429 Too Many Requests

### Versioning
Current API version: **v1**
Base URL: \`/api/v1\`
    `)
    .setVersion('1.0.0')
    .setContact('MyApp Team', 'https://myapp.com', 'api-support@myapp.com')
    .setLicense('MIT', 'https://opensource.org/licenses/MIT')

    .addServer(`http://localhost:${config.get('PORT', '3000')}`, 'Local Development')
    .addServer('https://api.staging.myapp.com', 'Staging')
    .addServer('https://api.myapp.com', 'Production')

    .addBearerAuth(
      {
        type: 'http',
        scheme: 'bearer',
        bearerFormat: 'JWT',
        name: 'JWT',
        description: 'Enter JWT access token from /auth/login response',
        in: 'header',
      },
      'JWT-auth',
    )

    .addTag('Authentication', 'Register, login, token refresh, logout')
    .addTag('Users', 'User profile management')
    .addTag('Products', 'Product catalog')
    .addTag('Orders', 'Order placement and tracking')
    .addTag('Health', 'Application health monitoring')

    .build();

  const document = SwaggerModule.createDocument(app, swaggerConfig, {
    extraModels: [],
    // Register models not used in controllers but referenced in generics
    operationIdFactory: (controllerKey: string, methodKey: string) =>
      `${controllerKey}_${methodKey}`,
    // Custom operation ID format for generated SDK clients
  });

  SwaggerModule.setup('docs', app, document, {
    swaggerOptions: {
      persistAuthorization: true,
      // Remember auth token across page refreshes
      docExpansion: 'none',
      // Collapse all sections by default (cleaner for large APIs)
      filter: true,
      // Show search box
      showRequestDuration: true,
      // Display response time for each request
      defaultModelExpandDepth: 3,
      // How deep to expand model schemas by default
      syntaxHighlight: { activate: true, theme: 'agate' },
    },
    customSiteTitle: 'MyApp API Documentation',
    customCss: `
      .swagger-ui .topbar { display: none }
      .swagger-ui .info .title { font-size: 2em }
    `,
    customFavIcon: 'https://myapp.com/favicon.ico',
  });

  console.log(`Swagger docs: http://localhost:${config.get('PORT')}/docs`);
}

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  const configService = app.get(ConfigService);
  await setupSwagger(app, configService);
  await app.listen(configService.get('PORT', 3000));
}
```

### Documenting DTOs and responses

```typescript
// src/users/dto/create-user.dto.ts

import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';

export class CreateUserDto {
  @ApiProperty({
    description: 'User\'s first name',
    example: 'Alice',
    minLength: 2,
    maxLength: 50,
  })
  @IsString()
  @MinLength(2)
  firstName: string;

  @ApiProperty({
    description: 'User\'s last name',
    example: 'Smith',
  })
  @IsString()
  lastName: string;

  @ApiProperty({
    description: 'User email address — must be unique',
    example: 'alice@example.com',
    format: 'email',
  })
  @IsEmail()
  email: string;

  @ApiProperty({
    description: 'Password — min 8 chars, must include uppercase, lowercase, number, special char',
    example: 'SecurePass@123',
    minLength: 8,
    format: 'password',
    // format: 'password' → Swagger UI renders as password field (hidden text)
  })
  @IsString()
  @MinLength(8)
  password: string;

  @ApiPropertyOptional({
    // @ApiPropertyOptional = @ApiProperty({ required: false })
    description: 'User role (only admins can set this)',
    enum: Role,
    default: Role.USER,
    example: Role.USER,
  })
  @IsEnum(Role)
  @IsOptional()
  role?: Role;
}

// src/users/entities/user.entity.ts — document the response schema
export class UserResponseDto {
  @ApiProperty({ example: 1 })
  id: number;

  @ApiProperty({ example: 'Alice' })
  firstName: string;

  @ApiProperty({ example: 'Smith' })
  lastName: string;

  @ApiProperty({ example: 'alice@example.com' })
  email: string;

  @ApiProperty({ enum: Role, example: Role.USER })
  role: Role;

  @ApiProperty({ example: true })
  isActive: boolean;

  @ApiProperty({ example: '2024-01-01T10:00:00.000Z' })
  createdAt: Date;
}

// src/users/users.controller.ts — document endpoints
@ApiTags('Users')
@ApiBearerAuth('JWT-auth')
@Controller('users')
export class UsersController {

  @ApiOperation({
    summary: 'Get all users (paginated)',
    description: 'Returns a paginated list of users. Requires ADMIN role.',
  })
  @ApiQuery({ name: 'page', required: false, type: Number, example: 1 })
  @ApiQuery({ name: 'limit', required: false, type: Number, example: 20 })
  @ApiQuery({ name: 'search', required: false, type: String })
  @ApiResponse({
    status: 200,
    description: 'Users retrieved successfully',
    schema: {
      properties: {
        data: {
          type: 'array',
          items: { $ref: '#/components/schemas/UserResponseDto' },
        },
        meta: {
          type: 'object',
          properties: {
            total: { type: 'number', example: 100 },
            page: { type: 'number', example: 1 },
            limit: { type: 'number', example: 20 },
            totalPages: { type: 'number', example: 5 },
          },
        },
      },
    },
  })
  @ApiResponse({ status: 401, description: 'Unauthorized' })
  @ApiResponse({ status: 403, description: 'Forbidden - insufficient permissions' })
  @Get()
  findAll(@Query() query: FilterUserDto) {}
}
```

---

## 9. Graceful Shutdown

```typescript
// src/main.ts — complete graceful shutdown

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Enable shutdown hooks — required for OnModuleDestroy to work
  app.enableShutdownHooks();
  // Without this: process.on('SIGTERM') and 'SIGINT' are not handled
  // NestJS calls onModuleDestroy() for providers that implement it

  await app.listen(3000);

  if (process.send) {
    process.send('ready');  // Signal PM2 ready for traffic
  }
}

// Each service that needs cleanup implements OnModuleDestroy:

@Injectable()
export class DatabaseService implements OnModuleInit, OnModuleDestroy {
  async onModuleInit() {
    await this.connect();
    console.log('Database connected');
  }

  async onModuleDestroy() {
    // Called when SIGTERM/SIGINT received
    await this.disconnect();
    // Finish in-flight queries, close connection pool
    console.log('Database disconnected gracefully');
  }
}

@Injectable()
export class QueueService implements OnModuleDestroy {
  async onModuleDestroy() {
    // Stop accepting new jobs, finish current ones
    await this.emailQueue.pause();
    await this.emailQueue.close();
    console.log('Queue processor stopped gracefully');
  }
}
```

---

## 10. CI/CD Pipeline Basics

### GitHub Actions workflow

```yaml
# .github/workflows/ci.yml
# Runs on every push and pull request

name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ── Job 1: Test ──────────────────────────────────────────────────────────
  test:
    runs-on: ubuntu-latest
    
    services:
      # Start PostgreSQL for integration tests
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_password
          POSTGRES_DB: test_db
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          # Cache npm dependencies between runs

      - name: Install dependencies
        run: npm ci

      - name: Run linting
        run: npm run lint

      - name: Run unit tests
        run: npm run test:cov
        env:
          NODE_ENV: test

      - name: Run e2e tests
        run: npm run test:e2e
        env:
          NODE_ENV: test
          DB_HOST: localhost
          DB_PORT: 5432
          DB_USERNAME: test_user
          DB_PASSWORD: test_password
          DB_DATABASE: test_db
          REDIS_HOST: localhost
          JWT_SECRET: test-secret-at-least-32-characters-long
          JWT_REFRESH_SECRET: test-refresh-secret-32-chars-long

      - name: Upload coverage report
        uses: codecov/codecov-action@v4
        with:
          file: ./coverage/lcov.info

  # ── Job 2: Build Docker image ────────────────────────────────────────────
  build:
    needs: test
    # Only build if tests pass
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    # Only build on push (not PRs)

    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}

    steps:
      - uses: actions/checkout@v4

      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=sha,prefix=sha-
            type=semver,pattern={{version}}
          # Tags: 'main', 'sha-abc123', '1.2.3'

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          # GitHub Actions cache for Docker layers — faster builds

  # ── Job 3: Deploy to Staging ─────────────────────────────────────────────
  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    environment: staging
    # GitHub environment: has protection rules and deployment tracking

    steps:
      - name: Deploy to staging
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USER }}
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            cd /app
            echo ${{ secrets.GITHUB_TOKEN }} | docker login ghcr.io -u ${{ github.actor }} --password-stdin
            docker pull ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:develop
            docker-compose -f docker-compose.staging.yml up -d --no-deps api
            sleep 10
            curl -f http://localhost:3000/health || exit 1
            # Health check after deploy — fail if unhealthy

  # ── Job 4: Deploy to Production ──────────────────────────────────────────
  deploy-production:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    # Manual approval required for production (GitHub environment setting)

    steps:
      - name: Deploy to production
        run: |
          # Example: trigger deployment via webhook to your infrastructure
          curl -X POST \
            -H "Authorization: Bearer ${{ secrets.DEPLOY_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d '{"image": "${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}"}' \
            https://deploy.myapp.com/webhook/deploy
```

---

## 11. Production Checklist

```typescript
// Complete production readiness checklist for NestJS apps

const productionChecklist = {
  security: [
    '✓ Helmet middleware (security headers)',
    '✓ CORS configured with specific origins',
    '✓ Rate limiting with Redis storage',
    '✓ JWT with short expiry + refresh rotation',
    '✓ Passwords hashed with bcrypt (cost 12)',
    '✓ No sensitive data in JWT payload',
    '✓ HTTPS enforced (TLS termination at load balancer)',
    '✓ SQL injection prevented (TypeORM parameterized queries)',
    '✓ XSS prevented (Content-Security-Policy header)',
    '✓ .env files in .gitignore',
    '✓ Secrets in environment variables, not code',
    '✓ Input validation with ValidationPipe (whitelist, forbidNonWhitelisted)',
  ],

  performance: [
    '✓ Gzip compression enabled',
    '✓ Redis caching for expensive queries',
    '✓ Database indexes on frequently queried columns',
    '✓ Connection pooling configured (DB + Redis)',
    '✓ PM2 cluster mode (all CPU cores used)',
    '✓ N+1 query prevention (relations, QueryBuilder)',
    '✓ Background job queues for slow operations (email, reports)',
    '✓ DB_SYNCHRONIZE=false in production',
    '✓ Migrations instead of synchronize',
  ],

  reliability: [
    '✓ Health check endpoints (liveness + readiness)',
    '✓ Graceful shutdown with onModuleDestroy',
    '✓ Database connection retry with exponential backoff',
    '✓ Circuit breaker for external services',
    '✓ Bull queues with retry and DLQ',
    '✓ PM2 autorestart on crash',
    '✓ Memory limits enforced (Docker --memory, PM2 max_memory_restart)',
  ],

  observability: [
    '✓ Structured JSON logging (Winston/Pino)',
    '✓ Request logging with correlation IDs',
    '✓ Slow query logging',
    '✓ Error tracking (Sentry)',
    '✓ Metrics collection (Prometheus/Datadog)',
    '✓ Alerts for error rate spikes',
    '✓ Alerts for high memory/CPU',
    '✓ Centralized log aggregation (CloudWatch/Datadog/ELK)',
  ],

  deployment: [
    '✓ Docker multi-stage build (small image size)',
    '✓ Docker Compose for local dev',
    '✓ .dockerignore (exclude node_modules, .env)',
    '✓ CI/CD pipeline (test → build → deploy)',
    '✓ Zero-downtime deployments (PM2 reload or K8s rolling update)',
    '✓ Rollback capability (Docker image tags)',
    '✓ Staging environment mirrors production',
    '✓ Automated e2e tests run before production deploy',
  ],

  documentation: [
    '✓ Swagger/OpenAPI docs generated',
    '✓ All endpoints documented with @ApiOperation',
    '✓ All DTOs documented with @ApiProperty',
    '✓ Auth requirements documented',
    '✓ README with setup instructions',
    '✓ Environment variable documentation (.env.example)',
    '✓ CHANGELOG maintained',
  ],
};
```

---

## 12. Interview Questions & Answers

**Q1: What is a multi-stage Docker build and why is it important for NestJS?**

> "A multi-stage build uses multiple `FROM` instructions in a single Dockerfile, each creating an intermediate image. For NestJS, I use three stages: deps (install only production dependencies), build (install all dependencies including TypeScript and compile the code), and production (minimal runtime image). The final production image copies only the compiled JavaScript from the build stage and production node_modules from deps — excluding the TypeScript compiler, NestJS CLI, jest, and all devDependencies. This results in an image 60-80% smaller than a naive single-stage build, which means faster deployments, less storage cost, and a smaller attack surface. A TypeScript NestJS app typically goes from ~1.5GB to ~250MB with multi-stage builds."

---

**Q2: What is the difference between liveness and readiness probes in Kubernetes?**

> "Liveness probes answer: 'Is this process still running and not stuck?' Kubernetes restarts the pod if the liveness probe fails. It checks fundamental app health — typically just memory usage or that the HTTP server is responding. Readiness probes answer: 'Is this pod ready to receive traffic?' Kubernetes removes the pod from the load balancer if readiness fails — but doesn't restart it. It checks service dependencies — database connectivity, Redis availability, external services. This distinction matters for deployments: when a new pod starts, its readiness probe fails until it finishes connecting to the database, so Kubernetes doesn't route traffic to it yet. Meanwhile, the old pod stays in rotation. Only after the new pod is ready does Kubernetes remove the old one, achieving zero-downtime deployment."

---

**Q3: Why use structured logging (JSON) instead of plain console.log?**

> "Plain console.log produces human-readable text fine for development but is problematic in production. When you have 10 server instances each writing millions of log lines, you need to search, filter, and aggregate logs centrally using tools like CloudWatch, Datadog, or Elasticsearch. These tools parse logs by their format — if you log JSON, they can extract fields like `userId`, `requestId`, `statusCode`, and `duration` automatically. You can then write queries like 'show all errors from userId=123 in the last hour' or 'alert when error rate exceeds 5%'. With plain text you'd need complex regex patterns. Structured logging also enables log correlation — by including a `requestId` in every log line for a request, you can trace a single request across multiple services in a microservices architecture."

---

**Q4: How do you handle secrets in a production Docker deployment?**

> "Secrets should never be baked into Docker images — they'd be visible in image layers and accessible to anyone who can pull the image. There are several production patterns. For Kubernetes, I use Kubernetes Secrets which are stored in etcd (optionally encrypted) and injected as environment variables or mounted as files into pods. For Docker on VMs, I use Docker Swarm Secrets or mount them from the host machine. For cloud deployments, I use the cloud provider's secrets service — AWS Secrets Manager, GCP Secret Manager, or Azure Key Vault — with the app fetching secrets at startup via SDK calls. For all approaches, the .env file is only for local development and is always in .gitignore. The Dockerfile never includes ENV instructions with real values — only defaults or placeholders. Runtime secrets come from the orchestration layer."

---

**Q5: What is graceful shutdown and why does it matter?**

> "Graceful shutdown means the application finishes its in-progress work before exiting, rather than being killed immediately. When Kubernetes or PM2 wants to stop a pod/process, it sends SIGTERM. A graceful app: stops accepting new HTTP requests, waits for in-flight requests to complete (typically with a timeout of 30 seconds), pauses queue workers from picking new jobs, finishes current queue jobs, flushes log buffers, and closes database connections cleanly. Without graceful shutdown: in-flight requests get connection reset errors, current queue jobs are abandoned (and may be partially completed causing data corruption), database connections are dropped mid-transaction potentially leaving locks. NestJS supports this with `app.enableShutdownHooks()` which calls `onModuleDestroy()` on providers that implement it. PM2's `kill_timeout` and `wait_ready` options coordinate with this."

---

**Q6: Describe your CI/CD pipeline for a NestJS application.**

> "My typical pipeline has four stages. First, the test stage: runs on every push and PR, installs dependencies, runs ESLint, unit tests with coverage, and e2e tests against real PostgreSQL and Redis services (using GitHub Actions service containers). If any fail, the pipeline stops. Second, the build stage: on push to main/develop, builds a multi-stage Docker image, tags it with the git SHA and branch name, and pushes to GitHub Container Registry. Third, staging deployment: automatically deploys to staging on pushes to the develop branch — pulls the image, does a rolling restart, and verifies the health endpoint returns 200. Fourth, production deployment: triggered on merges to main, requires manual approval via GitHub Environments. It uses zero-downtime deployment — either PM2 reload or Kubernetes rolling update. I also maintain the ability to roll back by keeping previous image tags in the registry. The whole pipeline from push to staging deployment takes about 5-7 minutes."

---

**Q7: What do you include in a health check endpoint?**

> "I implement two separate endpoints: `/health/liveness` and `/health/readiness`. Liveness checks only what indicates the process has gone zombie — primarily memory usage (heap under 500MB, RSS under 1GB). Restarting on memory issues is correct behavior. Readiness checks everything needed to serve traffic — database ping (SELECT 1), Redis ping, and any critical external service the app depends on synchronously. I make the health endpoint public (bypass JWT auth) since load balancers and monitoring systems don't have tokens. I also suppress health check request logging (too noisy — a health check every 30 seconds adds up). The response includes each check's status individually so dashboards can show which component is degraded. For custom dependencies like Redis, I write a custom HealthIndicator extending `@nestjs/terminus`'s base class."

---

## Quick Reference — Day 15 Cheat Sheet

```
Dockerfile multi-stage:
  FROM node:20-alpine AS deps   → npm ci --only=production
  FROM node:20-alpine AS build  → npm ci + npm run build
  FROM node:20-alpine AS prod   → copy from deps + build, run as non-root user
  CMD ["node", "dist/main"]     → NEVER 'nest start' in production

.dockerignore essentials:
  node_modules, dist, .git, .env, *.spec.ts, coverage, *.log

Docker Compose services:
  api, postgres, redis, rabbitmq (each with healthcheck + volumes)
  Service names = hostnames (DB_HOST: postgres in docker network)

Winston/Pino:
  Development: colorized human-readable (pino-pretty / winston colorize)
  Production: JSON structured logs (searchable in CloudWatch/Datadog)
  Always include: requestId, userId, statusCode, duration
  Never log: passwords, tokens, credit cards (use redact in Pino)

Health checks (@nestjs/terminus):
  GET /health/liveness  → memory checks (restart if OOM)
  GET /health/readiness → DB + Redis + disk (remove from LB if down)
  GET /health           → everything (for monitoring dashboards)
  @Public() on health controller — load balancers have no auth token

Swagger:
  DocumentBuilder: title, version, bearerAuth, tags, servers
  @ApiTags() on controller, @ApiBearerAuth() for protected
  @ApiProperty({ example, description }) on DTO fields
  @ApiOperation({ summary, description }) on methods
  SwaggerModule.setup('docs', app, document)
  ENABLE_SWAGGER=false to disable in production

Graceful shutdown:
  app.enableShutdownHooks()        → enables SIGTERM handling
  implements OnModuleDestroy        → cleanup in onModuleDestroy()
  process.send('ready')             → PM2 zero-downtime signal
  DB, Redis, Queue connections closed in onModuleDestroy

CI/CD stages:
  1. test    → lint + unit tests + e2e tests (block on failure)
  2. build   → docker build + push to registry (tag with git SHA)
  3. staging → auto-deploy on develop branch push
  4. prod    → manual approval required, rolling update

Production checklist categories:
  Security: helmet, CORS, rate limiting, bcrypt, HTTPS, validation
  Performance: compression, caching, indexes, PM2 cluster, queues
  Reliability: health checks, graceful shutdown, retries, circuit breaker
  Observability: structured logs, request IDs, error tracking, alerts
  Deployment: Docker, CI/CD, zero-downtime, rollback, staging env
```

---

## 🎉 All 15 Days Complete!

You now have comprehensive coverage of everything a 3-year NestJS backend engineer needs to know:

```
Week 1 — Foundations:
  Day 1:  Node.js & TypeScript fundamentals
  Day 2:  NestJS core architecture (modules, controllers, providers, DI)
  Day 3:  Request lifecycle (middleware, guards, interceptors, pipes, filters)
  Day 4:  DTOs, validation & serialization
  Day 5:  Configuration & environment management
  Day 6:  Database with TypeORM (entities, relations, QueryBuilder, migrations)
  Day 7:  Database with Prisma & MongoDB

Week 2 — Intermediate:
  Day 8:  Authentication (JWT, Passport, bcrypt, refresh tokens)
  Day 9:  Authorization (RBAC, ABAC, ownership checks, CASL)
  Day 10: REST API best practices (versioning, pagination, rate limiting)
  Day 11: WebSockets & real-time communication
  Day 12: Microservices & message brokers (RabbitMQ, Kafka)

Week 3 — Advanced & Production:
  Day 13: Testing (unit tests, mocking, integration, e2e with Supertest)
  Day 14: Performance, caching & queues (Redis, Bull, PM2)
  Day 15: Deployment, logging & monitoring (Docker, Winston, health checks, CI/CD)
```

**Interview preparation tips:**
- Practice explaining each concept out loud (not just reading)
- Code at least 3 days from scratch without notes
- Focus on WHY decisions are made, not just HOW
- Know the trade-offs: TypeORM vs Prisma, sync vs async, RBAC vs ABAC
- Be ready to whiteboard the request lifecycle and auth flow
- Know the production checklist by heart — shows real-world experience

Good luck with your interviews! 🚀
