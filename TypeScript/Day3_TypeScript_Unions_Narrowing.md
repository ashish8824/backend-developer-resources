# Day 3 — Union, Intersection & Advanced Narrowing
### Union Types, Intersection Types, Literal Types, Type Narrowing, Discriminated Unions, Custom Type Guards

> **Recap of Day 2:** Object types, `interface`, `interface` vs `type`, optional/readonly properties, functions, and overloading. Today we combine and narrow down types — this is where TypeScript starts to feel really powerful.

---

## 1. Union Types (`|`)

### What
A union type lets a variable (or parameter, or property) hold **one of several possible types**. You write it using the pipe symbol `|` between types.

### Why
Real data is often "this OR that" — a user ID might be a `number` in your database but a `string` when it comes from a URL. Without union types, you'd either have to pick one type (and lose accuracy) or use `any` (and lose all safety). Unions let you stay precise about *which* types are actually allowed, instead of opening the door to everything.

### How

```typescript
// A variable that can be EITHER a string OR a number
let id: string | number;

id = 101;      // ✅ allowed
id = "U101";   // ✅ allowed
id = true;     // ❌ Error: boolean is not part of the union

// Union types in function parameters
function printId(id: string | number) {
  console.log(`Your ID is: ${id}`);
}

printId(101);    // ✅
printId("U101"); // ✅
printId(true);   // ❌ Error

// Union of object types
type Cat = { type: "cat"; meow: () => void };
type Dog = { type: "dog"; bark: () => void };

let pet: Cat | Dog;
```

**Important catch:** When a variable has a union type, you can only directly use the methods/properties that exist on **all** the types in the union — TypeScript won't let you assume it's specifically one or the other without checking first (this is exactly why we need **narrowing**, covered in section 4).

```typescript
function printLength(value: string | number) {
  console.log(value.length); // ❌ Error: "length" doesn't exist on type "number"
}
```

### Teach-it-back
"A union type is like saying 'this seat is reserved for either Alice OR Bob' — only those two people are allowed to sit there, nobody else. But until you check *which* of them actually showed up, you can't assume what they'll do."

---

## 2. Intersection Types (`&`)

### What
An intersection type combines **multiple types into one**, requiring the value to satisfy **all** of them at once. You write it using the ampersand `&` between types.

### Why
Sometimes an object needs to be more than one thing simultaneously — e.g., something that is both a `Person` AND an `Employee`. Intersections let you build new types by merging existing ones, instead of redefining everything from scratch (similar in spirit to `extends` with interfaces, from Day 2, but works with `type`).

### How

```typescript
type Person = {
  name: string;
  age: number;
};

type Employee = {
  employeeId: number;
  department: string;
};

// Intersection: must have ALL properties from BOTH types
type StaffMember = Person & Employee;

const staff: StaffMember = {
  name: "Alice",
  age: 25,
  employeeId: 101,
  department: "Engineering"
}; // ✅ must satisfy every property from both types

const incomplete: StaffMember = {
  name: "Bob",
  age: 30,
  employeeId: 102
  // ❌ Error: missing "department"
};
```

**Intersections with primitive-like unions can become impossible (`never`):**

```typescript
type OnlyString = string & number; // ❌ this becomes type "never" — nothing can be both at once
```

### Union vs Intersection — Quick Comparison

| | Union (`\|`) | Intersection (`&`) |
|---|---|---|
| Meaning | "either / or" | "both / and" |
| Value must satisfy | At least one type | All types simultaneously |
| Common use | Variables that can take different forms | Combining object shapes |

### Teach-it-back
"Union is an 'OR' — either type A or type B. Intersection is an 'AND' — it must be type A *and* type B at the same time, like a job title that requires you to be both a manager *and* an engineer."

---

## 3. Literal Types

### What
A literal type restricts a value to one **specific, exact value** — not just a general type like `string` or `number`, but a particular string or number itself.

### Why
Sometimes "any string" is too loose. If a function only accepts `"small"`, `"medium"`, or `"large"` as valid sizes, typing the parameter as `string` would still allow `"tiny"` or `"xl"` — both technically strings, but invalid for your app. Literal types let you say exactly which values are acceptable.

### How

```typescript
// String literal type
let direction: "up" | "down" | "left" | "right";

direction = "up";       // ✅
direction = "diagonal"; // ❌ Error: not one of the allowed literal values

// Number literal type
let diceRoll: 1 | 2 | 3 | 4 | 5 | 6;
diceRoll = 4; // ✅
diceRoll = 7; // ❌ Error

// Using literal types in a function (very common real-world pattern)
function setVolume(level: "low" | "medium" | "high") {
  console.log(`Volume set to ${level}`);
}

setVolume("medium"); // ✅
setVolume("loud");   // ❌ Error: "loud" is not a valid literal

// Combining literal types with a type alias for readability
type Size = "small" | "medium" | "large";

function orderShirt(size: Size) {
  console.log(`Ordering a ${size} shirt`);
}
```

### Teach-it-back
"A literal type is like a multiple-choice question instead of a fill-in-the-blank one. Instead of accepting *any* string, it only accepts the exact options you've listed — nothing else."

---

## 4. Type Narrowing

### What
Type narrowing is the process of helping TypeScript figure out **exactly which type** a value is, when that value started out as a union (e.g., `string | number`). You do this using checks like `typeof`, `instanceof`, `in`, or simple truthiness checks. Once TypeScript "sees" the check, it narrows the type for you inside that code block.

### Why
As shown earlier in the Union Types section, TypeScript won't let you use type-specific methods/properties on a union until it's sure which type you're dealing with. Narrowing is *how* you prove that to TypeScript — it's the bridge between "this could be several things" and "this is now definitely one specific thing."

### How

**`typeof` narrowing** (for primitive types):

```typescript
function printLength(value: string | number) {
  if (typeof value === "string") {
    console.log(value.length);       // ✅ TypeScript knows it's a string here
  } else {
    console.log(value.toFixed(2));   // ✅ TypeScript knows it's a number here
  }
}
```

**`instanceof` narrowing** (for class instances):

```typescript
class Car {
  drive() { console.log("Driving..."); }
}

class Boat {
  sail() { console.log("Sailing..."); }
}

function operate(vehicle: Car | Boat) {
  if (vehicle instanceof Car) {
    vehicle.drive(); // ✅ TypeScript knows it's a Car
  } else {
    vehicle.sail();  // ✅ TypeScript knows it's a Boat
  }
}
```

**`in` narrowing** (checking if a property exists on an object):

```typescript
type Cat = { meow: () => void };
type Dog = { bark: () => void };

function makeSound(pet: Cat | Dog) {
  if ("meow" in pet) {
    pet.meow(); // ✅ narrowed to Cat
  } else {
    pet.bark(); // ✅ narrowed to Dog
  }
}
```

**Truthiness narrowing** (checking for `null`/`undefined`):

```typescript
function greet(name: string | null) {
  if (name) {
    console.log(`Hello, ${name.toUpperCase()}!`); // ✅ safe — name can't be null here
  } else {
    console.log("Hello, stranger!");
  }
}
```

### Teach-it-back
"Narrowing is like asking 'who's actually at the door?' before deciding what to say. TypeScript won't let you greet a person by name until you've confirmed it's actually a person and not, say, a package — once you check, it relaxes the rules for that specific case."

---

## 5. Discriminated Unions

### What
A discriminated union is a special, very common pattern where each type in a union shares a common literal property (usually called something like `type` or `kind`) that uniquely identifies which "version" of the union it is. This makes narrowing extremely clean and reliable.

### Why
Using `in` or `typeof` checks works fine for simple cases, but gets messy with several related object shapes. A discriminated union gives every shape a clear "tag," so TypeScript (and you) can instantly tell which shape you're working with — leading to cleaner, safer, and more maintainable code, especially as the number of variants grows.

### How

```typescript
// Each shape has a "kind" property with a unique literal value — this is the "discriminant"
type Circle = {
  kind: "circle";
  radius: number;
};

type Square = {
  kind: "square";
  sideLength: number;
};

type Rectangle = {
  kind: "rectangle";
  width: number;
  height: number;
};

type Shape = Circle | Square | Rectangle;

function getArea(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;      // ✅ narrowed to Circle
    case "square":
      return shape.sideLength ** 2;             // ✅ narrowed to Square
    case "rectangle":
      return shape.width * shape.height;        // ✅ narrowed to Rectangle
  }
}

const myCircle: Shape = { kind: "circle", radius: 10 };
console.log(getArea(myCircle)); // 314.159...
```

**Why this is better than checking individual properties:** TypeScript's `switch` statement, combined with the discriminant, gives you full autocomplete and ensures you've covered every case — if you add a new shape (e.g., `Triangle`) and forget to handle it in the switch, you can even get TypeScript to warn you using **exhaustiveness checking**:

```typescript
function getAreaSafe(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.sideLength ** 2;
    case "rectangle":
      return shape.width * shape.height;
    default:
      const _exhaustiveCheck: never = shape; // ❌ errors here if a case is missing
      return _exhaustiveCheck;
  }
}
```

### Teach-it-back
"A discriminated union is like a name tag at a conference. Every attendee (object) wears a tag that says exactly which group they belong to — 'circle,' 'square,' 'rectangle' — so instead of guessing who's who, you just read the tag and know exactly how to treat them."

---

## 6. Type Guards (Custom Guard Functions)

### What
A type guard is a function whose **return type** tells TypeScript "if this function returns `true`, the value is definitely of this specific type." You write it using the special return type syntax: `parameterName is SpecificType`.

### Why
Sometimes the check needed to identify a type is more complex than a simple `typeof` or `in` — you might want to reuse the same check across multiple functions. A custom type guard lets you wrap that logic into a reusable, named function, while still letting TypeScript narrow types correctly wherever it's used.

### How

```typescript
type Cat = { type: "cat"; meow: () => void };
type Dog = { type: "dog"; bark: () => void };

// Custom type guard function
function isCat(pet: Cat | Dog): pet is Cat {
  return pet.type === "cat";
}

function makeSound(pet: Cat | Dog) {
  if (isCat(pet)) {
    pet.meow(); // ✅ TypeScript narrows to Cat because of the type guard
  } else {
    pet.bark(); // ✅ narrowed to Dog
  }
}
```

**A more practical real-world example — validating unknown data (e.g., from an API):**

```typescript
type User = {
  name: string;
  age: number;
};

function isUser(value: unknown): value is User {
  return (
    typeof value === "object" &&
    value !== null &&
    "name" in value &&
    "age" in value &&
    typeof (value as User).name === "string" &&
    typeof (value as User).age === "number"
  );
}

function processData(data: unknown) {
  if (isUser(data)) {
    console.log(`Welcome, ${data.name}!`); // ✅ safely narrowed to User
  } else {
    console.log("Invalid user data");
  }
}
```

### Teach-it-back
"A type guard is like a bouncer with a very specific checklist. You hand them something of unknown identity, they run their check, and if they say 'yes, this is a User,' everyone past that point can safely treat it as a User — no more guessing."

---

## Quick Recap — Day 3 Checklist

By the end of today, you (and anyone you teach) should be able to confidently answer:

- [ ] What does a union type (`|`) mean, and why can't you freely use type-specific methods on it right away?
- [ ] What does an intersection type (`&`) mean, and how is it different from a union?
- [ ] What is a literal type, and when would `string` alone not be precise enough?
- [ ] What is type narrowing, and what are the main ways to do it (`typeof`, `instanceof`, `in`, truthiness)?
- [ ] What is a discriminated union, and why is it cleaner than manually checking properties?
- [ ] What is a custom type guard, and what does the `is` keyword actually do in its return type?

---

## Mini Practice Exercise (good for teaching/demoing)

```typescript
// 1. Union + literal types
type Status = "pending" | "shipped" | "delivered";

// 2. Discriminated union for different notification types
type EmailNotification = {
  type: "email";
  emailAddress: string;
};

type SMSNotification = {
  type: "sms";
  phoneNumber: string;
};

type Notification = EmailNotification | SMSNotification;

// 3. Function using narrowing via discriminant
function send(notification: Notification) {
  switch (notification.type) {
    case "email":
      console.log(`Sending email to ${notification.emailAddress}`);
      break;
    case "sms":
      console.log(`Sending SMS to ${notification.phoneNumber}`);
      break;
  }
}

send({ type: "email", emailAddress: "alice@example.com" });
send({ type: "sms", phoneNumber: "123-456-7890" });

// 4. Intersection type combining order info + status
type Order = {
  id: number;
  item: string;
};

type TrackedOrder = Order & { status: Status };

const order: TrackedOrder = { id: 1, item: "Laptop", status: "shipped" };

// 5. Custom type guard
function isEmailNotification(n: Notification): n is EmailNotification {
  return n.type === "email";
}

function logIfEmail(n: Notification) {
  if (isEmailNotification(n)) {
    console.log(`This is an email notification: ${n.emailAddress}`);
  }
}
```

**Tomorrow (Day 4):** Classes — properties, constructors, access modifiers (`public`/`private`/`protected`), getters/setters, abstract classes, implementing interfaces, and static members.
