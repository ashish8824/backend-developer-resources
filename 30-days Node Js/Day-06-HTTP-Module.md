# Day 6 — The Core `http` Module: Building a Server Without a Framework

> **Why this day matters:** Almost every Node backend engineer learns Express first and never looks underneath it. That's fine for shipping features, but it's a real liability in interviews — a common senior-leaning question is *"What does Express actually do for you that raw Node doesn't?"* You can only answer that well if you've built something with the raw `http` module yourself. Today closes that gap, and it'll make Day 8 (Express) feel like "oh, this is just a nicer wrapper around what I already understand" instead of new magic.

---

## 1. Background: what is "the server" really doing, at the lowest level?

Before touching code, let's build the mental model.

When your browser (or Postman, or another service) makes a request to `http://yourapi.com/users/5`, here's what's actually happening underneath all the abstractions:

1. A **TCP connection** is established between the client and your server's machine, on a specific **port** (e.g., port 3000, or port 80/443 in production behind a reverse proxy — Week 4 topic).
2. The client sends an **HTTP request** over that connection — which is really just a structured chunk of **text** (the request line, headers, and optionally a body), following the HTTP protocol's rules.
3. Your Node.js process, which has been "listening" on that port, receives this raw data.
4. Node parses that raw text into a usable JavaScript object (the `request` object) and gives your code a chance to react to it.
5. Your code decides what to send back, and Node serializes your response into the correctly-formatted HTTP response text, which gets sent back over the same TCP connection.
6. The connection is then closed (or kept alive for reuse, depending on headers — more on this below).

**The `http` module is the piece of Node that handles steps 3-5** — parsing raw incoming bytes into a friendly `req` object, and giving you a `res` object whose methods (`res.write()`, `res.end()`) get translated back into correctly-formatted raw HTTP response bytes. Express, when we get there, is built entirely on top of this module — it doesn't replace it, it just adds routing, middleware, and convenience methods around the same `req`/`res` objects.

---

## 2. Creating Your First Raw HTTP Server

```javascript
const http = require('http');

// http.createServer() takes a callback function — this function will be
// called by Node EVERY TIME a new HTTP request comes in. This is the
// EventEmitter pattern from Day 4 in disguise! Under the hood, the server
// object emits a 'request' event for every incoming request, and passing
// a callback here is just shorthand for server.on('request', callback).
const server = http.createServer((req, res) => {
  // 'req' = the incoming request object (what the client sent)
  // 'res' = the outgoing response object (what you'll send back)

  res.statusCode = 200;                          // HTTP status code
  res.setHeader('Content-Type', 'text/plain');   // tells the client what kind of data is coming
  res.end('Hello from raw Node.js!');             // sends the response body AND closes it
});

// .listen() starts the server, binding it to a port and beginning to
// accept incoming TCP connections on that port.
server.listen(3000, () => {
  console.log('Server running on http://localhost:3000');
});
```

Run this with `node server.js`, then visit `http://localhost:3000` in a browser or hit it with `curl http://localhost:3000` — you'll see "Hello from raw Node.js!" with no framework involved at all.

**Interview tip:** if asked "is `http.createServer()`'s callback synchronous or does it use the event loop," the precise answer is: it's registered as a listener for the server's internal `'request'` event (EventEmitter pattern), and gets invoked by the event loop's poll phase whenever a new request's data is ready — tying directly back to Day 2 and Day 4 concepts.

---

## 3. The Request Object (`req`) — what's actually inside it

```javascript
const server = http.createServer((req, res) => {
  console.log(req.method);   // 'GET', 'POST', 'PUT', 'DELETE', etc.
  console.log(req.url);      // the path + query string, e.g. '/users/5?active=true'
  console.log(req.headers);  // an object of all request headers (lowercased keys)
  console.log(req.headers['content-type']);
  console.log(req.headers['authorization']);

  res.end('Logged your request details to the console');
});
```

### Parsing the URL properly (interview-relevant: raw `req.url` is just a string!)

```javascript
const http = require('http');
const { URL } = require('url'); // Node's built-in URL parser

const server = http.createServer((req, res) => {
  // req.url is JUST A STRING like '/search?q=node&page=2' — Node does NOT
  // automatically parse query parameters for you. This is one of the
  // first things a framework like Express does FOR you that you'd
  // otherwise have to do manually.
  const fullUrl = new URL(req.url, `http://${req.headers.host}`);

  console.log(fullUrl.pathname);              // '/search'
  console.log(fullUrl.searchParams.get('q'));  // 'node'
  console.log(fullUrl.searchParams.get('page')); // '2'

  res.end(`You searched for: ${fullUrl.searchParams.get('q')}`);
});
```

### Reading the Request Body — THE part that surprises people coming from Express

**Critical background point:** `req` is a **Readable stream** (Day 5 connects directly here!). This means the request body does **NOT** arrive all at once as a ready-to-use object — it arrives in **chunks**, the same way any stream does. In Express, you're used to just accessing `req.body` directly — that convenience is something Express's `express.json()` middleware does FOR you, by manually doing exactly what's shown below, then attaching the parsed result to `req.body`.

```javascript
const server = http.createServer((req, res) => {
  if (req.method === 'POST' && req.url === '/data') {
    let body = [];

    // 'data' event fires once per chunk received (Day 5 stream concept)
    req.on('data', (chunk) => {
      body.push(chunk);
    });

    // 'end' event fires once the ENTIRE body has been received
    req.on('end', () => {
      const fullBody = Buffer.concat(body).toString(); // join all chunks, convert to string
      console.log('Raw body:', fullBody);

      try {
        const parsedData = JSON.parse(fullBody); // manually parse JSON ourselves
        res.statusCode = 200;
        res.setHeader('Content-Type', 'application/json');
        res.end(JSON.stringify({ message: 'Received', data: parsedData }));
      } catch (err) {
        res.statusCode = 400;
        res.end(JSON.stringify({ error: 'Invalid JSON' }));
      }
    });

    req.on('error', (err) => {
      res.statusCode = 500;
      res.end('Server error reading request');
    });
  } else {
    res.statusCode = 404;
    res.end('Not found');
  }
});
```

**Now you can answer with real depth:** *"What does `express.json()` middleware actually do?"* → It listens for `'data'` and `'end'` events on the request stream (exactly as shown above), accumulates the chunks into a Buffer, parses it as JSON once complete, and attaches the result to `req.body` — all before your route handler ever runs.

---

## 4. The Response Object (`res`) — methods you'll use constantly

```javascript
const server = http.createServer((req, res) => {
  // Setting status code — two equivalent ways
  res.statusCode = 201;
  // OR: res.writeHead(201, { 'Content-Type': 'application/json' });

  // Setting headers — must happen BEFORE res.end() or res.write() is called,
  // because once you start sending the body, the headers have already
  // been sent over the wire and can no longer be changed.
  res.setHeader('Content-Type', 'application/json');
  res.setHeader('X-Powered-By', 'Raw-Node');

  // res.write() can be called MULTIPLE times to send the body in pieces
  // (remember: res is a WRITABLE stream — Day 5!)
  res.write(JSON.stringify({ part: 1 }));

  // res.end() sends any final data AND signals "I'm done, close the response"
  // You can also call res.end() with no arguments if you've already
  // written everything via res.write().
  res.end(JSON.stringify({ part: 2 }));
});
```

**Common bug interviewers ask about:** *"What happens if you call `res.end()` twice, or try to set a header after calling `res.write()`?"*
> Calling `res.end()` twice throws an error ("write after end"). Setting a header after the body has started being sent (`res.write()` already called) throws `Error: Cannot set headers after they are sent to the client` — a VERY common real-world bug, usually caused by accidentally calling both `res.json()` and `res.send()` (in Express) or forgetting a `return` statement after sending a response in a route handler, causing code to fall through and try to respond again.

---

## 5. Basic Manual Routing (what Express's router replaces)

```javascript
const http = require('http');
const { URL } = require('url');

const server = http.createServer((req, res) => {
  const { pathname } = new URL(req.url, `http://${req.headers.host}`);

  // Manual routing: checking method + path combinations by hand.
  // This is EXACTLY the repetitive, error-prone pattern that Express's
  // app.get()/app.post()/etc. exists to eliminate.
  if (req.method === 'GET' && pathname === '/') {
    res.end('Home page');
  } else if (req.method === 'GET' && pathname === '/about') {
    res.end('About page');
  } else if (req.method === 'GET' && pathname.startsWith('/users/')) {
    const userId = pathname.split('/')[2]; // crude manual param extraction
    res.end(`User ID: ${userId}`);
  } else if (req.method === 'POST' && pathname === '/users') {
    res.end('Create a user (would parse body here)');
  } else {
    res.statusCode = 404;
    res.end('Not Found');
  }
});

server.listen(3000);
```

Notice how quickly this becomes unmanageable as routes grow — every new endpoint adds another `else if` branch, there's no clean way to extract URL parameters like `/users/:id/orders/:orderId`, and there's no concept of middleware (shared logic like auth checks or logging that should run before multiple routes). **This exact pain is the reason Express (and similar frameworks) exist** — and now you understand precisely what problem they solve, instead of treating routing as "just something Express does."

---

## 6. Server Events Worth Knowing (EventEmitter pattern again — Day 4 callback)

```javascript
const server = http.createServer((req, res) => {
  res.end('OK');
});

// Fires once the server has successfully started listening
server.on('listening', () => {
  console.log('Server is up');
});

// Fires if something goes wrong starting the server — e.g. port already in use
server.on('error', (err) => {
  if (err.code === 'EADDRINUSE') {
    console.error('Port already in use!');
  }
});

// Fires for EVERY new TCP connection (before any HTTP request is even parsed) —
// useful for low-level connection-count monitoring/limiting
server.on('connection', (socket) => {
  console.log('New TCP connection established');
});

server.listen(3000);
```

---

## 7. Keep-Alive and Connection Handling (a real production detail)

**Background:** by default, modern HTTP clients (browsers, most HTTP libraries) reuse the same underlying TCP connection for multiple requests instead of opening a brand-new connection every time — this is called **HTTP Keep-Alive**, and it significantly reduces latency since establishing a new TCP connection (and TLS handshake, for HTTPS) has real overhead.

```javascript
// Node's http.Server has a default keep-alive timeout. You CAN tune this,
// and it occasionally matters in production when working with load
// balancers (Week 4 topic!) — if your Node server's keep-alive timeout is
// SHORTER than your load balancer's, you can get intermittent connection
// reset errors, because the LB tries to reuse a connection your Node
// server already silently closed.
server.keepAliveTimeout = 65000; // milliseconds — slightly longer than
                                   // most load balancers' default (60s) is a common fix
```

This is a real, slightly advanced production gotcha — mentioning it in an interview about "have you debugged any tricky production networking issues" signals real-world depth.

---

## 8. How this connects to real backend work (3 YOE framing)

- **"What does Express give you over raw `http`?"** → Routing (`app.get/post/put/delete` with param extraction like `:id`), middleware chaining, automatic body parsing (`express.json()`), simplified response helpers (`res.json()`, `res.status()`), and error-handling middleware — all built on top of the exact `req`/`res` objects you used today.
- **"Why does `req.body` sometimes come back `undefined` in Express?"** → Usually because the relevant body-parsing middleware (`express.json()` for JSON, `express.urlencoded()` for form data) wasn't registered, OR the `Content-Type` header sent by the client doesn't match what the parser expects — Express's parsers check the header before attempting to parse, exactly mirroring how you'd have to manually check it in raw Node.
- **"You're debugging a 'Cannot set headers after they are sent' error — what usually causes this?"** → A code path that calls a response method (`res.send`/`res.json`/`res.end`) more than once for the same request — often because of a missing `return` after an early response inside a conditional, allowing code to fall through to a second response attempt later in the function.
- **"How would you build a simple health-check endpoint with zero dependencies?"** → Exactly the raw `http.createServer` + manual routing pattern shown in section 5, checking `req.url === '/health'`.

---

## 9. Hands-on practice for today

```javascript
// practice-day6.js
// Task: build a tiny raw-Node JSON API with 2 routes and manual body parsing,
// no Express, no third-party packages at all.

const http = require('http');
const { URL } = require('url');

const users = [{ id: 1, name: 'Alice' }, { id: 2, name: 'Bob' }];

const server = http.createServer((req, res) => {
  const { pathname } = new URL(req.url, `http://${req.headers.host}`);

  res.setHeader('Content-Type', 'application/json');

  if (req.method === 'GET' && pathname === '/users') {
    res.statusCode = 200;
    res.end(JSON.stringify(users));
    return; // IMPORTANT: prevents falling through to other checks below
  }

  if (req.method === 'POST' && pathname === '/users') {
    let body = [];
    req.on('data', (chunk) => body.push(chunk));
    req.on('end', () => {
      try {
        const newUser = JSON.parse(Buffer.concat(body).toString());
        newUser.id = users.length + 1;
        users.push(newUser);
        res.statusCode = 201;
        res.end(JSON.stringify(newUser));
      } catch (err) {
        res.statusCode = 400;
        res.end(JSON.stringify({ error: 'Invalid JSON body' }));
      }
    });
    return;
  }

  res.statusCode = 404;
  res.end(JSON.stringify({ error: 'Not found' }));
});

server.listen(3000, () => console.log('Listening on http://localhost:3000'));
```

Test it:
```bash
curl http://localhost:3000/users
curl -X POST http://localhost:3000/users -H "Content-Type: application/json" -d '{"name":"Charlie"}'
```

Notice everything you just did manually — parsing the body, routing by method+path, setting status codes — is exactly what Express does for you automatically. Tomorrow's revision day will have you build something similar again but reflect on these pain points before Week 2 introduces Express properly.

---

## 10. Interview Q&A Recap (say these out loud)

1. **Q: What's the relationship between Express and the core `http` module?**
   A: Express is built entirely on top of `http` — it doesn't replace it. Express's `req`/`res` objects extend Node's native request/response objects, adding convenience methods, routing, and middleware support around the same underlying mechanics.

2. **Q: Why doesn't `req.body` exist by default in raw Node, and how would you get it?**
   A: Because the request body arrives as a stream of chunks, not a ready-made object. You manually listen for `'data'` events to accumulate chunks and an `'end'` event to know when the full body has arrived, then parse it (e.g., `JSON.parse`) yourself.

3. **Q: What causes "Cannot set headers after they are sent to the client"?**
   A: Calling a response-sending method, or setting a header, after the response has already started being sent (e.g., after `res.write()` or a prior `res.end()`) — usually from a missing `return` after an early response in a route handler.

4. **Q: How does `req.url` differ from what you usually access in Express (`req.path`, `req.query`)?**
   A: `req.url` in raw Node is just the raw path + query string as one string (e.g., `/search?q=node`). Express parses this for you into `req.path` (just the path) and `req.query` (a parsed object of query parameters). In raw Node you'd use the `URL` class to do this parsing yourself.

5. **Q: What is HTTP Keep-Alive and why might its timeout setting matter in production?**
   A: It allows a TCP connection to be reused across multiple HTTP requests instead of opening a new connection each time, reducing latency overhead. Mismatched keep-alive timeouts between your Node server and an upstream load balancer can cause intermittent connection reset errors if the LB tries to reuse a connection Node already closed.

---

## Tomorrow (Day 7) Preview — Week 1 Revision Day
We consolidate everything from Days 1-6 (runtime/modules, event loop, async patterns, core modules, streams, raw HTTP) into one combined mini-project: a simple raw-Node file server with no dependencies. This is meant to cement the fundamentals before Week 2 introduces Express and databases, where things start moving faster.
