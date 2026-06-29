# DAY 14 — Error Handling, Async Patterns & Browser APIs (Complete, Interview-Grade Guide)

> Goal for today: connect everything from Day 10/11 (event loop, Promises) to REAL network requests with `fetch`, understand exactly why debouncing/throttling work the way they do (not just "use it for search boxes"), and understand browser storage and memory basics well enough to explain tradeoffs, not just syntax.

---

## How to use this file

For every topic:
- **WHAT** — the plain definition
- **WHY** — the real-world problem this concept solves
- **HOW IT WORKS INTERNALLY** — step-by-step mechanics
- **Every Detail Explained** — nothing glossed over
- **Explain Like You're Teaching** — a script you can say out loud to a student
- **Code Implementation** — runnable example, fully commented
- **Common Mistakes & Interview Follow-Ups** — the exact "but what if..." questions, answered in advance

---

## TOPIC 1: `try/catch/finally` — Full Mechanics

### WHAT
`try/catch/finally` is JavaScript's structure for handling errors gracefully: code in `try` runs normally; if it THROWS an error, execution immediately jumps to `catch`; `finally` runs regardless of whether an error occurred.

### WHY
Without error handling, a single unexpected error (bad user input, a failed network call, a typo causing `undefined.someProperty`) would CRASH your entire program (or at minimum, stop the current function execution and print a scary uncaught error). `try/catch` lets you anticipate "risky" operations and respond gracefully instead of crashing.

### HOW IT WORKS INTERNALLY — Exact Execution Order
```javascript
function riskyDivision(a, b) {
  try {
    if (b === 0) {
      throw new Error("Cannot divide by zero"); // "throw" - manually creates and raises an error
    }
    console.log("Result:", a / b);
    return a / b;
  } catch (error) {
    console.log("Caught an error:", error.message); // error.message holds the text passed to Error()
    return null;
  } finally {
    console.log("Division attempt finished"); // ALWAYS runs - success, failure, even after a "return"!
  }
}

riskyDivision(10, 2); // Result: 5  /  Division attempt finished
riskyDivision(10, 0); // Caught an error: Cannot divide by zero  /  Division attempt finished
```

**Critical, frequently-tested detail: `finally` runs even if `try` or `catch` contains a `return` statement**
```javascript
function test() {
  try {
    return "from try";
  } finally {
    console.log("finally STILL runs, even though try already returned!");
  }
}
console.log(test());
// "finally STILL runs, even though try already returned!"
// "from try"
// NOTICE: finally's console.log runs BEFORE the function's return value is actually delivered back
```

**What exactly does `error` (the caught value) contain?**
```javascript
try {
  throw new Error("Something specific went wrong");
} catch (error) {
  console.log(error.message); // "Something specific went wrong" - the human-readable description
  console.log(error.name);     // "Error" - the TYPE of error (could be TypeError, RangeError, etc.)
  console.log(error.stack);    // a full stack trace string - WHERE exactly the error originated (Day 5's call stack!)
}
```

### Explain Like You're Teaching
"`try` is you attempting something you suspect MIGHT go wrong. `throw` is the moment something actually DOES go wrong, and you're explicitly raising a red flag with a specific message attached. `catch` is the designated person waiting to receive that red flag and respond appropriately, instead of letting the whole operation collapse. `finally` is like the cleanup crew that shows up afterward NO MATTER WHAT happened — whether the attempt succeeded, failed, or even if someone tried to leave early via a `return` — they still do their cleanup job before anyone is allowed to actually leave."

### Code Implementation
```javascript
// REAL WORLD: a custom error class extending the built-in Error (connects to Day 9's inheritance!)
class ValidationError extends Error {
  constructor(message) {
    super(message);          // reuse Error's constructor (Day 9's super())
    this.name = "ValidationError"; // override the default "Error" name with something more specific
  }
}

function validateAge(age) {
  if (typeof age !== "number") {
    throw new ValidationError("Age must be a number");
  }
  if (age < 0 || age > 120) {
    throw new ValidationError("Age must be between 0 and 120");
  }
  return true;
}

function processSignup(age) {
  try {
    validateAge(age);
    console.log("Age is valid:", age);
  } catch (error) {
    if (error instanceof ValidationError) { // Day 9's instanceof, checking the SPECIFIC error type
      console.log("Validation failed:", error.message);
    } else {
      console.log("Unexpected error:", error.message); // catches anything else that wasn't anticipated
    }
  }
}

processSignup(25);    // Age is valid: 25
processSignup(-5);    // Validation failed: Age must be between 0 and 120
processSignup("abc"); // Validation failed: Age must be a number
```

### Common Mistakes & Interview Follow-Ups
**Q: "Why create a CUSTOM error class instead of just throwing a plain `Error` with a descriptive message?"**
Custom error classes let you use `instanceof` (Day 9) to distinguish between DIFFERENT categories of errors and handle each one differently (e.g., a `ValidationError` might just need a user-facing message, while a `NetworkError` might trigger a retry). A plain `Error` gives you no easy way to tell WHICH kind of problem occurred without inspecting the message text itself, which is fragile (text can change; a class type cannot).

**Q: "What's the difference between `try/catch` catching a SYNCHRONOUS error versus an ASYNCHRONOUS one?"**
```javascript
try {
  setTimeout(() => {
    throw new Error("This error happens LATER, asynchronously");
  }, 1000);
} catch (error) {
  console.log("This will NEVER run!"); // try/catch can't catch errors from code that runs LATER
}
// Uncaught Error: This error happens LATER, asynchronously  (crashes, uncaught!)
```
**WHY:** `try/catch` only watches the code that runs SYNCHRONOUSLY, DURING the try block's own execution (Day 5's call stack). By the time the `setTimeout` callback actually runs (Day 10 — after the call stack is empty, from the macrotask queue), the original `try/catch` has ALREADY finished and is no longer "listening." This is exactly why `await` (Day 11) inside an `async function`'s `try/catch` WORKS for catching async errors — because `await` makes the function PAUSE and stay inside that try block until the Promise settles, unlike a raw `setTimeout` callback running independently.

---

## TOPIC 2: The `fetch` API

### WHAT
`fetch(url, options)` is the modern, built-in browser function for making network requests (calling an API, loading data from a server) — it returns a Promise (Day 11), making it naturally compatible with `.then()` chains or `async`/`await`.

### WHY
Before `fetch`, network requests used an older, callback-based API (`XMLHttpRequest`) that was clunky and verbose. `fetch` modernizes this by being Promise-based from the ground up, directly connecting everything you learned on Day 11 to real-world data fetching.

### HOW IT WORKS INTERNALLY — The Two-Step Resolution (A Frequent Source of Confusion!)

```javascript
fetch("https://api.example.com/users")
  .then(response => {
    console.log(response.ok);     // true/false - did the request succeed (status 200-299)?
    console.log(response.status); // e.g., 200, 404, 500 - the actual HTTP status code
    return response.json();        // converts the response BODY into a usable JS object - ALSO returns a Promise!
  })
  .then(data => {
    console.log(data); // the ACTUAL parsed data you wanted, finally available here
  })
  .catch(error => {
    console.log("Network error:", error); // catches genuine network failures (no internet, DNS failure, etc.)
  });
```

**Why are there TWO separate `.then()` steps here? This is a genuinely important, frequently-misunderstood detail.**
The FIRST `.then()` resolves once the HTTP response HEADERS have arrived (you now know the status code, whether it succeeded) — but the actual response BODY (the JSON data) might still be streaming in and hasn't been fully read/parsed yet. Calling `.json()` STARTS that second, separate asynchronous step of reading and parsing the body, which is why it returns its OWN Promise, requiring a SECOND `.then()` to actually get the final, usable data.

**Critical gotcha: `fetch()`'s Promise does NOT reject on HTTP error statuses like 404 or 500!**
```javascript
fetch("https://api.example.com/nonexistent-endpoint")
  .then(response => {
    console.log(response.status); // 404
    console.log(response.ok);      // false
    // the Promise STILL resolved successfully here! fetch only REJECTS on a true network failure
    // (no internet connection, DNS lookup failure, CORS block) - NOT on HTTP error status codes!
    if (!response.ok) {
      throw new Error(`HTTP error: ${response.status}`); // YOU must manually check and throw if needed
    }
    return response.json();
  })
  .catch(error => {
    console.log("Caught:", error.message); // "Caught: HTTP error: 404" - only because WE threw it manually
  });
```
This is one of THE most commonly asked `fetch` interview questions: **"Does fetch reject on a 404?"** The honest, precise answer is **no** — you must check `response.ok` (or `response.status`) yourself and manually `throw` if you want a 404/500 to be treated as an error in your `.catch()`.

### `fetch` with `async`/`await` (the modern preferred style, connecting directly to Day 11)
```javascript
async function getUsers() {
  try {
    const response = await fetch("https://api.example.com/users");
    if (!response.ok) {
      throw new Error(`HTTP error: ${response.status}`);
    }
    const data = await response.json(); // awaiting the SECOND Promise too - both steps clearly visible
    return data;
  } catch (error) {
    console.log("Failed to fetch users:", error.message);
    return null;
  }
}
```

**Sending data WITH a request (POST), using the `options` parameter**
```javascript
async function createUser(userData) {
  const response = await fetch("https://api.example.com/users", {
    method: "POST",                              // default is "GET" if not specified
    headers: {
      "Content-Type": "application/json"          // tells the server what format the body is in
    },
    body: JSON.stringify(userData)                 // Day 13's JSON.stringify - body must be a STRING, not an object!
  });
  return response.json();
}

createUser({ name: "Maya", age: 28 });
```

### Explain Like You're Teaching
"`fetch` is like sending a letter to request information from a faraway office. The FIRST `.then()` is the moment you get an envelope back confirming 'yes, we received your request, here's a receipt showing whether it succeeded' (the response headers/status) — but the ACTUAL detailed report you asked for might still be on its way, sealed inside that envelope, needing a separate step to actually unseal and read it (`response.json()`, the SECOND `.then()`). Critically, getting a receipt that says '404: request not found' is STILL a successfully DELIVERED receipt — the mail system (fetch) did its job correctly, even though the OFFICE'S answer was 'no.' Only if the mail truck itself crashes and the letter never arrives at all (a true network failure) does `fetch`'s Promise actually reject."

### Code Implementation
```javascript
// REAL WORLD: a complete, robust fetch function with proper error handling
async function fetchUserProfile(userId) {
  try {
    const response = await fetch(`https://api.example.com/users/${userId}`);

    if (!response.ok) {
      if (response.status === 404) {
        throw new Error("User not found");
      }
      throw new Error(`Server error: ${response.status}`);
    }

    const userData = await response.json();
    return userData;

  } catch (error) {
    if (error instanceof TypeError) {
      // fetch throws a TypeError specifically for genuine network failures (Day 9's instanceof!)
      console.log("Network error - check your connection");
    } else {
      console.log("Error:", error.message);
    }
    return null;
  }
}
```

### Common Mistakes & Interview Follow-Ups
**Q: "What's the difference between the error you get from a failed network connection versus a 404 response, in terms of what `instanceof` check you'd use?"**
A genuine network failure (no internet, DNS failure) causes `fetch`'s Promise to REJECT with a `TypeError`. A 404/500 does NOT reject at all — it resolves NORMALLY with `response.ok = false`, and it only becomes a `catch`-able error if YOUR code explicitly throws one (as shown above). This distinction — true rejection vs. a successful-but-unsuccessful response — is precisely why you need BOTH a `response.ok` check AND a `catch` block for genuine network issues.

---

## TOPIC 3: Debouncing

### WHAT
Debouncing delays running a function until a certain amount of time has passed WITHOUT the triggering event happening again — if the event fires again before that delay finishes, the timer RESETS, and only the LAST call within a rapid burst actually executes.

### WHY
Some events fire RAPIDLY and REPEATEDLY in a very short time — typing in a search box fires a `keyup`/`input` event for EVERY single keystroke. If you triggered an expensive operation (like an API call) on EVERY keystroke, you'd send dozens of wasted requests for a single search a user is still in the middle of typing. Debouncing ensures the expensive operation only runs ONCE, after the user has genuinely paused.

### HOW IT WORKS INTERNALLY — Built From Scratch, Using Day 5's Closures + Day 10's setTimeout
```javascript
function debounce(callback, delayMs) {
  let timeoutId; // this variable persists across calls because of a CLOSURE (Day 5!)

  return function(...args) { // the rest parameter (Day 6) collects whatever arguments are passed in
    clearTimeout(timeoutId);  // CRITICAL: cancel any PREVIOUSLY scheduled call before scheduling a new one

    timeoutId = setTimeout(() => {
      callback(...args); // spread (Day 6) the collected arguments back out when FINALLY calling the real function
    }, delayMs);
  };
}
```

**Tracing through exactly what happens during rapid typing:**
```javascript
function search(query) {
  console.log("Searching for:", query);
}

const debouncedSearch = debounce(search, 500);

debouncedSearch("h");      // schedules a timer for 500ms - timeoutId = Timer#1
debouncedSearch("he");     // CANCELS Timer#1, schedules a NEW Timer#2 for 500ms
debouncedSearch("hel");    // CANCELS Timer#2, schedules Timer#3
debouncedSearch("hell");   // CANCELS Timer#3, schedules Timer#4
debouncedSearch("hello");  // CANCELS Timer#4, schedules Timer#5
// (user stops typing - 500ms pass with NO new calls)
// ONLY Timer#5 actually survives long enough to fire:
// Output (after 500ms of silence): "Searching for: hello"
// NOTICE: "h", "he", "hel", "hell" NEVER actually triggered search() at all - only the FINAL value did!
```

### Explain Like You're Teaching
"Debouncing is like an elevator that waits a few seconds after the LAST person presses the button before actually closing its doors and leaving — if ANOTHER person rushes in and presses the button again before those few seconds are up, the timer COMPLETELY RESETS. The elevator only actually departs once there's been a genuine PAUSE with nobody pressing anything for the full wait time. This is exactly why debouncing a search box means the API call only fires once the user has actually stopped typing, instead of firing on every single keystroke."

### Code Implementation
```javascript
// REAL WORLD: debouncing a live search input (connects directly to Day 12's event listeners!)
const searchInput = document.querySelector("#search-box");

function performSearch(query) {
  console.log("Sending API request for:", query); // imagine this is a real fetch() call (Topic 2)
}

const debouncedSearch = debounce(performSearch, 500);

searchInput.addEventListener("input", function(event) {
  debouncedSearch(event.target.value); // called on EVERY keystroke, but the ACTUAL search is debounced
});
// Result: rapid typing produces ONLY ONE actual "API request" log, 500ms after the user stops typing
```

### Common Mistakes & Interview Follow-Ups
**Q: "Why does the debounce function need to use a CLOSURE specifically? Could you write it without one?"**
The `timeoutId` variable MUST persist BETWEEN separate calls to the debounced function — it needs to "remember" the previous timer so it can cancel it. This is EXACTLY Day 5's closure mechanism: the returned inner function keeps access to `timeoutId` from the OUTER `debounce` function's scope, even though `debounce()` itself already finished running long ago. Without a closure (e.g., if `timeoutId` were declared freshly INSIDE the inner function every time), there'd be no way to remember or cancel the PREVIOUS timer at all.

---

## TOPIC 4: Throttling

### WHAT
Throttling ensures a function runs AT MOST once every fixed time interval, REGARDLESS of how many times the triggering event fires — unlike debouncing, throttling does NOT wait for a pause; it guarantees REGULAR, evenly-spaced execution during continuous activity.

### WHY
Some events fire CONSTANTLY during an ongoing action — scrolling, resizing a window, mouse movement — and unlike search-box typing, you often WANT regular updates DURING that continuous action (e.g., updating a "scroll progress" bar), not just a single update after it completely stops. Throttling caps the EXECUTION RATE without waiting for a full pause, which debouncing would force you to do.

### HOW IT WORKS INTERNALLY — Built From Scratch
```javascript
function throttle(callback, intervalMs) {
  let isWaiting = false; // a closure variable (Day 5) tracking whether we're currently "cooling down"

  return function(...args) {
    if (!isWaiting) {
      callback(...args);     // run IMMEDIATELY on the first call
      isWaiting = true;       // start the "cooldown" period
      setTimeout(() => {
        isWaiting = false;    // after the interval, allow the NEXT call through again
      }, intervalMs);
    }
    // if isWaiting is true, this call is simply IGNORED - no timer reset, unlike debounce!
  };
}
```

**Tracing through exactly what happens during continuous scrolling:**
```javascript
function logScroll() {
  console.log("Scroll position logged at", new Date().toLocaleTimeString());
}
const throttledScroll = throttle(logScroll, 1000); // at MOST once per second

// Imagine this fires 50 times in the first second alone (scroll events fire VERY rapidly)
// throttledScroll() called 50 times rapidly...
// Output: only ONE "Scroll position logged..." in that first second - the rest are ignored
// Then, if scrolling CONTINUES, ANOTHER log fires roughly every 1000ms, evenly spaced,
// for as long as scrolling continues - unlike debounce, which would only fire ONCE total
// if there's never a full pause in the scrolling.
```

### Debounce vs Throttle — Side-by-Side (THE Classic Interview Question)

| | Debounce | Throttle |
|---|---|---|
| Behavior during continuous activity | Waits for a PAUSE before running once | Runs at REGULAR intervals, even with no pause |
| Resets its timer on each new call? | Yes - every call cancels and restarts the wait | No - ignores calls during the cooldown, doesn't reset it |
| Best for | Search-as-you-type, form validation while typing | Scroll position tracking, resize handling, mouse-move tracking |
| Real-life analogy | An elevator waiting for the LAST person before departing | A bus leaving every 10 minutes on a fixed schedule, no matter how many people are at the stop |

### Explain Like You're Teaching
"If debouncing is an elevator waiting for a pause in button-pressing, throttling is a bus that leaves EVERY 10 minutes on a strict schedule, REGARDLESS of how many people show up in between — it doesn't wait for people to stop arriving, and it doesn't reset its schedule every time someone new shows up. This is exactly why you'd throttle a SCROLL handler (you want REGULAR updates while scrolling continues) but DEBOUNCE a search box (you only want ONE update, after the user genuinely stops typing)."

### Code Implementation
```javascript
// REAL WORLD: throttling a scroll event to update a "back to top" button's visibility
const backToTopButton = document.querySelector("#back-to-top");

function checkScrollPosition() {
  if (window.scrollY > 300) {
    backToTopButton.classList.add("visible");
  } else {
    backToTopButton.classList.remove("visible");
  }
}

const throttledCheck = throttle(checkScrollPosition, 200); // checks AT MOST every 200ms
window.addEventListener("scroll", throttledCheck);
// Without throttling, this would run potentially HUNDREDS of times per second while scrolling -
// throttling caps it to a reasonable, still-responsive-feeling rate
```

---

## TOPIC 5: `localStorage` and `sessionStorage`

### WHAT
Both are browser-provided, simple key-value storage mechanisms for saving data DIRECTLY in the user's browser, persisting BETWEEN page loads. `localStorage` persists indefinitely (until explicitly cleared). `sessionStorage` persists only for the duration of that ONE browser tab's session — closing the tab erases it.

### WHY
Sometimes you need to remember something about a user WITHOUT involving a server at all — a theme preference, a draft form entry, a "don't show this popup again" flag. Server-based storage requires network requests (Topic 2) and a backend; browser storage is instant, simple, and entirely client-side for these lighter use cases.

### HOW IT WORKS INTERNALLY
```javascript
// localStorage - persists even after closing the browser entirely
localStorage.setItem("theme", "dark");
console.log(localStorage.getItem("theme")); // "dark"
localStorage.removeItem("theme");
localStorage.clear(); // removes EVERYTHING this site has stored

// sessionStorage - same API, but cleared when the TAB closes
sessionStorage.setItem("draftMessage", "Hello, this is a draft...");
console.log(sessionStorage.getItem("draftMessage"));
```

**Critical limitation: BOTH only store STRINGS — you must manually use JSON (Day 13) for objects/arrays**
```javascript
const userPreferences = { theme: "dark", fontSize: 14 };

// localStorage.setItem("prefs", userPreferences); // WRONG - silently converts to the useless string "[object Object]"

localStorage.setItem("prefs", JSON.stringify(userPreferences)); // CORRECT - Day 13's JSON.stringify first
const savedPrefs = JSON.parse(localStorage.getItem("prefs"));    // CORRECT - JSON.parse to read it back
console.log(savedPrefs.theme); // "dark"
```

### `localStorage` vs `sessionStorage` vs Cookies (a real comparison interviewers ask about)

| | `localStorage` | `sessionStorage` | Cookies |
|---|---|---|---|
| Persists after closing the tab? | Yes, indefinitely | No, cleared on tab close | Yes, until expiry date set |
| Storage size limit | ~5-10MB | ~5-10MB | ~4KB only |
| Automatically sent to the server? | No - purely client-side | No - purely client-side | Yes, with EVERY matching request (can slow things down) |
| Typical use | Theme preference, cached data, "remember me" flags | Multi-step form drafts, per-tab state | Authentication sessions, server-readable data |

### Explain Like You're Teaching
"`localStorage` is like writing a sticky note and permanently taping it inside a specific drawer in your house — it stays there even after you leave the house and come back days later, until YOU specifically peel it off. `sessionStorage` is like writing that same sticky note on a piece of paper you're carrying around for just THIS ONE visit — the moment you leave (close the tab), that paper gets thrown away automatically. Both only understand plain text, so if you want to store something more complex (an object), you have to first 'flatten' it into text using `JSON.stringify()` (Day 13), exactly like flat-packing furniture before mailing it."

### Code Implementation
```javascript
// REAL WORLD: remembering a user's theme preference across visits
function saveTheme(theme) {
  localStorage.setItem("preferredTheme", theme);
}

function loadTheme() {
  return localStorage.getItem("preferredTheme") ?? "light"; // Day 13's ?? for a sensible default
}

document.body.className = loadTheme(); // apply the saved theme immediately when the page loads

document.querySelector("#dark-mode-toggle").addEventListener("click", function() {
  const newTheme = document.body.className === "dark" ? "light" : "dark";
  document.body.className = newTheme;
  saveTheme(newTheme); // persists for the user's NEXT visit too
});
```

### Common Mistakes & Interview Follow-Ups
**Q: "Is it safe to store sensitive data, like a password or auth token, in `localStorage`?"**
Generally NO — anything in `localStorage` is accessible to ANY JavaScript running on that page, including malicious injected scripts (recall Day 12's XSS discussion). If an attacker manages to run ANY script on your page, they can trivially read everything in `localStorage`. Sensitive authentication tokens are often better handled with secure, `httpOnly` cookies (which JavaScript CANNOT read at all, specifically as a security measure) rather than `localStorage`. This is a genuinely good security-awareness answer for an interview.

---

## TOPIC 6: Memory & Garbage Collection Basics

### WHAT
JavaScript automatically manages memory using a process called "garbage collection" — periodically scanning for objects/values that are NO LONGER REACHABLE from anywhere in your running code, and freeing up the memory they were using.

### WHY
In some older/lower-level languages, developers must MANUALLY allocate and free memory — forgetting to free it causes a "memory leak" that slowly consumes all available memory. JavaScript removes this manual burden almost entirely, but understanding the BASIC principle (reachability) helps you recognize and avoid the specific patterns that STILL cause memory leaks even in a garbage-collected language.

### HOW IT WORKS INTERNALLY — Reachability

```javascript
let user = { name: "Maya" }; // an object created, referenced by "user" - REACHABLE, kept in memory

user = null; // "user" no longer points to that object - it's now UNREACHABLE from anywhere
// the garbage collector will EVENTUALLY notice this and free that memory automatically
```

**The classic, REAL way memory leaks STILL happen in JavaScript despite automatic garbage collection: forgotten references, especially closures (Day 5) and event listeners (Day 12)**
```javascript
function setupHandler() {
  const hugeData = new Array(1000000).fill("some data"); // a genuinely large chunk of memory

  document.querySelector("#button").addEventListener("click", function() {
    console.log(hugeData.length); // this callback closure KEEPS "hugeData" alive/reachable, FOREVER,
    // for as long as the event listener itself remains attached - even if you never actually need
    // hugeData for anything else in your program! The closure (Day 5) holds onto it.
  });
}
// If this button/listener is never removed, "hugeData" can NEVER be garbage collected,
// even though logically your program might be completely done with it.
```

### Explain Like You're Teaching
"Garbage collection is like a diligent caretaker who periodically walks through every room in a house, and throws away anything that's no longer connected by a string to anyone who actually still needs it. As long as SOMETHING — a variable, a closure, an active event listener — still holds a 'string' connecting to an object, the caretaker leaves it alone, no matter how long ago it was created. The real danger isn't forgetting to throw things away yourself (the caretaker handles that automatically) — it's accidentally leaving an invisible STRING attached to something you THINK you're done with, like a closure silently keeping a giant piece of data alive just because of one small leftover reference, like the event listener example above."

### Code Implementation
```javascript
// REAL WORLD: avoiding a memory leak by properly cleaning up listeners/data when no longer needed
function createTemporaryWidget() {
  let widgetData = { /* some sizable data */ };

  function handleClick() {
    console.log(widgetData);
  }

  const button = document.querySelector("#widget-button");
  button.addEventListener("click", handleClick);

  return function cleanup() {
    button.removeEventListener("click", handleClick); // breaks the "string" - widgetData CAN be garbage collected now
    widgetData = null; // explicitly clearing the reference too, for clarity
  };
}

const cleanupWidget = createTemporaryWidget();
// ... later, when the widget is no longer needed:
cleanupWidget(); // properly releases the listener AND the data it was keeping alive
```

### Common Mistakes & Interview Follow-Ups
**Q: "If JavaScript has automatic garbage collection, why do memory leaks still happen at all?"**
Garbage collection only frees memory that is GENUINELY unreachable — it cannot read your MIND about whether you logically "still need" something. If ANY path still exists from a running part of your program to an object (an active closure, an attached event listener, an entry still sitting in an array or Map somewhere), that object remains reachable and is NEVER collected, regardless of whether you actually still use it. The most common real causes: forgotten event listeners (Day 12) on removed DOM elements, closures (Day 5) accidentally holding onto large data longer than needed, and growing arrays/Maps that data is added to but never removed from.

---

## Day 14 Summary Table (Quick Revision)

| Concept | One-Line Memory Hook |
|---|---|
| `try/catch/finally` | Red flag (throw), the person catching it (catch), cleanup crew that always shows up (finally) |
| `fetch` two-step resolution | Receipt confirming delivery (status), THEN unsealing the actual letter (`.json()`) |
| `fetch` doesn't reject on 404 | Getting a "no" answer is still a successfully DELIVERED receipt |
| Debounce | Elevator waiting for a pause before departing — timer RESETS on every new call |
| Throttle | A bus on a fixed schedule — runs at regular intervals, ignores extra calls in between |
| `localStorage`/`sessionStorage` | Permanent sticky note in a drawer vs a temporary note for just this one visit |
| Garbage collection | A caretaker who only discards things with NO string still connecting to them |
| Memory leaks | An invisible, forgotten string (closure/listener) keeping something alive longer than intended |

---

## Mini Practice Task (Do This Before Day 15)

```javascript
// 1. Write a function that throws a custom error class (extending Error) for invalid input,
//    and a caller that catches it specifically using instanceof, separate from other errors.
// 2. Use fetch() (or simulate with a Promise + setTimeout) to "load" data, properly checking
//    response.ok before treating it as success, with full try/catch + async/await.
// 3. Build your own debounce() function from scratch, then use it on a real text input's
//    "input" event, logging only after the user pauses typing for 500ms.
// 4. Build your own throttle() function from scratch, then use it on the window's "scroll"
//    or "resize" event, logging at most once every 300ms.
// 5. Save a small object (with nested data) to localStorage using JSON.stringify, reload
//    conceptually (or just re-read it), and parse it back with JSON.parse.
// 6. Write a short explanation (out loud or written) of a realistic scenario in your OWN
//    code from previous days that could cause a memory leak, and how you'd fix it.
```

## Teach-Back Challenge

Try explaining these out loud, in your own words, without checking notes:

1. Why does `finally` run even after a `return` statement inside `try` or `catch`? Walk through the exact order of what happens.
2. Why doesn't a regular `try/catch` around a `setTimeout` callback actually catch errors thrown inside it? Why does `async`/`await` behave differently?
3. Explain, with your own example, why `fetch()` does NOT reject on a 404 response, and what you must do to treat it as an error anyway.
4. What's the real, practical difference between debounce and throttle — and give one real scenario where using the WRONG one would create a bad user experience.
5. Why can a memory leak still happen in JavaScript despite automatic garbage collection? Give your own example involving a closure or event listener.

If you can explain all five confidently, you've mastered Day 14 at interview depth.
