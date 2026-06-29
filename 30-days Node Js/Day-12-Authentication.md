# Day 12 — Authentication: JWT, Sessions, bcrypt, OAuth, RBAC

> **Why this day matters:** Authentication is asked in nearly every backend interview, at every level — but at 3 YOE, the bar is higher than "I used `jwt.sign()`." Interviewers want to know: do you understand **what's actually inside a JWT and why it can be trusted**, **why we hash passwords the way we do**, **session vs token tradeoffs**, and **how to design access control for different user roles**. Today builds all of that from the ground up.

---

## 1. Background: what problem is authentication actually solving?

Two distinct concepts get conflated constantly — know the difference cold, because interviewers test this directly:

- **Authentication (AuthN)** — "Who are you?" Verifying identity (logging in with a password, a valid token, etc.)
- **Authorization (AuthZ)** — "What are you allowed to do?" Checking permissions AFTER identity is established (this connects directly back to Day 9's 401 vs 403 distinction — 401 is an AuthN failure, 403 is an AuthZ failure)

HTTP is **stateless** by design (Day 9) — the server doesn't inherently remember who you are between requests. Every authentication strategy is essentially a different answer to: **"How do we let the server recognize the same user across multiple separate, stateless requests?"**

---

## 2. Password Hashing with bcrypt — the foundation before any token exists

### Background: why can't we just store passwords as plain text, or even just encrypt them?

**Plain text** is obviously catastrophic — a single database breach exposes every user's actual password, and since people commonly reuse passwords across sites, this compromises their accounts elsewhere too.

**Encryption** (reversible) is also wrong here, because encryption implies you can decrypt it back to the original — meaning if an attacker steals your encryption key (often stored on the same compromised server), they get every password back in plain text too.

**Hashing** is the correct approach — it's a **one-way** transformation. You can verify a password is correct by hashing the input and comparing it to the stored hash, but you can never reverse a hash back into the original password. Even if your database is fully breached, attackers only get hashes, not usable passwords (assuming a strong hashing algorithm).

```javascript
const bcrypt = require('bcrypt');

// --- HASHING (at registration time) ---
// The SECOND argument, 10, is the "salt rounds" / "cost factor" — it
// controls how computationally expensive the hash is to compute.
// Higher = slower to hash, but ALSO slower for an attacker to brute-force
// guess. 10-12 is a common production default — it's a deliberate
// tradeoff between security and acceptable login latency.
async function registerUser(email, plainPassword) {
  const hashedPassword = await bcrypt.hash(plainPassword, 10);
  // Store hashedPassword in the DB — NEVER store plainPassword, ever, anywhere.
  return await User.create({ email, password: hashedPassword });
}

// --- VERIFYING (at login time) ---
async function loginUser(email, plainPassword) {
  const user = await User.findOne({ where: { email } });
  if (!user) return null;

  // bcrypt.compare() re-hashes the INPUT password using the SAME salt
  // that's embedded in the stored hash, and compares the results.
  // This is why you never need to "store the salt separately" — bcrypt
  // embeds it directly inside the hash string it produces.
  const isMatch = await bcrypt.compare(plainPassword, user.password);
  return isMatch ? user : null;
}
```

### Why bcrypt specifically (not MD5, not SHA-256 alone) — a real interview question

**The precise answer:** MD5 and plain SHA-256 are **fast** hashing algorithms — designed for speed, which is exactly the WRONG property for password hashing. Fast hashing means an attacker with a stolen hash database can try billions of password guesses per second (especially with GPU/ASIC acceleration) until one matches. **bcrypt (and similarly Argon2, scrypt) are deliberately slow**, and bcrypt specifically has a tunable cost factor that can be increased over time as hardware gets faster, keeping brute-force attacks computationally expensive even years later. This deliberate slowness is the entire point — it's a feature, not a flaw.

**Always use the async version of bcrypt in a server context** (`bcrypt.hash`, not `bcrypt.hashSync`) — remember Day 2's event loop lesson: a synchronous, CPU-intensive hash operation would block the single JS thread, freezing every other concurrent request for the duration of the hash computation. This is a direct, practical application of Day 2's theory to a real security library.

---

## 3. Sessions — the traditional, stateful approach

### Background: how did the web handle "staying logged in" before JWTs existed?

```
1. User logs in with correct credentials
2. Server creates a SESSION (a record, often in memory or Redis, containing
   the user's ID and any other relevant data) and generates a random,
   unguessable SESSION ID
3. Server sends that session ID back to the browser as a COOKIE
4. Browser automatically attaches that cookie to every subsequent request
5. Server looks up the session ID in its session store on every request,
   finds the matching user, and knows who's making the request
```

```javascript
const session = require('express-session');

app.use(session({
  secret: 'your-secret-key', // used to sign the session ID cookie, preventing tampering
  resave: false,
  saveUninitialized: false,
  cookie: { secure: true, httpOnly: true, maxAge: 24 * 60 * 60 * 1000 }, // 1 day
}));

app.post('/login', async (req, res) => {
  const user = await loginUser(req.body.email, req.body.password);
  if (!user) return res.status(401).json({ error: 'Invalid credentials' });

  // express-session handles creating the session and setting the cookie.
  // We just attach the data WE want to remember about this user.
  req.session.userId = user.id;
  res.json({ message: 'Logged in' });
});

app.get('/profile', (req, res) => {
  if (!req.session.userId) {
    return res.status(401).json({ error: 'Not logged in' });
  }
  res.json({ userId: req.session.userId });
});
```

**Key characteristic: sessions are STATEFUL** — the server must store session data somewhere (memory, a database, or Redis — which becomes essential once you have more than one server instance, foreshadowing Day 17). The cookie itself is just a pointer/key; the actual user data lives server-side.

---

## 4. JWT (JSON Web Tokens) — the stateless alternative

### Background: why move away from sessions at all?

The session model has a real scaling friction: if your session data lives in one server's memory, and you have **multiple server instances behind a load balancer** (Day 23's topic), a user's session might be created on Server A but their next request could land on Server B — which has no idea who they are. You'd need a shared session store (Redis) to fix this. JWTs sidestep this problem entirely by **not requiring the server to store anything about the session at all** — that's the core tradeoff to understand.

### What's actually inside a JWT — this is the part most engineers can't explain precisely

A JWT is a string made of **three parts separated by dots**: `header.payload.signature`

```
eyJhbGciOiJIUzI1NiJ9.eyJ1c2VySWQiOjEsInJvbGUiOiJhZG1pbiJ9.4f7f8f...
└──────── HEADER ─────────┘└─────────── PAYLOAD ───────────┘└─ SIGNATURE ─┘
```

**Each part, decoded:**

```javascript
// HEADER (base64-encoded JSON) — describes the token itself
{ "alg": "HS256", "typ": "JWT" } // which signing algorithm was used

// PAYLOAD (base64-encoded JSON) — the actual data/claims
{ "userId": 1, "role": "admin", "iat": 1719234000, "exp": 1719237600 }
// iat = issued at, exp = expiration time -- both standard, optional claims

// SIGNATURE — THIS is the critical, often misunderstood part.
// It's NOT encryption — anyone can decode the header and payload above
// (they're just base64, not encrypted — NEVER put secrets like a
// password in a JWT payload, since anyone holding the token can read it).
// The signature is a cryptographic HASH of the header + payload, combined
// with a SECRET KEY that only the server knows.
```

**Why the signature is the entire security model:** when the server later receives this token back, it recomputes the signature using the header + payload it received, combined with its own secret key, and checks if that matches the signature included in the token. **If even one character of the payload was tampered with** (e.g., someone tried to change `"role": "user"` to `"role": "admin"` by editing the base64 payload directly), the recomputed signature would NOT match the original signature, and the server rejects the token. This is what makes a JWT **tamper-evident** without the server needing to store anything — the proof of validity travels WITH the token itself.

```javascript
const jwt = require('jsonwebtoken');

// --- ISSUING a token (at login) ---
function generateToken(user) {
  return jwt.sign(
    { userId: user.id, role: user.role }, // the PAYLOAD — keep this small and non-sensitive
    process.env.JWT_SECRET,                // the SECRET — must be kept safe, never committed to git
    { expiresIn: '1h' }                    // sets the 'exp' claim automatically
  );
}

app.post('/login', async (req, res) => {
  const user = await loginUser(req.body.email, req.body.password);
  if (!user) return res.status(401).json({ error: 'Invalid credentials' });

  const token = generateToken(user);
  res.json({ token }); // client stores this (commonly in memory or localStorage on the frontend)
});

// --- VERIFYING a token (on every protected request) ---
function requireAuth(req, res, next) {
  const authHeader = req.headers.authorization; // expected format: "Bearer <token>"
  const token = authHeader?.split(' ')[1];

  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }

  try {
    // jwt.verify() recomputes the signature and checks it matches, AND
    // checks the 'exp' claim hasn't passed. Throws if either check fails.
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded; // attach decoded payload to req for later handlers to use
    next();
  } catch (err) {
    return res.status(401).json({ error: 'Invalid or expired token' });
  }
}

app.get('/profile', requireAuth, (req, res) => {
  res.json({ userId: req.user.userId, role: req.user.role });
});
```

### Sessions vs JWT — the comparison table interviewers expect

| Factor | Sessions | JWT |
|---|---|---|
| State | Stateful — server stores session data | Stateless — server stores nothing; all data is IN the token |
| Scaling across servers | Needs a shared store (Redis) | Works naturally across any number of servers (any server with the secret can verify) |
| Revocation (logging a user out early, before expiry) | Easy — just delete the session server-side | Hard — the token is valid until it expires, no built-in revocation (mitigations exist, see below) |
| Token size | Small (just an ID) | Larger (contains actual payload data, sent on every request) |
| Common transport | Cookie (often `httpOnly`, somewhat XSS-resistant by default) | Often sent as a header, frequently stored in frontend JS-accessible storage (more exposed to XSS if not handled carefully) |

**The hard problem with JWT revocation — a real, frequently-asked nuance:** if a JWT is stolen, or a user needs to be logged out immediately (e.g., they reported their account compromised), you can't simply "delete" a stateless token — it remains valid until its `exp` time, since the server never stored it to begin with. Common mitigations: keep JWT expiry SHORT (e.g., 15 minutes) paired with a separate, longer-lived **refresh token** that CAN be revoked (often stored server-side or in Redis), or maintain a server-side denylist of revoked token IDs (which reintroduces some statefulness, but only for revocation, not for every request). Being able to explain this tradeoff honestly is a strong senior-level signal.

---

## 5. Refresh Tokens — the standard production pattern

```javascript
// Access token: short-lived (minutes), used for actual API requests
const accessToken = jwt.sign({ userId: user.id }, process.env.ACCESS_SECRET, { expiresIn: '15m' });

// Refresh token: long-lived (days/weeks), used ONLY to get a new access
// token when the old one expires — and stored server-side (e.g., in a
// DB or Redis) so it CAN be revoked/invalidated if needed.
const refreshToken = jwt.sign({ userId: user.id }, process.env.REFRESH_SECRET, { expiresIn: '7d' });
await RefreshToken.create({ token: refreshToken, userId: user.id }); // stored for revocation capability

app.post('/refresh', async (req, res) => {
  const { refreshToken } = req.body;

  const storedToken = await RefreshToken.findOne({ where: { token: refreshToken } });
  if (!storedToken) {
    return res.status(401).json({ error: 'Invalid refresh token' }); // revoked or never existed
  }

  try {
    const decoded = jwt.verify(refreshToken, process.env.REFRESH_SECRET);
    const newAccessToken = jwt.sign({ userId: decoded.userId }, process.env.ACCESS_SECRET, { expiresIn: '15m' });
    res.json({ accessToken: newAccessToken });
  } catch {
    res.status(401).json({ error: 'Refresh token expired' });
  }
});
```

This pattern gives you the best of both worlds: most requests use a fast, stateless, short-lived access token, while the rarely-used refresh flow retains the ability to revoke access by deleting the stored refresh token.

---

## 6. OAuth — briefly, but know the core flow

### Background: what problem does OAuth solve that JWT/sessions don't?

OAuth answers: *"How do I let a user log in using their Google/GitHub account, without my app ever seeing or storing their Google password?"* It's a **delegation protocol** — the user authorizes a third party (Google) to confirm their identity to your app, without your app ever handling their actual credentials.

```
1. Your app redirects the user to Google's login/consent screen
2. User logs into Google (on Google's site, NOT yours) and approves your app's access
3. Google redirects back to YOUR app with an AUTHORIZATION CODE
4. Your server exchanges that code (server-to-server, with your app's secret)
   for an ACCESS TOKEN from Google
5. Your server uses that access token to fetch the user's basic profile
   info (email, name) FROM Google
6. Your server creates/finds a local user record matching that email, and
   issues YOUR OWN session/JWT for that user, exactly as in a normal login
```

**Interview tip:** the key insight to communicate is that OAuth is about **delegated authentication** — your app never sees the user's Google password, only a token Google issues confirming "yes, this person is who they say they are." Libraries like Passport.js (`passport-google-oauth20`) handle the full flow above for you in real projects, but understanding the underlying handshake is what interviewers are actually checking for.

---

## 7. Role-Based Access Control (RBAC)

### Background: why is this a separate topic from authentication?

Once you know WHO a user is (authentication), you still need to decide WHAT they're allowed to do (authorization). RBAC is the most common pattern: assign each user one or more **roles** (e.g., `user`, `editor`, `admin`), and grant permissions based on role rather than checking individual users one by one.

```javascript
// Middleware factory — generates a middleware function customized for
// whichever roles are allowed for a given route. This pattern (a function
// that RETURNS a middleware function) lets you reuse the same logic
// flexibly across many different routes with different allowed roles.
function requireRole(...allowedRoles) {
  return (req, res, next) => {
    // req.user was already attached by the requireAuth middleware (section 4)
    // — this demonstrates middleware CHAINING from Day 8: requireAuth runs
    // first to establish identity, THEN requireRole runs to check permission.
    if (!allowedRoles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
  };
}

app.delete('/users/:id', requireAuth, requireRole('admin'), (req, res) => {
  // Only reachable by authenticated users whose role is 'admin'
  res.json({ message: 'User deleted' });
});

app.put('/articles/:id', requireAuth, requireRole('editor', 'admin'), (req, res) => {
  // Reachable by EITHER editors OR admins
  res.json({ message: 'Article updated' });
});
```

**A more granular alternative worth mentioning if probed further: permission-based access control.** Instead of hardcoding role names into route logic, you assign explicit permissions (e.g., `'delete:users'`, `'edit:articles'`) to roles, and check for the PERMISSION rather than the role name directly. This scales better in systems with many fine-grained actions, since you can change what a role is allowed to do without touching route code — but it's also more complex to set up, and RBAC alone is sufficient for many real applications. Mentioning this distinction, and that the right choice depends on how complex the permission model needs to be, is a good senior-level answer.

---

## 8. How this connects to real backend work (3 YOE framing)

- **"How would you implement 'log out from all devices'?"** → A strong sign of understanding JWT's stateless tradeoff: maintain server-side refresh token revocation (delete all stored refresh tokens for that user), and accept that already-issued short-lived access tokens remain valid until they naturally expire (which is why keeping access token expiry short matters).
- **"Why is it bad practice to put sensitive data in a JWT payload?"** → The payload is only base64-ENCODED, not encrypted — anyone holding the token (including a user inspecting it in browser dev tools) can decode and read it.
- **"A user reports they can still access admin features after you removed their admin role — what's likely happening?"** → Their existing JWT was issued BEFORE the role change and still contains the old role in its payload; it remains valid (and "stale") until it expires, since JWTs aren't re-checked against the database by default — a real, important nuance of stateless tokens.
- **"How would you design access control for an app with both admins and various specialized staff roles with overlapping but distinct permissions?"** → Discuss RBAC vs more granular permission-based access control, and the tradeoff between simplicity and flexibility.

---

## 9. Hands-on practice for today

```javascript
// practice-day12.js
const express = require('express');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcrypt');

const app = express();
app.use(express.json());

const JWT_SECRET = 'practice-secret-change-in-real-app';
const users = []; // in-memory store for practice purposes only

app.post('/register', async (req, res) => {
  const { email, password } = req.body;
  const hashedPassword = await bcrypt.hash(password, 10);
  const user = { id: users.length + 1, email, password: hashedPassword, role: 'user' };
  users.push(user);
  res.status(201).json({ id: user.id, email: user.email });
});

app.post('/login', async (req, res) => {
  const { email, password } = req.body;
  const user = users.find((u) => u.email === email);
  if (!user) return res.status(401).json({ error: 'Invalid credentials' });

  const isMatch = await bcrypt.compare(password, user.password);
  if (!isMatch) return res.status(401).json({ error: 'Invalid credentials' });

  const token = jwt.sign({ userId: user.id, role: user.role }, JWT_SECRET, { expiresIn: '1h' });
  res.json({ token });
});

function requireAuth(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(401).json({ error: 'No token provided' });

  try {
    req.user = jwt.verify(token, JWT_SECRET);
    next();
  } catch {
    res.status(401).json({ error: 'Invalid or expired token' });
  }
}

function requireRole(...roles) {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
  };
}

app.get('/profile', requireAuth, (req, res) => {
  res.json({ userId: req.user.userId, role: req.user.role });
});

app.delete('/admin-only', requireAuth, requireRole('admin'), (req, res) => {
  res.json({ message: 'Deleted (you are an admin)' });
});

app.listen(3000, () => console.log('Running on http://localhost:3000'));
```

Test the full flow with curl: register → login (copy the token) → call `/profile` with the token → try `/admin-only` and confirm you get 403 (since this test user's role is `'user'`, not `'admin'`).

---

## 10. Interview Q&A Recap (say these out loud)

1. **Q: What's the difference between authentication and authorization?**
   A: Authentication verifies WHO you are; authorization determines WHAT you're allowed to do once your identity is known. A 401 status code maps to an authentication failure; a 403 maps to an authorization failure.

2. **Q: Why use bcrypt instead of a fast hash like SHA-256 for passwords?**
   A: Fast hashing algorithms let attackers brute-force guess passwords extremely quickly against a stolen hash database. bcrypt is deliberately slow, with a tunable cost factor that can be increased as hardware improves, keeping brute-force attacks computationally expensive.

3. **Q: Is a JWT encrypted?**
   A: No — the header and payload are only base64-encoded, fully readable by anyone holding the token. Only the signature provides security, by making any tampering with the header/payload detectable (the recomputed signature won't match).

4. **Q: What's the core tradeoff between sessions and JWTs?**
   A: Sessions are stateful (server stores session data, easy to revoke instantly, needs a shared store like Redis to scale across servers) versus JWTs being stateless (no server storage needed, scales naturally across servers, but hard to revoke before natural expiry).

5. **Q: Why use both an access token and a refresh token instead of just one long-lived JWT?**
   A: Short-lived access tokens limit the damage window if stolen, while a server-side-stored, revocable refresh token allows the system to retain the ability to log a user out immediately when needed — combining JWT's stateless performance benefits with practical revocation capability.

6. **Q: What does OAuth actually delegate, and what does your app never see?**
   A: OAuth delegates identity verification to a third party (e.g., Google) — your app receives a token proving the user's identity, but never sees or stores the user's actual password for that third-party account.

---

## Tomorrow (Day 13) Preview
We cover **centralized error handling, logging, environment configuration, and core security practices**: structured error classes, `winston`/`pino` logging, `dotenv` and config management, plus `helmet`, CORS, and input sanitization — the operational hygiene topics that every production Node app needs and that interviewers use to gauge real-world readiness.
