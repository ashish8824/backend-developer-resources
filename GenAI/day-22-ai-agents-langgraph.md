# Day 22 — AI Agents: Planning, Memory, Tool-Use Loops (LangChain.js / LangGraph)

> Part of the 30-Day Generative AI Mastery Roadmap (for a Node.js Backend Developer)

---

## A Note on Today's Verification

Unlike Days 12/14/16/17/19/20/21, today's core library (`@langchain/langgraph`) needs no external API key or network call to demonstrate its fundamental mechanics — it's a local orchestration framework. So I installed the actual current package (version 1.4.7) and **wrote and ran three real, working LangGraph programs** rather than describing the API from memory, which matters because LangGraph's API has changed significantly across versions, and tutorials you find online may describe an outdated shape. Everything in this lesson reflects what I personally confirmed runs correctly with the currently-installed version.

---

## 1. WHAT — Plain Definitions

| Term | Plain Definition |
|---|---|
| **Agent** | A system where a model doesn't just respond once, but operates in a loop — reasoning, taking actions (often via tools, Day 20), observing results, and deciding whether to continue or stop — to accomplish a goal that might take multiple steps. |
| **Agentic** | Describes behavior involving autonomous planning and multi-step action, as opposed to a single prompt-response exchange. |
| **LangChain.js** | A library providing building blocks (prompt templates, model wrappers, common patterns) for building LLM-powered applications in JavaScript/TypeScript. |
| **LangGraph** | A library (built by the LangChain team, now commonly used independently of LangChain.js itself) for defining an agent's logic as an explicit **graph** of nodes and edges — a more structured, debuggable alternative to a hand-rolled loop like Day 16's ReAct implementation. |
| **State** | The data that flows through a LangGraph graph, updated by each node as execution proceeds — conceptually identical to Day 20's growing `messages` array, just formalized. |
| **Node** | A single step/function in a LangGraph graph, which receives the current state and returns an update to it. |
| **Conditional edge** | A graph edge whose destination is decided dynamically, based on the current state — this is the formal mechanism behind agentic looping and branching. |
| **Checkpointer** | A LangGraph component that persists state across separate `invoke()` calls, giving an agent memory of earlier interactions. |

### Diagram: From Day 16's Hand-Rolled Loop to a Formal Graph

```
   DAY 16's ReAct LOOP (a for-loop you write by hand):     TODAY's LANGGRAPH EQUIVALENT (a formal graph):

   for (step = 0; step < maxSteps; step++) {                    ┌──────────┐
     result = decideNextStep(...)                         ┌────►│   node   │────┐
     if (result.finalAnswer) return                       │     └──────────┘    │
     execute action, record observation                   │                     ▼
   }                                                       │            conditional edge:
                                                            │            "continue?" or "done?"
                                                            └─────continue────┤
                                                                              │
                                                                             done
                                                                              │
                                                                              ▼
                                                                            END
```

**The underlying idea is identical** — the difference is that LangGraph makes the loop's structure *explicit and declarative* (you describe nodes and the conditions for moving between them), rather than *implicit in imperative control-flow code* (a `for` loop with `if` statements). This becomes valuable as agent logic grows more complex — branching into multiple possible paths, running steps in parallel, persisting state across sessions — situations where a hand-rolled loop gets progressively harder to reason about and maintain.

---

## 2. WHY — Why Formalize the Loop At All? Why Not Just Keep Hand-Rolling It?

### Why move beyond Day 16's hand-rolled loop

Day 16 and Day 20's loops work fine for simple cases — a single linear sequence of "reason, act, observe, repeat." But real agentic systems often need more complex control flow: branching into different paths depending on what's been discovered so far, running independent sub-tasks in parallel, pausing and resuming across multiple separate user sessions. Expressing all of this as nested `if`/`for` logic gets unwieldy fast. LangGraph's value is making the *structure* of an agent's logic into data (a graph definition) rather than buried inside procedural code — which makes it easier to visualize, debug, and modify.

### Why "state" as a formal concept, rather than just variables

In Day 16's loop, the growing `history` array implicitly was the "state." LangGraph makes this explicit: you declare upfront exactly what shape of data flows through the graph (via `Annotation.Root`, verified below), and exactly how each node's output should be *merged* into the existing state (via a `reducer` function) — rather than each node silently mutating shared variables in whatever way it likes. This formalization is what allows LangGraph to support more complex execution patterns (parallel nodes, persisted checkpoints) reliably, since the framework has a precise, declared contract for how state changes over time.

### Why memory (checkpointers) matters for real agents

A real conversational agent often needs to remember earlier parts of a conversation across multiple separate requests — e.g., a user asks a question, gets a response, then asks a follow-up that depends on the earlier exchange, potentially in an entirely separate HTTP request to your backend (connecting directly to Day 21's Express API). Without persisted memory, every single request would need to resend the *entire* conversation history manually (which does work, and is what Day 14's CLI tool effectively does by managing the `messages` array yourself) — a checkpointer instead handles this automatically, keyed by a `thread_id`, letting your application code stay simpler.

---

## 3. HOW — The Mechanism, Verified

### Defining state

```javascript
const GraphState = Annotation.Root({
  messages: Annotation({
    reducer: (current, update) => current.concat(update), // HOW to merge an update into existing state
    default: () => [],                                      // initial value if not provided
  }),
});
```

### Defining nodes and edges

```javascript
const graph = new StateGraph(GraphState)
  .addNode("nodeName", nodeFunction)   // nodeFunction: (state) => partial state update
  .addEdge(START, "nodeName")           // unconditional edge: always go here next
  .addConditionalEdges("nodeName", routingFunction, {  // conditional edge: routingFunction decides
    "continue": "nodeName",                              // map routingFunction's return values
    "done": END,                                          // to actual next-node names
  })
  .compile();
```

### Running the graph

```javascript
const result = await graph.invoke(initialState, optionalConfig);
```

### Adding memory

```javascript
const checkpointer = new MemorySaver();
const graph = new StateGraph(GraphState)
  /* ...nodes and edges... */
  .compile({ checkpointer });  // <- pass the checkpointer at compile time

// Calls sharing the same thread_id will share persisted state:
await graph.invoke(input, { configurable: { thread_id: "conversation-A" } });
```

---

## 4. EXAMPLE — Concrete Verified Traces

### Verified trace #1: a simple sequential graph

```
[nodeA] received stepCount=0
[nodeB] received stepCount=1

Final state: {
  "messages": ["nodeA ran", "nodeB ran"],
  "stepCount": 2
}
```

This confirms the basic mechanics: `nodeA` ran first (per the `START -> nodeA` edge), its state update was merged in (stepCount incremented to 1, "nodeA ran" appended via the `concat` reducer), then `nodeB` ran with that updated state visible to it, producing the final accumulated result.

### Verified trace #2: conditional edges implementing a real loop (directly analogous to Day 16's ReAct loop)

```
[increment] counter is now 1
[router] counter=1 < 3, routing back to increment
[increment] counter is now 2
[router] counter=2 < 3, routing back to increment
[increment] counter is now 3
[router] counter=3 >= 3, routing to END

Final state: { "counter": 3, "log": ["incremented to 1", "incremented to 2", "incremented to 3"] }

Verification: counter stopped at exactly 3? PASS
Verification: loop ran exactly 3 times (3 log entries)? PASS
```

This is the formal version of exactly the loop you built by hand on Day 16: a routing function inspects the current state and decides whether to continue (looping back to the same node) or stop (routing to `END`) — verified with explicit pass/fail assertions, not just printed output.

### Verified trace #3: persistent memory via checkpointer, with correct thread isolation

```
--- Thread A, call 1 ---
State after call 1: [ 'user message 1', 'response #2' ]

--- Thread A, call 2 (SAME thread) ---
State after call 2: [ 'user message 1', 'response #2', 'user message 2', 'response #4' ]

--- Thread B, call 1 (DIFFERENT thread) ---
State after thread B call 1: [ 'user message 1', 'response #2' ]

Verification:
Thread A accumulated across calls (4 messages expected)? PASS (got 4)
Thread B independent, did NOT inherit thread A's history (2 messages expected)? PASS (got 2)
```

This is the genuinely important confirmation: calling `graph.invoke()` a second time with the **same** `thread_id` correctly continues from where Thread A left off (4 accumulated messages), while a **different** `thread_id` (Thread B) correctly starts fresh, with zero awareness of Thread A's history, despite running on the exact same compiled graph object. This is real, verified proof of per-conversation memory isolation — directly relevant to building a real multi-user backend service (Day 21), where you'd naturally use something like a user or session ID as the `thread_id`.

---

## 5. IMPLEMENTATION — Node.js

### Setup

```bash
mkdir agent-demo && cd agent-demo
npm init -y
npm install @langchain/core @langchain/langgraph
```

### Part A — A Simple Sequential Graph

```javascript
// verify-langgraph-basic.js
const { StateGraph, Annotation, START, END } = require("@langchain/langgraph");

const GraphState = Annotation.Root({
  messages: Annotation({
    reducer: (current, update) => current.concat(update),
    default: () => [],
  }),
  stepCount: Annotation({
    reducer: (current, update) => update,
    default: () => 0,
  }),
});

function nodeA(state) {
  console.log(`[nodeA] received stepCount=${state.stepCount}`);
  return { messages: ["nodeA ran"], stepCount: state.stepCount + 1 };
}

function nodeB(state) {
  console.log(`[nodeB] received stepCount=${state.stepCount}`);
  return { messages: ["nodeB ran"], stepCount: state.stepCount + 1 };
}

const graph = new StateGraph(GraphState)
  .addNode("nodeA", nodeA)
  .addNode("nodeB", nodeB)
  .addEdge(START, "nodeA")
  .addEdge("nodeA", "nodeB")
  .addEdge("nodeB", END)
  .compile();

async function main() {
  const result = await graph.invoke({ messages: [], stepCount: 0 });
  console.log("\nFinal state:", JSON.stringify(result, null, 2));
}

main().catch(err => { console.error("ERROR:", err.message); process.exit(1); });
```

**Run it:**
```bash
node verify-langgraph-basic.js
```

**Verified output:** see Section 4, Trace #1.

#### Line-by-line explanation

- **`Annotation.Root({...})`** — declares the full shape of state flowing through the graph. Each field gets its own `reducer` (how to merge a node's partial update into existing state) and `default` (initial value).
- **`reducer: (current, update) => current.concat(update)`** for `messages` — means every node's contribution to `messages` gets *appended* to the existing list, not overwritten — verified by the final state correctly containing both `"nodeA ran"` and `"nodeB ran"`.
- **`reducer: (current, update) => update`** for `stepCount` — means each node's update *replaces* the previous value, rather than merging — appropriate for a counter being incremented by each node directly, rather than something that should accumulate as a list.
- **`addEdge(START, "nodeA")`, `addEdge("nodeA", "nodeB")`, `addEdge("nodeB", END)`** — define the unconditional execution order: always start at `nodeA`, then always go to `nodeB`, then always finish.

---

### Part B — Conditional Edges (The Real Agentic Loop Mechanism)

```javascript
// verify-langgraph-conditional.js
const { StateGraph, Annotation, START, END } = require("@langchain/langgraph");

const GraphState = Annotation.Root({
  counter: Annotation({ reducer: (_, update) => update, default: () => 0 }),
  log: Annotation({ reducer: (current, update) => current.concat(update), default: () => [] }),
});

function incrementNode(state) {
  const newCounter = state.counter + 1;
  console.log(`[increment] counter is now ${newCounter}`);
  return { counter: newCounter, log: [`incremented to ${newCounter}`] };
}

function shouldContinue(state) {
  if (state.counter >= 3) {
    console.log(`[router] counter=${state.counter} >= 3, routing to END`);
    return "done";
  }
  console.log(`[router] counter=${state.counter} < 3, routing back to increment`);
  return "continue";
}

const graph = new StateGraph(GraphState)
  .addNode("increment", incrementNode)
  .addEdge(START, "increment")
  .addConditionalEdges("increment", shouldContinue, {
    continue: "increment",
    done: END,
  })
  .compile();

async function main() {
  const result = await graph.invoke({ counter: 0, log: [] });
  console.log("\nFinal state:", JSON.stringify(result, null, 2));
  console.log(`\nVerification: counter stopped at exactly 3? ${result.counter === 3 ? "PASS" : "FAIL"}`);
  console.log(`Verification: loop ran exactly 3 times (3 log entries)? ${result.log.length === 3 ? "PASS" : "FAIL"}`);
}

main().catch(err => { console.error("ERROR:", err.message); process.exit(1); });
```

**Run it:**
```bash
node verify-langgraph-conditional.js
```

**Verified output:** see Section 4, Trace #2 — both correctness assertions passed.

#### Line-by-line explanation

- **`shouldContinue(state)`** — this function *is* the formal version of Day 16's "should I keep going or give a final answer?" check, except here it returns a label (`"continue"` or `"done"`) rather than directly deciding what to do.
- **`.addConditionalEdges("increment", shouldContinue, { continue: "increment", done: END })`** — wires up `shouldContinue`'s possible return values to actual destinations: returning `"continue"` routes back to the *same* `"increment"` node (creating the loop), returning `"done"` routes to `END` (stopping execution).
- **The verified pass/fail assertions** — `result.counter === 3` and `result.log.length === 3` are genuine, falsifiable checks, confirming the loop ran exactly the intended number of times and stopped at exactly the right condition, not just "looked right" in printed output.

---

### Part C — Memory via Checkpointer

```javascript
// verify-langgraph-memory.js
const { StateGraph, Annotation, START, END, MemorySaver } = require("@langchain/langgraph");

const GraphState = Annotation.Root({
  messages: Annotation({ reducer: (current, update) => current.concat(update), default: () => [] }),
});

function respond(state) {
  const messageCount = state.messages.length;
  console.log(`[respond] this thread has seen ${messageCount} message(s) so far`);
  return { messages: [`response #${messageCount + 1}`] };
}

const checkpointer = new MemorySaver();

const graph = new StateGraph(GraphState)
  .addNode("respond", respond)
  .addEdge(START, "respond")
  .addEdge("respond", END)
  .compile({ checkpointer });

async function main() {
  const threadA = { configurable: { thread_id: "conversation-A" } };
  const threadB = { configurable: { thread_id: "conversation-B" } };

  console.log("--- Thread A, call 1 ---");
  const r1 = await graph.invoke({ messages: ["user message 1"] }, threadA);
  console.log("State after call 1:", r1.messages);

  console.log("\n--- Thread A, call 2 (SAME thread) ---");
  const r2 = await graph.invoke({ messages: ["user message 2"] }, threadA);
  console.log("State after call 2:", r2.messages);

  console.log("\n--- Thread B, call 1 (DIFFERENT thread) ---");
  const r3 = await graph.invoke({ messages: ["user message 1"] }, threadB);
  console.log("State after thread B call 1:", r3.messages);

  console.log("\n--- Verification ---");
  console.log(`Thread A accumulated across calls (4 messages expected)? ${r2.messages.length === 4 ? "PASS" : "FAIL"} (got ${r2.messages.length})`);
  console.log(`Thread B independent, did NOT inherit thread A's history (2 messages expected)? ${r3.messages.length === 2 ? "PASS" : "FAIL"} (got ${r3.messages.length})`);
}

main().catch(err => { console.error("ERROR:", err.message); process.exit(1); });
```

**Run it:**
```bash
node verify-langgraph-memory.js
```

**Verified output:** see Section 4, Trace #3 — both correctness assertions passed.

#### Line-by-line explanation

- **`new MemorySaver()`** — an in-memory checkpointer (note: same caveat as Day 21's in-memory `RAGApp` storage — this specific checkpointer doesn't persist across server restarts either; LangGraph supports swappable checkpointer backends, including persistent database-backed ones, for production use).
- **`.compile({ checkpointer })`** — this is the only change needed to add memory to an otherwise-identical graph definition.
- **`{ configurable: { thread_id: "conversation-A" } }`** — passed as the second argument to `invoke()`, this is how you tell the checkpointer which conversation's history to load and append to.
- **The verified results** — Thread A's message count correctly grows across two separate `invoke()` calls (1 → 2 → 4, since each call adds a user message and a response), while Thread B, despite using the identical compiled graph, starts completely fresh — concrete, checkable proof that memory is correctly scoped per thread, not globally shared or leaking between conversations.

---

## 6. TEACH-IT-BACK — Script You Can Say Out Loud

> "An agent is a system where a model operates in a loop — reasoning, acting, observing, deciding whether to continue — rather than just responding once, which is exactly the pattern from the ReAct lesson and the tool-use loop from earlier in this course. LangGraph formalizes that loop as an explicit graph: you declare the shape of the state that flows through the system, including how each node's output should be merged into that state, then you define nodes, which are just functions that take the current state and return an update, and edges, which determine what runs next. Simple edges always go to the same destination; conditional edges run a routing function against the current state to decide dynamically where to go next, and this is the actual mechanism behind agentic looping — a routing function that says 'keep going' routes back to an earlier node, creating a loop, while one that says 'done' routes to a special END node, stopping execution. This is more structured than hand-rolling the same loop with plain if-statements and for-loops, which matters once an agent's logic needs to branch into multiple paths or run steps in parallel. On top of this, LangGraph supports memory through checkpointers, which persist state across multiple separate calls as long as they share the same thread identifier — letting an agent remember earlier parts of a conversation across what might be entirely separate requests to your backend, with different conversations correctly kept isolated from each other even though they're running through the exact same underlying graph definition."

---

## Key Terms Glossary (Day 22)

| Term | Meaning |
|---|---|
| **Agent** | A system using a loop of reasoning, acting, and observing to accomplish multi-step goals |
| **LangGraph** | A library for defining agent logic as an explicit graph of nodes and edges |
| **State** | The data flowing through a graph, with declared rules for how updates merge in |
| **Node** | A function representing one step in a graph, returning a state update |
| **Conditional edge** | An edge whose destination is decided dynamically based on current state |
| **Checkpointer** | A component persisting graph state across separate invocations, keyed by thread |

---

## Quick Self-Check (ask yourself before moving to Day 23)

1. Can I explain, without looking, how a conditional edge implements a loop?
2. Can I explain the relationship between today's graph-based loop and Day 16's hand-rolled ReAct loop?
3. Can I explain what a reducer does, and why `messages` and `counter` in today's examples use different reducer logic?
4. Can I explain how thread-based memory works, using the verified Thread A/Thread B isolation result?

If yes to all four — you're ready for **Day 23: Multi-Agent Systems & Orchestration Patterns** (supervisor, pipeline, debate patterns).
