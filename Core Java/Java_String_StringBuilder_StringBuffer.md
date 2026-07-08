# Java String, StringBuilder & StringBuffer — Complete Reference Guide

---

## Table of Contents

1. [Overview & Key Differences](#1-overview--key-differences)
2. [Detailed Comparison Table](#2-detailed-comparison-table)
3. [String Methods](#3-string-methods)
4. [StringBuilder Methods](#4-stringbuilder-methods)
5. [StringBuffer Methods](#5-stringbuffer-methods)
6. [When to Use Which](#6-when-to-use-which)
7. [Performance Benchmark](#7-performance-benchmark)

---

## 1. Overview & Key Differences

### String (java.lang.String)

`String` is **immutable** — once created, its value cannot be changed. Every modification (like concatenation) creates a **new String object** in the heap. String literals are stored in the **String Constant Pool**.

```java
String s = "Hello";
s = s + " World"; // New object created; original "Hello" still in memory
```

### StringBuilder (java.lang.StringBuilder)

`StringBuilder` is **mutable** — you can modify the content without creating new objects. It is **NOT thread-safe** (no synchronization). Best for use in **single-threaded** environments.

```java
StringBuilder sb = new StringBuilder("Hello");
sb.append(" World"); // Same object modified
```

### StringBuffer (java.lang.StringBuffer)

`StringBuffer` is **mutable** just like StringBuilder, but it is **thread-safe** because all its methods are `synchronized`. Best used in **multi-threaded** environments.

```java
StringBuffer sbf = new StringBuffer("Hello");
sbf.append(" World"); // Thread-safe modification
```

---

## 2. Detailed Comparison Table

| Feature              | String              | StringBuilder        | StringBuffer         |
|----------------------|---------------------|----------------------|----------------------|
| **Mutability**       | Immutable           | Mutable              | Mutable              |
| **Thread Safety**    | Yes (immutable)     | No                   | Yes (synchronized)   |
| **Performance**      | Slow (new objects)  | Fast                 | Moderate (sync overhead) |
| **Storage**          | String Pool + Heap  | Heap                 | Heap                 |
| **Introduced In**    | JDK 1.0             | JDK 1.5              | JDK 1.0              |
| **Use Case**         | Fixed strings       | Single-threaded ops  | Multi-threaded ops   |
| **Memory**           | More (extra objects)| Less                 | Less                 |
| **equals() override**| Yes                 | No (uses Object)     | No (uses Object)     |

### Memory Visualization

```
String s1 = "Hello";
String s2 = s1 + " World";

Stack          Heap / String Pool
------         ------------------
s1  ---------> [ "Hello" ]         ← Still exists
s2  ---------> [ "Hello World" ]   ← New object

StringBuilder sb = new StringBuilder("Hello");
sb.append(" World");

Stack          Heap
------         ----
sb  ---------> [ "Hello World" ]   ← Same object modified
```

---

## 3. String Methods

> All String methods return a **new String** (original is unchanged).

### 3.1 `length()`
Returns the number of characters in the string.

```java
String s = "Hello";
System.out.println(s.length()); // 5
```

---

### 3.2 `charAt(int index)`
Returns the character at the specified index (0-based).

```java
String s = "Hello";
System.out.println(s.charAt(1)); // 'e'
System.out.println(s.charAt(4)); // 'o'
```

---

### 3.3 `indexOf(String str)` / `indexOf(char ch)`
Returns the index of the **first occurrence** of the substring or character. Returns `-1` if not found.

```java
String s = "Hello World";
System.out.println(s.indexOf('o'));       // 4
System.out.println(s.indexOf("World"));  // 6
System.out.println(s.indexOf('z'));       // -1
```

---

### 3.4 `lastIndexOf(String str)`
Returns the index of the **last occurrence** of the substring.

```java
String s = "Hello World Hello";
System.out.println(s.lastIndexOf("Hello")); // 12
```

---

### 3.5 `substring(int beginIndex)` / `substring(int beginIndex, int endIndex)`
Extracts a portion of the string. `endIndex` is **exclusive**.

```java
String s = "Hello World";
System.out.println(s.substring(6));      // "World"
System.out.println(s.substring(0, 5));   // "Hello"
```

---

### 3.6 `toLowerCase()` / `toUpperCase()`
Converts the string to lower or upper case.

```java
String s = "Hello World";
System.out.println(s.toLowerCase()); // "hello world"
System.out.println(s.toUpperCase()); // "HELLO WORLD"
```

---

### 3.7 `trim()`
Removes leading and trailing **whitespace** (spaces, tabs, newlines).

```java
String s = "  Hello World  ";
System.out.println(s.trim()); // "Hello World"
```

---

### 3.8 `strip()` *(Java 11+)*
Similar to `trim()` but handles **Unicode whitespace** correctly.

```java
String s = "  Hello  ";
System.out.println(s.strip()); // "Hello"
```

---

### 3.9 `replace(char old, char new)` / `replace(String old, String new)`
Replaces all occurrences of a character or substring.

```java
String s = "Hello World";
System.out.println(s.replace('l', 'r'));         // "Herro Worrd"
System.out.println(s.replace("World", "Java"));  // "Hello Java"
```

---

### 3.10 `replaceAll(String regex, String replacement)`
Replaces all matches of a **regular expression**.

```java
String s = "Hello 123 World 456";
System.out.println(s.replaceAll("[0-9]+", "#")); // "Hello # World #"
```

---

### 3.11 `replaceFirst(String regex, String replacement)`
Replaces only the **first** regex match.

```java
String s = "Hello 123 World 456";
System.out.println(s.replaceFirst("[0-9]+", "#")); // "Hello # World 456"
```

---

### 3.12 `contains(CharSequence s)`
Returns `true` if the string contains the given sequence.

```java
String s = "Hello World";
System.out.println(s.contains("World")); // true
System.out.println(s.contains("Java"));  // false
```

---

### 3.13 `startsWith(String prefix)` / `endsWith(String suffix)`
Checks if the string starts or ends with a given string.

```java
String s = "Hello World";
System.out.println(s.startsWith("Hello")); // true
System.out.println(s.endsWith("World"));   // true
System.out.println(s.startsWith("World")); // false
```

---

### 3.14 `equals(Object o)` / `equalsIgnoreCase(String s)`
Compares string content. `==` compares references, not content!

```java
String s1 = "Hello";
String s2 = "hello";
System.out.println(s1.equals(s2));             // false
System.out.println(s1.equalsIgnoreCase(s2));   // true
```

---

### 3.15 `compareTo(String s)` / `compareToIgnoreCase(String s)`
Lexicographically compares two strings. Returns:
- `0` if equal
- Negative if current < argument
- Positive if current > argument

```java
System.out.println("apple".compareTo("banana")); // negative (a < b)
System.out.println("banana".compareTo("apple")); // positive
System.out.println("apple".compareTo("apple"));  // 0
```

---

### 3.16 `split(String regex)` / `split(String regex, int limit)`
Splits the string around matches of a regex. Returns a `String[]`.

```java
String s = "apple,banana,cherry";
String[] fruits = s.split(",");
for (String f : fruits) {
    System.out.println(f); // apple, banana, cherry (each on new line)
}

// With limit — stops splitting after 2 parts
String[] parts = s.split(",", 2);
System.out.println(parts[0]); // "apple"
System.out.println(parts[1]); // "banana,cherry"
```

---

### 3.17 `join(CharSequence delimiter, CharSequence... elements)`
Joins strings with a delimiter. Static method.

```java
String result = String.join(", ", "apple", "banana", "cherry");
System.out.println(result); // "apple, banana, cherry"

List<String> list = Arrays.asList("a", "b", "c");
System.out.println(String.join("-", list)); // "a-b-c"
```

---

### 3.18 `format(String format, Object... args)`
Creates a formatted string. Static method.

```java
String s = String.format("Name: %s, Age: %d, Score: %.2f", "Alice", 25, 98.5);
System.out.println(s); // "Name: Alice, Age: 25, Score: 98.50"
```

---

### 3.19 `valueOf(various types)`
Converts other types to String. Static method.

```java
System.out.println(String.valueOf(42));      // "42"
System.out.println(String.valueOf(3.14));    // "3.14"
System.out.println(String.valueOf(true));    // "true"
System.out.println(String.valueOf('A'));     // "A"
```

---

### 3.20 `toCharArray()`
Converts the string to a character array.

```java
String s = "Hello";
char[] chars = s.toCharArray();
for (char c : chars) {
    System.out.print(c + " "); // H e l l o
}
```

---

### 3.21 `isEmpty()` / `isBlank()` *(isBlank — Java 11+)*
`isEmpty()` — true if length is 0.
`isBlank()` — true if empty or contains only whitespace.

```java
System.out.println("".isEmpty());    // true
System.out.println(" ".isEmpty());   // false
System.out.println(" ".isBlank());   // true
System.out.println("Hi".isBlank());  // false
```

---

### 3.22 `intern()`
Returns the canonical representation of the string from the String Pool.

```java
String s1 = new String("Hello");
String s2 = s1.intern();
String s3 = "Hello";
System.out.println(s2 == s3); // true (both point to pool)
```

---

### 3.23 `matches(String regex)`
Returns `true` if the entire string matches the regex.

```java
System.out.println("12345".matches("[0-9]+"));    // true
System.out.println("Hello".matches("[A-Za-z]+"));  // true
System.out.println("Hello123".matches("[A-Za-z]+")); // false
```

---

### 3.24 `codePointAt(int index)`
Returns the Unicode code point of the character at the index.

```java
String s = "A";
System.out.println(s.codePointAt(0)); // 65
```

---

### 3.25 `concat(String str)`
Appends a string to the end. (Less commonly used than `+`)

```java
String s = "Hello";
System.out.println(s.concat(" World")); // "Hello World"
```

---

### 3.26 `repeat(int count)` *(Java 11+)*
Repeats the string `count` times.

```java
String s = "Ha";
System.out.println(s.repeat(3)); // "HaHaHa"
```

---

### 3.27 `lines()` *(Java 11+)*
Returns a stream of lines from the string.

```java
String s = "Line1\nLine2\nLine3";
s.lines().forEach(System.out::println);
// Line1
// Line2
// Line3
```

---

### 3.28 `stripLeading()` / `stripTrailing()` *(Java 11+)*
Removes whitespace from only the leading or only the trailing end.

```java
String s = "  Hello  ";
System.out.println(s.stripLeading());  // "Hello  "
System.out.println(s.stripTrailing()); // "  Hello"
```

---

### 3.29 `getBytes()`
Encodes the string into a byte array using the platform's default charset.

```java
String s = "Hello";
byte[] bytes = s.getBytes();
System.out.println(Arrays.toString(bytes)); // [72, 101, 108, 108, 111]
```

---

### 3.30 `hashCode()`
Returns the hash code of the string (consistent with `equals()`).

```java
System.out.println("Hello".hashCode()); // 69609650
System.out.println("Hello".hashCode()); // 69609650 (same always)
```

---

## 4. StringBuilder Methods

> `StringBuilder` is in `java.lang` and is mutable. All modification methods **return the same `StringBuilder` object** (enabling method chaining).

### 4.1 `append(various types)`
Appends data of any type to the end. Supports `String`, `int`, `double`, `boolean`, `char`, `char[]`, `Object`, etc.

```java
StringBuilder sb = new StringBuilder("Hello");
sb.append(" ").append("World").append("!").append(42);
System.out.println(sb); // "Hello World!42"
```

---

### 4.2 `insert(int offset, various types)`
Inserts data at the specified position.

```java
StringBuilder sb = new StringBuilder("Hello World");
sb.insert(5, ",");
System.out.println(sb); // "Hello, World"

sb.insert(0, ">>> ");
System.out.println(sb); // ">>> Hello, World"
```

---

### 4.3 `delete(int start, int end)`
Removes characters from `start` (inclusive) to `end` (exclusive).

```java
StringBuilder sb = new StringBuilder("Hello World");
sb.delete(5, 11);
System.out.println(sb); // "Hello"
```

---

### 4.4 `deleteCharAt(int index)`
Removes the character at the specified index.

```java
StringBuilder sb = new StringBuilder("Hello");
sb.deleteCharAt(1);
System.out.println(sb); // "Hllo"
```

---

### 4.5 `replace(int start, int end, String str)`
Replaces characters from `start` to `end` with the given string.

```java
StringBuilder sb = new StringBuilder("Hello World");
sb.replace(6, 11, "Java");
System.out.println(sb); // "Hello Java"
```

---

### 4.6 `reverse()`
Reverses the character sequence.

```java
StringBuilder sb = new StringBuilder("Hello");
sb.reverse();
System.out.println(sb); // "olleH"
```

---

### 4.7 `charAt(int index)`
Returns the character at the given index.

```java
StringBuilder sb = new StringBuilder("Hello");
System.out.println(sb.charAt(1)); // 'e'
```

---

### 4.8 `setCharAt(int index, char ch)`
Sets the character at the given index.

```java
StringBuilder sb = new StringBuilder("Hello");
sb.setCharAt(0, 'J');
System.out.println(sb); // "Jello"
```

---

### 4.9 `indexOf(String str)` / `lastIndexOf(String str)`
Finds the first/last occurrence of a substring.

```java
StringBuilder sb = new StringBuilder("Hello World Hello");
System.out.println(sb.indexOf("Hello"));     // 0
System.out.println(sb.lastIndexOf("Hello")); // 12
```

---

### 4.10 `substring(int start)` / `substring(int start, int end)`
Returns a substring (does **not** modify the StringBuilder).

```java
StringBuilder sb = new StringBuilder("Hello World");
System.out.println(sb.substring(6));      // "World"
System.out.println(sb.substring(0, 5));   // "Hello"
```

---

### 4.11 `length()`
Returns the current length of the character sequence.

```java
StringBuilder sb = new StringBuilder("Hello");
System.out.println(sb.length()); // 5
```

---

### 4.12 `capacity()`
Returns the current capacity (default is 16 + initial length). Capacity auto-grows as needed.

```java
StringBuilder sb = new StringBuilder();
System.out.println(sb.capacity()); // 16 (default)

StringBuilder sb2 = new StringBuilder("Hello");
System.out.println(sb2.capacity()); // 21 (16 + 5)
```

---

### 4.13 `ensureCapacity(int minCapacity)`
Ensures the capacity is at least the specified minimum.

```java
StringBuilder sb = new StringBuilder();
sb.ensureCapacity(50);
System.out.println(sb.capacity()); // at least 50
```

---

### 4.14 `trimToSize()`
Reduces the capacity to match the current length (saves memory).

```java
StringBuilder sb = new StringBuilder("Hello");
sb.ensureCapacity(100);
System.out.println(sb.capacity()); // 100+
sb.trimToSize();
System.out.println(sb.capacity()); // 5
```

---

### 4.15 `toString()`
Converts StringBuilder to a String.

```java
StringBuilder sb = new StringBuilder("Hello World");
String result = sb.toString();
System.out.println(result.getClass().getName()); // java.lang.String
```

---

### 4.16 `setLength(int newLength)`
Sets the length. If shorter, truncates. If longer, fills with null characters `'\u0000'`.

```java
StringBuilder sb = new StringBuilder("Hello World");
sb.setLength(5);
System.out.println(sb); // "Hello"
```

---

### 4.17 Method Chaining Example

```java
String result = new StringBuilder()
    .append("Java")
    .append(" ")
    .append("is")
    .append(" ")
    .append("awesome!")
    .reverse()
    .toString();
System.out.println(result); // "!emosewa si avaJ"
```

---

## 5. StringBuffer Methods

> `StringBuffer` has **exactly the same methods** as `StringBuilder`, but all methods are `synchronized`. Use it when multiple threads share a buffer.

### All methods below are identical to StringBuilder in behavior:

| Method | Description |
|--------|-------------|
| `append(...)` | Thread-safe append |
| `insert(int, ...)` | Thread-safe insert |
| `delete(int, int)` | Thread-safe delete |
| `deleteCharAt(int)` | Thread-safe char removal |
| `replace(int, int, String)` | Thread-safe replace |
| `reverse()` | Thread-safe reverse |
| `charAt(int)` | Get character |
| `setCharAt(int, char)` | Set character |
| `indexOf(String)` | Find substring |
| `lastIndexOf(String)` | Find last substring |
| `substring(int)` | Extract substring |
| `length()` | Current length |
| `capacity()` | Buffer capacity |
| `ensureCapacity(int)` | Ensure min capacity |
| `trimToSize()` | Trim to content size |
| `toString()` | Convert to String |
| `setLength(int)` | Resize buffer |

### 5.1 Multi-threaded Example — Why StringBuffer is Needed

```java
// WRONG — StringBuilder is not thread-safe
class Counter {
    private StringBuilder sb = new StringBuilder();

    public void addNumber(int n) {
        sb.append(n); // Race condition possible!
    }
}

// CORRECT — StringBuffer is thread-safe
class SafeCounter {
    private StringBuffer sb = new StringBuffer();

    public void addNumber(int n) {
        sb.append(n); // Synchronized — safe
    }
}

// Test with threads
StringBuffer sb = new StringBuffer();
Runnable task = () -> {
    for (int i = 0; i < 100; i++) {
        sb.append("A");
    }
};

Thread t1 = new Thread(task);
Thread t2 = new Thread(task);
t1.start(); t2.start();
t1.join(); t2.join();
System.out.println(sb.length()); // Guaranteed: 200
```

---

### 5.2 StringBuffer Specific: Thread-safe Operations

```java
StringBuffer buffer = new StringBuffer();

// All these are synchronized — safe across threads:
buffer.append("Hello");
buffer.insert(0, ">>> ");
buffer.replace(4, 9, "World");
buffer.delete(0, 4);
buffer.reverse();
System.out.println(buffer.toString());
```

---

## 6. When to Use Which

```
Use String     → When value won't change (config keys, constant messages, map keys)
Use StringBuilder → Single-threaded string building (loops, parsing, formatting)
Use StringBuffer  → Multi-threaded shared string building (rare in modern code)
```

### Decision Tree

```
Is the string going to change?
│
├─ NO  → Use String
│
└─ YES → Will multiple threads access it concurrently?
         │
         ├─ YES → Use StringBuffer
         │
         └─ NO  → Use StringBuilder  ✅ (most common choice)
```

### Practical Example — String Concatenation in a Loop

```java
// ❌ BAD — String (creates 10,000 new objects!)
String s = "";
for (int i = 0; i < 10000; i++) {
    s += i;  // Each iteration: new String object
}

// ✅ GOOD — StringBuilder (modifies same object)
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10000; i++) {
    sb.append(i);
}
String result = sb.toString();
```

---

## 7. Performance Benchmark

Approximate relative performance for 100,000 concatenations:

| Approach | Time (relative) | Memory |
|----------|----------------|--------|
| String (`+=`) | Very Slow (~1000x) | Very High |
| StringBuffer | Fast | Low |
| StringBuilder | Fastest | Low |

```java
// Quick benchmark
long start, end;

// String
start = System.nanoTime();
String s = "";
for (int i = 0; i < 100000; i++) s += "a";
end = System.nanoTime();
System.out.println("String:        " + (end - start) / 1_000_000 + " ms");

// StringBuilder
start = System.nanoTime();
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 100000; i++) sb.append("a");
sb.toString();
end = System.nanoTime();
System.out.println("StringBuilder: " + (end - start) / 1_000_000 + " ms");

// StringBuffer
start = System.nanoTime();
StringBuffer sbf = new StringBuffer();
for (int i = 0; i < 100000; i++) sbf.append("a");
sbf.toString();
end = System.nanoTime();
System.out.println("StringBuffer:  " + (end - start) / 1_000_000 + " ms");

// Typical output:
// String:        ~4500 ms
// StringBuilder: ~5 ms
// StringBuffer:  ~8 ms
```

---

## Quick Summary of All Methods

### String-Only Methods
| Method | Purpose |
|--------|---------|
| `intern()` | Get pooled instance |
| `matches(regex)` | Full regex match |
| `replaceAll(regex, str)` | Replace all regex matches |
| `replaceFirst(regex, str)` | Replace first regex match |
| `split(regex)` | Split into array |
| `join(delim, ...)` | Join with delimiter (static) |
| `format(...)` | Formatted string (static) |
| `valueOf(...)` | Convert to String (static) |
| `toCharArray()` | To char array |
| `getBytes()` | To byte array |
| `repeat(n)` | Repeat string (Java 11) |
| `lines()` | Stream of lines (Java 11) |
| `isBlank()` | Check whitespace-only (Java 11) |
| `stripLeading/Trailing()` | Unicode whitespace trim (Java 11) |

### StringBuilder/StringBuffer-Only Methods
| Method | Purpose |
|--------|---------|
| `append(...)` | Add to end |
| `insert(offset, ...)` | Insert at position |
| `delete(start, end)` | Remove range |
| `deleteCharAt(index)` | Remove one char |
| `reverse()` | Reverse content |
| `setCharAt(index, ch)` | Modify character |
| `capacity()` | Current buffer capacity |
| `ensureCapacity(min)` | Reserve capacity |
| `trimToSize()` | Shrink capacity |
| `setLength(n)` | Resize to n |

### Common to All Three
| Method | Purpose |
|--------|---------|
| `length()` | Get length |
| `charAt(index)` | Get char at index |
| `indexOf(str)` | Find first occurrence |
| `lastIndexOf(str)` | Find last occurrence |
| `substring(start[, end])` | Extract substring |
| `replace(...)` | Replace content |
| `toString()` | Convert to String |
| `equals(obj)` | Content comparison (String only overrides meaningfully) |
| `hashCode()` | Hash value |

---

*Reference: Java SE Documentation — java.lang.String, java.lang.StringBuilder, java.lang.StringBuffer*
