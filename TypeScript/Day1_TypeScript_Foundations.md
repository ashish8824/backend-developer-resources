# Day 1 — TypeScript Foundations
### Why TypeScript Exists, Basic Types, Arrays, Tuples, Type Inference & Type Aliases

> **How to use this file:** Every topic follows the same pattern — **What** (definition), **Why** (the problem it solves), **How** (syntax + code), and **Teach-it-back** (a simple way to explain it to someone else). Read top to bottom, run the code examples yourself, then try explaining each topic out loud before moving to the next.

---

## 1. What is TypeScript & Why It Exists

### What
TypeScript is a **superset of JavaScript**. That means every valid JavaScript file is already valid TypeScript — TypeScript just adds extra features on top, the biggest one being **static types**.

It was created by Microsoft and "compiles down" (transpiles) into plain JavaScript, because browsers and Node.js can only run JavaScript — they don't understand TypeScript directly.

### Why
JavaScript is a **dynamically typed** language — meaning a variable's type is decided at runtime, and it can change anytime.

```javascript
let age = 25;
age = "twenty five"; // JavaScript allows this. No error, no warning.
```

This flexibility is great for quick scripts, but in large applications it causes problems:

- Bugs that only show up when the app is *running* (in production, in front of users)
- No autocomplete/IntelliSense help in your editor because the editor doesn't know what type a variable "should" be
- Hard to understand what a function expects, especially in big teams

TypeScript solves this by checking types **before** the code ever runs — at "compile time."

```typescript
let age: number = 25;
age = "twenty five"; // ❌ Error caught immediately by TypeScript, before running anything
```

### How (the compilation process)
1. You write code in `.ts` files.
2. You run the TypeScript compiler (`tsc`), which:
   - Checks your code for type errors
   - Strips out the types
   - Outputs plain `.js` files
3. The browser/Node.js runs the resulting `.js` file — it never sees TypeScript at all.

```bash
# Install TypeScript globally
npm install -g typescript

# Check the version
tsc -v

# Compile a single file
tsc app.ts
# This produces app.js
```

**tsconfig.json** — this is the configuration file that tells the compiler *how* to behave (which folder to output to, how strict to be, which JS version to target, etc.)

```bash
# Generates a tsconfig.json file with sensible defaults
tsc --init
```

A minimal example of what's inside:

```json
{
  "compilerOptions": {
    "target": "ES2020",     // which JS version to compile down to
    "module": "commonjs",   // module system to use
    "strict": true,         // turns on all strict type-checking rules
    "outDir": "./dist"      // where compiled JS files go
  }
}
```

### Teach-it-back (one-liner)
"JavaScript trusts you to use the right type of data. TypeScript double-checks you — and tells you about mistakes *before* you run the code, not after."

---

## 2. Basic Types

### What
Basic types are the building blocks of TypeScript. They describe what *kind* of value a variable can hold.

| Type | Description | Example |
|---|---|---|
| `string` | Text values | `"hello"` |
| `number` | Any number (int, float, etc.) | `42`, `3.14` |
| `boolean` | true / false | `true` |
| `null` | Intentional absence of value | `null` |
| `undefined` | Variable declared but not assigned | `undefined` |
| `any` | Disables type checking (avoid this!) | could be anything |
| `unknown` | Safer version of `any` | could be anything, but must be checked first |
| `void` | A function that returns nothing | used as return type |
| `never` | A function that never finishes normally | used for errors/infinite loops |

### Why
Without explicitly knowing types, your editor and the compiler can't catch mistakes like adding a string to a number, calling a method that doesn't exist on that type, or passing the wrong kind of value into a function.

### How

```typescript
// string
let userName: string = "Alice";

// number
let userAge: number = 30;

// boolean
let isLoggedIn: boolean = true;

// null and undefined
let emptyValue: null = null;
let notAssignedYet: undefined = undefined;

// any -> AVOID where possible, it turns off type-checking entirely
let randomValue: any = 5;
randomValue = "now I'm a string"; // no error, but you lose all safety

// unknown -> safer alternative to "any"
let userInput: unknown = "could be anything";

// You MUST narrow/check an "unknown" type before using it:
if (typeof userInput === "string") {
  console.log(userInput.toUpperCase()); // ✅ Safe now, TypeScript knows it's a string
}

// void -> used when a function returns nothing
function logMessage(message: string): void {
  console.log(message);
}

// never -> used when a function never successfully returns
function throwError(message: string): never {
  throw new Error(message);
}
```

### `any` vs `unknown` (a common interview question!)

| | `any` | `unknown` |
|---|---|---|
| Type checking | Fully disabled | Still enforced |
| Can you call methods directly? | Yes (risky) | No, must check type first |
| Recommended? | Avoid | Prefer this when type is genuinely not known |

### Teach-it-back
"Basic types are labels you put on a variable's box, saying 'only text goes in here' or 'only numbers go in here.' `any` removes the label entirely (dangerous). `unknown` keeps the box locked until you check what's actually inside."

---

## 3. Arrays

### What
An array is a list of values of the **same type**.

### Why
In plain JavaScript, an array can mix any types — which often leads to bugs when you expect every item to behave the same way.

```javascript
let scores = [90, 85, "ninety"]; // JS allows mixed types — risky!
```

TypeScript lets you lock an array down to hold only one type, catching mistakes early.

### How

```typescript
// Method 1: type[]
let scores: number[] = [90, 85, 100];
scores.push(75);      // ✅ allowed
scores.push("100");   // ❌ Error: string not assignable to number

// Method 2: Array<type> (generic syntax, same meaning)
let names: Array<string> = ["Alice", "Bob"];

// Array of a custom/object type
let mixedAllowed: (string | number)[] = ["Alice", 25, "Bob", 30]; 
// ^ This is a union type array — explained fully on Day 3
```

### Teach-it-back
"An array type is like saying 'this is a basket that can only hold apples.' If you try to throw in a banana, TypeScript stops you immediately."

---

## 4. Tuples

### What
A tuple is a special kind of array with a **fixed number of elements**, where **each position has its own specific type**.

### Why
Sometimes you don't want a flexible list — you want exactly two or three values, in a specific order, with specific types each. E.g., representing a coordinate `(x, y)`, or a key-value pair.

Plain arrays can't enforce position-specific types — tuples can.

### How

```typescript
// A tuple representing [name, age]
let person: [string, number] = ["Alice", 25];

person[0] = "Bob";    // ✅ string in position 0 — fine
person[1] = 30;       // ✅ number in position 1 — fine
person[0] = 40;       // ❌ Error: number not assignable to string

// Trying to add extra unplanned elements also throws an error
let coordinate: [number, number] = [10, 20];
coordinate.push(30); // ⚠️ Allowed by TS (a known tuple quirk with push), 
                      // but direct assignment like coordinate[2] = 30 would error

// Real-world example: returning multiple values from a function
function getUserInfo(): [string, number] {
  return ["Alice", 25];
}

const [userName, userAge] = getUserInfo(); // destructuring a tuple
```

### Array vs Tuple — quick comparison

| | Array | Tuple |
|---|---|---|
| Length | Flexible (any number of items) | Fixed |
| Types per position | Same type throughout | Can differ per position |
| Example | `number[]` → `[1, 2, 3, 4]` | `[string, number]` → `["Alice", 25]` |

### Teach-it-back
"A tuple is like a labeled drawer set: drawer 1 must always have socks (string), drawer 2 must always have coins (number). The order and type of each drawer is fixed."

---

## 5. Type Inference vs Explicit Typing

### What
- **Type inference**: TypeScript automatically figures out the type of a variable based on the value you assign — you don't have to write the type yourself.
- **Explicit typing**: You manually declare the type using a colon (`:`) syntax.

### Why
Writing types everywhere can get repetitive. TypeScript is smart enough to infer obvious types on its own, so you only need to be explicit when:
- The type isn't obvious from the initial value
- You're declaring a variable without assigning a value yet
- You want to be extra clear for readability (e.g., in function signatures)

### How

```typescript
// Type INFERENCE — TypeScript figures out the type automatically
let city = "New York";  
// Hover in your editor and it will show: let city: string
city = 5; // ❌ Error — TypeScript inferred "string" and now enforces it

// Type inference also works for function return types
function add(a: number, b: number) {
  return a + b; 
  // TypeScript infers the return type is "number" — you didn't have to write it
}

// Explicit typing — useful when there's no initial value
let userScore: number;
userScore = 95; // assigned later, but type was declared upfront

// Explicit typing in function parameters (parameters are NEVER inferred — always type them)
function multiply(a: number, b: number): number {
  return a * b;
}
```

### When should you let TypeScript infer vs. be explicit?

| Situation | Recommendation |
|---|---|
| Variable declared with an immediate value | Let TypeScript infer |
| Function parameters | Always explicit (TS cannot guess these) |
| Function return type | Optional, but good practice for public/shared functions |
| Variable declared without a value | Explicit (otherwise it defaults to `any`) |

### Teach-it-back
"TypeScript is like a smart assistant: if you hand it a value, it can usually guess the type itself. But if you don't hand it anything yet, you have to tell it what to expect."

---

## 6. Type Aliases (`type`)

### What
A type alias lets you give a **custom name** to a type — especially useful for complex or reused types (like object shapes, unions, etc.).

### Why
Without aliases, you'd repeat the same long type definitions over and over, which is error-prone and hard to maintain. A type alias lets you define a type **once** and reuse it everywhere — like creating a variable, but for a type instead of a value.

### How

```typescript
// Without a type alias — repetitive and messy
function printUser1(user: { name: string; age: number }) {
  console.log(user.name, user.age);
}
function printUser2(user: { name: string; age: number }) {
  console.log(user.name, user.age);
}

// WITH a type alias — clean and reusable
type User = {
  name: string;
  age: number;
};

function printUser(user: User) {
  console.log(user.name, user.age);
}

const newUser: User = { name: "Alice", age: 25 };

// Type aliases also work for simple types and unions
type ID = string | number; // explained fully Day 3, but a quick preview here

let userId: ID = 101;     // ✅ number
userId = "U101";          // ✅ string also allowed

// Type aliases for function signatures
type MathOperation = (a: number, b: number) => number;

const add: MathOperation = (a, b) => a + b;
const subtract: MathOperation = (a, b) => a - b;
```

### Type Alias vs Explicit Annotation

| Without alias | With alias |
|---|---|
| `let user: { name: string; age: number }` | `let user: User` |
| Have to repeat the full shape everywhere | Define once, reuse everywhere |

> **Note:** We'll compare `type` vs `interface` in depth on **Day 2**, since interfaces are introduced there. For now, just know `type` is a way to **name** a type for reuse.

### Teach-it-back
"A type alias is like a nickname. Instead of describing 'a person with a name and age' every single time, you just say 'User' and everyone knows what you mean."

---

## Quick Recap — Day 1 Checklist

By the end of today, you (and anyone you teach) should be able to confidently answer:

- [ ] What problem does TypeScript solve that plain JavaScript has?
- [ ] How does TypeScript code actually run in a browser (compilation process)?
- [ ] What's the difference between `any` and `unknown`?
- [ ] When would you use `void` vs `never`?
- [ ] How is a tuple different from a regular array?
- [ ] When does TypeScript infer a type vs require you to write one explicitly?
- [ ] Why use a type alias instead of repeating object shapes?

---

## Mini Practice Exercise (good for teaching/demoing)

```typescript
// 1. Create a type alias for a Book
type Book = {
  title: string;
  author: string;
  pages: number;
  isAvailable: boolean;
};

// 2. Create a tuple for [bookTitle, rating]
let bookReview: [string, number] = ["The Hobbit", 5];

// 3. Create an array of Book type
let library: Book[] = [
  { title: "1984", author: "Orwell", pages: 328, isAvailable: true },
  { title: "Dune", author: "Herbert", pages: 412, isAvailable: false }
];

// 4. Function with explicit param types, inferred return type
function getBookSummary(book: Book) {
  return `${book.title} by ${book.author} (${book.pages} pages)`;
}

console.log(getBookSummary(library[0]));
```

**Tomorrow (Day 2):** Objects, Interfaces, `interface` vs `type`, optional/readonly properties, and functions in depth.
