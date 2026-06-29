# OAuth 2.0

## What is OAuth 2.0?

OAuth 2.0 is an **authorization framework** that allows a third-party application to access a user's resources on another service — without the user sharing their password with the third party.

**Real example:**

```
"Login with Google" on a food delivery app:

WITHOUT OAuth:
User gives food app their Google email + password
-> Food app now has your Google credentials — dangerous!

WITH OAuth:
User clicks "Login with Google"
-> Google asks: "Do you allow FoodApp to see your name and email?"
-> User says YES
-> Google gives FoodApp a token (not the password)
-> FoodApp uses token to get name + email from Google
-> User's Google password never leaves Google
```

## OAuth 2.0 vs Authentication vs Authorization

| Concept | Meaning |
|---|---|
| Authentication | WHO are you? (Login — proving identity) |
| Authorization | WHAT can you do? (Permissions — what you're allowed to access) |
| OAuth 2.0 | A framework for AUTHORIZATION — granting limited access to resources |

OAuth 2.0 is often used for authentication too (via OpenID Connect on top of it), but the protocol itself is about delegated authorization.

## Core Roles

| Role | Description | Example |
|---|---|---|
| **Resource Owner** | The user who owns the data | You (the Gmail user) |
| **Client** | The app requesting access | FoodApp |
| **Authorization Server** | Issues tokens after user consent | Google's auth server |
| **Resource Server** | Holds the protected resources | Google's API (Gmail, Profile) |

## Grant Types (Flows)

OAuth 2.0 has multiple flows for different scenarios.

### 1. Authorization Code Flow (Most Secure — Standard for Web Apps)

Used when the client is a web app with a backend server. The authorization code is exchanged for tokens server-side (tokens never exposed in browser URL).

**Step-by-step:**

```
Step 1: Client redirects user to Authorization Server
GET https://accounts.google.com/o/oauth2/auth
  ?response_type=code
  &client_id=YOUR_CLIENT_ID
  &redirect_uri=https://foodapp.com/callback
  &scope=openid email profile
  &state=random_csrf_token

Step 2: User logs in and grants consent on Google's page

Step 3: Google redirects back to your callback with an authorization code
GET https://foodapp.com/callback
  ?code=AUTHORIZATION_CODE
  &state=random_csrf_token

Step 4: Your backend server exchanges code for tokens (server-to-server, never in browser)
POST https://oauth2.googleapis.com/token
  grant_type=authorization_code
  &code=AUTHORIZATION_CODE
  &redirect_uri=https://foodapp.com/callback
  &client_id=YOUR_CLIENT_ID
  &client_secret=YOUR_CLIENT_SECRET

Response:
{
  "access_token": "ya29.xxx",
  "refresh_token": "1//xxx",
  "expires_in": 3600,
  "token_type": "Bearer",
  "id_token": "eyJhbGci..."  // OpenID Connect identity token
}

Step 5: Use access token to call Google APIs
GET https://www.googleapis.com/oauth2/v2/userinfo
Authorization: Bearer ya29.xxx

Response:
{
  "id": "123456789",
  "email": "user@gmail.com",
  "name": "John Doe",
  "picture": "https://..."
}
```

**Why a code instead of tokens directly?**

The authorization code appears in the browser URL (redirect) — if tokens were returned directly in the URL, they'd be visible in browser history, server logs, and referrer headers. The code is short-lived (60 seconds) and useless without the `client_secret`, which only lives on the server.

### 2. Authorization Code Flow with PKCE (For SPAs and Mobile Apps)

SPAs (React, Angular) and mobile apps can't safely store a `client_secret` (the source code is publicly visible). PKCE (Proof Key for Code Exchange) solves this without a client secret.

```
Client generates:
  code_verifier = random 43-128 char string (e.g. "dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk")
  code_challenge = BASE64URL(SHA256(code_verifier)) = "E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM"

Step 1: Authorization request includes code_challenge
GET /authorize
  ?response_type=code
  &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
  &code_challenge_method=S256

Step 4: Token exchange includes code_verifier (proves it's the same client)
POST /token
  grant_type=authorization_code
  &code=AUTHORIZATION_CODE
  &code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
  (no client_secret needed)

Auth server verifies: SHA256(code_verifier) == code_challenge -> issues tokens
```

An attacker who intercepts the code can't exchange it without the `code_verifier`, which never leaves the client.

### 3. Client Credentials Flow (Machine-to-Machine)

No user involved — a backend service authenticates directly as itself to access another service's API. No consent screen.

```
POST /token
  grant_type=client_credentials
  &client_id=SERVICE_CLIENT_ID
  &client_secret=SERVICE_CLIENT_SECRET
  &scope=read:orders

Response:
{
  "access_token": "xxx",
  "expires_in": 3600,
  "token_type": "Bearer"
}
```

Used for: microservice-to-microservice auth, cron jobs calling APIs, server-side integrations.

### 4. Refresh Token Flow

When an access token expires, use the refresh token to get a new one without user re-login.

```
POST /token
  grant_type=refresh_token
  &refresh_token=REFRESH_TOKEN
  &client_id=CLIENT_ID
  &client_secret=CLIENT_SECRET

Response:
{
  "access_token": "new_access_token",
  "refresh_token": "new_refresh_token",  // rotation
  "expires_in": 3600
}
```

### 5. Implicit Flow (Deprecated)

Tokens returned directly in the URL fragment — no backend needed. Security issues (tokens in URL) led to deprecation. SPAs should use Authorization Code + PKCE instead.

### Grant Type Summary

| Grant Type | Who Uses It | User Interaction |
|---|---|---|
| Authorization Code | Web apps with backend | Yes — consent screen |
| Authorization Code + PKCE | SPAs, Mobile apps | Yes — consent screen |
| Client Credentials | Server-to-server | No user |
| Refresh Token | Any client | No (silent refresh) |
| Implicit | Deprecated | N/A |

## OpenID Connect (OIDC)

OAuth 2.0 is about authorization (access to resources). OpenID Connect is a thin identity layer on top of OAuth 2.0 that adds authentication (who the user is).

```
OAuth 2.0:  "Here's a token to access the user's Google Drive"
OIDC:       "Here's a token that tells you WHO the user is (email, name, etc.)"
```

OIDC adds an **ID Token** — a JWT containing user identity claims:

```json
{
  "iss": "https://accounts.google.com",
  "sub": "1234567890",
  "aud": "YOUR_CLIENT_ID",
  "exp": 1735689600,
  "iat": 1735686000,
  "email": "user@gmail.com",
  "name": "John Doe",
  "picture": "https://..."
}
```

When you implement "Login with Google/GitHub/Facebook", you're using OIDC on top of OAuth 2.0.

## Scopes

Scopes define what the access token is allowed to do — they're the permissions the user grants.

```
scope=openid email profile
  openid  -> OIDC (get ID token)
  email   -> read user's email
  profile -> read name, picture, etc.

scope=read:repos write:repos
  -> GitHub: read and write repositories

scope=https://www.googleapis.com/auth/gmail.readonly
  -> Read Gmail messages only
```

The principle of least privilege: request only the scopes your app actually needs.

## Spring Boot OAuth 2.0 (Social Login)

### Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

### Configuration (application.yml)

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: YOUR_GOOGLE_CLIENT_ID
            client-secret: YOUR_GOOGLE_CLIENT_SECRET
            scope: openid, email, profile
            redirect-uri: "{baseUrl}/login/oauth2/code/google"

          github:
            client-id: YOUR_GITHUB_CLIENT_ID
            client-secret: YOUR_GITHUB_CLIENT_SECRET
            scope: read:user, user:email
```

### Security Configuration

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Autowired
    private CustomOAuth2UserService oAuth2UserService;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/login", "/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .userInfoEndpoint(userInfo -> userInfo
                    .userService(oAuth2UserService)
                )
                .successHandler(oAuth2AuthenticationSuccessHandler())
                .failureHandler(oAuth2AuthenticationFailureHandler())
            );
        return http.build();
    }
}
```

### Custom OAuth2 User Service (Save/Update User on Login)

```java
@Service
public class CustomOAuth2UserService extends DefaultOAuth2UserService {

    @Autowired
    private UserRepository userRepo;

    @Override
    public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {

        OAuth2User oAuth2User = super.loadUser(userRequest);

        String provider = userRequest.getClientRegistration().getRegistrationId(); // "google"
        String providerId = oAuth2User.getAttribute("sub");   // Google user ID
        String email = oAuth2User.getAttribute("email");
        String name = oAuth2User.getAttribute("name");

        // Save or update user in your DB
        User user = userRepo.findByProviderAndProviderId(provider, providerId)
                .orElseGet(() -> {
                    User newUser = new User();
                    newUser.setProvider(provider);
                    newUser.setProviderId(providerId);
                    newUser.setEmail(email);
                    newUser.setName(name);
                    return newUser;
                });

        user.setName(name);  // update name on every login
        userRepo.save(user);

        return oAuth2User;
    }
}
```

### Success Handler (Issue JWT after OAuth login)

```java
@Component
public class OAuth2AuthenticationSuccessHandler
        extends SimpleUrlAuthenticationSuccessHandler {

    @Autowired
    private JwtService jwtService;

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request,
                                        HttpServletResponse response,
                                        Authentication authentication) throws IOException {

        OAuth2User oAuth2User = (OAuth2User) authentication.getPrincipal();
        String email = oAuth2User.getAttribute("email");

        // Issue your own JWT
        String jwt = jwtService.generateAccessToken(email);

        // Redirect to frontend with token (or set cookie)
        String redirectUrl = "https://yourapp.com/oauth2/redirect?token=" + jwt;
        getRedirectStrategy().sendRedirect(request, response, redirectUrl);
    }
}
```

## Node.js OAuth 2.0 (Passport.js)

```javascript
const passport = require('passport');
const GoogleStrategy = require('passport-google-oauth20').Strategy;

passport.use(new GoogleStrategy({
    clientID: process.env.GOOGLE_CLIENT_ID,
    clientSecret: process.env.GOOGLE_CLIENT_SECRET,
    callbackURL: '/auth/google/callback'
}, async (accessToken, refreshToken, profile, done) => {
    try {
        let user = await User.findOne({ googleId: profile.id });
        if (!user) {
            user = await User.create({
                googleId: profile.id,
                email: profile.emails[0].value,
                name: profile.displayName
            });
        }
        return done(null, user);
    } catch (err) {
        return done(err);
    }
}));

// Routes
app.get('/auth/google',
    passport.authenticate('google', { scope: ['openid', 'email', 'profile'] })
);

app.get('/auth/google/callback',
    passport.authenticate('google', { session: false }),
    (req, res) => {
        // Issue JWT
        const token = jwt.sign({ userId: req.user.id }, process.env.JWT_SECRET);
        res.redirect(`/dashboard?token=${token}`);
    }
);
```

## The `state` Parameter (CSRF Protection)

The `state` parameter in the OAuth flow is a random token generated by your server and verified when Google redirects back. It prevents CSRF attacks on the OAuth callback.

```
1. Your server generates: state = random_string
2. Store state in session/cookie
3. Include state in /authorize redirect
4. When Google redirects back with code + state
5. Verify: received state == stored state
6. If mismatch -> CSRF attack, reject
```

## Security Checklist for OAuth Implementation

- Always verify the `state` parameter to prevent CSRF
- Use HTTPS everywhere — tokens in URLs/headers must be encrypted in transit
- Validate `redirect_uri` exactly (whitelist specific URIs, not wildcards)
- Use short-lived access tokens (1 hour) + refresh token rotation
- Never log access tokens or authorization codes
- Use PKCE for SPAs and mobile apps (no client_secret)
- Validate the `aud` claim in ID tokens to prevent token substitution attacks

## Interview Questions

**Q1: What is the difference between OAuth 2.0 and OpenID Connect?**

OAuth 2.0 is an authorization framework — it grants a client limited access to a user's resources (e.g. "read their Google Drive"). OpenID Connect is an identity layer on top of OAuth 2.0 — it tells your app who the user is (email, name, user ID) via an ID Token. "Login with Google" uses OIDC; "Allow this app to post tweets on your behalf" uses OAuth 2.0.

**Q2: Why is the Authorization Code flow more secure than Implicit?**

In Authorization Code flow, the authorization code (useless alone) appears in the URL, but the actual tokens are exchanged server-side in an HTTPS POST — they never appear in the browser URL, history, or server logs. In Implicit flow, tokens were returned directly in the URL fragment — visible in browser history, server logs, and referrer headers. Implicit is now deprecated.

**Q3: What is PKCE and why is it needed?**

PKCE (Proof Key for Code Exchange) is an extension to the Authorization Code flow for clients that can't safely store a `client_secret` (SPAs, mobile apps). The client generates a random `code_verifier` and sends a hash (`code_challenge`) with the auth request. When exchanging the code for tokens, it sends the original `code_verifier`. The auth server verifies the hash — proving the same client that initiated the flow is completing it.

**Q4: What is the Client Credentials flow used for?**

Machine-to-machine authorization where no user is involved — one backend service authenticates to another. The service directly exchanges its `client_id` and `client_secret` for an access token. Common in microservices (Service A calls Service B's API), scheduled jobs, and server-side integrations with third-party APIs.

**Q5: What does the `state` parameter do in OAuth?**

It's a CSRF protection mechanism. Your app generates a random value, stores it in the session/cookie, and includes it in the authorization redirect. When the auth server redirects back with the code, it echoes the state back. Your app verifies it matches — if not, the callback is rejected as a potential CSRF attack.
