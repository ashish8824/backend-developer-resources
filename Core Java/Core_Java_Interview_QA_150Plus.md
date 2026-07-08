# 📘 Core Java — Complete Interview Q&A Bank (150+ Questions)

> **How to use this file:** Each answer is written so you can EXPLAIN it out loud, not just recite it. Where relevant, I've added a **"If they cross-question..."** note — this anticipates the natural follow-up an interviewer will ask, so you're never caught off guard. Read each answer, then try explaining it in your own words before moving to the next.

---

## SECTION A: Java Fundamentals & JVM

### 1. What is Java, and why is it called platform-independent?
**Answer:** Java is a compiled + interpreted, object-oriented programming language. It's platform-independent because Java source code compiles into an intermediate format called **bytecode** (`.class` files), not directly into machine code. This SAME bytecode file runs on ANY operating system, as long as that OS has a JVM installed. The JVM is what's actually platform-DEPENDENT (a different JVM exists for Windows, Linux, Mac) — it handles translating the universal bytecode into OS-specific machine instructions.

**If they cross-question "Isn't the JVM itself platform-dependent, so how is Java platform-independent?"** — Say: "Yes, exactly — the JVM is platform-specific, but that's precisely the point. Since the JVM absorbs all the OS-specific differences, the bytecode ABOVE it never needs to change. You write once, and any machine with a matching JVM can run it unmodified."

---

### 2. Explain JDK vs JRE vs JVM.
**Answer:**
- **JVM (Java Virtual Machine):** The engine that executes bytecode. Includes the Class Loader, Bytecode Verifier, and Execution Engine (interpreter + JIT compiler).
- **JRE (Java Runtime Environment):** JVM + core libraries (java.lang, java.util, etc.) needed to RUN a Java program.
- **JDK (Java Development Kit):** JRE + development tools (compiler `javac`, debugger) needed to WRITE and compile Java programs.

**Relationship:** JDK ⊃ JRE ⊃ JVM (JDK contains JRE, which contains JVM).

**If they cross-question "If I only want to run a compiled .jar file, do I need JDK?"** — Say: "No, JRE is enough to RUN a Java program. You only need the full JDK if you're COMPILING/writing code."

---

### 3. Explain how a Java program executes, step by step.
**Answer:** 1) Source code (`.java`) is written. 2) `javac` compiles it into bytecode (`.class`). 3) JVM's Class Loader loads the bytecode. 4) Bytecode Verifier checks it's safe. 5) Execution Engine runs it — either interpreting line-by-line, or using the JIT (Just-In-Time) compiler to convert frequently-used bytecode directly into native machine code for speed.

**If they cross-question "Is Java compiled or interpreted?"** — Say: "Both — source code is COMPILED to bytecode, and that bytecode is then INTERPRETED (with JIT optimization) by the JVM. It's a hybrid model."

---

### 4. What is JIT compilation?
**Answer:** JIT (Just-In-Time) compilation is a JVM optimization where frequently-executed bytecode ("hot code") gets compiled DIRECTLY into native machine code at runtime, instead of being interpreted line-by-line every time. This makes repeated execution much faster, since native code runs faster than interpreted bytecode.

---

### 5. Why is Java not considered "100% pure" object-oriented?
**Answer:** Because Java has 8 **primitive data types** (int, char, boolean, etc.) that are NOT objects — they don't have methods, and don't follow OOP principles. A truly "pure" OOP language (like Smalltalk) treats EVERYTHING, even numbers, as objects.

**If they cross-question "But doesn't Java have wrapper classes like Integer for this?"** — Say: "Yes, wrapper classes (Integer, Double, etc.) let primitives be TREATED as objects when needed (e.g., in Collections), via autoboxing — but the underlying primitive types themselves still exist and are used directly for performance reasons, which is why Java isn't 100% pure OOP."

---

### 6. What are wrapper classes? Why do we need them?
**Answer:** Wrapper classes provide an OBJECT representation for each primitive type: `Integer` for `int`, `Double` for `double`, `Character` for `char`, `Boolean` for `boolean`, etc. We need them because Collections (ArrayList, HashMap) can ONLY store OBJECTS, not primitives — so to put an `int` into an `ArrayList<Integer>`, Java needs the wrapper class.

---

### 7. What is autoboxing and unboxing?
**Answer:** **Autoboxing** = automatic conversion of a primitive into its wrapper object (`int` → `Integer`). **Unboxing** = the reverse (`Integer` → `int`). Java does this AUTOMATICALLY behind the scenes.
```java
Integer num = 10;       // autoboxing: int -> Integer
int n = num;              // unboxing: Integer -> int
```
**If they cross-question "What's the risk with autoboxing?"** — Say: "Performance overhead in loops (creating many wrapper objects unnecessarily), and the Integer Caching gotcha — `Integer` values from -128 to 127 are CACHED and reused, so `==` comparisons can behave inconsistently for values inside vs outside that range."

---

### 8. Explain the Integer Caching behavior (`Integer a = 127` vs `200`).
**Answer:**
```java
Integer a = 127, b = 127;
System.out.println(a == b);   // true - both use the CACHED object (range -128 to 127)

Integer c = 200, d = 200;
System.out.println(c == d);   // false - 200 is outside the cache range, separate objects
```
Java caches `Integer` objects for the range -128 to 127 (via `Integer.valueOf()`) to save memory for commonly-used small numbers. Values outside this range create NEW objects each time. This is why `==` behaves inconsistently — always use `.equals()` for wrapper object comparisons, not `==`.

---

### 9. Difference between `==` and `.equals()`?
**Answer:** `==` compares REFERENCES for objects (i.e., "do these point to the exact same memory address?") — for primitives, it compares actual VALUES. `.equals()` is a METHOD (inherited from `Object`, often overridden) that compares LOGICAL content. By default, `.equals()` behaves just like `==` UNLESS a class specifically overrides it (like `String` does) to compare actual content instead.

**If they cross-question "So does `==` ever make sense for objects?"** — Say: "Yes — when you specifically want to check if two references point to the EXACT SAME object in memory (identity check), not just equivalent content. For example, checking `if (obj == null)`."

---

### 10. Widening vs Narrowing type casting?
**Answer:** **Widening (implicit)** — converting a SMALLER type to a LARGER one (int → double); done AUTOMATICALLY by Java, no data loss risk. **Narrowing (explicit)** — converting a LARGER type to a SMALLER one (double → int); must be done MANUALLY with `(type)`, and can lose data (truncates, doesn't round).
```java
int x = (int) 9.99;   // 9, NOT 10 - truncated, not rounded
```

---

### 11. What's the difference between local, instance, and static variables?
**Answer:** **Local** — declared inside a method, exists only during that method's execution, stored on the Stack. **Instance** — declared inside a class but outside any method, belongs to a SPECIFIC OBJECT, one copy per object, stored on the Heap. **Static** — declared with `static` keyword, belongs to the CLASS itself, ONE shared copy across all objects.

---

### 12. What is the default value of local variables vs instance variables?
**Answer:** **Local variables have NO default value** — Java forces you to explicitly initialize them before use, or it's a compile error. **Instance/static variables DO get automatic default values** (0 for numeric types, false for boolean, null for objects) if not explicitly initialized.

**If they cross-question "Why the difference?"** — Say: "Instance/static fields are initialized as part of object/class creation, following well-defined default-value rules. Local variables exist only transiently on the stack during a method call, and Java requires explicit initialization to avoid ambiguous/uninitialized-use bugs."

---

### 13. Difference between static and instance methods?
**Answer:** **Static methods** belong to the class, called via `ClassName.method()`, CANNOT directly access instance variables/methods (no object context). **Instance methods** belong to a specific object, called via `object.method()`, CAN access both instance AND static members.

---

### 14. Why is the `main()` method static?
**Answer:** Because the JVM needs to call `main()` to START the program, and at that point, NO OBJECT of the class exists yet. Since static methods belong to the CLASS (not any object), JVM can call `main()` without first needing to create an instance.

---

### 15. Can we overload the `main()` method?
**Answer:** Yes, technically you CAN write multiple `main()` methods with different parameter lists (overloading, same rules as any method) — but the JVM will ONLY ever automatically call the STANDARD signature: `public static void main(String[] args)`. Any other overloaded version must be called MANUALLY from within your code; it won't be used as the program's entry point.

---

## SECTION B: Object-Oriented Programming (OOP)

### 16. What are the four pillars of OOP? Explain each briefly.
**Answer:**
- **Encapsulation** — bundling data + behavior, hiding internal state (private fields + public getters/setters)
- **Inheritance** — a class reusing another class's fields/methods (`extends`)
- **Polymorphism** — same method behaves differently (overloading = compile-time, overriding = runtime)
- **Abstraction** — hiding complexity, exposing only essential behavior (abstract classes, interfaces)

---

### 17. Difference between a Class and an Object?
**Answer:** A **class** is a blueprint/template that defines fields and methods — no memory is allocated for it directly. An **object** is an actual INSTANCE of that class, created using `new`, with real memory allocated on the Heap and actual values assigned to its fields.

---

### 18. What is a constructor? How is it different from a regular method?
**Answer:** A constructor initializes a newly created object. Differences from a method: 1) Constructor name MUST match the class name exactly; a method can have any name. 2) Constructor has NO return type (not even void); methods must have one. 3) Constructor is called AUTOMATICALLY when you use `new`; methods must be called explicitly.

---

### 19. What is constructor overloading?
**Answer:** Having MULTIPLE constructors in the same class with DIFFERENT parameter lists, so objects can be created in different ways depending on what information is available at creation time.
```java
class Car {
    Car() { }
    Car(String brand) { }
    Car(String brand, int speed) { }
}
```

---

### 20. Can a constructor be private? Why would you do that?
**Answer:** Yes. A private constructor PREVENTS the class from being instantiated directly from OUTSIDE the class. This is commonly used in the **Singleton design pattern** (see Q156), where you want to control object creation entirely from WITHIN the class itself (e.g., via a static factory method).

---

### 21. Can we override a static method?
**Answer:** No — static methods are NOT polymorphic. If a subclass defines a static method with the SAME signature as the parent's static method, this is called **method hiding**, not overriding. The method that gets called depends on the REFERENCE TYPE (compile-time), not the actual object type — the OPPOSITE behavior of true overriding.
```java
class A { static void show() { System.out.println("A"); } }
class B extends A { static void show() { System.out.println("B"); } }
A obj = new B();
obj.show();   // prints "A" - decided at COMPILE-time based on reference type, NOT "B"!
```
**If they cross-question "Why can't static methods be overridden?"** — Say: "Overriding relies on dynamic method dispatch, which is based on the RUNTIME type of an object. Static methods are resolved at COMPILE-time based on the class reference, since they don't belong to any specific object — there's no 'runtime object' to dispatch on."

---

### 22. What is `this` keyword used for? Give all its uses.
**Answer:** `this` refers to the CURRENT object. Uses: 1) Resolving naming conflicts between instance fields and parameters (`this.name = name`). 2) Calling another constructor in the same class (`this(...)`, must be FIRST line). 3) Returning the current object for method chaining (`return this;`). 4) Passing the current object as an argument to another method.

---

### 23. What is `super` keyword used for?
**Answer:** `super` refers to the immediate PARENT class. Uses: 1) Accessing a parent's field/method that's been hidden/overridden by the child (`super.fieldName`, `super.method()`). 2) Calling the parent's constructor (`super(...)`, must be FIRST line in child constructor).

---

### 24. Difference between method overloading and overriding — explain deeply.
**Answer:**

| | Overloading | Overriding |
|---|---|---|
| Location | Same class | Parent-child (inheritance) |
| Signature | MUST differ (params) | MUST be identical |
| Resolved at | Compile-time | Runtime |
| Also called | Static/compile-time polymorphism | Dynamic/runtime polymorphism |

**If they cross-question "Why is one compile-time and the other runtime?"** — Say: "Overloading is resolved by looking at the NUMBER/TYPE of arguments you pass — the compiler can figure this out just by reading your code, without running it. Overriding is resolved based on the ACTUAL object's type at runtime (dynamic method dispatch) — since a parent reference could point to ANY subclass object, Java can't know WHICH version to call until the program is actually running and the real object type is known."

---

### 25. What is Dynamic Method Dispatch?
**Answer:** The mechanism by which Java decides, AT RUNTIME, which overridden method to actually execute — based on the ACTUAL object type (not the reference type). This is the core mechanism behind runtime polymorphism.
```java
Animal a = new Dog();
a.sound();   // calls Dog's sound(), decided at RUNTIME, even though 'a' is typed as Animal
```

---

### 26. Can we override the `main()` method?
**Answer:** No — `main()` is `static`, and static methods can't be overridden (only hidden, per Q21). You CAN technically write a `main()` in a subclass with the same signature, but it's method HIDING, not overriding — and JVM would still just call whichever class you explicitly run.

---

### 27. Abstract class vs Interface — full comparison.
**Answer:**

| | Abstract Class | Interface |
|---|---|---|
| Keyword | `abstract class` / `extends` | `interface` / `implements` |
| Inheritance | Single only | Multiple allowed |
| Constructors | Can have | Cannot have |
| Fields | Any type | `public static final` only (constants) |
| Methods | Abstract + concrete mixed freely | Traditionally abstract; now also default/static (Java 8+) |
| Use case | Shared code + strong "IS-A" relationship | Shared capability among unrelated classes |

**If they cross-question "When exactly would you pick one over the other?"** — Say: "If my classes are CLOSELY related and share common state/code (like Dog, Cat, Lion all extending Animal, sharing name/age fields and some default behavior), I use an abstract class. If I need to guarantee a CAPABILITY across totally unrelated classes (like Bird and Airplane both being Flyable), I use an interface, since a class can implement multiple interfaces but extend only one class."

---

### 28. Can an abstract class have a constructor? If it can't be instantiated, why would it need one?
**Answer:** Yes, it CAN have a constructor. Even though you can't do `new AbstractClass()` directly, the constructor still runs when a SUBCLASS object is created — because the subclass constructor implicitly (or explicitly via `super()`) calls the abstract class's constructor to initialize the inherited part of the object.

---

### 29. Can an interface have a constructor?
**Answer:** No. Interfaces cannot be instantiated at all (even indirectly the way abstract classes are, since interface "state" is limited to constants), so there's no need for — and no allowance for — a constructor in an interface.

---

### 30. How does Java achieve "multiple inheritance" using interfaces?
**Answer:** A class can `implements` MULTIPLE interfaces (unlike `extends`, limited to one class). Since interface methods (traditionally) have no implementation, there's no ambiguity about "whose version" to use — the implementing class ALWAYS provides its own single implementation. This sidesteps the Diamond Problem entirely.
```java
class FlyingCar implements Drivable, Flyable { ... }   // multiple interfaces - fine!
```

---

### 31. What is the Diamond Problem, and why doesn't Java allow multiple class inheritance?
**Answer:** If class C could extend BOTH class A and class B, and both A and B have a method with the SAME signature, C would face AMBIGUITY about which version to inherit — this ambiguity is called the Diamond Problem. Java's designers chose to simply DISALLOW multiple class inheritance entirely, to avoid this ambiguity, and instead solved the underlying need via interfaces (see Q30).

**If they cross-question "But interfaces with default methods CAN have implementations now — doesn't the Diamond Problem come back?"** — Say: "Good catch — if a class implements two interfaces that both provide the SAME default method, Java forces the implementing class to explicitly OVERRIDE that method and resolve the conflict itself (e.g., by explicitly choosing `InterfaceA.super.method()`). So Java pushes the decision to the DEVELOPER instead of allowing silent ambiguity."

---

### 32. What is a marker interface? Give an example.
**Answer:** A marker interface has NO methods at all — it exists purely to "tag"/mark a class as having a certain property, which other code can check via `instanceof`. Example: `Serializable` (marks a class as eligible for serialization — see Q158), `Cloneable` (marks a class as supporting `.clone()`).

---

### 33. What is a functional interface?
**Answer:** An interface with EXACTLY ONE abstract method (default/static methods don't count toward this limit). Enables lambda expressions to be used as its implementation. Marked optionally with `@FunctionalInterface` (compiler verifies the "one abstract method" rule).
```java
@FunctionalInterface
interface Calculator { int calculate(int a, int b); }
Calculator add = (a, b) -> a + b;
```

---

### 34. Explain all four access modifiers with their scope.
**Answer:**
- `public` — accessible everywhere
- `protected` — same package + subclasses (even in different packages)
- *default* (no modifier) — same package only
- `private` — same class only

Order from most to least restrictive: `private < default < protected < public`

---

### 35. Why use encapsulation (private fields + getters/setters) instead of public fields?
**Answer:** 1) **Validation** — setters can reject invalid values before storing them. 2) **Read-only/write-only control** — provide only a getter (no setter) to make a field effectively immutable from outside. 3) **Flexibility** — internal implementation can change later without breaking code that calls the getter/setter (the external "contract" stays the same). 4) **Debugging** — you can add logging inside setters to track exactly when/how values change.

---

### 36. Explain the three meanings of `final`.
**Answer:** `final` variable — value can't be REASSIGNED after initialization. `final` method — CANNOT be overridden by a subclass. `final` class — CANNOT be extended/inherited at all (e.g., `String` is final).

**If they cross-question "If I make an object reference final, can I still modify the object's internal state?"** — Say: "Yes — `final` only prevents REASSIGNING the reference to a different object. If the object itself is mutable (like a StringBuilder), you can still change its internal content freely."

---

### 37. Can a class be both `abstract` and `final`?
**Answer:** No — this is a direct contradiction. `abstract` means "this class MUST be extended to be useful" (can't instantiate directly). `final` means "this class CANNOT be extended." Combining them would create a class that can neither be instantiated NOR extended — completely useless — so Java disallows this combination as a compile error.

---

### 38. Difference between an abstract method and an interface method?
**Answer:** An abstract method (inside an abstract class) has NO body, and the containing class must be declared `abstract`. A traditional interface method is IMPLICITLY `public abstract` (no body) without needing an explicit `abstract` keyword. Since Java 8, interfaces can ALSO have `default` and `static` methods WITH a body — something abstract classes achieve via regular concrete methods instead.

---

### 39. Difference between Composition and Inheritance? When would you prefer one over the other?
**Answer:** **Inheritance** = "IS-A" relationship (Car IS-A Vehicle) — reuses code by EXTENDING a class. **Composition** = "HAS-A" relationship (Car HAS-A Engine) — reuses code by INCLUDING an object of another class as a field.
```java
class Car {
    Engine engine;   // composition - Car HAS an Engine
}
```
**Prefer composition when:** the relationship isn't a TRUE "is-a" (e.g., a Car isn't really a type of Engine), or when you want more FLEXIBILITY (you can swap out the Engine object at runtime; you can't swap out a parent class at runtime). There's a well-known OOP design principle: **"favor composition over inheritance"** — because inheritance creates tight coupling between parent and child, while composition is more flexible and modular.

---

### 40. Explain Association, Aggregation, and Composition.
**Answer:** All three describe relationships between classes, in increasing order of "ownership strength":
- **Association** — a general relationship where objects know about each other (e.g., a Teacher and a Student), but neither owns the other, and both can exist independently.
- **Aggregation** — a "HAS-A" relationship where one object contains another, but the contained object CAN exist independently (e.g., a Department HAS Employees, but Employees can exist even if the Department is deleted).
- **Composition** — a STRONGER "HAS-A" relationship where the contained object CANNOT exist independently of the container (e.g., a House HAS Rooms — if the House is destroyed, the Rooms cease to exist too).

---

### 41. What is object cloning? Shallow copy vs deep copy?
**Answer:** Cloning creates a COPY of an object using the `Cloneable` interface and `.clone()` method. **Shallow copy** — copies the object's primitive fields directly, but for REFERENCE fields (objects), it copies only the REFERENCE (address) — so both the original and the copy point to the SAME nested object. **Deep copy** — recursively copies EVERY nested object too, so the copy is COMPLETELY independent of the original.

```java
class Address { String city; }
class Person implements Cloneable {
    String name;
    Address address;
    // Shallow clone (default Object.clone()) - address reference is SHARED
    // Deep clone requires MANUALLY cloning address too, inside overridden clone()
}
```

**If they cross-question "What's the risk of a shallow copy?"** — Say: "If you modify the nested object through the COPY, it unexpectedly affects the ORIGINAL too, since they share the same reference — this is a common, subtle bug source."

---

### 42. Difference between instance initializer block and static block?
**Answer:** **Static block** — runs ONCE, when the class is first LOADED into memory (before any object is created). **Instance initializer block** — runs EVERY time an object is created, right BEFORE the constructor executes. Execution order: static block (once) → instance block (per object) → constructor (per object).

---

### 43. Can we have multiple classes in a single `.java` file?
**Answer:** Yes, but with ONE restriction: at most ONE class in the file can be `public`, and if there IS a public class, the FILE NAME must exactly match that public class's name. Any number of non-public (default access) classes can also exist in the same file.

---

### 44. What happens if you don't write ANY constructor in a class?
**Answer:** Java automatically provides a hidden **default no-argument constructor** that does nothing except allocate the object with default field values. However, if you write ANY constructor yourself (even a parameterized one), Java does NOT provide this default constructor anymore — you'd need to explicitly write a no-arg one yourself if you still want it.

---

### 45. What is the "IS-A" vs "HAS-A" test used for in OOP design?
**Answer:** It's a quick mental check to decide between Inheritance and Composition: ask "Is a Car A Vehicle?" (yes → inheritance makes sense) versus "Does a Car HAVE an Engine?" (yes → composition makes sense, since a Car isn't literally a type of Engine). This test helps avoid inheritance being misused just to "reuse code" when the relationship isn't truly hierarchical.

---

## SECTION C: Strings

### 46. Why is String immutable in Java? Give all the reasons.
**Answer:** 1) **String Pool safety** — if Strings were mutable, changing one variable could accidentally affect every other variable sharing that same pooled object. 2) **Security** — Strings are used for file paths, DB URLs, passwords; immutability prevents them being changed after a security check passes. 3) **Thread-safety** — immutable objects can be freely shared across threads without synchronization, since they can never change. 4) **Hashcode caching** — since content never changes, `hashCode()` can be computed once and cached, speeding up HashMap/HashSet operations using Strings as keys.

---

### 47. What is the String Constant Pool?
**Answer:** A special memory area (inside the Heap) where String LITERALS are stored. When you write `"Hello"`, Java checks if an identical String already exists in the pool — if yes, it reuses that SAME object (via a reference); if no, it creates a new one in the pool. This saves memory by avoiding duplicate String objects for identical literal values.

---

### 48. Difference between String, StringBuilder, and StringBuffer?
**Answer:** **String** — immutable; every "modification" creates a new object. **StringBuilder** — mutable, NOT thread-safe, faster; use in single-threaded code (most common case). **StringBuffer** — mutable, thread-safe (synchronized methods), slower; use only when multiple threads modify the same buffer.

---

### 49. Why does `s1 == s2` return true for two String literals, but `s1 == s3` return false when `s3` uses `new String(...)`?
**Answer:**
```java
String s1 = "Hello";
String s2 = "Hello";
String s3 = new String("Hello");
System.out.println(s1 == s2);   // true - both reuse the SAME pooled object
System.out.println(s1 == s3);   // false - 'new' FORCES creation of a separate object, bypassing the pool
```
Literals check the pool first and reuse existing objects; `new String(...)` explicitly creates a brand new object EVEN IF an identical value already exists in the pool.

---

### 50. How many objects are created by `new String("test")`?
**Answer:** Potentially **TWO**: one in the String Pool (for the literal `"test"`, IF it doesn't already exist there), and one in the regular Heap (from the explicit `new` call). If `"test"` ALREADY exists in the pool from earlier code, then only ONE new object is created (just the heap one) — the pool reference is reused, not recreated.

---

### 51. What does `String.intern()` do?
**Answer:** `.intern()` manually forces a String to be added to (or reused from) the String Pool, returning the pooled reference.
```java
String s1 = new String("Hello");   // heap object, NOT in pool
String s2 = s1.intern();             // now s2 points to the POOLED "Hello"
String s3 = "Hello";                  // also points to the pooled "Hello"
System.out.println(s2 == s3);        // true!
```

---

### 52. Why is the `String` class declared `final`?
**Answer:** To prevent subclassing. If `String` could be extended, a subclass could override its methods and break the immutability guarantees the ENTIRE Java ecosystem relies on (pool safety, security, hashcode caching — see Q46). Making it `final` ensures no one can create a "mutable String" via inheritance.

---

### 53. Difference between `.equals()` and `.compareTo()` for Strings?
**Answer:** `.equals()` returns a `boolean` — TRUE if content is identical, FALSE otherwise. `.compareTo()` returns an `int` — negative if the calling String comes BEFORE the argument alphabetically, zero if equal, positive if it comes AFTER. `.compareTo()` is used for SORTING; `.equals()` is used for equality CHECKS.

---

### 54. Can we use a String in a `switch` statement?
**Answer:** Yes, since Java 7. Java internally compares using `.equals()` and `hashCode()` under the hood (not `==`), so it correctly matches String VALUES, not references.

---

### 55. Why does StringBuilder outperform String concatenation in a loop?
**Answer:** Since String is immutable, `result = result + i` inside a loop creates a BRAND NEW String object on EVERY iteration, discarding the old one — wasteful for large loops. `StringBuilder` is MUTABLE — `.append()` modifies the SAME internal buffer object repeatedly, without creating throwaway objects each time, making it far more memory/time efficient for repeated modifications.

---

## SECTION D: Arrays & Collections Framework

### 56. 🔥 What is the difference between Array and ArrayList? (Very commonly asked!)
**Answer:**

| | Array | ArrayList |
|---|---|---|
| Size | Fixed at creation, cannot change | Dynamic — grows/shrinks automatically |
| Data types | Can hold primitives directly (`int[]`) | Can only hold OBJECTS (uses wrapper classes for primitives, via autoboxing) |
| Performance | Slightly faster (no overhead) | Slightly slower (dynamic resizing overhead) |
| Built-in methods | Very few (just `.length` property) | Many built-in methods (`add`, `remove`, `contains`, etc. — Day 9's full list) |
| Multi-dimensional | Directly supported (`int[][]`) | Requires ArrayList of ArrayLists |
| Type safety with generics | Not generic | Generic — compile-time type safety (`ArrayList<String>`) |

**If they cross-question "So when would you use a plain array instead of ArrayList?"** — Say: "When the size is KNOWN and FIXED upfront, and performance is critical (avoiding the dynamic-resize overhead), or when working with primitives directly without wrapper-class overhead — e.g., a fixed-size buffer or a matrix for numeric computation."

---

### 57. Difference between Array and LinkedList?
**Answer:** Array — fixed size, contiguous memory, fast random access by index (O(1)), SLOW insertion/deletion in the middle (must shift elements). LinkedList — dynamic size, non-contiguous memory (nodes linked via pointers), SLOW random access (must traverse from the start, O(n)), FAST insertion/deletion at the beginning/middle (just relink pointers, O(1)).

---

### 58. Can an array store elements of different types?
**Answer:** Not directly with a typed array (`int[]`, `String[]`) — all elements must be the SAME declared type. However, an `Object[]` array CAN store different types, since every class ultimately extends `Object` (Day 5) — though you'd need to cast elements back to their specific type before using type-specific methods.

---

### 59. Difference between ArrayList and LinkedList (in Collections context)?
**Answer:** ArrayList is backed by a dynamic array — fast `get(index)` (O(1)), slow insertion/deletion at the beginning/middle (O(n), must shift elements). LinkedList is backed by a doubly-linked list — slow `get(index)` (O(n), must traverse), fast insertion/deletion at the beginning/middle (O(1), just relink nodes). **Rule of thumb:** use ArrayList for frequent random access; use LinkedList for frequent insert/delete at the ends or middle.

---

### 60. Difference between ArrayList and Vector?
**Answer:** They're functionally very similar (both backed by a dynamic array), but **Vector is synchronized (thread-safe)**, while **ArrayList is NOT**. This makes Vector slower due to synchronization overhead. Vector is considered a LEGACY class (from Java 1.0) — modern code almost always prefers `ArrayList`, and uses `Collections.synchronizedList()` or `CopyOnWriteArrayList` (Q85) if thread-safety is specifically needed.

---

### 61. Difference between HashMap and Hashtable?
**Answer:**

| | HashMap | Hashtable |
|---|---|---|
| Thread-safe? | No | Yes (synchronized methods) |
| Null keys/values | ONE null key, multiple null values allowed | NO null keys OR values allowed (throws NullPointerException) |
| Performance | Faster | Slower (synchronization overhead) |
| Legacy? | Modern (part of Collections Framework) | Legacy (predates Collections Framework) |

**If they cross-question "So what should I use for thread-safety instead of Hashtable?"** — Say: "`ConcurrentHashMap` (Q81) — it's thread-safe like Hashtable, but with much better performance due to smarter internal locking (segment-based, not locking the entire map for every operation)."

---

### 62. Difference between HashMap and TreeMap?
**Answer:** HashMap — no guaranteed order, fastest (O(1) average for get/put). TreeMap — sorted by KEY automatically, slower (O(log n) for get/put, due to Red-Black Tree structure). Use TreeMap when you need keys in sorted order; use HashMap for pure fast lookup with no order requirement.

---

### 63. Difference between HashMap and LinkedHashMap?
**Answer:** Both are hash-based with similar performance, but LinkedHashMap ADDITIONALLY maintains INSERTION ORDER (via an internal doubly-linked list), while HashMap gives no order guarantee at all.

---

### 64. Difference between HashSet and TreeSet?
**Answer:** HashSet — no guaranteed order, fastest. TreeSet — sorted order (natural or via Comparator), slower due to tree-balancing operations.

---

### 65. Difference between List, Set, and Map?
**Answer:** **List** — ordered, allows duplicates, indexed access. **Set** — no duplicates, order depends on implementation. **Map** — key-value pairs, keys unique, NOT part of the `Collection` interface hierarchy at all (unlike List/Set).

---

### 66. 🔥 Explain HashMap's internal working in full detail.
**Answer:** HashMap uses an internal ARRAY of "buckets." When you `put(key, value)`: 1) Java computes `key.hashCode()`. 2) An internal formula converts this hash code into a bucket INDEX. 3) The entry is stored in that bucket. If a DIFFERENT key hashes to the SAME bucket (a "collision"), both entries are stored together in that bucket as a linked list (or a balanced tree, if the list grows too long — Java 8+ optimization). When you `get(key)`: Java recomputes the hash, finds the bucket, then walks through entries in that bucket calling `.equals()` on each key until it finds an EXACT match.

**If they cross-question "What happens when the map gets too full?"** — Say: "Once the map crosses its LOAD FACTOR threshold (default 0.75, meaning 75% full), Java automatically RESIZES the internal array (typically doubling it) and REHASHES every existing entry into new bucket positions, to keep lookups fast as the map grows."

---

### 67. Why must `equals()` and `hashCode()` be overridden TOGETHER?
**Answer:** Hash-based collections (HashMap, HashSet) use `hashCode()` to decide WHICH bucket to search, then `equals()` to confirm an EXACT match within that bucket. If you override `equals()` (changing what "equal" means) but leave the DEFAULT `hashCode()` (based on memory address), two "equal" objects could produce DIFFERENT hash codes, land in DIFFERENT buckets, and the collection would fail to recognize them as duplicates — breaking the collection's correctness.

**The formal contract:** if `a.equals(b)` is `true`, then `a.hashCode()` MUST equal `b.hashCode()`. (The reverse isn't required — unequal objects CAN share a hash code; that's just a normal collision.)

---

### 68. What is the equals() contract (reflexive, symmetric, transitive, consistent)?
**Answer:** A correctly implemented `equals()` must satisfy:
- **Reflexive:** `a.equals(a)` must be `true`
- **Symmetric:** if `a.equals(b)` is true, then `b.equals(a)` must ALSO be true
- **Transitive:** if `a.equals(b)` and `b.equals(c)`, then `a.equals(c)` must be true
- **Consistent:** repeated calls to `a.equals(b)` must return the same result, as long as neither object's relevant fields change
- **Non-null:** `a.equals(null)` must return `false`, never throw an exception

---

### 69. Comparable vs Comparator — full comparison.
**Answer:** **Comparable** — ONE natural sorting order, defined INSIDE the class via `compareTo()`, used by `Collections.sort(list)` with no second argument. **Comparator** — MULTIPLE custom sorting orders, defined OUTSIDE the class (separate class or lambda) via `compare()`, used by `Collections.sort(list, comparator)`.

**If they cross-question "Which package is each from?"** — Say: "Comparable is `java.lang`; Comparator is `java.util`. This itself hints at their design intent — Comparable is a core language-level contract about an object's own natural ordering, while Comparator is a utility for FLEXIBLE, EXTERNAL sorting strategies."

---

### 70. What is a Fail-Fast iterator vs Fail-Safe iterator?
**Answer:** **Fail-Fast** (e.g., ArrayList's default iterator) — throws `ConcurrentModificationException` IMMEDIATELY if the underlying collection is structurally modified WHILE iterating (except through the iterator's own `.remove()`). **Fail-Safe** (e.g., `CopyOnWriteArrayList`'s iterator) — iterates over a CLONED snapshot of the collection, so concurrent modifications to the original DON'T throw an exception (though the iterator won't reflect those changes either).

---

### 71. Why does removing an element during a for-each loop throw ConcurrentModificationException?
**Answer:** A for-each loop internally uses an Iterator behind the scenes. When you call `list.remove(x)` DIRECTLY (not through the iterator), the collection's internal "modification count" changes, but the hidden iterator doesn't know about it — on its NEXT `.next()` call, it detects this mismatch and throws the exception to protect you from undefined behavior. **Fix:** use the Iterator's OWN `.remove()` method instead, which properly updates the iterator's internal state.

---

### 72. What is the difference between the `Collection` interface and the `Collections` class?
**Answer:** `Collection` (singular) is the ROOT INTERFACE of the Collections Framework hierarchy (List, Set, Queue all extend it). `Collections` (plural) is a UTILITY CLASS full of static helper methods (`sort()`, `reverse()`, `max()`, `min()`, etc.) that operate ON collections.

---

### 73. Difference between HashMap and ConcurrentHashMap?
**Answer:** HashMap is NOT thread-safe — concurrent modification from multiple threads can corrupt its internal state or throw `ConcurrentModificationException`. `ConcurrentHashMap` IS thread-safe, but uses a more EFFICIENT locking strategy than the old `Hashtable` — instead of locking the ENTIRE map for every operation, it divides the map into segments/buckets and only locks the specific segment being modified, allowing multiple threads to work on DIFFERENT segments simultaneously.

---

### 74. Can a HashMap have a null key or null value?
**Answer:** Yes — HashMap allows exactly ONE `null` key, and MULTIPLE `null` values (across different keys). `Hashtable` and `ConcurrentHashMap`, by contrast, do NOT allow null keys OR null values at all (they throw `NullPointerException`).

---

### 75. How would you make an ArrayList thread-safe?
**Answer:** Options: 1) Wrap it using `Collections.synchronizedList(new ArrayList<>())` — adds synchronization to all methods, but you still need to manually synchronize during ITERATION. 2) Use `CopyOnWriteArrayList` (Q85) — better for read-heavy, write-rare scenarios. 3) Use explicit `synchronized` blocks around your OWN access code.

---

### 76. What is CopyOnWriteArrayList, and when would you use it?
**Answer:** A thread-safe variant of ArrayList where every WRITE operation (add/remove/set) creates a NEW COPY of the underlying array, leaving existing iterators/readers unaffected (they keep working on the OLD array). Best suited for scenarios with FREQUENT reads and RARE writes (e.g., a list of event listeners) — since copying the array on every write is expensive if writes are frequent.

---

### 77. Difference between `poll()`, `peek()`, and `remove()` methods (in Queue/Deque context)?
**Answer:** `peek()` — returns the element WITHOUT removing it; returns `null` if empty (no exception). `poll()` — removes AND returns the element; returns `null` if empty (no exception). `remove()` — removes and returns the element, but THROWS an exception (`NoSuchElementException`) if empty, instead of returning null.

---

### 78. What is a PriorityQueue, and how does it order elements?
**Answer:** A `PriorityQueue` is a queue where elements are NOT processed in strict insertion order — instead, they're ordered by PRIORITY, determined by their natural ordering (`Comparable`) or a supplied `Comparator`. The element with the HIGHEST priority (smallest value, by default — it's a MIN-heap) is always at the front, ready to be removed first via `poll()`.

---

### 79. What is Arrays.asList() and what's a common pitfall with it?
**Answer:** `Arrays.asList(array)` converts an array into a `List` view. **Pitfall:** the returned list is BACKED BY the original array and has a FIXED SIZE — calling `.add()` or `.remove()` on it throws `UnsupportedOperationException`. To get a truly resizable list, wrap it: `new ArrayList<>(Arrays.asList(array))`.

---

### 80. Why can't we use primitive types directly in generic collections (e.g., `ArrayList<int>`)?
**Answer:** Generics in Java work through **type erasure** — at RUNTIME, generic type information is erased, and the JVM essentially treats everything as `Object`. Since primitives (`int`, `char`, etc.) are NOT objects (Day 1), they can't be represented this way — hence Java REQUIRES wrapper classes (`Integer`, `Character`) for use with generics, and automatically autoboxes/unboxes as needed (`ArrayList<Integer>`, not `ArrayList<int>`).

---

## SECTION E: Exception Handling

### 81. Explain the full Exception hierarchy.
**Answer:** `Throwable` (root) → splits into `Error` (serious, unrecoverable JVM-level problems like OutOfMemoryError — generally NOT caught) and `Exception` (recoverable problems, which we handle). `Exception` further splits into CHECKED exceptions (like IOException, must be handled/declared) and `RuntimeException` (UNCHECKED exceptions, like NullPointerException, ArithmeticException — not enforced by the compiler).

---

### 82. Checked vs Unchecked exceptions — full explanation.
**Answer:** **Checked** — verified at COMPILE-time; the compiler FORCES you to either catch it or declare it with `throws`; represents EXTERNAL problems (file I/O, database, network) outside your code's direct control. **Unchecked** — NOT verified at compile-time; represents PROGRAMMING BUGS (null references, bad array indexes, division by zero) that should ideally be FIXED in the code, not just defensively caught everywhere.

**If they cross-question "How does Java know which is which?"** — Say: "By CLASS HIERARCHY — any exception class extending `RuntimeException` (or `Error`) is unchecked; anything else extending `Exception` directly is checked. It's a design-time classification by whoever WROTE the exception class, not a runtime property."

---

### 83. Explain the rules of try-catch-finally.
**Answer:** 1) `catch` blocks are checked TOP TO BOTTOM — order from MOST specific to LEAST specific, or you get a compile error (unreachable code). 2) `finally` ALWAYS executes — whether an exception was thrown, caught, uncaught, or even if there's a `return` in the try/catch (finally runs BEFORE the method actually returns). 3) The ONLY way `finally` doesn't run is if `System.exit()` is called, or the JVM crashes/is killed externally.

---

### 84. What is a multi-catch block?
**Answer:** Since Java 7, you can catch MULTIPLE exception types in ONE catch block using `|`, if you want to handle them the SAME way:
```java
try {
    // risky code
} catch (IOException | SQLException e) {
    System.out.println("Error: " + e.getMessage());
}
```
Reduces code duplication when different exceptions need identical handling logic.

---

### 85. How do you create a custom exception? When would you need one?
**Answer:** Extend `Exception` (for checked) or `RuntimeException` (for unchecked), and typically provide a constructor that passes a message to `super()`.
```java
class InsufficientBalanceException extends Exception {
    public InsufficientBalanceException(String message) { super(message); }
}
```
Use custom exceptions when built-in ones don't precisely describe your BUSINESS-SPECIFIC error scenario — they make error handling far more READABLE (an `InsufficientBalanceException` immediately communicates intent, versus a generic `Exception`).

---

### 86. Difference between `Error` and `Exception`?
**Answer:** `Error` represents SERIOUS, typically UNRECOVERABLE problems at the JVM/system level (OutOfMemoryError, StackOverflowError) — you generally DON'T try to catch/handle these. `Exception` represents problems your PROGRAM can and should handle gracefully (file not found, invalid input, etc.).

---

### 87. Can we have a try block WITHOUT a catch block?
**Answer:** Yes — `try` can be paired with JUST `finally` (no catch at all), if you simply want guaranteed cleanup code without actually handling any specific exception (the exception would still propagate up after `finally` runs).
```java
try {
    // risky code
} finally {
    // cleanup - always runs
}
```

---

### 88. Is there ANY scenario where `finally` does NOT execute?
**Answer:** Yes — if `System.exit()` is called INSIDE the try or catch block, the JVM terminates IMMEDIATELY, and `finally` is SKIPPED. Also, if the JVM itself crashes or is forcibly killed (e.g., power failure, `kill -9` on the process), `finally` obviously can't run either.

---

### 89. What is exception chaining?
**Answer:** Wrapping one exception INSIDE another, to preserve the ORIGINAL cause while throwing a higher-level, more meaningful exception.
```java
try {
    // low-level operation
} catch (SQLException e) {
    throw new RuntimeException("Failed to process order", e);   // 'e' is the CAUSE, chained in
}
```
This preserves the FULL stack trace of the original problem, accessible later via `.getCause()`, which is invaluable for debugging.

---

### 90. Difference between `throw` and `throws`?
**Answer:** `throw` is a STATEMENT that actually throws ONE exception object right now (`throw new IllegalArgumentException(...)`). `throws` is a DECLARATION in a method's SIGNATURE, warning callers that this method MIGHT throw a certain exception type, so they need to handle it.

---

### 91. What happens if an exception is thrown INSIDE a `finally` block?
**Answer:** If the `finally` block itself throws an exception, it SUPPRESSES/overrides any exception that was already being propagated from the try/catch — the NEW exception (from finally) is the one that actually propagates up, and the original one is essentially LOST (unless explicitly captured as a "suppressed exception," an advanced detail related to try-with-resources).

---

### 92. What is try-with-resources, and why is it preferred over manual finally-based cleanup?
**Answer:** A modern (Java 7+) syntax where resources implementing `AutoCloseable`/`Closeable` are declared inside `try(...)` parentheses, and Java AUTOMATICALLY calls `.close()` on them when the block finishes — success or failure — WITHOUT needing an explicit `finally` block. It's preferred because it's less error-prone (no risk of forgetting to close, or mishandling exceptions THROWN by close() itself) and much more concise than nested try-finally.

---

### 93. What are common causes of NullPointerException, and how do you prevent it?
**Answer:** Common causes: calling a method on a reference that's `null` (never initialized, or explicitly set to null, or returned as null from another method). Prevention: explicit null checks before use, using `Optional<T>` (Day 13) to make "might be missing" explicit in method signatures, and using `Objects.requireNonNull()` for early, clear failure at the source of the problem rather than deep inside unrelated code.

---

### 94. Can a method throw multiple different exceptions? How do you declare that?
**Answer:** Yes — list them comma-separated in the `throws` clause:
```java
void process() throws IOException, SQLException {
    // ...
}
```

---

### 95. What's the difference between `NullPointerException` and `ClassCastException`?
**Answer:** `NullPointerException` — occurs when you try to USE (call a method on, access a field of) a `null` reference. `ClassCastException` — occurs when you try to CAST an object to a type it's NOT actually compatible with (e.g., casting an `Animal` reference that's actually a `Cat` object into a `Dog`).

---

## SECTION F: Multithreading & Concurrency

### 96. Difference between a Process and a Thread?
**Answer:** A **process** is an independent program with its OWN dedicated memory space, running in isolation from other processes. A **thread** is a lightweight unit of execution WITHIN a process — multiple threads in the SAME process SHARE that process's memory (specifically the Heap), while each still has its own private Stack.

---

### 97. Thread vs Runnable — which is preferred, and why?
**Answer:** `Runnable` is generally preferred: 1) Java doesn't support multiple class inheritance — if your class `extends Thread`, it can't extend anything else; `implements Runnable` leaves that door open. 2) Cleaner separation of concerns — `Runnable` represents "a task," `Thread` represents "the worker executing it." 3) The SAME `Runnable` instance can be reused across MULTIPLE threads if needed.

---

### 98. Explain the full Thread lifecycle.
**Answer:** **New** (created, `.start()` not yet called) → **Runnable** (`.start()` called, ready, waiting for CPU) → **Running** (actively executing) → **Blocked/Waiting** (paused — waiting for a lock, or explicit `wait()`/`sleep()`) → **Terminated** (finished executing `run()`, or was stopped). The exact transition from Runnable to Running is controlled by the OS/JVM's thread SCHEDULER, not something the developer directly controls.

---

### 99. Difference between `.start()` and `.run()`?
**Answer:** `.start()` creates a genuinely NEW thread of execution, and the JVM calls `run()` ON that new thread. `.run()` called DIRECTLY just executes like a NORMAL method call, on the CURRENT thread — no new thread is created at all, completely defeating the purpose of using threads.

---

### 100. Difference between `wait()` and `sleep()`?
**Answer:**

| | `wait()` | `sleep()` |
|---|---|---|
| Belongs to | `Object` class | `Thread` class (static) |
| Releases the lock? | YES — releases the monitor lock while waiting | NO — keeps holding any lock it has |
| Needs `synchronized`? | YES — must be called within a synchronized context | NO — can be called anywhere |
| Resumes via | Another thread calling `notify()`/`notifyAll()` | Automatically, after the specified time elapses |

---

### 101. What is a Race Condition? Give a concrete example.
**Answer:** A bug that occurs when MULTIPLE threads access and modify SHARED data concurrently, without proper synchronization, leading to unpredictable/incorrect results. Classic example: two threads both executing `count++` on a shared `count` variable — since `count++` is actually 3 steps internally (READ, ADD, WRITE), both threads can READ the SAME old value before either WRITES back, causing one increment to be silently LOST.

---

### 102. What is a Deadlock? How do you prevent it?
**Answer:** A deadlock occurs when 2+ threads are stuck FOREVER, each waiting for a lock that the OTHER thread currently holds — neither can proceed. **Prevention:** always acquire MULTIPLE locks in a CONSISTENT, predetermined order across ALL threads (breaking the "circular wait" condition); avoid unnecessary nested locking; or use timeout-based lock attempts (`tryLock()`).

---

### 103. What are the four conditions necessary for a deadlock (Coffman conditions)?
**Answer:** 1) **Mutual Exclusion** — a resource can only be held by one thread at a time. 2) **Hold and Wait** — a thread holds one resource while waiting for another. 3) **No Preemption** — a lock can't be forcibly taken away; must be voluntarily released. 4) **Circular Wait** — a cycle of threads exists, each waiting on the next. ALL FOUR must be true simultaneously for a deadlock to occur — breaking even ONE condition prevents it.

---

### 104. Difference between a synchronized method and a synchronized block?
**Answer:** A synchronized METHOD locks the ENTIRE method body (coarse-grained). A synchronized BLOCK lets you lock ONLY the SPECIFIC critical section that touches shared data (fine-grained), leaving non-critical code OUTSIDE the block free to run without any locking overhead — generally BETTER for performance, since less code is locked.

---

### 105. What object does a synchronized method lock on? What about a static synchronized method?
**Answer:** A regular (instance-level) synchronized method locks on `this` (the current OBJECT). A `static synchronized` method locks on the CLASS object itself (e.g., `MyClass.class`) — since static members belong to the class, not any specific instance, and there's only ONE Class object regardless of how many instances exist.

---

### 106. What is the volatile keyword? How is it different from synchronized?
**Answer:** `volatile` ensures that reads/writes to a variable go DIRECTLY to main memory, rather than being cached in a thread's LOCAL CPU cache — guaranteeing that ALL threads always see the MOST RECENT value. However, `volatile` does NOT provide MUTUAL EXCLUSION (it doesn't prevent multiple threads from modifying the variable at the "same" time) — it only guarantees VISIBILITY of changes, not ATOMICITY of compound operations like `count++`. `synchronized` provides BOTH visibility AND mutual exclusion (only one thread executes the block at a time), but with more performance overhead.

**If they cross-question "So when would volatile be enough, without needing synchronized?"** — Say: "When you have a SIMPLE flag variable that one thread WRITES and others just READ (like a `stop` flag to signal a loop to end) — no compound read-modify-write operation is happening, just a straightforward visibility guarantee, which `volatile` handles efficiently without the overhead of full locking."

---

### 107. What is a Daemon thread?
**Answer:** A background thread that does NOT prevent the JVM from exiting. Once all NORMAL (non-daemon) threads finish, the JVM shuts down, abruptly terminating any remaining daemon threads. Real-world example: Java's own Garbage Collector runs as a daemon thread. Must call `setDaemon(true)` BEFORE `.start()`.

---

### 108. What is thread priority, and is it reliable?
**Answer:** An integer (1-10, default 5) that HINTS to the thread scheduler which threads should be preferred for CPU time. It is NOT a guarantee — actual behavior is platform/JVM-dependent. Never rely on priority for program CORRECTNESS; use proper synchronization primitives (`join()`, locks) for anything requiring guaranteed order/timing.

---

### 109. What is the Executor Framework / Thread Pool, and why use it over manually creating threads?
**Answer:** Creating a brand-new `Thread` for every task has real overhead (memory, CPU for thread creation/destruction). The Executor Framework (`ExecutorService`, `Executors.newFixedThreadPool(n)`) maintains a FIXED pool of REUSABLE worker threads that pick up tasks from an internal queue as they become free — far more efficient for applications handling MANY short-lived tasks, since threads are created ONCE and reused repeatedly.

---

### 110. Difference between Runnable and Callable?
**Answer:** `Runnable`'s `run()` method returns `void` and CANNOT throw checked exceptions. `Callable<V>`'s `call()` method RETURNS a value of type `V` and CAN throw checked exceptions. `Callable` is used with `ExecutorService.submit()`, returning a `Future<V>` object that lets you retrieve the result later (once the task completes).

---

### 111. What is a Future/CompletableFuture (brief overview)?
**Answer:** A `Future<V>` represents the RESULT of an asynchronous computation that may not have completed yet — you can check `.isDone()`, or call `.get()` to BLOCK until the result is available. `CompletableFuture` (Java 8+) is a more powerful, flexible version that supports CHAINING callbacks (e.g., "when this completes, THEN do this next"), without blocking — useful for building non-blocking, asynchronous pipelines.

---

### 112. What is ThreadLocal?
**Answer:** `ThreadLocal<T>` gives EACH thread its OWN independent copy of a variable — even though the `ThreadLocal` object itself might be shared/static, each thread reading/writing through it sees only ITS OWN value, completely isolated from other threads. Useful for per-thread context data (e.g., a database connection or user session specific to the currently executing thread) without needing explicit synchronization.

---

### 113. Explain wait(), notify(), and notifyAll() together with an example concept.
**Answer:** These enable THREAD COMMUNICATION. `wait()` — pauses the current thread and RELEASES its lock, until notified. `notify()` — wakes up ONE waiting thread (unspecified which, if multiple are waiting). `notifyAll()` — wakes up ALL waiting threads. Classic use case: Producer-Consumer pattern — a producer calls `notify()` after adding data, waking a consumer that was `wait()`-ing for data to become available.

---

### 114. Difference between Deadlock, Livelock, and Starvation?
**Answer:** **Deadlock** — threads stuck forever, each waiting for the other. **Livelock** — threads are ACTIVELY running/responding to each other (not blocked), but make NO real progress (e.g., two threads repeatedly yielding to each other, neither ever actually proceeding). **Starvation** — a thread is PERPETUALLY denied access to a needed resource because OTHER threads are repeatedly given priority (e.g., a low-priority thread that never gets CPU time because higher-priority threads keep getting scheduled first).

---

### 115. What are Atomic classes (e.g., AtomicInteger)? Why use them over synchronized?
**Answer:** Classes like `AtomicInteger`, `AtomicLong` (from `java.util.concurrent.atomic`) provide THREAD-SAFE operations on single variables WITHOUT needing explicit locks — using low-level CPU instructions (like Compare-And-Swap) internally. For SIMPLE operations like incrementing a counter, they're typically FASTER than `synchronized`, since they avoid the overhead of acquiring/releasing a full lock.
```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();   // thread-safe, no synchronized needed
```

---

## SECTION G: Java 8+ Modern Features

### 116. What is a Lambda expression? What problem does it solve?
**Answer:** A concise syntax `(params) -> body` for implementing a functional interface's single method, WITHOUT writing a full anonymous class. It solves the VERBOSITY problem of pre-Java-8 code, where even a simple one-line behavior required an entire `new SomeInterface() { public void method() {...} }` block.

---

### 117. What are the main built-in functional interfaces? Explain each.
**Answer:** `Supplier<T>` — takes NOTHING, RETURNS a value (`T get()`). `Consumer<T>` — takes a value, returns NOTHING (`void accept(T t)`). `Function<T,R>` — takes T, RETURNS R (`R apply(T t)`) — a transformation. `Predicate<T>` — takes a value, returns `boolean` (`boolean test(T t)`) — a condition check.

---

### 118. What is a method reference? Give an example.
**Answer:** A shorthand (`::`) for a lambda that JUST calls one existing method, without adding any extra logic.
```java
names.forEach(name -> System.out.println(name));   // lambda
names.forEach(System.out::println);                  // method reference - same behavior, shorter
```

---

### 119. Explain default and static methods in interfaces. Why were they introduced?
**Answer:** `default` methods have a BODY, and implementing classes can use them AS-IS or override them. `static` methods belong to the interface itself, called via the interface name. Introduced primarily for BACKWARD COMPATIBILITY — allowing NEW methods to be added to EXISTING interfaces (like in Java's standard library) WITHOUT breaking every class that already implements them (they simply inherit the default behavior automatically).

---

### 120. What are intermediate and terminal operations in Streams?
**Answer:** **Intermediate** — return ANOTHER stream, so you can KEEP chaining (`filter`, `map`, `sorted`, `distinct`, `limit`, `skip`). **Terminal** — END the pipeline and produce a final RESULT (`collect`, `forEach`, `count`, `reduce`). Streams are LAZY — intermediate operations don't actually execute until a terminal operation triggers the whole pipeline.

---

### 121. Difference between `map()` and `flatMap()` in Streams?
**Answer:** `map()` transforms each element 1-to-1 (one input → one output). `flatMap()` is used when each INPUT element itself produces a STREAM/COLLECTION of outputs (e.g., a `List<List<Integer>>`) — it FLATTENS all those nested streams into ONE single-level stream.
```java
List<List<Integer>> nested = Arrays.asList(Arrays.asList(1,2), Arrays.asList(3,4));
List<Integer> flat = nested.stream()
    .flatMap(List::stream)
    .collect(Collectors.toList());
System.out.println(flat);   // [1, 2, 3, 4]
```

---

### 122. Why can a Stream only be used/consumed once?
**Answer:** Once a TERMINAL operation runs on a Stream, the pipeline is considered "consumed" — internally, the stream's state is marked as closed, and attempting to reuse it throws `IllegalStateException`. This is a deliberate design choice — Streams model a ONE-TIME processing pipeline, not a reusable/storable data structure (that's what Collections are for).

---

### 123. What is the Optional class, and why use it over returning null?
**Answer:** `Optional<T>` explicitly represents "a value that might be present, or might be absent," making this possibility VISIBLE in the METHOD SIGNATURE itself. This forces callers to consciously HANDLE the missing case (via `.orElse()`, `.ifPresent()`, `.map()`) instead of accidentally forgetting a null check and hitting `NullPointerException` in production.

---

### 124. Is it bad practice to call `.get()` on an Optional directly? Why?
**Answer:** Yes — calling `.get()` WITHOUT first checking `.isPresent()` (or using safer alternatives like `.orElse()`, `.ifPresent()`, `.orElseThrow()`) just moves the crash risk from `NullPointerException` to `NoSuchElementException` — defeating the entire PURPOSE of using `Optional` in the first place.

---

### 125. What is `Collectors.groupingBy()` vs `Collectors.partitioningBy()`?
**Answer:** `groupingBy()` — groups stream elements into a `Map`, with potentially MANY different keys/groups, based on a classifier function. `partitioningBy()` — always splits elements into EXACTLY TWO groups (`true`/`false`), based on a `Predicate`.

---

### 126. What is a parallel stream? What's a risk with using it?
**Answer:** `.parallelStream()` (or `.stream().parallel()`) processes stream elements CONCURRENTLY across MULTIPLE threads (using a shared thread pool internally), potentially speeding up processing for LARGE datasets with CPU-intensive operations. **Risk:** if the operations involve SHARED MUTABLE STATE (like updating a shared counter without synchronization), you can reintroduce RACE CONDITIONS (Q101) — parallel streams are best used with STATELESS, independent operations.

---

### 127. What is the `var` keyword (Java 10+)? Is Java becoming dynamically typed?
**Answer:** `var` enables LOCAL variable TYPE INFERENCE — the compiler figures out the type from the assigned value, so you don't need to write it explicitly.
```java
var list = new ArrayList<String>();   // compiler infers: ArrayList<String>
```
**Important:** Java is STILL statically typed — the type is determined and FIXED at COMPILE-time (just inferred rather than explicitly written); it's NOT dynamically typed like Python, where a variable's type can change at runtime.

---

### 128. What are Records (Java 14+)? (Brief overview)
**Answer:** A concise syntax for creating IMMUTABLE data-carrier classes, automatically generating a constructor, getters, `equals()`, `hashCode()`, and `toString()` — eliminating a huge amount of REPETITIVE boilerplate for simple "data holder" classes.
```java
record Point(int x, int y) { }   // automatically gets constructor, getters, equals, hashCode, toString
```

---

## SECTION H: Memory Management & JVM Internals

### 129. Stack vs Heap memory — full comparison.
**Answer:**

| | Stack | Heap |
|---|---|---|
| Per-thread or shared? | One stack PER thread | SHARED across all threads |
| Stores | Method frames, local variables, references | Actual OBJECTS |
| Managed by | Automatic (LIFO push/pop on method call/return) | Garbage Collector |
| Speed | Very fast | Slower (more complex management) |
| Size | Limited (StackOverflowError if exceeded) | Larger, but can also run out (OutOfMemoryError) |

---

### 130. How does Garbage Collection determine what's "garbage"?
**Answer:** An object is eligible for GC when it's no longer REACHABLE — meaning no chain of references leads to it starting from any "GC Root" (active thread stacks, static variables, etc.). The GC uses a **Mark-and-Sweep** approach: MARK phase traces all reachable objects starting from roots; SWEEP phase reclaims memory from anything NOT marked; an optional COMPACT phase moves remaining objects together to reduce memory fragmentation.

---

### 131. Can you force garbage collection with `System.gc()`?
**Answer:** You can CALL `System.gc()`, but it's only a SUGGESTION/REQUEST to the JVM — there's NO GUARANTEE it will actually run immediately, or at all. Never rely on this for program correctness or timing.

---

### 132. Why is `finalize()` deprecated? What should be used instead?
**Answer:** `finalize()` was historically meant to let an object clean up resources before GC destroyed it, but it's DEPRECATED (since Java 9) because: 1) there's no guarantee it will EVER run (if GC never collects that object). 2) No guarantee of WHEN it runs, even if it does. 3) Performance overhead for objects that define it. 4) Can cause resource leaks due to unpredictable timing. **Modern replacement:** try-with-resources with `AutoCloseable`, which gives DETERMINISTIC, IMMEDIATE cleanup.

---

### 133. Is a memory leak possible in Java, despite automatic Garbage Collection?
**Answer:** Yes! A memory leak happens when objects remain REACHABLE (so GC can't collect them) but your program will NEVER actually use them again. Common causes: 1) `static` collections that keep growing and are never cleared. 2) Unclosed resources (files, DB connections). 3) Registered listeners/callbacks that are never unregistered. 4) Improperly implemented `equals()`/`hashCode()` causing HashMap/HashSet to accumulate orphaned entries.

---

### 134. What are ClassLoaders? Name the main types.
**Answer:** ClassLoaders are responsible for LOADING `.class` files into the JVM at runtime. The main built-in types (in a hierarchy): **Bootstrap ClassLoader** (loads core Java classes like `java.lang.*`), **Extension/Platform ClassLoader** (loads extension libraries), **Application/System ClassLoader** (loads YOUR application's classes, from the classpath). They follow a "delegation" model — a child classloader first asks its PARENT to load a class before trying itself.

---

### 135. What is the difference between Metaspace and the Heap (Java 8+)?
**Answer:** Before Java 8, class METADATA (class structure info, method info) was stored in a fixed-size area called "PermGen." Since Java 8, this was replaced by **Metaspace**, which is stored in NATIVE memory (not the regular Heap) and can GROW DYNAMICALLY (unlike PermGen's fixed size), reducing a common historical cause of `OutOfMemoryError: PermGen space`.

---

## SECTION I: Design Patterns & Miscellaneous

### 136. Explain the Singleton design pattern, and how to implement it in Java.
**Answer:** Ensures a class has EXACTLY ONE instance throughout the application, with a global access point to it. Implementation: private constructor (prevents outside instantiation, Q20) + a static method that returns the SAME single instance every time.
```java
class Singleton {
    private static Singleton instance;
    private Singleton() { }

    public static synchronized Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```
**If they cross-question "Why is `getInstance()` synchronized?"** — Say: "Without synchronization, TWO threads could BOTH see `instance == null` simultaneously, and each create a SEPARATE instance — violating the Singleton guarantee. Synchronizing this method prevents that race condition." (A more advanced answer might mention **double-checked locking** for better performance, avoiding synchronization overhead on every call after the instance is already created.)

---

### 137. What is Serialization? What is the `transient` keyword used for?
**Answer:** Serialization converts an object into a byte stream (for storage in a file, or transmission over a network), and Deserialization reverses this. A class must implement the MARKER interface `Serializable` (Q32) to support this. `transient` marks a field to be EXCLUDED from serialization — useful for sensitive data (like passwords) or fields that don't make sense to persist (like a temporary cache or a non-serializable object reference).

---

### 138. What is the Cloneable interface, and how does `.clone()` relate to it?
**Answer:** `Cloneable` is a MARKER interface (Q32) — implementing it signals that a class SUPPORTS cloning via `Object`'s `.clone()` method. If a class does NOT implement `Cloneable` and you call `.clone()` on it, it throws `CloneNotSupportedException`. By default, `.clone()` performs a SHALLOW copy (Q41) — you must override it manually for a DEEP copy.

---

### 139. What is Reflection in Java?
**Answer:** Reflection (`java.lang.reflect`) lets your program INSPECT and even MODIFY its own structure at RUNTIME — examining classes, methods, fields, and even INVOKING methods or accessing PRIVATE fields dynamically, without knowing them at compile-time. Used heavily by frameworks (like Spring) for dependency injection, and by testing libraries for accessing private members. **Trade-off:** powerful, but slower than direct code, and can break encapsulation if misused.

---

### 140. What is an enum in Java? Can it implement an interface?
**Answer:** `enum` defines a fixed set of NAMED CONSTANTS (e.g., `enum Day { MONDAY, TUESDAY, ... }`). Under the hood, each enum constant is actually a full OBJECT (an instance of the enum "class"), which means enums CAN have fields, constructors, and methods — and YES, an enum CAN implement an interface (though it can't EXTEND another class, since it implicitly already extends `java.lang.Enum`).
```java
enum Status implements Describable {
    ACTIVE, INACTIVE;
    public String describe() { return "Status: " + name(); }
}
```

---

### 141. What is the `instanceof` operator used for?
**Answer:** Checks whether an object is an instance of a specific class (or any of its subclasses/implemented interfaces), returning a `boolean`. Commonly used BEFORE downcasting, to avoid a `ClassCastException` at runtime (Q95).

---

### 142. What are the different types of nested classes in Java?
**Answer:** 1) **Static nested class** — declared `static` inside another class; doesn't need an instance of the outer class to exist. 2) **Inner class (non-static)** — needs an instance of the outer class; has implicit access to the outer object's members. 3) **Local class** — defined INSIDE a method body, scoped only to that method. 4) **Anonymous class** — a class with NO name, defined and instantiated in a SINGLE expression, often used before lambdas became common for implementing simple interfaces on-the-fly.

---

### 143. What is the difference between `length`, `length()`, and `size()`?
**Answer:** `length` — a FIELD (no parentheses) on ARRAYS (`arr.length`). `length()` — a METHOD on `String` (`str.length()`). `size()` — a METHOD on Collections (`list.size()`, `map.size()`). A very common beginner mix-up, and interviewers sometimes test this exact distinction directly.

---

### 144. What is the difference between shallow copy and a reference assignment?
**Answer:** A REFERENCE ASSIGNMENT (`Person p2 = p1;`) doesn't create ANY new object at all — both `p1` and `p2` point to the EXACT SAME object; modifying through either affects both identically. A SHALLOW COPY (Q41) creates a genuinely NEW top-level object, but its NESTED reference fields still point to the SAME underlying nested objects as the original.

---

### 145. What is the difference between `String.format()` and simple string concatenation?
**Answer:** `String.format()` uses PLACEHOLDER syntax (`%s`, `%d`, `%.2f`) to build formatted strings in a more READABLE, TEMPLATE-like way, especially useful for combining many values or controlling numeric formatting (decimal places, padding). Simple `+` concatenation works fine for SIMPLE cases, but becomes harder to read with MANY variables or specific formatting needs.
```java
String s = String.format("Name: %s, Age: %d", name, age);
```

---

### 146. What is the difference between an interface and an abstract class in terms of "state"?
**Answer:** Abstract classes CAN hold genuine mutable STATE (regular instance fields, changeable throughout the object's life). Interfaces can only hold `public static final` CONSTANTS (Q27) — they cannot maintain per-object mutable state at all, since there are no true "fields" in the traditional sense in an interface.

---

### 147. What is the purpose of the `Objects` utility class (e.g., `Objects.hash()`, `Objects.equals()`)?
**Answer:** `java.util.Objects` provides NULL-SAFE utility methods to simplify common tasks: `Objects.equals(a, b)` — safely compares two objects, correctly handling `null` (no NullPointerException if either is null). `Objects.hash(field1, field2, ...)` — conveniently combines MULTIPLE field values into one well-distributed hash code (exactly what's typically used inside an overridden `hashCode()` method, Q67).

---

### 148. What is method chaining, and how is it implemented?
**Answer:** A design pattern where methods RETURN the current object (`return this;`) so multiple method calls can be CHAINED together in a single expression, improving readability.
```java
class Builder {
    Builder setName(String name) { this.name = name; return this; }
    Builder setAge(int age) { this.age = age; return this; }
}
Builder b = new Builder().setName("Alice").setAge(25);   // chained calls
```
This is the foundation of the **Builder design pattern**, commonly used for objects with MANY optional configuration parameters.

---

### 149. What is the difference between a checked exception and an Error, in terms of design philosophy?
**Answer:** Checked exceptions represent problems your APPLICATION is expected to ANTICIPATE and RECOVER from (a file might not exist — handle it, maybe ask the user for a different path). `Error` represents problems SO SEVERE (running out of memory, JVM internal corruption) that MEANINGFUL recovery generally isn't possible — the philosophy is that catching an `Error` and "handling" it usually just delays an inevitable crash, rather than genuinely fixing anything.

---

### 150. If someone asks you to summarize "what makes Java, Java" in one minute, what would you say?
**Answer (a genuinely useful closing answer for interviews):** "Java is a statically-typed, object-oriented language that compiles to platform-independent bytecode, executed by the JVM. Its core strengths are: automatic memory management via garbage collection, a rich standard library (especially the Collections Framework), strong support for OOP principles (encapsulation, inheritance, polymorphism, abstraction), built-in multithreading support, and — since Java 8 — modern functional-style features like lambdas and Streams that make code more concise without sacrificing type safety. The combination of platform independence, strong tooling, and backward compatibility is why it's remained dominant in enterprise backend development for decades."

---

## 🎯 Final Tips for the Actual Interview

1. **Never just recite a definition — always have ONE concrete code example ready** for every topic above. Interviewers trust "I can show you" far more than "I can tell you."
2. **When you don't know something, say so honestly, then reason through it out loud.** Interviewers often care MORE about your problem-solving approach than a perfect memorized answer.
3. **Always mention the "why," not just the "what."** E.g., don't just say "ArrayList is faster for random access" — explain WHY (contiguous memory, direct index-based addressing) — this is what separates candidates who've MEMORIZED answers from those who genuinely UNDERSTAND.
4. **Expect follow-ups on EVERY answer.** If you explain HashMap, expect "what about collisions?" next. If you explain overriding, expect "what about static methods?" next. This file's **"If they cross-question..."** notes are designed to pre-empt exactly this pattern.
5. **Practice explaining OUT LOUD, not just reading silently.** Explaining forces you to organize your thoughts the way an interview actually demands — reading alone doesn't build that muscle.

Good luck! 🚀
