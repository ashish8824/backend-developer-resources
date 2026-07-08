# Day 13: File Handling, JSON & REST APIs

> **Goal for today:** Learn how to read and write files, encode/decode JSON data using struct tags, build a simple REST API with `net/http`, and make outgoing HTTP requests. This is where Go really shows its strength for real-world backend development.

---

## 1. File Handling: Reading Files

Go's `os` package (introduced briefly on Day 11) provides the tools for working with files on disk.

### Reading an Entire File at Once (Simplest Method)

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    data, err := os.ReadFile("notes.txt")
    if err != nil {
        fmt.Println("Error reading file:", err)
        return
    }

    fmt.Println(string(data))
}
```

**Breakdown:**
- `os.ReadFile("notes.txt")` — reads the ENTIRE contents of the file in one go, returning the data as a `[]byte` (a slice of bytes — think of it as raw data, before we interpret it as text) and an `error` (the familiar `(result, error)` pattern from Day 8 — always check it!).
- `string(data)` — converts the `[]byte` into a readable `string` (a type conversion, like we learned on Day 1), so we can print/use it as normal text.

**When is this suitable?** `os.ReadFile` is great for small-to-medium files, where loading the WHOLE thing into memory at once is fine. For very large files, you'd want to read in smaller CHUNKS instead (using `bufio.Scanner` or similar), to avoid using excessive memory — but that's a more advanced topic beyond today's scope.

### Writing to a File

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    content := []byte("Hello, this is written by Go!")

    err := os.WriteFile("output.txt", content, 0644)
    if err != nil {
        fmt.Println("Error writing file:", err)
        return
    }

    fmt.Println("File written successfully!")
}
```

**Breakdown:**
- `os.WriteFile(filename, data, permissions)` — writes `data` (a `[]byte`) to the given file, CREATING it if it doesn't exist, or OVERWRITING it if it does.
- `0644` — this is a **file permission code**, written in a special number format (octal). Don't worry about memorizing the exact meaning right now — just know `0644` is an extremely common, standard permission setting meaning "the file's owner can read and write it, and everyone else can only read it." You'll see this exact value used very often in real Go code.

### Appending to a File

To ADD content to an existing file WITHOUT overwriting what's already there, you need a bit more control, using `os.OpenFile`:

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    file, err := os.OpenFile("output.txt", os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
    if err != nil {
        fmt.Println("Error opening file:", err)
        return
    }
    defer file.Close() // remember Day 8! guaranteed cleanup

    _, err = file.WriteString("\nAppending a new line!")
    if err != nil {
        fmt.Println("Error writing:", err)
        return
    }

    fmt.Println("Appended successfully!")
}
```

**Breakdown:**
- `os.O_APPEND|os.O_CREATE|os.O_WRONLY` — these are FLAGS (settings) combined together using the `|` (bitwise OR) operator, telling Go exactly how to open the file: `O_APPEND` (add to the end, don't overwrite), `O_CREATE` (create the file if it doesn't already exist), `O_WRONLY` (open it specifically for writing only).
- `defer file.Close()` — exactly the pattern from Day 8! This GUARANTEES the file gets closed properly once we're done, no matter how the function exits.
- `file.WriteString(...)` — writes a string directly to the open file.
- `_, err = ...` — `WriteString` actually returns TWO values: the number of bytes written, and an error. Here we use `_` (the blank identifier from Day 2/4) to ignore the byte count, since we only care about checking the error in this case.

---

## 2. JSON — Encoding and Decoding

**JSON** (JavaScript Object Notation) is a widely-used, human-readable text format for representing structured data — it's the standard way most web APIs send and receive data. Go's `encoding/json` package (part of the standard library) handles converting between Go structs and JSON text.

### From Go Struct to JSON (Marshaling / Encoding)

```go
package main

import (
    "encoding/json"
    "fmt"
)

type Person struct {
    Name string
    Age  int
    City string
}

func main() {
    p := Person{Name: "Rahul", Age: 25, City: "Bangalore"}

    jsonData, err := json.Marshal(p)
    if err != nil {
        fmt.Println("Error marshaling:", err)
        return
    }

    fmt.Println(string(jsonData))
}
```
Output:
```
{"Name":"Rahul","Age":25,"City":"Bangalore"}
```

**Breakdown:**
- The process of converting a Go value INTO JSON text is called **marshaling** (some other languages call this "serializing").
- `json.Marshal(p)` — takes our struct `p` and converts it into JSON, returning `([]byte, error)` — the familiar pattern again.
- Notice: only EXPORTED fields (capitalized, from Day 11!) get included in the JSON output. If `Name` were lowercase (`name`), it would be silently SKIPPED by `json.Marshal`, since unexported fields aren't accessible outside the package — this trips up MANY beginners, so remember it clearly: **JSON encoding only works with exported struct fields.**

### Struct Tags — Customizing JSON Field Names

Notice the output above has capitalized JSON keys (`"Name"`, `"Age"`), matching our Go field names exactly. But most real-world APIs use lowercase, or specific naming conventions (like `snake_case`). We control this using **struct tags**.

```go
type Person struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
    City string `json:"city,omitempty"`
}
```

**Breakdown:**
- The text inside the backticks after each field — `` `json:"name"` `` — is called a **struct tag**. It's metadata attached to a struct field, readable by other packages (like `encoding/json`) to customize their behavior.
- `json:"name"` — tells the JSON encoder to use `"name"` (lowercase) as the key in the JSON output, instead of the Go field name `Name`.
- `json:"city,omitempty"` — the `omitempty` option means: "if this field is EMPTY (its zero value, like `""` for a string or `0` for an int), leave it OUT of the JSON output entirely" — useful for optional fields.

```go
p := Person{Name: "Rahul", Age: 25, City: "Bangalore"}
jsonData, _ := json.Marshal(p)
fmt.Println(string(jsonData))
// Output: {"name":"Rahul","age":25,"city":"Bangalore"}
```

### From JSON to Go Struct (Unmarshaling / Decoding)

```go
package main

import (
    "encoding/json"
    "fmt"
)

type Person struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
    City string `json:"city"`
}

func main() {
    jsonInput := `{"name":"Priya","age":28,"city":"Mumbai"}`

    var p Person
    err := json.Unmarshal([]byte(jsonInput), &p)
    if err != nil {
        fmt.Println("Error unmarshaling:", err)
        return
    }

    fmt.Println(p.Name, p.Age, p.City)
}
```
Output:
```
Priya 28 Mumbai
```

**Breakdown:**
- The reverse process — converting JSON text INTO a Go value — is called **unmarshaling** (or "deserializing").
- `json.Unmarshal([]byte(jsonInput), &p)` — takes the JSON data (converted to `[]byte`) and a POINTER to the struct (`&p`, from Day 6!) where the decoded data should be stored. It NEEDS a pointer because `Unmarshal` must MODIFY the original `p` variable directly — passing `p` by value would only let it modify a useless copy (exactly like the value receiver lessons from Day 5/6).
- Go automatically MATCHES JSON keys (`"name"`, `"age"`, `"city"`) to the corresponding struct fields, based on the `json:"..."` struct tags we defined.

### Visual: Marshal vs Unmarshal

```
   Go Struct                              JSON Text
   ┌─────────────┐   json.Marshal()    ┌───────────────────────┐
   │ Name: "Rahul" │  ───────────────>  │ {"name":"Rahul",...}    │
   │ Age: 25        │                    └───────────────────────┘
   └─────────────┘   <───────────────
                       json.Unmarshal()
```

---

## 3. Building a Simple REST API with `net/http`

A **REST API** is a way for programs to communicate over HTTP (the protocol web browsers/servers use), typically exchanging JSON data. Go's `net/http` package (built into the standard library — no external framework needed for basics!) makes building one straightforward.

### A Basic HTTP Server

```go
package main

import (
    "fmt"
    "net/http"
)

func homeHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "Welcome to my Go API!")
}

func main() {
    http.HandleFunc("/", homeHandler)

    fmt.Println("Server starting on port 8080...")
    http.ListenAndServe(":8080", nil)
}
```

**Breakdown:**
- `func homeHandler(w http.ResponseWriter, r *http.Request)` — this is a **handler function** — the code that runs whenever someone visits a particular URL on our server. It takes two special parameters:
  - `w http.ResponseWriter` — used to WRITE the response back to whoever made the request (like a browser).
  - `r *http.Request` — contains information ABOUT the incoming request (what URL was requested, what data was sent, headers, etc.).
- `fmt.Fprintln(w, ...)` — similar to `fmt.Println`, but writes the text to `w` (the response writer) instead of the console — this sends the text back to whoever made the request.
- `http.HandleFunc("/", homeHandler)` — registers `homeHandler` to run whenever someone visits the root URL path (`"/"`).
- `http.ListenAndServe(":8080", nil)` — starts the actual web server, listening for incoming requests on port `8080`. This function BLOCKS (keeps running) — the server keeps running until you stop the program.

**Testing it:** Run `go run main.go`, then visit `http://localhost:8080` in a browser, or use a tool like `curl http://localhost:8080` — you'd see "Welcome to my Go API!"

### Building a JSON API Endpoint

```go
package main

import (
    "encoding/json"
    "net/http"
)

type Person struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}

func personHandler(w http.ResponseWriter, r *http.Request) {
    p := Person{Name: "Rahul", Age: 25}

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(p)
}

func main() {
    http.HandleFunc("/person", personHandler)
    http.ListenAndServe(":8080", nil)
}
```

**Breakdown:**
- `w.Header().Set("Content-Type", "application/json")` — tells whoever is receiving this response, "the data I'm sending you is JSON" — a standard, expected practice for JSON APIs, so clients know how to interpret the response correctly.
- `json.NewEncoder(w).Encode(p)` — a convenient shortcut that combines marshaling AND writing to the response in one step; it directly encodes `p` as JSON and streams it to `w`.

Visiting `http://localhost:8080/person` would show:
```json
{"name":"Rahul","age":25}
```

### Handling Different HTTP Methods (GET, POST, etc.)

Real APIs need to handle different types of requests — GET (retrieving data), POST (creating data), etc. Here's an example handling a POST request with a JSON body:

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
)

type Person struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}

func createPersonHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodPost {
        http.Error(w, "Only POST method is allowed", http.StatusMethodNotAllowed)
        return
    }

    var p Person
    err := json.NewDecoder(r.Body).Decode(&p)
    if err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }

    fmt.Println("Received:", p)

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(p)
}

func main() {
    http.HandleFunc("/person", createPersonHandler)
    http.ListenAndServe(":8080", nil)
}
```

**Breakdown:**
- `r.Method != http.MethodPost` — checks WHICH HTTP method was used for this request; if it's not `POST`, we reject it using `http.Error`, which sends back an error response with the given message and status code.
- `http.StatusMethodNotAllowed` (405), `http.StatusBadRequest` (400), `http.StatusCreated` (201) — these are named constants representing standard HTTP status codes, more readable than remembering raw numbers.
- `json.NewDecoder(r.Body).Decode(&p)` — reads the JSON data sent in the request's BODY (`r.Body`), and decodes it directly into our `p` struct (again, needing a pointer `&p` so it can actually modify it).
- `w.WriteHeader(http.StatusCreated)` — explicitly sets the response's status code to 201 ("Created"), a standard convention for successful creation operations.

**Testing this with `curl`:**
```bash
curl -X POST http://localhost:8080/person -d '{"name":"Amit","age":30}'
```

---

## 4. Making Outgoing HTTP Requests (Being a Client, Not Just a Server)

Sometimes your Go program needs to CALL another API, rather than just serving one. `net/http` handles this too.

```go
package main

import (
    "encoding/json"
    "fmt"
    "io"
    "net/http"
)

type Post struct {
    UserID int    `json:"userId"`
    ID     int    `json:"id"`
    Title  string `json:"title"`
}

func main() {
    resp, err := http.Get("https://jsonplaceholder.typicode.com/posts/1")
    if err != nil {
        fmt.Println("Error making request:", err)
        return
    }
    defer resp.Body.Close() // Day 8 pattern again — guaranteed cleanup

    body, err := io.ReadAll(resp.Body)
    if err != nil {
        fmt.Println("Error reading response:", err)
        return
    }

    var post Post
    err = json.Unmarshal(body, &post)
    if err != nil {
        fmt.Println("Error unmarshaling:", err)
        return
    }

    fmt.Println("Title:", post.Title)
}
```

**Breakdown:**
- `http.Get(url)` — makes a simple GET request to the given URL, returning `(*http.Response, error)`.
- `defer resp.Body.Close()` — the response body is a resource that must be closed when we're done reading it, using the SAME `defer` cleanup pattern from Day 8.
- `io.ReadAll(resp.Body)` — reads the entire response body into a `[]byte`.
- We then `json.Unmarshal` it into our `Post` struct, exactly as we learned earlier today.

---

## 5. Common Beginner Mistakes

1. **Forgetting that only EXPORTED struct fields get included in JSON** — a lowercase field will be silently skipped during `Marshal`/`Unmarshal`, with no error to warn you.
2. **Passing a struct by VALUE instead of a pointer to `json.Unmarshal`** — it needs `&p` to actually populate your struct; passing `p` directly would fail or do nothing useful.
3. **Forgetting to check errors from file operations, JSON operations, or HTTP calls** — all of these can genuinely fail (missing file, malformed JSON, network issues), and Go's `(result, error)` pattern relies on YOU checking it, every time.
4. **Forgetting to close files and HTTP response bodies** — always use `defer file.Close()` / `defer resp.Body.Close()` to avoid resource leaks.
5. **Not setting `Content-Type: application/json`** when building a JSON API — many clients rely on this header to correctly interpret the response.
6. **Confusing Marshal (Go → JSON) and Unmarshal (JSON → Go)** — a common terminology mix-up; remember "Un-marshal" UNDOES marshaling, converting back FROM JSON.

---

## 6. Day 13 Practice Exercise

Write a program that:
1. Defines a struct `Product` with fields `Name`, `Price` (float64), and `InStock` (bool), using appropriate JSON struct tags (lowercase field names).
2. Writes a function to save a `Product` as JSON to a file called `product.json`.
3. Writes a function to read `product.json` back and print the loaded `Product`.
4. BONUS: Builds a minimal HTTP server with a `/product` endpoint that returns the `Product` as JSON.

<details>
<summary>Click to see solution</summary>

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "os"
)

type Product struct {
    Name    string  `json:"name"`
    Price   float64 `json:"price"`
    InStock bool    `json:"in_stock"`
}

func saveProduct(p Product, filename string) error {
    data, err := json.Marshal(p)
    if err != nil {
        return err
    }
    return os.WriteFile(filename, data, 0644)
}

func loadProduct(filename string) (Product, error) {
    var p Product
    data, err := os.ReadFile(filename)
    if err != nil {
        return p, err
    }
    err = json.Unmarshal(data, &p)
    return p, err
}

func productHandler(w http.ResponseWriter, r *http.Request) {
    p, err := loadProduct("product.json")
    if err != nil {
        http.Error(w, "Could not load product", http.StatusInternalServerError)
        return
    }
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(p)
}

func main() {
    product := Product{Name: "Laptop", Price: 55999.99, InStock: true}

    if err := saveProduct(product, "product.json"); err != nil {
        fmt.Println("Error saving:", err)
        return
    }

    loaded, err := loadProduct("product.json")
    if err != nil {
        fmt.Println("Error loading:", err)
        return
    }
    fmt.Println("Loaded product:", loaded)

    http.HandleFunc("/product", productHandler)
    fmt.Println("Server running on port 8080...")
    http.ListenAndServe(":8080", nil)
}
```
</details>

---

## 7. Day 13 Interview Questions (Quick Prep)

1. **Q: Why must a struct field be exported (capitalized) to appear in JSON output?**
   A: Because `encoding/json` (like any other package) can only access exported fields, following Go's package-level visibility rules from Day 11.

2. **Q: What is a struct tag, and what's it used for with JSON?**
   A: Metadata attached to a struct field (e.g., `` `json:"name"` ``), used by packages like `encoding/json` to customize behavior — such as renaming the JSON key or omitting empty fields with `omitempty`.

3. **Q: What's the difference between Marshal and Unmarshal?**
   A: `Marshal` converts a Go value INTO JSON (encoding); `Unmarshal` converts JSON text BACK INTO a Go value (decoding). Unmarshal requires a pointer to the destination.

4. **Q: What package does Go provide for building HTTP servers without any external framework?**
   A: `net/http`, part of the standard library.

5. **Q: How do you read the JSON body of an incoming HTTP request in a handler?**
   A: Using `json.NewDecoder(r.Body).Decode(&target)`.

6. **Q: Why is it important to `defer resp.Body.Close()` after making an HTTP request?**
   A: To ensure the response body's underlying resources are properly released, avoiding resource leaks — the same cleanup discipline as file handling.

---

## Summary of Day 13

- `os.ReadFile`/`os.WriteFile` handle simple file reading/writing; `os.OpenFile` with flags allows appending; always `defer file.Close()`.
- `encoding/json`'s `Marshal`/`Unmarshal` convert between Go structs and JSON text; only EXPORTED fields participate.
- Struct tags (`` `json:"name,omitempty"` ``) customize JSON field names and behavior.
- `net/http` (standard library, no external framework needed) builds HTTP servers via `http.HandleFunc` and `http.ListenAndServe`, and makes outgoing requests via `http.Get`/`http.Post`.
- Handler functions receive `http.ResponseWriter` (to write responses) and `*http.Request` (to read incoming request data).
- `json.NewEncoder(w).Encode(...)` and `json.NewDecoder(r.Body).Decode(...)` are convenient shortcuts for JSON API request/response handling.

**Tomorrow (Day 14):** We'll cover Advanced Topics — Generics (Go 1.18+), the `context` package for timeouts/cancellation, memory management and garbage collection basics, and common Go idioms/best practices.
