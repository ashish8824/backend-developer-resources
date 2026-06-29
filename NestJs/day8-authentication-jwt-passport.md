# Day 8 — Authentication with JWT & Passport
### NestJS Interview Prep | 3-Year Backend Engineer Level

> **Goal for today:** Master the complete authentication system in NestJS — from user login to JWT token generation, route protection, refresh token rotation, and bcrypt password hashing. This is one of the MOST asked topics in Node.js backend interviews. A 3-year engineer must implement this from scratch and explain every decision.

---

## 🧭 First-Timer Guidance — Read This First

**What is Authentication vs Authorization?**

```
Authentication → WHO are you?
  "Prove you are Alice by providing your email and password"
  After proof: you get a TOKEN (like a hotel key card)

Authorization → WHAT can you do?
  "Alice is allowed to edit her own profile, but not delete other users"
  Covered in Day 9 (RBAC)
```

**The complete auth flow we build today:**

```
1. REGISTER:
   Client → POST /auth/register { email, password }
   Server → hash password, save user, return tokens

2. LOGIN:
   Client → POST /auth/login { email, password }
   Server → verify password, return { accessToken, refreshToken }

3. ACCESS PROTECTED ROUTE:
   Client → GET /users/profile
             Authorization: Bearer <accessToken>
   Server → verify token, extract user, return profile

4. REFRESH TOKENS:
   Client → POST /auth/refresh
             Authorization: Bearer <refreshToken>
   Server → verify refresh token, return NEW { accessToken, refreshToken }

5. LOGOUT:
   Client → POST /auth/logout
             Authorization: Bearer <refreshToken>
   Server → invalidate refresh token, return success
```

**Why two tokens? (Access + Refresh)**

```
Access Token:
  - Short-lived: 15 minutes
  - Used for EVERY API request
  - If stolen: attacker has access for max 15 minutes
  - NOT stored in database — verified by signature only

Refresh Token:
  - Long-lived: 7 days
  - Used ONLY to get a new access token
  - Stored in database — can be revoked (logout, security breach)
  - If stolen: can be invalidated by deleting from database
```

**Setup — install packages:**
```bash
npm install @nestjs/passport passport passport-local passport-jwt
npm install @nestjs/jwt
npm install bcrypt
npm install @types/passport-local @types/passport-jwt @types/bcrypt --save-dev
```

---

## Table of Contents

1. [Passport.js — How It Works in NestJS](#1-passportjs--how-it-works-in-nestjs)
2. [bcrypt — Password Hashing](#2-bcrypt--password-hashing)
3. [JWT — JSON Web Tokens Explained](#3-jwt--json-web-tokens-explained)
4. [Local Strategy — Email + Password Login](#4-local-strategy--email--password-login)
5. [JWT Strategy — Protecting Routes](#5-jwt-strategy--protecting-routes)
6. [AuthService — Business Logic](#6-authservice--business-logic)
7. [AuthController — HTTP Endpoints](#7-authcontroller--http-endpoints)
8. [AuthModule — Wiring Everything Together](#8-authmodule--wiring-everything-together)
9. [Refresh Token System](#9-refresh-token-system)
10. [Complete Auth Flow — End to End](#10-complete-auth-flow--end-to-end)
11. [Security Best Practices](#11-security-best-practices)
12. [Interview Questions & Answers](#12-interview-questions--answers)

---

## 1. Passport.js — How It Works in NestJS

### What is Passport?

Passport is an **authentication middleware** for Node.js. It provides a standard interface for many authentication methods (called "strategies"):

- `passport-local` → username + password login
- `passport-jwt` → JWT token verification
- `passport-google-oauth20` → Google OAuth login
- `passport-github` → GitHub OAuth login
- Over 500 strategies available

**The key concept: a Strategy is a class that validates credentials and either succeeds (returns a user) or fails (throws an error).**

### How Passport integrates with NestJS

```
HTTP Request with credentials
         ↓
  AuthGuard('strategy-name')
         ↓
  Passport runs the Strategy
         ↓
  Strategy.validate() method
         ↓
  Returns user object → attached to request.user
  OR throws UnauthorizedException → request blocked
         ↓
  Controller method receives clean request.user
```

```typescript
// The AuthGuard factory — NestJS wraps Passport strategies as Guards
import { AuthGuard } from '@nestjs/passport';

// AuthGuard('local') → runs LocalStrategy
// AuthGuard('jwt') → runs JwtStrategy
// These string names MUST match the name in the strategy class

@UseGuards(AuthGuard('local'))
@Post('login')
login(@Req() req: Request) {
  // If LocalStrategy.validate() succeeds, req.user is set
  // If it fails, 401 Unauthorized is returned automatically
  return req.user;
}
```

### Strategy naming

```typescript
// Default: strategy name = 'local', 'jwt', 'google', etc.
// You can override with @PassportStrategy(Strategy, 'custom-name')

// NestJS convention: create custom guard classes for cleaner code
@Injectable()
export class LocalAuthGuard extends AuthGuard('local') {}
// Now use @UseGuards(LocalAuthGuard) instead of @UseGuards(AuthGuard('local'))
// Cleaner, more readable, easier to add extra logic later
```

---

## 2. bcrypt — Password Hashing

### Why hash passwords?

```
NEVER store passwords as plain text. If your database is breached:
  Plain text: "password123" → attacker has everyone's password immediately
  MD5/SHA1:   "482c811da5d5b4bc6d497ffa98491e38" → cracked in seconds (rainbow tables)
  bcrypt:     "$2b$12$EixZaYVK1fsbw1ZfbX3OXePaWxn96p36WQoeG6Lruj3vjPGga31lW"
              → takes years to crack even with GPU clusters (cost factor)
```

### How bcrypt works

```
bcrypt(password, saltRounds):
  1. Generate a RANDOM SALT (16 bytes of randomness)
  2. Combine: password + salt
  3. Run expensive hashing algorithm bcrypt_encrypt(password+salt) × 2^saltRounds times
  4. Encode: $2b$12$<22-char-salt><31-char-hash>
              ↑    ↑
              version  cost factor (2^12 = 4096 iterations)

SAME password + DIFFERENT salt = DIFFERENT hash
"password123" → "$2b$12$abc...xyz"  (different every time!)
"password123" → "$2b$12$def...mno"

This defeats rainbow table attacks — you can't precompute hashes
```

```typescript
// src/common/utils/bcrypt.util.ts

import * as bcrypt from 'bcrypt';

// ── Hashing ────────────────────────────────────────────────────────────────
export async function hashPassword(password: string): Promise<string> {
  const SALT_ROUNDS = 12;
  // saltRounds (cost factor): controls how slow the hashing is
  // Higher = more secure, but slower
  // 
  // Cost factor benchmarks (on typical server):
  // 10 → ~100ms per hash  (minimum acceptable)
  // 12 → ~300ms per hash  (recommended for most apps)
  // 14 → ~1000ms per hash (high security apps)
  //
  // 300ms for login is acceptable UX
  // But means an attacker can only try ~3 passwords/second
  // vs billions per second without bcrypt

  return bcrypt.hash(password, SALT_ROUNDS);
  // Returns: "$2b$12$<salt><hash>"
  // The hash INCLUDES the salt — you don't need to store salt separately
}

// ── Comparing ──────────────────────────────────────────────────────────────
export async function comparePassword(
  plaintext: string,
  hash: string,
): Promise<boolean> {
  return bcrypt.compare(plaintext, hash);
  // bcrypt.compare():
  // 1. Extracts salt from the stored hash ($2b$12$<22-char-salt>)
  // 2. Hashes plaintext with that same salt
  // 3. Compares — returns true if match, false if not
  // Returns false (doesn't throw) if hash is invalid

  // NEVER do: hash(plaintext) === storedHash
  // This is vulnerable to timing attacks!
  // bcrypt.compare() uses constant-time comparison
}

// ── Timing attack prevention note ─────────────────────────────────────────
// bcrypt.compare() is safe against timing attacks because:
// The comparison time is determined by the bcrypt algorithm, not string length
// An attacker can't determine if they're "getting warmer" by measuring response time
```

---

## 3. JWT — JSON Web Tokens Explained

### What is a JWT?

A JWT is a **self-contained, signed token** that proves who you are without the server needing to look up a database.

```
JWT Structure:
  eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9    ← Header (base64)
  .eyJ1c2VySWQiOjEsImVtYWlsIjoiYWxpY2VAZ  ← Payload (base64)
  .SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c  ← Signature

Header (decoded): { "alg": "HS256", "typ": "JWT" }
  alg: which algorithm signs the token (HS256 = HMAC-SHA256)

Payload (decoded): {
  "sub": "1",             ← subject (user ID)
  "email": "alice@test.com",
  "role": "user",
  "iat": 1704067200,      ← issued at (Unix timestamp)
  "exp": 1704068100       ← expires at (15 minutes after iat)
}

Signature: HMACSHA256(
  base64(header) + "." + base64(payload),
  JWT_SECRET
)
// Signature proves the token wasn't tampered with
// Only the server with JWT_SECRET can create valid signatures
```

### JWT verification flow

```
Client sends: Authorization: Bearer eyJhbGci...

Server:
1. Split on '.' → [header, payload, signature]
2. Recompute: HMAC(header + '.' + payload, JWT_SECRET)
3. Compare computed signature with provided signature
4. If match → token is authentic (not tampered)
5. Check 'exp' field → is token expired?
6. If valid → extract payload, attach to request.user
7. If invalid → throw 401 Unauthorized

NO DATABASE LOOKUP NEEDED for access tokens!
This is why JWTs are fast for stateless auth.
```

### JWT payload — what to put in it

```typescript
// The JWT payload (claims) — what you embed in the token

interface JwtPayload {
  sub: string;     // 'sub' (subject) = user ID — standard JWT claim
  // Use 'sub' not 'userId' — follows JWT standard (RFC 7519)

  email: string;   // Useful for logging, doesn't need DB lookup

  role: string;    // For basic authorization checks in guards

  // iat: number;  // "Issued At" — auto-added by @nestjs/jwt
  // exp: number;  // "Expires At" — auto-calculated from expiresIn

  // WHAT NOT TO PUT in JWT:
  // ❌ password (ever)
  // ❌ sensitive data (SSN, credit card)
  // ❌ Large objects (increases token size, sent on every request)
  // ❌ Data that changes frequently (you can't update token without reissue)
  // ✅ Only: identifiers, roles, stable profile data
}

// Remember: JWT payload is BASE64 ENCODED, not encrypted!
// Anyone can decode it — don't put secrets in payload
// Security comes from the SIGNATURE, not the encoding
```

---

## 4. Local Strategy — Email + Password Login

### Creating the LocalStrategy

```typescript
// src/auth/strategies/local.strategy.ts

import { Injectable, UnauthorizedException } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { Strategy } from 'passport-local';
// 'Strategy' from passport-local = the Local strategy implementation

@Injectable()
export class LocalStrategy extends PassportStrategy(Strategy, 'local') {
  // PassportStrategy(Strategy): NestJS wrapper that makes Passport strategies
  // work as injectable NestJS providers
  // Second argument 'local': the name used in AuthGuard('local')
  // Default is already 'local' for passport-local, but being explicit is good

  constructor(private readonly authService: AuthService) {
    super({
      // Options for passport-local Strategy:
      usernameField: 'email',
      // Default: passport-local looks for 'username' field in request body
      // We use 'email' — this tells passport-local to use req.body.email
      // as the first argument to validate()

      passwordField: 'password',
      // Default is already 'password', but being explicit

      // passReqToCallback: true,
      // If true: validate() receives (req, email, password)
      // Useful if you need req.headers or other request data in validate()
    });
  }

  // validate() is the CORE METHOD — Passport calls this automatically
  // Passport extracts email and password from req.body (using usernameField/passwordField)
  // and passes them as arguments
  async validate(email: string, password: string): Promise<any> {
    // 1. Find user by email
    // 2. Verify password matches hash
    // 3. Return user if valid, throw if not

    const user = await this.authService.validateUser(email, password);
    // authService.validateUser handles all the verification logic

    if (!user) {
      // Passport expects either a return value (success) or an exception (failure)
      // If you return null/false, Passport throws 401 Unauthorized automatically
      // But throwing explicitly gives better control over the error message
      throw new UnauthorizedException('Invalid email or password');
      // Important: NEVER say "email not found" or "wrong password"
      // Always say "Invalid email or password" — don't reveal which is wrong
      // This prevents user enumeration attacks
    }

    // Return the user object
    // Passport attaches this to request.user
    // Your controller can then access request.user or use @CurrentUser()
    return user;
  }
}
```

### Creating the LocalAuthGuard

```typescript
// src/auth/guards/local-auth.guard.ts

import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class LocalAuthGuard extends AuthGuard('local') {
  // Extends AuthGuard('local') — runs LocalStrategy when applied
  // Having a named class is better than @UseGuards(AuthGuard('local'))
  // because:
  // 1. More readable: @UseGuards(LocalAuthGuard) vs @UseGuards(AuthGuard('local'))
  // 2. Can add extra logic (override canActivate, handleRequest)
  // 3. Can inject services for additional checks

  // You can override handleRequest() to customize error handling:
  // handleRequest(err: any, user: any, info: any) {
  //   if (err || !user) {
  //     throw err || new UnauthorizedException(info?.message);
  //   }
  //   return user;
  // }
}
```

---

## 5. JWT Strategy — Protecting Routes

### Creating the JwtStrategy

```typescript
// src/auth/strategies/jwt.strategy.ts

import { Injectable, UnauthorizedException } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
// ExtractJwt: utility to extract token from different places
// Strategy: the JWT strategy implementation

import { ConfigService } from '@nestjs/config';
import { UsersService } from '../../users/users.service';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy, 'jwt') {
  constructor(
    private readonly configService: ConfigService,
    private readonly usersService: UsersService,
  ) {
    super({
      // ── How to extract the JWT from the request ────────────────────────
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      // Looks in: Authorization: Bearer <token>
      // This is the standard — ALWAYS use Bearer tokens for API auth
      //
      // Other options (rarely used):
      // ExtractJwt.fromUrlQueryParameter('token') → ?token=xxx (insecure!)
      // ExtractJwt.fromHeader('x-auth-token') → custom header
      // ExtractJwt.fromBodyField('token') → request body field
      // ExtractJwt.fromExtractors([fn1, fn2]) → try multiple extractors

      // ── What to do with expired tokens ────────────────────────────────
      ignoreExpiration: false,
      // false (default): if token is expired → 401 Unauthorized
      // true: accept expired tokens (only for special refresh endpoints!)
      // NEVER set true globally — defeats the purpose of expiry

      // ── The secret used to verify the signature ────────────────────────
      secretOrKey: configService.getOrThrow<string>('JWT_SECRET'),
      // This MUST match the secret used to SIGN the token
      // If secrets differ → signature verification fails → 401

      // For RS256 (asymmetric): use secretOrKeyProvider with public key
      // secretOrKey: fs.readFileSync('public.key')  ← public key for verification
      // Signing uses private key, verification uses public key
    });
  }

  // validate() is called AFTER Passport verifies the JWT signature and expiry
  // The decoded payload is passed as argument
  async validate(payload: JwtPayload): Promise<User> {
    // payload: { sub: '1', email: 'alice@test.com', role: 'user', iat: ..., exp: ... }
    // This payload was extracted from the token — we know it's authentic
    // because Passport already verified the signature

    // OPTION 1: Trust the payload completely (stateless — fastest)
    // return { id: payload.sub, email: payload.email, role: payload.role };
    // No database lookup — works for basic auth

    // OPTION 2: Fetch fresh user from database (more secure)
    const user = await this.usersService.findOne(parseInt(payload.sub));
    // Why fetch from DB?
    // - Get latest user data (role might have changed after token was issued)
    // - Check if user is still active (might have been deactivated)
    // - Check if token was explicitly revoked (blacklist pattern)
    //
    // Trade-off: DB lookup on EVERY request vs fresh data
    // For most apps: trust the payload (Option 1) for performance
    // For high-security apps: always fetch from DB (Option 2)

    if (!user || !user.isActive) {
      throw new UnauthorizedException('User not found or inactive');
    }

    return user;
    // This returned user is attached to request.user
    // Available in controllers via @Req() req → req.user
    // Or via @CurrentUser() custom decorator (Day 3)
  }
}

// The JWT payload type
interface JwtPayload {
  sub: string;      // User ID as string
  email: string;
  role: string;
  iat: number;      // Issued at
  exp: number;      // Expires at
}
```

### JwtAuthGuard

```typescript
// src/auth/guards/jwt-auth.guard.ts

import { Injectable, ExecutionContext, UnauthorizedException } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { AuthGuard } from '@nestjs/passport';
import { IS_PUBLIC_KEY } from '../decorators/public.decorator';
import { Observable } from 'rxjs';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private reflector: Reflector) {
    super();
  }

  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    // Check if the route is marked as @Public()
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),  // Method-level decorator
      context.getClass(),    // Class-level decorator
    ]);

    // Public routes skip JWT verification entirely
    if (isPublic) return true;

    // Non-public routes: run Passport JWT strategy
    return super.canActivate(context);
    // super.canActivate() runs JwtStrategy.validate()
    // Returns true if token is valid, throws 401 if not
  }

  // Override handleRequest to customize error messages
  handleRequest(err: any, user: any, info: any) {
    if (err || !user) {
      // info: Passport error info
      // info.message: 'jwt expired', 'jwt malformed', 'No auth token', etc.

      if (info?.name === 'TokenExpiredError') {
        throw new UnauthorizedException('Token has expired. Please refresh your session.');
      }
      if (info?.name === 'JsonWebTokenError') {
        throw new UnauthorizedException('Invalid token.');
      }
      if (info?.message === 'No auth token') {
        throw new UnauthorizedException('Authentication token is required.');
      }

      throw err || new UnauthorizedException('Authentication failed.');
    }
    return user;
  }
}
```

### The @Public() decorator

```typescript
// src/auth/decorators/public.decorator.ts

import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';

export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
// Usage: @Public() on routes that don't require authentication
// The JwtAuthGuard reads this metadata and skips JWT check

// Usage in controller:
// @Public()
// @Post('register')
// register(@Body() dto: RegisterDto) { ... }
```

---

## 6. AuthService — Business Logic

```typescript
// src/auth/auth.service.ts

import {
  Injectable,
  UnauthorizedException,
  ConflictException,
  BadRequestException,
} from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { ConfigService } from '@nestjs/config';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import * as bcrypt from 'bcrypt';
import { User } from '../users/entities/user.entity';
import { RegisterDto } from './dto/register.dto';
import { LoginResponseDto } from './dto/login-response.dto';

@Injectable()
export class AuthService {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
    private readonly jwtService: JwtService,
    private readonly configService: ConfigService,
  ) {}

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // CALLED BY LOCAL STRATEGY — validates credentials
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  async validateUser(email: string, password: string): Promise<User | null> {
    // Step 1: Find user by email
    // addSelect('user.password'): override @Column({ select: false })
    // We need the password hash for comparison
    const user = await this.userRepository
      .createQueryBuilder('user')
      .where('user.email = :email', { email: email.toLowerCase() })
      .addSelect('user.password')   // Explicitly include password (select: false on entity)
      .getOne();

    if (!user) {
      // SECURITY: Don't reveal that email doesn't exist
      // Still perform a bcrypt comparison to prevent timing attacks
      // (takes same time whether user exists or not)
      await bcrypt.compare(password, '$2b$12$dummyhashforTimingAttackPrevention');
      return null;
    }

    // Step 2: Compare provided password with stored hash
    const isPasswordValid = await bcrypt.compare(password, user.password);

    if (!isPasswordValid) {
      return null;
    }

    // Step 3: Check account is active
    if (!user.isActive) {
      throw new UnauthorizedException(
        'Your account has been deactivated. Please contact support.',
      );
    }

    // Remove password from returned object — never pass it further
    const { password: _, ...userWithoutPassword } = user;
    return userWithoutPassword as User;
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // REGISTER
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  async register(registerDto: RegisterDto): Promise<LoginResponseDto> {
    const { email, password, firstName, lastName } = registerDto;

    // Check if email already exists
    const existingUser = await this.userRepository.findOne({
      where: { email: email.toLowerCase() },
    });

    if (existingUser) {
      throw new ConflictException(
        'An account with this email already exists.',
      );
    }

    // Hash password with bcrypt
    const hashedPassword = await bcrypt.hash(password, 12);

    // Create and save new user
    const user = this.userRepository.create({
      email: email.toLowerCase().trim(),
      password: hashedPassword,
      firstName: firstName.trim(),
      lastName: lastName.trim(),
    });

    const savedUser = await this.userRepository.save(user);

    // Generate tokens and return
    // Same as login — register = implicit login
    return this.generateTokens(savedUser);
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // LOGIN — called after LocalStrategy validates credentials
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  async login(user: User): Promise<LoginResponseDto> {
    // user is already validated by LocalStrategy.validate()
    // Just generate and return tokens

    const tokens = await this.generateTokens(user);

    // Update last login timestamp
    await this.userRepository.update(user.id, {
      lastLoginAt: new Date(),
    });

    return tokens;
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // TOKEN GENERATION
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  async generateTokens(user: User): Promise<LoginResponseDto> {
    const payload: JwtPayload = {
      sub: user.id.toString(),
      // 'sub' is the JWT standard claim for subject (who the token is about)
      email: user.email,
      role: user.role,
    };

    // Sign ACCESS token — short lived
    const accessToken = this.jwtService.sign(payload, {
      secret: this.configService.getOrThrow('JWT_SECRET'),
      expiresIn: this.configService.get('JWT_EXPIRES_IN', '15m'),
      // 15m = 15 minutes
    });

    // Sign REFRESH token — long lived, different secret
    const refreshToken = this.jwtService.sign(payload, {
      secret: this.configService.getOrThrow('JWT_REFRESH_SECRET'),
      // DIFFERENT secret for refresh tokens!
      // If access token secret is compromised, refresh tokens are still safe
      expiresIn: this.configService.get('JWT_REFRESH_EXPIRES_IN', '7d'),
    });

    // Store HASHED refresh token in database
    // Never store plain refresh tokens — hash for same reason as passwords
    const hashedRefreshToken = await bcrypt.hash(refreshToken, 10);
    // Cost factor 10 (not 12) — refresh token storage doesn't need to be
    // as slow as password hashing, and is done on every login/refresh

    await this.userRepository.update(user.id, {
      refreshToken: hashedRefreshToken,
      // Store hash, not the actual token
    });

    return {
      accessToken,
      refreshToken,
      user: {
        id: user.id,
        email: user.email,
        firstName: user.firstName,
        lastName: user.lastName,
        role: user.role,
      },
    };
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // REFRESH TOKENS
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  async refreshTokens(
    userId: number,
    refreshToken: string,
  ): Promise<LoginResponseDto> {
    // Find user with stored refresh token hash
    const user = await this.userRepository
      .createQueryBuilder('user')
      .where('user.id = :id', { id: userId })
      .addSelect('user.refreshToken')  // select: false — explicitly include
      .getOne();

    if (!user || !user.refreshToken) {
      throw new UnauthorizedException('Access denied. Please log in again.');
    }

    // Verify the provided refresh token matches the stored hash
    const isRefreshTokenValid = await bcrypt.compare(
      refreshToken,
      user.refreshToken,
    );

    if (!isRefreshTokenValid) {
      // Token doesn't match — possible token theft!
      // Clear all tokens for this user (force re-login)
      await this.userRepository.update(userId, { refreshToken: null });
      throw new UnauthorizedException(
        'Refresh token is invalid. Please log in again.',
      );
    }

    // Token is valid — generate NEW tokens (rotation)
    // Old refresh token is now invalid (overwritten in DB)
    return this.generateTokens(user);
    // generateTokens() overwrites refreshToken in DB with new hash
    // This is REFRESH TOKEN ROTATION — each refresh gives a new refresh token
    // If old refresh token is used again → it won't match → security alert
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // LOGOUT
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  async logout(userId: number): Promise<void> {
    // Invalidate refresh token by setting it to null
    // The access token can't be invalidated (stateless)
    // but it expires in 15 minutes anyway
    await this.userRepository.update(userId, {
      refreshToken: null,
    });
    // After this, even if someone has the refresh token, they can't get a new access token
    // The access token they already have will expire in 15 minutes
  }

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // CHANGE PASSWORD
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  async changePassword(
    userId: number,
    currentPassword: string,
    newPassword: string,
  ): Promise<void> {
    const user = await this.userRepository
      .createQueryBuilder('user')
      .where('user.id = :id', { id: userId })
      .addSelect('user.password')
      .getOne();

    // Verify current password
    const isCurrentPasswordValid = await bcrypt.compare(
      currentPassword,
      user.password,
    );

    if (!isCurrentPasswordValid) {
      throw new BadRequestException('Current password is incorrect.');
    }

    // Hash new password
    const hashedNewPassword = await bcrypt.hash(newPassword, 12);

    // Update password AND invalidate all refresh tokens
    // Forces re-login on all devices after password change — security best practice
    await this.userRepository.update(userId, {
      password: hashedNewPassword,
      refreshToken: null,  // Logout from all devices
    });
  }
}

// JWT Payload interface
interface JwtPayload {
  sub: string;
  email: string;
  role: string;
}
```

---

## 7. AuthController — HTTP Endpoints

```typescript
// src/auth/auth.controller.ts

import {
  Controller,
  Post,
  Body,
  UseGuards,
  Req,
  HttpCode,
  HttpStatus,
  Get,
  Patch,
} from '@nestjs/common';
import { Request } from 'express';
import { AuthService } from './auth.service';
import { LocalAuthGuard } from './guards/local-auth.guard';
import { JwtAuthGuard } from './guards/jwt-auth.guard';
import { JwtRefreshGuard } from './guards/jwt-refresh.guard';
import { Public } from './decorators/public.decorator';
import { CurrentUser } from './decorators/current-user.decorator';
import { RegisterDto } from './dto/register.dto';
import { LoginDto } from './dto/login.dto';
import { ChangePasswordDto } from './dto/change-password.dto';
import { User } from '../users/entities/user.entity';

@Controller('auth')
// All routes in this controller are prefixed with /auth
// We're applying JwtAuthGuard globally in AppModule
// So we need @Public() on routes that don't require auth
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  // ── POST /auth/register ────────────────────────────────────────────────
  @Public()
  // @Public(): skip JWT auth guard for this route
  @Post('register')
  @HttpCode(HttpStatus.CREATED)
  async register(@Body() registerDto: RegisterDto) {
    // RegisterDto validated by global ValidationPipe
    // If validation fails → 400 Bad Request before this method runs
    return this.authService.register(registerDto);
    // Returns: { accessToken, refreshToken, user }
  }

  // ── POST /auth/login ───────────────────────────────────────────────────
  @Public()
  @Post('login')
  @HttpCode(HttpStatus.OK)
  @UseGuards(LocalAuthGuard)
  // LocalAuthGuard runs BEFORE this method
  // It calls LocalStrategy.validate(email, password) from request body
  // If valid: sets req.user to the validated User object
  // If invalid: throws 401 Unauthorized (method never called)
  async login(@Req() req: Request) {
    // req.user is the User object returned by LocalStrategy.validate()
    // We pass it to AuthService to generate tokens
    return this.authService.login(req.user as User);
  }

  // ── POST /auth/refresh ─────────────────────────────────────────────────
  @Public()
  @Post('refresh')
  @HttpCode(HttpStatus.OK)
  @UseGuards(JwtRefreshGuard)
  // JwtRefreshGuard: a SEPARATE JWT strategy that uses JWT_REFRESH_SECRET
  // It verifies the refresh token (different from access token)
  // Sets req.user.id and the raw refreshToken on the request
  async refresh(@Req() req: Request) {
    const userId = (req.user as any).id;
    const refreshToken = (req.user as any).refreshToken;
    // The JwtRefreshStrategy extracts the raw token and attaches it to user

    return this.authService.refreshTokens(userId, refreshToken);
    // Returns new { accessToken, refreshToken }
  }

  // ── POST /auth/logout ──────────────────────────────────────────────────
  @Post('logout')
  @HttpCode(HttpStatus.NO_CONTENT)
  // 204: success but no body — standard for logout
  @UseGuards(JwtAuthGuard)
  // Must be authenticated to logout (need to know WHO is logging out)
  async logout(@CurrentUser() user: User) {
    await this.authService.logout(user.id);
    // Clears refresh token from database
    // Client should also delete stored tokens on their end
  }

  // ── GET /auth/me ──────────────────────────────────────────────────────
  @Get('me')
  @UseGuards(JwtAuthGuard)
  getProfile(@CurrentUser() user: User) {
    // @CurrentUser() extracts user from request (set by JwtStrategy)
    // Returns the authenticated user's profile
    return user;
  }

  // ── PATCH /auth/change-password ────────────────────────────────────────
  @Patch('change-password')
  @UseGuards(JwtAuthGuard)
  @HttpCode(HttpStatus.NO_CONTENT)
  async changePassword(
    @CurrentUser() user: User,
    @Body() changePasswordDto: ChangePasswordDto,
  ) {
    await this.authService.changePassword(
      user.id,
      changePasswordDto.currentPassword,
      changePasswordDto.newPassword,
    );
  }
}
```

### DTOs for auth endpoints

```typescript
// src/auth/dto/register.dto.ts

import {
  IsEmail, IsString, MinLength, MaxLength,
  IsNotEmpty, Matches,
} from 'class-validator';
import { Transform } from 'class-transformer';

export class RegisterDto {
  @IsNotEmpty()
  @IsString()
  @MinLength(2)
  @MaxLength(50)
  @Transform(({ value }) => value?.trim())
  firstName: string;

  @IsNotEmpty()
  @IsString()
  @MinLength(2)
  @MaxLength(50)
  @Transform(({ value }) => value?.trim())
  lastName: string;

  @IsEmail({}, { message: 'Please provide a valid email address' })
  @Transform(({ value }) => value?.toLowerCase().trim())
  email: string;

  @IsString()
  @MinLength(8, { message: 'Password must be at least 8 characters' })
  @MaxLength(128)
  @Matches(
    /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])/,
    {
      message:
        'Password must contain at least one uppercase letter, one lowercase letter, one number, and one special character (@$!%*?&)',
    },
  )
  password: string;
}


// src/auth/dto/login.dto.ts

import { IsEmail, IsString, IsNotEmpty } from 'class-validator';
import { Transform } from 'class-transformer';

export class LoginDto {
  @IsEmail()
  @Transform(({ value }) => value?.toLowerCase().trim())
  email: string;

  @IsString()
  @IsNotEmpty()
  password: string;
  // Note: No MinLength/Matches on login
  // We validate the stored hash, not the format of incoming password
  // A valid password from before you changed your requirements should still work
}


// src/auth/dto/login-response.dto.ts

export class LoginResponseDto {
  accessToken: string;
  refreshToken: string;
  user: {
    id: number;
    email: string;
    firstName: string;
    lastName: string;
    role: string;
  };
}


// src/auth/dto/change-password.dto.ts

import { IsString, IsNotEmpty, MinLength, Matches } from 'class-validator';

export class ChangePasswordDto {
  @IsString()
  @IsNotEmpty()
  currentPassword: string;

  @IsString()
  @MinLength(8)
  @MaxLength(128)
  @Matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])/, {
    message: 'New password must be strong (uppercase, lowercase, number, special char)',
  })
  newPassword: string;

  @IsString()
  @IsNotEmpty()
  confirmNewPassword: string;
  // We validate matching in service, not DTO
  // (DTO can't compare two fields easily without custom validator)
}
```

---

## 8. AuthModule — Wiring Everything Together

```typescript
// src/auth/auth.module.ts

import { Module } from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';
import { PassportModule } from '@nestjs/passport';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { TypeOrmModule } from '@nestjs/typeorm';

import { AuthController } from './auth.controller';
import { AuthService } from './auth.service';
import { LocalStrategy } from './strategies/local.strategy';
import { JwtStrategy } from './strategies/jwt.strategy';
import { JwtRefreshStrategy } from './strategies/jwt-refresh.strategy';
import { User } from '../users/entities/user.entity';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [
    // ── TypeORM ─────────────────────────────────────────────────────────
    TypeOrmModule.forFeature([User]),
    // Register User repository for injection in AuthService

    // ── Passport ─────────────────────────────────────────────────────────
    PassportModule.register({
      defaultStrategy: 'jwt',
      // Sets JWT as the default strategy
      // So @UseGuards(AuthGuard()) without a name = @UseGuards(AuthGuard('jwt'))
    }),

    // ── JWT ──────────────────────────────────────────────────────────────
    JwtModule.registerAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        secret: config.getOrThrow('JWT_SECRET'),
        // The ACCESS token secret
        // This is the default secret for jwtService.sign()
        // We override it for refresh tokens

        signOptions: {
          expiresIn: config.get('JWT_EXPIRES_IN', '15m'),
          // Default expiry for jwtService.sign() calls
          // Can be overridden per-call
        },
      }),
    }),

    // ── Other modules ─────────────────────────────────────────────────────
    UsersModule,
    // Import UsersModule so JwtStrategy can inject UsersService
    // (for fetching fresh user data on each request)
  ],

  controllers: [AuthController],

  providers: [
    AuthService,
    LocalStrategy,
    // Registers LocalStrategy as an injectable provider
    // Passport automatically discovers strategies from the DI container
    JwtStrategy,
    JwtRefreshStrategy,
  ],

  exports: [
    AuthService,
    JwtModule,
    // Export JwtModule so other modules can use JwtService
    // (e.g., to verify tokens or generate special tokens)
  ],
})
export class AuthModule {}
```

### Registering JwtAuthGuard globally

```typescript
// src/app.module.ts
// Apply JwtAuthGuard to ALL routes — safer than manually adding to each

import { APP_GUARD } from '@nestjs/core';
import { JwtAuthGuard } from './auth/guards/jwt-auth.guard';

@Module({
  imports: [
    // ... ConfigModule, TypeOrmModule, AuthModule, UsersModule ...
  ],
  providers: [
    {
      provide: APP_GUARD,
      useClass: JwtAuthGuard,
      // Global guard: every route requires JWT authentication
      // Exception: routes decorated with @Public()
    },
  ],
})
export class AppModule {}

// With this setup:
// - Every route: requires valid JWT by default
// - @Public() routes: no JWT required
// - This is the INVERTED DEFAULT — secure by default, opt-out for public routes
// Much safer than opt-in (forgetting @UseGuards() on a sensitive route)
```

---

## 9. Refresh Token System

### JwtRefreshStrategy — a separate strategy for refresh tokens

```typescript
// src/auth/strategies/jwt-refresh.strategy.ts

import { Injectable, UnauthorizedException } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { ConfigService } from '@nestjs/config';
import { Request } from 'express';

@Injectable()
export class JwtRefreshStrategy extends PassportStrategy(Strategy, 'jwt-refresh') {
  // Name 'jwt-refresh' distinguishes from 'jwt' (access token strategy)

  constructor(private readonly configService: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      // Same extraction: Authorization: Bearer <refresh_token>

      secretOrKey: configService.getOrThrow('JWT_REFRESH_SECRET'),
      // DIFFERENT secret from access token!
      // This strategy ONLY verifies refresh tokens

      passReqToCallback: true,
      // Pass the Express Request object to validate()
      // We need the raw token to compare with stored hash
    });
  }

  validate(req: Request, payload: JwtPayload): any {
    // req: Express request (because passReqToCallback: true)
    // payload: decoded JWT payload (signature and expiry already verified)

    // Extract the raw token from Authorization header
    const refreshToken = req
      .get('Authorization')
      ?.replace('Bearer', '')
      .trim();

    if (!refreshToken) {
      throw new UnauthorizedException('Refresh token not provided');
    }

    // Return user info + raw token
    // AuthController.refresh() uses this to verify against DB hash
    return {
      id: parseInt(payload.sub),
      email: payload.email,
      role: payload.role,
      refreshToken,
      // Attach raw token so we can compare with stored hash in AuthService
    };
  }
}
```

### JwtRefreshGuard

```typescript
// src/auth/guards/jwt-refresh.guard.ts

import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtRefreshGuard extends AuthGuard('jwt-refresh') {
  // Uses JwtRefreshStrategy (verified with JWT_REFRESH_SECRET)
  // Applied only to the /auth/refresh endpoint
}
```

### Token refresh flow — visual

```
Client has:
  accessToken (expired or expiring)
  refreshToken (valid, 7 days)

Client sends:
  POST /auth/refresh
  Authorization: Bearer <refreshToken>

Server (JwtRefreshGuard):
  1. Extract token from Authorization header
  2. Verify signature using JWT_REFRESH_SECRET
  3. Check expiry (not expired)
  4. Decode payload → { sub: '1', email: 'alice@test.com' }
  5. JwtRefreshStrategy.validate() → returns { id: 1, refreshToken: '<raw>' }
  6. req.user = { id: 1, refreshToken: '<raw>' }

Server (AuthController.refresh):
  7. authService.refreshTokens(userId=1, refreshToken='<raw>')

Server (AuthService.refreshTokens):
  8. Load user from DB, get stored refreshToken hash
  9. bcrypt.compare(raw, hash) → match!
  10. Generate NEW accessToken + NEW refreshToken
  11. Hash new refreshToken, store in DB (old one now invalid)
  12. Return { accessToken: '<new>', refreshToken: '<new>' }

Client:
  13. Store new tokens
  14. Retry original request with new accessToken
```

### User entity update for refresh tokens

```typescript
// Add to User entity:

@Exclude()
@Column({ nullable: true, select: false })
// select: false: excluded from default queries (security)
// nullable: true: null when logged out
refreshToken: string | null;

@Column({ nullable: true })
lastLoginAt: Date | null;
```

---

## 10. Complete Auth Flow — End to End

### Complete example: protect routes with roles

```typescript
// src/users/users.controller.ts
// Shows how auth + role-based protection work together

import { Controller, Get, Patch, Delete, Param, Body, ParseIntPipe } from '@nestjs/common';
import { JwtAuthGuard } from '../auth/guards/jwt-auth.guard';
import { RolesGuard } from '../auth/guards/roles.guard';
import { Roles } from '../auth/decorators/roles.decorator';
import { CurrentUser } from '../auth/decorators/current-user.decorator';
import { Public } from '../auth/decorators/public.decorator';

@Controller('users')
// JwtAuthGuard is global (registered in AppModule)
// No need to add @UseGuards(JwtAuthGuard) here
export class UsersController {

  // Public endpoint — no auth needed
  @Public()
  @Get('count')
  getCount() {
    return this.usersService.count();
  }

  // Requires auth (JWT) — handled by global JwtAuthGuard
  @Get('me')
  getMyProfile(@CurrentUser() user: User) {
    return user;
  }

  // Requires auth AND admin role
  @Get()
  @UseGuards(RolesGuard)
  @Roles('admin')
  findAll() {
    return this.usersService.findAll();
  }

  // Users can view their own profile OR admins can view any profile
  @Get(':id')
  async findOne(
    @Param('id', ParseIntPipe) id: number,
    @CurrentUser() currentUser: User,
  ) {
    // If user is not admin, they can only see their own profile
    if (currentUser.role !== 'admin' && currentUser.id !== id) {
      throw new ForbiddenException('You can only view your own profile');
    }
    return this.usersService.findOne(id);
  }
}
```

### CurrentUser decorator — clean parameter extraction

```typescript
// src/auth/decorators/current-user.decorator.ts

import { createParamDecorator, ExecutionContext } from '@nestjs/common';
import { User } from '../../users/entities/user.entity';

export const CurrentUser = createParamDecorator(
  (data: keyof User | undefined, ctx: ExecutionContext): User | any => {
    const request = ctx.switchToHttp().getRequest();
    const user: User = request.user;
    // request.user is set by JwtStrategy.validate()

    // If a field name is passed, return just that field
    // @CurrentUser('email') → 'alice@test.com'
    // @CurrentUser('id') → 1
    // @CurrentUser() → full User object
    return data ? user?.[data] : user;
  },
);

// Usage examples:
@Get('profile')
getProfile(@CurrentUser() user: User) { }

@Get('my-role')
getRole(@CurrentUser('role') role: string) { }

@Get('my-id')
getId(@CurrentUser('id') id: number) { }
```

---

## 11. Security Best Practices

```typescript
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// 1. RATE LIMITING — prevent brute force attacks
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

// npm install @nestjs/throttler
import { ThrottlerModule, ThrottlerGuard } from '@nestjs/throttler';
import { APP_GUARD } from '@nestjs/core';

// In AppModule:
ThrottlerModule.forRoot([{
  ttl: 60000,   // 60 second window
  limit: 10,    // Max 10 requests per window per IP
}])

// For login endpoint: stricter limits
@Throttle({ default: { ttl: 300000, limit: 5 } })
// 5 login attempts per 5 minutes
// After 5 failed attempts, client gets 429 Too Many Requests
@Post('login')
login() {}


// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// 2. NEVER EXPOSE WHAT FAILED — prevent user enumeration
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

// ❌ Wrong — reveals if email exists
if (!user) throw new UnauthorizedException('Email not found');
if (!passwordMatch) throw new UnauthorizedException('Wrong password');

// ✅ Correct — same message for both failures
if (!user || !passwordMatch) {
  throw new UnauthorizedException('Invalid email or password');
}


// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// 3. CONSTANT TIME COMPARISON — prevent timing attacks
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

// Timing attack: attacker measures response time to determine if email exists
// If "email not found" returns instantly but "wrong password" takes 300ms (bcrypt)
// → attacker knows which emails are registered

// Fix: ALWAYS run bcrypt.compare(), even if user doesn't exist
if (!user) {
  // Dummy hash — takes same time as real comparison
  await bcrypt.compare(password, '$2b$12$EixZaYVK1fsbw1ZfbX3OXe');
  return null;
}
const isValid = await bcrypt.compare(password, user.password);
// Same response time whether user exists or not ✅


// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// 4. SECURE TOKEN STORAGE — client side
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

// Options for storing tokens on client:
// localStorage: simple but vulnerable to XSS
// sessionStorage: same XSS risk, cleared on tab close
// httpOnly cookies: safe from XSS, vulnerable to CSRF
// Memory (in-memory): most secure, lost on page refresh

// Best practice for SPAs: httpOnly cookies for refresh token + memory for access token
// The refresh token never touches JavaScript → safe from XSS
// Access token in memory → lost on refresh → use refresh token to get new one


// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// 5. REFRESH TOKEN ROTATION — detect token theft
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

// Each time /auth/refresh is called:
// - A NEW refresh token is issued
// - The OLD refresh token is invalidated (overwritten in DB)

// If token is stolen and attacker refreshes:
// - Attacker gets new tokens
// - Real user's refresh token no longer matches DB (overwritten)
// - Real user's next refresh attempt fails → they get logged out
// - They know something is wrong → security alert

// Without rotation: attacker can silently use stolen refresh token forever


// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// 6. JWT SECRET REQUIREMENTS
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

// Minimum 256 bits (32 characters) of randomness
// Generate: node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"

// ❌ Too weak — dictionary word or predictable
JWT_SECRET=mysecret
JWT_SECRET=password123

// ✅ Cryptographically random
JWT_SECRET=a7f3d8e2c1b4f6a9d3e8c2b5f7a1d4e6c8b2f5a9d3e7c1b4f6a8d2e5c7b1f3
```

---

## 12. Interview Questions & Answers

**Q1: What is the difference between authentication and authorization?**

> "Authentication is proving identity — 'who are you?' The user provides credentials (email and password), the server verifies them and issues a token. Authorization is controlling access — 'what are you allowed to do?' After authentication, the system checks if the authenticated user has permission to perform a specific action. In NestJS, Passport strategies handle authentication (verifying tokens and credentials), while Guards handle authorization (checking roles and permissions). The JWT token proves identity, while RBAC guards use the role inside the token to authorize actions."

---

**Q2: Why use two tokens — access token and refresh token — instead of one long-lived token?**

> "It's a security-usability trade-off. A single long-lived token is convenient but risky — if stolen, the attacker has access for days or weeks and there's no way to invalidate it since JWTs are stateless. A single short-lived token is secure but requires the user to log in every 15 minutes — terrible UX. Two tokens solves both: the access token is short-lived (15 minutes), stateless and fast (no database lookup needed). If stolen, it's only valid for 15 minutes. The refresh token is long-lived (7 days), stored in the database, and can be explicitly revoked on logout or suspected breach. The access token is used for every API request; the refresh token is only sent to one endpoint `/auth/refresh` to get a new access token."

---

**Q3: How does Passport.js work with NestJS strategies?**

> "Passport strategies are classes that define HOW to extract and validate credentials. In NestJS, you extend `PassportStrategy` which makes the strategy an injectable provider. The key method is `validate()` — Passport calls it after extracting credentials. For LocalStrategy, Passport extracts email and password from the request body. For JwtStrategy, Passport extracts and verifies the JWT from the Authorization header. If `validate()` returns a value, Passport attaches it to `request.user`. If `validate()` throws, Passport returns 401. NestJS's `AuthGuard` wraps a strategy into a Guard — `AuthGuard('local')` or `AuthGuard('jwt')`. Creating named guard classes like `JwtAuthGuard extends AuthGuard('jwt')` is the NestJS convention for cleaner code."

---

**Q4: Why hash the refresh token before storing it in the database?**

> "If your database is breached, an attacker would have access to all refresh tokens and could impersonate every user. Hashing the refresh token with bcrypt before storing means even with database access, the attacker can't use the tokens — they'd need to brute-force each one individually. This is the same principle as hashing passwords. On token validation, we use `bcrypt.compare(incomingToken, storedHash)` to verify. The cost factor for refresh tokens is lower (10 vs 12 for passwords) because refresh tokens are long random strings that can't be dictionary-attacked — the bcrypt cost mainly protects against brute force if someone gets the hash."

---

**Q5: What is refresh token rotation and why does it help detect token theft?**

> "Token rotation means every time you use a refresh token to get a new access token, the server also issues a NEW refresh token and invalidates the old one. If a refresh token is stolen and the attacker uses it, the original user's next refresh attempt will fail because their refresh token no longer matches the database — the attacker's use replaced it. This alerts the real user that something is wrong. Without rotation, a stolen refresh token can be used silently forever. When we detect reuse of an invalidated refresh token, the most secure response is to clear ALL refresh tokens for that user (force re-login on all devices) as this indicates a potential breach."

---

**Q6: Can you invalidate a JWT access token before it expires?**

> "Not directly — that's both the strength and weakness of JWTs. Since they're stateless and verified only by signature, there's no built-in revocation mechanism. There are three patterns to handle this. First, keep access token lifetimes short (15 minutes) — an attacker's window is limited. Second, implement a token blacklist — store revoked token JTIs (JWT IDs, added with `jwtid: uuid()` in signOptions) in Redis with TTL matching token expiry; check against this blacklist in JwtStrategy. Third, require database lookup on every request in JwtStrategy — check if the user is still active or if a 'tokenVersion' counter matches the one in the token. I use the short lifetime approach for most apps and add Redis blacklisting for high-security scenarios like immediate account suspension."

---

**Q7: What is a timing attack and how does your auth service prevent it?**

> "A timing attack exploits the fact that code takes different amounts of time depending on the data. In authentication: if 'user not found' returns in 0ms but 'wrong password' takes 300ms (bcrypt comparison), an attacker can measure response times to discover which emails are registered. My auth service prevents this by always running `bcrypt.compare()` even when the user doesn't exist — comparing against a dummy hash. This makes 'user not found' and 'wrong password' take the same time (~300ms), making the responses indistinguishable. Additionally, I return the same error message for both cases: 'Invalid email or password' — never revealing which part was wrong."

---

**Q8: What is the difference between `select: false` on the password column and `@Exclude()`?**

> "They solve the same problem at different layers. `@Column({ select: false })` on the TypeORM entity makes TypeORM exclude the `password` field from ALL SELECT queries by default. It's never loaded unless you explicitly call `.addSelect('user.password')`. This is the database layer protection. `@Exclude()` from class-transformer tells the `ClassSerializerInterceptor` to remove the field from JSON serialization before sending the HTTP response. This is the API response layer protection. Using both provides defense in depth: TypeORM won't accidentally load the password, and even if it somehow gets loaded (explicitly for login verification), it won't be sent to the client in the response."

---

## Quick Reference — Day 8 Cheat Sheet

```
Packages:
  npm install @nestjs/passport passport passport-local passport-jwt @nestjs/jwt bcrypt
  npm install @types/passport-local @types/passport-jwt @types/bcrypt --save-dev

Strategy pattern:
  LocalStrategy  → validates email + password → sets req.user
  JwtStrategy    → validates Bearer token  → sets req.user
  JwtRefreshStrategy → validates refresh token → sets req.user + raw token

Guards:
  LocalAuthGuard     extends AuthGuard('local')        → login endpoint
  JwtAuthGuard       extends AuthGuard('jwt')          → protected routes
  JwtRefreshGuard    extends AuthGuard('jwt-refresh')  → refresh endpoint
  Apply globally via APP_GUARD → @Public() for exceptions

JWT:
  Access token:  15m, JWT_SECRET, stateless (no DB lookup needed)
  Refresh token: 7d, JWT_REFRESH_SECRET, stored as HASH in DB
  Payload: { sub: userId, email, role } — never sensitive data
  Header: Authorization: Bearer <token>

bcrypt:
  bcrypt.hash(password, 12)         → creates hash
  bcrypt.compare(plain, hash)       → validates (constant time!)
  saltRounds: 12 for passwords, 10 for refresh tokens
  ALWAYS run compare() even if user not found (timing attack prevention)

Security checklist:
  ✓ Same error message for "email not found" and "wrong password"
  ✓ Always run bcrypt.compare() (timing attack prevention)
  ✓ Hash refresh tokens before storing in DB
  ✓ Refresh token rotation (new token on every refresh)
  ✓ Short access token lifetime (15 min)
  ✓ @Column({ select: false }) on password
  ✓ @Exclude() on password entity field
  ✓ Rate limiting on /auth/login (5 attempts per 5 min)
  ✓ HTTPS in production (tokens in plain text otherwise!)
  ✓ Different secrets for access and refresh tokens

Token lifecycle:
  Register/Login → returns { accessToken (15m), refreshToken (7d) }
  API call → Authorization: Bearer <accessToken>
  Token expired → POST /auth/refresh with refreshToken → new tokens
  Logout → DELETE refreshToken from DB (access token expires naturally)

Key env vars:
  JWT_SECRET=<32+ random chars>
  JWT_EXPIRES_IN=15m
  JWT_REFRESH_SECRET=<different 32+ random chars>
  JWT_REFRESH_EXPIRES_IN=7d
```

---

*Day 8 complete. Tomorrow — Day 9: Authorization & RBAC — Role-based access control, custom @Roles() decorator, RolesGuard, ownership checks, attribute-based access control, and combining multiple guards.*
