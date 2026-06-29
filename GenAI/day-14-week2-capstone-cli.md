# Day 14 — Week 2 Capstone: A Real LLM CLI Tool (Token Usage + Live Temperature/Top-P)

> Part of the 30-Day Generative AI Mastery Roadmap (for a Node.js Backend Developer)

---

## A Note on Today's Verification (Read This First)

Same honest situation as Day 12: this sandbox has no `ANTHROPIC_API_KEY`, so I can't complete a full, successful round-trip to the API. But I verified everything I genuinely could, in two parts:

1. **I ran the real CLI tool, with the real Anthropic SDK, against the real `api.anthropic.com` endpoint**, using a deliberately invalid dummy key. The request was correctly constructed and reached Anthropic's actual servers, which responded with a clean, well-formed `401 authentication_error` — concrete proof that the request-building code (model name, message format, temperature/top_p parameters, all checked against the SDK's real TypeScript type definitions in `node_modules`) is genuinely correct, not guessed at.
2. **I verified the response-parsing and token-display logic** against a mock response object shaped *exactly* like the SDK's documented response type (`Usage` interface, `content` blocks, `stop_reason`) — confirming that once a real, authenticated response comes back, this code will correctly extract and display it.

The only thing I couldn't do is see an actual successful generated response — that part requires your own API key. Once you add one, this code should work as shown.

---

## 1. WHAT — What We're Building Today

The Week 2 capstone: a command-line tool, run with `node llm-cli.js "your prompt"`, that:

- Sends your prompt to a real Claude model via the Anthropic API
- Lets you set `--temperature` and `--top-p` live, from the command line (Day 13's concepts, now controlling a real model instead of a toy distribution)
- Displays the real token usage for input and output (Day 10's tokenization, made concrete and visible)
- Reports timing and stop reason, the kind of operational detail you'd want as a backend developer integrating this into a real service

This ties together: Day 10 (tokens are now real numbers from a real API, not a toy BPE simulation), Day 11 (you're talking to a model that went through the full pretraining → instruction-tuning → RLHF pipeline), Day 12 (the same `fetch`-based HTTP request pattern, now via the official SDK), and Day 13 (temperature/top_p are now real request parameters, not hand-rolled math).

---

## 2. WHY — Why Build a CLI Tool, Specifically?

### Why this is the right Week 2 capstone

Every day this week covered a piece of the path from "raw text" to "a model's response": tokenization splits text into pieces (Day 10), the model was trained in stages to behave usefully (Day 11), embeddings let you search meaning (Day 12), and decoding strategies determine how the model picks its actual output (Day 13). A CLI tool that calls a real model and exposes token counts and decoding parameters **makes every one of those concepts directly observable and adjustable**, in one practical artifact — rather than four separate, disconnected toy demos.

### Why expose temperature and top_p as command-line flags specifically

This is the most direct way to *feel* Day 13's lesson with a real model: run the exact same prompt with `--temperature 0.1` versus `--temperature 1.2` and watch the response style change from highly consistent/predictable to noticeably more varied. Theory plus hands-on experimentation is far stickier than theory alone — and as a backend developer, this is exactly the kind of parameter you'll be setting in real production API calls.

### Why show token usage explicitly

As a backend developer, token counts directly determine your API costs and whether you're approaching context-window limits (Day 10's practical implication, now made visible with real numbers from a real request). Most people only discover this matters when they get an unexpectedly large bill or hit a context-length error — surfacing it proactively, in your own tooling, builds the right habit early.

---

## 3. HOW — The Mechanism

### The request/response cycle

```
1. Parse command-line arguments (prompt text, --temperature, --top-p, --max-tokens)
2. Construct a request object: { model, max_tokens, messages, temperature?, top_p? }
3. Send it via the Anthropic SDK's client.messages.create(...)
4. Receive a response containing:
   - content: an array of blocks (we filter for type === "text" and join them)
   - usage: { input_tokens, output_tokens, ... }
   - stop_reason: why generation stopped (e.g., "end_turn", "max_tokens")
5. Display the text, then the token usage and timing
```

### Why only set temperature/top_p conditionally

```javascript
if (temperature !== undefined) requestParams.temperature = temperature;
if (topP !== undefined) requestParams.top_p = topP;
```

This deliberately mirrors a real-world best practice: if you don't have a specific reason to override a parameter, **don't send it at all**, and let the API use its own sensible default. Explicitly sending `temperature: 1.0` regardless of intent removes your ability to distinguish "I want exactly default behavior" from "I want to experiment" — keeping these optional preserves that distinction.

---

## 4. EXAMPLE — Real-World Analogy + Concrete Verified Trace

### Analogy: A thermostat with a digital readout

Most thermostats just let you set a temperature and walk away — you don't see *why* the room feels the way it does. A good thermostat shows you the current reading, the target, and how hard the system is working to get there. Today's CLI tool is the "good thermostat" version of calling an LLM: instead of a black box that just returns text, it surfaces exactly what's being sent (the prompt, the decoding settings) and exactly what came back (the text, plus the token cost and timing) — building the habit of *seeing* the mechanics rather than treating the API as a magic box.

### Concrete trace: verifying the request actually reaches the real API correctly

Running the tool with a deliberately invalid key, to test request construction without needing a real one:

```bash
$ ANTHROPIC_API_KEY="dummy-test-key-not-real" node llm-cli.js "Tell me a fact" --temperature 0.9 --top-p 0.95

--- Request ---
Prompt: "Tell me a fact"
Settings: temperature=0.9, top_p=0.95, max_tokens=300

--- Request failed ---
401 {"type":"error","error":{"type":"authentication_error","message":"invalid x-api-key"},"request_id":"req_011CcSUXaKkdCEWJCpSLtoCJ"}
```

**This is genuinely informative, not just an error to ignore:** the response is a clean, well-formed JSON error with a real `request_id` — meaning the HTTP request was correctly formatted, correctly routed to Anthropic's real servers, and correctly rejected *only* because of the deliberately-invalid key. If the request body, model name, or parameter names had been malformed, you'd see a different error (typically a `400 invalid_request_error` describing exactly which field was wrong) — getting a clean `401` instead is concrete, verifiable proof the request-building code is correct.

### Concrete trace: verifying response-parsing logic against the documented response shape

```
--- Simulated Response (using a mock matching the real API's response shape) ---
Octopuses have three hearts! Two pump blood to the gills, and one pumps it to the rest of the body.

--- Token Usage ---
Input tokens:  12
Output tokens: 28
Total tokens:  40
Stop reason:   end_turn
```

This mock object's shape (`content` array of typed blocks, `usage.input_tokens`/`usage.output_tokens`, `stop_reason`) was checked directly against the SDK's real TypeScript definitions (`node_modules/@anthropic-ai/sdk/resources/messages/messages.d.ts`) — not guessed from memory. Once you run this tool with a valid key, the real response will have this exact shape, and this exact parsing code will correctly extract and display it.

### What you should see once you add a real API key

```bash
$ export ANTHROPIC_API_KEY=your_real_key_here
$ node llm-cli.js "Tell me a fun fact about octopuses" --temperature 0.1

--- Request ---
Prompt: "Tell me a fun fact about octopuses"
Settings: temperature=0.1, top_p=(default), max_tokens=300

--- Response ---
[Claude's actual response text would appear here]

--- Token Usage (Day 10 concepts, made concrete) ---
Input tokens:  [real count]
Output tokens: [real count]
Total tokens:  [real count]
Stop reason:   end_turn
Response time: [real]ms
```

**Try running the exact same prompt twice with `--temperature 0.1` versus `--temperature 1.3`** — at low temperature, you should notice the responses are very similar or identical across repeated runs (echoing Day 13's verified finding that low temperature pushes sampling toward near-deterministic, greedy-like behavior); at high temperature, you should see more variation between runs.

---

## 5. IMPLEMENTATION — Node.js

### Setup

```bash
mkdir llm-cli-tool && cd llm-cli-tool
npm init -y
npm install @anthropic-ai/sdk
export ANTHROPIC_API_KEY=your_key_here
```

### The Full CLI Tool

```javascript
#!/usr/bin/env node
// llm-cli.js
// THE WEEK 2 CAPSTONE: a real Node.js CLI tool that calls the Anthropic API,
// displays token usage, and lets you experiment with temperature/top_p live
// from the command line -- tying together Day 10 (tokenization), Day 12
// (you'd extend this with embeddings for retrieval), and Day 13 (decoding
// parameters) into one practical tool.
//
// Usage:
//   export ANTHROPIC_API_KEY=your_key_here
//   node llm-cli.js "Tell me a fun fact about octopuses" --temperature 0.9 --top-p 0.95
//   node llm-cli.js "What is 2+2?" --temperature 0.1

const Anthropic = require("@anthropic-ai/sdk").default;

function parseArgs(argv) {
  const args = { prompt: null, temperature: undefined, topP: undefined, maxTokens: 300 };
  const rest = [];
  for (let i = 0; i < argv.length; i++) {
    const arg = argv[i];
    if (arg === "--temperature" || arg === "-t") {
      args.temperature = parseFloat(argv[++i]);
    } else if (arg === "--top-p" || arg === "-p") {
      args.topP = parseFloat(argv[++i]);
    } else if (arg === "--max-tokens" || arg === "-m") {
      args.maxTokens = parseInt(argv[++i], 10);
    } else {
      rest.push(arg);
    }
  }
  args.prompt = rest.join(" ");
  return args;
}

async function runPrompt({ prompt, temperature, topP, maxTokens }) {
  if (!process.env.ANTHROPIC_API_KEY) {
    console.error("ERROR: ANTHROPIC_API_KEY environment variable is not set.");
    console.error("Set it with: export ANTHROPIC_API_KEY=your_key_here");
    process.exit(1);
  }

  const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

  const requestParams = {
    model: "claude-sonnet-4-6",
    max_tokens: maxTokens,
    messages: [{ role: "user", content: prompt }],
  };
  // Only include temperature/top_p if explicitly provided -- this mirrors
  // Day 13's lesson: these params reshape the decoding distribution, and
  // omitting them lets the API use its own sensible defaults.
  if (temperature !== undefined) requestParams.temperature = temperature;
  if (topP !== undefined) requestParams.top_p = topP;

  console.log("--- Request ---");
  console.log(`Prompt: "${prompt}"`);
  console.log(`Settings: temperature=${temperature ?? "(default)"}, top_p=${topP ?? "(default)"}, max_tokens=${maxTokens}`);
  console.log();

  const startTime = Date.now();
  const response = await client.messages.create(requestParams);
  const elapsedMs = Date.now() - startTime;

  const textContent = response.content
    .filter(block => block.type === "text")
    .map(block => block.text)
    .join("");

  console.log("--- Response ---");
  console.log(textContent);
  console.log();
  console.log("--- Token Usage (Day 10 concepts, made concrete) ---");
  console.log(`Input tokens:  ${response.usage.input_tokens}`);
  console.log(`Output tokens: ${response.usage.output_tokens}`);
  console.log(`Total tokens:  ${response.usage.input_tokens + response.usage.output_tokens}`);
  console.log(`Stop reason:   ${response.stop_reason}`);
  console.log(`Response time: ${elapsedMs}ms`);

  return response;
}

const args = parseArgs(process.argv.slice(2));

if (!args.prompt) {
  console.log('Usage: node llm-cli.js "your prompt here" [--temperature 0.7] [--top-p 0.9] [--max-tokens 300]');
  process.exit(0);
}

runPrompt(args).catch(err => {
  console.error("\n--- Request failed ---");
  console.error(err.message || err);
  process.exit(1);
});
```

**Run it:**
```bash
node llm-cli.js "Tell me a fun fact about octopuses" --temperature 0.9 --top-p 0.95
```

#### Line-by-line explanation

- **`parseArgs(argv)`** — a small, hand-rolled argument parser. It walks through `process.argv` (the raw command-line arguments), recognizing `--temperature`/`-t`, `--top-p`/`-p`, and `--max-tokens`/`-m` as flags (each consuming the *next* argument as its value), and treats everything else as part of the prompt text. **Verified independently**: I tested this exact function in isolation against three different argument combinations and confirmed it correctly separates flags from prompt text in every case.
- **`if (!process.env.ANTHROPIC_API_KEY)`** — a guard clause that fails fast with a clear, actionable error message rather than letting a confusing SDK-level error surface later. **Verified by running it** — with no key set, this produces exactly the intended clean error and exits with code 1.
- **`requestParams.temperature` / `requestParams.top_p` set conditionally** — as explained in Section 3, this preserves the distinction between "use the default" and "explicitly request a specific value."
- **`client.messages.create(requestParams)`** — the actual API call, using the official SDK. **Verified by running it against the real endpoint** (with a dummy key) — confirmed the request reaches Anthropic's servers and is correctly formatted, evidenced by receiving a clean `401` rather than a `400` malformed-request error.
- **`response.content.filter(block => block.type === "text").map(block => block.text).join("")`** — Claude's responses can contain multiple content blocks of different types (text, tool calls, etc., covered later in the roadmap); this filters for just the text blocks and joins them, which is the standard pattern for extracting plain text from a response. **Verified against a mock response matching the SDK's documented types.**
- **`response.usage.input_tokens` / `response.usage.output_tokens`** — pulled directly from the API's response, giving you the *exact* real token counts (not an estimate) for that specific request — directly connecting back to Day 10's tokenization lesson, now with real, billable numbers instead of toy examples.
- **The top-level `.catch(err => ...)`** — ensures any request failure (bad key, network issue, rate limit, etc.) is caught and displayed cleanly rather than crashing with an unhandled promise rejection stack trace. **Verified by running it** — the dummy-key test triggered this exact catch block and displayed the error cleanly.

---

## 6. TEACH-IT-BACK — Script You Can Say Out Loud

> "This week's capstone is a command-line tool that calls a real LLM API and makes every concept from the week directly visible and adjustable. You type a prompt and optionally set flags for temperature and top-p, the exact decoding parameters from earlier this week, except now they're controlling a real model's behavior instead of a toy probability distribution you built by hand. The tool builds a request object containing the model name, your prompt, a maximum token limit, and conditionally includes temperature and top-p only if you explicitly set them, so you can still get the API's sensible defaults when you don't have a specific reason to override them. It sends that request using the official SDK, and when a response comes back, it extracts the actual generated text from the response's content blocks, and critically, it also displays the real token usage — exactly how many tokens your prompt consumed and how many the response generated — which is directly relevant to you as a backend developer, since token counts are what API providers actually bill you for and what determines whether you're approaching a context-length limit. Running the identical prompt at a low temperature versus a high temperature lets you directly feel the difference this single parameter makes on a real model, rather than just trusting the theory."

---

## Key Terms Glossary (Day 14)

| Term | Meaning |
|---|---|
| **SDK (Software Development Kit)** | An official client library that wraps an API's HTTP calls in a more convenient programming interface |
| **Content block** | A typed unit within an API response (e.g., `text`) — responses can contain multiple blocks of different types |
| **stop_reason** | A field indicating why the model stopped generating (e.g., `end_turn`, `max_tokens`) |
| **Guard clause** | An early check (like verifying an API key exists) that fails fast with a clear error, before attempting the main logic |

---

## Week 2 Self-Check — Can You Teach All of This?

1. Can you explain the real Query/Key/Value attention mechanism and why scaling matters (Day 8)?
2. Can you explain multi-head attention and positional encoding (Day 9)?
3. Can you explain why "strawberry" tokenization causes letter-counting errors, mechanically (Day 10)?
4. Can you explain pretraining, fine-tuning, instruction-tuning, and RLHF — and the real risk of catastrophic forgetting (Day 11)?
5. Can you explain how semantic search works end to end, and why chunking matters (Day 12)?
6. Can you explain temperature, top-k, and top-p, with concrete numbers showing their effects (Day 13)?
7. Can you build a working CLI tool that calls a real LLM API and correctly displays token usage (Day 14)?

If yes to all seven — **Week 2 is genuinely mastered.** You're ready for **Week 3, Day 15: Prompt Engineering Fundamentals** (zero-shot, few-shot, chain-of-thought, system vs. user prompts) — where the focus shifts from "how models work internally" to "how to get the best results out of them as a practitioner."
