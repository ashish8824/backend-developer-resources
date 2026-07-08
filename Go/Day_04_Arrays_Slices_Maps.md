# Day 4: Arrays, Slices, and Maps

> **Goal for today:** Understand Go's core data structures for storing collections of data — Arrays, Slices (and their internals: length vs capacity, which is a heavily asked interview topic), and Maps. This is one of the most important days in the whole course.

---

## 1. Why Do We Need Collections?

So far, we've stored single values in variables: `age := 25`, `name := "Rahul"`. But real programs need to store **groups of related data** — like a list of student names, or a list of product prices.

**Analogy:** A single variable is like one labeled box. A collection is like a **shelf of numbered boxes**, or a **filing cabinet with labeled folders** — a structured way to hold many related items together.

Go gives us three main tools for this: **Arrays**, **Slices**, and **Maps**. Let's understand each, starting with Arrays because Slices are built on top of them.

---

## 2. Arrays

An **array** is a fixed-size collection of elements, all of the SAME data type, stored in a contiguous (side-by-side) block of memory.

### Declaring an Array

```go
package main

import "fmt"

func main() {
    var numbers [5]int // an array of exactly 5 integers
    numbers[0] = 10
    numbers[1] = 20
    numbers[2] = 30

    fmt.Println(numbers) // Output: [10 20 30 0 0]
}
```

**Breakdown:**
- `[5]int` — this means "an array that holds EXACTLY 5 elements, each of type `int`." The size `5` is PART of the array's type — this is a crucial point we'll return to.
- `numbers[0] = 10` — arrays are **zero-indexed**, meaning the first element is at position `0`, not `1`. So `numbers[0]` is the first box, `numbers[1]` is the second, and so on.
- Since we only assigned values to positions 0, 1, and 2, positions 3 and 4 keep their **default value** — for `int`, that default is `0`.

### Visual: Memory Layout of an Array

```
Index:    0    1    2    3    4
        ┌────┬────┬────┬────┬────┐
numbers │ 10 │ 20 │ 30 │  0 │  0 │
        └────┴────┴────┴────┴────┘
```
All 5 boxes sit right next to each other in memory — that's what "contiguous" means, and it's why accessing any element by index is extremely fast.

### Declaring with values directly (array literal)

```go
fruits := [3]string{"apple", "banana", "cherry"}
fmt.Println(fruits[1]) // Output: banana
```

You can also let Go count the size for you using `...`:
```go
fruits := [...]string{"apple", "banana", "cherry"}
// Go automatically figures out this array has size 3
```

### The Big Limitation of Arrays

**The size of an array is FIXED and is part of its type.** This means `[5]int` and `[10]int` are considered completely DIFFERENT types in Go — you cannot resize an array, and you cannot pass a `[5]int` to a function that expects a `[10]int`.

```go
var a [3]int
var b [5]int
// a and b are NOT the same type - you cannot assign one to the other
```

This rigidity is exactly why Go gives us something more flexible: **Slices**. In real-world Go code, you will use **slices far more often than arrays**. Arrays exist mostly as the foundation slices are built on.

---

## 3. Slices (The Tool You'll Actually Use Daily)

A **slice** is a flexible, resizable view into an underlying array. Think of it as a "smart, growable list."

### Declaring a Slice

```go
package main

import "fmt"

func main() {
    fruits := []string{"apple", "banana", "cherry"}
    fmt.Println(fruits) // Output: [apple banana cherry]
}
```

**Spot the difference from an array:** Notice there's **no number** inside the square brackets — `[]string` instead of `[3]string`. That missing number is exactly what makes this a slice instead of an array. This is a very common "gotcha" question in interviews: *"What's the syntax difference between declaring an array and a slice?"* Answer: the presence (array) or absence (slice) of a size inside `[ ]`.

### Creating a Slice with `make`

```go
numbers := make([]int, 5) // creates a slice of 5 integers, all initialized to 0
fmt.Println(numbers)      // Output: [0 0 0 0 0]
```
`make` is a built-in Go function used to create slices (and maps, and channels — we'll see those later). Here, `make([]int, 5)` says "create an int slice with a length of 5."

### Adding elements with `append`

Since slices are resizable, you use the built-in `append` function to add new elements:

```go
numbers := []int{1, 2, 3}
numbers = append(numbers, 4)
numbers = append(numbers, 5, 6) // you can append multiple values at once
fmt.Println(numbers) // Output: [1 2 3 4 5 6]
```

**Very important:** `append` does NOT modify the slice "in place" reliably — it RETURNS a new slice, which is why we always write `numbers = append(numbers, ...)`, reassigning the result back into `numbers`. Forgetting to reassign is a classic beginner bug.

### Accessing and Modifying Elements

```go
fruits := []string{"apple", "banana", "cherry"}
fmt.Println(fruits[0])  // apple
fruits[1] = "mango"
fmt.Println(fruits)     // [apple mango cherry]
```
Just like arrays, slices are zero-indexed.

---

## 4. Slice Internals: Length vs Capacity (CRITICAL Interview Topic)

This is one of THE most commonly asked Go interview topics, so let's build a very clear mental model.

A slice is actually a small structure made of THREE parts internally:
1. A **pointer** to an underlying array in memory (where the actual data lives).
2. A **length** — how many elements the slice currently "sees" / is using.
3. A **capacity** — how many elements the underlying array COULD hold, starting from that slice's starting point, before it needs to grow.

```
Slice header (what a slice actually is under the hood):
┌───────────────────────────┐
│  Pointer → [underlying array]   │
│  Length   = 3                    │
│  Capacity = 5                    │
└───────────────────────────┘
```

### Checking length and capacity

```go
package main

import "fmt"

func main() {
    numbers := make([]int, 3, 5) // length 3, capacity 5
    fmt.Println("Length:", len(numbers))
    fmt.Println("Capacity:", cap(numbers))
    fmt.Println(numbers)
}
```
Output:
```
Length: 3
Capacity: 5
[0 0 0]
```
- `len()` — built-in function that gives the current number of elements the slice is using.
- `cap()` — built-in function that gives the total space available in the underlying array before Go needs to allocate a new, bigger array.

### Visual Explanation

```
Underlying array (capacity 5):  [ 0 ][ 0 ][ 0 ][ _ ][ _ ]
                                  └──────┬──────┘
                             slice currently uses
                             the first 3 (length = 3)
                             but has room to grow up to 5 (capacity = 5)
```

### What happens when you `append` beyond capacity?

```go
numbers := make([]int, 3, 3) // length 3, capacity 3 — FULL, no room to grow
fmt.Println(len(numbers), cap(numbers)) // 3 3

numbers = append(numbers, 100) // this exceeds capacity!
fmt.Println(len(numbers), cap(numbers)) // 4 6 (capacity often DOUBLES)
```

**What's happening internally:** When you `append` and there's no more room in the current underlying array (capacity is full), Go automatically:
1. Creates a brand NEW, bigger underlying array (commonly double the old capacity).
2. Copies all existing elements over to this new array.
3. Adds the new element.
4. Updates the slice to point to this new array.

This is why capacity often jumps in unpredictable-looking ways (like 3 → 6, or sometimes different depending on Go version) — Go's internal growth strategy aims to balance memory use and performance.

**Why does this matter practically?** If you know roughly how many elements you'll need in advance, pre-allocating capacity with `make([]int, 0, expectedSize)` avoids repeated copying/reallocation, making your program faster. This is a common **performance optimization** question in interviews.

### Slicing a Slice (creating a sub-slice)

```go
numbers := []int{10, 20, 30, 40, 50}
sub := numbers[1:3] // elements from index 1 up to (but NOT including) index 3
fmt.Println(sub)    // Output: [20 30]
```

**Syntax:** `slice[startIndex:endIndex]` — includes `startIndex`, excludes `endIndex`.

**A subtle but important trap (frequently tested in interviews):** A sub-slice SHARES the same underlying array as the original slice! Modifying the sub-slice can affect the original slice too.

```go
numbers := []int{10, 20, 30, 40, 50}
sub := numbers[1:3]
sub[0] = 999
fmt.Println(numbers) // Output: [10 999 30 40 50]  <- original changed too!
```

**Why?** Because `sub` doesn't copy the data — it just points to the SAME underlying array as `numbers`, starting from a different position. Changing an element through `sub` changes the shared underlying data, so it's visible through `numbers` too.

### Copying a slice properly (to avoid the shared-array trap)

If you genuinely want an independent copy, use the built-in `copy` function:
```go
original := []int{1, 2, 3}
duplicate := make([]int, len(original))
copy(duplicate, original)

duplicate[0] = 999
fmt.Println(original)  // [1 2 3]   <- unaffected
fmt.Println(duplicate) // [999 2 3]
```

### Array vs Slice — Quick Comparison Table

| Feature | Array | Slice |
|---|---|---|
| Size | Fixed, part of the type (`[5]int`) | Dynamic, can grow (`[]int`) |
| Declared with | `[5]int{...}` | `[]int{...}` or `make([]int, len)` |
| Common usage | Rare in everyday Go code | Used constantly |
| Passed to functions | Copies the ENTIRE array (can be slow for large arrays) | Passes a small reference (pointer+len+cap), much cheaper |

---

## 5. Maps

A **map** stores data as **key-value pairs** — similar to a real dictionary, where you look up a "word" (key) to find its "meaning" (value). In other languages, this might be called a "hash map," "dictionary," or "object."

### Declaring and Using a Map

```go
package main

import "fmt"

func main() {
    ages := map[string]int{
        "Rahul": 25,
        "Priya": 30,
    }

    fmt.Println(ages["Rahul"]) // Output: 25
}
```

**Breakdown:**
- `map[string]int` — this declares a map where every **key** is a `string`, and every **value** is an `int`.
- `ages["Rahul"]` — looks up the value associated with the key `"Rahul"`.

### Creating an empty map with `make`

```go
scores := make(map[string]int)
scores["Math"] = 90
scores["Science"] = 85
fmt.Println(scores) // map[Math:90 Science:85]
```

### Checking if a Key Exists (Very Important Pattern)

This is a genuinely important, commonly-tested Go pattern. If you look up a key that doesn't exist in a map, Go does NOT throw an error — it just quietly gives you the **zero value** for that type (e.g., `0` for `int`), which can be misleading.

```go
scores := map[string]int{"Math": 90}

value := scores["English"]
fmt.Println(value) // Output: 0 (but English was never actually added!)
```

This is dangerous because you can't tell the difference between "the key exists with a value of 0" and "the key doesn't exist at all" just by looking at this.

**The correct way to check existence — the "comma ok" idiom:**

```go
value, exists := scores["English"]
if exists {
    fmt.Println("English score:", value)
} else {
    fmt.Println("English score not found")
}
```

**Breakdown:** When you access a map with TWO variables on the left side (`value, exists :=`), Go gives you:
1. `value` — the value found (or the zero value if not found)
2. `exists` — a `bool`: `true` if the key was actually present, `false` if not

This "comma ok" pattern shows up again later (Day 7 with type assertions, Day 10 with channels) — it's a recurring idiom in Go, so get comfortable with it now.

### Deleting a Key

```go
delete(scores, "Math")
fmt.Println(scores) // Math is now removed
```
`delete` is a built-in function: `delete(mapName, key)`.

### Looping Through a Map

```go
scores := map[string]int{"Math": 90, "Science": 85}

for subject, score := range scores {
    fmt.Println(subject, "-", score)
}
```

**Important interview point:** Map iteration order in Go is **NOT guaranteed** — every time you run this loop, the order of key-value pairs might come out differently. If you need a specific order, you must sort the keys yourself (commonly by extracting keys into a slice and sorting that slice, using the `sort` package).

### Visual: Map vs Slice

```
Slice (ordered, indexed by position):
   Index:  0        1        2
         [ "apple", "banana", "cherry" ]

Map (unordered, indexed by key):
   Key:     "Rahul"     "Priya"
   Value:      25          30
   (no guaranteed order when looping)
```

---

## 6. Common Beginner Mistakes

1. **Confusing array and slice syntax** — `[5]int` (array, fixed size) vs `[]int` (slice, dynamic).
2. **Forgetting to reassign after `append`** — `append(numbers, 4)` alone does nothing useful; you must write `numbers = append(numbers, 4)`.
3. **Assuming a sub-slice is an independent copy** — it shares the underlying array; modifying one can affect the other. Use `copy()` for a true independent copy.
4. **Accessing a non-existent map key without checking** — always use the "comma ok" pattern (`value, exists := myMap[key]`) when you need to know if a key truly exists.
5. **Expecting maps to maintain insertion order** — they don't; order is randomized on each run.
6. **Trying to compare slices directly with `==`** — Go does NOT allow this (`if slice1 == slice2` won't compile), because slices can contain other slices and comparing them isn't straightforward. You must compare element-by-element manually, or use `reflect.DeepEqual` (a standard library function for comparing complex values, discussed more later). Note: this differs for arrays — arrays actually CAN be compared directly with `==` since their size is fixed and known.

---

## 7. Day 4 Practice Exercise

Write a program that:
1. Creates a slice of your 5 favorite foods.
2. Appends 2 more foods to it.
3. Prints the length and capacity of the slice at each stage.
4. Creates a map storing 3 friends' names as keys and their ages as values.
5. Safely checks (using "comma ok") whether a friend named "Zara" exists in the map, and prints an appropriate message either way.

<details>
<summary>Click to see solution</summary>

```go
package main

import "fmt"

func main() {
    foods := []string{"Pizza", "Biryani", "Pasta", "Sushi", "Tacos"}
    fmt.Println("Initial:", foods, "Len:", len(foods), "Cap:", cap(foods))

    foods = append(foods, "Burger")
    fmt.Println("After 1 append:", foods, "Len:", len(foods), "Cap:", cap(foods))

    foods = append(foods, "Noodles")
    fmt.Println("After 2 appends:", foods, "Len:", len(foods), "Cap:", cap(foods))

    friendsAges := map[string]int{
        "Aman":  28,
        "Sneha": 26,
        "Karan": 30,
    }

    age, exists := friendsAges["Zara"]
    if exists {
        fmt.Println("Zara's age is", age)
    } else {
        fmt.Println("Zara is not in the friends list")
    }
}
```
</details>

---

## 8. Day 4 Interview Questions (Quick Prep)

1. **Q: What's the difference between an array and a slice in Go?**
   A: An array has a fixed size that's part of its type (`[5]int`); a slice is a flexible, resizable view over an underlying array (`[]int`), and is used far more often in practice.

2. **Q: Explain length vs capacity of a slice.**
   A: Length is the number of elements currently in use in the slice. Capacity is the total number of elements the underlying array can hold before Go needs to allocate a new, larger array.

3. **Q: What happens when you `append` to a slice that's at full capacity?**
   A: Go allocates a new, larger underlying array (commonly doubling capacity), copies the existing elements over, adds the new element, and the slice now points to this new array.

4. **Q: If you take a sub-slice and modify it, does it affect the original slice? Why?**
   A: Yes, because a sub-slice shares the same underlying array as the original — it doesn't create a copy. Use the `copy()` function for an independent copy.

5. **Q: How do you check if a key exists in a map in Go?**
   A: Using the "comma ok" idiom: `value, exists := myMap[key]`. If `exists` is `false`, the key wasn't present.

6. **Q: Is map iteration order guaranteed in Go?**
   A: No — Go intentionally randomizes map iteration order each time you loop through it.

7. **Q: Can you compare two slices directly with `==` in Go?**
   A: No, slices cannot be compared with `==` (except against `nil`). Arrays CAN be compared with `==` since their size is fixed.

---

## Summary of Day 4

- Arrays have a FIXED size that's part of their type; rarely used directly in everyday Go code.
- Slices are dynamic, resizable, and built on top of arrays — the tool you'll use constantly.
- A slice internally holds a pointer to an underlying array, a length, and a capacity.
- `append` may trigger reallocation of a new underlying array when capacity is exceeded.
- Sub-slices share the underlying array with the original — modifying one can affect the other; use `copy()` for independence.
- Maps store key-value pairs; use the "comma ok" pattern (`value, exists := m[key]`) to safely check for key existence.
- Map iteration order is never guaranteed.

**Tomorrow (Day 5):** We'll learn Structs and Methods — how Go groups related data together (like a blueprint for custom data types), how to attach behavior (methods) to those structs, the crucial difference between value receivers and pointer receivers, and struct embedding (Go's alternative to traditional inheritance).
