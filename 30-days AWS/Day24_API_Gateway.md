# Day 24: API Gateway
### *(The Front Door to Your Serverless API — Turning Lambda Functions Into a Real HTTP API)*

> **Roadmap reference:** Week 4, Day 24 — "API Gateway"

---

## Why This Matters

Right now your Lambda functions are invoked directly from the AWS console or other AWS services. But real users and frontend apps communicate over **HTTP** — they need a URL like `https://api.yourapp.com/users`, not an AWS Lambda ARN.

**API Gateway is the bridge between the HTTP world and your Lambda functions.** It receives HTTP requests, routes them to the right Lambda, and returns the response — all without any server you manage. Together, API Gateway + Lambda is the most popular serverless backend architecture in AWS.

---

## 1. What is API Gateway?

**Amazon API Gateway is a fully managed service for creating, publishing, securing, and monitoring HTTP/REST/WebSocket APIs at any scale — acting as the "front door" to your backend services.**

```
WITHOUT API Gateway:
  Lambda functions have no HTTP endpoint
  → Can only be invoked by AWS services or directly via SDK
  → Not accessible to browsers, mobile apps, or external clients

WITH API Gateway:
  Client sends: GET https://abc123.execute-api.ap-south-1.amazonaws.com/users
          ↓
  API Gateway receives the HTTP request
          ↓
  Routes it to the correct Lambda function
          ↓
  Lambda executes, returns a result
          ↓
  API Gateway formats it as an HTTP response
          ↓
  Client receives: HTTP 200 + JSON body
```

---

## 2. Three Types of API Gateway APIs

```
┌─────────────────────────────────────────────────────────────┐
│  HTTP API (v2)          ← USE THIS for most Node.js backends │
│  → Simpler, faster, cheaper than REST API                    │
│  → Supports Lambda, HTTP backends                            │
│  → Best for: standard REST APIs, microservices               │
├─────────────────────────────────────────────────────────────┤
│  REST API (v1)                                               │
│  → More features: request/response transformation,           │
│    caching, API keys, usage plans                            │
│  → More expensive and complex to configure                   │
│  → Best for: enterprise APIs needing advanced features       │
├─────────────────────────────────────────────────────────────┤
│  WebSocket API                                               │
│  → Persistent two-way connections                            │
│  → Best for: chat apps, live dashboards, real-time games     │
└─────────────────────────────────────────────────────────────┘
```

> For a standard Node.js backend REST API, always start with **HTTP API** — it's simpler, 70% cheaper than REST API, and handles 99% of use cases. Use REST API only when you specifically need its advanced features (response caching, request transformation, usage plans).

---

## 3. Core API Gateway Concepts

```
API
  → The top-level container (e.g., "my-app-api")

ROUTE
  → A combination of HTTP method + path that maps to a backend
  → e.g., GET /users → Lambda:get-users-function
  → e.g., POST /users → Lambda:create-user-function
  → e.g., ANY /{proxy+} → Lambda:express-app (proxy integration)

INTEGRATION
  → Defines WHAT handles the route
  → Lambda function, HTTP endpoint, AWS service, or mock response

STAGE
  → A named deployment of your API (e.g., "dev", "staging", "prod")
  → Each stage gets its own URL
  → e.g., https://abc.execute-api.region.amazonaws.com/prod/users

DEPLOYMENT
  → A snapshot of your API at a point in time
  → Must deploy to a stage for changes to take effect
```

---

## 4. The Lambda Event Object from API Gateway

When API Gateway triggers a Lambda, the `event` object looks very different from a direct invocation — it contains the full HTTP request details:

```javascript
// What event looks like when triggered from HTTP API (v2):
{
  "version": "2.0",
  "routeKey": "GET /users",
  "rawPath": "/users",
  "rawQueryString": "page=1&limit=10",
  "headers": {
    "content-type": "application/json",
    "authorization": "Bearer eyJhbGci..."
  },
  "queryStringParameters": {
    "page": "1",
    "limit": "10"
  },
  "pathParameters": {
    "id": "42"          // if route is GET /users/{id}
  },
  "requestContext": {
    "http": {
      "method": "GET",
      "path": "/users",
      "sourceIp": "203.0.113.42"
    }
  },
  "body": "{\"name\":\"Ashish\"}",  // for POST/PUT (always a STRING)
  "isBase64Encoded": false
}
```

```javascript
// What your Lambda must RETURN for API Gateway:
{
  "statusCode": 200,
  "headers": {
    "Content-Type": "application/json",
    "Access-Control-Allow-Origin": "*"    // for CORS
  },
  "body": "{\"users\":[...]}"             // MUST be a string, not an object
}
```

> **The two most common mistakes with Lambda + API Gateway:**
> 1. Forgetting to `JSON.stringify()` the body — API Gateway requires a string, not an object
> 2. Forgetting to handle CORS headers — browsers will block responses without `Access-Control-Allow-Origin`

---

## 5. Building a Full CRUD API — Lambda + API Gateway

### Step 1: Create Lambda Functions for Each Route

```javascript
// users-handler.js — One Lambda handling all /users routes

exports.handler = async (event) => {
  console.log('Event:', JSON.stringify(event, null, 2));

  const method = event.requestContext.http.method;
  const path   = event.rawPath;

  try {
    // Route based on method + path
    if (method === 'GET' && path === '/users') {
      return await getUsers(event);
    }
    else if (method === 'GET' && path.startsWith('/users/')) {
      return await getUserById(event);
    }
    else if (method === 'POST' && path === '/users') {
      return await createUser(event);
    }
    else if (method === 'PUT' && path.startsWith('/users/')) {
      return await updateUser(event);
    }
    else if (method === 'DELETE' && path.startsWith('/users/')) {
      return await deleteUser(event);
    }
    else {
      return response(404, { error: 'Route not found' });
    }
  } catch (err) {
    console.error('Handler error:', err);
    return response(500, { error: 'Internal server error' });
  }
};

// ── Route handlers ────────────────────────────────────────────

async function getUsers(event) {
  const page  = parseInt(event.queryStringParameters?.page)  || 1;
  const limit = parseInt(event.queryStringParameters?.limit) || 10;

  // In a real function: const result = await pool.query('SELECT * FROM users...')
  const users = [
    { id: 1, name: 'Ashish', email: 'ashish@example.com' },
    { id: 2, name: 'Rahul',  email: 'rahul@example.com'  }
  ];

  return response(200, {
    users,
    page,
    limit,
    total: users.length
  });
}

async function getUserById(event) {
  const id = event.pathParameters?.id;
  if (!id) return response(400, { error: 'User ID required' });

  // In a real function: const result = await pool.query('SELECT * FROM users WHERE id=$1', [id])
  return response(200, { id: parseInt(id), name: 'Ashish', email: 'ashish@example.com' });
}

async function createUser(event) {
  const body = JSON.parse(event.body || '{}');  // body arrives as string — parse it
  const { name, email } = body;

  if (!name || !email) {
    return response(400, { error: 'name and email are required' });
  }

  // In a real function: await pool.query('INSERT INTO users...')
  return response(201, { id: 3, name, email, createdAt: new Date().toISOString() });
}

async function updateUser(event) {
  const id   = event.pathParameters?.id;
  const body = JSON.parse(event.body || '{}');
  return response(200, { id: parseInt(id), ...body, updatedAt: new Date().toISOString() });
}

async function deleteUser(event) {
  const id = event.pathParameters?.id;
  return response(200, { message: `User ${id} deleted` });
}

// ── Response helper ───────────────────────────────────────────
function response(statusCode, body) {
  return {
    statusCode,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*',     // CORS — allows browser requests
      'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type, Authorization'
    },
    body: JSON.stringify(body)                // MUST be a string
  };
}
```

### Step 2: Create the HTTP API

```
AWS Console → API Gateway → Create API
  → HTTP API → Build

  Integrations:
    → Add integration → Lambda
    → Lambda function: users-handler
    → Version: 2.0

  Configure routes:
    Method: ANY       Route: /users        Integration: users-handler
    Method: ANY       Route: /users/{id}   Integration: users-handler
    (using ANY covers GET/POST/PUT/DELETE with one integration)

  Define stages:
    → Stage name: prod
    → Auto-deploy: enabled

  → Create
```

```
After creation, copy your API URL:
  https://abc123xyz.execute-api.ap-south-1.amazonaws.com/
```

### Step 3: Test Every Route

```bash
BASE_URL="https://abc123xyz.execute-api.ap-south-1.amazonaws.com"

# GET all users
curl $BASE_URL/users

# GET all users with pagination
curl "$BASE_URL/users?page=1&limit=5"

# GET one user
curl $BASE_URL/users/42

# CREATE a user
curl -X POST $BASE_URL/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Ashish","email":"ashish@example.com"}'

# UPDATE a user
curl -X PUT $BASE_URL/users/42 \
  -H "Content-Type: application/json" \
  -d '{"name":"Ashish Anand"}'

# DELETE a user
curl -X DELETE $BASE_URL/users/42
```

---

## 6. Authorization — Securing Your API

```
Three ways to secure an API Gateway endpoint:

1. NO AUTH (default)
   → Anyone with the URL can call it
   → Fine only for public endpoints

2. IAM AUTH
   → Callers must sign requests with AWS credentials
   → Good for internal AWS-to-AWS communication

3. JWT AUTHORIZER  ← most common for real user-facing APIs
   → Caller provides a JWT token (e.g., from Cognito or Auth0)
   → API Gateway validates the token before invoking Lambda
   → Invalid/missing token → 401 Unauthorized without Lambda running
```

### Adding a JWT Authorizer

```
API Gateway → your API → Authorization
  → Create and attach authorizer
  → Authorizer type: JWT
  → Identity source: $request.header.Authorization
  → Issuer URL: https://cognito-idp.ap-south-1.amazonaws.com/your-pool-id
  → Audience: your-app-client-id
```

```
Flow with JWT auth:
  Client sends: GET /users
                Authorization: Bearer eyJhbGci...
          ↓
  API Gateway validates JWT signature + expiry
          ↓
  Valid token → Lambda invoked with decoded claims in event.requestContext
  Invalid token → 401 returned immediately, Lambda never fires
```

---

## 7. Custom Domain Names — Replacing the Default URL

The default API Gateway URL (`abc123xyz.execute-api.ap-south-1.amazonaws.com`) is not suitable for production. You'd replace it with your own domain:

```
Default:   https://abc123xyz.execute-api.ap-south-1.amazonaws.com/users
Custom:    https://api.pulsebloom.com/users

Steps:
  1. Register domain in Route 53 (Day 29)
  2. Create an ACM (Certificate Manager) SSL certificate for api.pulsebloom.com
  3. API Gateway → Custom domain names → Create
     → Domain name: api.pulsebloom.com
     → TLS certificate: your ACM cert
  4. API mapping: map "/" to your prod stage
  5. Route 53: create an A record (alias) pointing to the
     API Gateway domain name
```

---

## 8. API Gateway vs ALB — When to Use Which

```
USE API GATEWAY when:
  ✓ Serverless architecture (Lambda backend)
  ✓ Need built-in auth (JWT, IAM, Cognito)
  ✓ Need rate limiting / throttling per client
  ✓ Multiple separate microservices or Lambda functions
  ✓ Event-driven, variable traffic

USE ALB when:
  ✓ EC2 or ECS container backend
  ✓ WebSocket connections (ALB supports this too)
  ✓ Already have VPC-based infrastructure
  ✓ Steady, high-volume HTTP traffic
  ✓ Need connection draining, sticky sessions
```

---

## 9. Rate Limiting and Throttling

API Gateway includes built-in rate limiting — protecting your Lambda functions from abuse:

```
Default limits (per account, per region):
  10,000 requests per second steady state
  5,000 requests burst

Configure per-stage limits:
  API Gateway → your API → Stages → prod
    → Default route throttling:
        Rate:  1000 req/second
        Burst: 500

Configure per-route limits (e.g., limit login endpoint):
  Route throttling on POST /login:
    Rate:  10 req/second  ← prevent brute-force attacks
    Burst: 5
```

---

## 10. The Full Serverless Architecture

```
                    Internet
                        │
                        │  HTTPS requests
                        ▼
              ┌────────────────────┐
              │   API Gateway        │  ← managed, auto-scaling,
              │   (HTTP API)          │    no servers to manage
              │   Routes:             │
              │   GET  /users    ──┐  │
              │   POST /users    ──┤  │
              │   GET  /users/id ──┤  │
              └────────────────────┘  │
                                       │
              ┌────────────────────────┘
              │   Lambda Functions
              │   (users-handler)
              │   → scales 0 to thousands
              │     of concurrent executions
              └───────────┬────────────
                           │
              ┌────────────▼────────────┐
              │   DynamoDB / RDS          │  ← data layer
              └───────────────────────────┘
```

> This complete stack — API Gateway + Lambda + DynamoDB — is **Portfolio Project 3's foundation** (Serverless URL Shortener, which you'll build tomorrow on Day 25).

---

## 11. Interview Questions (Practice Explaining These Out Loud)

**Q1: What is API Gateway and why is it needed for Lambda?**
> API Gateway is a managed service that creates HTTP endpoints and routes requests to Lambda functions. Lambda by itself has no HTTP interface — API Gateway bridges the gap, turning functions into a callable REST API accessible to browsers, mobile apps, and external services.

**Q2: What is the difference between HTTP API and REST API in API Gateway?**
> HTTP API is simpler, faster, and ~70% cheaper — ideal for standard Lambda-backed REST APIs. REST API has more advanced features like request/response transformation, caching, and usage plans, at higher cost and complexity. Start with HTTP API and only upgrade if specific REST API features are needed.

**Q3: Why must the Lambda response body be a string, not an object?**
> API Gateway expects a specific response format where body must be a serialized string. Returning a JavaScript object instead of `JSON.stringify(object)` causes API Gateway to return an empty or malformed response body to the client.

**Q4: What is a Stage in API Gateway?**
> A named deployment environment (e.g., dev, staging, prod) — each with its own URL and configuration. Changes to an API must be deployed to a stage to take effect. Different stages can have different throttling limits, logging levels, and environment variables.

**Q5: How would you protect an API Gateway endpoint from unauthorized access?**
> Add a JWT Authorizer — API Gateway validates the token from the Authorization header before invoking Lambda. Invalid or missing tokens return 401 immediately. Alternatively, use IAM authorization for internal AWS service-to-service calls.

**Q6: When would you choose ALB over API Gateway?**
> When your backend is EC2 or ECS containers (not Lambda), you need sticky sessions, or you're already invested in VPC-based infrastructure with steady high-volume traffic. ALB is lower-cost for always-on servers with consistent traffic, while API Gateway excels for Lambda-based and variable workloads.

---

## 12. Hands-On Assignment for Today

1. Create the **users-handler Lambda** function with the code from Section 5.

2. Create an **HTTP API** in API Gateway with:
   - Two routes: `ANY /users` and `ANY /users/{id}`
   - Both integrated with `users-handler`
   - Auto-deploy enabled on `prod` stage

3. **Test all 5 operations** using curl commands from Section 5 — confirm each returns the expected status code and body.

4. Add **rate limiting** on the `POST /users` route: 10 requests/second.

5. Check **CloudWatch Logs** for your Lambda — confirm request events are being logged with full request details.

6. **Bonus:** Add a `GET /health` route that returns `{"status":"healthy"}` without invoking Lambda — use the API Gateway **mock integration** (returns a fixed response without any backend):
   ```
   Route: GET /health
   Integration type: Mock
   Response: {"status":"healthy"}
   ```

7. Write a short note covering:
   - Why body must be JSON.stringify'd in the Lambda response
   - The difference between a Route and an Integration
   - One real scenario where you'd use JWT authorization

---

## 13. Day 24 Goal — Can You Teach This?

By the end of today, you should be able to explain, without notes:

> *"API Gateway creates HTTP endpoints that route requests to Lambda functions. I define Routes (method + path), integrate them with Lambda functions, and deploy to a Stage to get a live URL. My Lambda must return a properly formatted response with a stringified body and CORS headers. For security, I add a JWT Authorizer that validates tokens before Lambda ever runs. API Gateway + Lambda gives me a fully serverless API that scales to zero cost when idle and to thousands of concurrent requests during peaks — with no servers to manage."*

---

**Next up — Day 25: Lambda + API Gateway Project (Portfolio Project 3: Serverless URL Shortener)**
