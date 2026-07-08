# Java Exception Handling — Complete Guide (Basic to Advanced)

---

## Table of Contents

1. [What is an Exception?](#1-what-is-an-exception)
2. [Exception Hierarchy](#2-exception-hierarchy)
3. [Types of Exceptions](#3-types-of-exceptions)
4. [try-catch Block](#4-try-catch-block)
5. [Multiple catch Blocks](#5-multiple-catch-blocks)
6. [finally Block](#6-finally-block)
7. [try-with-resources](#7-try-with-resources)
8. [throw Keyword](#8-throw-keyword)
9. [throws Keyword](#9-throws-keyword)
10. [Checked vs Unchecked Exceptions](#10-checked-vs-unchecked-exceptions)
11. [Custom Exceptions](#11-custom-exceptions)
12. [Exception Chaining](#12-exception-chaining)
13. [Multi-catch (Java 7+)](#13-multi-catch-java-7)
14. [Re-throwing Exceptions](#14-re-throwing-exceptions)
15. [Common Built-in Exceptions](#15-common-built-in-exceptions)
16. [Exception in Inheritance & Overriding](#16-exception-in-inheritance--overriding)
17. [Best Practices & Anti-Patterns](#17-best-practices--anti-patterns)
18. [Summary Cheat Sheet](#18-summary-cheat-sheet)

---

## 1. What is an Exception?

An **exception** is an **unexpected event** that occurs during program execution and disrupts the normal flow of instructions.

Without exception handling, your program would **crash** the moment something goes wrong.

```java
public class WithoutHandling {
    public static void main(String[] args) {
        int[] arr = {1, 2, 3};
        System.out.println(arr[10]); // ❌ Program crashes here!
        System.out.println("This line never runs.");
    }
}
```

**Output (crash):**
```
Exception in thread "main" java.lang.ArrayIndexOutOfBoundsException: Index 10 out of bounds for length 3
    at WithoutHandling.main(WithoutHandling.java:4)
```

With exception handling, you can **catch** the error and recover gracefully:

```java
public class WithHandling {
    public static void main(String[] args) {
        int[] arr = {1, 2, 3};
        try {
            System.out.println(arr[10]); // risky code
        } catch (ArrayIndexOutOfBoundsException e) {
            System.out.println("Caught: Index does not exist! " + e.getMessage());
        }
        System.out.println("Program continues normally."); // ✅ This runs
    }
}
```

**Output:**
```
Caught: Index does not exist! Index 10 out of bounds for length 3
Program continues normally.
```

---

## 2. Exception Hierarchy

Everything in Java's exception system is a class. Here's the full hierarchy:

```
java.lang.Object
    └── java.lang.Throwable
            ├── java.lang.Error                     ← JVM-level, don't catch these
            │       ├── OutOfMemoryError
            │       ├── StackOverflowError
            │       └── VirtualMachineError
            │
            └── java.lang.Exception                 ← Application-level problems
                    ├── IOException                 ← Checked Exceptions
                    │       ├── FileNotFoundException
                    │       └── SocketException
                    ├── SQLException
                    ├── ClassNotFoundException
                    │
                    └── RuntimeException            ← Unchecked Exceptions
                            ├── NullPointerException
                            ├── ArrayIndexOutOfBoundsException
                            ├── ClassCastException
                            ├── NumberFormatException
                            ├── ArithmeticException
                            ├── IllegalArgumentException
                            └── IllegalStateException
```

### Key Points:
- **`Throwable`** — root of all exceptions and errors
- **`Error`** — serious problems, JVM-level, you should NOT catch these
- **`Exception`** — problems your program can and should handle
- **`RuntimeException`** — subclass of Exception; unchecked (compiler doesn't force you to handle)

---

## 3. Types of Exceptions

### Checked Exceptions
- Must be handled at compile time (compiler enforces it)
- Subclasses of `Exception` but NOT `RuntimeException`
- Examples: `IOException`, `SQLException`, `ClassNotFoundException`

### Unchecked Exceptions
- NOT required to be caught or declared (compiler doesn't enforce)
- Subclasses of `RuntimeException`
- Examples: `NullPointerException`, `ArithmeticException`, `ArrayIndexOutOfBoundsException`

### Errors
- JVM-level severe problems
- You should NEVER catch these in normal applications
- Examples: `OutOfMemoryError`, `StackOverflowError`

```java
public class TypesDemo {
    public static void main(String[] args) {

        // ── Checked Exception ──────────────────────────────────
        // Compiler forces you to handle this:
        try {
            java.io.FileInputStream file = new java.io.FileInputStream("missing.txt");
        } catch (java.io.FileNotFoundException e) {
            System.out.println("Checked: " + e.getMessage());
        }

        // ── Unchecked Exception ────────────────────────────────
        // Compiler does NOT force you — but it will crash at runtime if unhandled
        try {
            int result = 10 / 0;
        } catch (ArithmeticException e) {
            System.out.println("Unchecked: " + e.getMessage()); // / by zero
        }

        // ── Error (do NOT catch in real apps) ──────────────────
        // Shown for understanding only
        try {
            recurse(); // causes StackOverflowError
        } catch (StackOverflowError e) {
            System.out.println("Error caught (only for demo!): StackOverflow");
        }
    }

    static void recurse() {
        recurse(); // infinite recursion
    }
}
```

---

## 4. try-catch Block

The fundamental structure for handling exceptions:

```java
try {
    // Code that might throw an exception
} catch (ExceptionType variableName) {
    // Code to handle the exception
}
```

### Detailed Example

```java
public class TryCatchDemo {
    public static void main(String[] args) {
        System.out.println("Step 1: Program starts");

        try {
            System.out.println("Step 2: Inside try block");
            String str = null;
            System.out.println(str.length()); // ← throws NullPointerException
            System.out.println("Step 3: This line is SKIPPED");
        } catch (NullPointerException e) {
            System.out.println("Step 3: Caught NullPointerException!");
            System.out.println("  Message : " + e.getMessage());
            System.out.println("  Class   : " + e.getClass().getName());
        }

        System.out.println("Step 4: Program continues after try-catch");
    }
}
```

**Output:**
```
Step 1: Program starts
Step 2: Inside try block
Step 3: Caught NullPointerException!
  Message : Cannot invoke "String.length()" because "str" is null
  Class   : java.lang.NullPointerException
Step 4: Program continues after try-catch
```

### Useful Exception Methods

```java
try {
    int[] arr = new int[5];
    arr[10] = 99;
} catch (ArrayIndexOutOfBoundsException e) {
    e.getMessage();          // Short message about the error
    e.toString();            // Class name + message
    e.printStackTrace();     // Full stack trace to console (great for debugging)
    e.getClass().getName();  // "java.lang.ArrayIndexOutOfBoundsException"
    e.getCause();            // What caused this exception (if chained)
}
```

---

## 5. Multiple catch Blocks

One `try` block can have many `catch` blocks — Java tries them **top to bottom** and runs the **first match**:

```java
public class MultipleCatchDemo {
    public static void main(String[] args) {
        int[] numbers = {10, 0, 5};
        String[] words = {"hello", null, "world"};

        for (int i = 0; i <= 3; i++) {  // i=3 causes ArrayIndexOutOfBoundsException
            try {
                System.out.println("\n--- Iteration " + i + " ---");
                int result = 100 / numbers[i];           // could be ArithmeticException
                System.out.println("Division result: " + result);
                System.out.println("Word length: " + words[i].length()); // could be NPE
            } catch (ArithmeticException e) {
                System.out.println("Cannot divide by zero: " + e.getMessage());
            } catch (NullPointerException e) {
                System.out.println("String is null, can't get length!");
            } catch (ArrayIndexOutOfBoundsException e) {
                System.out.println("Index out of range: " + e.getMessage());
            }
        }
    }
}
```

**Output:**
```
--- Iteration 0 ---
Division result: 10
Word length: 5

--- Iteration 1 ---
Cannot divide by zero: / by zero

--- Iteration 2 ---
Division result: 20
Word length is null, can't get length!

--- Iteration 3 ---
Index out of range: Index 3 out of bounds for length 3
```

### ⚠️ Order Matters — Parent Must Come After Child

```java
// ❌ WRONG — Exception is parent of ArithmeticException
// The second catch is UNREACHABLE — compiler error!
try {
    int x = 1 / 0;
} catch (Exception e) {          // catches everything
    System.out.println("General");
} catch (ArithmeticException e) { // ❌ Compile error: already caught above
    System.out.println("Specific");
}

// ✅ CORRECT — specific (child) before general (parent)
try {
    int x = 1 / 0;
} catch (ArithmeticException e) { // specific first
    System.out.println("ArithmeticException caught");
} catch (Exception e) {           // general last
    System.out.println("Some other exception");
}
```

---

## 6. finally Block

The `finally` block **ALWAYS executes** — whether an exception occurred or not. Perfect for cleanup code (closing files, DB connections, etc.).

```java
try {
    // risky code
} catch (Exception e) {
    // handle exception
} finally {
    // ALWAYS runs — cleanup here
}
```

### Example

```java
public class FinallyDemo {
    public static void main(String[] args) {
        System.out.println("=== Case 1: No exception ===");
        divide(10, 2);

        System.out.println("\n=== Case 2: Exception occurs ===");
        divide(10, 0);
    }

    static void divide(int a, int b) {
        try {
            System.out.println("Trying to divide...");
            int result = a / b;
            System.out.println("Result: " + result);
        } catch (ArithmeticException e) {
            System.out.println("Caught: " + e.getMessage());
        } finally {
            System.out.println("Finally always runs! Cleanup done.");
        }
    }
}
```

**Output:**
```
=== Case 1: No exception ===
Trying to divide...
Result: 5
Finally always runs! Cleanup done.

=== Case 2: Exception occurs ===
Trying to divide...
Caught: / by zero
Finally always runs! Cleanup done.
```

### finally vs return

`finally` even runs when there's a `return` in try/catch:

```java
public class FinallyWithReturn {
    public static void main(String[] args) {
        System.out.println("Returned: " + getValue());
    }

    static String getValue() {
        try {
            return "from try";     // return is noted...
        } finally {
            System.out.println("finally still runs!");  // ...but this runs first
            // If you return here, it OVERRIDES the try's return
        }
    }
}
```

**Output:**
```
finally still runs!
Returned: from try
```

---

## 7. try-with-resources

Introduced in **Java 7**. Automatically closes resources (files, connections, streams) that implement `AutoCloseable`. No need for a `finally` block to close them.

### Old Way (Before Java 7)

```java
import java.io.*;

public class OldWay {
    public static void main(String[] args) {
        BufferedReader reader = null;
        try {
            reader = new BufferedReader(new FileReader("data.txt"));
            String line = reader.readLine();
            System.out.println(line);
        } catch (IOException e) {
            System.out.println("Error: " + e.getMessage());
        } finally {
            if (reader != null) {
                try {
                    reader.close(); // must manually close, even this can throw!
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

### New Way — try-with-resources ✅

```java
import java.io.*;

public class TryWithResourcesDemo {
    public static void main(String[] args) {
        // Resource is declared in the parentheses — auto-closed when block exits
        try (BufferedReader reader = new BufferedReader(new FileReader("data.txt"))) {
            String line;
            while ((line = reader.readLine()) != null) {
                System.out.println(line);
            }
        } catch (IOException e) {
            System.out.println("Error reading file: " + e.getMessage());
        }
        // reader.close() is called AUTOMATICALLY — even if exception occurs
    }
}
```

### Multiple Resources (closed in reverse order)

```java
import java.io.*;

public class MultipleResources {
    public static void main(String[] args) {
        // Both are auto-closed; closed in REVERSE order: writer first, then reader
        try (
            BufferedReader reader = new BufferedReader(new FileReader("input.txt"));
            BufferedWriter writer = new BufferedWriter(new FileWriter("output.txt"))
        ) {
            String line;
            while ((line = reader.readLine()) != null) {
                writer.write(line.toUpperCase());
                writer.newLine();
            }
            System.out.println("File copied successfully!");
        } catch (IOException e) {
            System.out.println("IO Error: " + e.getMessage());
        }
    }
}
```

### Custom AutoCloseable Resource

```java
public class CustomResource implements AutoCloseable {
    private String name;

    public CustomResource(String name) {
        this.name = name;
        System.out.println(name + ": Opened");
    }

    public void doWork() {
        System.out.println(name + ": Doing work...");
    }

    @Override
    public void close() {
        System.out.println(name + ": Closed automatically!");
    }
}

public class CustomResourceDemo {
    public static void main(String[] args) {
        try (CustomResource res = new CustomResource("DatabaseConnection")) {
            res.doWork();
            // exception or not — close() will be called
        }
    }
}
```

**Output:**
```
DatabaseConnection: Opened
DatabaseConnection: Doing work...
DatabaseConnection: Closed automatically!
```

---

## 8. throw Keyword

Use `throw` to **manually throw** an exception anywhere in your code:

```java
throw new SomeException("message");
```

### Example

```java
public class ThrowDemo {
    static int divide(int a, int b) {
        if (b == 0) {
            throw new ArithmeticException("Denominator cannot be zero!"); // manual throw
        }
        return a / b;
    }

    static void validateAge(int age) {
        if (age < 0) {
            throw new IllegalArgumentException("Age cannot be negative: " + age);
        }
        if (age < 18) {
            throw new IllegalStateException("Must be 18 or older. Given: " + age);
        }
        System.out.println("Age " + age + " is valid.");
    }

    public static void main(String[] args) {
        // Test divide
        try {
            System.out.println(divide(10, 2));  // 5
            System.out.println(divide(10, 0));  // throws!
        } catch (ArithmeticException e) {
            System.out.println("Caught: " + e.getMessage());
        }

        // Test validateAge
        try { validateAge(25); }   catch (Exception e) { System.out.println(e.getMessage()); }
        try { validateAge(-5); }   catch (Exception e) { System.out.println(e.getMessage()); }
        try { validateAge(15); }   catch (Exception e) { System.out.println(e.getMessage()); }
    }
}
```

**Output:**
```
5
Caught: Denominator cannot be zero!
Age 25 is valid.
Age cannot be negative: -5
Must be 18 or older. Given: 15
```

---

## 9. throws Keyword

Use `throws` in a **method signature** to declare that the method **may throw** a checked exception. This shifts responsibility to the **caller**:

```java
returnType methodName() throws CheckedException { ... }
```

### Example

```java
import java.io.*;

public class ThrowsDemo {

    // Method declares it may throw IOException — caller must handle it
    static String readFile(String path) throws IOException {
        BufferedReader reader = new BufferedReader(new FileReader(path));
        String content = reader.readLine();
        reader.close();
        return content;
    }

    // This method also propagates the exception upward
    static void processFile(String path) throws IOException {
        String data = readFile(path);  // doesn't handle — passes to caller
        System.out.println("Data: " + data);
    }

    public static void main(String[] args) {
        // main() is the final caller — must handle it here
        try {
            processFile("nonexistent.txt");
        } catch (IOException e) {
            System.out.println("File error: " + e.getMessage());
        }
    }
}
```

### throw vs throws — Key Difference

| Feature   | `throw`                              | `throws`                                    |
|-----------|--------------------------------------|---------------------------------------------|
| Purpose   | Actually throws an exception object  | Declares that method *might* throw          |
| Location  | Inside method body                   | In method signature                         |
| Followed by | An exception instance              | Exception class name(s)                     |
| Number    | Only one exception at a time         | Multiple: `throws IOException, SQLException`|

```java
// throw — inside method body, throws an instance
void checkAge(int age) {
    throw new IllegalArgumentException("Too young"); // ← instance
}

// throws — method declaration, lists class names
void readData() throws IOException, SQLException { // ← class names
    // ...
}
```

---

## 10. Checked vs Unchecked Exceptions

### Checked Exceptions — must be handled

```java
import java.io.*;
import java.sql.*;

public class CheckedDemo {

    // Must declare throws or use try-catch — compiler enforces this
    static void readFile() throws IOException {
        FileInputStream fis = new FileInputStream("file.txt"); // checked
        fis.close();
    }

    static void connectDB() throws SQLException {
        // If using JDBC
        // Connection conn = DriverManager.getConnection("jdbc:mysql://..."); // checked
    }

    public static void main(String[] args) {
        // Option 1: Handle with try-catch
        try {
            readFile();
        } catch (IOException e) {
            System.out.println("Handled: " + e.getMessage());
        }

        // Option 2: Propagate with throws (in main — less common)
    }
}
```

### Unchecked Exceptions — optional to catch

```java
public class UncheckedDemo {

    // No throws declaration needed — compiles fine either way
    static int getElement(int[] arr, int index) {
        return arr[index]; // may throw ArrayIndexOutOfBoundsException
    }

    static String toUpperCase(String s) {
        return s.toUpperCase(); // may throw NullPointerException
    }

    public static void main(String[] args) {
        // Catching unchecked — optional but good practice when you expect it
        try {
            System.out.println(getElement(new int[]{1, 2, 3}, 10));
        } catch (ArrayIndexOutOfBoundsException e) {
            System.out.println("Bad index: " + e.getMessage());
        }

        try {
            System.out.println(toUpperCase(null));
        } catch (NullPointerException e) {
            System.out.println("Null string passed!");
        }
    }
}
```

---

## 11. Custom Exceptions

You can create your **own exception classes** for domain-specific errors.

### Basic Custom Exception

```java
// Custom checked exception — extends Exception
class InsufficientFundsException extends Exception {
    private double amount;

    public InsufficientFundsException(double amount) {
        super("Insufficient funds. Short by: ₹" + amount);
        this.amount = amount;
    }

    public double getShortfall() {
        return amount;
    }
}

class BankAccount {
    private String owner;
    private double balance;

    public BankAccount(String owner, double balance) {
        this.owner = owner;
        this.balance = balance;
    }

    public void withdraw(double amount) throws InsufficientFundsException {
        if (amount > balance) {
            throw new InsufficientFundsException(amount - balance);
        }
        balance -= amount;
        System.out.println("Withdrawn: ₹" + amount + " | Balance: ₹" + balance);
    }

    public double getBalance() { return balance; }
}

public class CustomExceptionDemo {
    public static void main(String[] args) {
        BankAccount account = new BankAccount("Alice", 5000.0);

        try {
            account.withdraw(2000); // ✅ OK
            account.withdraw(4000); // ❌ Not enough funds
        } catch (InsufficientFundsException e) {
            System.out.println("Transaction failed: " + e.getMessage());
            System.out.println("Shortfall amount: ₹" + e.getShortfall());
        }
    }
}
```

**Output:**
```
Withdrawn: ₹2000.0 | Balance: ₹3000.0
Transaction failed: Insufficient funds. Short by: ₹1000.0
Shortfall amount: ₹1000.0
```

---

### Custom Unchecked Exception

```java
// Custom unchecked exception — extends RuntimeException
class InvalidProductException extends RuntimeException {
    private String productCode;

    public InvalidProductException(String productCode) {
        super("Product not found: " + productCode);
        this.productCode = productCode;
    }

    // Chaining constructor
    public InvalidProductException(String productCode, Throwable cause) {
        super("Product lookup failed for: " + productCode, cause);
        this.productCode = productCode;
    }

    public String getProductCode() { return productCode; }
}

class ProductService {
    public String getProduct(String code) {
        if (code == null || code.isEmpty()) {
            throw new IllegalArgumentException("Product code cannot be empty");
        }
        if (!code.startsWith("PRD")) {
            throw new InvalidProductException(code); // no try-catch needed at call site
        }
        return "Product: " + code;
    }
}

public class UncheckedCustomDemo {
    public static void main(String[] args) {
        ProductService service = new ProductService();

        try {
            System.out.println(service.getProduct("PRD001")); // ✅
            System.out.println(service.getProduct("ABC999")); // ❌ invalid
        } catch (InvalidProductException e) {
            System.out.println("Error: " + e.getMessage());
            System.out.println("Code: " + e.getProductCode());
        }
    }
}
```

---

### Custom Exception Hierarchy

```java
// Base exception for your application
class AppException extends RuntimeException {
    private int errorCode;

    public AppException(int errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }

    public int getErrorCode() { return errorCode; }
}

// Specific exceptions extending the base
class DatabaseException extends AppException {
    public DatabaseException(String message) {
        super(1001, message);
    }
}

class NetworkException extends AppException {
    public NetworkException(String message) {
        super(1002, message);
    }
}

class AuthenticationException extends AppException {
    public AuthenticationException(String username) {
        super(1003, "Authentication failed for user: " + username);
    }
}

public class ExceptionHierarchyDemo {
    static void login(String user, String pass) {
        if (!pass.equals("secret")) {
            throw new AuthenticationException(user);
        }
        System.out.println("Login successful for: " + user);
    }

    public static void main(String[] args) {
        try {
            login("alice", "wrong");
        } catch (AuthenticationException e) {
            System.out.println("Auth error [" + e.getErrorCode() + "]: " + e.getMessage());
        } catch (AppException e) {
            // Catches any other AppException subclass
            System.out.println("App error [" + e.getErrorCode() + "]: " + e.getMessage());
        }
    }
}
```

---

## 12. Exception Chaining

When you catch one exception and throw another, **wrap the original** as the cause so you don't lose the root error:

```java
import java.io.*;

class DataLoadException extends Exception {
    public DataLoadException(String message, Throwable cause) {
        super(message, cause); // ← wrap the original cause
    }
}

class UserRepository {
    public String loadUser(int id) throws DataLoadException {
        try {
            // Simulating a file read failure
            throw new IOException("Disk read error on sector 7");
        } catch (IOException e) {
            // Wrap the low-level IOException in a high-level DataLoadException
            throw new DataLoadException("Failed to load user with ID: " + id, e);
        }
    }
}

public class ExceptionChainingDemo {
    public static void main(String[] args) {
        UserRepository repo = new UserRepository();

        try {
            repo.loadUser(42);
        } catch (DataLoadException e) {
            System.out.println("High-level error: " + e.getMessage());
            System.out.println("Root cause: " + e.getCause().getMessage());
            System.out.println("\n--- Full Stack Trace ---");
            e.printStackTrace();
        }
    }
}
```

**Output:**
```
High-level error: Failed to load user with ID: 42
Root cause: Disk read error on sector 7

--- Full Stack Trace ---
DataLoadException: Failed to load user with ID: 42
    at UserRepository.loadUser(...)
Caused by: java.io.IOException: Disk read error on sector 7
    at UserRepository.loadUser(...)
```

---

## 13. Multi-catch (Java 7+)

Catch multiple unrelated exception types in a **single catch block** using `|`:

```java
import java.io.*;
import java.sql.*;

public class MultiCatchDemo {
    public static void main(String[] args) {
        // ❌ Old way — repetitive code
        try {
            riskyMethod();
        } catch (IOException e) {
            System.out.println("Logging error: " + e.getMessage());
        } catch (SQLException e) {
            System.out.println("Logging error: " + e.getMessage()); // duplicate!
        }

        // ✅ New way — clean multi-catch (Java 7+)
        try {
            riskyMethod();
        } catch (IOException | SQLException e) {
            // One handler for multiple unrelated exceptions
            System.out.println("Caught: " + e.getClass().getSimpleName() + " → " + e.getMessage());
        }
    }

    static void riskyMethod() throws IOException, SQLException {
        // Simulating exception
        throw new IOException("File not found");
    }
}
```

> **Note:** Multi-catch variable `e` is implicitly `final` — you cannot reassign it inside the catch block.

```java
// ❌ This causes compile error
catch (IOException | SQLException e) {
    e = new IOException("new");  // Error: cannot reassign multi-catch variable
}
```

---

## 14. Re-throwing Exceptions

Sometimes you catch an exception, do some logging/cleanup, then **re-throw** it:

```java
import java.io.*;

public class RethrowDemo {

    // Re-throw same exception
    static void processFile(String path) throws IOException {
        try {
            BufferedReader reader = new BufferedReader(new FileReader(path));
            reader.readLine();
        } catch (IOException e) {
            System.out.println("Logging error: " + e.getMessage());
            throw e; // re-throw to caller
        }
    }

    // Wrap and re-throw as different exception
    static void loadConfig(String path) throws RuntimeException {
        try {
            processFile(path);
        } catch (IOException e) {
            throw new RuntimeException("Config load failed", e); // wrap + rethrow
        }
    }

    public static void main(String[] args) {
        try {
            loadConfig("config.properties");
        } catch (RuntimeException e) {
            System.out.println("Final handler: " + e.getMessage());
            System.out.println("Original cause: " + e.getCause().getMessage());
        }
    }
}
```

---

## 15. Common Built-in Exceptions

### NullPointerException

```java
public class NPEDemo {
    public static void main(String[] args) {
        String s = null;

        try {
            System.out.println(s.length()); // NPE
        } catch (NullPointerException e) {
            System.out.println("NPE: " + e.getMessage());
        }

        // Prevention — null checks
        if (s != null) {
            System.out.println(s.length());
        }

        // Modern — Optional (Java 8+)
        java.util.Optional<String> opt = java.util.Optional.ofNullable(s);
        System.out.println(opt.map(String::length).orElse(-1)); // -1
    }
}
```

### ArrayIndexOutOfBoundsException

```java
public class ArrayDemo {
    public static void main(String[] args) {
        int[] arr = {10, 20, 30};

        try {
            System.out.println(arr[5]); // valid: 0, 1, 2
        } catch (ArrayIndexOutOfBoundsException e) {
            System.out.println("Bad index: " + e.getMessage());
        }

        // Prevention
        int index = 5;
        if (index >= 0 && index < arr.length) {
            System.out.println(arr[index]);
        } else {
            System.out.println("Index out of range");
        }
    }
}
```

### NumberFormatException

```java
public class NumberFormatDemo {
    public static void main(String[] args) {
        String[] inputs = {"123", "45.6", "abc", null};

        for (String input : inputs) {
            try {
                int num = Integer.parseInt(input);
                System.out.println("Parsed: " + num);
            } catch (NumberFormatException e) {
                System.out.println("Cannot parse '" + input + "': " + e.getMessage());
            } catch (NullPointerException e) {
                System.out.println("Input was null!");
            }
        }
    }
}
```

**Output:**
```
Parsed: 123
Cannot parse '45.6': For input string: "45.6"
Cannot parse 'abc': For input string: "abc"
Input was null!
```

### ClassCastException

```java
public class CastDemo {
    public static void main(String[] args) {
        Object obj = "Hello World";

        try {
            Integer num = (Integer) obj; // String cannot be cast to Integer!
        } catch (ClassCastException e) {
            System.out.println("Cast failed: " + e.getMessage());
        }

        // Prevention — use instanceof
        if (obj instanceof String) {
            String s = (String) obj; // safe now
            System.out.println("Safe cast: " + s);
        }

        // Java 16+ — pattern matching
        if (obj instanceof String s) { // cast + bind in one
            System.out.println("Pattern match: " + s.toUpperCase());
        }
    }
}
```

### StackOverflowError

```java
public class StackOverflowDemo {
    static int count = 0;

    static void recurse() {
        count++;
        recurse(); // infinite recursion → StackOverflowError
    }

    public static void main(String[] args) {
        try {
            recurse();
        } catch (StackOverflowError e) {
            System.out.println("Stack overflow after " + count + " calls!");
        }
    }
}
```

---

## 16. Exception in Inheritance & Overriding

There are strict rules about exceptions when overriding methods:

### Rules

1. Overriding method can declare **fewer or no checked exceptions**
2. Overriding method **cannot declare new or broader checked exceptions**
3. Overriding method **can declare any unchecked exception** freely

```java
import java.io.*;

class Parent {
    void method() throws IOException {
        System.out.println("Parent method");
    }
}

class ChildA extends Parent {
    @Override
    void method() throws FileNotFoundException { // ✅ FileNotFoundException is narrower than IOException
        System.out.println("ChildA method");
    }
}

class ChildB extends Parent {
    @Override
    void method() { // ✅ No exception — also valid (fewer exceptions)
        System.out.println("ChildB method");
    }
}

// ❌ This would be a compile error:
// class ChildC extends Parent {
//     @Override
//     void method() throws Exception { // ❌ Exception is BROADER than IOException
//     }
// }

class ChildD extends Parent {
    @Override
    void method() throws RuntimeException { // ✅ Unchecked — always OK
        System.out.println("ChildD method");
    }
}
```

---

## 17. Best Practices & Anti-Patterns

### ✅ Best Practices

```java
// 1. Catch specific exceptions, not Exception
// ❌ Bad
try { ... } catch (Exception e) { ... }

// ✅ Good
try { ... } catch (FileNotFoundException e) { ... }


// 2. Never silently swallow exceptions
// ❌ Very bad — hides bugs!
try {
    riskyOperation();
} catch (Exception e) {
    // empty — error vanishes silently
}

// ✅ At minimum, log it
try {
    riskyOperation();
} catch (Exception e) {
    System.err.println("Error in riskyOperation: " + e.getMessage());
    // or use a logging framework: logger.error("...", e);
}


// 3. Use finally or try-with-resources for cleanup
// ✅ Always close resources
try (Connection conn = getConnection()) {
    // use connection
} catch (SQLException e) {
    System.out.println("DB error: " + e.getMessage());
}


// 4. Don't use exceptions for flow control
// ❌ Bad — expensive and misleading
try {
    int value = Integer.parseInt(input);
} catch (NumberFormatException e) {
    value = 0; // using exception as an if-else
}

// ✅ Good — check first
if (input.matches("\\d+")) {
    int value = Integer.parseInt(input);
} else {
    int value = 0;
}


// 5. Preserve the original cause when re-throwing
// ❌ Loses root cause
catch (IOException e) {
    throw new ServiceException("Failed"); // original cause lost!
}

// ✅ Wrap with cause
catch (IOException e) {
    throw new ServiceException("Failed", e); // root cause preserved
}


// 6. Restore interrupt status when catching InterruptedException
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // ✅ restore the flag
    return;
}


// 7. Add context to exception messages
// ❌ Vague
throw new IllegalArgumentException("Invalid input");

// ✅ Specific and actionable
throw new IllegalArgumentException(
    "Expected age between 0 and 150, got: " + age + " for user: " + userId
);
```

### ❌ Anti-Patterns

```java
// Anti-pattern 1: Catching Throwable / Error
try { ... } catch (Throwable t) { ... }  // ❌ catches OutOfMemoryError etc.


// Anti-pattern 2: Returning null in catch block (leads to NPE later)
String readConfig() {
    try {
        return readFile("config.txt");
    } catch (IOException e) {
        return null; // ❌ caller gets null and NPE elsewhere
    }
}
// ✅ Use Optional or throw a meaningful exception


// Anti-pattern 3: Using printStackTrace() in production
catch (Exception e) {
    e.printStackTrace(); // ❌ goes to stderr, not your log system
}
// ✅ Use a logger: logger.error("message", e);


// Anti-pattern 4: catch block that's wider than needed
try {
    connectToDatabase();
    readFile();
    parseData();
} catch (Exception e) {
    handleIt(e); // ❌ Which operation failed? Can't tell!
}
// ✅ Separate try-catch blocks or specific exceptions


// Anti-pattern 5: Exception in finally block
try {
    doWork();
} finally {
    close(); // ❌ If close() throws, it hides the original exception from doWork()!
}
// ✅ Wrap finally code in its own try-catch
```

---

## 18. Summary Cheat Sheet

### Keywords

| Keyword   | Used For                                                    |
|-----------|-------------------------------------------------------------|
| `try`     | Wrap code that might throw an exception                     |
| `catch`   | Handle a specific exception type                            |
| `finally` | Code that always runs (cleanup)                             |
| `throw`   | Manually throw an exception object                          |
| `throws`  | Declare that a method may throw a checked exception         |

### Checked vs Unchecked

| Feature              | Checked Exception         | Unchecked Exception           |
|----------------------|---------------------------|-------------------------------|
| Superclass           | `Exception`               | `RuntimeException`            |
| Must handle?         | ✅ Yes (compiler enforced) | ❌ No (optional)               |
| Common examples      | `IOException`, `SQLException` | `NPE`, `ArithmeticException`  |
| When to use (custom) | Recoverable scenarios     | Programming bugs / logic errors|

### Exception Execution Flow

```
try block starts
    ↓
Exception thrown?
    ├── NO  → try block finishes → finally runs → continue
    └── YES → skip rest of try → find matching catch
                  ├── Match found  → catch block runs → finally runs → continue
                  └── No match     → finally runs → exception propagates up
```

### Custom Exception Template

```java
// Checked custom exception
public class MyCheckedException extends Exception {
    private int errorCode;

    public MyCheckedException(int code, String message) {
        super(message);
        this.errorCode = code;
    }

    public MyCheckedException(int code, String message, Throwable cause) {
        super(message, cause); // always provide chaining constructor
        this.errorCode = code;
    }

    public int getErrorCode() { return errorCode; }
}

// Unchecked custom exception
public class MyRuntimeException extends RuntimeException {
    public MyRuntimeException(String message) { super(message); }
    public MyRuntimeException(String message, Throwable cause) { super(message, cause); }
}
```

### Quick Decision Guide

```
Should I catch this exception?
        ↓
Is it a checked exception?
    YES → must catch or declare throws
    NO  → optional; only if you can meaningfully handle it

Can I recover from this exception?
    YES → catch and handle gracefully
    NO  → let it propagate (or wrap in RuntimeException)

Should I create a custom exception?
    YES if: domain-specific error, need extra fields, clearer API
    NO if: standard exception (IllegalArgumentException, IOException) is sufficient
```

---

*Master exceptions and your Java programs will be robust, clear, and maintainable. 🛡️*
