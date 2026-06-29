# Session-Based Authentication

## What is Session-Based Authentication?

Session-based authentication is a stateful authentication mechanism where the server creates and stores a session for each logged-in user. The client receives a session ID (stored in a cookie) and sends it on every request. The server looks up the session to identify and authenticate the user.

```
Login:
Client -> POST /login { email, password }
Server -> verifies credentials
       -> creates session: { sessionId: "abc123", userId: 1, roles: ["USER"] }
       -> stores session in memory/Redis/DB
       -> sends Set-Cookie: JSESSIONID=abc123; HttpOnly; Secure
Client -> stores cookie (browser does this automatically)

Subsequent requests:
Client -> GET /profile
          Cookie: JSESSIONID=abc123   (browser sends automatically)
Server -> looks up session "abc123"
       -> finds { userId: 1, roles: ["USER"] }
       -> serves request
```

## Session-Based vs JWT (Token-Based)

| Feature | Session-Based | JWT (Token-Based) |
|---|---|---|
| State | Stateful — server stores sessions | Stateless — no server storage |
| Storage | Server memory / Redis / DB | Client (cookie or localStorage) |
| Scalability | Needs shared session store for multiple servers | Any server can verify (no shared state) |
| Revocation | Instant — delete session from store | Hard — token valid until expiry |
| Data per request | One lookup per request | Token decoded locally (no lookup) |
| Best for | Traditional web apps, admin panels | Microservices, SPAs, mobile APIs |

## How Sessions Work Internally

```
Session Lifecycle:

1. LOGIN
   POST /login { email, password }
   Server: verify credentials
   Server: session = { id: UUID, userId: 42, roles: ["USER"], createdAt: ..., lastAccessed: ... }
   Server: store session in Redis (key: "session:UUID", value: session, TTL: 30min)
   Server: Set-Cookie: SESSION=UUID; HttpOnly; Secure; SameSite=Lax; Path=/

2. AUTHENTICATED REQUEST
   GET /dashboard
   Cookie: SESSION=UUID
   Server: get "session:UUID" from Redis
   Server: if found and not expired -> authenticated
   Server: update lastAccessed time

3. LOGOUT
   POST /logout
   Cookie: SESSION=UUID
   Server: delete "session:UUID" from Redis
   Server: Set-Cookie: SESSION=; Expires=Thu, 01 Jan 1970... (clear cookie)

4. EXPIRY
   Redis TTL expires session automatically
   Next request: session not found -> 401 Unauthorized -> redirect to login
```

## Session Storage Options

### 1. In-Memory (Development Only)

```java
// Spring Boot default — sessions stored in application memory
// Works for single server, dies on restart, lost on scale-out
```

**Problems:**
- Server restart = all users logged out
- Multiple servers = user's session only on one server (needs sticky sessions)
- Memory grows with active users

### 2. Redis (Production Standard)

```java
// Sessions stored in Redis — shared across all servers
// Fast, TTL support, survives app restarts
```

Architecture:
```
App Server 1 ---+
App Server 2 ---+--> Redis (session store) --> All servers share sessions
App Server 3 ---+
```

### 3. Database

```sql
CREATE TABLE http_sessions (
    id           VARCHAR(255) PRIMARY KEY,
    user_id      BIGINT REFERENCES users(id),
    data         JSONB,         -- session attributes
    created_at   TIMESTAMP DEFAULT NOW(),
    last_accessed TIMESTAMP,
    expires_at   TIMESTAMP,
    CONSTRAINT not_expired CHECK (expires_at > NOW())
);
```

Slower than Redis but persistent and queryable (useful for "show all active sessions" feature).

## Spring Boot Session Implementation

### Dependencies

```xml
<!-- Spring Session with Redis -->
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

### Configuration

```yaml
spring:
  redis:
    host: localhost
    port: 6379

  session:
    store-type: redis
    timeout: 30m           # session TTL
    redis:
      namespace: myapp:sessions
      flush-mode: on-save  # write to Redis on session modification

server:
  servlet:
    session:
      cookie:
        name: SESSION
        http-only: true
        secure: true        # HTTPS only
        same-site: lax
```

### Security Config

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**", "/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginProcessingUrl("/api/auth/login")
                .successHandler(loginSuccessHandler())
                .failureHandler(loginFailureHandler())
            )
            .logout(logout -> logout
                .logoutUrl("/api/auth/logout")
                .logoutSuccessHandler(logoutSuccessHandler())
                .invalidateHttpSession(true)  // destroy session
                .deleteCookies("SESSION")     // clear cookie
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
                .maximumSessions(1)               // one session per user
                .maxSessionsPreventsLogin(false)  // new login invalidates old session
            );

        return http.build();
    }

    // Return JSON instead of redirect (for REST APIs)
    @Bean
    public AuthenticationSuccessHandler loginSuccessHandler() {
        return (request, response, auth) -> {
            response.setContentType("application/json");
            response.setStatus(200);
            User user = (User) auth.getPrincipal();
            response.getWriter().write(
                "{\"message\": \"Login successful\", \"userId\": \"" + user.getUsername() + "\"}"
            );
        };
    }

    @Bean
    public AuthenticationFailureHandler loginFailureHandler() {
        return (request, response, ex) -> {
            response.setContentType("application/json");
            response.setStatus(401);
            response.getWriter().write("{\"error\": \"Invalid credentials\"}");
        };
    }

    @Bean
    public LogoutSuccessHandler logoutSuccessHandler() {
        return (request, response, auth) -> {
            response.setContentType("application/json");
            response.setStatus(200);
            response.getWriter().write("{\"message\": \"Logged out successfully\"}");
        };
    }
}
```

### Custom Login (REST-based)

```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {

    @Autowired
    private AuthenticationManager authManager;

    @Autowired
    private UserRepository userRepo;

    @PostMapping("/login")
    public ResponseEntity<?> login(@RequestBody LoginRequest req,
                                   HttpServletRequest request) {
        try {
            Authentication auth = authManager.authenticate(
                    new UsernamePasswordAuthenticationToken(req.getEmail(), req.getPassword())
            );

            // Store authentication in session
            SecurityContextHolder.getContext().setAuthentication(auth);
            HttpSession session = request.getSession(true);
            session.setAttribute(
                    HttpSessionSecurityContextRepository.SPRING_SECURITY_CONTEXT_KEY,
                    SecurityContextHolder.getContext()
            );

            return ResponseEntity.ok(Map.of(
                    "message", "Login successful",
                    "sessionId", session.getId()
            ));

        } catch (BadCredentialsException e) {
            return ResponseEntity.status(401).body(Map.of("error", "Invalid credentials"));
        }
    }

    @PostMapping("/logout")
    public ResponseEntity<?> logout(HttpServletRequest request) {
        HttpSession session = request.getSession(false);
        if (session != null) {
            session.invalidate();
        }
        SecurityContextHolder.clearContext();
        return ResponseEntity.ok(Map.of("message", "Logged out"));
    }

    // Get current user from session
    @GetMapping("/me")
    public ResponseEntity<?> currentUser(HttpSession session) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth == null || !auth.isAuthenticated()) {
            return ResponseEntity.status(401).body("Not authenticated");
        }
        return ResponseEntity.ok(Map.of("user", auth.getName(), "roles", auth.getAuthorities()));
    }
}
```

### Storing Custom Data in Session

```java
@PostMapping("/login")
public ResponseEntity<?> login(@RequestBody LoginRequest req, HttpSession session) {

    User user = authenticate(req);

    // Store anything in session
    session.setAttribute("userId", user.getId());
    session.setAttribute("userEmail", user.getEmail());
    session.setAttribute("roles", user.getRoles());
    session.setAttribute("loginTime", LocalDateTime.now());
    session.setAttribute("ipAddress", request.getRemoteAddr());

    return ResponseEntity.ok("Login successful");
}

// Access session data anywhere
@GetMapping("/orders")
public List<Order> getOrders(HttpSession session) {
    Long userId = (Long) session.getAttribute("userId");
    return orderService.getOrdersForUser(userId);
}
```

## Node.js Implementation (Express + express-session)

```javascript
const express = require('express');
const session = require('express-session');
const RedisStore = require('connect-redis').default;
const { createClient } = require('redis');
const bcrypt = require('bcrypt');

const app = express();

// Redis client
const redisClient = createClient({ url: 'redis://localhost:6379' });
redisClient.connect();

// Session middleware
app.use(session({
    store: new RedisStore({ client: redisClient }),
    secret: process.env.SESSION_SECRET,
    name: 'SESSION',
    resave: false,
    saveUninitialized: false,
    cookie: {
        httpOnly: true,
        secure: process.env.NODE_ENV === 'production',
        sameSite: 'lax',
        maxAge: 30 * 60 * 1000  // 30 minutes
    }
}));

// Auth middleware
function requireAuth(req, res, next) {
    if (!req.session.userId) {
        return res.status(401).json({ error: 'Not authenticated' });
    }
    next();
}

// Login
app.post('/api/auth/login', async (req, res) => {
    const { email, password } = req.body;

    const user = await User.findOne({ email });
    if (!user || !await bcrypt.compare(password, user.passwordHash)) {
        return res.status(401).json({ error: 'Invalid credentials' });
    }

    // Regenerate session ID on login (prevent session fixation)
    req.session.regenerate((err) => {
        if (err) return res.status(500).json({ error: 'Session error' });

        req.session.userId = user.id;
        req.session.email = user.email;
        req.session.roles = user.roles;

        res.json({ message: 'Login successful', userId: user.id });
    });
});

// Logout
app.post('/api/auth/logout', (req, res) => {
    req.session.destroy((err) => {
        res.clearCookie('SESSION');
        res.json({ message: 'Logged out' });
    });
});

// Protected route
app.get('/api/orders', requireAuth, async (req, res) => {
    const orders = await Order.find({ userId: req.session.userId });
    res.json(orders);
});
```

## Session Security

### Session Fixation Attack

Attacker sets their own session ID in the victim's browser, then the victim logs in — now the attacker knows the valid session ID.

**Fix:** always regenerate the session ID on login:

```java
// Spring Security does this automatically on login
// In Node.js: req.session.regenerate() before setting auth data
```

### Session Hijacking

Attacker steals a valid session cookie (via XSS, network sniffing, etc.).

**Fixes:**
- `HttpOnly` cookie — JS can't read it (blocks XSS theft)
- `Secure` cookie — only sent over HTTPS (blocks network sniffing)
- `SameSite` cookie — reduces CSRF risk
- IP binding — validate session is from the same IP (breaks mobile users, use with care)
- Short TTL — sessions expire quickly

### Concurrent Session Control

```java
// Allow only 1 session per user — new login kicks old session
.sessionManagement(session -> session
    .maximumSessions(1)
    .maxSessionsPreventsLogin(false)  // false = kick old, true = block new login
    .expiredSessionStrategy(event ->
        event.getResponse().sendError(401, "Session expired — logged in elsewhere"))
)
```

## Viewing and Managing Active Sessions

With Spring Session + Redis, you can implement "active sessions" management:

```java
@Service
public class SessionManagementService {

    @Autowired
    private FindByIndexNameSessionRepository<? extends Session> sessionRepo;

    // Get all sessions for a user
    public List<SessionInfo> getActiveSessions(String username) {
        Map<String, ? extends Session> sessions =
                sessionRepo.findByPrincipalName(username);

        return sessions.values().stream()
                .map(s -> new SessionInfo(
                        s.getId(),
                        s.getCreationTime(),
                        s.getLastAccessedTime(),
                        s.getAttribute("ipAddress")
                ))
                .toList();
    }

    // Revoke a specific session (force logout)
    public void revokeSession(String sessionId) {
        sessionRepo.deleteById(sessionId);
    }

    // Revoke all sessions for a user (force logout everywhere)
    public void revokeAllSessions(String username) {
        sessionRepo.findByPrincipalName(username)
                .keySet()
                .forEach(sessionRepo::deleteById);
    }
}
```

## Interview Questions

**Q1: What is the difference between session-based and JWT authentication?**

Session-based is stateful — the server stores session data, the client holds only an ID. JWT is stateless — all data is in the token, the server stores nothing. Sessions allow instant revocation (delete from store) but require a shared store for multi-server deployments. JWTs need no shared state but can't be revoked before expiry. Sessions suit traditional web apps; JWTs suit microservices and SPAs.

**Q2: Why store sessions in Redis instead of server memory?**

In-memory sessions are lost on server restart and don't work across multiple servers — a user logged into Server 1 gets rejected by Server 2. Redis provides a shared, fast, persistent session store all servers access. TTL support auto-expires sessions. Sub-millisecond lookups make session validation negligible overhead.

**Q3: What is session fixation and how do you prevent it?**

An attacker tricks the browser into using a known session ID before login. When the victim logs in, that session ID becomes valid, giving the attacker access. Prevent it by regenerating the session ID immediately on successful login — the old (attacker-known) ID becomes invalid and a new random ID is assigned.

**Q4: What is the `HttpOnly` flag on a session cookie and why is it critical?**

`HttpOnly` prevents JavaScript from accessing the cookie via `document.cookie`. Even if an XSS vulnerability exists, malicious scripts can't read the session cookie to steal it. The cookie is still sent by the browser on HTTP requests — it's only invisible to JavaScript.

**Q5: How do you implement "log out of all devices" with session-based auth?**

Store a reference to all session IDs per user (Spring Session's `findByPrincipalName` does this with Redis). On "logout all", iterate and delete every session ID for that user from Redis. All their sessions are immediately invalidated — next request on any device gets a 401.
