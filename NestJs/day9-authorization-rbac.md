# Day 9 — Authorization & RBAC
### NestJS Interview Prep | 3-Year Backend Engineer Level

> **Goal for today:** Master Authorization in NestJS — the difference between authentication and authorization, Role-Based Access Control (RBAC), Attribute-Based Access Control (ABAC), ownership checks, permission systems, and combining multiple guards. A 3-year backend engineer must design authorization systems that are flexible, maintainable, and secure — not just slap `@Roles('admin')` on every route.

---

## 🧭 First-Timer Guidance — Read This First

**Authentication vs Authorization — always start here in interviews:**

```
Authentication (Day 8) → "Who are you?"
  Passport + JWT proves identity → req.user is set

Authorization (Today) → "What are you allowed to do?"
  Guards check req.user's permissions → allow or block
```

**Three levels of authorization you'll implement today:**

```
Level 1: RBAC (Role-Based Access Control)
  "Only ADMINS can delete users"
  Based on: user's ROLE (a single field in DB)
  Simple, coarse-grained — works for most apps

Level 2: Ownership Check
  "Users can only edit THEIR OWN profile"
  Based on: does user.id match resource.userId?
  Fine-grained — resource-level security

Level 3: ABAC (Attribute-Based Access Control)
  "ADMINS can delete, MANAGERS can update, USERS can only read
   AND only within their own department AND only during business hours"
  Based on: multiple attributes of user, resource, environment
  Very granular — enterprise apps, complex permission systems
```

**Real-world authorization rule: Principle of Least Privilege**
> "Give every user, service, and system only the minimum access they need to do their job — nothing more."

**Setup — no new packages needed (using Day 8 setup):**
```bash
# Already installed from Day 8:
# @nestjs/passport passport passport-jwt @nestjs/jwt
# We'll use Reflector from @nestjs/core (already there)
```

---

## Table of Contents

1. [RBAC — Role-Based Access Control Foundation](#1-rbac--role-based-access-control-foundation)
2. [Roles Decorator — Setting Metadata](#2-roles-decorator--setting-metadata)
3. [RolesGuard — Reading and Enforcing Roles](#3-rolesguard--reading-and-enforcing-roles)
4. [Combining Guards — Auth + Roles Together](#4-combining-guards--auth--roles-together)
5. [Ownership Guard — Resource-Level Security](#5-ownership-guard--resource-level-security)
6. [Permission-Based System — Beyond Simple Roles](#6-permission-based-system--beyond-simple-roles)
7. [ABAC — Attribute-Based Access Control](#7-abac--attribute-based-access-control)
8. [Policy Guards — Clean ABAC Implementation](#8-policy-guards--clean-abac-implementation)
9. [Multi-Tenant Authorization](#9-multi-tenant-authorization)
10. [Decorators for Authorization — Clean Controllers](#10-decorators-for-authorization--clean-controllers)
11. [Interview Questions & Answers](#11-interview-questions--answers)

---

## 1. RBAC — Role-Based Access Control Foundation

### What is RBAC?

RBAC assigns users to **roles**, and roles determine what actions users can perform. Instead of assigning permissions to individual users, you assign them to roles and assign roles to users.

```
WITHOUT RBAC (user-level permissions — doesn't scale):
  User Alice → can_create_post, can_edit_post, can_delete_post, can_manage_users
  User Bob   → can_create_post, can_edit_post
  User Carol → can_create_post, can_edit_post, can_delete_post, can_manage_users
  Problem: Managing permissions for 10,000 users is impossible

WITH RBAC (role-level permissions — scales perfectly):
  Role ADMIN      → all permissions
  Role MODERATOR  → can_create_post, can_edit_post, can_delete_post
  Role USER       → can_create_post, can_edit_post
  
  User Alice → ADMIN role
  User Bob   → USER role
  User Carol → ADMIN role
  
  Now: change ADMIN permissions once → affects all admins automatically
```

### Designing your role enum

```typescript
// src/common/enums/role.enum.ts

// Simple flat enum — good for most applications
export enum Role {
  USER = 'user',
  MODERATOR = 'moderator',
  ADMIN = 'admin',
  SUPER_ADMIN = 'super_admin',
}

// The role hierarchy (implied, not enforced by NestJS automatically):
// USER < MODERATOR < ADMIN < SUPER_ADMIN
// Each role includes the permissions of roles below it
// BUT: NestJS doesn't enforce this automatically — you do it in the guard

// For hierarchical roles, define the order:
export const RoleHierarchy: Record<Role, number> = {
  [Role.USER]: 1,
  [Role.MODERATOR]: 2,
  [Role.ADMIN]: 3,
  [Role.SUPER_ADMIN]: 4,
};

// Helper: does roleA have at least the level of roleB?
export function hasMinimumRole(userRole: Role, requiredRole: Role): boolean {
  return RoleHierarchy[userRole] >= RoleHierarchy[requiredRole];
}
// hasMinimumRole(Role.ADMIN, Role.MODERATOR) → true (admin >= moderator)
// hasMinimumRole(Role.USER, Role.ADMIN) → false (user < admin)
```

### User entity with role

```typescript
// src/users/entities/user.entity.ts (relevant part)

import { Role } from '../../common/enums/role.enum';

@Entity('users')
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  email: string;

  @Column({
    type: 'enum',
    enum: Role,
    default: Role.USER,
    // Every new user starts as USER role
    // Only admins/super admins should be able to elevate roles
  })
  role: Role;

  // For more complex permission systems — array of roles
  // @Column('simple-array', { default: '' })
  // roles: Role[];  // User can have multiple roles
}
```

---

## 2. Roles Decorator — Setting Metadata

### The @Roles() decorator

```typescript
// src/auth/decorators/roles.decorator.ts

import { SetMetadata } from '@nestjs/common';
import { Role } from '../../common/enums/role.enum';

// The KEY used to store and retrieve role metadata
// Export it so RolesGuard can read it
export const ROLES_KEY = 'roles';

// @Roles() is a METADATA decorator, not a Guard
// It does NOT enforce anything by itself
// It just MARKS a route with the required roles
// The RolesGuard READS this metadata and enforces it

export const Roles = (...roles: Role[]) => SetMetadata(ROLES_KEY, roles);

// Usage:
// @Roles(Role.ADMIN)              → only admins
// @Roles(Role.ADMIN, Role.MODERATOR) → admins OR moderators
// @Roles(Role.USER)               → any authenticated user with USER role

// How SetMetadata works:
// SetMetadata(key, value) → attaches {key: value} to the route handler
// The Reflector service reads it: reflector.get(key, handler)
// This is NestJS's standard metadata system — same as how @Public() works
```

### Combining with other decorators

```typescript
// src/auth/decorators/admin.decorator.ts

import { applyDecorators, UseGuards } from '@nestjs/common';
import { Roles } from './roles.decorator';
import { RolesGuard } from '../guards/roles.guard';
import { JwtAuthGuard } from '../guards/jwt-auth.guard';
import { Role } from '../../common/enums/role.enum';

// Convenience composite decorator for admin-only routes
// Instead of writing @UseGuards(JwtAuthGuard, RolesGuard) @Roles(Role.ADMIN) every time
export const AdminOnly = () =>
  applyDecorators(
    Roles(Role.ADMIN, Role.SUPER_ADMIN),
    // Both Admin AND Super Admin can access
    UseGuards(RolesGuard),
    // Apply the guard that reads @Roles() metadata
  );

// Convenience for moderator+
export const ModeratorOrAbove = () =>
  applyDecorators(
    Roles(Role.MODERATOR, Role.ADMIN, Role.SUPER_ADMIN),
    UseGuards(RolesGuard),
  );

// Usage in controller:
// @AdminOnly()
// @Delete(':id')
// deleteUser(@Param('id') id: string) {}
```

---

## 3. RolesGuard — Reading and Enforcing Roles

```typescript
// src/auth/guards/roles.guard.ts

import {
  Injectable,
  CanActivate,
  ExecutionContext,
  ForbiddenException,
} from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Role, RoleHierarchy } from '../../common/enums/role.enum';
import { ROLES_KEY } from '../decorators/roles.decorator';
import { User } from '../../users/entities/user.entity';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {
    // Reflector: NestJS utility to read metadata set by decorators
    // It's provided by @nestjs/core — no need to install anything
  }

  canActivate(context: ExecutionContext): boolean {
    // Step 1: Read the @Roles() metadata from the route handler
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(
      ROLES_KEY,
      // The key used in SetMetadata(ROLES_KEY, roles)
      [
        context.getHandler(),
        // Check METHOD-level decorator first (more specific)
        // e.g., @Roles() on the specific method

        context.getClass(),
        // Then check CLASS-level decorator (less specific)
        // e.g., @Roles() on the entire controller

        // getAllAndOverride: returns the FIRST non-undefined value
        // Method decorator overrides class decorator
      ],
    );

    // Step 2: If no @Roles() decorator → allow all authenticated users
    if (!requiredRoles || requiredRoles.length === 0) {
      return true;
      // No role restriction — any authenticated user can access
      // JwtAuthGuard already handled authentication
    }

    // Step 3: Get the authenticated user from request
    // This was set by JwtStrategy.validate() which runs before RolesGuard
    const request = context.switchToHttp().getRequest();
    const user: User = request.user;

    if (!user) {
      // Should never happen if JwtAuthGuard runs first
      // But defensive programming is good practice
      throw new ForbiddenException('User not authenticated');
    }

    // Step 4: Check if user's role is in the required roles list
    const hasRole = requiredRoles.some((role) => user.role === role);
    // .some(): returns true if user has AT LEAST ONE of the required roles
    // @Roles(Role.ADMIN, Role.MODERATOR) → user needs admin OR moderator

    if (!hasRole) {
      throw new ForbiddenException(
        `Access denied. Required role(s): ${requiredRoles.join(', ')}. Your role: ${user.role}`,
        // Clear error message — what role is needed vs what you have
        // In production: consider less verbose to not leak role info
      );
    }

    return true;
  }
}

// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// HIERARCHICAL ROLES GUARD
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

@Injectable()
export class HierarchicalRolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(
      ROLES_KEY,
      [context.getHandler(), context.getClass()],
    );

    if (!requiredRoles || requiredRoles.length === 0) return true;

    const { user }: { user: User } = context.switchToHttp().getRequest();

    // Find the MINIMUM required role level
    const minRequiredLevel = Math.min(
      ...requiredRoles.map((role) => RoleHierarchy[role]),
    );

    // Check if user's role level is >= the minimum required level
    const userLevel = RoleHierarchy[user.role] ?? 0;
    const hasAccess = userLevel >= minRequiredLevel;

    if (!hasAccess) {
      throw new ForbiddenException(
        `Insufficient permissions. Your role level: ${userLevel}, required: ${minRequiredLevel}`,
      );
    }

    return true;
  }
}

// With HierarchicalRolesGuard:
// @Roles(Role.MODERATOR) → MODERATOR, ADMIN, SUPER_ADMIN can all access
// Because their level >= MODERATOR's level
// Without: ADMIN cannot access MODERATOR routes unless explicitly listed
```

---

## 4. Combining Guards — Auth + Roles Together

### How multiple guards work

```typescript
// When you use multiple guards, they run in ORDER
// ALL guards must pass for the request to proceed
// First failure → request blocked, subsequent guards don't run

@Controller('admin')
@UseGuards(JwtAuthGuard, RolesGuard)
// Order matters:
// 1. JwtAuthGuard runs first → verifies token → sets req.user
// 2. RolesGuard runs second → reads req.user.role → checks @Roles()
// If JwtAuthGuard fails → 401 Unauthorized (RolesGuard never runs)
// If JwtAuthGuard passes but RolesGuard fails → 403 Forbidden

@Roles(Role.ADMIN)
// @Roles() on the CLASS → applies to ALL methods in this controller
export class AdminController {

  @Get('users')
  // Inherits: @UseGuards(JwtAuthGuard, RolesGuard) + @Roles(Role.ADMIN)
  getAllUsers() {}

  @Get('stats')
  @Roles(Role.ADMIN, Role.SUPER_ADMIN)
  // METHOD-level @Roles() OVERRIDES class-level @Roles()
  // (getAllAndOverride takes the method-level value first)
  getStats() {}

  @Delete('user/:id')
  @Roles(Role.SUPER_ADMIN)
  // Only super admin can delete — more restrictive than class default
  deleteUser(@Param('id') id: string) {}
}
```

### Global guards with per-route overrides

```typescript
// src/app.module.ts

// Recommended architecture:
// 1. JwtAuthGuard → global (every route requires auth by default)
// 2. RolesGuard → global (every route checks @Roles() if present)
// 3. @Public() → opt-out for public routes
// 4. @Roles() → opt-in for role-restricted routes

@Module({
  providers: [
    // Guard 1: Authentication (runs first)
    {
      provide: APP_GUARD,
      useClass: JwtAuthGuard,
      // Checks JWT on every request
      // @Public() skips this check
    },
    // Guard 2: Authorization (runs second)
    {
      provide: APP_GUARD,
      useClass: RolesGuard,
      // Checks @Roles() on every request
      // If no @Roles() decorator → allows through (any authenticated user)
    },
  ],
})
export class AppModule {}

// Result in controllers:
@Controller('users')
export class UsersController {

  @Public()              // Skip ALL guards
  @Get('count')
  count() {}             // No auth, no role check

  @Get('profile')
  getProfile() {}        // Auth required, no role restriction

  @Roles(Role.ADMIN)
  @Get('all')
  findAll() {}           // Auth required + must be admin

  @Roles(Role.ADMIN, Role.MODERATOR)
  @Patch(':id/ban')
  banUser() {}           // Auth required + must be admin OR moderator
}
```

---

## 5. Ownership Guard — Resource-Level Security

### The problem ownership solves

```
RBAC alone is not enough:
  "Users can edit posts" → a user should only edit THEIR posts
  Without ownership check: user Alice can edit user Bob's posts!

  @Roles(Role.USER) on PATCH /posts/:id
  → Any USER can edit ANY post
  → Alice can edit Bob's posts → SECURITY BUG

Ownership check adds:
  "Users can edit posts — but only their own posts"
  → Check: post.authorId === req.user.id
```

### Generic Ownership Guard

```typescript
// src/auth/guards/ownership.guard.ts

import {
  Injectable,
  CanActivate,
  ExecutionContext,
  ForbiddenException,
  NotFoundException,
} from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Role } from '../../common/enums/role.enum';

// We need to know WHICH entity to check ownership for
// Use a decorator to specify this
export const OWNERSHIP_KEY = 'ownership';

export interface OwnershipOptions {
  entity: any;           // The TypeORM entity class
  ownerField: string;    // The field on the entity that holds the owner's ID
  paramField?: string;   // The route param that holds the resource ID (default: 'id')
  adminBypass?: boolean; // Can admins bypass ownership check? (default: true)
}

export const CheckOwnership = (options: OwnershipOptions) =>
  SetMetadata(OWNERSHIP_KEY, options);

// The guard itself
@Injectable()
export class OwnershipGuard implements CanActivate {
  // We can't @InjectRepository here directly since we don't know the entity
  // We'll use DataSource to get any repository dynamically

  constructor(
    private readonly reflector: Reflector,
    private readonly dataSource: DataSource,
    // DataSource from TypeORM — allows getting any repository
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    // Read ownership options from @CheckOwnership() decorator
    const options = this.reflector.getAllAndOverride<OwnershipOptions>(
      OWNERSHIP_KEY,
      [context.getHandler(), context.getClass()],
    );

    // No @CheckOwnership() decorator → skip this guard
    if (!options) return true;

    const request = context.switchToHttp().getRequest();
    const user = request.user;
    const { entity, ownerField, paramField = 'id', adminBypass = true } = options;

    // Admin bypass: admins can access any resource regardless of ownership
    if (adminBypass && (user.role === Role.ADMIN || user.role === Role.SUPER_ADMIN)) {
      return true;
      // Admins skip ownership check — they can manage all resources
    }

    // Get the resource ID from route parameters
    const resourceId = request.params[paramField];
    if (!resourceId) {
      throw new ForbiddenException('Resource ID not found in route parameters');
    }

    // Dynamically get the repository for the entity
    const repository = this.dataSource.getRepository(entity);

    // Load the resource from database
    const resource = await repository.findOne({
      where: { id: parseInt(resourceId) },
      select: [ownerField, 'id'],
      // Only load the ownership field — faster query
    });

    if (!resource) {
      throw new NotFoundException(`Resource not found`);
    }

    // Check ownership: does the resource belong to the current user?
    if (resource[ownerField] !== user.id) {
      throw new ForbiddenException(
        'You do not have permission to access this resource',
      );
    }

    // Optionally attach the resource to the request
    // So the controller doesn't need to fetch it again
    request.resource = resource;

    return true;
  }
}
```

### Using OwnershipGuard in controllers

```typescript
// src/posts/posts.controller.ts

import { CheckOwnership, OwnershipGuard } from '../auth/guards/ownership.guard';
import { Post as PostEntity } from './entities/post.entity';

@Controller('posts')
export class PostsController {

  // Anyone can read posts
  @Public()
  @Get(':id')
  findOne(@Param('id', ParseIntPipe) id: number) {
    return this.postsService.findOne(id);
  }

  // Anyone authenticated can create posts
  @Post()
  create(@Body() dto: CreatePostDto, @CurrentUser() user: User) {
    return this.postsService.create(dto, user.id);
  }

  // Only the POST OWNER (or admin) can update
  @Patch(':id')
  @UseGuards(OwnershipGuard)
  @CheckOwnership({
    entity: PostEntity,     // Which entity to check
    ownerField: 'authorId', // Which field holds the owner's userId
    paramField: 'id',       // Route param :id is the post ID
    adminBypass: true,      // Admins can edit any post
  })
  update(
    @Param('id', ParseIntPipe) id: number,
    @Body() dto: UpdatePostDto,
  ) {
    return this.postsService.update(id, dto);
  }

  // Only the POST OWNER (or admin) can delete
  @Delete(':id')
  @UseGuards(OwnershipGuard)
  @CheckOwnership({
    entity: PostEntity,
    ownerField: 'authorId',
    adminBypass: true,
  })
  @HttpCode(HttpStatus.NO_CONTENT)
  remove(@Param('id', ParseIntPipe) id: number) {
    return this.postsService.remove(id);
  }
}
```

### Simpler Inline Ownership Check

```typescript
// For simple cases, ownership check can be in the service directly
// This is often cleaner than a complex generic guard

// src/posts/posts.service.ts
@Injectable()
export class PostsService {
  async update(
    postId: number,
    updateDto: UpdatePostDto,
    requestingUserId: number,
    requestingUserRole: Role,
  ): Promise<Post> {
    const post = await this.postRepository.findOne({
      where: { id: postId },
    });

    if (!post) throw new NotFoundException(`Post #${postId} not found`);

    // Ownership check inline:
    const isOwner = post.authorId === requestingUserId;
    const isAdmin = [Role.ADMIN, Role.SUPER_ADMIN].includes(requestingUserRole);

    if (!isOwner && !isAdmin) {
      throw new ForbiddenException(
        'You can only edit your own posts',
      );
    }

    Object.assign(post, updateDto);
    return this.postRepository.save(post);
  }
}

// In controller — pass user info to service:
@Patch(':id')
update(
  @Param('id', ParseIntPipe) id: number,
  @Body() dto: UpdatePostDto,
  @CurrentUser() user: User,
) {
  return this.postsService.update(id, dto, user.id, user.role);
}
```

---

## 6. Permission-Based System — Beyond Simple Roles

### When simple roles aren't enough

```
Simple roles problem:
  Role MODERATOR can: ban users, delete posts
  Role MANAGER can: create users, view reports
  
  You need a user who can BOTH ban users AND create users
  But is neither MODERATOR nor MANAGER...
  
  Solution: Permission-based system
  User has a SET OF PERMISSIONS, not just one role
```

### Permission enum and entity

```typescript
// src/common/enums/permission.enum.ts

export enum Permission {
  // User permissions
  USER_READ = 'user:read',
  USER_CREATE = 'user:create',
  USER_UPDATE = 'user:update',
  USER_DELETE = 'user:delete',
  USER_BAN = 'user:ban',

  // Post permissions
  POST_READ = 'post:read',
  POST_CREATE = 'post:create',
  POST_UPDATE = 'post:update',
  POST_DELETE = 'post:delete',
  POST_PUBLISH = 'post:publish',

  // Product permissions
  PRODUCT_READ = 'product:read',
  PRODUCT_CREATE = 'product:create',
  PRODUCT_UPDATE = 'product:update',
  PRODUCT_DELETE = 'product:delete',

  // Report permissions
  REPORT_VIEW = 'report:view',
  REPORT_EXPORT = 'report:export',

  // System permissions
  SYSTEM_CONFIG = 'system:config',
  SYSTEM_LOGS = 'system:logs',
}

// Permission groups — map roles to permission sets
export const RolePermissions: Record<string, Permission[]> = {
  user: [
    Permission.POST_READ,
    Permission.POST_CREATE,
    Permission.POST_UPDATE,  // Their own posts only
    Permission.PRODUCT_READ,
    Permission.USER_READ,    // Their own profile only
  ],
  moderator: [
    ...RolePermissions?.user ?? [],  // All user permissions
    Permission.POST_DELETE,
    Permission.POST_PUBLISH,
    Permission.USER_BAN,
    Permission.REPORT_VIEW,
  ],
  admin: [
    ...Object.values(Permission).filter(
      p => !p.startsWith('system:')
    ),  // All permissions EXCEPT system
  ],
  super_admin: [
    ...Object.values(Permission),  // ALL permissions
  ],
};
```

### Permission decorator and guard

```typescript
// src/auth/decorators/permissions.decorator.ts

import { SetMetadata } from '@nestjs/common';
import { Permission } from '../../common/enums/permission.enum';

export const PERMISSIONS_KEY = 'permissions';

// @RequirePermissions() marks a route with required permissions
// User must have ALL listed permissions (AND logic)
export const RequirePermissions = (...permissions: Permission[]) =>
  SetMetadata(PERMISSIONS_KEY, permissions);

// For OR logic (any of these permissions):
export const PERMISSIONS_ANY_KEY = 'permissions_any';
export const RequireAnyPermission = (...permissions: Permission[]) =>
  SetMetadata(PERMISSIONS_ANY_KEY, permissions);


// src/auth/guards/permissions.guard.ts

@Injectable()
export class PermissionsGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    // Read required permissions from route metadata
    const requiredPermissions = this.reflector.getAllAndOverride<Permission[]>(
      PERMISSIONS_KEY,
      [context.getHandler(), context.getClass()],
    );

    const anyPermissions = this.reflector.getAllAndOverride<Permission[]>(
      PERMISSIONS_ANY_KEY,
      [context.getHandler(), context.getClass()],
    );

    // No permission restriction
    if (!requiredPermissions?.length && !anyPermissions?.length) return true;

    const { user } = context.switchToHttp().getRequest();
    // user.permissions: string[] of Permission values
    // Populated by JwtStrategy from DB or JWT payload

    // Check ALL required permissions (AND logic)
    if (requiredPermissions?.length) {
      const hasAll = requiredPermissions.every(p =>
        user.permissions?.includes(p),
      );
      if (!hasAll) {
        throw new ForbiddenException(
          `Missing required permissions: ${requiredPermissions.join(', ')}`,
        );
      }
    }

    // Check ANY permission (OR logic)
    if (anyPermissions?.length) {
      const hasAny = anyPermissions.some(p =>
        user.permissions?.includes(p),
      );
      if (!hasAny) {
        throw new ForbiddenException(
          `Need at least one of: ${anyPermissions.join(', ')}`,
        );
      }
    }

    return true;
  }
}

// Usage:
@RequirePermissions(Permission.USER_READ, Permission.REPORT_VIEW)
// User must have BOTH user:read AND report:view
@Get('user-report')
getUserReport() {}

@RequireAnyPermission(Permission.POST_DELETE, Permission.POST_PUBLISH)
// User must have EITHER post:delete OR post:publish
@Delete('post/:id')
deletePost() {}
```

### Loading permissions into JWT / request

```typescript
// src/auth/strategies/jwt.strategy.ts — include permissions

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy, 'jwt') {
  async validate(payload: JwtPayload) {
    const user = await this.usersService.findOne(parseInt(payload.sub));
    if (!user || !user.isActive) throw new UnauthorizedException();

    // Get permissions for user's role
    const permissions = RolePermissions[user.role] || [];

    // Also include any user-specific extra permissions
    const extraPermissions = user.extraPermissions || [];

    return {
      ...user,
      permissions: [...new Set([...permissions, ...extraPermissions])],
      // Deduplicate permissions using Set
    };
    // req.user now has both role AND permissions array
  }
}
```

---

## 7. ABAC — Attribute-Based Access Control

### What is ABAC?

ABAC makes authorization decisions based on **attributes** — properties of the user, the resource, the action, and the environment.

```
RBAC rule: "Admins can delete posts"
  Decision factors: user.role = admin

ABAC rule: "Managers can update employee records 
            for employees in their department
            during business hours
            if the record's sensitivity level <= manager's clearance level"
  Decision factors:
    - user.role = manager
    - user.department = employee.department
    - current_time = business hours (9am-6pm Mon-Fri)
    - employee.sensitivityLevel <= user.clearanceLevel

ABAC is more complex but enables very fine-grained control
```

### ABAC with Policies

```typescript
// src/auth/policies/post.policy.ts
// A policy defines WHO can do WHAT with a specific resource

import { Injectable } from '@nestjs/common';
import { User } from '../../users/entities/user.entity';
import { Post } from '../../posts/entities/post.entity';
import { Role } from '../../common/enums/role.enum';

// Action enum for posts
export enum PostAction {
  READ = 'read',
  CREATE = 'create',
  UPDATE = 'update',
  DELETE = 'delete',
  PUBLISH = 'publish',
}

@Injectable()
export class PostPolicy {
  // can(): returns true if user can perform action on post
  can(user: User, action: PostAction, post?: Post): boolean {
    switch (action) {
      case PostAction.READ:
        // Anyone can read published posts
        // Authors can read their own unpublished posts
        // Admins can read any post
        if (!post) return true;
        return (
          post.isPublished ||
          post.authorId === user.id ||
          this.isAdmin(user)
        );

      case PostAction.CREATE:
        // Any authenticated user can create posts
        return true;

      case PostAction.UPDATE:
        if (!post) return false;
        // Owner can update their own post (any status)
        // Moderators can update any post
        // Admins can update any post
        return (
          post.authorId === user.id ||
          this.isModerator(user) ||
          this.isAdmin(user)
        );

      case PostAction.DELETE:
        if (!post) return false;
        // Owner can delete their own DRAFT posts
        // Moderators can delete any post
        // Admins can delete any post
        return (
          (post.authorId === user.id && !post.isPublished) ||
          this.isModerator(user) ||
          this.isAdmin(user)
        );

      case PostAction.PUBLISH:
        if (!post) return false;
        // Only moderators and admins can publish
        // (not the author themselves — requires review)
        return this.isModerator(user) || this.isAdmin(user);

      default:
        return false;
    }
  }

  private isAdmin(user: User): boolean {
    return [Role.ADMIN, Role.SUPER_ADMIN].includes(user.role as Role);
  }

  private isModerator(user: User): boolean {
    return [Role.MODERATOR, Role.ADMIN, Role.SUPER_ADMIN].includes(
      user.role as Role,
    );
  }
}
```

---

## 8. Policy Guards — Clean ABAC Implementation

### The Casl library (industry standard for Node.js ABAC)

```bash
npm install @casl/ability @casl/mongoose  # or @casl/prisma
```

```typescript
// src/auth/casl/casl-ability.factory.ts

import { Injectable } from '@nestjs/common';
import {
  AbilityBuilder,
  createMongoAbility,
  ExtractSubjectType,
  InferSubjects,
  MongoAbility,
  AbilityClass,
} from '@casl/ability';
import { User } from '../../users/entities/user.entity';
import { Post } from '../../posts/entities/post.entity';
import { Role } from '../../common/enums/role.enum';

// Define subjects (resources) that abilities apply to
type Subjects = InferSubjects<typeof User | typeof Post> | 'all';
// 'all' means the ability applies to ALL subjects

// Define actions
type Actions = 'manage' | 'create' | 'read' | 'update' | 'delete' | 'publish';
// 'manage' = all actions (CASL special keyword)

// The Ability type — what a user CAN do
export type AppAbility = MongoAbility<[Actions, Subjects]>;

@Injectable()
export class CaslAbilityFactory {
  // Create abilities for a specific user
  createForUser(user: User): AppAbility {
    const { can, cannot, build } = new AbilityBuilder<AppAbility>(
      createMongoAbility,
    );
    // AbilityBuilder: fluent API to define what a user CAN and CANNOT do
    // can(action, subject): user CAN perform action on subject
    // cannot(action, subject): user CANNOT (overrides can)

    if (user.role === Role.SUPER_ADMIN) {
      can('manage', 'all');
      // Super admin can do EVERYTHING
      // 'manage' = all actions, 'all' = all subjects

    } else if (user.role === Role.ADMIN) {
      can('manage', 'all');
      cannot('delete', User, { role: Role.SUPER_ADMIN });
      // Admins can do everything EXCEPT delete super admins
      // cannot() overrides a previous can()

    } else if (user.role === Role.MODERATOR) {
      can('read', 'all');
      // Can read everything

      can(['update', 'delete'], Post);
      // Can update and delete any post

      can('publish', Post);
      // Can publish posts

      can(['read', 'update'], User, { id: user.id });
      // Can only update their own user profile

    } else {
      // Regular USER
      can('read', Post, { isPublished: true });
      // Can only read PUBLISHED posts
      // { isPublished: true } = condition — must match for the rule to apply

      can('create', Post);
      // Can create posts

      can(['update', 'delete'], Post, { authorId: user.id });
      // Can update/delete posts WHERE authorId === user.id
      // CASL automatically enforces the condition

      can(['read', 'update'], User, { id: user.id });
      // Can only read/update their own user profile

      cannot('update', User, ['role', 'isActive']);
      // CANNOT update role or isActive fields on any User
      // Field-level restriction!
    }

    return build({
      detectSubjectType: (item) =>
        item.constructor as ExtractSubjectType<Subjects>,
      // Tells CASL how to determine the subject type from an object
    });
  }
}
```

### PoliciesGuard — using CASL abilities

```typescript
// src/auth/guards/policies.guard.ts

import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { CaslAbilityFactory, AppAbility } from '../casl/casl-ability.factory';

// Interface for policy handlers
export interface IPolicyHandler {
  handle(ability: AppAbility): boolean;
}

// Policy handler as a function type
type PolicyHandlerCallback = (ability: AppAbility) => boolean;
export type PolicyHandler = IPolicyHandler | PolicyHandlerCallback;

export const CHECK_POLICIES_KEY = 'check_policies';
export const CheckPolicies = (...handlers: PolicyHandler[]) =>
  SetMetadata(CHECK_POLICIES_KEY, handlers);

@Injectable()
export class PoliciesGuard implements CanActivate {
  constructor(
    private reflector: Reflector,
    private caslAbilityFactory: CaslAbilityFactory,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const policyHandlers = this.reflector.get<PolicyHandler[]>(
      CHECK_POLICIES_KEY,
      context.getHandler(),
    );

    // No @CheckPolicies() → allow
    if (!policyHandlers?.length) return true;

    const { user } = context.switchToHttp().getRequest();

    // Create abilities for this specific user
    const ability = this.caslAbilityFactory.createForUser(user);
    // ability contains all the CAN/CANNOT rules for this user

    // Check ALL policy handlers — ALL must pass
    return policyHandlers.every((handler) =>
      this.execPolicyHandler(handler, ability),
    );
  }

  private execPolicyHandler(handler: PolicyHandler, ability: AppAbility): boolean {
    if (typeof handler === 'function') {
      return handler(ability);
      // Function handler: (ability) => ability.can('read', Post)
    }
    return handler.handle(ability);
    // Class handler: policyHandler.handle(ability)
  }
}

// ── Policy handler examples ────────────────────────────────────────────────

// Inline function handler
export const readPost = (ability: AppAbility) => ability.can('read', Post);
export const createPost = (ability: AppAbility) => ability.can('create', Post);
export const deletePost = (ability: AppAbility) => ability.can('delete', Post);

// Class-based handler (more reusable, can inject services)
export class ReadPostPolicyHandler implements IPolicyHandler {
  handle(ability: AppAbility): boolean {
    return ability.can('read', Post);
  }
}

// Usage in controller:
@Controller('posts')
@UseGuards(JwtAuthGuard, PoliciesGuard)
export class PostsController {

  @Get()
  @CheckPolicies(readPost)
  // Or: @CheckPolicies((ability) => ability.can('read', Post))
  findAll() {}

  @Post()
  @CheckPolicies(createPost)
  create(@Body() dto: CreatePostDto) {}

  @Delete(':id')
  @CheckPolicies(deletePost)
  remove(@Param('id') id: string) {}

  // Checking with conditions (on specific resource):
  @Patch(':id')
  @CheckPolicies((ability: AppAbility) => {
    // This checks the ACTION type only (not against a specific instance)
    // For instance-level checks, do it in the service
    return ability.can('update', Post);
  })
  update(@Param('id') id: string, @Body() dto: UpdatePostDto) {}
}
```

---

## 9. Multi-Tenant Authorization

### What is multi-tenancy?

Multi-tenancy = one application serving multiple organizations (tenants). Users from Company A should NEVER see Company B's data.

```
Tenant A (Company A):
  User Alice (admin of A) → can manage Company A's data only
  User Bob (user of A) → can read Company A's data only

Tenant B (Company B):
  User Carol (admin of B) → can manage Company B's data only
  
Alice CANNOT access Company B's data, even if she's an admin
```

### Tenant guard implementation

```typescript
// src/auth/guards/tenant.guard.ts

import { Injectable, CanActivate, ExecutionContext, ForbiddenException } from '@nestjs/common';

@Injectable()
export class TenantGuard implements CanActivate {
  constructor(
    @InjectRepository(Resource)
    private resourceRepository: Repository<Resource>,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    const resourceId = request.params.id;

    if (!resourceId) return true;
    // No resource ID in route → skip tenant check (list endpoints handled differently)

    // Load the resource
    const resource = await this.resourceRepository.findOne({
      where: { id: parseInt(resourceId) },
      select: ['id', 'tenantId'],
    });

    if (!resource) return true;
    // Resource not found → let the controller handle 404

    // Critical check: does the resource belong to the user's tenant?
    if (resource.tenantId !== user.tenantId) {
      throw new ForbiddenException(
        'You do not have access to resources in this organization',
      );
      // NEVER say "this resource belongs to another tenant"
      // That reveals information about other tenants' data
      // Treat it the same as "resource not found" in some designs
    }

    return true;
  }
}

// Add tenantId to User entity:
// @Column()
// tenantId: number;

// Add tenantId to all resource entities:
// @Column()
// tenantId: number;
// @ManyToOne(() => Tenant)
// tenant: Tenant;

// Automatically scope all queries by tenantId:
// In service:
async findAll(tenantId: number) {
  return this.resourceRepository.find({
    where: { tenantId },
    // Every query filtered by tenant — tenant isolation at query level
  });
}

// Even better: TypeORM global filter (always adds tenantId to WHERE)
// Or Prisma middleware that always adds tenantId
```

---

## 10. Decorators for Authorization — Clean Controllers

### Building a clean authorization decorator API

```typescript
// src/auth/decorators/index.ts
// One file to rule all authorization decorators

import { applyDecorators, UseGuards } from '@nestjs/common';
import { Role } from '../../common/enums/role.enum';
import { Permission } from '../../common/enums/permission.enum';

// ── @Authenticated() — requires valid JWT ──────────────────────────────
export const Authenticated = () =>
  applyDecorators(
    UseGuards(JwtAuthGuard),
    // Just authentication, no role/permission required
  );

// ── @AdminRoute() — requires admin role ──────────────────────────────────
export const AdminRoute = () =>
  applyDecorators(
    Roles(Role.ADMIN, Role.SUPER_ADMIN),
    UseGuards(JwtAuthGuard, RolesGuard),
  );

// ── @ModeratorRoute() — requires moderator role or above ─────────────────
export const ModeratorRoute = () =>
  applyDecorators(
    Roles(Role.MODERATOR, Role.ADMIN, Role.SUPER_ADMIN),
    UseGuards(JwtAuthGuard, RolesGuard),
  );

// ── @ApiKey() — alternative auth for server-to-server ────────────────────
export const ApiKey = () =>
  applyDecorators(
    UseGuards(ApiKeyGuard),
    // Uses a different guard that checks X-API-Key header
  );

// Controllers become beautifully clean:
@Controller('admin/users')
export class AdminUsersController {

  @AdminRoute()
  @Get()
  findAll() {}

  @AdminRoute()
  @Post()
  create(@Body() dto: CreateUserDto) {}

  @ModeratorRoute()
  @Patch(':id/ban')
  banUser(@Param('id') id: string) {}

  @AdminRoute()
  @Delete(':id')
  remove(@Param('id') id: string) {}
}
```

### Complete real-world controller with authorization

```typescript
// src/posts/posts.controller.ts — full example

@Controller('posts')
export class PostsController {
  constructor(private readonly postsService: PostsService) {}

  // ── Public: no auth needed ─────────────────────────────────────────────
  @Public()
  @Get()
  findAll(@Query() query: QueryPostDto) {
    return this.postsService.findPublished(query);
  }

  @Public()
  @Get(':id')
  findOne(@Param('id', ParseIntPipe) id: number) {
    return this.postsService.findOnePublished(id);
  }

  // ── Authenticated users ────────────────────────────────────────────────
  @Post()
  create(
    @Body() createPostDto: CreatePostDto,
    @CurrentUser() user: User,
  ) {
    return this.postsService.create(createPostDto, user.id);
  }

  @Get('my/posts')
  getMyPosts(
    @CurrentUser() user: User,
    @Query() query: QueryPostDto,
  ) {
    return this.postsService.findByAuthor(user.id, query);
  }

  // ── Owner or admin: edit own post ──────────────────────────────────────
  @Patch(':id')
  update(
    @Param('id', ParseIntPipe) id: number,
    @Body() updatePostDto: UpdatePostDto,
    @CurrentUser() user: User,
  ) {
    // Ownership + admin check in service
    return this.postsService.update(id, updatePostDto, user);
  }

  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  remove(
    @Param('id', ParseIntPipe) id: number,
    @CurrentUser() user: User,
  ) {
    return this.postsService.remove(id, user);
  }

  // ── Moderator only: publish post ───────────────────────────────────────
  @ModeratorRoute()
  @Patch(':id/publish')
  publish(@Param('id', ParseIntPipe) id: number) {
    return this.postsService.publish(id);
  }

  // ── Admin only: view all posts including unpublished ───────────────────
  @AdminRoute()
  @Get('admin/all')
  findAllAdmin(@Query() query: QueryPostDto) {
    return this.postsService.findAll(query);
  }
}
```

---

## 11. Interview Questions & Answers

**Q1: What is the difference between RBAC and ABAC?**

> "RBAC (Role-Based Access Control) grants permissions based on a user's ROLE — a discrete label like 'admin', 'moderator', or 'user'. It's simple, fast to check, and works well for most applications. The limitation is that roles are coarse-grained — you can't say 'managers can update employees in their own department only'. ABAC (Attribute-Based Access Control) makes decisions based on multiple ATTRIBUTES: user attributes (role, department, clearance level), resource attributes (owner, sensitivity, status), action (read/write/delete), and environment (time of day, IP address). ABAC is more expressive and flexible but more complex to implement and reason about. In practice: start with RBAC for most features, add ownership checks for resource-level control, and reach for ABAC only when role + ownership checks aren't expressive enough."

---

**Q2: How does NestJS's Reflector work in Guards?**

> "The Reflector is a NestJS service that reads metadata attached to controllers and route handlers by decorators. When you call `SetMetadata(key, value)` via a decorator like `@Roles(Role.ADMIN)`, it stores the value against the key on the handler function using `Reflect.defineMetadata`. In a Guard, you inject Reflector and call `reflector.getAllAndOverride(key, [handler, class])` — this reads the metadata from the METHOD first, then the CLASS, returning whichever is defined. `getAllAndOverride` gives priority to method-level over class-level. There's also `getAllAndMerge` which combines both (useful if you want to accumulate roles from both levels) and `get` which only reads from one target."

---

**Q3: What is the order of execution when multiple guards are applied?**

> "Guards execute in ORDER — left to right when specified with `@UseGuards(Guard1, Guard2)`, or in registration order for `APP_GUARD` providers. All guards must return true for the request to proceed — if any guard returns false or throws, subsequent guards are skipped and the request is rejected. For global guards registered via `APP_GUARD`, they run in the order they're defined in the providers array. The typical order for a protected route is: JwtAuthGuard first (sets req.user), then RolesGuard (reads req.user.role), then custom guards like OwnershipGuard. This order matters because RolesGuard needs req.user which JwtAuthGuard sets."

---

**Q4: How would you implement ownership checks at scale?**

> "For a handful of routes, inline service-level checks are cleanest — the service method receives the requesting user and compares user.id against resource.ownerId, throwing ForbiddenException if no match. For many routes, a generic OwnershipGuard with a `@CheckOwnership()` decorator keeps it DRY — the decorator specifies the entity, owner field, and route parameter, and the guard loads the resource and does the comparison. For very complex ownership rules (like 'team members can access resources owned by anyone on their team'), I'd use CASL where the ability factory encodes 'user can update Post where post.teamId === user.teamId' as a condition-based rule. CASL is particularly powerful because the same rules work for both API authorization AND frontend UI (showing/hiding buttons)."

---

**Q5: What is refresh token rotation and how does it relate to authorization?**

> "Refresh token rotation ensures each refresh token is single-use — using a refresh token issues a new one and invalidates the old. While this is primarily an authentication mechanism (Day 8), it has authorization implications: when a user's role changes (e.g., promoted to admin), the change is reflected immediately in the database. But their existing JWT payload still shows the old role until the token expires. This is the stateless JWT trade-off. Solutions: keep access tokens short-lived (15 minutes) so role changes take effect quickly, store a 'tokenVersion' counter in the database and include it in JWT — if the counter in DB doesn't match the token, reject it. Refresh token rotation also helps here: each refresh generates a new access token with the latest role from the database."

---

**Q6: How do you prevent a regular user from escalating their own role?**

> "Multiple layers of defense. First, the `UpdateUserDto` should not include a `role` field — or use `OmitType(UserDto, ['role'])`. Even if a client sends a `role` field, ValidationPipe's `whitelist: true` strips it before it reaches the controller. Second, in the service's update method, explicitly exclude the role field from the update: `const { role: _, ...safeUpdate } = updateUserDto`. Third, role changes should be a separate, admin-only endpoint: `PATCH /admin/users/:id/role` protected by `@AdminRoute()`. Finally, any endpoint that does allow role updates should verify the requesting user is an admin and validate the new role is within the hierarchy they're allowed to set — a regular admin shouldn't be able to create a SUPER_ADMIN."

---

**Q7: How does CASL handle field-level authorization?**

> "CASL supports restricting access to specific FIELDS on a resource, not just the resource itself. You define it with `cannot('update', User, ['role', 'isActive'])` — this means the user can update User records but NOT the role or isActive fields. In the service, you check with `ability.can('update', user, 'role')` before applying the role update, or use `permittedFieldsOf(ability, 'update', User)` to get the list of fields the user IS allowed to update. This is powerful for endpoints where different users can update the same resource but with different field restrictions — a user can update their own name and email, but not their role or account status. It's cleaner than having separate endpoints for each case."

---

## Quick Reference — Day 9 Cheat Sheet

```
Authorization levels:
  RBAC       → role-based: user.role === 'admin'
  Ownership  → resource-based: resource.ownerId === user.id
  ABAC/CASL  → attribute-based: complex conditions and field-level rules

Key decorators:
  @Roles(Role.ADMIN)         → metadata for RolesGuard
  @CheckOwnership({ entity, ownerField })  → for OwnershipGuard
  @CheckPolicies(handler)    → for PoliciesGuard (CASL)
  @Public()                  → bypass JwtAuthGuard
  @RequirePermissions(perm)  → for PermissionsGuard

RolesGuard reads:
  reflector.getAllAndOverride(ROLES_KEY, [handler, class])
  Method-level @Roles() overrides class-level @Roles()

Guard execution order matters:
  1. JwtAuthGuard  → sets req.user (must be first)
  2. RolesGuard    → reads req.user.role
  3. OwnershipGuard → loads resource, checks req.user.id

Registering globally in AppModule:
  { provide: APP_GUARD, useClass: JwtAuthGuard }
  { provide: APP_GUARD, useClass: RolesGuard }
  // Order in providers array = execution order

Composite decorators (clean code):
  @AdminRoute() = @Roles(ADMIN, SUPER_ADMIN) + @UseGuards(RolesGuard)
  @ModeratorRoute() = @Roles(MOD, ADMIN, SUPER_ADMIN) + @UseGuards(RolesGuard)

CASL:
  AbilityBuilder: can(), cannot(), build()
  can('manage', 'all') → super admin
  can('update', Post, { authorId: user.id }) → condition-based
  cannot('update', User, ['role']) → field-level restriction
  ability.can('read', post) → check at runtime

Multi-tenancy rule:
  ALWAYS filter queries by tenantId
  ALWAYS verify resource.tenantId === user.tenantId before returning
  Treat cross-tenant access as "not found" not "forbidden" (less info leak)

Security:
  Principle of Least Privilege → minimum access needed
  Never let users set their own role (separate admin endpoint)
  whitelist: true in ValidationPipe strips extra fields
  Short access token lifetime = role changes take effect quickly
```

---

*Day 9 complete. Tomorrow — Day 10: REST API Best Practices — Versioning, pagination, filtering, sorting, HATEOAS, rate limiting, compression, and security headers. The day for building production-grade APIs.*
