# CORS & CSRF

## CORS (Cross-Origin Resource Sharing)

### What is CORS?

CORS is a browser security mechanism that controls which domains are allowed to make HTTP requests to your server from a web page. By default, browsers block cross-origin requests — CORS allows you to explicitly permit certain origins.

**Origin** = protocol + domain + port

```
https://app.com         -> origin: https://app.com
https://api.app.com     -> DIFFERENT origin (different subdomain)
http://app.com          -> DIFFERENT origin (different protocol)
https://app.com:3000    -> DIFFERENT origin (different port)
```

### The Same-Origin Policy

Browsers enforce the Same-Origin Policy — JavaScript on `https://myapp.com` is NOT allowed to make fetch/XHR requests to `https://api.myapp.com` (different subdomain) without explicit permission from the server.

```
Browser page: https://myapp.com
JavaScript: fetch('https://api.myapp.com/users')
               |
               |-- Browser: this is cross-origin!
               |-- Browser: am I allowed? Ask the server...
               |-- Browser sends OPTIONS preflight request to api.myapp.com
               |-- If server says "allowed" -> request goes through
               |-- If server says nothing or "not allowed" -> BLOCKED
```

CORS is enforced by the **browser** — not by the server. The server receives and processes the request regardless; the browser blocks the response from reaching the JavaScript if CORS headers are missing/incorrect.

### Preflight Request (OPTIONS)

For non-simple requests (POST with JSON, PUT, DELETE, custom headers), the browser first sends an OPTIONS preflight to ask "is this allowed?":

```
OPTIONS /api/users HTTP/1.1
Host: api.myapp.com
Origin: https://myapp.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type, Authorization

Server responds:
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://myapp.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 3600    <- browser caches preflight for 1 hour
```

Then the actual POST request goes through.

### Simple Requests (No Preflight)

Simple requests (GET/POST with no custom headers, content-type is form data or text/plain) skip the preflight:

- `GET`, `HEAD`, `POST` with `Content-Type: application/x-www-form-urlencoded`, `multipart/form-data`, or `text/plain`
- No custom headers
- No credentials

### CORS Headers

| Header | Direction | Meaning |
|---|---|---|
| `Access-Control-Allow-Origin` | Response | Which origins are allowed |
| `Access-Control-Allow-Methods` | Response | Which HTTP methods are allowed |
| `Access-Control-Allow-Headers` | Response | Which request headers are allowed |
| `Access-Control-Allow-Credentials` | Response | Whether cookies/auth headers can be included |
| `Access-Control-Max-Age` | Response | How long to cache preflight response (seconds) |
| `Access-Control-Expose-Headers` | Response | Which response headers JS can access |
| `Origin` | Request | The origin of the requesting page |

### Spring Boot CORS Configuration

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins(
                        "https://myapp.com",
                        "https://www.myapp.com",
                        "http://localhost:3000"  // for local development
                )
                .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
                .allowedHeaders("Content-Type", "Authorization", "X-Requested-With")
                .allowCredentials(true)   // allow cookies to be sent
                .maxAge(3600);            // cache preflight for 1 hour
    }
}
```

For Spring Security:

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .cors(cors -> cors.configurationSource(corsConfigurationSource()))
        // ... other config
    return http.build();
}

@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://myapp.com", "http://localhost:3000"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
    config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
    config.setAllowCredentials(true);
    config.setMaxAge(3600L);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}
```

### Node.js CORS (cors package)

```javascript
const cors = require('cors');

// Permissive (dev only — never in production)
app.use(cors());

// Production — explicit whitelist
const corsOptions = {
    origin: (origin, callback) => {
        const allowedOrigins = [
            'https://myapp.com',
            'https://www.myapp.com',
            'http://localhost:3000'
        ];

        if (!origin || allowedOrigins.includes(origin)) {
            callback(null, true);
        } else {
            callback(new Error(`CORS policy: origin ${origin} not allowed`));
        }
    },
    methods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
    allowedHeaders: ['Content-Type', 'Authorization'],
    credentials: true,  // allow cookies
    maxAge: 3600
};

app.use(cors(corsOptions));
```

### Common CORS Mistakes

**1. Using wildcard `*` with credentials:**

```
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
```

Browsers reject this combination — you can't use a wildcard origin AND allow credentials. Specify the exact origin.

**2. Missing CORS config on Spring Security:**

If you have both `WebMvcConfigurer` CORS and Spring Security, the security filter runs first and may block requests before CORS headers are added. Always configure CORS through Spring Security's `.cors()` method when using Spring Security.

**3. Incorrect preflight handling:**

Make sure OPTIONS requests to your endpoints return 200/204 immediately (without auth checks) — otherwise your preflight fails with 401 and the actual request never goes through.

---

## CSRF (Cross-Site Request Forgery)

### What is CSRF?

CSRF is an attack where a malicious website tricks an authenticated user's browser into making unauthorized requests to your server, exploiting the fact that the browser automatically sends cookies with every request.

### How CSRF Works

```
1. User logs into bank.com
   Browser stores session cookie: session_id=abc123

2. User visits evil.com (while still logged in to bank.com)

3. evil.com's page contains:
   <form action="https://bank.com/transfer" method="POST" id="f">
     <input name="to" value="attacker_account">
     <input name="amount" value="10000">
   </form>
   <script>document.getElementById('f').submit()</script>

4. Browser submits form to bank.com
   Browser AUTOMATICALLY includes session cookie: session_id=abc123
   bank.com sees a valid authenticated session -> transfers $10,000!

5. User never clicked anything intentionally
```

The bank's server can't distinguish between a legitimate request from the user and a forged request from evil.com — both carry the same session cookie.

### Why Doesn't CORS Prevent CSRF?

CORS prevents JavaScript from reading cross-origin responses, but a form `<form method="POST">` submission doesn't use JavaScript — it's a native browser action and bypasses CORS. Also, SameSite cookies are the modern fix, but CSRF tokens are still needed for older browsers and complex scenarios.

### CSRF Prevention Strategies

#### 1. Synchronizer Token Pattern (CSRF Tokens) — Classic

Server generates a random, unpredictable token per session. Embeds it in every form. On form submission, validates the token. evil.com can't read the token (Same-Origin Policy prevents cross-origin page reads), so it can't forge a valid request.

```html
<form method="POST" action="/transfer">
    <input type="hidden" name="_csrf" value="random-token-abc123">
    <input name="to" value="...">
    <button>Transfer</button>
</form>
```

```java
// Spring Security enables CSRF protection by default
// It injects the token automatically into forms
// Validates it on POST/PUT/DELETE requests

// For REST APIs, read token from cookie and send in header
```

#### 2. Double Submit Cookie Pattern

Server sets a random value in a cookie. JavaScript reads the cookie and sends the same value in a request header. Server checks that cookie value matches header value.

```
Server sets: Set-Cookie: csrf_token=random123; SameSite=Strict
JS reads:    document.cookie -> csrf_token=random123
JS sends:    X-CSRF-Token: random123

Server checks: X-CSRF-Token header == csrf_token cookie
evil.com can't read the cookie (Same-Origin Policy) -> can't forge the header
```

#### 3. SameSite Cookie Attribute (Modern, Recommended)

Set cookies with `SameSite=Strict` or `SameSite=Lax` — browsers only send the cookie when the request originates from the same site.

```
Set-Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Strict
```

| SameSite Value | Cookie Sent When |
|---|---|
| `Strict` | Only for same-site requests (no cross-site, even from links) |
| `Lax` | For same-site + top-level navigation GET requests (links, redirects) |
| `None` | Always sent (must also set `Secure`) — used for third-party embedding |

`SameSite=Lax` is the browser default now (Chrome 80+). `SameSite=Strict` is safest but breaks OAuth login flows (the redirect from the OAuth server is cross-site). For most apps, `SameSite=Lax` is the right balance.

#### 4. Custom Request Headers

For SPA + REST API setups: require a custom header (e.g. `X-Requested-With: XMLHttpRequest`). Browsers won't include custom headers in cross-site form submissions or simple requests — they trigger CORS preflight, which the attacker can't pass. This isn't as strong as CSRF tokens but works as defense-in-depth.

### Spring Boot CSRF Configuration

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // CSRF enabled by default in Spring Security (for form-based apps)

            // For REST APIs with JWT (stateless), CSRF is usually disabled because:
            // - No session cookie (using JWT in Authorization header instead)
            // - JWTs can't be sent by a form submission automatically
            .csrf(csrf -> csrf.disable())  // OK for stateless JWT APIs

            // For session-based apps, keep CSRF enabled:
            // .csrf(csrf -> csrf
            //     .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
            //     // CookieCsrfTokenRepository stores token in cookie;
            //     // JS reads it and sends in X-XSRF-TOKEN header (Angular default)
            // )
            ;
        return http.build();
    }
}
```

**When to disable CSRF:**

If your API is stateless (JWT in `Authorization` header, no session cookies) and doesn't serve HTML forms, CSRF is not a threat — a forged cross-site request can't include the `Authorization: Bearer ...` header automatically. Disable CSRF to avoid overhead.

**When to keep CSRF enabled:**

If your app uses session cookies for authentication (traditional server-rendered apps, or cookies containing JWTs), CSRF protection is essential.

### CSRF in Node.js (csurf / manual)

```javascript
const cookieParser = require('cookie-parser');
const { randomBytes } = require('crypto');

// Middleware: generate and validate CSRF token
function csrfProtection(req, res, next) {
    if (['GET', 'HEAD', 'OPTIONS'].includes(req.method)) {
        // Generate token and set in cookie for safe methods
        if (!req.cookies['csrf_token']) {
            const token = randomBytes(32).toString('hex');
            res.cookie('csrf_token', token, {
                httpOnly: false,   // JS must read this
                secure: true,
                sameSite: 'Strict'
            });
        }
        return next();
    }

    // For state-changing methods, validate token
    const cookieToken = req.cookies['csrf_token'];
    const headerToken = req.headers['x-csrf-token'];

    if (!cookieToken || cookieToken !== headerToken) {
        return res.status(403).json({ error: 'CSRF token invalid' });
    }

    next();
}

// Frontend sends: headers: { 'X-CSRF-Token': getCookieValue('csrf_token') }
```

## CORS vs CSRF — Key Differences

| | CORS | CSRF |
|---|---|---|
| What it is | Browser security mechanism for cross-origin requests | Attack exploiting automatic cookie sending |
| Enforced by | Browser (blocks response from JS) | Application (validates tokens, SameSite cookies) |
| Goal | Prevent unauthorized domains from reading your API responses | Prevent malicious sites from performing actions as a logged-in user |
| Affects | Cross-origin JavaScript API calls | Cross-site form submissions and requests with cookies |
| Fix | Configure allowed origins in response headers | CSRF tokens, SameSite cookies, custom headers |

## Interview Questions

**Q1: What is CORS and why does it exist?**

CORS is a browser security policy that prevents JavaScript on one origin from reading responses from a different origin. It exists to protect users — without it, a malicious site could make API calls to your bank on your behalf and read the responses. CORS is enforced by the browser; the server still processes requests but the browser blocks the response from reaching the JS if CORS headers are missing or wrong.

**Q2: Does configuring CORS on the server prevent unauthorized access to your API?**

No — CORS only controls what browser JavaScript can read. Non-browser clients (curl, Postman, server-to-server) are completely unaffected by CORS headers. CORS is purely a browser security feature, not an API security feature. Server-side authentication and authorization are still required.

**Q3: What is CSRF and how does it differ from XSS?**

CSRF tricks the browser into making an unintended authenticated request (exploiting cookies). XSS injects malicious JavaScript into the page that runs in the victim's browser. CSRF exploits the trust a site has in the user's browser; XSS exploits the trust a user has in a site's content. They're separate attacks requiring separate defenses.

**Q4: Why is CSRF not a concern for REST APIs using JWT in Authorization headers?**

A CSRF attack relies on the browser automatically attaching authentication credentials (cookies) to cross-origin requests. If authentication uses a JWT in the `Authorization: Bearer ...` header, the browser does NOT attach this header automatically to cross-site requests — JavaScript has to explicitly set it. A forged form submission from evil.com can include cookies, but it can't include a custom `Authorization` header. So stateless JWT APIs are inherently CSRF-resistant.

**Q5: What does `SameSite=Lax` mean on a cookie?**

The browser sends the cookie on same-site requests AND on cross-site navigation GET requests (clicking a link). It does NOT send it on cross-site POST, PUT, DELETE. This prevents most CSRF attacks while still allowing normal OAuth login flows (where the OAuth redirect is a cross-site GET). Modern browsers default to `Lax` if `SameSite` is not specified.

**Q6: What is the preflight request and when is it triggered?**

A preflight is an OPTIONS request the browser sends automatically before a cross-origin request, to ask the server if the actual request is allowed. It's triggered for non-simple requests: POST with JSON (`Content-Type: application/json`), PUT, DELETE, PATCH, or any request with custom headers (`Authorization`, `X-Custom-Header`). If the server responds with appropriate `Access-Control-Allow-*` headers, the browser proceeds with the actual request.
