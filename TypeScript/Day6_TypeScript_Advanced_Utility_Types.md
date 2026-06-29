# Day 6 — Advanced Types & Utility Types
### `keyof`, `typeof`, Indexed Access Types, Mapped Types, Conditional Types, Built-in Utility Types, Template Literal Types

> **Recap of Day 5:** Generics — functions, interfaces, classes, constraints, defaults, and utility patterns. Today combines generics with TypeScript's type-level "operators" — this is where TypeScript stops feeling like "JavaScript with labels" and starts feeling like its own powerful language for describing types.

---

## 1. `keyof` Operator

### What
`keyof` takes an object type and produces a **union of its property names** (as string literal types).

### Why
Sometimes you need to restrict a value to "must be one of this object's actual keys" — for example, a function that safely accesses a property by name. Without `keyof`, you'd have to use `string`, which is far too loose (it would allow keys that don't actually exist on the object).

### How

```typescript
interface Person {
  name: string;
  age: number;
  email: string;
}

// keyof Person produces: "name" | "age" | "email"
type PersonKeys = keyof Person;

let key: PersonKeys;
key = "name";  // ✅
key = "age";   // ✅
key = "phone"; // ❌ Error: "phone" is not a key of Person

// A practical use — safely accessing a property by key
function getProperty(person: Person, key: keyof Person) {
  return person[key];
}

const alice: Person = { name: "Alice", age: 25, email: "alice@example.com" };
console.log(getProperty(alice, "name"));  // ✅ "Alice"
console.log(getProperty(alice, "phone")); // ❌ Error: "phone" not assignable to keyof Person
```

### Teach-it-back
"`keyof` is like asking an object 'what are all your label names?' and getting back a list you can use as a type — so you can only ever refer to labels that actually exist."

---

## 2. `typeof` Operator (in the type system)

### What
You already know `typeof` from JavaScript as a runtime check (`typeof value === "string"`). In TypeScript's **type system**, `typeof` does something different: it extracts the **type** of a variable or value, so you can reuse that type elsewhere.

### Why
Sometimes you already have a real value (like a config object) and you want a type that exactly matches its shape — without manually rewriting the entire shape as a separate `type` or `interface`. `typeof` lets you derive the type directly from the value itself, keeping a single source of truth.

### How

```typescript
const config = {
  appName: "MyApp",
  version: 1.0,
  isProduction: false
};

// typeof config -> { appName: string; version: number; isProduction: boolean }
type Config = typeof config;

function printConfig(cfg: Config) {
  console.log(cfg.appName, cfg.version, cfg.isProduction);
}

printConfig(config); // ✅ matches exactly

// A common pairing: typeof + keyof together
type ConfigKeys = keyof typeof config; // "appName" | "version" | "isProduction"
```

**Useful with functions too:**

```typescript
function add(a: number, b: number): number {
  return a + b;
}

type AddFunction = typeof add; // (a: number, b: number) => number
```

### `typeof` — Runtime vs Type-level (don't confuse these!)

| Context | Example | Purpose |
|---|---|---|
| Runtime (JavaScript) | `if (typeof value === "string")` | Checks a value's type while the program runs |
| Type-level (TypeScript only) | `type Config = typeof config` | Extracts a type from a value, used only at compile time |

### Teach-it-back
"Runtime `typeof` asks 'what kind of thing is sitting in this box right now?' while compile-time `typeof` asks 'what shape did this box's blueprint use, so I can reuse that exact blueprint elsewhere?'"

---

## 3. Indexed Access Types

### What
Indexed access types let you pull out the **type of a specific property** from another type, using bracket syntax: `SomeType["propertyName"]`.

### Why
Instead of duplicating a property's type by hand (which can drift out of sync if the original type changes), you can reference it directly — guaranteeing it always stays accurate, even if the original type is updated later.

### How

```typescript
interface Person {
  name: string;
  age: number;
  address: {
    city: string;
    zip: string;
  };
}

// Extract the type of a single property
type Age = Person["age"];          // number
type Address = Person["address"];  // { city: string; zip: string }
type City = Person["address"]["city"]; // string — chaining works too!

// Using keyof + indexed access together to get a union of ALL property types
type PersonValues = Person[keyof Person]; // string | number | { city: string; zip: string }
```

**A practical use — extracting an array's element type:**

```typescript
type Users = { name: string; age: number }[];
type SingleUser = Users[number]; // { name: string; age: number } — "number" here means "any index"
```

### Teach-it-back
"Indexed access types are like saying 'show me what's inside drawer labeled `age`' instead of redescribing the drawer's contents by hand. If the drawer's actual contents change later, your reference automatically stays correct."

---

## 4. Mapped Types

### What
A mapped type **builds a new type by transforming every property** of an existing type, using a `for...in`-like syntax: `[K in keyof T]`.

### Why
Often you want a variation of an existing type — e.g., the same shape but with every property made optional, or read-only, or turned into a different type — without manually retyping every single property by hand (which is repetitive and error-prone if the original type changes).

### How

```typescript
interface Person {
  name: string;
  age: number;
  email: string;
}

// Make every property optional
type PartialPerson = {
  [K in keyof Person]?: Person[K];
};
// equivalent to: { name?: string; age?: number; email?: string }

// Make every property readonly
type ReadonlyPerson = {
  readonly [K in keyof Person]: Person[K];
};

// Transform every property's TYPE (e.g., turn everything into a string)
type StringifiedPerson = {
  [K in keyof Person]: string;
};
// equivalent to: { name: string; age: string; email: string }
```

**Mapped types are also how TypeScript's built-in utility types (Section 6) are actually implemented internally** — e.g., `Partial<T>` is essentially:

```typescript
type Partial<T> = {
  [K in keyof T]?: T[K];
};
```

### Teach-it-back
"A mapped type is like a factory machine that walks through every drawer in a cabinet (every property in a type) and applies the same modification to each one — making them all optional, all read-only, or all converted to a new type — without you touching each drawer individually."

---

## 5. Conditional Types

### What
A conditional type chooses between two types based on a condition, using syntax that mirrors JavaScript's ternary operator: `T extends U ? X : Y`. If `T` is assignable to (i.e., "matches" or "extends") `U`, the result is `X` — otherwise, it's `Y`.

### Why
Sometimes the correct type for something depends entirely on another type — e.g., "if this is an array, give me its element type; otherwise, give me the type itself." Conditional types let you express this kind of type-level logic, instead of being stuck with one fixed type regardless of input.

### How

```typescript
// Basic conditional type
type IsString<T> = T extends string ? "yes" : "no";

type A = IsString<string>;  // "yes"
type B = IsString<number>;  // "no"

// A more practical conditional type — extracting array element types
type ElementType<T> = T extends (infer U)[] ? U : T;

type Test1 = ElementType<string[]>; // string
type Test2 = ElementType<number>;   // number (not an array, so it stays as-is)
```

**`infer` keyword** — used inside conditional types to "capture" a type for reuse:

```typescript
// Extract a function's return type
type ReturnTypeOf<T> = T extends (...args: any[]) => infer R ? R : never;

function getUser() {
  return { name: "Alice", age: 25 };
}

type User = ReturnTypeOf<typeof getUser>; // { name: string; age: number }
```

**Distributive conditional types** (applied automatically across a union):

```typescript
type ToArray<T> = T extends any ? T[] : never;

type Result = ToArray<string | number>; // string[] | number[]
```

### Teach-it-back
"A conditional type is an 'if-statement' for types: 'IF this type fits that shape, THEN use this other type; otherwise use a different one.' It lets you write types that adapt their result based on what's plugged in, exactly like a function adapts its output based on its input."

---

## 6. Built-in Utility Types

### What
TypeScript ships with a set of ready-made generic types — built using the mapped/conditional type techniques above — that solve extremely common type-transformation needs, so you don't have to write them yourself every time.

### Why
The patterns these utility types solve (making properties optional, picking a subset of fields, excluding fields, etc.) come up constantly in real projects. TypeScript provides them out of the box, fully tested and optimized, so you can focus on your actual logic instead of reinventing these transformations.

### How

```typescript
interface Person {
  name: string;
  age: number;
  email: string;
}

// Partial<T> — makes ALL properties optional
type PartialPerson = Partial<Person>;
// { name?: string; age?: number; email?: string }
const update: PartialPerson = { age: 26 }; // ✅ only updating one field is fine

// Required<T> — makes ALL properties required (opposite of Partial)
interface Settings {
  theme?: string;
  fontSize?: number;
}
type RequiredSettings = Required<Settings>;
// { theme: string; fontSize: number } — now optional fields are forced

// Readonly<T> — makes ALL properties readonly
type ReadonlyPerson = Readonly<Person>;
const lockedPerson: ReadonlyPerson = { name: "Alice", age: 25, email: "a@a.com" };
lockedPerson.age = 26; // ❌ Error: read-only property

// Pick<T, Keys> — creates a type with ONLY the specified properties
type PersonNameOnly = Pick<Person, "name" | "email">;
// { name: string; email: string }

// Omit<T, Keys> — creates a type with all properties EXCEPT the specified ones
type PersonWithoutEmail = Omit<Person, "email">;
// { name: string; age: number }

// Record<Keys, ValueType> — creates an object type with specific keys, all mapped to one value type
type Roles = "admin" | "editor" | "viewer";
type RolePermissions = Record<Roles, boolean>;
// { admin: boolean; editor: boolean; viewer: boolean }

const permissions: RolePermissions = { admin: true, editor: true, viewer: false };

// Exclude<T, U> — removes types from a union that match U
type Status = "pending" | "shipped" | "delivered" | "cancelled";
type ActiveStatus = Exclude<Status, "cancelled">;
// "pending" | "shipped" | "delivered"

// Extract<T, U> — keeps only the types from a union that match U
type FinishedStatus = Extract<Status, "delivered" | "cancelled">;
// "delivered" | "cancelled"
```

### Quick Reference Table

| Utility Type | What it does |
|---|---|
| `Partial<T>` | Makes all properties optional |
| `Required<T>` | Makes all properties required |
| `Readonly<T>` | Makes all properties readonly |
| `Pick<T, K>` | Keeps only the specified keys |
| `Omit<T, K>` | Removes the specified keys |
| `Record<K, V>` | Builds an object type from a union of keys, all mapped to one value type |
| `Exclude<T, U>` | Removes matching members from a union |
| `Extract<T, U>` | Keeps only matching members from a union |

### Teach-it-back
"Utility types are like pre-made kitchen tools instead of building your own from raw metal every time. `Partial` is a tool that makes everything optional. `Pick` is a tool that grabs only the ingredients you name. `Omit` is the opposite — it grabs everything *except* what you name."

---

## 7. Template Literal Types

### What
Template literal types let you build new **string literal types** by combining other types, using the same backtick syntax as JavaScript template literals (`` `${...}` ``), but at the type level.

### Why
Sometimes valid string values follow a predictable pattern based on other types — like CSS classes (`"text-red"`, `"text-blue"`), event names (`"onClick"`, `"onHover"`), or API routes. Template literal types let you generate and enforce these patterns automatically, instead of manually listing every single valid combination.

### How

```typescript
type Color = "red" | "blue" | "green";
type Size = "small" | "medium" | "large";

// Combine two unions into every possible string combination
type ClassName = `${Size}-${Color}`;
// "small-red" | "small-blue" | "small-green" | "medium-red" | ... (9 total combinations)

let myClass: ClassName;
myClass = "small-red";   // ✅
myClass = "huge-purple"; // ❌ Error: not a valid combination
```

**A practical real-world example — generating event handler names automatically:**

```typescript
type EventName = "click" | "hover" | "focus";
type HandlerName = `on${Capitalize<EventName>}`;
// "onClick" | "onHover" | "onFocus"

interface ComponentProps {
  onClick?: () => void;
  onHover?: () => void;
  onFocus?: () => void;
}
```

**Built-in string manipulation types often paired with template literals:**

| Type | Effect | Example |
|---|---|---|
| `Uppercase<T>` | Converts to UPPERCASE | `Uppercase<"abc">` → `"ABC"` |
| `Lowercase<T>` | Converts to lowercase | `Lowercase<"ABC">` → `"abc"` |
| `Capitalize<T>` | Capitalizes first letter | `Capitalize<"abc">` → `"Abc"` |
| `Uncapitalize<T>` | Lowercases first letter | `Uncapitalize<"Abc">` → `"abc"` |

### Teach-it-back
"Template literal types are like a label printer that automatically generates every valid combination of stickers from two lists you give it — 'small-red,' 'medium-blue,' etc. — and refuses to print any sticker that wasn't one of those exact valid combinations."

---

## Quick Recap — Day 6 Checklist

By the end of today, you (and anyone you teach) should be able to confidently answer:

- [ ] What does `keyof` actually produce, and why is it safer than typing a key as `string`?
- [ ] What's the difference between `typeof` at runtime (JavaScript) and `typeof` in the type system (TypeScript)?
- [ ] How does an indexed access type let you "look up" a property's type from another type?
- [ ] What is a mapped type, and how does `[K in keyof T]` work?
- [ ] What does a conditional type do, and what role does `infer` play inside one?
- [ ] Can you explain what `Partial`, `Pick`, `Omit`, and `Record` each do, from memory?
- [ ] How do template literal types generate valid string combinations from unions?

---

## Mini Practice Exercise (good for teaching/demoing)

```typescript
// 1. Base interface
interface Product {
  id: number;
  name: string;
  price: number;
  category: string;
}

// 2. keyof + indexed access
type ProductKey = keyof Product;          // "id" | "name" | "price" | "category"
type PriceType = Product["price"];         // number

// 3. Built-in utility types in a real pattern (update form pattern)
type ProductUpdate = Partial<Omit<Product, "id">>;
// { name?: string; price?: number; category?: string }  -- id excluded, rest optional

function updateProduct(id: number, updates: ProductUpdate) {
  console.log(`Updating product ${id} with`, updates);
}

updateProduct(1, { price: 999 }); // ✅ only updating price is fine

// 4. Mapped type — make every property a function that validates it
type Validators<T> = {
  [K in keyof T]: (value: T[K]) => boolean;
};

const productValidators: Validators<Product> = {
  id: (value) => value > 0,
  name: (value) => value.length > 0,
  price: (value) => value >= 0,
  category: (value) => value.length > 0
};

// 5. Conditional type + infer
type UnwrapPromise<T> = T extends Promise<infer U> ? U : T;

async function fetchProduct(): Promise<Product> {
  return { id: 1, name: "Laptop", price: 1000, category: "Electronics" };
}

type FetchedProduct = UnwrapPromise<ReturnType<typeof fetchProduct>>; // Product

// 6. Template literal type
type ProductCategory = "electronics" | "clothing" | "books";
type CategoryRoute = `/products/${ProductCategory}`;
// "/products/electronics" | "/products/clothing" | "/products/books"

const route: CategoryRoute = "/products/electronics"; // ✅
```

**Tomorrow (Day 7 — Final Day):** Modules (`import`/`export`), declaration files (`.d.ts`), working with third-party JS libraries (`@types`), enums, `tsconfig.json` deep dive, real-world project structure, and common pitfalls/best practices.
