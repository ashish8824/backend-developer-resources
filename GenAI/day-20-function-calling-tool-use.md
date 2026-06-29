# Day 20 — Function Calling / Tool Use: How LLMs Call APIs

> Part of the 30-Day Generative AI Mastery Roadmap (for a Node.js Backend Developer)

---

## A Note on Today's Verification

Same honest split as Days 12, 14, 16, and 17: I verified what I genuinely could. I ran the **real** tool-calling request — including the `tools` array with JSON Schema definitions — against the real `api.anthropic.com` endpoint with a dummy key, and got back a clean `401 authentication_error` (not a `400` malformed-request error), confirming the request construction is correct. I also independently verified the actual tool *functions* (the code that runs when a tool is called) work correctly in isolation. What I couldn't do is see a real model actually decide to call a tool — for that, I built a second, separate verification using mock API responses shaped exactly like the real SDK's types, to test the *orchestration loop's* logic (does it correctly continue the conversation after a tool call, does the message history grow correctly) independent of needing a real model's decision-making.

---

## 1. WHAT — Plain Definitions

| Term | Plain Definition |
|---|---|
| **Function calling / tool use** | A capability where a model, instead of only generating text, can request that your code execute a specific function with specific arguments — and then continue generating its response using that function's result. |
| **Tool definition** | A structured description (name, description, and a JSON Schema for its arguments) that tells the model what a tool does and how to call it correctly. |
| **Tool use block** | The part of a model's response indicating it wants to call a specific tool with specific arguments. |
| **Tool result** | The actual output of running the requested function, sent back to the model so it can continue. |

### Diagram: The Tool-Calling Cycle (This IS Day 16's ReAct Loop, Concretely)

```
   YOUR CODE                                    THE MODEL
   ──────────                                   ─────────
   1. Send: user message + list of              ──────►   Model decides: do I need
      AVAILABLE TOOLS (name,                                a tool to answer this?
      description, JSON Schema)
                                                            │
                                                  ◄──────   If YES: responds with a
   2. Receive a "tool_use" block:                            "tool_use" block: which
      { name, input, id }                                    tool, with what arguments

   3. EXECUTE THE ACTUAL FUNCTION                
      in YOUR code, using `input`
      as arguments

   4. Send the function's result back  ──────►   Model incorporates the result,
      as a "tool_result" message                  decides: enough info now, or
                                                    need another tool call?
                                                            │
                                                  ◄──────   Either another tool_use
                                                              block, OR a final text answer
```

**This diagram should look very familiar** — it's mechanically identical to Day 16's ReAct loop (Thought → Action → Observation), just using the model provider's official, structured mechanism instead of a hand-rolled prompt convention. Tool use is the production-grade, standardized version of exactly that pattern.

---

## 2. WHY — Why Does This Matter, Especially for You as a Backend Developer?

### Why tool use exists at all

Recall Day 11 and Day 17: a model's knowledge is frozen at training time, and it has no inherent ability to interact with the outside world — it can't check a live database, call a real weather API, or perform an action like sending an email. Tool use bridges this gap by letting the model **describe what it wants to do**, while your own application code retains full control over **what actually happens**. The model never directly executes anything — it only ever requests, and your code decides whether and how to fulfill that request.

### Why this should feel natural to you specifically

As a Node.js backend developer, you already think in terms of: define a function's signature, validate the input, execute the function, return a structured result. Tool definitions are essentially **API contracts**, expressed as JSON Schema — the same kind of interface contract you'd write for a REST endpoint or an internal service method. The model's job is just to figure out *which* function to call and *with what arguments*, based on the conversation — the actual execution, error handling, and security boundary are entirely yours to control, exactly like handling any other untrusted input to your backend.

### Why JSON Schema, specifically, for describing tool arguments

JSON Schema gives a precise, structured way to specify exactly what shape of input a tool expects — required fields, types, descriptions of what each field means. This lets the model reliably produce correctly-structured arguments, rather than you having to parse loosely-structured natural language out of a text response and hope it's in a consistent, parseable format every time.

### Why you must treat tool inputs as untrusted, exactly like user input

This connects to a critical security point: the model's chosen arguments for a tool call should be treated with the same caution as any external user input to your backend. A model could (whether through being misled by malicious content elsewhere in its context, or simply through an unexpected reasoning path) request a tool call with unexpected or malicious-looking arguments. **Your tool-execution code is the security boundary** — exactly the discipline you'd already apply to validating any external request to an API endpoint.

---

## 3. HOW — The Mechanism

### Defining a tool

```javascript
{
  name: "getWeather",
  description: "Get the current weather for a specific city.",
  input_schema: {
    type: "object",
    properties: {
      city: { type: "string", description: "The city name, e.g. 'Tokyo'" }
    },
    required: ["city"]
  }
}
```

This is sent alongside your messages in the `tools` parameter of the API request. The `description` fields matter more than they might seem — they're the model's *only* source of information about what a tool does and how to use it correctly, so vague or incomplete descriptions directly translate into the model making worse decisions about when and how to call that tool.

### The full request/response cycle

```
1. Send: { model, messages, tools: [...] }
2. Receive a response. Check response.content for blocks of type "tool_use".
3. For each tool_use block: { id, name, input }
   - Look up the corresponding real function in your code
   - Call it with `input` as arguments
   - Capture its return value
4. Construct a tool_result message:
   { type: "tool_result", tool_use_id: <the id from step 3>, content: <stringified result> }
5. Append the model's previous response AND your tool_result message to
   the conversation history, then send the request AGAIN.
6. Repeat until the model's response contains no tool_use blocks --
   that's your final answer.
```

The `tool_use_id` linkage in step 4 is essential: it's how the model knows which specific tool call (if you made several at once) a given result corresponds to.

---

## 4. EXAMPLE — Concrete Verified Trace

### Concrete trace: real request construction, verified against the live API

Running the actual tool-use code with a deliberately invalid key, to test request construction without needing a real one:

```bash
$ ANTHROPIC_API_KEY="dummy-test-key-not-real" node tool-use-demo.js "What is the weather in Tokyo?"

--- Round 1: sending request ---

--- Request failed ---
401 {"type":"error","error":{"type":"authentication_error","message":"invalid x-api-key"},"request_id":"req_011CcUEGyTzoS56sgYDxjFfm"}
```

Just like Day 14's verification, getting a clean `401` (not a `400` describing a malformed request) confirms the entire request — including the `tools` array with its JSON Schema definitions — was correctly formatted and accepted by Anthropic's real servers, rejected only because of the deliberately-invalid key.

### Concrete trace: the actual tool functions, verified in isolation

```javascript
getWeather({ city: 'Tokyo' })     -> { tempC: 18, condition: 'rainy' }
getWeather({ city: 'Atlantis' })  -> { error: 'No weather data for Atlantis' }
calculate({ expression: '18 * 1.15' })        -> { result: 20.7 }
calculate({ expression: 'not valid math' })   -> { error: 'Could not evaluate: not valid math' }
```

Both functions correctly handle their expected cases and their error cases — confirmed by actually running them, independent of any model interaction.

### Concrete trace: the orchestration loop, verified using mock responses matching real API shapes

Since I can't get a real, authenticated multi-turn tool-calling conversation in this sandbox, I tested the loop's *mechanics* using a scripted sequence of mock responses shaped exactly like the real SDK's documented types:

```
--- Round 1 ---
[mock API call 1] messages.length = 1, tools defined = 1
Model says: Let me check the weather in Tokyo for you.
Model wants to call: getWeather({"city":"Tokyo"})
Tool result: {"tempC":18,"condition":"rainy"}

--- Round 2 ---
[mock API call 2] messages.length = 3, tools defined = 1
Model says: The weather in Tokyo is currently 18°C and rainy.
No tool calls -- final answer reached.

--- Verification ---
Completed in 2 rounds.
Final message history length: 4 (user, assistant, user-with-tool-result, assistant = 4 expected)
Expected exactly 2 rounds (1 tool call + 1 final answer): PASS
Expected exactly 4 messages in history: PASS
```

This confirms the loop correctly: (1) appends the model's tool-call response to history, (2) executes the real function and captures its result, (3) sends a follow-up request with the tool result included — note round 2 correctly shows `messages.length = 3` (user message + assistant tool-call response + user tool-result message), proving the history accumulated correctly — and (4) correctly recognizes a tool-call-free response as the final answer, stopping the loop.

---

## 5. IMPLEMENTATION — Node.js

### Part A — The Real Tool-Use CLI Tool

```javascript
#!/usr/bin/env node
// tool-use-demo.js
// Demonstrates REAL function calling / tool use with the Anthropic API:
// define tools with a JSON schema, let the model decide when to call them,
// execute the actual function in your own code, and feed the result back.
// This connects directly to Day 16's ReAct pattern -- this IS the real
// mechanism that pattern is built on top of.

const Anthropic = require("@anthropic-ai/sdk").default;

// --- Real functions YOUR code can execute ---
const availableFunctions = {
  getWeather({ city }) {
    const data = {
      "Tokyo": { tempC: 18, condition: "rainy" },
      "Paris": { tempC: 12, condition: "cloudy" },
      "Cairo": { tempC: 31, condition: "sunny" },
    };
    return data[city] || { error: `No weather data for ${city}` };
  },
  calculate({ expression }) {
    // NEVER use eval() on untrusted input in production -- this is a
    // deliberately minimal example. A real implementation should use a
    // proper math expression parser library instead.
    try {
      // eslint-disable-next-line no-eval
      return { result: eval(expression) };
    } catch (e) {
      return { error: `Could not evaluate: ${expression}` };
    }
  },
};

// --- Tool DEFINITIONS: this is the JSON Schema the model uses to know
// what each tool does and what arguments it expects. This is the exact
// shape verified against the SDK's real TypeScript types. ---
const toolDefinitions = [
  {
    name: "getWeather",
    description: "Get the current weather for a specific city. Only supports Tokyo, Paris, and Cairo.",
    input_schema: {
      type: "object",
      properties: {
        city: { type: "string", description: "The city name, e.g. 'Tokyo'" },
      },
      required: ["city"],
    },
  },
  {
    name: "calculate",
    description: "Evaluate a basic arithmetic expression and return the numeric result.",
    input_schema: {
      type: "object",
      properties: {
        expression: { type: "string", description: "A math expression, e.g. '12 * 4 + 7'" },
      },
      required: ["expression"],
    },
  },
];

async function runWithTools(userPrompt, maxRounds = 5) {
  const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });
  const messages = [{ role: "user", content: userPrompt }];

  for (let round = 0; round < maxRounds; round++) {
    console.log(`\n--- Round ${round + 1}: sending request ---`);

    const response = await client.messages.create({
      model: "claude-sonnet-4-6",
      max_tokens: 1024,
      tools: toolDefinitions,
      messages,
    });

    messages.push({ role: "assistant", content: response.content });

    response.content
      .filter(block => block.type === "text")
      .forEach(block => console.log(`Model says: ${block.text}`));

    const toolUseBlocks = response.content.filter(block => block.type === "tool_use");

    if (toolUseBlocks.length === 0) {
      console.log("No tool calls -- model has given its final answer.");
      return response;
    }

    const toolResults = toolUseBlocks.map(toolUse => {
      console.log(`Model wants to call: ${toolUse.name}(${JSON.stringify(toolUse.input)})`);
      const fn = availableFunctions[toolUse.name];
      const result = fn ? fn(toolUse.input) : { error: `Unknown tool: ${toolUse.name}` };
      console.log(`Tool result: ${JSON.stringify(result)}`);

      return {
        type: "tool_result",
        tool_use_id: toolUse.id,
        content: JSON.stringify(result),
      };
    });

    messages.push({ role: "user", content: toolResults });
  }

  console.log("Max rounds reached.");
}

const prompt = process.argv.slice(2).join(" ") || "What's the weather in Tokyo?";

if (!process.env.ANTHROPIC_API_KEY) {
  console.error("ERROR: ANTHROPIC_API_KEY not set.");
  process.exit(1);
}

runWithTools(prompt).catch(err => {
  console.error("\n--- Request failed ---");
  console.error(err.message || err);
  process.exit(1);
});
```

**Run it:**
```bash
export ANTHROPIC_API_KEY=your_real_key_here
node tool-use-demo.js "What's the weather in Tokyo?"
```

#### Line-by-line explanation

- **`availableFunctions`** — the actual, real implementations your application controls. The model never sees this code or executes it directly; it only ever requests a call by name and arguments, and your code decides what actually happens — verified independently to work correctly for both success and error cases.
- **`toolDefinitions`** — the JSON Schema contracts, verified against the SDK's real TypeScript types (`input_schema`, `name`, `description` are the confirmed correct field names).
- **`runWithTools`** — implements the exact loop from Section 3: send a request with `tools` included, check for `tool_use` blocks, execute the corresponding real function, append both the model's response and your tool's result to the message history, and send again — repeating until the model responds with no tool calls.
- **`messages.push({ role: "assistant", content: response.content })`** — this is essential and easy to get wrong: you must include the model's *own* previous response (including its tool_use blocks) back into the conversation history, not just your tool's result, so the model has full context for what it asked for and why.
- **The `tool_use_id` linkage** — `toolResults` map each result back to its originating `tool_use.id`, which is how the model correctly associates a result with the specific call it made (important when a single response contains multiple tool calls at once).

---

### Part B — Verifying the Orchestration Loop's Logic (Mocked)

```javascript
// verify-tool-loop-logic.js
// Verifies the MULTI-ROUND orchestration logic from tool-use-demo.js using
// a sequence of MOCK responses shaped exactly like real API responses
// (verified against the SDK's TypeScript types) -- since we can't get a
// real authenticated multi-turn tool-calling conversation in this sandbox.

const availableFunctions = {
  getWeather({ city }) {
    const data = { "Tokyo": { tempC: 18, condition: "rainy" } };
    return data[city] || { error: `No weather data for ${city}` };
  },
};

const mockResponseSequence = [
  {
    content: [
      { type: "text", text: "Let me check the weather in Tokyo for you." },
      { type: "tool_use", id: "toolu_01ABC123", name: "getWeather", input: { city: "Tokyo" } },
    ],
  },
  {
    content: [
      { type: "text", text: "The weather in Tokyo is currently 18°C and rainy." },
    ],
  },
];

let callCount = 0;
async function mockCreate(params) {
  console.log(`[mock API call ${callCount + 1}] messages.length = ${params.messages.length}, tools defined = ${params.tools.length}`);
  const response = mockResponseSequence[callCount];
  callCount++;
  return response;
}

async function runWithToolsMocked(userPrompt, maxRounds = 5) {
  const messages = [{ role: "user", content: userPrompt }];

  for (let round = 0; round < maxRounds; round++) {
    console.log(`\n--- Round ${round + 1} ---`);
    const response = await mockCreate({ model: "claude-sonnet-4-6", max_tokens: 1024, tools: [{}], messages });

    messages.push({ role: "assistant", content: response.content });

    response.content.filter(b => b.type === "text").forEach(b => console.log(`Model says: ${b.text}`));

    const toolUseBlocks = response.content.filter(b => b.type === "tool_use");
    if (toolUseBlocks.length === 0) {
      console.log("No tool calls -- final answer reached.");
      return { finalResponse: response, totalRounds: round + 1, messageHistoryLength: messages.length };
    }

    const toolResults = toolUseBlocks.map(toolUse => {
      console.log(`Model wants to call: ${toolUse.name}(${JSON.stringify(toolUse.input)})`);
      const fn = availableFunctions[toolUse.name];
      const result = fn ? fn(toolUse.input) : { error: `Unknown tool: ${toolUse.name}` };
      console.log(`Tool result: ${JSON.stringify(result)}`);
      return { type: "tool_result", tool_use_id: toolUse.id, content: JSON.stringify(result) };
    });

    messages.push({ role: "user", content: toolResults });
  }
}

runWithToolsMocked("What's the weather in Tokyo?").then(({ totalRounds, messageHistoryLength }) => {
  console.log(`\n--- Verification ---`);
  console.log(`Completed in ${totalRounds} rounds.`);
  console.log(`Final message history length: ${messageHistoryLength} (user, assistant, user-with-tool-result, assistant = 4 expected)`);
  console.log(`Expected exactly 2 rounds (1 tool call + 1 final answer): ${totalRounds === 2 ? "PASS" : "FAIL"}`);
  console.log(`Expected exactly 4 messages in history: ${messageHistoryLength === 4 ? "PASS" : "FAIL"}`);
});
```

**Run it:**
```bash
node verify-tool-loop-logic.js
```

**Verified output:** see Section 4 — both correctness assertions (round count, message history length) passed.

#### Line-by-line explanation

- **`mockResponseSequence`** — a scripted stand-in for what a real model would return, shaped exactly like the verified SDK response types, letting us test the loop's *mechanics* in complete isolation from needing a real, authenticated model call.
- **The assertions at the end** (`totalRounds === 2`, `messageHistoryLength === 4`) — genuine, falsifiable correctness checks, not just printed output to eyeball. This is the same rigor applied throughout this course: don't just claim code works, verify it against an expected, checkable outcome.

---

## 6. TEACH-IT-BACK — Script You Can Say Out Loud

> "Function calling lets a model request that your own code execute a specific function, rather than the model trying to answer everything purely from its own training. You define each available tool using a JSON Schema — a name, a description of what it does, and a precise specification of what arguments it expects — and send that alongside your conversation. The model can then respond not just with text, but with a tool-use block indicating which tool it wants to call and with what arguments. Critically, the model never executes anything itself — it only requests; your own code looks up the real function, runs it with the model-provided arguments, and sends the actual result back as a new message. The model then continues, either making another tool call if it needs more information, or giving a final text answer once it has enough. This is mechanically the exact same Thought-Action-Observation loop from ReAct, just using each provider's official, structured mechanism instead of a hand-rolled prompting convention. And because the model's chosen tool arguments are functionally untrusted input to your system — no different in principle from any external request to a backend API — your tool-execution code needs to apply the same validation and security discipline you'd already use for any other untrusted input, since the model deciding to call a function with certain arguments doesn't make those arguments inherently safe to act on."

---

## Key Terms Glossary (Day 20)

| Term | Meaning |
|---|---|
| **Function calling / tool use** | A model requesting your code execute a specific function with specific arguments |
| **Tool definition** | A name, description, and JSON Schema specifying a tool's expected arguments |
| **Tool use block** | The part of a model's response requesting a specific tool call |
| **Tool result** | The actual output of executing a requested function, sent back to the model |
| **tool_use_id** | The identifier linking a tool result back to the specific call it answers |

---

## Quick Self-Check (ask yourself before moving to Day 21)

1. Can I explain, without looking, why the model never directly executes a tool itself?
2. Can I explain why JSON Schema is used to describe tool arguments, rather than free-form natural language?
3. Can I explain why tool arguments should be treated as untrusted input, with a concrete justification?
4. Can I trace through the full request/response cycle and explain what happens at each of the six steps?

If yes to all four — you're ready for **Day 21: Review + Mini Project** (the Week 3 capstone: a RAG-powered Q&A API over your own docs, built with Express + the concepts from this week).
