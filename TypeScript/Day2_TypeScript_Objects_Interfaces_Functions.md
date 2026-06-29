# Day 2 — Objects, Interfaces & Functions
### Object Typing, `interface`, `interface` vs `type`, Optional/Readonly Properties, Functions in Depth

> **Recap of Day 1:** Basic types, arrays, tuples, type inference, and type aliases. Today builds directly on type aliases — interfaces are another (often better) way to describe object shapes.

---

## 1. Object Types

### What
An object type describes the **shape** of an object — what properties it has, and what type each property's value must be.

### Why
In plain JavaScript, an object can have any properties, any values, and you can add or remove properties freely. That flexibility causes bugs: you call `user.naem` (a typo) and JavaScript just gives you `undefined` instead of an error — and you don't find out until something breaks at runtime.

TypeScript checks the object's shape ahead of time, so typos and missing/extra properties get caught immediately.

```javascript
// Plain JavaScript — no safety net
const user = { name: "Alice", age: 25 };
console.log(user.naem); // undefined — typo, but no error. Bug hides until later.
```

### How

```typescript
// Inline object type annotation
let user: { name: string; age: number };

user = { name: "Alice", age: 25 };  // ✅ matches the shape
user = { name: "Alice" };           // ❌ Error: missing "age"
user = { name: "Alice", age: "25" }; // ❌ Error: age should be number, not string

console.log(user.naem); // ❌ Error caught immediately — "naem" doesn't exist on this type
```

Using a **type alias** (from Day 1) to avoid repeating the shape:

```typescript
type User = {
  name: string;
  age: number;
};

function greetUser(user: User) {
  console.log(`Hello, ${user.name}! You are ${user.age}.`);
}

greetUser({ name: "Bob", age: 30 }); // ✅
```

### Teach-it-back
"An object type is a blueprint. It says exactly which rooms (properties) a house (object) must have, and what each room is for (its type). If the house doesn't match the blueprint, TypeScript flags it before you ever move in."

---

## 2. `interface` — What, Why, How

### What
An `interface` is another way (alongside `type`) to define the shape of an object. It's TypeScript's dedicated tool for describing structure — especially for objects and classes.

### Why
While `type` aliases *can* describe object shapes, `interface` was specifically designed for this purpose and offers some extra abilities that are especially useful in object-oriented code (classes implementing interfaces — covered Day 4) and in scenarios where the shape needs to be **extended or merged** later.

### How

```typescript
// Defining an interface
interface User {
  name: string;
  age: number;
}

// Using it exactly like a type alias
function greetUser(user: User) {
  console.log(`Hello, ${user.name}! You are ${user.age}.`);
}

const newUser: User = { name: "Alice", age: 25 };
greetUser(newUser); // ✅
```

**Extending interfaces** (one of `interface`'s key superpowers):

```typescript
interface Person {
  name: string;
  age: number;
}

// Employee "extends" Person — inherits all its properties, plus adds its own
interface Employee extends Person {
  employeeId: number;
  department: string;
}

const employee: Employee = {
  name: "Alice",
  age: 25,
  employeeId: 101,
  department: "Engineering"
}; // ✅ must satisfy BOTH Person and Employee's properties
```

**Declaration merging** (a feature unique to `interface`, not available with `type`):

```typescript
interface Car {
  brand: string;
}

// You can declare the SAME interface again, and TypeScript merges them
interface Car {
  model: string;
}

// Now Car requires BOTH brand and model
const myCar: Car = { brand: "Toyota", model: "Corolla" }; // ✅
```

### Teach-it-back
"An interface is like a contract template — it defines what an object (or a class, later) must have. Unlike a one-time blueprint, a contract can be extended (added to by inheritance) or reopened and added to later (merging)."

---

## 3. `interface` vs `type` — Key Differences

### What
Both `interface` and `type` can describe object shapes — but they have important differences in capability and intended use.

### Why
This is one of the **most commonly asked TypeScript questions** — knowing exactly when to reach for which one shows real understanding, not just memorized syntax.

### How — Side-by-side Comparison

| Feature | `interface` | `type` |
|---|---|---|
| Describe object shape | ✅ Yes | ✅ Yes |
| Extend/inherit | ✅ `extends` keyword | ✅ via `&` (intersection) |
| Declaration merging (reopen and add later) | ✅ Yes | ❌ No (causes an error if redeclared) |
| Describe primitives (`string`, `number`, etc.) | ❌ No | ✅ Yes |
| Describe unions (`string \| number`) | ❌ No | ✅ Yes |
| Describe tuples | ⚠️ Awkward | ✅ Yes, naturally |
| Used with classes (`implements`) | ✅ Common use case | ✅ Also works, less idiomatic |

```typescript
// type CAN do things interface cannot:
type ID = string | number;            // ✅ union — interface can't do this directly
type Coordinates = [number, number];  // ✅ tuple

// interface CAN do things type struggles with:
interface Animal {
  name: string;
}
interface Animal { // ✅ merges automatically
  sound: string;
}

type Vehicle = { brand: string };
type Vehicle = { model: string }; // ❌ Error: Duplicate identifier 'Vehicle'

// Extending — both work, slightly different syntax
interface Person { name: string; }
interface Student extends Person { school: string; } // interface extension

type Person2 = { name: string; };
type Student2 = Person2 & { school: string; }; // type intersection (& explained Day 3)
```

### Rule of Thumb (industry convention)
- Use **`interface`** when defining the shape of **objects or classes** — especially in libraries/APIs that might need extending later.
- Use **`type`** when working with **unions, tuples, primitives, or function types** — anything beyond a plain object shape.

### Teach-it-back
"Think of `interface` as a contract specifically for describing 'things with properties' — flexible to extend later. Think of `type` as a more general labeling tool — it can name *any* kind of type, not just object shapes."

---

## 4. Optional (`?`) and Readonly Properties

### What
- **Optional property (`?`)**: A property that may or may not be present on the object.
- **Readonly property**: A property that can be set once (usually at creation) but never reassigned afterward.

### Why
Real-world objects often have fields that aren't always required (e.g., a `middleName` not everyone has) and fields that should never change after being set (e.g., a database `id`, a user's `dateOfBirth`). Without marking these explicitly, TypeScript would either force every field to always be present, or allow accidental overwrites of values that should be permanent.

### How

```typescript
interface User {
  id: number;
  name: string;
  middleName?: string;     // optional — may or may not exist
  readonly createdAt: Date; // readonly — can be set once, never changed again
}

const user: User = {
  id: 1,
  name: "Alice",
  createdAt: new Date()
  // middleName omitted — totally fine, it's optional
};

console.log(user.middleName); // undefined, no error

user.createdAt = new Date(); // ❌ Error: Cannot assign to 'createdAt' because it is read-only
user.name = "Alicia";         // ✅ Allowed — "name" wasn't marked readonly
```

**Optional properties in function parameters:**

```typescript
function createUser(name: string, age?: number) {
  if (age) {
    console.log(`${name} is ${age} years old`);
  } else {
    console.log(`${name}'s age is unknown`);
  }
}

createUser("Bob");        // ✅ age omitted, that's fine
createUser("Bob", 30);    // ✅ age provided
```

**Combining `?` with default values:**

```typescript
function createUserWithDefault(name: string, age: number = 18) {
  console.log(`${name} is ${age} years old`);
}

createUserWithDefault("Charlie"); // age defaults to 18
```

### Teach-it-back
"`?` means 'this drawer might be empty, and that's okay.' `readonly` means 'whatever you put in this drawer first stays there forever — no swapping it out later.'"

---

## 5. Function Types in Depth

### What
Function typing means specifying the types of a function's **parameters** and its **return value**, so TypeScript can verify the function is called correctly and used correctly wherever it returns a value.

### Why
Functions are where most real bugs happen — wrong number of arguments, wrong argument types, forgetting to return a value, etc. Typing functions catches these mistakes the moment you write the code, not when a user clicks a broken button in production.

### How

**Basic typed function:**

```typescript
function add(a: number, b: number): number {
  return a + b;
}

add(5, 10);     // ✅ 15
add(5, "10");   // ❌ Error: argument of type string not assignable to number
```

**Optional parameters** (must come after required ones):

```typescript
function greet(name: string, greeting?: string): string {
  return `${greeting ?? "Hello"}, ${name}!`;
}

greet("Alice");              // "Hello, Alice!"
greet("Alice", "Welcome");   // "Welcome, Alice!"
```

**Default parameters:**

```typescript
function calculateTotal(price: number, taxRate: number = 0.1): number {
  return price + price * taxRate;
}

calculateTotal(100);       // uses default taxRate of 0.1 → 110
calculateTotal(100, 0.2);  // uses provided taxRate → 120
```

**Rest parameters** (collect any number of extra arguments into an array):

```typescript
function sumAll(...numbers: number[]): number {
  return numbers.reduce((total, n) => total + n, 0);
}

sumAll(1, 2, 3);       // 6
sumAll(1, 2, 3, 4, 5); // 15
```

**Function type expressed as a type alias** (useful for callbacks):

```typescript
type MathOperation = (a: number, b: number) => number;

const multiply: MathOperation = (a, b) => a * b;
const divide: MathOperation = (a, b) => a / b;

function calculate(a: number, b: number, operation: MathOperation): number {
  return operation(a, b);
}

calculate(10, 5, multiply); // 50
calculate(10, 5, divide);   // 2
```

**Arrow functions with types:**

```typescript
const square = (n: number): number => n * n;
```

### Teach-it-back
"Typing a function is like writing a recipe card: it lists exactly what ingredients (parameters) go in, in what order, and exactly what dish (return value) comes out. Optional and default parameters are ingredients you can skip or that have a 'usual amount' built in. Rest parameters are like saying 'add as many extra toppings as you want.'"

---

## 6. Function Overloading

### What
Function overloading lets you define **multiple call signatures** for the same function name — so the function can accept different combinations/types of parameters and behave (or type-check) appropriately for each.

### Why
Sometimes a single function genuinely needs to support different input types with different resulting behavior or return types, and a simple union type isn't precise enough to capture the relationship between the *specific* input and the *specific* output.

### How

```typescript
// Overload signatures (no body — just declares the valid call patterns)
function getInfo(id: number): string;
function getInfo(name: string): number;

// Implementation signature (the actual function — must handle all overload cases)
function getInfo(value: number | string): string | number {
  if (typeof value === "number") {
    return `User ID is ${value}`;     // returns a string when given a number
  } else {
    return value.length;               // returns a number (name length) when given a string
  }
}

console.log(getInfo(101));      // "User ID is 101"
console.log(getInfo("Alice"));  // 5 (length of "Alice")
```

**A more realistic example — combining two values:**

```typescript
function combine(a: string, b: string): string;
function combine(a: number, b: number): number;

function combine(a: any, b: any): any {
  return a + b;
}

combine("Hello, ", "World!"); // "Hello, World!"  (string + string)
combine(5, 10);               // 15               (number + number)
combine("5", 10);             // ❌ Error: no overload matches this call
```

### Teach-it-back
"Overloading is like having one phone number that rings differently depending on who's calling — same function name, but TypeScript checks that the *combination* of inputs and expected output makes sense for each specific case."

---

## Quick Recap — Day 2 Checklist

By the end of today, you (and anyone you teach) should be able to confidently answer:

- [ ] How do you describe the shape of an object in TypeScript?
- [ ] What is an `interface` and how is it different from just using an inline object type?
- [ ] What can `type` do that `interface` cannot, and vice versa?
- [ ] What's the difference between an optional property (`?`) and a `readonly` property?
- [ ] How do default parameters differ from optional parameters in functions?
- [ ] What do rest parameters let you do?
- [ ] What is function overloading, and when would you actually need it?

---

## Mini Practice Exercise (good for teaching/demoing)

```typescript
// 1. Interface with optional + readonly properties
interface Product {
  readonly id: number;
  name: string;
  price: number;
  discount?: number;
}

// 2. Function using the interface, with a default parameter
function getFinalPrice(product: Product, taxRate: number = 0.05): number {
  const discountedPrice = product.price - (product.discount ?? 0);
  return discountedPrice + discountedPrice * taxRate;
}

const laptop: Product = { id: 1, name: "Laptop", price: 1000, discount: 100 };
console.log(getFinalPrice(laptop)); // applies discount + default tax

// 3. Extending an interface
interface DigitalProduct extends Product {
  downloadLink: string;
}

const ebook: DigitalProduct = {
  id: 2,
  name: "TypeScript Guide",
  price: 20,
  downloadLink: "https://example.com/download"
};

// 4. Function overload example
function describe(item: Product): string;
function describe(item: DigitalProduct): string;
function describe(item: any): string {
  return "downloadLink" in item
    ? `${item.name} is a digital product.`
    : `${item.name} is a physical product.`;
}

console.log(describe(laptop)); // "Laptop is a physical product."
console.log(describe(ebook));  // "TypeScript Guide is a digital product."
```

**Tomorrow (Day 3):** Union types, intersection types, literal types, type narrowing, discriminated unions, and custom type guards.
