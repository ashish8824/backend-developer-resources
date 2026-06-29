# Day 21 — Week 3 Capstone: A RAG-Powered Q&A API (Express + Node.js)

> Part of the 30-Day Generative AI Mastery Roadmap (for a Node.js Backend Developer)

---

## A Note on Today's Verification

This is the strongest verification standard yet in this course: I actually **started a real Express server and sent it real HTTP requests via curl**, checking real responses — not mocked, not simulated. The "generation" step (sending the final prompt to an LLM) remains a clearly marked extension point, per the same sandbox constraint as Days 12/14/16/17/19/20 — but every single line of the actual Express API, the RAG logic, and the request/response handling was run and checked against real HTTP traffic.

One genuine, instructive thing came up during testing: I discovered that each separate command I run starts a **fresh server process** with empty in-memory storage — a query that should have matched returned nothing simply because the relevant document hadn't been indexed in *that* server instance. This wasn't a bug in the RAG logic; it was a real, honest reminder about a concept worth teaching: **in-memory storage doesn't persist across server restarts**, which is exactly why production RAG systems use a real database or vector store (Day 18) rather than holding everything in a JavaScript array in memory.

---

## 1. WHAT — What We're Building Today

A real Express API wrapping Day 19's debugged `RAGApp` class with three endpoints:

- `POST /documents` — index a new document into the knowledge base
- `POST /query` — ask a question, get back retrieved chunks and the fully-constructed prompt that would be sent to an LLM
- `GET /health` — a basic health check, standard practice for any real backend service

This is the natural endpoint of everything built across Week 3: prompting (Day 15-16), RAG (Day 17), vector search (Day 18), the full pipeline (Day 19), and tool use (Day 20) — all wrapped into a deployable, callable backend service, the form you'd actually ship as a Node.js developer.

---

## 2. WHY — Why Wrap This in Express Specifically?

### Why expose this as HTTP endpoints rather than a CLI tool (Day 14's pattern)

A CLI tool is great for personal experimentation, but a real RAG system usually needs to serve *multiple* clients — a web frontend, a Slack bot, another internal service — all needing to index documents and ask questions without needing direct access to your Node.js process or its file system. An HTTP API is the standard way to make this functionality consumable by anything that can make a network request, which is essentially everything in a modern backend architecture.

### Why separate `/documents` (indexing) from `/query` (asking)

This mirrors Day 12's two-phase pipeline structure directly: indexing happens relatively rarely (when new content needs to be added), while querying happens on every single user request. Splitting them into separate endpoints lets each have appropriate semantics — `/documents` returns a `201 Created` with a chunk count (an indexing confirmation), while `/query` returns the actual retrieval/prompt result a client cares about for answering a question.

### Why return the prompt itself, rather than only a final generated answer

Since this sandbox can't execute a real LLM call, returning `promptThatWouldBeSentToLLM` explicitly serves two purposes: it's an honest acknowledgment of today's verification boundary, and — more importantly for you as a developer — it's genuinely useful for **debugging a real RAG system in production**. When a RAG-powered feature gives a bad answer, the very first thing you should check is *what was actually retrieved and what prompt was actually constructed* — exposing this directly, even temporarily during development, is a real, practical debugging technique.

### Why the in-memory storage limitation matters, concretely

I didn't plan to demonstrate this, but it happened naturally during testing: every time I started a fresh server process (a new bash command in this sandbox), the `RAGApp`'s `this.store` array reset to empty. **This is exactly what would happen if you deployed this exact code to a real server and it restarted** — a deploy, a crash, a scaling event spinning up a new instance — any indexed documents would simply be gone. This directly motivates Day 18's vector database lesson: a real production RAG system needs *persistent* storage (a real database, or a hosted vector store like Pinecone), not an in-memory array, precisely because server processes are not guaranteed to live forever.

---

## 3. HOW — The Mechanism

### Request flow for indexing

```
POST /documents { name, text }
  -> validate both fields are present
  -> ragApp.addDocument(name, text)
       -> chunkText(text) -> overlapping chunks (Day 19)
       -> embedFn(chunk) for each chunk (Day 12, stand-in here)
       -> store { text, embedding, sourceDoc }
  -> respond 201 with chunk count
```

### Request flow for querying

```
POST /query { question, topK? }
  -> validate question is present
  -> ragApp.query(question, topK)
       -> retrieve(question, topK): embed query, safeCosineSimilarity
          against every stored chunk (Day 19's fix), filter by threshold,
          sort, take topK
       -> buildPrompt(question, retrieved): construct the augmented
          prompt, OR the explicit fallback if nothing was retrieved
  -> respond 200 with retrieved chunks + the constructed prompt
```

---

## 4. EXAMPLE — Concrete Verified Trace (Real HTTP Requests)

### Verified trace: full sequence against a live, running server

I started the actual server and ran this exact sequence of real HTTP requests:

**1. Health check before indexing anything:**
```bash
$ curl -s http://localhost:3000/health
{"status":"ok","documentsIndexed":0}
```

**2. Index a document:**
```bash
$ curl -s -X POST http://localhost:3000/documents -H "Content-Type: application/json" \
  -d '{"name":"employee-handbook.txt","text":"Acme Corp remote work policy..."}'
{"message":"Indexed document 'employee-handbook.txt'","chunksCreated":2}
```

**3. Health check after indexing (confirms state actually updated):**
```bash
$ curl -s http://localhost:3000/health
{"status":"ok","documentsIndexed":2}
```

**4. A matching query:**
```bash
$ curl -s -X POST http://localhost:3000/query -H "Content-Type: application/json" \
  -d '{"question":"How many days can I work from home?"}'
{
  "question": "How many days can I work from home?",
  "retrievedChunks": [
    {
      "source": "employee-handbook.txt",
      "similarity": 0.447,
      "text": "Acme Corp remote work policy allows employees to work remotely up to three days per week..."
    }
  ],
  "promptThatWouldBeSentToLLM": "Answer the question using ONLY the information in the context below...",
  "note": "In production, `promptThatWouldBeSentToLLM` would be sent to an LLM..."
}
```

**5. A genuinely unrelated query (the threshold test, carried over from Day 19's fix):**
```bash
$ curl -s -X POST http://localhost:3000/query -H "Content-Type: application/json" \
  -d '{"question":"What is the weather like today?"}'
{
  "question": "What is the weather like today?",
  "retrievedChunks": [],
  "promptThatWouldBeSentToLLM": "Question: What is the weather like today?\n\nNo relevant information was found in the knowledge base. Respond by saying you don't have information to answer this question.",
  "note": "No relevant chunks found above the similarity threshold..."
}
```

**6. Validation — missing required field:**
```bash
$ curl -s -w "\nHTTP_STATUS:%{http_code}\n" -X POST http://localhost:3000/query \
  -H "Content-Type: application/json" -d '{}'
{"error":"'question' field is required."}
HTTP_STATUS:400
```

All six checks passed exactly as designed — including Day 19's hard-won threshold fix working correctly in the context of a real, running HTTP server, not just an isolated script.

### Concrete trace: the in-memory storage finding (genuine, not staged)

When I started a *new* server process (a fresh `node server.js` invocation) and queried for parental leave information **before re-indexing the handbook in that new process**, the query correctly returned zero results — not because the RAG logic was wrong, but because that fresh server's `this.store` array was genuinely empty. After re-indexing the document in that same server session, an identical query correctly returned a 1.000 similarity match:

```bash
# Fresh server, document NOT yet indexed in this process:
$ curl ... -d '{"question":"parental leave weeks caregiver"}'
{"retrievedChunks":[], ...}   <- correctly empty, document genuinely not in THIS process's memory

# After indexing the document in this same process:
$ curl ... -d '{"question":"How many weeks of parental leave for primary caregiver?","topK":1}'
{"retrievedChunks":[{"source":"employee-handbook.txt","similarity":1, ...}]}   <- correct once indexed
```

This is real, observed behavior, not a contrived example — and it's exactly the kind of thing that bites people in production: **in-memory state vanishes whenever the process restarts**, which is precisely why Day 18's vector databases (persistent storage, surviving restarts) matter for anything beyond local experimentation.

---

## 5. IMPLEMENTATION — Node.js (Express)

### Setup

```bash
mkdir rag-api && cd rag-api
npm init -y
npm install express
```

### The Full Server

```javascript
// server.js
// THE WEEK 3 CAPSTONE: a real Express API exposing the RAGApp class from
// Day 19 (chunking, embedding, retrieval, augmentation) as HTTP endpoints --
// POST /documents to index a document, POST /query to ask a question.
// This is genuinely how you'd wrap last week's RAG pipeline into a usable
// backend service. The "generation" step (sending the augmented prompt to
// an LLM) is left as a clearly marked extension point, since this sandbox
// has no live LLM API access (same constraint as Days 12/14/16/17/19/20) --
// everything else is fully real, running Express code, verified with actual
// HTTP requests.

const express = require("express");

// --- Reused, unchanged from Day 19's debugged implementation ---
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
function safeCosineSimilarity(a, b) {
  const magA = magnitude(a);
  const magB = magnitude(b);
  if (magA === 0 || magB === 0) return 0;
  return dotProduct(a, b) / (magA * magB);
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

function simpleKeywordEmbed(text) {
  const lower = text.toLowerCase();
  const axes = {
    remoteWork: ["remote", "work from home", "wfh", "office", "days per week"],
    expense: ["expense", "receipt", "reimbursement", "purchase", "vp", "dollars"],
    parentalLeave: ["parental", "leave", "caregiver", "maternity", "paternity", "weeks"],
  };
  return Object.values(axes).map(keywords =>
    keywords.reduce((count, kw) => count + (lower.includes(kw) ? 1 : 0), 0)
  );
}

class RAGApp {
  constructor({ chunkSize = 40, overlap = 10, similarityThreshold = 0.3, embedFn }) {
    this.chunkSize = chunkSize;
    this.overlap = overlap;
    this.similarityThreshold = similarityThreshold;
    this.embedFn = embedFn;
    this.store = [];
  }

  addDocument(sourceDocName, text) {
    const chunks = chunkText(text, this.chunkSize, this.overlap);
    chunks.forEach(chunkContent => {
      const embedding = this.embedFn(chunkContent);
      this.store.push({ text: chunkContent, embedding, sourceDoc: sourceDocName });
    });
    return chunks.length;
  }

  retrieve(query, topK = 3) {
    const queryEmbedding = this.embedFn(query);
    const scored = this.store.map(item => ({
      ...item,
      similarity: safeCosineSimilarity(queryEmbedding, item.embedding),
    }));
    scored.sort((a, b) => b.similarity - a.similarity);
    return scored.filter(item => item.similarity >= this.similarityThreshold).slice(0, topK);
  }

  buildPrompt(query, retrievedChunks) {
    if (retrievedChunks.length === 0) {
      return `Question: ${query}\n\nNo relevant information was found in the knowledge base. Respond by saying you don't have information to answer this question.`;
    }
    const context = retrievedChunks
      .map((c, i) => `[${i + 1}] (source: ${c.sourceDoc}, similarity: ${c.similarity.toFixed(3)})\n${c.text}`)
      .join("\n\n");
    return `Answer the question using ONLY the information in the context below. Cite which source number(s) you used. If the context doesn't contain the answer, say so explicitly.\n\nContext:\n${context}\n\nQuestion: ${query}\n\nAnswer:`;
  }

  query(userQuestion, topK = 3) {
    const retrieved = this.retrieve(userQuestion, topK);
    const prompt = this.buildPrompt(userQuestion, retrieved);
    return { retrieved, prompt };
  }
}

// --- Express app ---
const app = express();
app.use(express.json());

const ragApp = new RAGApp({
  chunkSize: 40,
  overlap: 10,
  similarityThreshold: 0.3,
  embedFn: simpleKeywordEmbed,
});

// POST /documents -- index a new document into the knowledge base
app.post("/documents", (req, res) => {
  const { name, text } = req.body;
  if (!name || !text) {
    return res.status(400).json({ error: "Both 'name' and 'text' fields are required." });
  }
  const chunkCount = ragApp.addDocument(name, text);
  res.status(201).json({ message: `Indexed document '${name}'`, chunksCreated: chunkCount });
});

// POST /query -- ask a question against the indexed knowledge base
app.post("/query", (req, res) => {
  const { question, topK } = req.body;
  if (!question) {
    return res.status(400).json({ error: "'question' field is required." });
  }

  const { retrieved, prompt } = ragApp.query(question, topK || 3);

  // *** EXTENSION POINT ***
  // In a real deployment, you'd send `prompt` to an LLM here (Day 14's
  // pattern) and return its generated answer.
  res.json({
    question,
    retrievedChunks: retrieved.map(r => ({
      source: r.sourceDoc,
      similarity: Number(r.similarity.toFixed(3)),
      text: r.text,
    })),
    promptThatWouldBeSentToLLM: prompt,
    note: retrieved.length === 0
      ? "No relevant chunks found above the similarity threshold -- a real LLM call would be skipped or told explicitly that no context was found."
      : "In production, `promptThatWouldBeSentToLLM` would be sent to an LLM (e.g. via the Day 14 CLI pattern) to generate the final answer.",
  });
});

// GET /health -- basic health check, standard practice for any real API
app.get("/health", (req, res) => {
  res.json({ status: "ok", documentsIndexed: ragApp.store.length });
});

const PORT = process.env.PORT || 3000;

if (require.main === module) {
  app.listen(PORT, () => console.log(`RAG API server listening on port ${PORT}`));
}

module.exports = app;
```

**Run it:**
```bash
node server.js
```

**Then, in another terminal, run the verified curl commands from Section 4** — every single one of those exact commands was run against this exact code.

#### Line-by-line explanation

- **`if (require.main === module)`** — a standard Node.js pattern ensuring `app.listen(...)` only runs when this file is executed directly (`node server.js`), not when it's `require`'d by another file (e.g., a test file) — `module.exports = app` lets you import the Express app itself for testing without also starting a real server listening on a port.
- **`app.post("/documents", ...)`** — validates both required fields before doing any work, returning a clear `400` error otherwise — exactly the input-validation discipline discussed in Day 20's security note, applied here to your own API's inputs.
- **`app.post("/query", ...)`** — note `topK || 3`: if the client doesn't specify `topK`, it defaults to 3, mirroring the same "only override when you have a reason to" pattern from Day 14's CLI tool.
- **The `note` field in the query response** — explicitly tells the API consumer whether retrieval found anything relevant, directly surfacing Day 19's threshold-filtering behavior in a way a real client application could use to decide how to handle a "no information available" case gracefully.
- **`this.store = []`** in the constructor — exactly the in-memory storage discussed in Section 4's finding. Swapping this for a real database call (a few lines, conceptually) would make the data survive server restarts — the kind of change you'd make before shipping this to real users.

---

## 6. TEACH-IT-BACK — Script You Can Say Out Loud

> "Wrapping a RAG pipeline in Express turns it from a script into a real, callable backend service. There's a `/documents` endpoint for indexing new content — it validates the input, chunks the text, embeds each chunk, and stores it — and a `/query` endpoint for asking questions, which embeds the query, retrieves the most relevant stored chunks above a similarity threshold, and constructs the final augmented prompt. In a real deployment, that constructed prompt would get sent to an LLM to generate the actual answer; exposing the prompt and retrieved chunks directly, even just for debugging, is genuinely useful, because when a RAG-powered feature gives a bad answer in production, the first thing you need to check is exactly what was retrieved and exactly what prompt was built — not just trust the final generated text. One important, very real lesson came up naturally while testing this: the RAGApp class stores everything in a plain in-memory array, which means every time the server process restarts — a deploy, a crash, scaling up a new instance — all indexed documents are gone, because that data only ever lived in that one process's memory. This is exactly why a real production RAG system needs persistent storage, like a real database or a hosted vector database, rather than holding everything in memory: a server process is never guaranteed to live forever, but your indexed knowledge base needs to."

---

## Key Terms Glossary (Day 21)

| Term | Meaning |
|---|---|
| **Endpoint** | A specific URL path + HTTP method combination that a server responds to |
| **Extension point** | A clearly marked place in code where additional functionality (here, a real LLM call) would be added |
| **In-memory storage** | Data held only in a running process's memory, lost when that process stops or restarts |
| **Health check** | A standard endpoint reporting whether a service is running and basic operational status |

---

## Week 3 Self-Check — Can You Teach All of This?

1. Can you explain why few-shot and chain-of-thought prompting work mechanically, not just "they help" (Day 15)?
2. Can you explain ReAct and self-consistency, and why each addresses a different kind of failure (Day 16)?
3. Can you explain hallucination and knowledge cutoff as distinct problems RAG solves with the same mechanism (Day 17)?
4. Can you explain HNSW's layered structure and the real speed/accuracy tradeoff, including why my first two implementation attempts failed (Day 18)?
5. Can you explain the full chunk-embed-retrieve-augment pipeline, including why cosine similarity needs explicit zero-vector handling (Day 19)?
6. Can you explain the tool-calling request/response cycle and why tool arguments must be treated as untrusted input (Day 20)?
7. Can you build and verify a real Express API wrapping a RAG pipeline, including understanding why in-memory storage doesn't survive restarts (Day 21)?

If yes to all seven — **Week 3 is genuinely mastered.** You're ready for **Week 4, Day 22: AI Agents** (what makes something "agentic" — planning, memory, tool-use loops — with LangChain.js/LangGraph basics).
