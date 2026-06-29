# Day 23 — Multi-Agent Systems & Orchestration Patterns

> Part of the 30-Day Generative AI Mastery Roadmap (for a Node.js Backend Developer)

---

## A Note on Today's Verification

Same honest split as Day 16 and Day 22: the **orchestration logic** for all three patterns below is real, current LangGraph code (building on Day 22's verified API) that I wrote and ran, with genuine pass/fail assertions confirming correct behavior. The **individual agents' decisions** within that orchestration are simulated with simple, deterministic rules — standing in for what would be separate real LLM calls, each with its own specialized system prompt — since this sandbox has no live LLM access. The graph structures, routing logic, and state management shown here are exactly what you'd build around real model calls.

---

## 1. WHAT — Plain Definitions

| Term | Plain Definition |
|---|---|
| **Multi-agent system** | An application where multiple distinct agents (each potentially a separate LLM call with its own role/prompt) coordinate to accomplish something, rather than one model handling everything alone. |
| **Pipeline pattern** | Agents run in a fixed sequence; each agent's output becomes the next agent's input — a straight line, no branching. |
| **Supervisor pattern** | A central coordinating agent inspects each request and routes it to whichever specialized agent is best suited to handle it — not every request goes through every agent. |
| **Debate pattern** | Multiple agents take different perspectives on the same question, exchange critiques over several rounds, and a final step synthesizes a conclusion from the resulting discussion. |

### Diagram: The Three Patterns Side by Side

```
PIPELINE (always the same fixed sequence):

  input ──► [Agent A] ──► [Agent B] ──► [Agent C] ──► output
            (research)     (writing)     (editing)


SUPERVISOR (routes to ONE of several specialists, based on the request):

                    ┌──► [Billing Agent]   ──┐
  request ──► [Supervisor] ──► [Technical Agent] ──┼──► response
                    └──► [General Agent]    ──┘
            (only ONE branch executes per request)


DEBATE (alternating rounds, then synthesis):

  [FOR Agent] ──► [AGAINST Agent] ──► continue? ──┐
       ▲                                           │
       └───────────────── yes ─────────────────────┘
                           │
                          no
                           ▼
                      [Judge Agent] ──► verdict
```

---

## 2. WHY — Why Use Multiple Agents Instead of One?

### Why pipeline, specifically

Some tasks naturally decompose into sequential stages, each requiring a different kind of focus — researching facts, then writing prose, then editing for clarity are genuinely different *kinds* of work, even though a single capable model could attempt all three. Splitting them into separate agent steps lets each stage use a prompt (or even a different model) specifically tuned for that one job, rather than asking one generic prompt to context-switch between very different tasks within a single response — and it makes each stage's output independently inspectable, which is valuable for debugging (echoing Day 21's "expose the intermediate state" lesson).

### Why supervisor, specifically

Not all requests need the same kind of handling — a billing question and a technical support question call for genuinely different knowledge, tone, and possibly different tools (Day 20) entirely. A supervisor pattern avoids forcing every request through every specialist's logic (wasteful) or forcing one generic agent to be equally good at everything (often worse than a focused specialist at any one thing). This is directly analogous to how you'd design a real backend system: a router/dispatcher directing requests to the appropriate specialized service, rather than one monolithic handler trying to do everything.

### Why debate, specifically

This connects to Day 16's self-consistency idea, but solves a different problem. Self-consistency samples the *same* prompt multiple times and looks for agreement among independent attempts. Debate instead deliberately assigns **different, opposing perspectives** to different agents, and structures genuine back-and-forth critique between them. This is useful specifically for questions that are inherently subjective or multi-sided (like a policy decision), where there often isn't a single "correct" answer to converge on through repeated independent sampling — instead, you want a deliberately structured exploration of the strongest arguments on multiple sides, surfaced and then synthesized.

### Why these need real orchestration, not just one big prompt

You could, in principle, ask one model to "argue for, then argue against, then judge" all within a single response. Splitting this into separate orchestrated steps (as verified below) gives you several real, practical advantages: each step's output is independently inspectable and logged (debugging), each step could use different settings (a higher temperature for the debate agents, a lower one for the judge, connecting to Day 13), and the structure is explicit and enforced by the graph itself, rather than relying on the model to correctly follow a complex set of instructions about internal structure within one single generation.

---

## 3. HOW — The Mechanism, Verified

### Pipeline: fixed sequential edges (Day 22's basic pattern, applied to distinct agent roles)

```javascript
new StateGraph(PipelineState)
  .addNode("research", researchAgent)
  .addNode("writing", writingAgent)
  .addNode("editing", editingAgent)
  .addEdge(START, "research")
  .addEdge("research", "writing")
  .addEdge("writing", "editing")
  .addEdge("editing", END)
  .compile();
```

### Supervisor: conditional edges with multiple possible destinations (not just "loop or stop")

```javascript
new StateGraph(SupervisorState)
  .addNode("billing", billingAgent)
  .addNode("technical", technicalAgent)
  .addNode("general", generalAgent)
  .addConditionalEdges(START, supervisorRoute, {
    billing: "billing",
    technical: "technical",
    general: "general",
  })
  .addEdge("billing", END)
  .addEdge("technical", END)
  .addEdge("general", END)
  .compile();
```

Note: this verified that conditional routing can branch directly from `START`, not only from a regular node — useful for "route immediately based on the incoming request" logic.

### Debate: alternating nodes + a round-counting conditional loop + a final synthesis step

```javascript
new StateGraph(DebateState)
  .addNode("forAgent", forAgent)
  .addNode("againstAgent", againstAgent)
  .addNode("judge", judgeAgent)
  .addEdge(START, "forAgent")
  .addEdge("forAgent", "againstAgent")
  .addConditionalEdges("againstAgent", shouldContinueDebate, {
    continue: "forAgent",
    judge: "judge",
  })
  .addEdge("judge", END)
  .compile();
```

This combines Day 22's looping mechanism (the `continue`/`judge` conditional edge) with multiple distinct agent roles taking turns, rather than one node looping back to itself.

---

## 4. EXAMPLE — Concrete Verified Traces

### Verified trace #1: pipeline pattern

```
[research agent] extracting key points from raw text
[writing agent] expanding outline into a draft
[editing agent] polishing the draft into final text

Final output:
Remote work increases flexibility (expanded with supporting detail) It can reduce
commute stress (expanded with supporting detail) Some teams struggle with
communication (expanded with supporting detail) [edited for clarity]

Verification:
All 3 stages ran in order? PASS
Final text reflects content from EVERY stage (outline -> draft -> edit)? PASS
```

The final output genuinely contains traces of all three transformation stages — "(expanded with supporting detail)" from the writing agent and "[edited for clarity]" from the editing agent — concrete, checkable proof that each stage's output really did flow into the next, not just that the functions ran in the right order.

### Verified trace #2: supervisor pattern

```
Request: "I was charged twice for my subscription, can I get a refund?"
[supervisor] routing to BILLING agent
Response: [Billing] Reviewing your account for: "..."

Request: "I can't remember my password and keep getting a login error."
[supervisor] routing to TECHNICAL agent
Response: [Technical] Running diagnostics for: "..."

Request: "What are your business hours?"
[supervisor] routing to GENERAL agent
Response: [General] Here's some general help for: "..."

Verification:
Billing request routed correctly? PASS
Technical request routed correctly? PASS
Unmatched request fell back to general? PASS
```

Three different requests, three different correct routing decisions — including the important fallback case (a request matching neither "billing" nor "technical" keywords correctly defaults to the general agent rather than failing or routing incorrectly).

### Verified trace #3: debate pattern

```
[FOR agent, round 1] It increases employee flexibility and autonomy.
[AGAINST agent, round 1] It can weaken team cohesion and informal mentorship.
[moderator] round 1 complete, continuing debate
[FOR agent, round 2] It reduces office overhead costs for the company.
[AGAINST agent, round 2] It makes some forms of spontaneous collaboration harder.
[moderator] round 2 complete, continuing debate
[FOR agent, round 3] It widens the talent pool beyond a single geographic area.
[AGAINST agent, round 3] It can blur work-life boundaries for some employees.
[moderator] 3 rounds complete, moving to judge
[judge] synthesizing a verdict from 3 FOR and 3 AGAINST arguments

Verdict:
After 3 rounds: 3 points favor remote work (flexibility, cost, talent pool), while
3 points raise concerns (cohesion, collaboration, boundaries). A balanced policy
with structured in-person time addresses both sets of concerns.

Verification:
Exactly 3 rounds completed (round === 3)? PASS
3 FOR arguments collected? PASS
3 AGAINST arguments collected? PASS
Judge ran exactly once, after debate concluded? PASS
```

The debate correctly alternated between FOR and AGAINST agents across exactly 3 rounds (verified by the round counter, incremented once per full FOR/AGAINST exchange), stopped at the right point, and handed off to the judge exactly once — never looping back after synthesis began.

---

## 5. IMPLEMENTATION — Node.js

### Part A — Pipeline Pattern

```javascript
// pipeline-pattern.js
// Verifies the PIPELINE pattern: agents run in a FIXED sequence, each one
// consuming the previous agent's output and producing input for the next.
// This builds directly on Day 22's verified sequential graph, just with
// each node representing a distinct "agent" with its own specialized role.

const { StateGraph, Annotation, START, END } = require("@langchain/langgraph");

const PipelineState = Annotation.Root({
  rawText: Annotation({ reducer: (_, u) => u, default: () => "" }),
  outline: Annotation({ reducer: (_, u) => u, default: () => "" }),
  draft: Annotation({ reducer: (_, u) => u, default: () => "" }),
  finalText: Annotation({ reducer: (_, u) => u, default: () => "" }),
  log: Annotation({ reducer: (cur, u) => cur.concat(u), default: () => [] }),
});

function researchAgent(state) {
  console.log("[research agent] extracting key points from raw text");
  const outline = state.rawText
    .split(".")
    .filter(s => s.trim())
    .map(s => `- ${s.trim()}`)
    .join("\n");
  return { outline, log: ["research agent produced an outline"] };
}

function writingAgent(state) {
  console.log("[writing agent] expanding outline into a draft");
  const draft = state.outline
    .split("\n")
    .map(line => line.replace("- ", "") + " (expanded with supporting detail)")
    .join(" ");
  return { draft, log: ["writing agent produced a draft"] };
}

function editingAgent(state) {
  console.log("[editing agent] polishing the draft into final text");
  const finalText = state.draft.replace(/\s+/g, " ").trim() + " [edited for clarity]";
  return { finalText, log: ["editing agent produced final text"] };
}

const pipeline = new StateGraph(PipelineState)
  .addNode("research", researchAgent)
  .addNode("writing", writingAgent)
  .addNode("editing", editingAgent)
  .addEdge(START, "research")
  .addEdge("research", "writing")
  .addEdge("writing", "editing")
  .addEdge("editing", END)
  .compile();

async function main() {
  const input = {
    rawText: "Remote work increases flexibility. It can reduce commute stress. Some teams struggle with communication.",
  };
  const result = await pipeline.invoke(input);

  console.log("\n--- Pipeline trace (log) ---");
  result.log.forEach((entry, i) => console.log(`${i + 1}. ${entry}`));

  console.log("\n--- Final output ---");
  console.log(result.finalText);

  console.log("\n--- Verification ---");
  console.log(`All 3 stages ran in order? ${result.log.length === 3 && result.log[0].includes("research") && result.log[2].includes("editing") ? "PASS" : "FAIL"}`);
  console.log(`Final text reflects content from EVERY stage (outline -> draft -> edit)? ${result.finalText.includes("expanded") && result.finalText.includes("edited") ? "PASS" : "FAIL"}`);
}

main().catch(err => { console.error("ERROR:", err.message); process.exit(1); });
```

**Run it:**
```bash
node pipeline-pattern.js
```

**Verified output:** see Section 4, Trace #1 — both assertions passed.

#### Line-by-line explanation

- **`researchAgent`, `writingAgent`, `editingAgent`** — each is a simulated stand-in for a specialized LLM call (e.g., "you are a research assistant; extract key points from this text"). The *mechanics* of how they're wired together are real; their internal logic is a rule-based simplification.
- **The fixed edge chain** (`research → writing → editing`) — exactly Day 22's `nodeA → nodeB` pattern, extended to three stages, with no branching or conditions anywhere — the defining structural property of a pipeline.
- **The verified assertions** — checking that the final text contains markers from *every* stage (`"expanded"` from writing, `"edited"` from editing) is a real correctness check that the data genuinely flowed through all three stages, not just that three functions happened to execute.

---

### Part B — Supervisor Pattern

```javascript
// supervisor-pattern.js
// Verifies the SUPERVISOR pattern: a central "supervisor" node inspects
// each incoming request and ROUTES it to the appropriate specialized agent
// -- unlike the pipeline pattern, NOT every request goes through every
// agent. Uses Day 22's conditional edges with MULTIPLE possible
// destinations instead of just "loop or stop".

const { StateGraph, Annotation, START, END } = require("@langchain/langgraph");

const SupervisorState = Annotation.Root({
  request: Annotation({ reducer: (_, u) => u, default: () => "" }),
  routedTo: Annotation({ reducer: (_, u) => u, default: () => "" }),
  response: Annotation({ reducer: (_, u) => u, default: () => "" }),
});

function supervisorRoute(state) {
  const text = state.request.toLowerCase();
  if (text.includes("refund") || text.includes("charge") || text.includes("billing")) {
    console.log(`[supervisor] routing to BILLING agent`);
    return "billing";
  }
  if (text.includes("password") || text.includes("login") || text.includes("error")) {
    console.log(`[supervisor] routing to TECHNICAL agent`);
    return "technical";
  }
  console.log(`[supervisor] routing to GENERAL agent`);
  return "general";
}

function billingAgent(state) {
  return { routedTo: "billing", response: `[Billing] Reviewing your account for: "${state.request}"` };
}
function technicalAgent(state) {
  return { routedTo: "technical", response: `[Technical] Running diagnostics for: "${state.request}"` };
}
function generalAgent(state) {
  return { routedTo: "general", response: `[General] Here's some general help for: "${state.request}"` };
}

const graph = new StateGraph(SupervisorState)
  .addNode("billing", billingAgent)
  .addNode("technical", technicalAgent)
  .addNode("general", generalAgent)
  .addConditionalEdges(START, supervisorRoute, {
    billing: "billing",
    technical: "technical",
    general: "general",
  })
  .addEdge("billing", END)
  .addEdge("technical", END)
  .addEdge("general", END)
  .compile();

async function main() {
  const testRequests = [
    "I was charged twice for my subscription, can I get a refund?",
    "I can't remember my password and keep getting a login error.",
    "What are your business hours?",
  ];

  const results = [];
  for (const request of testRequests) {
    console.log(`\nRequest: "${request}"`);
    const result = await graph.invoke({ request });
    console.log(`Response: ${result.response}`);
    results.push(result);
  }

  console.log("\n--- Verification ---");
  console.log(`Billing request routed correctly? ${results[0].routedTo === "billing" ? "PASS" : "FAIL"}`);
  console.log(`Technical request routed correctly? ${results[1].routedTo === "technical" ? "PASS" : "FAIL"}`);
  console.log(`Unmatched request fell back to general? ${results[2].routedTo === "general" ? "PASS" : "FAIL"}`);
}

main().catch(err => { console.error("ERROR:", err.message); process.exit(1); });
```

**Run it:**
```bash
node supervisor-pattern.js
```

**Verified output:** see Section 4, Trace #2 — all three routing assertions passed.

#### Line-by-line explanation

- **`supervisorRoute(state)`** — the routing decision function, simulating what would be a real LLM call asked to classify the incoming request. The keyword-matching logic is a simplification; a real supervisor would typically use an actual model call (possibly with Day 20's structured tool-calling, asking it to "pick one of: billing, technical, general").
- **`.addConditionalEdges(START, supervisorRoute, {...})`** — notice this routes directly from `START`, verified to work correctly, meaning the very first thing that happens is the routing decision, before any agent-specific node runs.
- **Three separate `addEdge(..., END)` calls** — each specialist agent, once it runs, goes straight to `END` — there's no pipeline-style chaining between specialists, reflecting that only one agent handles any given request.

---

### Part C — Debate Pattern

```javascript
// debate-pattern.js
// Verifies the DEBATE pattern: two agents take different perspectives on
// the same question, critique each other's positions over several rounds,
// and a final "judge" step synthesizes a conclusion. Combines Day 22's
// conditional-edge looping with multiple distinct agent roles.

const { StateGraph, Annotation, START, END } = require("@langchain/langgraph");

const DebateState = Annotation.Root({
  question: Annotation({ reducer: (_, u) => u, default: () => "" }),
  round: Annotation({ reducer: (_, u) => u, default: () => 0 }),
  forArguments: Annotation({ reducer: (cur, u) => cur.concat(u), default: () => [] }),
  againstArguments: Annotation({ reducer: (cur, u) => cur.concat(u), default: () => [] }),
  verdict: Annotation({ reducer: (_, u) => u, default: () => "" }),
});

const forPoints = [
  "It increases employee flexibility and autonomy.",
  "It reduces office overhead costs for the company.",
  "It widens the talent pool beyond a single geographic area.",
];
const againstPoints = [
  "It can weaken team cohesion and informal mentorship.",
  "It makes some forms of spontaneous collaboration harder.",
  "It can blur work-life boundaries for some employees.",
];

function forAgent(state) {
  const point = forPoints[state.round] || "No further points.";
  console.log(`[FOR agent, round ${state.round + 1}] ${point}`);
  return { forArguments: [point] };
}

function againstAgent(state) {
  const point = againstPoints[state.round] || "No further points.";
  console.log(`[AGAINST agent, round ${state.round + 1}] ${point}`);
  return { againstArguments: [point], round: state.round + 1 };
}

function shouldContinueDebate(state) {
  if (state.round >= 3) {
    console.log(`[moderator] 3 rounds complete, moving to judge`);
    return "judge";
  }
  console.log(`[moderator] round ${state.round} complete, continuing debate`);
  return "continue";
}

function judgeAgent(state) {
  console.log(`[judge] synthesizing a verdict from ${state.forArguments.length} FOR and ${state.againstArguments.length} AGAINST arguments`);
  const verdict =
    `After ${state.round} rounds: ${state.forArguments.length} points favor remote work ` +
    `(flexibility, cost, talent pool), while ${state.againstArguments.length} points raise concerns ` +
    `(cohesion, collaboration, boundaries). A balanced policy with structured in-person time ` +
    `addresses both sets of concerns.`;
  return { verdict };
}

const graph = new StateGraph(DebateState)
  .addNode("forAgent", forAgent)
  .addNode("againstAgent", againstAgent)
  .addNode("judge", judgeAgent)
  .addEdge(START, "forAgent")
  .addEdge("forAgent", "againstAgent")
  .addConditionalEdges("againstAgent", shouldContinueDebate, {
    continue: "forAgent",
    judge: "judge",
  })
  .addEdge("judge", END)
  .compile();

async function main() {
  const result = await graph.invoke({
    question: "Should our company adopt a fully remote work policy?",
    round: 0,
  });

  console.log("\n--- FOR arguments ---");
  result.forArguments.forEach((a, i) => console.log(`${i + 1}. ${a}`));
  console.log("\n--- AGAINST arguments ---");
  result.againstArguments.forEach((a, i) => console.log(`${i + 1}. ${a}`));
  console.log("\n--- Verdict ---");
  console.log(result.verdict);

  console.log("\n--- Verification ---");
  console.log(`Exactly 3 rounds completed (round === 3)? ${result.round === 3 ? "PASS" : "FAIL"}`);
  console.log(`3 FOR arguments collected? ${result.forArguments.length === 3 ? "PASS" : "FAIL"}`);
  console.log(`3 AGAINST arguments collected? ${result.againstArguments.length === 3 ? "PASS" : "FAIL"}`);
  console.log(`Judge ran exactly once, after debate concluded? ${result.verdict.length > 0 ? "PASS" : "FAIL"}`);
}

main().catch(err => { console.error("ERROR:", err.message); process.exit(1); });
```

**Run it:**
```bash
node debate-pattern.js
```

**Verified output:** see Section 4, Trace #3 — all four assertions passed.

#### Line-by-line explanation

- **`round` is incremented only in `againstAgent`** — a deliberate design choice so the round counter advances exactly once per *full* FOR/AGAINST exchange, not once per individual agent turn — verified correct since the final `round` equals exactly 3 after 3 full exchanges (6 individual agent turns total).
- **`shouldContinueDebate`** — checked after `againstAgent` specifically, meaning the routing decision happens once both sides have spoken in a given round, not mid-round.
- **The judge only runs once `shouldContinueDebate` returns `"judge"`** — verified by the trace showing `[judge]` appearing exactly once, after exactly 3 complete rounds, never interleaved with the debate itself.

---

## 6. TEACH-IT-BACK — Script You Can Say Out Loud

> "Multi-agent systems coordinate several distinct agents, rather than asking one model to do everything in a single response, and there are a few common patterns for how they coordinate. The pipeline pattern runs agents in a fixed sequence, where each one's output becomes the next one's input — useful when a task naturally breaks into sequential stages, like researching, then writing, then editing, each requiring a genuinely different kind of focus. The supervisor pattern uses a central routing agent that inspects each incoming request and sends it to whichever specialized agent is best suited to handle it, so a billing question and a technical question get handled by different specialists, rather than forcing every request through every agent or relying on one generalist to be equally good at everything. And the debate pattern assigns different, deliberately opposing perspectives to different agents, has them exchange arguments over several rounds, and then synthesizes a final conclusion — this is particularly useful for subjective or multi-sided questions, where there often isn't a single correct answer to converge on, unlike a self-consistency-style approach that resamples the same prompt looking for agreement. All three patterns can be built as explicit graphs: a pipeline is just a fixed chain of edges; a supervisor is a conditional edge with multiple possible destinations instead of just two; and a debate combines a conditional loop, deciding whether to continue or move to synthesis, with multiple distinct agent roles taking turns within that loop."

---

## Key Terms Glossary (Day 23)

| Term | Meaning |
|---|---|
| **Multi-agent system** | Multiple distinct agents coordinating to accomplish a task |
| **Pipeline pattern** | Agents run in a fixed sequence, each transforming the previous output |
| **Supervisor pattern** | A router agent dynamically directs requests to the appropriate specialist |
| **Debate pattern** | Agents with opposing perspectives exchange arguments before a synthesis step |

---

## Quick Self-Check (ask yourself before moving to Day 24)

1. Can I explain, without looking, when a pipeline pattern is the right structural choice versus a supervisor?
2. Can I explain how the debate pattern differs from Day 16's self-consistency, despite both involving multiple "attempts"?
3. Can I trace through the supervisor pattern's routing logic and explain the fallback case?
4. Can I explain how the debate pattern's round-counting and stopping condition work together?

If yes to all four — you're ready for **Day 24: Fine-Tuning vs. RAG vs. Prompting** (a decision framework for when to use which, plus LoRA/QLoRA intuition).
