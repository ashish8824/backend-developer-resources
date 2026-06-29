# Day 4 — Classes & OOP in TypeScript
### Classes, Access Modifiers, Getters/Setters, Abstract Classes, Implementing Interfaces, Static Members

> **Recap of Day 3:** Union types, intersection types, literal types, narrowing, discriminated unions, and type guards. Today we move into Object-Oriented Programming (OOP) — TypeScript adds real, enforced structure to JavaScript classes.

---

## 1. Classes: Properties, Constructors, Methods

### What
A class is a **blueprint for creating objects** that share the same structure (properties) and behavior (methods). TypeScript adds type-checking on top of JavaScript classes, so every property and method is verified at compile time.

### Why
Plain JavaScript classes let you add, remove, or mistype properties freely, and they don't enforce what types those properties should hold. TypeScript classes ensure every instance of a class is built exactly the way it's supposed to be — catching missing properties, wrong types, and typos before the code ever runs.

### How

```typescript
class Person {
  // Properties (must declare their type)
  name: string;
  age: number;

  // Constructor — runs automatically when a new object is created
  constructor(name: string, age: number) {
    this.name = name;
    this.age = age;
  }

  // Method
  introduce(): string {
    return `Hi, I'm ${this.name} and I'm ${this.age} years old.`;
  }
}

// Creating an instance ("instantiating" the class)
const alice = new Person("Alice", 25);
console.log(alice.introduce()); // "Hi, I'm Alice and I'm 25 years old."

alice.age = "twenty five"; // ❌ Error: string not assignable to number
```

**Shorthand: parameter properties** (a very common TypeScript convenience — declares AND assigns in one step):

```typescript
class Person {
  // No need to repeat "name" and "age" as separate property declarations —
  // adding a modifier (public/private/etc.) in the constructor does both at once
  constructor(public name: string, public age: number) {}

  introduce(): string {
    return `Hi, I'm ${this.name} and I'm ${this.age} years old.`;
  }
}

const bob = new Person("Bob", 30); // same result, less code
```

### Teach-it-back
"A class is a cookie-cutter. It defines the exact shape every cookie (object) made from it will have. The constructor is the moment you press the cutter into the dough — that's when the specific values (name, age, etc.) get filled in."

---

## 2. Access Modifiers: `public`, `private`, `protected`

### What
Access modifiers control **where a property or method can be accessed from**:

| Modifier | Accessible from... |
|---|---|
| `public` (default) | Anywhere — inside the class, subclasses, and outside code |
| `private` | Only inside the **same class** |
| `protected` | Inside the same class **and** subclasses (but not from outside) |

### Why
Not every property should be freely changeable from anywhere in your code. For example, a bank account's `balance` shouldn't be directly settable from outside the class (`account.balance = 1000000`) — it should only change through controlled methods like `deposit()` or `withdraw()`. Access modifiers let you enforce these boundaries directly in the type system.

### How

```typescript
class BankAccount {
  public accountHolder: string;   // accessible from anywhere
  private balance: number;         // accessible ONLY inside this class
  protected accountNumber: string; // accessible here AND in subclasses

  constructor(accountHolder: string, balance: number, accountNumber: string) {
    this.accountHolder = accountHolder;
    this.balance = balance;
    this.accountNumber = accountNumber;
  }

  deposit(amount: number): void {
    this.balance += amount; // ✅ allowed — we're inside the class
  }

  getBalance(): number {
    return this.balance; // ✅ controlled access to a private property
  }
}

const account = new BankAccount("Alice", 1000, "ACC123");

console.log(account.accountHolder); // ✅ public — accessible
console.log(account.balance);       // ❌ Error: "balance" is private
account.deposit(500);                // ✅ allowed via public method
console.log(account.getBalance());  // ✅ 1500
```

**`protected` in action — only visible to the class and its subclasses:**

```typescript
class SavingsAccount extends BankAccount {
  showAccountNumber(): string {
    return `Account Number: ${this.accountNumber}`; // ✅ allowed — protected is visible to subclasses
  }
}

const savings = new SavingsAccount("Bob", 2000, "ACC456");
console.log(savings.showAccountNumber()); // ✅ works
console.log(savings.accountNumber);        // ❌ Error: protected, not accessible from outside
```

### Teach-it-back
"Think of a house: `public` is the front yard — anyone walking by can see it. `private` is your bedroom — only you (this exact class) can go in. `protected` is a family room — you and your kids (subclasses) can use it, but guests (outside code) can't."

---

## 3. `readonly` in Classes

### What
Just like with interfaces (Day 2), `readonly` on a class property means the value can be assigned **once** — typically in the constructor — but never changed again afterward.

### Why
Some properties should be permanently fixed once an object is created — like a unique `id`, a `createdAt` timestamp, or a `socialSecurityNumber`. Marking them `readonly` prevents accidental (or malicious) reassignment anywhere else in the code.

### How

```typescript
class User {
  readonly id: number;
  name: string;

  constructor(id: number, name: string) {
    this.id = id;     // ✅ allowed — this is the initial assignment
    this.name = name;
  }

  changeId(newId: number) {
    this.id = newId; // ❌ Error: Cannot assign to 'id' because it is a read-only property
  }
}

const user = new User(1, "Alice");
user.id = 2; // ❌ Error — can't reassign from outside either
```

### Teach-it-back
"`readonly` on a class property is like engraving a serial number onto a product at the factory. It gets stamped once, at creation — after that, no one can change it, not even the manufacturer."

---

## 4. Getters and Setters

### What
Getters (`get`) and setters (`set`) let you define **methods that behave like properties**. A getter runs custom logic when you *read* a value; a setter runs custom logic when you *assign* a value.

### Why
Sometimes you want a property to look simple from the outside (`account.balance`) but actually run validation or computation behind the scenes (e.g., preventing a negative balance, or formatting a value). Getters/setters give you that control while keeping the calling code clean and readable.

### How

```typescript
class BankAccount {
  private _balance: number;

  constructor(initialBalance: number) {
    this._balance = initialBalance;
  }

  // Getter — looks like a property when accessed: account.balance
  get balance(): number {
    return this._balance;
  }

  // Setter — looks like a property assignment: account.balance = 500
  set balance(amount: number) {
    if (amount < 0) {
      throw new Error("Balance cannot be negative");
    }
    this._balance = amount;
  }
}

const account = new BankAccount(1000);

console.log(account.balance); // 1000 — calls the GETTER, no parentheses needed!
account.balance = 1500;        // calls the SETTER
console.log(account.balance); // 1500

account.balance = -100; // ❌ Throws Error: "Balance cannot be negative"
```

### Teach-it-back
"A getter/setter is like a vending machine's front panel. You press a button (read/write a property) and it *looks* simple, but behind the panel, the machine is checking stock, validating coins, etc. — all that logic is hidden from you as the user."

---

## 5. Abstract Classes vs Interfaces

### What
An **abstract class** is a class that **cannot be instantiated directly** — it exists only to be extended by other classes. It can contain both fully implemented methods AND `abstract` methods (methods with no body, that subclasses *must* implement).

### Why
Sometimes you want to share some real, working code among related classes (like an interface can't do — interfaces only describe shape, never implementation), while still forcing certain methods to be defined differently by each subclass. Abstract classes give you both: shared logic + enforced structure.

### How

```typescript
abstract class Shape {
  abstract getArea(): number; // no implementation — each subclass MUST provide one

  // Regular method — shared logic, available to all subclasses automatically
  describe(): string {
    return `This shape has an area of ${this.getArea()}`;
  }
}

// const shape = new Shape(); // ❌ Error: Cannot create an instance of an abstract class

class Circle extends Shape {
  constructor(public radius: number) {
    super(); // must call the parent constructor
  }

  getArea(): number {
    return Math.PI * this.radius ** 2; // required implementation
  }
}

class Square extends Shape {
  constructor(public sideLength: number) {
    super();
  }

  getArea(): number {
    return this.sideLength ** 2;
  }
}

const circle = new Circle(10);
console.log(circle.describe()); // "This shape has an area of 314.159..."

const square = new Square(4);
console.log(square.describe()); // "This shape has an area of 16"
```

### Abstract Class vs Interface — Quick Comparison

| | Abstract Class | Interface |
|---|---|---|
| Can contain actual implemented code | ✅ Yes | ❌ No (shape only) |
| Can be instantiated directly | ❌ No | ❌ No (not a class at all) |
| A class can extend/implement how many? | Only **one** abstract class (`extends`) | **Multiple** interfaces (`implements`) |
| Use when... | You want shared logic + enforced structure | You only need to describe shape/contract |

### Teach-it-back
"An abstract class is like a half-finished recipe template — some steps are already written for you (shared methods), but a few steps say 'figure this part out yourself' (abstract methods). An interface, by contrast, is just a checklist with no actual cooking instructions at all."

---

## 6. Implementing Interfaces in Classes

### What
A class can use the `implements` keyword to promise that it will provide everything an interface requires — essentially saying "I follow this contract."

### Why
This lets you define a contract (interface) separately from any specific implementation, so multiple unrelated classes can all guarantee they support the same shape — useful for writing flexible code that works with "anything that fulfills this contract," regardless of how it's actually built internally.

### How

```typescript
interface Drivable {
  speed: number;
  drive(): void;
}

class Car implements Drivable {
  speed: number = 0;

  drive(): void {
    this.speed = 60;
    console.log(`Driving at ${this.speed} km/h`);
  }
}

class Bicycle implements Drivable {
  speed: number = 0;

  drive(): void {
    this.speed = 15;
    console.log(`Pedaling at ${this.speed} km/h`);
  }
}

// A function that works with ANY class implementing Drivable
function startJourney(vehicle: Drivable) {
  vehicle.drive();
}

startJourney(new Car());      // "Driving at 60 km/h"
startJourney(new Bicycle());  // "Pedaling at 15 km/h"
```

**A class can implement multiple interfaces** (something abstract classes can't do with `extends`):

```typescript
interface Swimmable {
  swim(): void;
}

interface Flyable {
  fly(): void;
}

class Duck implements Swimmable, Flyable {
  swim(): void {
    console.log("Duck is swimming");
  }
  fly(): void {
    console.log("Duck is flying");
  }
}
```

### Teach-it-back
"`implements` is a class raising its hand and saying 'I promise I have everything on this checklist.' If it's missing even one required item, TypeScript refuses to let it make that promise."

---

## 7. Static Members

### What
A `static` property or method belongs to the **class itself**, not to individual instances. You access it directly on the class name, not on an object created from the class.

### Why
Some data or behavior is shared across *all* instances rather than belonging to any one specific object — like a counter tracking how many instances have been created, or a utility/helper method that doesn't need any particular instance's data to work.

### How

```typescript
class User {
  static totalUsers: number = 0; // shared across ALL instances

  name: string;

  constructor(name: string) {
    this.name = name;
    User.totalUsers++; // accessed via the CLASS name, not "this"
  }

  static getTotalUsers(): number { // static method
    return User.totalUsers;
  }
}

const user1 = new User("Alice");
const user2 = new User("Bob");

console.log(User.getTotalUsers()); // 2
console.log(User.totalUsers);       // 2

console.log(user1.totalUsers); // ❌ Error: "totalUsers" is a static property, not accessible on an instance
```

**A practical static utility example:**

```typescript
class MathUtils {
  static square(n: number): number {
    return n * n;
  }
}

// No need to create an instance — call it directly on the class
console.log(MathUtils.square(5)); // 25
```

### Teach-it-back
"Instance properties belong to one specific cookie made from the cutter. Static properties belong to the cookie cutter itself — shared by everyone, accessible without needing any individual cookie at all."

---

## Quick Recap — Day 4 Checklist

By the end of today, you (and anyone you teach) should be able to confidently answer:

- [ ] How does a TypeScript class constructor differ from just setting properties later?
- [ ] What's the practical difference between `public`, `private`, and `protected`?
- [ ] How is `readonly` on a class property different from a regular property?
- [ ] Why would you use a getter/setter instead of a plain public property?
- [ ] What can an abstract class do that an interface cannot — and vice versa?
- [ ] How does a class "implement" an interface, and can it implement more than one?
- [ ] What's the difference between a static member and an instance member?

---

## Mini Practice Exercise (good for teaching/demoing)

```typescript
// 1. Interface as a contract
interface Payable {
  calculatePay(): number;
}

// 2. Abstract class with shared logic + one enforced abstract method
abstract class Employee implements Payable {
  protected static employeeCount: number = 0;

  constructor(public name: string, protected baseSalary: number) {
    Employee.employeeCount++;
  }

  abstract calculatePay(): number; // every subclass must define this differently

  describe(): string {
    return `${this.name} earns $${this.calculatePay()}`;
  }

  static getEmployeeCount(): number {
    return Employee.employeeCount;
  }
}

// 3. Concrete subclass implementing the abstract method
class Manager extends Employee {
  constructor(name: string, baseSalary: number, private bonus: number) {
    super(name, baseSalary);
  }

  calculatePay(): number {
    return this.baseSalary + this.bonus;
  }
}

// 4. Getter/setter for safe access
class SecureManager extends Manager {
  private _bonusApproved: boolean = false;

  get bonusApproved(): boolean {
    return this._bonusApproved;
  }

  set bonusApproved(value: boolean) {
    this._bonusApproved = value;
  }
}

const mgr = new Manager("Alice", 50000, 5000);
console.log(mgr.describe());                    // "Alice earns $55000"
console.log(Employee.getEmployeeCount());        // 1

const secureMgr = new SecureManager("Bob", 60000, 8000);
secureMgr.bonusApproved = true;
console.log(secureMgr.bonusApproved);            // true
console.log(Employee.getEmployeeCount());        // 2
```

**Tomorrow (Day 5):** Generics — what they are, why we need them, generic functions, generic interfaces/types, generic classes, generic constraints (`extends`), and default generic types.
