# DAY 13 — Modern JS & Modules (Complete, Interview-Grade Guide)

> Goal for today: understand WHY each of these modern features was added — what specific pain point existed before it — not just the syntax. Several of today's topics (Map vs Object, modules vs scripts) are classic "explain the tradeoff" interview questions, so we'll cover both sides of each comparison in depth.

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

## TOPIC 1: ES6 Modules (`import`/`export`)

### WHAT
Modules let you split JavaScript code across MULTIPLE files, where each file explicitly controls what it shares (`export`) and explicitly pulls in what it needs from other files (`import`) — instead of every script sharing ONE single global scope.

### WHY
Before modules existed, every `<script>` tag on a page shared the SAME global scope (Day 5) — meaning a variable named `total` in one file could silently CLASH with a variable named `total` in another file, both fighting over the same global namespace. As applications grew to hundreds of files, this became unmanageable and bug-prone. Modules solve this by giving EVERY file its OWN private scope by default — nothing leaks out unless you explicitly `export` it, and nothing comes in unless you explicitly `import` it.

### HOW IT WORKS INTERNALLY

**Named exports — export MULTIPLE specific things from a file, each referenced by name**
```javascript
// file: mathUtils.js
export function add(a, b) {
  return a + b;
}
export function multiply(a, b) {
  return a * b;
}
export const PI = 3.14159;
```
```javascript
// file: main.js
import { add, multiply, PI } from "./mathUtils.js"; // MUST match the exact exported names

console.log(add(2, 3));      // 5
console.log(multiply(2, 3)); // 6
console.log(PI);             // 3.14159
```

**Default exports — export ONE main thing per file, which can be imported under ANY name you choose**
```javascript
// file: User.js
export default class User {
  constructor(name) {
    this.name = name;
  }
}
```
```javascript
// file: main.js
import User from "./User.js";          // no curly braces - and the name "User" here is YOUR choice
import SomeOtherName from "./User.js"; // this would ALSO work - default exports aren't tied to a fixed name
```

**Renaming imports/exports with `as`**
```javascript
import { add as sum } from "./mathUtils.js"; // importing "add" but using it locally as "sum"
console.log(sum(1, 2)); // 3
```

**Why does each MODULE file automatically get its own private scope, unlike regular scripts?**
```html
<script src="file1.js"></script> <!-- regular scripts share ONE global scope -->
<script src="file2.js"></script>

<script type="module" src="file1.js"></script> <!-- modules each get their OWN scope -->
<script type="module" src="file2.js"></script>
```
The `type="module"` attribute (or using `.mjs`, or Node's `"type": "module"` config) is what tells the browser/Node "treat this file as a module" — automatically wrapping it in its own private scope and enabling `import`/`export` syntax, which is NOT available in regular, non-module scripts at all.

### Explain Like You're Teaching
"Before modules, writing JavaScript across multiple files was like several people writing on the SAME shared whiteboard at the same time — if two people both wrote a note labeled `total`, the second one would just overwrite the first, total chaos in a big team. Modules give EVERY file its OWN private whiteboard. If file A wants to SHARE something with file B, it has to explicitly write that specific thing on a piece of paper and physically hand it over (`export` → `import`) — nothing leaks between whiteboards automatically anymore. `export default` is like saying 'here's the ONE main thing this file is fundamentally ABOUT' — whoever imports it can call it whatever they like, since there's only one main thing to refer to."

### Code Implementation
```javascript
// file: cartUtils.js
export function calculateTotal(items) {
  return items.reduce((sum, item) => sum + item.price, 0); // using Day 6's reduce!
}

export function applyDiscount(total, percent) {
  return total - (total * percent / 100);
}

export default function formatCurrency(amount) {
  return "$" + amount.toFixed(2);
}
```
```javascript
// file: app.js
import formatCurrency, { calculateTotal, applyDiscount } from "./cartUtils.js";
// notice: default import (no braces) and named imports (with braces) can be combined in ONE line

const cart = [{ price: 100 }, { price: 250 }];
const total = calculateTotal(cart);
const discounted = applyDiscount(total, 10);
console.log(formatCurrency(discounted)); // "$315.00"
```

### Common Mistakes & Interview Follow-Ups
**Q: "What's the practical difference between using modules and just using multiple regular `<script>` tags?"**
Regular scripts: shared global scope (risk of naming collisions), load order matters strictly, no explicit dependency declaration (you just have to KNOW file B depends on file A and order your `<script>` tags correctly by hand). Modules: each file has private scope, dependencies are explicit and declared right in the code (`import` statements show EXACTLY what a file needs), and browsers can even optimize loading because the dependency graph is clear upfront.

---

## TOPIC 2: Template Literals

### WHAT
Template literals use backticks (`` ` ``) instead of quotes, allowing embedded expressions (`${...}`) directly inside a string, and allowing actual multi-line strings without special characters.

### WHY
Before template literals (ES6/2015), building a string with embedded variables required clunky string concatenation with `+`, which became hard to read with multiple variables, and multi-line strings required awkward `\n` characters or `+` chained across multiple lines. Template literals make string-building dramatically more readable.

### HOW IT WORKS INTERNALLY
```javascript
let name = "Maya";
let age = 28;

// OLD WAY - string concatenation
let oldMessage = "Hello, my name is " + name + " and I am " + age + " years old.";

// NEW WAY - template literal, embedded expressions directly
let newMessage = `Hello, my name is ${name} and I am ${age} years old.`;

console.log(oldMessage === newMessage); // true - SAME resulting string, very different to write/read

// ANY valid JavaScript expression can go inside ${...}, not just variables!
console.log(`Next year I'll be ${age + 1}`);            // Next year I'll be 29
console.log(`Status: ${age >= 18 ? "Adult" : "Minor"}`); // Status: Adult (using Day 2's ternary!)

// Multi-line strings - no \n needed, just press Enter inside the backticks
let multiLine = `Line one
Line two
Line three`;
console.log(multiLine);
// Line one
// Line two
// Line three
```

### Explain Like You're Teaching
"Template literals are like a Mad Libs form with FILL-IN-THE-BLANK boxes already built directly into the sentence — `${name}` is a transparent window where JavaScript peeks in, calculates whatever's inside, and drops the RESULT directly into that exact spot in the string. This avoids the error-prone juggling of opening/closing quotes and `+` signs every time you want to mix text with a variable's value."

### Code Implementation
```javascript
// REAL WORLD: building a dynamic HTML snippet (connects directly to Day 12!)
function createUserCard(user) {
  return `
    <div class="user-card">
      <h3>${user.name}</h3>
      <p>Age: ${user.age}</p>
      <p>Status: ${user.isActive ? "Active" : "Inactive"}</p>
    </div>
  `;
}

const user = { name: "Tom", age: 25, isActive: true };
console.log(createUserCard(user));
// Produces a clean, multi-line, readable HTML string with all values correctly embedded
```

### Common Mistakes & Interview Follow-Ups
**Q: "Can you nest template literals inside each other?"**
```javascript
let isVip = true;
let name = "Sara";
// Yes - though it gets harder to read, here's a valid nested example:
let message = `Welcome ${isVip ? `VIP guest ${name}` : `guest ${name}`}!`;
console.log(message); // "Welcome VIP guest Sara!"
```

---

## TOPIC 3: `Map`

### WHAT
`Map` is a built-in data structure (ES6/2015) for storing key-value pairs — similar to a plain object, but with important differences: keys can be of ANY type (not just strings), it maintains insertion order reliably, and it has a genuine `.size` property and proper iteration support.

### WHY
Plain objects (Day 7) have several quiet limitations as a "dictionary"/lookup structure: object keys are ALWAYS converted to strings (even if you use a number or object as a key!), objects have NO direct `.length`/`.size`, and objects inherit some properties from `Object.prototype` (Day 9) which can occasionally cause unexpected collisions. `Map` was designed specifically to be a CLEANER, more predictable key-value store when these issues matter.

### HOW IT WORKS INTERNALLY

```javascript
const userRoles = new Map();

userRoles.set("alice", "admin");   // .set(key, value) - adds or updates an entry
userRoles.set("bob", "editor");
userRoles.set("carol", "viewer");

console.log(userRoles.get("alice")); // "admin" - .get(key) retrieves a value
console.log(userRoles.has("bob"));    // true - .has(key) checks existence
console.log(userRoles.size);          // 3 - a REAL property, unlike objects which need Object.keys().length

userRoles.delete("carol");
console.log(userRoles.size); // 2

// Iterating a Map - maintains INSERTION order reliably, and works directly with for...of (Day 3!)
for (const [key, value] of userRoles) {
  console.log(key, value);
}
// alice admin
// bob editor
```

**The KEY differentiator: Map keys can be ANY type, including objects, and they stay that exact type — objects ALWAYS stringify their keys**
```javascript
const objKey = { id: 1 };

// Using a plain OBJECT as a key
let plainObj = {};
plainObj[objKey] = "some value"; // objKey gets converted to the STRING "[object Object]" as the key!
console.log(Object.keys(plainObj)); // ["[object Object]"] - the original object key is LOST/stringified

// Using a Map
let map = new Map();
map.set(objKey, "some value"); // the ACTUAL object reference is used as the key, no stringification!
console.log(map.get(objKey)); // "some value" - works correctly, because Map preserves the real key type

const objKey2 = { id: 1 }; // a DIFFERENT object, even with the same shape/content
console.log(map.get(objKey2)); // undefined! Map compares object keys by REFERENCE, not by content
```

### `Map` vs Plain Object — Side-by-Side (A Genuine Interview Question)

| | Plain Object | `Map` |
|---|---|---|
| Key types allowed | Only strings/symbols (others get converted to strings!) | ANY type, including objects, numbers, functions - preserved exactly |
| Size | Must manually compute with `Object.keys(obj).length` | Direct `.size` property |
| Iteration order | Guaranteed for string keys since ES2015, but historically inconsistent | ALWAYS guaranteed insertion order |
| Inherited properties | Has inherited properties from `Object.prototype` (Day 9!) that can cause subtle bugs | Completely clean, no inherited "junk" keys |
| Performance for frequent add/remove | Generally fine for small sets of keys | Optimized specifically for frequent additions/removals |
| JSON support | Directly supported by `JSON.stringify()` | NOT directly supported - needs manual conversion |

### Explain Like You're Teaching
"A plain object is like a filing cabinet where EVERY drawer label gets automatically retyped as plain text, even if you tried to label a drawer with an actual physical KEY (an object) — the cabinet just converts it to the words '[object Object]' and loses the real key entirely. A `Map` is a SPECIALIZED cabinet built specifically to accept ANY kind of label — a key, a number, a photograph (an object) — and remembers EXACTLY which physical item you used as the label, never converting it to text behind your back."

### Code Implementation
```javascript
// REAL WORLD: caching expensive function results, keyed by the actual input object
const cache = new Map();

function expensiveCalculation(inputObject) {
  if (cache.has(inputObject)) {
    console.log("Returning cached result");
    return cache.get(inputObject);
  }
  console.log("Calculating fresh result...");
  const result = inputObject.value * 1000; // pretend this is expensive
  cache.set(inputObject, result);
  return result;
}

const myInput = { value: 5 };
console.log(expensiveCalculation(myInput)); // Calculating fresh result... 5000
console.log(expensiveCalculation(myInput)); // Returning cached result, 5000 (same object reference, cache hit!)
```

---

## TOPIC 4: `Set`

### WHAT
`Set` is a built-in data structure (ES6/2015) that stores a collection of UNIQUE values — any duplicate you try to add is silently ignored.

### WHY
Removing duplicates from an array used to require manual loop logic or filtering tricks. `Set` provides this as a fundamental, built-in guarantee — anytime you need "a list of unique things" (unique tags, unique visitor IDs, unique selected items), `Set` directly enforces that uniqueness automatically.

### HOW IT WORKS INTERNALLY
```javascript
const uniqueNumbers = new Set();

uniqueNumbers.add(1);
uniqueNumbers.add(2);
uniqueNumbers.add(2); // duplicate - SILENTLY ignored, no error, no effect
uniqueNumbers.add(3);

console.log(uniqueNumbers.size); // 3 - NOT 4, the duplicate "2" was never actually added twice

console.log(uniqueNumbers.has(2)); // true
uniqueNumbers.delete(2);
console.log(uniqueNumbers.has(2)); // false

// Iterating - also maintains insertion order, works with for...of
for (const num of uniqueNumbers) {
  console.log(num);
}
```

**The most popular real-world `Set` trick: instantly de-duplicating an array, using spread (Day 6) to convert back**
```javascript
const numbersWithDuplicates = [1, 2, 2, 3, 3, 3, 4];
const uniqueArray = [...new Set(numbersWithDuplicates)]; // Set removes dupes, spread converts back to an array
console.log(uniqueArray); // [1, 2, 3, 4]
```

### Explain Like You're Teaching
"A `Set` is like a guest list at an exclusive event where the bouncer specifically checks if a name is ALREADY on the list before adding it again — if you try to add 'John' twice, the bouncer just shrugs and says 'he's already on the list,' with zero error, zero duplicate entry. The popular `[...new Set(array)]` trick is like taking a messy guest list with repeated names, handing it to this strict bouncer to rebuild CLEANLY (the Set), and then copying the final clean list back onto a normal piece of paper (an array) using spread."

### Code Implementation
```javascript
// REAL WORLD: tracking unique visitors to a page
const visitorIds = new Set();

function recordVisit(userId) {
  const wasNew = !visitorIds.has(userId);
  visitorIds.add(userId);
  console.log(wasNew ? "New visitor!" : "Returning visitor");
}

recordVisit("user123"); // New visitor!
recordVisit("user456"); // New visitor!
recordVisit("user123"); // Returning visitor
console.log("Total unique visitors:", visitorIds.size); // 2

// REAL WORLD: removing duplicate tags from a blog post
const tags = ["js", "react", "js", "webdev", "react"];
const uniqueTags = [...new Set(tags)];
console.log(uniqueTags); // ["js", "react", "webdev"]
```

### Common Mistakes & Interview Follow-Ups
**Q: "Does `Set` consider two DIFFERENT objects with the same content as duplicates?"**
```javascript
const set = new Set();
set.add({ name: "Sam" });
set.add({ name: "Sam" }); // a DIFFERENT object, even though it looks identical
console.log(set.size); // 2! Not 1 - Set compares objects by REFERENCE, exactly like Map's keys (Topic 3)
// Only primitive values (strings, numbers) are compared by their actual VALUE for uniqueness purposes
```

---

## TOPIC 5: Symbols (Basics)

### WHAT
`Symbol()` creates a value that is **guaranteed to be completely unique**, even if you create two symbols with the identical description text — used primarily as object property keys that won't accidentally collide with any other key.

### WHY
If two different parts of a large codebase (or two different libraries) both want to add a property to the SAME object, using a regular string key risks accidentally overwriting each other's data if they happen to pick the same key name. Symbols guarantee that NEVER happens, because every symbol is unique, even ones created with the exact same description.

### HOW IT WORKS INTERNALLY
```javascript
const sym1 = Symbol("id");
const sym2 = Symbol("id"); // SAME description text as sym1

console.log(sym1 === sym2); // false! Even with identical descriptions, they are NEVER equal

const user = {
  name: "Maya",
  [sym1]: "unique-id-12345" // using a symbol as a computed property key (bracket notation, Day 7!)
};

console.log(user[sym1]); // "unique-id-12345"
console.log(Object.keys(user)); // ["name"] - symbol keys are HIDDEN from normal enumeration methods!
```

### Explain Like You're Teaching
"A Symbol is like a custom-made, one-of-a-kind key that a locksmith creates JUST for you, where it's PHYSICALLY IMPOSSIBLE for anyone else to accidentally create a matching duplicate, even if they describe their request to the locksmith using the exact same words you did. This guarantees that whatever drawer you use that key to lock will NEVER accidentally get opened (or overwritten) by someone else's similarly-described key."

### Code Implementation
```javascript
// REAL WORLD (simplified illustration): avoiding property name collisions between two "libraries"
const libraryASymbol = Symbol("metadata");
const libraryBSymbol = Symbol("metadata"); // SAME description, but a genuinely different symbol

const sharedObject = {};
sharedObject[libraryASymbol] = "Library A's private data";
sharedObject[libraryBSymbol] = "Library B's private data";

console.log(sharedObject[libraryASymbol]); // "Library A's private data"
console.log(sharedObject[libraryBSymbol]); // "Library B's private data" - NO collision, despite same description!
```

> Note: Symbols are a fairly advanced, less frequently used feature in everyday code — interviewers mostly want to confirm you know WHY they're unique and that they don't show up in normal property enumeration, rather than expecting deep daily usage.

---

## TOPIC 6: The `Date` Object

### WHAT
`Date` is JavaScript's built-in object for working with dates and times — creating a specific date, reading parts of it (year, month, day), and performing date-related calculations.

### WHY
Nearly every real application needs to display dates, calculate durations, or compare timestamps (a post's creation date, an event countdown, age calculation). `Date` is the standard built-in tool for all of this.

### HOW IT WORKS INTERNALLY

```javascript
const now = new Date(); // current date AND time, right now
console.log(now); // e.g., Thu Jun 25 2026 10:30:00 GMT+0000

const specificDate = new Date(2026, 5, 25); // year, MONTH (0-indexed!), day
console.log(specificDate); // June 25, 2026 - NOTICE: month 5 means JUNE, not May!

const fromString = new Date("2026-06-25"); // ISO format string - widely supported and recommended
console.log(fromString);

// Reading parts of a date
console.log(now.getFullYear());  // e.g., 2026
console.log(now.getMonth());     // 0-11 (ZERO-indexed - January is 0, December is 11!)
console.log(now.getDate());      // 1-31 (day of the MONTH - this one is NOT zero-indexed)
console.log(now.getDay());       // 0-6 (day of the WEEK - Sunday is 0)
console.log(now.getHours());     // 0-23
console.log(now.getTime());      // milliseconds since Jan 1, 1970 (the "Unix epoch") - a single huge number
```

**Why is `getMonth()` zero-indexed, but `getDate()` is not? (a genuinely confusing, frequently-asked detail)**
This is a historical quirk of JavaScript's `Date` design (modeled after an older Java API from the 1990s) — `getMonth()` returns 0-11 (matching array-index style, Day 6), while `getDate()` (day of the month) returns 1-31 normally, and `getDay()` (day of the WEEK) returns 0-6. There's no deep logical reason for the inconsistency — it's simply a known quirk you must memorize, and a real source of off-by-one bugs (e.g., displaying "May" instead of "June" because someone forgot the zero-indexing).

**Calculating durations between dates**
```javascript
const start = new Date(2026, 0, 1);   // Jan 1, 2026
const end = new Date(2026, 5, 25);     // Jun 25, 2026

const differenceInMs = end - start; // subtracting two Dates gives the difference in MILLISECONDS
const differenceInDays = differenceInMs / (1000 * 60 * 60 * 24); // converting ms -> seconds -> minutes -> hours -> days
console.log(differenceInDays); // 175 (approximately)
```

### Explain Like You're Teaching
"Think of a `Date` object as a single, precise instant in time, internally stored as ONE giant number (milliseconds since January 1st, 1970 — `getTime()`). All the `.getFullYear()`, `.getMonth()`, etc. methods are just convenient TRANSLATIONS of that one underlying number into human-friendly terms. The one quirk worth tattooing on your brain: `getMonth()` counts from 0 (January = 0), while almost every OTHER date-related method counts normally — this single inconsistency causes a disproportionate number of real bugs, so always double-check it explicitly when displaying a month to a user."

### Code Implementation
```javascript
// REAL WORLD: calculating someone's age from their birthdate
function calculateAge(birthDateString) {
  const birthDate = new Date(birthDateString);
  const today = new Date();

  let age = today.getFullYear() - birthDate.getFullYear();

  // adjust if their birthday hasn't happened yet THIS year
  const hasHadBirthdayThisYear =
    today.getMonth() > birthDate.getMonth() ||
    (today.getMonth() === birthDate.getMonth() && today.getDate() >= birthDate.getDate());

  if (!hasHadBirthdayThisYear) {
    age--;
  }

  return age;
}

console.log(calculateAge("2000-03-15")); // calculates correctly, accounting for whether the birthday already passed
```

### Common Mistakes & Interview Follow-Ups
**Q: "Why might `new Date(2026, 5, 25)` confuse someone expecting May 25th?"**
Exactly the zero-indexing quirk above — `5` means JUNE (the 6th month, but the 5th INDEX, counting from 0), not May. This is a classic, real-world off-by-one bug source, and a good interview answer explicitly calls out that `getMonth()`/the month constructor argument are zero-indexed while day-of-month is not.

---

## TOPIC 7: JSON — `JSON.stringify()` and `JSON.parse()`

### WHAT
JSON (JavaScript Object Notation) is a TEXT-based format for representing structured data, widely used for sending data between a browser and a server (Day 14's `fetch`). `JSON.stringify()` converts a JavaScript object/array INTO a JSON-formatted string. `JSON.parse()` converts a JSON-formatted string BACK INTO a real JavaScript object/array.

### WHY
Network requests, file storage, and browser storage (Day 14) can only handle plain TEXT — they cannot directly send or store a live JavaScript object in memory. JSON is the universal, language-independent TEXT format that gets used to represent that object's data instead, which is why converting back and forth (`stringify`/`parse`) is such a fundamental, constantly-used skill.

### HOW IT WORKS INTERNALLY

```javascript
const user = {
  name: "Maya",
  age: 28,
  isActive: true,
  hobbies: ["reading", "coding"]
};

const jsonString = JSON.stringify(user);
console.log(jsonString); // '{"name":"Maya","age":28,"isActive":true,"hobbies":["reading","coding"]}'
console.log(typeof jsonString); // "string" - it's now PLAIN TEXT, not a real object anymore!

const parsedBack = JSON.parse(jsonString);
console.log(parsedBack); // { name: "Maya", age: 28, isActive: true, hobbies: ["reading", "coding"] }
console.log(typeof parsedBack); // "object" - back to being a real, usable JavaScript object
```

**Important limitation: NOT everything can be converted to/from JSON cleanly**
```javascript
const complexObject = {
  name: "Test",
  greet: function() { console.log("hi"); }, // FUNCTIONS
  createdAt: undefined,                      // UNDEFINED values
  uniqueId: Symbol("id")                      // SYMBOLS
};

console.log(JSON.stringify(complexObject));
// '{"name":"Test"}' - functions, undefined, AND symbols are SILENTLY DROPPED ENTIRELY!
// JSON only supports: strings, numbers, booleans, null, plain objects, and arrays - nothing else.
```

**`JSON.stringify()`'s optional second and third arguments (a detail interviewers like to probe)**
```javascript
const data = { name: "Test", password: "secret123", age: 30 };

// Second argument: a "replacer" function or array - controls WHICH keys get included
const filtered = JSON.stringify(data, ["name", "age"]); // ONLY include these specific keys
console.log(filtered); // '{"name":"Test","age":30}' - "password" excluded entirely

// Third argument: indentation spacing, for human-readable, "pretty-printed" output
const pretty = JSON.stringify(data, null, 2); // 2 spaces of indentation
console.log(pretty);
// {
//   "name": "Test",
//   "password": "secret123",
//   "age": 30
// }
```

### Explain Like You're Teaching
"Imagine a live JavaScript object is a fully assembled piece of furniture sitting in your room — useful, but impossible to MAIL to someone else as-is. `JSON.stringify()` is the process of disassembling that furniture into a flat-packed box with written instructions (a plain TEXT string) that CAN be mailed/stored/transmitted. `JSON.parse()` is the recipient opening that box and reassembling the furniture back into its original, usable form. The catch: some 'furniture features' simply can't survive the flat-packing process at all — a function (a living, running piece of behavior) can't be flattened into a written instruction sheet, so it just gets left out entirely when you flat-pack (stringify) it."

### Code Implementation
```javascript
// REAL WORLD: saving and restoring application data
const appState = {
  username: "dev_jay",
  preferences: { theme: "dark", fontSize: 14 },
  recentSearches: ["javascript", "react", "node"]
};

// "Saving" - converting to a string (this is EXACTLY what happens before sending data over a network, Day 14)
const savedData = JSON.stringify(appState);
console.log(savedData);

// "Loading" - converting back into a usable object
const restoredState = JSON.parse(savedData);
console.log(restoredState.preferences.theme); // "dark" - fully restored, nested data intact too!
```

### Common Mistakes & Interview Follow-Ups
**Q: "What happens if you try to `JSON.parse()` a string that isn't valid JSON?"**
```javascript
// JSON.parse("{name: 'test'}"); // ERROR! Invalid JSON - keys MUST be in double quotes, not single
// JSON.parse("just some text");  // ERROR! Not valid JSON syntax at all

// CORRECT valid JSON requires DOUBLE QUOTES around all keys and string values:
JSON.parse('{"name": "test"}'); // works fine - { name: "test" }
```
Always wrap `JSON.parse()` calls in a `try/catch` (Day 11) when dealing with data from an unpredictable source (like a network response or user input), since malformed JSON throws a real error that will crash your code if uncaught.

---

## TOPIC 8: Modern ES2020+ Features — `??` and `?.` Recap

### WHAT
- **`??` (Nullish Coalescing Operator)** — returns the RIGHT side ONLY if the left side is specifically `null` or `undefined` (nothing else counts).
- **`?.` (Optional Chaining)** — covered fully on Day 7; recapped briefly here since it's part of this same ES2020 feature wave.

### WHY
Recall Day 2's truthy/falsy rules: `||` returns its right side if the left side is ANY falsy value (`0`, `""`, `false`, `null`, `undefined`, `NaN`) — which causes a REAL bug when `0` or `""` are perfectly valid, intentional values that you DON'T want replaced with a fallback. `??` was introduced specifically to fix this, by checking ONLY for `null`/`undefined`, ignoring all other falsy values.

### HOW IT WORKS INTERNALLY — The Critical Difference From `||`

```javascript
let userScore = 0; // a valid, intentional score of ZERO

// Using || (the OLD, problematic way)
let displayScoreOld = userScore || "No score yet";
console.log(displayScoreOld); // "No score yet" !! WRONG - 0 is falsy, so || incorrectly replaced it!

// Using ?? (the CORRECT, modern way)
let displayScoreNew = userScore ?? "No score yet";
console.log(displayScoreNew); // 0 - CORRECT! ?? only replaces null/undefined, and 0 is neither

// Proving exactly which values trigger ?? vs ||
console.log(0 ?? "fallback");          // 0           (?? does NOT trigger - 0 is not null/undefined)
console.log(0 || "fallback");          // "fallback"  (|| DOES trigger - 0 is falsy)
console.log("" ?? "fallback");         // ""          (?? does NOT trigger)
console.log("" || "fallback");         // "fallback"  (|| DOES trigger - "" is falsy)
console.log(null ?? "fallback");       // "fallback"  (?? DOES trigger - this is its actual job)
console.log(undefined ?? "fallback");  // "fallback"  (?? DOES trigger)
console.log(false ?? "fallback");      // false       (?? does NOT trigger - false is not null/undefined)
```

### Explain Like You're Teaching
"`||` asks the broad question 'is the left side ANYTHING falsy at all?' — which sweeps up `0`, empty strings, and `false` along with genuinely 'missing' values, often by mistake. `??` asks a much more PRECISE question: 'is the left side SPECIFICALLY missing — meaning `null` or `undefined` — or does it actually have a real value, even if that value happens to be `0`, `false`, or an empty string?' This precision is exactly why `??` exists: to correctly distinguish between 'the user provided a value of zero' and 'the user provided NO value at all,' which `||` simply cannot tell apart."

### Code Implementation
```javascript
// REAL WORLD: applying ?? for settings that have legitimate falsy values
function createSettings(userSettings) {
  return {
    volume: userSettings.volume ?? 50,        // if user explicitly set volume to 0 (mute), KEEP 0!
    notifications: userSettings.notifications ?? true,
    username: userSettings.username ?? "Guest"
  };
}

console.log(createSettings({ volume: 0, username: "" }));
// { volume: 0, notifications: true, username: "" }
// CORRECT: volume stays 0 (intentional mute), username stays "" (intentionally blank) -
// neither gets incorrectly overwritten, because neither is null/undefined

// Combining ?? with optional chaining (Day 7) - an extremely common real-world pairing
const apiResponse = { user: { profile: {} } };
console.log(apiResponse.user.profile.bio ?? "No bio provided");
// "No bio provided" - profile.bio is undefined, so ?? provides the fallback safely
```

### Common Mistakes & Interview Follow-Ups
**Q: "Can you mix `??` directly with `&&` or `||` in the same expression without parentheses?"**
```javascript
// console.log(true || false ?? "fallback"); // SyntaxError!
// JavaScript deliberately FORBIDS mixing ?? directly with || or && without explicit parentheses,
// because the intended logic would be genuinely ambiguous. You MUST add parentheses to clarify:
console.log((true || false) ?? "fallback"); // true - now valid, parentheses make intent explicit
```
This is a real, enforced syntax rule (not just a style suggestion) — know that JS will throw a SyntaxError, not just produce a confusing result, if you mix them without parentheses.

---

## Map vs Set vs Object vs Array — Full Decision Guide

| Situation | Best Choice |
|---|---|
| Ordered list of similar items | Array |
| Labeled data describing ONE "thing" | Object |
| Key-value pairs with NON-string keys, or needing reliable `.size`/order | Map |
| A collection that must never contain duplicates | Set |
| Need to send data over a network or store as text | `JSON.stringify()` first, regardless of structure |

---

## Day 13 Summary Table (Quick Revision)

| Concept | One-Line Memory Hook |
|---|---|
| ES6 Modules | Each file gets its OWN whiteboard; explicit `export`/`import` to share |
| Template literals | Mad Libs with built-in fill-in-the-blank `${}` windows |
| `Map` | A cabinet that accepts ANY key type and never auto-converts it to text |
| `Set` | A strict guest-list bouncer — no duplicates allowed, ever |
| Symbols | A locksmith-made key guaranteed to never duplicate, even with the same description |
| `Date` | One giant number (ms since 1970) translated into human terms — watch the zero-indexed month! |
| `JSON.stringify/parse` | Flat-packing furniture into mailable text, and reassembling it later |
| `??` vs `\|\|` | `??` only catches null/undefined; `\|\|` catches ALL falsy values (including valid 0/"") |

---

## Mini Practice Task (Do This Before Day 14)

```javascript
// 1. Create two files conceptually: export 3 named functions from one, and a default
//    class from another, then import and use all 4 in a third "main" file.
// 2. Rewrite a string built with + concatenation (at least 3 variables) as a template literal.
// 3. Create a Map storing 3 users keyed by their (object) session tokens, not strings.
// 4. Take an array with duplicate values and de-duplicate it using Set + spread, in one line.
// 5. Write a function calculateAge(birthDateString) using the Date object (watch the
//    zero-indexed month bug!).
// 6. Take a nested object, JSON.stringify() it with pretty-print indentation (2 spaces),
//    then JSON.parse() it back and confirm the nested data survived intact.
// 7. Rewrite 3 places in your own code (anywhere from previous days) that use || for a
//    fallback value, switching to ?? where appropriate - explain WHY each one should or
//    shouldn't change.
```

## Teach-Back Challenge

Try explaining these out loud, in your own words, without checking notes:

1. Why does every JS module file get its own private scope automatically, while regular `<script>` tags don't?
2. Give a concrete real-world example of when a plain Object's key-stringification behavior would actually cause a bug, that using a `Map` would fix.
3. Why is `getMonth()` zero-indexed while `getDate()` is not? Is there a deep logical reason, or is it just a quirk to memorize?
4. What specific types of data get silently DROPPED when you use `JSON.stringify()`? Why does this happen?
5. Walk through, with your own example, exactly why `0 || "fallback"` and `0 ?? "fallback"` give different results, and explain WHY this difference matters in real code.

If you can explain all five confidently, you've mastered Day 13 at interview depth.
