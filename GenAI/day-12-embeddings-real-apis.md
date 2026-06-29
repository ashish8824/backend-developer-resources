# Day 12 — Embeddings Deep Dive: Real APIs and Semantic Search

> Part of the 30-Day Generative AI Mastery Roadmap (for a Node.js Backend Developer)

---

## A Note on Today's Verification (Read This First)

Days 1-11 included Node.js code that I personally ran and verified in a sandboxed environment before writing it into each lesson. **Today is different, and I want to be upfront about why:** real embedding APIs (OpenAI, Voyage AI, etc.) require network access and an API key that aren't available in my sandbox — and I tried installing a local embedding model as a substitute, but it failed due to a blocked dependency (a native image-processing library required by the package, even though we'd only be embedding text). So, for today:

- **The semantic search pipeline (chunking, cosine ranking, top-K retrieval) is fully built and verified**, using realistic stand-in vectors in place of real API output — every piece of *logic* you'll actually write yourself is tested and correct.
- **The actual embedding API call code is accurate, standard, well-established API usage** that I'm confident is correct based on stable, well-documented API patterns — but I have not executed it myself today, so I'd encourage you to treat it as "should work, please verify against current docs" rather than "personally confirmed," the way everything else in this course has been.
- For Anthropic-specific product questions (e.g., "does Claude have its own embeddings endpoint?"), I'll flag clearly where you should double check current docs, since offerings can change after my knowledge cutoff.

This is exactly the kind of honest calibration I'd want you to practice yourself when teaching others: **say clearly what you've verified versus what you're confident about but haven't personally tested.**

---

## 1. WHAT — Plain Definitions

Day 4 used hand-crafted toy embeddings. Day 7 trained tiny, real embeddings from scratch on a toy corpus. Today connects this to **production embedding models** — the actual APIs you'd call in a real application — and the standard pattern built on top of them: **semantic search**.

| Term | Plain Definition |
|---|---|
| **Embedding model** | A (typically large, pretrained) neural network whose entire job is to convert text into a meaningful vector — nothing else. |
| **Embedding API** | A hosted service (OpenAI, Voyage AI, Cohere, etc.) that runs an embedding model for you and returns vectors over HTTP, so you don't have to run the model yourself. |
| **Semantic search** | Finding relevant documents by comparing the *meaning* of a query to the *meaning* of documents (via embeddings + cosine similarity), rather than matching exact keywords. |
| **Chunking** | Splitting a long document into smaller pieces before embedding, since embedding models work best on (and often have hard limits on) shorter spans of text. |
| **Vector store / vector database** | A system optimized for storing many embeddings and efficiently finding the most similar ones to a given query vector (full implementation coming Day 18). |

### Diagram: The Semantic Search Pipeline

```
   INDEXING (done once, ahead of time)              QUERYING (done per user request)
   ──────────────────────────────────               ─────────────────────────────────
   documents ──► chunk ──► embed each chunk          user query ──► embed the query
                              │                                          │
                              ▼                                          ▼
                      store [text, vector]                      compare query vector to
                      pairs somewhere                            EVERY stored vector via
                      (array, DB, vector store)                  cosine similarity (Day 4)
                                                                          │
                                                                          ▼
                                                                  sort by similarity,
                                                                  return top-K matches
```

This is the exact architecture behind every "ask questions about your documents" feature you've seen, and it's the foundation for RAG (Day 17-19).

---

## 2. WHY — Why Use a Hosted Embedding API Instead of Day 7's Approach?

### Why not just train your own embeddings, like Day 7?

Day 7's embeddings were trained on a tiny, 9-word toy corpus — they only "understand" the handful of words they saw. Production embedding models are trained on enormous, diverse corpora (often billions of sentences), giving them rich, general-purpose representations of meaning across nearly any topic, in multiple languages, without you needing to train anything yourself. Calling an API to get these is dramatically more practical than training a comparable model from scratch, which would require massive data and compute most teams don't have.

### Why semantic search instead of traditional keyword search?

Traditional keyword search (like a SQL `LIKE` query or basic full-text search) only finds documents containing the *exact words* you searched for. If a user searches "how do I train my dog?" but your document says "puppy obedience tips," keyword search might miss it entirely — no shared words. Semantic search instead compares *meaning*: since "dog," "puppy," "train," and "obedience" all land near each other in embedding space (just like Day 4's "king"/"queen" example), the query and the relevant document end up with high cosine similarity even though they share almost no exact words.

### Why chunking matters

Embedding models typically have a maximum input length, and even when they don't strictly enforce one, embedding an entire long document into a *single* vector tends to "average out" or dilute its meaning — a 50-page document covering five different topics can't be faithfully compressed into one vector that's still useful for finding any *specific* one of those topics. Splitting documents into smaller, semantically coherent chunks (a paragraph, a section) before embedding means each resulting vector represents a more focused, specific piece of meaning — which makes retrieval far more precise.

---

## 3. HOW — The Mechanism

### The indexing phase (done once, ahead of time)

```
1. Take your raw documents
2. Split each into chunks (e.g., by paragraph, or every N tokens with some overlap)
3. Send each chunk to an embedding API, get back a vector
4. Store (chunk_text, vector) pairs somewhere you can search later
```

### The query phase (done every time a user asks something)

```
1. Take the user's query text
2. Send it to the SAME embedding API/model used for indexing (this matters --
   mixing embeddings from different models produces meaningless comparisons,
   since different models place meaning in different, incompatible vector spaces)
3. Compute cosine similarity (Day 4's formula) between the query vector and
   EVERY stored document vector
4. Sort by similarity, descending
5. Return the top-K highest-scoring chunks
```

This is *exactly* the `findClosest` pattern you built and verified back on Day 4 — today just swaps "toy hand-crafted vectors" for "real API-generated vectors," and "5 words" for "a real document collection."

---

## 4. EXAMPLE — Concrete Verified Trace (Pipeline Logic) + Real API Reference

### Concrete trace: the semantic search pipeline, verified end to end

Using a small collection of 8 documents spanning 4 topics (pets, cooking, finance, weather), with realistic stand-in vectors placed along those 4 semantic axes:

| Query | Top Match Found | Similarity |
|---|---|---|
| "How do I train my dog?" | "The best way to train a puppy is with positive reinforcement." | 1.000 |
| "What's a good pasta recipe?" | "To make a good risotto, stir constantly and add stock slowly." | 0.999 |
| "How should I invest for retirement?" | "Diversifying your portfolio reduces overall investment risk." | 0.999 |
| "Will it rain tomorrow?" | "Expect heavy rain and thunderstorms across the region today." | 0.999 |

In every single case, **the correct-topic document was ranked first**, with similarity scores above 0.99, while documents from unrelated topics scored between 0.11 and 0.26 — a clear, decisive separation. This confirms the ranking and retrieval *logic* (the part you actually write as a developer) works correctly: feed it real embeddings from any provider, and this exact code will correctly surface the most semantically relevant content.

### Real API reference: OpenAI embeddings (widely used, stable, standard pattern)

```javascript
// Standard pattern for calling OpenAI's embeddings endpoint.
// NOTE: this code follows OpenAI's well-documented, stable API format,
// but has not been executed in today's verification process (see the
// note at the top of this lesson) -- double check against current docs
// before relying on it in production.

async function getEmbedding(text, apiKey) {
  const response = await fetch("https://api.openai.com/v1/embeddings", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": `Bearer ${apiKey}`,
    },
    body: JSON.stringify({
      model: "text-embedding-3-small",
      input: text,
    }),
  });

  if (!response.ok) {
    throw new Error(`Embedding API error: ${response.status} ${await response.text()}`);
  }

  const data = await response.json();
  return data.data[0].embedding; // an array of floats, e.g. 1536 numbers long
}
```

### Real API reference: Voyage AI embeddings

Anthropic's own API does not (as of my knowledge cutoff) offer a native embeddings endpoint directly — Anthropic has pointed developers toward partners for embeddings, with **Voyage AI** being a commonly recommended option. **I'd treat this as worth double-checking against current Anthropic documentation** (`https://docs.claude.com`), since product offerings can change.

```javascript
// Standard pattern for calling Voyage AI's embeddings endpoint.
// Same caveat as above: follows their documented API shape, not executed today.

async function getVoyageEmbedding(text, apiKey) {
  const response = await fetch("https://api.voyageai.com/v1/embeddings", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": `Bearer ${apiKey}`,
    },
    body: JSON.stringify({
      model: "voyage-3",
      input: [text],
    }),
  });

  if (!response.ok) {
    throw new Error(`Voyage API error: ${response.status} ${await response.text()}`);
  }

  const data = await response.json();
  return data.data[0].embedding;
}
```

**The key point regardless of provider:** whichever API you use, the vector it returns plugs directly into the exact same `cosineSimilarity` and `semanticSearch` functions verified below — the downstream pipeline logic doesn't care which company generated the embedding, only that the *same model* was used consistently for both indexing and querying.

---

## 5. IMPLEMENTATION — Node.js

### Part A — The Verified Semantic Search Pipeline

```javascript
// semantic-search-pipeline.js
// Verifies the FULL semantic search pipeline structure (the part that's
// identical whether you get embeddings from OpenAI, Voyage AI, or anywhere
// else): store documents + their embeddings, embed a query, rank by cosine
// similarity, return top matches.
//
// NOTE: the embeddings below are realistic STAND-IN vectors (not from a real
// API call -- this sandbox has no network access to embedding providers).
// They're constructed to mimic genuine semantic relationships, so the
// RANKING LOGIC itself is fully verified, even though these specific numbers
// aren't from an actual model. The real API code is shown separately and
// uses this exact same downstream pipeline unchanged.

function dotProduct(a, b) {
  return a.reduce((sum, val, i) => sum + val * b[i], 0);
}
function magnitude(v) {
  return Math.sqrt(v.reduce((sum, x) => sum + x * x, 0));
}
function cosineSimilarity(a, b) {
  return dotProduct(a, b) / (magnitude(a) * magnitude(b));
}

// Stand-in "embeddings" for a tiny document collection (normally returned
// by a real embeddings API call -- see real-embeddings-api.js for that code).
// Constructed along 4 semantic axes: [animals, cooking, finance, weather]
const documents = [
  { id: 1, text: "The best way to train a puppy is with positive reinforcement.", embedding: [0.9, 0.1, 0.0, 0.0] },
  { id: 2, text: "Cats often sleep over 12 hours a day.",                          embedding: [0.85, 0.15, 0.0, 0.05] },
  { id: 3, text: "To make a good risotto, stir constantly and add stock slowly.",  embedding: [0.05, 0.9, 0.05, 0.0] },
  { id: 4, text: "Searing meat before braising adds a lot of flavor.",             embedding: [0.1, 0.85, 0.0, 0.05] },
  { id: 5, text: "Diversifying your portfolio reduces overall investment risk.",   embedding: [0.0, 0.05, 0.9, 0.05] },
  { id: 6, text: "Compound interest can significantly grow retirement savings.",  embedding: [0.0, 0.0, 0.85, 0.1] },
  { id: 7, text: "Expect heavy rain and thunderstorms across the region today.",  embedding: [0.0, 0.0, 0.05, 0.9] },
  { id: 8, text: "A cold front will drop temperatures by 15 degrees overnight.",  embedding: [0.05, 0.0, 0.0, 0.85] },
];

function embedQuery(text) {
  // In a real system, this calls an embeddings API. Here, we use a stand-in
  // vector for each test query, placed deliberately along the same semantic
  // axes as the documents above, so the ranking logic can be verified.
  const queryEmbeddings = {
    "How do I train my dog?": [0.88, 0.08, 0.0, 0.02],
    "What's a good pasta recipe?": [0.08, 0.88, 0.02, 0.0],
    "How should I invest for retirement?": [0.0, 0.02, 0.88, 0.05],
    "Will it rain tomorrow?": [0.0, 0.0, 0.02, 0.92],
  };
  return queryEmbeddings[text];
}

function semanticSearch(queryText, topK = 3) {
  const queryEmbedding = embedQuery(queryText);

  const scored = documents.map(doc => ({
    ...doc,
    similarity: cosineSimilarity(queryEmbedding, doc.embedding),
  }));

  scored.sort((a, b) => b.similarity - a.similarity);
  return scored.slice(0, topK);
}

const testQueries = [
  "How do I train my dog?",
  "What's a good pasta recipe?",
  "How should I invest for retirement?",
  "Will it rain tomorrow?",
];

testQueries.forEach(query => {
  console.log(`\nQuery: "${query}"`);
  const results = semanticSearch(query, 3);
  results.forEach((r, rank) => {
    console.log(`  ${rank + 1}. [similarity=${r.similarity.toFixed(3)}] "${r.text}"`);
  });
});
```

**Run it:**
```bash
node semantic-search-pipeline.js
```

**Verified output:**
```
Query: "How do I train my dog?"
  1. [similarity=1.000] "The best way to train a puppy is with positive reinforcement."
  2. [similarity=0.996] "Cats often sleep over 12 hours a day."
  3. [similarity=0.207] "Searing meat before braising adds a lot of flavor."

Query: "What's a good pasta recipe?"
  1. [similarity=0.999] "To make a good risotto, stir constantly and add stock slowly."
  2. [similarity=0.998] "Searing meat before braising adds a lot of flavor."
  3. [similarity=0.262] "Cats often sleep over 12 hours a day."

Query: "How should I invest for retirement?"
  1. [similarity=0.999] "Diversifying your portfolio reduces overall investment risk."
  2. [similarity=0.998] "Compound interest can significantly grow retirement savings."
  3. [similarity=0.112] "Expect heavy rain and thunderstorms across the region today."

Query: "Will it rain tomorrow?"
  1. [similarity=0.999] "Expect heavy rain and thunderstorms across the region today."
  2. [similarity=0.998] "A cold front will drop temperatures by 15 degrees overnight."
  3. [similarity=0.138] "Compound interest can significantly grow retirement savings."
```

#### Line-by-line explanation

- **`dotProduct`, `magnitude`, `cosineSimilarity`** — the exact same functions from Day 4, completely unchanged. This is worth pausing on: the *math* you learned on Day 4 with hand-crafted 3-dimensional vectors is *identical* to what powers real production semantic search systems using 1536-dimensional (or larger) vectors from real APIs. Nothing new to learn here — just applied at scale.
- **`documents` array with `embedding` fields** — in a real application, you'd compute these once (the "indexing phase" from Section 3) by calling an embedding API for each document chunk, then store the results — in a simple array for a small project, or in a proper vector database (Day 18) for anything larger.
- **`embedQuery(text)`** — in this verification, a lookup table standing in for a real API call. In production code, this function's body would be replaced with an actual `await getEmbedding(text, apiKey)` call (shown in Section 4) — **the rest of the pipeline requires zero changes**, since `semanticSearch` only cares that it receives back an array of numbers, regardless of where they came from.
- **`semanticSearch`** — maps every document to a `{...doc, similarity}` object, sorts descending by similarity, and returns the top K. This is the entire "retrieval" half of RAG (Day 17-19) — you're looking at the actual core retrieval algorithm today, in advance.
- **The verified results** — every query correctly surfaces same-topic documents first, with a clean numerical gap (0.99+ vs 0.1-0.26) between relevant and irrelevant results, confirming the ranking logic is sound and ready to be plugged into a real embedding source.

---

### Part B — Real Embedding API Code (Documented Pattern, Not Executed Today)

```javascript
// real-embeddings-api.js
// Standard, well-documented API call patterns for getting REAL embeddings.
// IMPORTANT: per this lesson's verification note, this code follows stable,
// well-established API conventions but was NOT executed in today's sandbox
// (no network access to these providers). Verify against current docs
// before relying on it in production: https://platform.openai.com/docs/api-reference/embeddings

async function getEmbedding(text, apiKey) {
  const response = await fetch("https://api.openai.com/v1/embeddings", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": `Bearer ${apiKey}`,
    },
    body: JSON.stringify({
      model: "text-embedding-3-small",
      input: text,
    }),
  });

  if (!response.ok) {
    throw new Error(`Embedding API error: ${response.status} ${await response.text()}`);
  }

  const data = await response.json();
  return data.data[0].embedding; // an array of floats, e.g. 1536 numbers long
}

// Example usage (would need a real API key to actually run):
// const vector = await getEmbedding("How do I train my dog?", process.env.OPENAI_API_KEY);
// console.log(vector.length); // 1536 for text-embedding-3-small
// console.log(vector.slice(0, 5)); // first 5 numbers, e.g. [0.0123, -0.0456, ...]

module.exports = { getEmbedding };
```

#### Line-by-line explanation

- **`fetch(...)`** — a standard HTTP POST request; Node.js 18+ has `fetch` built in globally, no extra package needed for this specific call.
- **`model: "text-embedding-3-small"`** — specifies which embedding model to use. Different models produce different-sized vectors and have different quality/cost tradeoffs — this is a detail worth checking current docs for, since model offerings and names can change.
- **`data.data[0].embedding`** — the response shape for this API wraps the embedding inside a `data` array (supporting batch requests with multiple inputs at once) — `data[0]` gets the first (and here, only) result's embedding.
- **Once you have this real vector**, you plug it directly into Part A's `documents` array (replacing the stand-in `embedding` field) and `embedQuery` function (replacing the lookup table with a real `await getEmbedding(...)` call) — **no other code changes needed.**

---

## 6. TEACH-IT-BACK — Script You Can Say Out Loud

> "Production embedding models work exactly like the toy and trained-from-scratch embeddings from earlier in this course, just trained on vastly more data, giving them rich, general-purpose representations of meaning. You access them through a hosted API — send text, get back a vector, typically over a thousand numbers long. The standard application built on top of this is semantic search: instead of matching exact keywords, you embed both your documents and the user's query using the same model, then use cosine similarity — the exact formula from earlier in this course — to find which documents are closest in meaning to the query, even if they don't share any of the same words. Before embedding documents, you typically split them into smaller chunks first, because cramming an entire long document into one vector tends to dilute or average out its meaning, making it less useful for finding any one specific part of it. The full pipeline has two phases: an indexing phase, done once ahead of time, where you chunk your documents, embed each chunk, and store the text-vector pairs somewhere; and a query phase, done every time a user asks something, where you embed their query with the same model, compare it against every stored vector, sort by similarity, and return the top matches. This exact retrieval pattern — embed, compare, rank, return top results — is the foundation of RAG, which we'll build in full over the next several days."

---

## Key Terms Glossary (Day 12)

| Term | Meaning |
|---|---|
| **Embedding model** | A model dedicated to converting text into meaningful vectors |
| **Embedding API** | A hosted service that runs an embedding model and returns vectors over HTTP |
| **Semantic search** | Finding relevant content by comparing meaning (via embeddings), not exact keywords |
| **Chunking** | Splitting long documents into smaller pieces before embedding |
| **Indexing phase** | The one-time setup step: chunk, embed, and store document vectors |
| **Query phase** | The per-request step: embed the query and compare against stored vectors |

---

## Quick Self-Check (ask yourself before moving to Day 13)

1. Can I explain, without looking, why semantic search can find relevant results even with zero shared keywords?
2. Can I explain why chunking matters before embedding long documents?
3. Can I describe the two phases of a semantic search pipeline (indexing vs. querying) and what happens in each?
4. Can I explain why today's verification looked different from previous days, and why that distinction matters when you're learning or teaching from any source?

If yes to all four — you're ready for **Day 13: Decoding strategies** (temperature, top-k, top-p, greedy decoding — why the same prompt gives different answers).
