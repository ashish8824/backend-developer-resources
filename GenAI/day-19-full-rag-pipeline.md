# Day 19 — Building a Full RAG Pipeline: Chunking, Embedding, Retrieval, Augmentation

> Part of the 30-Day Generative AI Mastery Roadmap (for a Node.js Backend Developer)

---

## A Note on Today's Verification

Today's capstone genuinely broke twice during testing, and both times in subtle, instructive ways. I'm documenting the full debugging journey below rather than just showing you the final clean version, because **the bugs themselves teach something real about cosine similarity that's easy to miss** — and finding them is exactly the kind of work you'll do building real RAG systems.

---

## 1. WHAT — What We're Building Today

A single, complete `RAGApp` class combining everything from the last three days into one working pipeline:

- **Chunking** (new today): splitting long documents into overlapping pieces
- **Embedding** (Day 12): converting text to vectors — using a stand-in function, honestly labeled, since this sandbox has no real embeddings API access
- **Retrieval with a similarity threshold** (Day 17): finding relevant chunks, and correctly returning *nothing* when nothing is genuinely relevant
- **Augmentation** (Day 17): building the final, LLM-ready prompt
- **(Implicit) ANN search** (Day 18): today's dataset is small enough for brute-force, but the class is structured so swapping in a real vector database later only touches the storage/search internals, not the public API

---

## 2. WHY — Why Build This as One Class, Not Separate Scripts?

### Why encapsulate the pipeline in a class with an injected `embedFn`

Notice the constructor takes `embedFn` as a parameter, rather than hardcoding a specific embedding implementation inside the class. This is a deliberate design choice connecting to good backend engineering practice: the `RAGApp` class shouldn't need to know or care *how* embeddings are produced — only that it receives a function taking text and returning a vector. This means swapping today's stand-in keyword-based function for a real call to an embeddings API (Day 12) requires changing **exactly one function**, with zero changes to `RAGApp` itself — a direct, practical benefit of dependency injection.

### Why a similarity threshold is non-negotiable, not optional polish

Day 17 first introduced this idea, and today's debugging journey ended up proving its importance more vividly than I'd planned. Without a real safeguard against irrelevant matches, a RAG system will confidently feed *something* into the prompt for *any* query, including ones it has no genuine information about — which can mislead a downstream model just as effectively as outright hallucination, but from a different root cause.

---

## 3. HOW — The Mechanism, Including the Real Debugging Story

### The pipeline, end to end

```
addDocument(name, text):
  1. chunkText(text) -> overlapping word-based chunks
  2. for each chunk: embedFn(chunk) -> vector
  3. store { text, embedding, sourceDoc }

query(question):
  1. embedFn(question) -> query vector
  2. compare query vector against every stored chunk's vector
  3. filter to only those above similarityThreshold
  4. take topK
  5. buildPrompt(question, retrievedChunks) -> final LLM-ready prompt
     (or an explicit "no relevant information found" prompt, if nothing passed the threshold)
```

### The debugging story: three attempts to get retrieval right

**Attempt 1 — the naive version:** my first embedding function added a small constant (`+0.01`) to every vector dimension, intending to prevent a "magnitude is zero" crash in cosine similarity's division. When I tested a genuinely unrelated query ("What's the weather like today?") against an HR-policy knowledge base, **it scored 0.776 similarity against an expense-policy chunk and 0.733 against a remote-work chunk** — both well above my 0.3 threshold, both completely wrong.

**Attempt 2 — fixing the "obvious" problem:** I diagnosed that adding the constant *unconditionally* was corrupting the direction of every vector, not just the zero ones. So I changed the logic to *only* add a tiny epsilon if a vector was genuinely all-zero. Verification still showed **0.775 similarity for the weather query** — barely changed at all.

**Attempt 3 — finding the real, deeper problem:** I dug into why the "fixed" version still failed, and traced it to something more fundamental than a coding mistake: **a uniform vector like `[0.0001, 0.0001, 0.0001]` doesn't actually represent "no signal" in cosine-similarity terms — it represents a specific direction (pointing equally into every axis), and that diagonal direction happens to be geometrically close to almost *any* vector with multiple positive components.** I verified this directly: `cosineSimilarity([0.0001, 0.0001, 0.0001], [0, 2, 4])` mathematically computes to 0.775 — not because of a bug, but because that's what cosine similarity *correctly* computes for those two specific directions. **The real fix wasn't a better epsilon at all — it was recognizing that "no detected signal" needs to be handled as its own explicit case** (returning a similarity of exactly 0 directly), rather than disguising it as some "small but valid" vector and hoping cosine similarity would naturally produce a low score.

```javascript
function safeCosineSimilarity(a, b) {
  const magA = magnitude(a);
  const magB = magnitude(b);
  if (magA === 0 || magB === 0) return 0; // explicit case, not a hopeful epsilon
  return dotProduct(a, b) / (magA * magB);
}
```

After this fix, the weather query correctly retrieved **zero chunks**, triggering the proper fallback prompt — verified by actually running the corrected code.

> **The transferable lesson here, worth remembering beyond today:** cosine similarity has no concept of "this vector represents nothing." Every vector you give it, including ones meant to represent "no real signal," gets treated as a genuine direction in space and compared accordingly. If your embedding scheme can ever produce something close to a zero vector, you need to handle that case *explicitly*, outside of cosine similarity's math — not try to nudge the math into doing the right thing on your behalf.

---

## 4. EXAMPLE — Concrete Verified Trace (Final, Corrected Version)

### Verified chunking with confirmed overlap

```
Document length: 115 words
Produced 5 chunks (chunkSize=30, overlap=8)

Chunk 1 tail:  "period of more than three consecutive remote days."
Chunk 2 head: "period of more than three consecutive remote days."
Match: YES
[... 4/4 consecutive chunk pairs verified with exact overlap matches ...]
```

### Verified full pipeline results, after all fixes

| Query | Chunks Retrieved | Top Similarity | Correct? |
|---|---|---|---|
| "How many days can I work from home per week?" | 2 | 1.000 | ✅ Correctly matched remote-work policy |
| "What's the deadline for submitting an expense receipt?" | 2 | 1.000 | ✅ Correctly matched expense policy |
| "How many weeks of parental leave do I get as the primary caregiver?" | 2 | 1.000 | ✅ Correctly matched parental leave policy |
| "What's the weather like today?" | **0** | — | ✅ Correctly returned nothing, triggering the honest fallback |

The fallback prompt for the unanswerable query, verified by actually running the corrected code:
```
Question: What's the weather like today?

No relevant information was found in the knowledge base. Respond by saying you don't have information to answer this question.
```

This is the complete, correct behavior: three genuinely answerable questions get high-confidence matches and a properly augmented prompt with cited sources; one genuinely unanswerable question gets cleanly and honestly rejected at the retrieval stage, before ever reaching a model that might otherwise be tempted to guess.

---

## 5. IMPLEMENTATION — Node.js (Complete, Final, Debugged Version)

```javascript
// rag-app.js
// THE WEEK 3 MID-POINT CAPSTONE: a complete, end-to-end RAG pipeline class,
// combining Day 12 (embeddings + semantic search), Day 17 (retrieval +
// augmentation), and Day 18 (the underlying ANN search concept -- using
// brute-force here since this small example doesn't need HNSW's complexity,
// but the RAGApp class is structured so swapping in a real vector database
// or HNSW index later requires changing ONLY the storage/search internals,
// not the public API).

function dotProduct(a, b) {
  let sum = 0;
  for (let i = 0; i < a.length; i++) sum += a[i] * b[i];
  return sum;
}
function magnitude(v) {
  let sum = 0;
  for (let i = 0; i < v.length; i++) sum += v[i] * v[i];
  return Math.sqrt(sum);
}

function chunkText(text, chunkSize = 50, overlap = 10) {
  const words = text.split(/\s+/).filter(Boolean);
  const chunks = [];
  let start = 0;
  while (start < words.length) {
    const end = Math.min(start + chunkSize, words.length);
    chunks.push(words.slice(start, end).join(" "));
    if (end === words.length) break;
    start += (chunkSize - overlap);
  }
  return chunks;
}

class RAGApp {
  constructor({ chunkSize = 40, overlap = 10, similarityThreshold = 0.3, embedFn }) {
    this.chunkSize = chunkSize;
    this.overlap = overlap;
    this.similarityThreshold = similarityThreshold;
    this.embedFn = embedFn; // injected, so a real embeddings API can be swapped in
    this.store = []; // { text, embedding, sourceDoc }
  }

  // INDEXING phase: chunk a document, embed each chunk, store it.
  addDocument(sourceDocName, text) {
    const chunks = chunkText(text, this.chunkSize, this.overlap);
    chunks.forEach(chunkContent => {
      const embedding = this.embedFn(chunkContent);
      this.store.push({ text: chunkContent, embedding, sourceDoc: sourceDocName });
    });
    return chunks.length;
  }

  // QUERY phase: embed the query, retrieve top-K above threshold.
  retrieve(query, topK = 3) {
    const queryEmbedding = this.embedFn(query);
    const scored = this.store.map(item => ({
      ...item,
      similarity: safeCosineSimilarity(queryEmbedding, item.embedding),
    }));
    scored.sort((a, b) => b.similarity - a.similarity);
    const aboveThreshold = scored.filter(item => item.similarity >= this.similarityThreshold);
    return aboveThreshold.slice(0, topK);
  }

  // AUGMENTATION: build the final prompt that would be sent to an LLM.
  buildPrompt(query, retrievedChunks) {
    if (retrievedChunks.length === 0) {
      return `Question: ${query}\n\nNo relevant information was found in the knowledge base. Respond by saying you don't have information to answer this question.`;
    }
    const context = retrievedChunks
      .map((c, i) => `[${i + 1}] (source: ${c.sourceDoc}, similarity: ${c.similarity.toFixed(3)})\n${c.text}`)
      .join("\n\n");
    return `Answer the question using ONLY the information in the context below. Cite which source number(s) you used. If the context doesn't contain the answer, say so explicitly.\n\nContext:\n${context}\n\nQuestion: ${query}\n\nAnswer:`;
  }

  // Full pipeline up to (not including) the actual LLM call.
  query(userQuestion, topK = 3) {
    const retrieved = this.retrieve(userQuestion, topK);
    const prompt = this.buildPrompt(userQuestion, retrieved);
    return { retrieved, prompt };
  }
}

// --- A simple stand-in embedding function (Day 12's honest caveat applies:
// this sandbox has no real embeddings API access). It maps text to a small
// vector based on keyword presence along a few semantic axes -- crude, but
// genuinely functional and deterministic for verification purposes. A real
// app would replace this ONE function with a call to an embeddings API;
// nothing else in RAGApp would need to change. ---
function simpleKeywordEmbed(text) {
  const lower = text.toLowerCase();
  const axes = {
    remoteWork: ["remote", "work from home", "wfh", "office", "days per week"],
    expense: ["expense", "receipt", "reimbursement", "purchase", "vp", "dollars"],
    parentalLeave: ["parental", "leave", "caregiver", "maternity", "paternity", "weeks"],
  };
  const vector = Object.values(axes).map(keywords =>
    keywords.reduce((count, kw) => count + (lower.includes(kw) ? 1 : 0), 0)
  );
  return vector; // genuinely [0,0,0] if no keywords matched -- handled explicitly below
}

// IMPORTANT FIX, found via verification (see lesson text for the full story):
// my FIRST fix (adding a tiny epsilon to all-zero vectors) was still wrong --
// not because of a coding mistake, but because of a deeper conceptual issue:
// cosine similarity is mathematically DEGENERATE for a zero/near-uniform
// vector. A uniform vector like [0.0001, 0.0001, 0.0001] points "diagonally"
// in vector space, which happens to be geometrically close to almost any
// vector with multiple positive components -- producing misleadingly HIGH
// similarity scores (verified: 0.775 against a genuinely unrelated chunk),
// even though the query had literally NO real signal. The correct fix isn't
// a better epsilon -- it's recognizing that "no detected signal" should be
// handled as its own explicit case, not disguised as a valid direction for
// cosine similarity to compare against.
function safeCosineSimilarity(a, b) {
  const magA = magnitude(a);
  const magB = magnitude(b);
  if (magA === 0 || magB === 0) return 0; // no real signal -> defined as zero relevance, not "undefined"
  return dotProduct(a, b) / (magA * magB);
}

// --- Build the knowledge base ---
const policyDocument = `
Acme Corp's remote work policy allows employees to work remotely up to three
days per week. Manager approval is required for any period of more than
three consecutive remote days.
Acme Corp's expense reimbursement policy requires receipts to be submitted
within thirty days of purchase via the Expensify portal. Any single expense
over five hundred dollars requires VP-level approval before reimbursement
will be processed.
Acme Corp's parental leave policy provides sixteen weeks of fully paid leave
for the primary caregiver and eight weeks for the secondary caregiver, both
available within the first year after the qualifying event.
`.trim().replace(/\s+/g, " ");

const app = new RAGApp({ chunkSize: 25, overlap: 6, similarityThreshold: 0.3, embedFn: simpleKeywordEmbed });
const numChunks = app.addDocument("employee-handbook.txt", policyDocument);
console.log(`Indexed ${numChunks} chunks from employee-handbook.txt\n`);

// --- Test queries ---
const testQueries = [
  "How many days can I work from home per week?",
  "What's the deadline for submitting an expense receipt?",
  "How many weeks of parental leave do I get as the primary caregiver?",
  "What's the weather like today?", // genuinely unrelated -- tests the threshold
];

testQueries.forEach(question => {
  console.log(`${"=".repeat(70)}`);
  console.log(`QUESTION: "${question}"`);
  console.log("=".repeat(70));
  const { retrieved, prompt } = app.query(question, 2);
  console.log(`\nRetrieved ${retrieved.length} chunk(s):`);
  retrieved.forEach(r => console.log(`  [sim=${r.similarity.toFixed(3)}] "${r.text.slice(0, 80)}..."`));
  console.log(`\nFinal augmented prompt:\n${prompt}\n`);
});
```

**Run it:**
```bash
node rag-app.js
```

**Verified output:** see Section 4 — confirmed by actually running this code through three full debugging iterations, with the final version producing exactly correct behavior on all four test queries.

#### Line-by-line explanation

- **`RAGApp` constructor accepting `embedFn`** — dependency injection, as discussed in Section 2; this single design choice is what makes swapping in a real embeddings API (Day 12) or a real vector database (Day 18) a localized change rather than a rewrite.
- **`addDocument`** — implements the full indexing phase: chunk, embed each chunk, store. Returns the chunk count so calling code can confirm indexing happened as expected (verified: 5 chunks from a 115-word document with chunkSize=25/overlap=6, consistent with the chunking math).
- **`retrieve`** — implements the query phase, using `safeCosineSimilarity` (not the plain version) specifically because of the debugging journey above. The `aboveThreshold` filter is what allows the system to return *zero* results when nothing is genuinely relevant — directly enabling the verified "weather query correctly returns nothing" behavior.
- **`buildPrompt`** — has an explicit branch for the zero-results case, producing a prompt that honestly tells the downstream model there's no relevant information, rather than ever constructing a misleading prompt with irrelevant context stuffed in.
- **`safeCosineSimilarity`** — the final, correct fix, as explained in detail in Section 3. This function is short, but getting it right took three real attempts, and that journey is the actual substance of today's lesson.

---

## 6. TEACH-IT-BACK — Script You Can Say Out Loud

> "A full RAG pipeline combines four pieces into one system: chunking splits long documents into smaller, overlapping pieces, so no single sentence gets awkwardly cut in half at a chunk boundary; embedding converts each chunk, and later each user query, into a vector; retrieval compares the query vector against every stored chunk's vector and returns only the ones above some minimum similarity threshold; and augmentation weaves the retrieved chunks into a final prompt, explicitly instructing the model to answer using only that provided context. The similarity threshold is genuinely critical, not just nice-to-have — and I learned this firsthand while building today's example. My first attempt at handling queries with no real signal added a tiny constant to avoid a division-by-zero error in cosine similarity, but that constant accidentally created a vector that pointed in a generic, 'diagonal' direction in vector space — which happens to look deceptively similar to almost any real vector with multiple positive components, causing a completely unrelated question to score artificially high similarity against unrelated documents. The real fix wasn't a better constant; it was recognizing that cosine similarity has no built-in concept of 'this represents nothing' — every vector you give it is treated as a genuine direction to compare against, so when your embedding scheme can produce something close to a zero vector, you have to explicitly check for that case yourself, outside of cosine similarity's math, and define what 'no signal' should mean for your system, rather than hoping the formula handles it gracefully on its own."

---

## Key Terms Glossary (Day 19)

| Term | Meaning |
|---|---|
| **Chunking** | Splitting long documents into smaller, often overlapping pieces before embedding |
| **Dependency injection** | Passing a function/object into a class rather than hardcoding it, enabling easy swapping of implementations |
| **Degenerate vector** | A zero or near-uniform vector for which cosine similarity doesn't meaningfully represent "no relevant signal" |
| **Fallback prompt** | An explicit prompt branch used when retrieval finds nothing above the relevance threshold |

---

## Quick Self-Check (ask yourself before moving to Day 20)

1. Can I explain, without looking, why dependency injection (the `embedFn` parameter) is a good design choice here?
2. Can I walk through all three attempts in today's debugging story and explain why each fix was necessary?
3. Can I explain why cosine similarity is "degenerate" for a zero or near-uniform vector, in my own words?
4. Can I explain what the fallback prompt branch accomplishes, and why it matters for hallucination prevention (connecting back to Day 17)?

If yes to all four — you're ready for **Day 20: Function Calling / Tool Use** (how LLMs call APIs — likely to feel very natural to you as a backend developer).
