# DAY 2 — Operators & Control Flow

> Goal for today: By the end of this, you should be able to **teach someone else** how JavaScript performs calculations and comparisons, why `==` and `===` are different (a classic interview trap!), and how to make your code "think" and choose different paths using `if/else` and `switch`.

---

## How to use this file

For every topic:
- **WHAT** — the plain definition
- **WHY** — the real-world problem this concept solves
- **HOW** — the syntax / mechanics
- **Explain Like You're Teaching** — a script you can say out loud to a student
- **Code Implementation** — runnable example with comments

---

## TOPIC 1: Arithmetic Operators

### WHAT
Arithmetic operators perform mathematical calculations: `+` (add), `-` (subtract), `*` (multiply), `/` (divide), `%` (modulus/remainder), `**` (exponent/power).

### WHY
Almost every real program needs to calculate something — a bill total, a discount, a countdown timer, a game score. Arithmetic operators are the basic toolkit for any numeric logic.

### HOW
```javascript
let sum = 5 + 3;        // 8
let diff = 5 - 3;       // 2
let product = 5 * 3;    // 15
let quotient = 6 / 3;   // 2
let remainder = 7 % 3;  // 1  (7 divided by 3 leaves remainder 1)
let power = 2 ** 3;     // 8  (2 to the power of 3)
```

### Explain Like You're Teaching
"These work exactly like the math you learned in school — except for `%`, the modulus operator, which is new to most beginners. Think of `%` as 'what's left over.' If you have 7 candies and want to share them equally among 3 friends, each gets 2, and `7 % 3` tells you there's **1 candy left over**. This single operator is secretly used everywhere — checking if a number is even/odd, creating patterns, building clocks/timers."

### Code Implementation
```javascript
// Basic calculator example
let num1 = 10;
let num2 = 3;

console.log(num1 + num2); // 13
console.log(num1 - num2); // 7
console.log(num1 * num2); // 30
console.log(num1 / num2); // 3.3333333333333335
console.log(num1 % num2); // 1

// REAL WORLD use: checking even or odd
let number = 7;
if (number % 2 === 0) {
  console.log(number, "is even");
} else {
  console.log(number, "is odd");
}
// 7 is odd
```

---

## TOPIC 2: Assignment Operators

### WHAT
Assignment operators store or update a value in a variable: `=`, `+=`, `-=`, `*=`, `/=`, `%=`.

### WHY
You constantly need to update existing values — like adding points to a score, or reducing stock count after a sale — without retyping the whole formula. Shorthand assignment operators make code shorter and easier to read.

### HOW
```javascript
let x = 10;

x += 5;  // same as: x = x + 5  → 15
x -= 3;  // same as: x = x - 3  → 12
x *= 2;  // same as: x = x * 2  → 24
x /= 4;  // same as: x = x / 4  → 6
x %= 4;  // same as: x = x % 4  → 2
```

### Explain Like You're Teaching
"Imagine your bank balance. Instead of saying 'my new balance equals my old balance plus 500,' you'd just say 'add 500 to my balance.' That's exactly what `+=` does — it's shorthand for 'update this variable based on its current value.' It's not a new concept, just a faster way to write something you'd do anyway."

### Code Implementation
```javascript
// Shopping cart total example
let cartTotal = 0;

cartTotal += 250; // added an item worth 250
console.log(cartTotal); // 250

cartTotal += 100; // added another item worth 100
console.log(cartTotal); // 350

cartTotal -= 50; // applied a discount of 50
console.log(cartTotal); // 300
```

---

## TOPIC 3: Comparison Operators (`==` vs `===`)

### WHAT
Comparison operators compare two values and return `true` or `false`: `==` (loose equal), `===` (strict equal), `!=` (loose not equal), `!==` (strict not equal), `>`, `<`, `>=`, `<=`.

### WHY
This is one of the **most important concepts** in JavaScript and a classic interview question. `==` checks only the VALUE and tries to convert types to match (called "type coercion"). `===` checks BOTH the value AND the type, with no conversion. Using `==` carelessly causes confusing bugs, because JS will quietly convert types behind your back.

### HOW
```javascript
console.log(5 == "5");   // true  (converts "5" to number 5, then compares)
console.log(5 === "5");  // false (5 is a number, "5" is a string — different types!)

console.log(0 == false); // true  (false is converted to 0)
console.log(0 === false);// false (different types: number vs boolean)

console.log(null == undefined);  // true
console.log(null === undefined); // false
```

**Golden Rule: Always use `===` and `!==` in real code. Avoid `==` and `!=` entirely.**

### Explain Like You're Teaching
"Imagine comparing a person's age written as the NUMBER `25` versus the WORD `'25'` on a form. `==` is like a lazy security guard who says 'eh, close enough, let them in' — it ignores the fact that one is a number and one is text. `===` is a strict security guard who checks BOTH the value AND the exact 'ID type' — number must match number, string must match string. In real programming, you almost always want the strict guard, because the lazy one causes sneaky, hard-to-find bugs."

### Code Implementation
```javascript
let userAge = "25";    // came from a form input (always a string!)
let requiredAge = 25;  // a number in our code

// Using loose equality (risky)
console.log(userAge == requiredAge);  // true (works, but accidentally)

// Using strict equality (safe and predictable)
console.log(userAge === requiredAge); // false (correctly flags type mismatch)

// The SAFE way: convert type explicitly, then compare strictly
console.log(Number(userAge) === requiredAge); // true (intentional, no surprises)

// Other comparisons
console.log(10 > 5);   // true
console.log(10 < 5);   // false
console.log(10 >= 10); // true
console.log(10 <= 9);  // false
```

---

## TOPIC 4: Logical Operators

### WHAT
Logical operators combine or invert boolean conditions: `&&` (AND), `||` (OR), `!` (NOT).

### WHY
Real-world decisions rarely depend on just ONE condition. "Allow login IF username is correct AND password is correct." "Show free shipping IF cart total is over 500 OR user has a premium membership." Logical operators let your code handle these multi-condition decisions.

### HOW
```javascript
// && (AND) - ALL conditions must be true
true && true;   // true
true && false;  // false

// || (OR) - AT LEAST ONE condition must be true
true || false;  // true
false || false; // false

// ! (NOT) - flips the boolean
!true;  // false
!false; // true
```

### Explain Like You're Teaching
"Think of `&&` like a strict bouncer at a club: 'You need ID **AND** you need to be on the guest list' — miss either one, you're not getting in. `||` is a flexible bouncer: 'You need ID **OR** a personal invite from the owner' — just one is enough. `!` is simply a flip switch — it turns true into false and false into true, like a light switch in reverse."

### Code Implementation
```javascript
let hasTicket = true;
let isVIP = false;

// AND - both must be true
console.log(hasTicket && isVIP); // false (isVIP is false, so fails)

// OR - just one needs to be true
console.log(hasTicket || isVIP); // true (hasTicket is true, that's enough)

// NOT - flips the value
console.log(!hasTicket); // false

// REAL WORLD example: login validation
let username = "admin";
let password = "1234";

if (username === "admin" && password === "1234") {
  console.log("Access granted");
} else {
  console.log("Access denied");
}
// Access granted
```

---

## TOPIC 5: Ternary Operator

### WHAT
The ternary operator is a shorthand, one-line version of `if/else`. Syntax: `condition ? valueIfTrue : valueIfFalse`.

### WHY
For simple yes/no decisions, writing a full `if/else` block can feel bulky. The ternary operator lets you assign a value based on a condition in a single, compact line — extremely common in real code, especially when displaying UI text.

### HOW
```javascript
let result = condition ? "do this if true" : "do this if false";
```

### Explain Like You're Teaching
"The ternary operator is just a fast way to ask a yes/no question and immediately act on the answer. Read it left to right like a sentence: 'IS this true? if yes, give me THIS, otherwise give me THAT.' It's the same logic as `if/else`, just squeezed into one line for simple cases."

### Code Implementation
```javascript
let age = 20;

// Using if/else (longer)
let message;
if (age >= 18) {
  message = "You are an adult";
} else {
  message = "You are a minor";
}
console.log(message);

// Using ternary (shorter, same result)
let message2 = age >= 18 ? "You are an adult" : "You are a minor";
console.log(message2); // You are an adult

// Common real-world use: conditional UI text
let isLoggedIn = false;
console.log(isLoggedIn ? "Welcome back!" : "Please log in");
// Please log in
```

---

## TOPIC 6: `if / else` Statements

### WHAT
`if/else` lets your program execute different blocks of code depending on whether a condition is true or false.

### WHY
This is the foundation of "decision-making" in programming. Every app needs to branch its behavior — show an error if a form is empty, give a discount if a cart total is high enough, display different content based on user role. Without `if/else`, code could only ever run in one fixed sequence.

### HOW
```javascript
if (condition) {
  // runs if condition is true
} else if (anotherCondition) {
  // runs if first was false, but this one is true
} else {
  // runs if none of the above were true
}
```

### Explain Like You're Teaching
"`if/else` is exactly how you make decisions in real life: 'IF it's raining, take an umbrella. ELSE IF it's sunny, wear sunglasses. ELSE just go normally.' Your code checks each condition in order, top to bottom, and runs the FIRST block whose condition is true — then skips the rest."

### Code Implementation
```javascript
let marks = 75;

if (marks >= 90) {
  console.log("Grade: A");
} else if (marks >= 75) {
  console.log("Grade: B");
} else if (marks >= 50) {
  console.log("Grade: C");
} else {
  console.log("Grade: F");
}
// Grade: B

// REAL WORLD example: simple weather advice
let temperature = 35;

if (temperature > 30) {
  console.log("It's hot, drink water!");
} else if (temperature > 15) {
  console.log("Pleasant weather");
} else {
  console.log("It's cold, wear a jacket");
}
// It's hot, drink water!
```

---

## TOPIC 7: `switch` Statement

### WHAT
`switch` checks a single value against multiple possible matches ("cases") and runs the matching block of code.

### WHY
When you have MANY `else if` checks comparing the SAME variable against different fixed values, the code becomes long and harder to read. `switch` is a cleaner, more organized alternative for exactly that situation (comparing one value against many specific options).

### HOW
```javascript
switch (value) {
  case option1:
    // code
    break;
  case option2:
    // code
    break;
  default:
    // code if nothing matches
}
```

**Important:** Always include `break;` — without it, JS will keep running the NEXT case too ("fall-through"), which is rarely what you want (unless intentional).

### Explain Like You're Teaching
"Think of `switch` like a vending machine. You press a button (your value), and the machine checks: 'Is it button A? No. Button B? No. Button C? Yes!' — and gives you exactly that snack, then stops. The `break` is like the machine saying 'done, no need to check further buttons.' Without `break`, it's like the machine keeps dispensing snacks for every button after the one you pressed too — which is messy and usually a mistake."

### Code Implementation
```javascript
let day = 3;
let dayName;

switch (day) {
  case 1:
    dayName = "Monday";
    break;
  case 2:
    dayName = "Tuesday";
    break;
  case 3:
    dayName = "Wednesday";
    break;
  case 4:
    dayName = "Thursday";
    break;
  case 5:
    dayName = "Friday";
    break;
  default:
    dayName = "Weekend";
}

console.log(dayName); // Wednesday

// Example WITHOUT break, showing fall-through (usually a bug!)
function testFallThrough(num) {
  switch (num) {
    case 1:
      console.log("one");
      // no break here - falls through to next case!
    case 2:
      console.log("two");
      break;
    default:
      console.log("other");
  }
}
testFallThrough(1); // prints "one" AND "two" (because no break after case 1)
```

---

## TOPIC 8: Truthy & Falsy Values

### WHAT
Every value in JavaScript, when used in a condition (`if`, `&&`, `||`, etc.), is treated as either "truthy" (acts like `true`) or "falsy" (acts like `false`) — even if it's not literally a boolean.

**The complete list of FALSY values in JavaScript (memorize these — there are only 6!):**
```
false, 0, "" (empty string), null, undefined, NaN
```
**Everything else is truthy** — including `"0"` (string zero), `[]` (empty array), and `{}` (empty object), which often surprises beginners!

### WHY
JS lets you use ANY value inside an `if` condition, not just `true`/`false`. This is incredibly common in real code — like checking if a variable exists, if a string has content, or if an array has items — without writing extra comparison code. But it ALSO causes bugs when developers assume something is falsy when it's actually truthy (like an empty array `[]`, which is truthy!).

### HOW
```javascript
if (value) {
  // runs if value is truthy
}
```

### Explain Like You're Teaching
"JavaScript doesn't force you to write `if (isLoggedIn === true)`. Instead, you can just write `if (isLoggedIn)`, and JS automatically figures out whether to treat that value as true-like or false-like. Almost everything is treated as 'true-like' (truthy) EXCEPT six specific 'empty' or 'nothing' values: `false`, `0`, `''`, `null`, `undefined`, and `NaN`. Memorize this short list, and you'll never be confused — anything NOT on this list is truthy, even tricky ones like an empty array `[]` or the string `'0'`."

### Code Implementation
```javascript
// Falsy values - all of these will go to the ELSE block
if (0) { console.log("truthy"); } else { console.log("falsy: 0"); }
if ("") { console.log("truthy"); } else { console.log("falsy: empty string"); }
if (null) { console.log("truthy"); } else { console.log("falsy: null"); }
if (undefined) { console.log("truthy"); } else { console.log("falsy: undefined"); }
if (NaN) { console.log("truthy"); } else { console.log("falsy: NaN"); }
if (false) { console.log("truthy"); } else { console.log("falsy: false"); }

// Tricky truthy values (common beginner mistake!)
if ([]) { console.log("truthy: empty array"); }   // this RUNS! empty array is truthy
if ({}) { console.log("truthy: empty object"); }  // this RUNS! empty object is truthy
if ("0") { console.log("truthy: string zero"); }  // this RUNS! non-empty string is truthy

// REAL WORLD use: checking if a variable has a value before using it
let userName = "";
if (userName) {
  console.log("Welcome,", userName);
} else {
  console.log("Please enter your name"); // this runs, because "" is falsy
}
```

---

## TOPIC 9: Type Coercion Basics

### WHAT
Type coercion is when JavaScript **automatically converts** a value from one type to another, usually during an operation like `+`, `==`, or inside an `if` condition.

### WHY
JS is a "loosely typed" language, meaning it doesn't strictly enforce types like some other languages do. This flexibility is convenient sometimes but causes confusing, unexpected results if you don't understand the rules — this is one of the most "famous" sources of JS bugs and memes online (e.g., `[] + []` gives `""`).

### HOW

**Key rules to remember:**
```javascript
// Number + String = String (the number gets converted to text)
console.log(1 + "1");     // "11" (string)

// Number - String(number) = Number (the string gets converted to a number)
console.log(5 - "2");     // 3 (number)

// Boolean in math = Number (true becomes 1, false becomes 0)
console.log(true + 1);    // 2

// Anything + String = String
console.log(1 + 2 + "3"); // "33"  (1+2=3 first, then 3 + "3" = "33")
console.log("1" + 2 + 3); // "123" (string concatenation happens left to right)
```

### Explain Like You're Teaching
"Type coercion is JS trying to be 'helpful' by guessing what you meant when you mix types — but its guesses follow strict rules, not randomness. The easiest rule to remember: **the `+` operator prefers strings.** If EITHER side of a `+` is a string, JS converts everything to a string and glues them together (concatenation). But for `-`, `*`, `/`, JS prefers numbers and tries to convert strings into numbers instead. Once you know `+` favors text and the other math operators favor numbers, 90% of coercion confusion disappears."

### Code Implementation
```javascript
// Classic coercion gotchas - try predicting these before running!
console.log("5" + 3);      // "53"  (string wins with +)
console.log("5" - 3);      // 2     (number wins with -)
console.log("5" * "2");    // 10    (both strings converted to numbers)
console.log(true + true);  // 2     (both booleans become 1 + 1)
console.log("5" + true);   // "5true" (true becomes string "true")

// Best practice: convert types EXPLICITLY yourself, don't rely on coercion
let formInput = "42"; // from an HTML form, always a string
let actualNumber = Number(formInput); // explicit conversion
console.log(actualNumber + 8); // 50 (clean, predictable, no guessing)

let num = 42;
let asString = String(num); // explicit conversion the other way
console.log(asString + " is now text"); // "42 is now text"
```

---

## TOPIC 10: Type Conversion (Explicit Conversion)

### WHAT
Type conversion is when **YOU**, the programmer, deliberately convert a value from one type to another, using built-in functions: `Number()`, `String()`, `Boolean()`, `parseInt()`, `parseFloat()`.

This is the **explicit** counterpart to type coercion. Coercion (Topic 9) is JS converting types automatically and silently behind the scenes. Conversion is YOU telling JS exactly what to convert and into what — on purpose, visibly, in your own code.

### WHY
Relying on automatic coercion is risky because it follows hidden rules you have to memorize, and it can silently produce wrong results without any warning (e.g., `"5" + 3` becoming `"53"` when you actually wanted `8`). Explicit conversion fixes this by making your intention **visible in the code itself** — anyone reading it (including you, the teacher) can see exactly what type the value will become, with zero guessing. This is considered a best practice in professional JavaScript.

### HOW

**1. Converting to a Number — `Number()`, `parseInt()`, `parseFloat()`**
```javascript
Number("42");       // 42
Number("42.5");     // 42.5
Number("");         // 0 (empty string converts to 0 — a common gotcha!)
Number("abc");      // NaN (Not a Number — text that isn't a valid number)
Number(true);       // 1
Number(false);      // 0
Number(null);       // 0
Number(undefined);  // NaN

parseInt("42px");   // 42   (reads digits until it hits a non-digit, then stops)
parseInt("3.99");   // 3    (parseInt ignores everything after the decimal point)
parseFloat("3.99"); // 3.99 (parseFloat KEEPS the decimal portion)
```

**2. Converting to a String — `String()` or `.toString()`**
```javascript
String(42);        // "42"
String(true);      // "true"
String(null);      // "null"
String(undefined); // "undefined"

let num = 100;
num.toString();     // "100" (works on numbers/booleans, but NOT on null/undefined directly)
```

**3. Converting to a Boolean — `Boolean()`**
```javascript
Boolean(1);         // true
Boolean(0);         // false
Boolean("hello");   // true
Boolean("");        // false
Boolean(null);      // false
Boolean(undefined); // false
// Rule of thumb: Boolean() just applies the same truthy/falsy rules from Topic 8, explicitly.
```

### Explain Like You're Teaching
"Coercion is JavaScript quietly converting things for you in the background, like a waiter who changes your order slightly without telling you. Conversion is YOU walking into the kitchen and converting it yourself, on purpose, so there's no surprise. `Number()`, `String()`, and `Boolean()` are like labeled conversion machines: drop a value into the `Number()` machine, and it always tries to hand you back a number — clearly, predictably, with no hidden behavior. This is why professional code almost always uses explicit conversion instead of letting coercion happen accidentally."

### Code Implementation
```javascript
// REAL WORLD: handling form input (always arrives as a string)
let ageInput = "28";          // from an HTML form field
let age = Number(ageInput);   // explicit conversion to number
console.log(age + 2);         // 30 (correct math, no coercion surprises)

// Comparing parseInt vs Number on messy strings
console.log(Number("42px"));   // NaN  (Number() is strict, fails on extra characters)
console.log(parseInt("42px")); // 42   (parseInt() reads only the valid leading digits)

// Converting a number for display in the UI
let price = 499.5;
let displayPrice = "Rs. " + String(price); // explicit, clear intention
console.log(displayPrice); // "Rs. 499.5"

// Converting to Boolean for a feature flag check
let userInputValue = "";
let hasValue = Boolean(userInputValue);
console.log(hasValue); // false (empty string is falsy)
```

### Coercion vs Conversion — Side by Side (Commonly Confused!)

| | Type Coercion | Type Conversion |
|---|---|---|
| Who does it? | JavaScript, automatically | You, the programmer, explicitly |
| Visibility | Hidden / implicit | Visible in the code |
| Risk | Higher — can silently produce wrong results | Lower — intention is clear |
| Example | `"5" + 3` → `"53"` (silent) | `Number("5") + 3` → `8` (deliberate) |
| Best practice | Avoid relying on this | Prefer this in real projects |

---

## BONUS TOPIC: Cohesion (Quick Note)

You asked about "cohesion" — since it wasn't clear which meaning you needed, here's a short take on both:

**1. If you meant "coercion"** — that's Topic 9 above (type coercion). Easy mix-up since the words look similar.

**2. If you meant "code cohesion"** (a real software design term) — here's the short version:

### WHAT
Cohesion describes **how closely related the responsibilities of a single function/module are.** "High cohesion" means a function does ONE clear job. "Low cohesion" means a function is doing several unrelated things mixed together.

### WHY
Highly cohesive functions are easier to read, test, reuse, and fix, because each one has a single, clear purpose. Low cohesion makes code harder to maintain, because a function that does five unrelated things becomes risky to change (you might break something unrelated to what you intended to fix).

### Explain Like You're Teaching
"Imagine a kitchen worker whose ONLY job is chopping vegetables — that's high cohesion, focused and simple. Now imagine one worker who chops vegetables, AND answers phones, AND cleans the floor, AND manages money — that's low cohesion. If something goes wrong, it's harder to tell which 'job' caused the problem, because they're all tangled into one role."

### Code Implementation
```javascript
// LOW COHESION - this one function does too many unrelated things
function processOrder(order) {
  console.log("Saving order to database..."); // job 1: saving
  console.log("Sending email to customer...");  // job 2: emailing
  console.log("Updating inventory stock...");   // job 3: inventory
}

// HIGH COHESION - each function has ONE clear responsibility
function saveOrder(order) {
  console.log("Saving order to database...");
}
function sendOrderEmail(order) {
  console.log("Sending email to customer...");
}
function updateInventory(order) {
  console.log("Updating inventory stock...");
}

// Now you can call them together, but each piece is independently
// readable, testable, and reusable
function handleNewOrder(order) {
  saveOrder(order);
  sendOrderEmail(order);
  updateInventory(order);
}
```

> Note: This is a code-design principle, not JS-specific syntax — it applies in every programming language. We'll revisit this idea again on Day 8 (Advanced Functions) and Day 15 (Design Patterns).

---

## Day 2 Summary Table (Quick Revision)

| Concept | One-Line Memory Hook |
|---------|----------------------|
| Arithmetic operators | School math + `%` for "what's left over" |
| Assignment operators | Shorthand for updating a variable based on itself |
| `==` vs `===` | Lazy guard vs strict guard — always use `===` |
| `&&` / `\|\|` / `!` | Strict bouncer / flexible bouncer / light switch |
| Ternary operator | One-line `if/else`: `condition ? yes : no` |
| `if/else` | Top-to-bottom decision making, like real-life choices |
| `switch` | Vending machine matching one value to many cases |
| Truthy/Falsy | Only 6 falsy values: `false, 0, "", null, undefined, NaN` |
| Type coercion | `+` prefers strings, other math operators prefer numbers |
| Type conversion | `Number()`, `String()`, `Boolean()` — explicit, visible, intentional |
| Cohesion (bonus) | A function should do ONE clear job — like one focused kitchen worker |

---

## Mini Practice Task (Do This Before Day 3)

```javascript
// 1. Create two number variables and use all 6 arithmetic operators on them
// 2. Write an if/else chain that grades a test score (A/B/C/F)
// 3. Rewrite a simple if/else as a ternary operator
// 4. Write a switch statement for days of the week
// 5. Predict the output of these WITHOUT running them first, then check:
//    "10" + 5
//    "10" - 5
//    true + true
//    [] ? "yes" : "no"
// 6. Convert the string "99" to a number using THREE different methods
//    (Number(), parseInt(), parseFloat()) and compare results
// 7. Write one function that does 3 unrelated things (low cohesion),
//    then split it into 3 separate high-cohesion functions
```

## Teach-Back Challenge

Try explaining these out loud, in your own words, without checking notes:

1. Why should you always use `===` instead of `==`? Give a real example of a bug `==` could cause.
2. What are the 6 falsy values in JavaScript, and why does this list matter when writing `if` conditions?
3. Explain the difference between how `+` and `-` handle a string and a number differently.
4. What's the real difference between type coercion and type conversion? Give one example of each.
5. In your own words, what does "high cohesion" mean for a function, and why does it make code easier to maintain?

If you can explain these confidently, you've mastered Day 2.
