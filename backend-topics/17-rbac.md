# RBAC (Role-Based Access Control)

## What is RBAC?

Role-Based Access Control (RBAC) is an authorization model where permissions are assigned to roles, and roles are assigned to users. Users get permissions indirectly through their roles — not by direct permission assignment.

```
Without RBAC (direct permission assignment — messy):
User Alice: can_read_orders, can_write_orders, can_read_users, can_delete_users...
User Bob:   can_read_orders, can_write_orders...
User Carol: can_read_orders...
-> Adding a new permission requires updating every user individually

With RBAC:
Role ADMIN  -> [can_read_orders, can_write_orders, can_read_users, can_delete_users]
Role EDITOR -> [can_read_orders, can_write_orders]
Role VIEWER -> [can_read_orders]

User Alice -> ADMIN
User Bob   -> EDITOR
User Carol -> VIEWER
-> Adding a new permission to EDITOR updates ALL editors instantly
```

## Why RBAC?

### 1. Principle of Least Privilege

Users only get the permissions their role requires. A customer support agent doesn't need to delete user accounts — they get a SUPPORT role with read-only access.

### 2. Manageability at Scale

With 10,000 users, you don't assign permissions one by one. You assign roles. When permissions need to change, you update the role — all users in that role get the change automatically.

### 3. Auditability

Clear mapping: "who has what access and why?" The role is the reason. Easier to audit and demonstrate compliance (GDPR, SOC2, HIPAA).

### 4. Separation of Duties

Different roles enforce different responsibilities. An employee can't both create and approve a payment (prevents fraud).

## Core Concepts

### Role

A named collection of permissions. Examples: `ADMIN`, `MANAGER`, `EDITOR`, `VIEWER`, `SUPPORT`, `BILLING`.

### Permission (Privilege)

A specific action on a specific resource. Usually expressed as `action:resource`.

```
read:orders       -> can read order data
write:orders      -> can create/update orders
delete:users      -> can delete user accounts
approve:payments  -> can approve payments
manage:roles      -> can assign/remove roles
```

### User-Role Assignment

A user is assigned one or more roles. A role can be assigned to many users.

```
user_roles table:
user_id=1 -> ADMIN
user_id=2 -> EDITOR, SUPPORT    (multiple roles)
user_id=3 -> VIEWER
```

## Database Schema

```sql
-- Roles table
CREATE TABLE roles (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(50) UNIQUE NOT NULL,   -- 'ADMIN', 'EDITOR', 'VIEWER'
    description TEXT
);

-- Permissions table
CREATE TABLE permissions (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(100) UNIQUE NOT NULL,  -- 'read:orders', 'delete:users'
    description TEXT
);

-- Role-Permission mapping (many-to-many)
CREATE TABLE role_permissions (
    role_id       BIGINT REFERENCES roles(id) ON DELETE CASCADE,
    permission_id BIGINT REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

-- User-Role mapping (many-to-many)
CREATE TABLE user_roles (
    user_id BIGINT REFERENCES users(id) ON DELETE CASCADE,
    role_id BIGINT REFERENCES roles(id) ON DELETE CASCADE,
    PRIMARY KEY (user_id, role_id)
);
```

## Hierarchy: Flat RBAC vs Hierarchical RBAC

### Flat RBAC

No inheritance between roles. Each role is independent. Most common and simple.

```
ADMIN  -> [read:orders, write:orders, delete:orders, read:users, delete:users]
EDITOR -> [read:orders, write:orders]
VIEWER -> [read:orders]
```

### Hierarchical RBAC

Roles inherit permissions from lower roles. Less duplication.

```
VIEWER -> [read:orders]
EDITOR -> inherits VIEWER + [write:orders]
ADMIN  -> inherits EDITOR + [delete:orders, delete:users]
```

In practice, flat RBAC with explicit permission assignment is more maintainable and auditable than inheritance chains that become hard to reason about.

## Java Implementation (Spring Security)

### Entity Classes

```java
@Entity
public class Role {
    @Id @GeneratedValue
    private Long id;

    private String name;  // "ROLE_ADMIN", "ROLE_EDITOR", "ROLE_VIEWER"

    @ManyToMany(fetch = FetchType.EAGER)
    @JoinTable(
        name = "role_permissions",
        joinColumns = @JoinColumn(name = "role_id"),
        inverseJoinColumns = @JoinColumn(name = "permission_id")
    )
    private Set<Permission> permissions;
}

@Entity
public class Permission {
    @Id @GeneratedValue
    private Long id;

    private String name;  // "read:orders", "write:orders", "delete:users"
}

@Entity
public class User {
    @Id @GeneratedValue
    private Long id;

    private String email;
    private String password;

    @ManyToMany(fetch = FetchType.EAGER)
    @JoinTable(
        name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    private Set<Role> roles;
}
```

### UserDetailsService — Load User + Roles + Permissions

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {

    @Autowired
    private UserRepository userRepo;

    @Override
    public UserDetails loadUserByUsername(String email) throws UsernameNotFoundException {

        User user = userRepo.findByEmail(email)
                .orElseThrow(() -> new UsernameNotFoundException("User not found: " + email));

        // Flatten roles + permissions into GrantedAuthority list
        Set<GrantedAuthority> authorities = new HashSet<>();

        for (Role role : user.getRoles()) {
            // Add role itself as authority
            authorities.add(new SimpleGrantedAuthority("ROLE_" + role.getName()));

            // Add all permissions of this role
            for (Permission perm : role.getPermissions()) {
                authorities.add(new SimpleGrantedAuthority(perm.getName()));
            }
        }

        return new org.springframework.security.core.userdetails.User(
                user.getEmail(),
                user.getPassword(),
                authorities
        );
    }
}
```

### JWT — Include Roles and Permissions in Token

```java
public String generateToken(User user) {

    List<String> roles = user.getRoles().stream()
            .map(Role::getName)
            .toList();

    List<String> permissions = user.getRoles().stream()
            .flatMap(r -> r.getPermissions().stream())
            .map(Permission::getName)
            .distinct()
            .toList();

    return Jwts.builder()
            .setSubject(user.getEmail())
            .claim("roles", roles)
            .claim("permissions", permissions)
            .setExpiration(new Date(System.currentTimeMillis() + 900_000))
            .signWith(secretKey)
            .compact();
}
```

Including roles/permissions in the JWT means downstream services don't need to query the DB on every request — the JWT is self-contained.

### Security Config — Role and Permission-Based Access

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                // Public endpoints
                .requestMatchers("/api/auth/**").permitAll()

                // Role-based
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers(HttpMethod.DELETE, "/api/**").hasRole("ADMIN")

                // Permission-based (more granular)
                .requestMatchers(HttpMethod.GET, "/api/orders/**").hasAuthority("read:orders")
                .requestMatchers(HttpMethod.POST, "/api/orders/**").hasAuthority("write:orders")

                .anyRequest().authenticated()
            );
        return http.build();
    }
}
```

### Method-Level Security (Fine-Grained)

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @GetMapping
    @PreAuthorize("hasAuthority('read:orders')")
    public List<Order> getAllOrders() { ... }

    @PostMapping
    @PreAuthorize("hasAuthority('write:orders')")
    public Order createOrder(@RequestBody OrderRequest req) { ... }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN') or hasAuthority('delete:orders')")
    public void deleteOrder(@PathVariable Long id) { ... }

    @GetMapping("/my")
    @PreAuthorize("hasAuthority('read:orders') and #userId == authentication.principal.id")
    public List<Order> getMyOrders(@RequestParam Long userId) { ... }
}
```

### Custom Permission Evaluator

```java
@Component
public class CustomPermissionEvaluator implements PermissionEvaluator {

    @Autowired
    private OrderRepository orderRepo;

    @Override
    public boolean hasPermission(Authentication auth, Object targetDomainObject, Object permission) {
        if (targetDomainObject instanceof Order order) {
            String username = auth.getName();
            if ("read".equals(permission)) {
                // Users can read their own orders; admins can read all
                return order.getUserEmail().equals(username)
                        || auth.getAuthorities().stream()
                                .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"));
            }
        }
        return false;
    }

    @Override
    public boolean hasPermission(Authentication auth, Serializable targetId, String targetType, Object permission) {
        if ("Order".equals(targetType)) {
            Order order = orderRepo.findById((Long) targetId).orElse(null);
            return order != null && hasPermission(auth, order, permission);
        }
        return false;
    }
}
```

Usage:

```java
@PreAuthorize("hasPermission(#id, 'Order', 'read')")
public Order getOrder(@PathVariable Long id) { ... }
```

## Node.js Implementation

### Middleware

```javascript
// Check role middleware
function requireRole(...roles) {
    return (req, res, next) => {
        const userRoles = req.user.roles || [];
        const hasRole = roles.some(role => userRoles.includes(role));

        if (!hasRole) {
            return res.status(403).json({ error: 'Forbidden: insufficient role' });
        }
        next();
    };
}

// Check permission middleware
function requirePermission(permission) {
    return (req, res, next) => {
        const userPermissions = req.user.permissions || [];

        if (!userPermissions.includes(permission)) {
            return res.status(403).json({ error: `Forbidden: missing permission '${permission}'` });
        }
        next();
    };
}

// Routes
router.get('/orders', requirePermission('read:orders'), getOrders);
router.post('/orders', requirePermission('write:orders'), createOrder);
router.delete('/orders/:id', requireRole('ADMIN'), deleteOrder);
router.get('/admin', requireRole('ADMIN', 'MANAGER'), adminDashboard);
```

## RBAC vs ABAC vs ACL

| Model | Based On | Flexibility | Complexity | Best For |
|---|---|---|---|---|
| RBAC | Roles | Medium | Low | Most applications |
| ABAC (Attribute-Based) | Attributes of user, resource, environment | Very High | High | Complex policies (e.g. "only during business hours from corporate IP") |
| ACL (Access Control List) | Per-resource permission list per user | High | Very High | File systems, fine-grained resource ownership |

RBAC covers 90% of real-world backend authorization needs.

## Common RBAC Patterns in Real Systems

### SaaS Multi-Tenant RBAC

In a SaaS app, each organization has its own roles:

```sql
-- Roles are scoped per organization
CREATE TABLE org_roles (
    id      BIGSERIAL PRIMARY KEY,
    org_id  BIGINT NOT NULL,
    name    VARCHAR(50) NOT NULL,
    UNIQUE (org_id, name)
);

-- User membership in an org with a role
CREATE TABLE org_memberships (
    user_id BIGINT,
    org_id  BIGINT,
    role_id BIGINT,
    PRIMARY KEY (user_id, org_id)
);
```

Alice can be ADMIN in Org A but only VIEWER in Org B — roles are scoped per tenant.

### Super Admin vs Tenant Admin

```
SUPER_ADMIN    -> global permissions (manage all tenants, platform settings)
TENANT_ADMIN   -> admin within their org only
TENANT_MEMBER  -> regular user within their org
```

## Interview Questions

**Q1: What is the difference between authentication and authorization?**

Authentication proves identity — "who are you?" (login, JWT validation). Authorization controls access — "what are you allowed to do?" (RBAC, permission checks). OAuth 2.0 and JWT handle authentication; RBAC handles authorization. Spring Security's `@PreAuthorize` is authorization — it happens after authentication.

**Q2: Why use permission-based authorization instead of just role-based?**

Roles are coarse-grained ("ADMIN", "EDITOR"). Permissions are fine-grained ("write:orders", "delete:users"). A role is just a named group of permissions. Checking permissions directly makes authorization logic more precise and more maintainable — you can add a new permission to an existing role without changing any code. Checking `hasRole('ADMIN')` everywhere means admin semantics are hardcoded in code rather than in the DB.

**Q3: Why include roles/permissions in the JWT instead of loading them from DB on each request?**

Loading from DB on every request adds latency and DB load. Including them in the JWT makes the token self-contained — any service can validate authorization without a DB call. Tradeoff: if you revoke a role, the user's JWT still contains the old roles until it expires. Mitigate with short-lived tokens (15 min) and refresh token flows.

**Q4: What is the Principle of Least Privilege?**

Users and services should only have the minimum permissions required to do their job. A customer support agent should have read:orders but not delete:users. A payment service should have write:payments but not read:users. This limits blast radius if an account is compromised or misused.

**Q5: How do you handle multi-tenant RBAC in a SaaS application?**

Scope roles per organization — a user can be ADMIN in one org and VIEWER in another. Store memberships as (user_id, org_id, role_id) tuples. When authorizing, always include org_id in the check — don't just check the role in isolation. The JWT or session should carry the active org context.
