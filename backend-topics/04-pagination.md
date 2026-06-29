# Pagination

## What is Pagination?

Pagination is the technique of splitting a large dataset into smaller chunks (pages) and returning one chunk at a time instead of fetching everything at once.

**Without pagination:**

```
GET /products
-> returns 1,000,000 rows from DB
-> 500MB of JSON in response
-> DB overloaded
-> Client crashes or times out
```

**With pagination:**

```
GET /products?page=1&size=20
-> returns 20 rows
-> Fast DB query
-> Small response
-> Happy client
```

## Why do we need it?

### 1. Performance

Fetching millions of rows at once:
- Kills DB query time
- Wastes network bandwidth
- Slows or crashes the client

### 2. Memory Efficiency

The server doesn't need to load an entire dataset into memory just to return it.

### 3. Better User Experience

Users rarely need all records at once. Showing 20 results at a time is faster, more readable, and good UX.

### 4. Reduced Cost

Fewer rows fetched = fewer IOPS = lower cloud DB bill.

## Types of Pagination

### 1. Offset-Based Pagination (Page Number Pagination)

The most common and intuitive style. You tell the DB: "skip the first N rows, give me the next M rows."

**API:**

```
GET /products?page=3&size=10
```

**SQL:**

```sql
SELECT * FROM products
ORDER BY id
LIMIT 10 OFFSET 20;   -- page 3 of size 10: skip 20 rows, take 10
```

**Offset formula:**

```
OFFSET = (page - 1) * size
```

**Response:**

```json
{
  "data": [...],
  "page": 3,
  "size": 10,
  "totalElements": 1050,
  "totalPages": 105
}
```

**Pros:**
- Simple for clients — they can jump to any page directly
- Easy to show "Page 5 of 105" UI
- Familiar pattern for both developers and users

**Cons — the OFFSET problem:**

Large offsets are expensive. The DB doesn't skip rows for free — it scans *and discards* them.

```sql
SELECT * FROM products ORDER BY id LIMIT 10 OFFSET 1000000;
```

The DB reads 1,000,010 rows internally, throws away 1,000,000, and returns 10. Gets progressively slower as you go deeper into the dataset.

**Data consistency problem:**

If a new row is inserted while the user is paginating:

```
Page 1: rows 1-10 fetched
New row inserted at position 1
Page 2: rows 11-20 fetched — but the dataset shifted, so the user might see row 10 again (duplicated) or miss a row
```

Not a problem for slowly-changing data, but a real issue for live feeds.

### 2. Cursor-Based Pagination (Keyset Pagination)

Instead of saying "skip N rows," you say "give me rows *after* this specific value (the cursor)."

**API:**

```
GET /products?cursor=eyJpZCI6MTAwfQ==&size=10
```

(The cursor is usually a base64-encoded pointer to the last seen record — e.g. `{"id": 100}`.)

**SQL:**

```sql
SELECT * FROM products
WHERE id > 100           -- "after cursor"
ORDER BY id
LIMIT 10;
```

**Response:**

```json
{
  "data": [...],
  "nextCursor": "eyJpZCI6MTEwfQ==",
  "hasMore": true
}
```

The client uses `nextCursor` from the current response as the `cursor` parameter in the next request.

**Pros:**
- Constant-time performance — the `WHERE id > ?` clause + an index on `id` means the DB seeks directly to the right row regardless of how deep into the dataset you are
- No duplicate or skipped rows if data is inserted/deleted mid-session (stable cursor)
- The natural choice for infinite scroll (feeds, timelines)

**Cons:**
- Cannot jump to arbitrary pages — you can only go forward (and backward if you track the previous cursor too)
- No "Page X of Y" display possible — you don't know total count cheaply
- Client must manage cursor state
- Cursor must be based on a unique, indexed, monotonically-ordered column (or a composite of columns)

### 3. Seek Method (Composite Keyset)

An extension of cursor-based pagination for sorting by non-unique columns.

**Problem:** sorting by `price` (non-unique) — many products may share the same price, so `WHERE price > ?` is ambiguous and may skip or repeat rows.

**Fix:** use a composite cursor — sort by `(price, id)` together, where `id` breaks ties:

```sql
SELECT * FROM products
WHERE (price, id) > (49.99, 100)   -- "after this exact position"
ORDER BY price ASC, id ASC
LIMIT 10;
```

The pair `(price, id)` is always unique and stable. A composite index on `(price, id)` makes this fast.

### 4. Time-Based Pagination

A special case of cursor-based where the cursor is a timestamp. Common for feeds and activity logs.

```sql
SELECT * FROM posts
WHERE created_at < '2025-01-01T10:00:00'   -- "before this point in time"
ORDER BY created_at DESC
LIMIT 20;
```

**Watch out:** timestamps may not be unique (two rows can have the exact same millisecond). Use a composite of `(created_at, id)` as the cursor to avoid duplicates/skips.

## Comparison Table

| Feature | Offset-Based | Cursor-Based |
|---|---|---|
| Jumping to arbitrary page | Yes | No |
| Show "Page X of Y" | Yes | No (no cheap total count) |
| Performance at deep pages | Poor (OFFSET scans + discards) | Constant (index seek) |
| Stable under inserts/deletes | No (rows can shift) | Yes |
| Implementation complexity | Simple | Moderate |
| Best for | Admin tables, search results, static datasets | Infinite scroll, live feeds, large datasets |

## Total Count — The Hidden Expense

Offset-based pagination usually returns `totalElements` and `totalPages` so the client can render a pager control.

```sql
SELECT COUNT(*) FROM products WHERE category = 'electronics';
```

For large tables, this `COUNT(*)` can be expensive and slow, especially with filters. Common approaches:

- **Exact count** for small/medium tables — just run `COUNT(*)`.
- **Approximate count** for huge tables — many databases maintain rough row counts in system statistics (e.g. `pg_class.reltuples` in PostgreSQL, `information_schema` in MySQL). Cheap but possibly slightly stale.
- **Cache the count** — run the `COUNT(*)` once, cache in Redis with a short TTL, return the cached value. Accept slight staleness.
- **Skip the count entirely** — cursor-based pagination doesn't need a total count; the response just has `hasMore: true/false`. Simplest and most scalable.

## Cursor Design Best Practices

### Opaque cursors

Expose cursors as opaque, base64-encoded strings rather than raw values so the client has no dependency on the internal structure:

```
Raw:    {"id": 100, "created_at": "2025-01-01T10:00:00"}
Cursor: eyJpZCI6MTAwLCJjcmVhdGVkX2F0IjoiMjAyNS0wMS0wMVQxMDowMDowMCJ9
```

This lets you change the cursor's internal structure (e.g. switch from single-column to composite) without breaking clients.

### Signed / encrypted cursors

If the cursor encodes internal DB values (like raw IDs or prices), consider signing or encrypting it — preventing clients from crafting arbitrary cursors that probe your DB structure.

## Java Implementation

### Offset-Based

```java
import org.springframework.data.domain.*;

@GetMapping("/products")
public Page<Product> getProducts(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size) {

    Pageable pageable = PageRequest.of(page, size, Sort.by("id").ascending());
    return productRepository.findAll(pageable);
}
```

Spring Data returns a `Page<T>` object containing `content`, `totalElements`, `totalPages`, `number`, etc. — everything the client needs to render a pager.

### Cursor-Based

```java
@GetMapping("/products")
public CursorPage<Product> getProductsByCursor(
        @RequestParam(required = false) String cursor,
        @RequestParam(defaultValue = "20") int size) {

    Long afterId = decodeCursor(cursor); // base64 decode -> extract id

    List<Product> results;
    if (afterId == null) {
        results = productRepository.findTopNOrderById(size + 1);  // +1 to detect hasMore
    } else {
        results = productRepository.findByIdGreaterThanOrderById(afterId, size + 1);
    }

    boolean hasMore = results.size() > size;
    if (hasMore) results.remove(results.size() - 1); // trim the extra row

    String nextCursor = hasMore ? encodeCursor(results.get(results.size() - 1).getId()) : null;

    return new CursorPage<>(results, nextCursor, hasMore);
}

private Long decodeCursor(String cursor) {
    if (cursor == null) return null;
    String decoded = new String(Base64.getDecoder().decode(cursor));
    return Long.parseLong(decoded);
}

private String encodeCursor(Long id) {
    return Base64.getEncoder().encodeToString(id.toString().getBytes());
}
```

The `size + 1` trick: fetch one extra row — if it exists, there are more pages (`hasMore = true`); trim it before returning.

**Repository query:**

```java
@Query("SELECT p FROM Product p WHERE p.id > :afterId ORDER BY p.id ASC")
List<Product> findByIdGreaterThanOrderById(@Param("afterId") Long afterId, Pageable pageable);
```

## Pagination Response Formats

### Offset-based response

```json
{
  "data": [ ... ],
  "pagination": {
    "page": 3,
    "size": 20,
    "totalElements": 1050,
    "totalPages": 53,
    "hasNext": true,
    "hasPrevious": true
  }
}
```

### Cursor-based response

```json
{
  "data": [ ... ],
  "pagination": {
    "nextCursor": "eyJpZCI6MTIwfQ==",
    "previousCursor": "eyJpZCI6MTAxfQ==",
    "hasMore": true
  }
}
```

## Pagination in Real Systems

| System | Approach | Reason |
|---|---|---|
| Twitter / X feed | Cursor-based | Live, high-write feed; offset would duplicate/skip posts |
| Google Search | Offset-based | Static results per query; users jump between pages |
| GitHub API | Cursor-based (Link header) | Large repo data, stable traversal |
| Stripe API | Cursor-based (object ID) | Append-only financial ledger; consistency critical |
| E-commerce catalog | Offset-based | Products rarely change mid-session; "page 5 of 53" UI needed |

## Indexes and Pagination — Why They Matter Together

Pagination without the right index degrades to a full table scan.

**For offset-based:**

```sql
-- Needs an index on the ORDER BY column
CREATE INDEX idx_products_id ON products(id);
```

**For cursor-based:**

```sql
-- Must have an index on the cursor column(s) — the WHERE clause uses them
CREATE INDEX idx_products_id ON products(id);

-- Composite sort: index must match column order
CREATE INDEX idx_products_price_id ON products(price, id);
```

Without the index, `WHERE id > 100` becomes a full scan — cursor-based pagination loses its main performance advantage.

## Interview Questions

**Q1: What's the problem with OFFSET at large page numbers?**

The DB still reads and discards all the skipped rows internally. `LIMIT 20 OFFSET 1000000` forces the DB to read 1,000,020 rows and throw away 1,000,000 of them. The deeper the page, the slower the query — regardless of indexes.

**Q2: Why is cursor-based pagination faster?**

`WHERE id > 100 ORDER BY id LIMIT 20` with an index on `id` lets the DB seek directly to the right position via the index tree in O(log N) time, then scan forward 20 rows. No rows are read and discarded.

**Q3: When would you NOT use cursor-based pagination?**

When the client needs to jump to arbitrary page numbers (e.g. "go to page 47"), or when you need to display a total count/total pages — cursor-based gives you neither without extra queries.

**Q4: How do you paginate when sorting by a non-unique column like price?**

Use a composite cursor of `(price, id)` — the `id` breaks ties and makes every row uniquely addressable. The SQL becomes `WHERE (price, id) > (49.99, 100)` with a composite index on `(price, id)`.

**Q5: How would you design pagination for a social media feed (e.g. Twitter timeline)?**

Cursor-based with a timestamp + ID composite cursor. New posts shouldn't shift what the user sees mid-scroll (so no offset). The cursor encodes the `created_at` + `id` of the last seen post; the next page fetches posts older than that point. The response includes `nextCursor` and `hasMore`, not a total count.

**Q6: How do you make total count affordable at scale?**

Option 1: cache the `COUNT(*)` result in Redis with a short TTL. Option 2: use approximate counts from DB statistics (fast, slightly stale). Option 3: switch to cursor-based pagination and drop the total count requirement entirely. The right choice depends on how exact the count needs to be and how frequently it changes.
