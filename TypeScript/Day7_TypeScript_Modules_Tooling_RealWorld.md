# Day 7 — Modules, Tooling & Real-World TypeScript
### Modules, Declaration Files, Third-Party Libraries, Enums, `tsconfig.json` Deep Dive, Real Projects, Best Practices

> **Recap of Day 6:** `keyof`, `typeof`, indexed access types, mapped types, conditional types, utility types, template literal types. Today is the final day — we move from "knowing the language" to "using it like a professional," covering modules, configuration, working with real libraries, and avoiding common mistakes.

---

## 1. Modules: `import` / `export`

### What
A module is simply **a file** that can export (share) values, functions, classes, or types, and import (use) values from other files. TypeScript fully supports JavaScript's ES Module syntax, and adds the ability to import/export **types** too.

### Why
As an app grows past a few hundred lines, keeping everything in one file becomes unmanageable. Modules let you split code into focused, reusable files, while still type-checking everything correctly across file boundaries — TypeScript tracks types across imports/exports just as strictly as within a single file.

### How

**Named exports/imports** (export multiple things by name):

```typescript
// mathUtils.ts
export function add(a: number, b: number): number {
  return a + b;
}

export function subtract(a: number, b: number): number {
  return a - b;
}

export const PI = 3.14159;
```

```typescript
// app.ts
import { add, subtract, PI } from "./mathUtils";

console.log(add(5, 3));      // 8
console.log(subtract(5, 3)); // 2
console.log(PI);             // 3.14159
```

**Default export** (one main thing per file):

```typescript
// User.ts
export default class User {
  constructor(public name: string) {}
}
```

```typescript
// app.ts
import User from "./User"; // no curly braces needed for default imports

const user = new User("Alice");
```

**Exporting/importing types specifically:**

```typescript
// types.ts
export interface Product {
  id: number;
  name: string;
}

export type Status = "active" | "inactive";
```

```typescript
// app.ts
import type { Product, Status } from "./types";
// "import type" makes it clear (to humans AND tools) this import is ONLY used for types,
// and gets fully removed from the compiled JavaScript output

const item: Product = { id: 1, name: "Laptop" };
```

**Renaming imports/exports (`as`):**

```typescript
import { add as sum } from "./mathUtils";
console.log(sum(2, 3)); // 5
```

**Namespace import (grab everything from a file under one name):**

```typescript
import * as MathUtils from "./mathUtils";
console.log(MathUtils.add(1, 2));
```

### Older alternative: Namespaces (`namespace` keyword)

Before ES Modules were standard, TypeScript had its own `namespace` feature to group related code:

```typescript
namespace Geometry {
  export function areaOfSquare(side: number): number {
    return side * side;
  }
}

console.log(Geometry.areaOfSquare(4)); // 16
```

> **Note:** Namespaces are largely **legacy** today. Modern TypeScript projects almost always use ES Modules (`import`/`export`) instead — namespaces are mainly seen in older codebases or specific declaration-file scenarios.

### Teach-it-back
"A module is a file acting like a shop — `export` puts items in the shop window for others to use, and `import` is walking into that shop and picking out exactly what you need, by name."

---

## 2. Declaration Files (`.d.ts`) and `declare`

### What
A declaration file (ending in `.d.ts`) contains **only type information** — no actual runnable code. It tells TypeScript "here's the shape of this code," even if the real implementation is plain JavaScript (or comes from somewhere TypeScript can't see directly).

### Why
TypeScript can only type-check code it understands the shape of. When you use plain JavaScript libraries, or define global variables that exist outside your normal files (like a script tag loading a library into `window`), TypeScript needs a way to know what's available — without that, you'd either get errors everywhere or have to use `any` constantly.

### How

**A simple custom `.d.ts` file:**

```typescript
// math-helpers.d.ts
declare function double(value: number): number;
declare const VERSION: string;
```

This tells TypeScript "trust me, a function called `double` and a constant called `VERSION` exist somewhere at runtime" — without TypeScript needing to see their actual implementation.

```typescript
// app.ts
double(5);      // ✅ TypeScript allows this — it trusts the declaration
console.log(VERSION);
```

**Declaring types for a global variable** (e.g., injected by a `<script>` tag):

```typescript
// globals.d.ts
declare global {
  interface Window {
    myAppConfig: {
      apiUrl: string;
    };
  }
}

// Now anywhere in your project:
console.log(window.myAppConfig.apiUrl); // ✅ TypeScript knows this property exists
```

**Auto-generating declaration files from your own TypeScript code** (useful when publishing a library):

```json
// tsconfig.json
{
  "compilerOptions": {
    "declaration": true // generates matching .d.ts files alongside your compiled .js files
  }
}
```

### Teach-it-back
"A `.d.ts` file is like a furniture assembly diagram with no actual furniture inside the box — it just describes the shape of what's supposed to be there, so TypeScript knows what to expect, even when it can't see the real code."

---

## 3. Working with Third-Party JS Libraries (`@types`)

### What
Many popular JavaScript libraries (like `lodash`, `express`, `react`) are written in plain JavaScript and don't include TypeScript types themselves. The community maintains a massive collection of type definitions for these libraries, published as separate npm packages under the `@types` scope (this collection is called **DefinitelyTyped**).

### Why
Without type definitions, importing a JS library into TypeScript would either throw errors (TypeScript can't verify an unknown shape) or force you to type everything as `any`, losing all the safety TypeScript provides. Installing the matching `@types` package gives you full autocomplete and type-checking for libraries that weren't originally written in TypeScript.

### How

```bash
# Installing a library and its types separately
npm install lodash
npm install --save-dev @types/lodash
```

```typescript
import _ from "lodash";

const numbers = [1, 2, 3, 4, 5];
const chunked = _.chunk(numbers, 2); // ✅ fully typed, autocomplete works
console.log(chunked); // [[1, 2], [3, 4], [5]]
```

**Some libraries include their own types already** (no separate `@types` package needed) — you can tell by checking if the package itself ships a `.d.ts` file, or just trying to import it; TypeScript will warn you if types are missing.

**If absolutely no types exist anywhere**, you can declare a quick fallback yourself:

```typescript
// shims.d.ts
declare module "some-untyped-library" {
  export function doSomething(value: string): void;
}
```

This is a last resort — it tells TypeScript "trust me" without real type safety, similar to using `any`.

### Teach-it-back
"`@types` packages are like subtitle files for a movie that wasn't originally made with subtitles. The movie (the library) plays exactly the same, but now TypeScript can 'read along' and understand what's happening."

---

## 4. Enums (`enum` vs `const enum`)

### What
An `enum` ("enumerated type") is a way to define a set of **named constants** — similar in spirit to literal unions (Day 3), but represented as an actual object at runtime, with some extra capabilities.

### Why
Sometimes you want named constants that are clearly grouped together logically, can be referenced by name throughout your code, and (in some cases) need actual runtime values — not just compile-time-only literal types.

### How

**Numeric enum** (default — auto-assigns increasing numbers):

```typescript
enum Direction {
  Up,    // 0
  Down,  // 1
  Left,  // 2
  Right  // 3
}

let move: Direction = Direction.Up;
console.log(move); // 0
console.log(Direction[0]); // "Up" — numeric enums support reverse lookup
```

**String enum** (more readable, no reverse lookup, but clearer values):

```typescript
enum Status {
  Pending = "PENDING",
  Shipped = "SHIPPED",
  Delivered = "DELIVERED"
}

let orderStatus: Status = Status.Shipped;
console.log(orderStatus); // "SHIPPED"
```

**`const enum`** — same syntax, but fully removed/inlined at compile time (no runtime object created at all), resulting in smaller, faster compiled output:

```typescript
const enum Color {
  Red,
  Green,
  Blue
}

let favoriteColor = Color.Green;
// After compiling, this becomes: let favoriteColor = 1;
// The "Color" object itself never exists in the final JavaScript
```

### Enum vs Literal Union — Quick Comparison

| | `enum` | Literal Union (`"a" \| "b"`) |
|---|---|---|
| Exists at runtime as a real object | ✅ Yes (except `const enum`) | ❌ No — purely a compile-time type |
| Reverse lookup (`Enum[0]` → name) | ✅ Yes (numeric enums) | ❌ No |
| Bundle size impact | Slightly larger | None — disappears after compiling |
| Modern recommendation | Use when you need a real runtime object | Often preferred for simplicity in modern TS |

> **Note:** Many modern TypeScript style guides (including parts of the TypeScript team itself) now lean toward **literal unions** over `enum` for simple cases, since they add zero runtime overhead. `enum` is still very common and useful, especially in older codebases or when you genuinely need the runtime object.

### Teach-it-back
"An enum is like a labeled set of light switches on a wall panel — `Direction.Up`, `Direction.Down`, etc. — each with a real, physical switch behind it. A literal union is more like a sticky note listing the same names, but with no real switch behind it at all — useful for the same purpose, just lighter weight."

---

## 5. `tsconfig.json` Deep Dive

### What
`tsconfig.json` is the configuration file that controls how the TypeScript compiler behaves — what JavaScript version to output, how strict type-checking should be, which files to include, and more.

### Why
Different projects need different settings — a Node.js backend compiles differently than a React frontend; a strict, safety-critical project wants maximum type-checking, while a quick prototype might relax some rules. Understanding the key options lets you configure TypeScript correctly for any project instead of just copying a random config blindly.

### How

```json
{
  "compilerOptions": {
    "target": "ES2020",            // Which JS version the compiled output should support
    "module": "commonjs",          // Module system: "commonjs" (Node), "ESNext" (modern bundlers)
    "outDir": "./dist",            // Where compiled .js files go
    "rootDir": "./src",            // Where your .ts source files live
    "strict": true,                // Enables ALL strict type-checking options at once (recommended!)
    "esModuleInterop": true,       // Improves compatibility when importing CommonJS libraries
    "skipLibCheck": true,          // Skips type-checking of .d.ts files (faster builds)
    "declaration": true,           // Generates .d.ts files alongside compiled output
    "sourceMap": true,             // Generates .map files for easier debugging
    "noImplicitAny": true,         // Errors if a type can't be inferred and defaults to "any"
    "strictNullChecks": true,      // null and undefined are NOT assignable to other types by default
    "noUnusedLocals": true,        // Errors on declared-but-unused local variables
    "noUnusedParameters": true     // Errors on unused function parameters
  },
  "include": ["src/**/*"],          // Which files to compile
  "exclude": ["node_modules", "dist"] // Which files/folders to ignore
}
```

### Key options explained individually

**`strict: true`** is actually a shortcut that enables many sub-options at once, including `strictNullChecks`, `noImplicitAny`, `strictFunctionTypes`, and others. **Always turn this on for new projects** — it's the single biggest factor in how much safety TypeScript actually gives you.

```typescript
// Without strictNullChecks:
let name: string = null; // ✅ allowed (dangerous!)

// With strictNullChecks: true
let name: string = null; // ❌ Error: null is not assignable to string
let name2: string | null = null; // ✅ you must explicitly allow null
```

**`target`** affects what JavaScript syntax features are available in the output (e.g., targeting `ES5` would convert modern arrow functions into old-style `function` keywords for older browser support).

**`module`** affects the import/export syntax style in the compiled output — `commonjs` produces `require()`/`module.exports` (classic Node.js), while `ESNext` keeps modern `import`/`export` syntax (used with modern bundlers like Vite or webpack).

### Teach-it-back
"`tsconfig.json` is like the settings panel on a camera. `strict: true` is turning on every safety feature at once — recommended for almost everyone. The other settings (`target`, `module`, etc.) adjust *how* the final photo (compiled JavaScript) comes out, depending on which 'device' (browser, Node.js version) needs to display it."

---

## 6. TypeScript with a Real Project (Node/React Basics)

### What
This section shows how the concepts from all 7 days come together in two extremely common real-world setups: a small Node.js backend, and a small React component.

### Why
Knowing individual features is one thing — seeing them combined in a realistic file is what actually prepares you to work on (or teach from) a genuine codebase.

### How

**A small Node.js + Express-style example (conceptual, simplified):**

```typescript
// types.ts
export interface User {
  id: number;
  name: string;
  email: string;
}

// userService.ts
import { User } from "./types";

class UserService {
  private users: User[] = [];

  addUser(user: Omit<User, "id">): User {
    const newUser: User = { id: this.users.length + 1, ...user };
    this.users.push(newUser);
    return newUser;
  }

  findUser(id: number): User | undefined {
    return this.users.find((u) => u.id === id);
  }

  getAllUsers(): User[] {
    return this.users;
  }
}

export default new UserService();
```

```typescript
// app.ts
import userService from "./userService";

const newUser = userService.addUser({ name: "Alice", email: "alice@example.com" });
console.log(newUser); // { id: 1, name: "Alice", email: "alice@example.com" }
console.log(userService.findUser(1));
```

**A small React component example (conceptual — using generics, interfaces, and union types together):**

```typescript
// Props interface — TypeScript + React's most common pattern
interface UserCardProps {
  name: string;
  age: number;
  status: "active" | "inactive"; // literal union from Day 3
}

function UserCard({ name, age, status }: UserCardProps) {
  return (
    <div>
      <h3>{name} ({age})</h3>
      <span>{status === "active" ? "🟢 Online" : "⚪ Offline"}</span>
    </div>
  );
}

// Generic component example
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
}

function List<T>({ items, renderItem }: ListProps<T>) {
  return <ul>{items.map((item, index) => <li key={index}>{renderItem(item)}</li>)}</ul>;
}
```

### Teach-it-back
"A real project is just all the pieces you've learned — interfaces for shapes, generics for reusable logic, unions for limited states — working together inside actual files that import from each other, instead of standalone snippets."

---

## 7. Common Pitfalls and Best Practices

### What
A summary of mistakes that even experienced developers make when starting with TypeScript, and the recommended habits that avoid them.

### Why
Knowing the *features* of TypeScript isn't the same as using them *well*. These are the practical lessons that separate "technically using TypeScript" from "actually getting its benefits."

### How — Pitfalls & Fixes

**Pitfall 1 — Overusing `any`:**
```typescript
// ❌ Defeats the entire purpose of TypeScript
function process(data: any) { ... }

// ✅ Use "unknown" + narrowing, or a proper specific/generic type
function process(data: unknown) {
  if (typeof data === "string") { ... }
}
```

**Pitfall 2 — Not enabling `strict` mode:**
```json
// ❌ Leaving this off (or false) silently allows many unsafe patterns
{ "compilerOptions": { "strict": false } }

// ✅ Always start new projects with:
{ "compilerOptions": { "strict": true } }
```

**Pitfall 3 — Type assertions (`as`) used to silence errors instead of fixing them:**
```typescript
// ❌ Forcing TypeScript to "trust you" when you might be wrong
const value = someApiResponse as User; // no actual verification happened

// ✅ Use a type guard (Day 3) or proper validation library to ACTUALLY verify the shape
if (isUser(someApiResponse)) {
  const value = someApiResponse; // safely narrowed, not just asserted
}
```

**Pitfall 4 — Repeating object shapes instead of reusing types:**
```typescript
// ❌ Same shape typed out 3 separate times across a file
function a(user: { name: string; age: number }) {}
function b(user: { name: string; age: number }) {}

// ✅ Define once, reuse everywhere (Day 1 & 2)
type User = { name: string; age: number };
function a(user: User) {}
function b(user: User) {}
```

**Pitfall 5 — Ignoring function return types on public/shared functions:**
```typescript
// ⚠️ Works (inferred), but unclear at a glance for anyone reading/reusing this function
export function calculateTotal(price: number, tax: number) {
  return price + tax;
}

// ✅ Explicit return types on exported/shared functions improve readability and catch mistakes early
export function calculateTotal(price: number, tax: number): number {
  return price + tax;
}
```

**Pitfall 6 — Misusing non-null assertion (`!`) instead of proper null checks:**
```typescript
// ❌ Tells TypeScript "trust me, this is never null" — dangerous if you're wrong
function greet(name: string | null) {
  console.log(name!.toUpperCase()); // crashes at runtime if name actually IS null
}

// ✅ Use proper narrowing instead (Day 3)
function greet(name: string | null) {
  if (name) {
    console.log(name.toUpperCase());
  }
}
```

### Best Practices Summary

| Do | Avoid |
|---|---|
| Enable `strict: true` from day one | Leaving strict mode off |
| Use `unknown` + narrowing for uncertain data | Using `any` everywhere |
| Reuse `type`/`interface` definitions | Repeating object shapes inline |
| Use type guards to verify shapes | Using `as` to force/assume a type |
| Explicit return types on public functions | Relying on inference for shared/exported code |
| Use discriminated unions for related variants | Manually checking properties with many `if` chains |

### Teach-it-back
"Best practices in TypeScript boil down to one idea: don't fight the type system or silence it — work *with* it. Every shortcut that bypasses type-checking (`any`, forced `as`, `!`) just moves the bug from 'caught now, at compile time' to 'discovered later, at runtime,' which is the exact problem TypeScript was built to prevent."

---

## Quick Recap — Day 7 Checklist (Final Day!)

By the end of today, you (and anyone you teach) should be able to confidently answer:

- [ ] What's the difference between a named export and a default export?
- [ ] What is a `.d.ts` file for, and when would you write one yourself?
- [ ] Why do some npm packages need a separate `@types` package installed?
- [ ] What's the difference between a regular `enum`, a `const enum`, and a literal union?
- [ ] What does `strict: true` in `tsconfig.json` actually do, and why is it recommended?
- [ ] Can you point to at least one pitfall (like overusing `any` or `as`) and explain why it's risky?

---

## Full 7-Day Mastery Recap

| Day | Core Theme |
|---|---|
| 1 | Foundations — why TypeScript, basic types, arrays, tuples, inference, type aliases |
| 2 | Objects, interfaces, `interface` vs `type`, optional/readonly, functions |
| 3 | Unions, intersections, literals, narrowing, discriminated unions, type guards |
| 4 | Classes, access modifiers, getters/setters, abstract classes, static members |
| 5 | Generics — functions, interfaces, classes, constraints, defaults |
| 6 | Advanced types — `keyof`, `typeof`, mapped/conditional types, utility types |
| 7 | Modules, declarations, third-party types, enums, `tsconfig`, real projects, best practices |

**Congratulations — that's the complete 7-day TypeScript curriculum, covering every topic from the original plan with What/Why/How explanations, working code, and teach-it-back summaries for each one.**

If you'd like, a natural next step would be a **combined final practice project** (e.g., a small typed REST API or React app) that pulls together concepts from all 7 days into one cohesive build — just ask.
