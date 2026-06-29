# Day 5 — Generics
### What Are Generics, Generic Functions, Generic Interfaces/Types/Classes, Constraints, Default Types, Utility Patterns

> **Recap of Day 4:** Classes, access modifiers, readonly, getters/setters, abstract classes, implementing interfaces, static members. Today is widely considered the **most important day** in mastering TypeScript — generics unlock writing flexible, reusable code without losing type safety.

---

## 1. What Are Generics, and Why We Need Them

### What
A generic is a **placeholder for a type** that gets filled in later, when the function/class/interface is actually used. Think of it as a "type variable" — instead of hardcoding one specific type, you let the caller decide what type to plug in, while TypeScript still keeps everything fully type-checked.

### Why
Without generics, you're stuck choosing between two bad options:

**Option 1 — write one function per type (repetitive, doesn't scale):**

```typescript
function wrapStringInArray(value: string): string[] {
  return [value];
}

function wrapNumberInArray(value: number): number[] {
  return [value];
}
// ...and you'd need a new version for every type you ever use. Unmanageable.
```

**Option 2 — use `any` (works for every type, but loses all safety):**

```typescript
function wrapInArray(value: any): any[] {
  return [value];
}

const result = wrapInArray("hello");
result[0].toUpperCase(); // TypeScript has NO IDEA this is a string — no autocomplete, no error checking
```

**Generics solve this** — one function, works with any type, AND keeps full type safety:

```typescript
function wrapInArray<T>(value: T): T[] {
  return [value];
}

const strings = wrapInArray("hello");  // TypeScript infers T = string
const numbers = wrapInArray(42);        // TypeScript infers T = number

strings[0].toUpperCase(); // ✅ TypeScript KNOWS this is a string — full autocomplete & safety
numbers[0].toFixed(2);    // ✅ TypeScript KNOWS this is a number
```

`T` here is just a name (by convention, a single capital letter — `T` for "Type") — it's a variable, but for types instead of values.

### Teach-it-back
"A generic is like a vending machine slot labeled 'insert item type here.' The machine (function) works the exact same way no matter what you put in — but it remembers exactly what you inserted, so what comes out is the same type, not a mystery box like with `any`."

---

## 2. Generic Functions

### What
A generic function is a function that declares one or more type placeholders (like `<T>`) right after its name, which can then be used anywhere inside the function's parameters and return type.

### Why
This lets a single function work correctly — with full type-checking — across many different types, instead of writing a separate version for each one, or giving up type safety with `any`.

### How

```typescript
// Single generic type parameter
function getFirstElement<T>(arr: T[]): T {
  return arr[0];
}

const firstNum = getFirstElement([1, 2, 3]);         // T inferred as number
const firstStr = getFirstElement(["a", "b", "c"]);   // T inferred as string

console.log(firstNum); // 1
console.log(firstStr); // "a"
```

**Multiple generic type parameters:**

```typescript
function pair<A, B>(first: A, second: B): [A, B] {
  return [first, second];
}

const result = pair("Alice", 25); // A = string, B = number
console.log(result); // ["Alice", 25]
```

**Explicitly specifying the type (instead of letting TypeScript infer it):**

```typescript
function createEmptyArray<T>(): T[] {
  return [];
}

const numArray = createEmptyArray<number>(); // explicitly tell TS what T should be
const strArray = createEmptyArray<string>();
```

**A practical real-world example — a generic "filter" function:**

```typescript
function filterArray<T>(arr: T[], predicate: (item: T) => boolean): T[] {
  return arr.filter(predicate);
}

const evenNumbers = filterArray([1, 2, 3, 4, 5, 6], (n) => n % 2 === 0);
console.log(evenNumbers); // [2, 4, 6]

const longNames = filterArray(["Al", "Alexandra", "Bo"], (name) => name.length > 2);
console.log(longNames); // ["Alexandra"]
```

### Teach-it-back
"A generic function is like a photocopier that adapts to whatever paper size you feed it — A4 in, A4 out; Letter in, Letter out — but it never mixes them up or loses track of the size, the way `any` would."

---

## 3. Generic Interfaces and Types

### What
Just like functions, `interface` and `type` definitions can also accept type parameters — letting you describe a *shape* that works generically across different data types.

### Why
Many real data structures are "the same shape, different contents" — a box that holds a string, a box that holds a number, a box that holds a User object. Rather than writing a new interface for every possible content type, a generic interface/type describes the shape once, parameterized by whatever it holds.

### How

**Generic interface:**

```typescript
interface Box<T> {
  contents: T;
}

const stringBox: Box<string> = { contents: "Hello" };
const numberBox: Box<number> = { contents: 42 };

interface User {
  name: string;
}

const userBox: Box<User> = { contents: { name: "Alice" } };
```

**Generic type alias:**

```typescript
type ApiResponse<T> = {
  data: T;
  success: boolean;
  errorMessage?: string;
};

const userResponse: ApiResponse<User> = {
  data: { name: "Alice" },
  success: true
};

const numberResponse: ApiResponse<number> = {
  data: 100,
  success: true
};
```

**A practical real-world example — a generic API result wrapper used across an entire app:**

```typescript
type Product = { id: number; name: string; price: number };

function fetchData<T>(url: string): ApiResponse<T> {
  // (pretend this actually calls an API)
  return { data: {} as T, success: true };
}

const productResult = fetchData<Product>("/api/products/1");
console.log(productResult.data.name); // ✅ TypeScript knows "data" is a Product
```

### Teach-it-back
"A generic interface is like a labeled storage container template — 'Box of ___.' You decide what goes in the blank each time you use it: Box of Strings, Box of Numbers, Box of Users — same container design, different contents, and TypeScript always remembers which is which."

---

## 4. Generic Classes

### What
A class can also declare a type parameter (e.g., `class Container<T>`), allowing the entire class — its properties and methods — to work generically with whatever type is provided when an instance is created.

### Why
Common data-structure classes (stacks, queues, collections, caches) need to work with many different types of stored data, while keeping every method on that class fully type-safe for the type actually stored — generics make this possible without duplicating the class per type.

### How

```typescript
class Box<T> {
  private contents: T;

  constructor(value: T) {
    this.contents = value;
  }

  getContents(): T {
    return this.contents;
  }

  setContents(value: T): void {
    this.contents = value;
  }
}

const stringBox = new Box<string>("Hello");
console.log(stringBox.getContents()); // "Hello"
stringBox.setContents(42); // ❌ Error: number not assignable to string

const numberBox = new Box<number>(100);
console.log(numberBox.getContents()); // 100
```

**A practical real-world example — a generic Stack class:**

```typescript
class Stack<T> {
  private items: T[] = [];

  push(item: T): void {
    this.items.push(item);
  }

  pop(): T | undefined {
    return this.items.pop();
  }

  peek(): T | undefined {
    return this.items[this.items.length - 1];
  }

  isEmpty(): boolean {
    return this.items.length === 0;
  }
}

const numberStack = new Stack<number>();
numberStack.push(1);
numberStack.push(2);
console.log(numberStack.pop()); // 2

const stringStack = new Stack<string>();
stringStack.push("a");
stringStack.push("b");
console.log(stringStack.peek()); // "b"
```

### Teach-it-back
"A generic class is a reusable container *factory*. `Stack<number>` and `Stack<string>` are both built from the exact same blueprint, but each version only ever lets you push/pop the type you specified — no mixing socks into the coin drawer."

---

## 5. Generic Constraints (`extends`)

### What
A generic constraint restricts what types are allowed to be used for a type parameter, using the `extends` keyword. It says: "T can be *any* type, **as long as** it satisfies this minimum requirement."

### Why
Sometimes a fully open type parameter (`T` = literally anything) is too permissive — your function might need to rely on a property or method existing on whatever type is passed in (like `.length`, or a specific field). Without a constraint, TypeScript won't let you access that property, because it can't guarantee just *any* type will have it.

### How

```typescript
// Without a constraint — TypeScript has no idea if T has a "length" property
function logLength<T>(item: T): void {
  console.log(item.length); // ❌ Error: Property 'length' does not exist on type 'T'
}

// WITH a constraint — T must have a "length" property
interface HasLength {
  length: number;
}

function logLength<T extends HasLength>(item: T): void {
  console.log(item.length); // ✅ Allowed — TypeScript knows every valid T has "length"
}

logLength("Hello");        // ✅ strings have "length"
logLength([1, 2, 3]);      // ✅ arrays have "length"
logLength({ length: 10 }); // ✅ any object with a "length" property works
logLength(42);             // ❌ Error: number doesn't have "length"
```

**A very common real-world constraint pattern — restricting `T` to a known set of object keys:**

```typescript
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const person = { name: "Alice", age: 25 };

console.log(getProperty(person, "name")); // ✅ "Alice"
console.log(getProperty(person, "age"));  // ✅ 25
console.log(getProperty(person, "email")); // ❌ Error: "email" is not a key of person
```
*(`keyof` is covered fully on Day 6 — this is just a preview of how naturally it pairs with generics.)*

### Teach-it-back
"A generic constraint is like a job posting that says 'any applicant is welcome, as long as they have a valid driver's license.' You're not limited to one specific person, but you ARE guaranteed every accepted applicant has that one required qualification."

---

## 6. Default Generic Types

### What
You can give a generic type parameter a **default type** — if the caller doesn't explicitly specify a type, TypeScript falls back to the default instead of requiring it every time.

### Why
Sometimes most usages of a generic type will use the same type, and forcing every caller to always specify it explicitly is repetitive. A default keeps things concise for the common case, while still allowing full flexibility when something different is needed.

### How

```typescript
interface ApiResponse<T = string> {
  data: T;
  success: boolean;
}

// No type specified — defaults to "string"
const defaultResponse: ApiResponse = { data: "OK", success: true };

// Explicitly overriding the default
const numberResponse: ApiResponse<number> = { data: 200, success: true };
```

**Default types in generic functions/classes:**

```typescript
class Container<T = string> {
  constructor(public value: T) {}
}

const defaultContainer = new Container("Hello");      // T defaults to string here (inferred anyway)
const explicitContainer = new Container<number>(42);  // explicitly overridden to number
```

### Teach-it-back
"A default generic type is like a restaurant's 'usual order' — if you don't specify anything different, you automatically get the regular. But you can always ask for something else instead."

---

## 7. Utility-Style Generic Patterns

### What
This section shows a few common **patterns** built using generics — not new syntax, but practical combinations of everything above that you'll see constantly in real TypeScript code (and which set up Day 6's built-in utility types perfectly).

### Why
Seeing generics applied to realistic problems cements *why* they matter — they're not just academic syntax, they're the backbone of reusable, type-safe utility code in almost every TypeScript codebase and library.

### How

**Pattern 1 — A generic "wrapper" for handling success/failure results:**

```typescript
type Result<T> =
  | { success: true; data: T }
  | { success: false; error: string };

function divide(a: number, b: number): Result<number> {
  if (b === 0) {
    return { success: false, error: "Cannot divide by zero" };
  }
  return { success: true, data: a / b };
}

const result = divide(10, 2);
if (result.success) {
  console.log(result.data); // ✅ narrowed — TypeScript knows "data" exists here
} else {
  console.log(result.error); // ✅ narrowed — TypeScript knows "error" exists here
}
```

**Pattern 2 — A generic function that merges two objects (a simplified version of how libraries implement object merging):**

```typescript
function merge<T, U>(obj1: T, obj2: U): T & U {
  return { ...obj1, ...obj2 };
}

const merged = merge({ name: "Alice" }, { age: 25 });
console.log(merged.name); // "Alice"
console.log(merged.age);  // 25
// merged's type is inferred as { name: string } & { age: number }
```

**Pattern 3 — A generic "repository" pattern (extremely common in real backend apps):**

```typescript
class Repository<T extends { id: number }> {
  private items: T[] = [];

  add(item: T): void {
    this.items.push(item);
  }

  findById(id: number): T | undefined {
    return this.items.find((item) => item.id === id);
  }

  getAll(): T[] {
    return this.items;
  }
}

type Product = { id: number; name: string; price: number };

const productRepo = new Repository<Product>();
productRepo.add({ id: 1, name: "Laptop", price: 1000 });
console.log(productRepo.findById(1)); // { id: 1, name: "Laptop", price: 1000 }
```

### Teach-it-back
"These patterns are like reusable kitchen tools — a `Result<T>` is a tray that always tells you whether the dish succeeded or failed before you even look at what's on it. A `Repository<T>` is a generic filing cabinet that works for any kind of labeled folder, as long as each folder has an ID tag."

---

## Quick Recap — Day 5 Checklist

By the end of today, you (and anyone you teach) should be able to confidently answer:

- [ ] What problem do generics solve that `any` and writing duplicate functions both fail to solve?
- [ ] How does TypeScript usually figure out what a generic type parameter actually is, without you specifying it?
- [ ] How do you write a generic interface or type alias, and why would you need one?
- [ ] How does a generic class let one class definition work safely with many different types?
- [ ] What does a generic constraint (`extends`) actually restrict, and why is it needed?
- [ ] What's the benefit of giving a generic type parameter a default type?
- [ ] Can you describe a real use case (like a Result type or Repository pattern) where generics genuinely simplify the code?

---

## Mini Practice Exercise (good for teaching/demoing)

```typescript
// 1. Generic interface with a constraint
interface Identifiable {
  id: number;
}

// 2. Generic class enforcing the constraint
class Collection<T extends Identifiable> {
  private items: T[] = [];

  add(item: T): void {
    this.items.push(item);
  }

  findById(id: number): T | undefined {
    return this.items.find((item) => item.id === id);
  }

  remove(id: number): void {
    this.items = this.items.filter((item) => item.id !== id);
  }

  getAll(): T[] {
    return this.items;
  }
}

// 3. Concrete type used with the generic class
type Task = {
  id: number;
  title: string;
  completed: boolean;
};

const taskList = new Collection<Task>();
taskList.add({ id: 1, title: "Learn Generics", completed: false });
taskList.add({ id: 2, title: "Practice TypeScript", completed: false });

console.log(taskList.findById(1)); // { id: 1, title: "Learn Generics", completed: false }

// 4. Generic function with a default type, returning a Result-style type
type Result<T = string> =
  | { success: true; data: T }
  | { success: false; error: string };

function completeTask(taskId: number): Result<Task> {
  const task = taskList.findById(taskId);
  if (!task) {
    return { success: false, error: "Task not found" };
  }
  task.completed = true;
  return { success: true, data: task };
}

const outcome = completeTask(1);
if (outcome.success) {
  console.log(`Completed: ${outcome.data.title}`);
} else {
  console.log(outcome.error);
}
```

**Tomorrow (Day 6):** Advanced types — `keyof`, `typeof`, indexed access types, mapped types, conditional types, built-in utility types (`Partial`, `Pick`, `Omit`, `Record`, etc.), and template literal types.
