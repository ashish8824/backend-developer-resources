# Day 15 — Prompt Engineering Fundamentals

> Part of the 30-Day Generative AI Mastery Roadmap (for a Node.js Backend Developer)

---

## A Note on Today's Shift

Weeks 1-2 were about *how models work internally* — architecture, training, tokenization, decoding. Starting today, Week 3 is about *how to use them well* as a practitioner. This is a real, important shift: prompt engineering isn't a separate "magic trick" layered on top of everything you've learned — every technique today is explainable using mechanisms you've already verified (attention, Day 6/8; training stages, Day 11; decoding, Day 13). I'll point back to those connections explicitly, rather than presenting prompting as a new, disconnected skill.

---

## 1. WHAT — Plain Definitions

| Term | Plain Definition |
|---|---|
| **Prompt** | The text you send to a model to get a response — includes your actual question/instruction, plus anything else you include alongside it. |
| **Zero-shot prompting** | Asking a model to do a task by just describing it, with no examples given. |
| **Few-shot prompting** | Giving the model a small number of worked examples of the task, directly in the prompt, before asking it to do the same thing on a new input. |
| **Chain-of-thought (CoT) prompting** | Explicitly asking the model to reason step by step before giving a final answer, rather than jumping straight to a conclusion. |
| **System prompt** | A special instruction, separate from the user's message, that sets the model's overall role, tone, or behavior for the whole conversation. |
| **User prompt** | The actual message/question from the person interacting with the model, in a given turn. |

### Diagram: Where Each Technique Lives in a Request

```
   API REQUEST
   ┌─────────────────────────────────────────────┐
   │  SYSTEM PROMPT (sets persistent behavior)     │
   │  "You are a sentiment classifier. Respond     │
   │   with exactly one word."                     │
   ├─────────────────────────────────────────────┤
   │  USER MESSAGE (the actual task/question)      │
   │                                                │
   │  [ZERO-SHOT]: just the task description       │
   │       -- OR --                                │
   │  [FEW-SHOT]: task + worked examples + new input│
   │       -- OR --                                │
   │  [CHAIN-OF-THOUGHT]: task + "think step        │
   │   by step" instruction                         │
   └─────────────────────────────────────────────┘
```

---

## 2. WHY — Why Do These Techniques Actually Work? (Grounded in What You Already Know)

### Why does few-shot prompting help, mechanically?

Here's the connection to Day 6 and Day 8 that most prompting guides skip: **when you put examples in your prompt, those examples become tokens in the same sequence as your actual question.** Recall self-attention's defining property, verified back on Day 6 and Day 8: every token in a sequence can directly attend to every other token, regardless of distance, weighted by relevance rather than position. This means when the model processes your new input (near the end of the prompt), it can directly attend back to your worked examples — including their outputs — and use that pattern to inform its own answer. **Few-shot prompting isn't a separate mechanism bolted onto the model; it's the exact same attention mechanism you already understand, just given more relevant context to attend to.**

### Why does zero-shot prompting work at all, without examples?

This connects to Day 11: a well-instruction-tuned model has already seen enormous numbers of "instruction → ideal response" pairs during training. When you give it a clear, zero-shot instruction, you're relying on it having generalized the *pattern* of "follow an instruction like this" from training, even though it's never seen your *specific* instruction before. Zero-shot prompting works best when the task is common/well-represented in typical training data (e.g., "summarize this," "translate this") and tends to work less reliably for unusual, highly specific, or idiosyncratic tasks the model likely saw little of during training — which is exactly when few-shot examples become more valuable, since they let you specify the pattern directly rather than relying on the model having already generalized it.

### Why does chain-of-thought prompting improve accuracy on complex tasks?

This connects to how the model generates text at all: token by token, left to right (recall the decoder architecture from Day 6, and decoding strategies from Day 13). If you ask a model to jump straight to a final answer on a multi-step problem, it has to get the *entire* reasoning chain right *implicitly*, in one shot, with no opportunity to "check its work" along the way. If you instead ask it to reason step by step first, **each intermediate reasoning token becomes part of the context that influences the next token** — meaning the model's own previous reasoning steps become inputs it can attend back to (again, Day 6/8's attention mechanism) when producing later steps and the final answer. This tends to make complex, multi-step problems more tractable, because the model is no longer required to silently compute the entire answer in a single forward pass with no intermediate "scratch space."

### Why separate system and user prompts at all?

This is a practical, structural distinction: the system prompt is typically meant to persist across an entire conversation (e.g., "always respond in JSON," "you are a customer support agent for Acme Corp"), while user messages change with every turn. Keeping them separate lets you set behavior *once* rather than repeating instructions in every single message, and it also gives the underlying model a clearer signal about which parts of the input are "configuration" versus "the actual request to handle right now."

---

## 3. HOW — The Mechanism

### Zero-shot: structure

```
[Task description]
[Input to process]
```

No examples — relies entirely on the model's generalized understanding from training (Day 11).

### Few-shot: structure

```
[Task description]

[Example 1 input] -> [Example 1 output]
[Example 2 input] -> [Example 2 output]
[Example 3 input] -> [Example 3 output]

[New input] ->
```

The model is implicitly being asked to infer the pattern from the examples and continue it — this is fundamentally a next-token-prediction task (Day 7) where the "next token" the model needs to predict is the output for your new input, and the preceding examples are powerful context for that prediction.

### Chain-of-thought: structure

```
[Task description]
[Input to process]

Let's think through this step by step before giving the final answer.
```

The explicit instruction nudges the model toward generating intermediate reasoning tokens before committing to a final answer token.

### System/user separation: structure

```json
{
  "system": "[Persistent instruction/role/behavior]",
  "messages": [
    { "role": "user", "content": "[This turn's specific request]" }
  ]
}
```

---

## 4. EXAMPLE — Concrete Verified Trace (Prompt Construction Logic)

### Concrete trace: the same task, built four different ways (verified by running the code)

Task: classify the sentiment of a product review. New input: *"The packaging was damaged but the product itself works great."* — deliberately chosen because it's genuinely ambiguous (negative packaging experience, positive product experience), making it a good test case for seeing how different prompting strategies might help a model reason about a non-trivial case.

**Zero-shot output:**
```
Classify the sentiment of this product review as positive, negative, or neutral.

The packaging was damaged but the product itself works great.
```

**Few-shot output:**
```
Classify the sentiment of this product review as positive, negative, or neutral.

Here are some examples:

Input: This phone case is amazing, fits perfectly!
Output: positive

Input: Arrived broken and customer service never responded.
Output: negative

Input: It's fine, does what it says, nothing special.
Output: neutral

Now do the same for:
Input: The packaging was damaged but the product itself works great.
Output:
```

**Chain-of-thought output:**
```
Classify the sentiment of this product review as positive, negative, or neutral.

The packaging was damaged but the product itself works great.

Let's think through this step by step before giving the final answer.
```

**System/user structured output (verified JSON shape, matching how a real API call is actually sent):**
```json
{
  "system": "You are a sentiment classification assistant. Always respond with exactly one word: positive, negative, or neutral.",
  "messages": [
    {
      "role": "user",
      "content": "The packaging was damaged but the product itself works great."
    }
  ]
}
```

**Why this specific example is a good teaching case:** a zero-shot prompt gives the model no guidance on how to handle *mixed* sentiment (negative packaging, positive product) — it has to rely entirely on its own generalized judgment about how to weigh the two. The few-shot version, by contrast, doesn't include a mixed-sentiment example, so the model would need to extrapolate from the three given (clearly positive, clearly negative, clearly neutral) examples to this trickier case — illustrating that few-shot examples help most when they actually cover the relevant edge cases, not just any examples of the task in general. The chain-of-thought version explicitly invites the model to reason about the conflict (packaging vs. product) before committing to one label, which connects directly to the attention/context argument from Section 2: that reasoning becomes part of what later tokens can attend back to.

---

## 5. IMPLEMENTATION — Node.js: A Reusable Prompt Template System

```javascript
// prompt-templates.js
// A reusable prompt-construction system demonstrating zero-shot, few-shot,
// and chain-of-thought prompting patterns programmatically. This is genuinely
// useful production infrastructure: instead of hand-writing prompt strings
// inline throughout your codebase, you build them from structured, testable
// templates -- much like how you wouldn't hand-concatenate HTML strings in
// a real web app.

function buildZeroShotPrompt(task, input) {
  return `${task}\n\n${input}`;
}

function buildFewShotPrompt(task, examples, input) {
  const exampleText = examples
    .map(ex => `Input: ${ex.input}\nOutput: ${ex.output}`)
    .join("\n\n");
  return `${task}\n\nHere are some examples:\n\n${exampleText}\n\nNow do the same for:\nInput: ${input}\nOutput:`;
}

function buildChainOfThoughtPrompt(task, input) {
  return `${task}\n\n${input}\n\nLet's think through this step by step before giving the final answer.`;
}

function buildSystemUserPrompt(systemInstruction, userMessage) {
  return {
    system: systemInstruction,
    messages: [{ role: "user", content: userMessage }],
  };
}

// --- Demonstration: the SAME underlying task, built four different ways ---

const task = "Classify the sentiment of this product review as positive, negative, or neutral.";
const newReview = "The packaging was damaged but the product itself works great.";

console.log("=== ZERO-SHOT ===");
console.log(buildZeroShotPrompt(task, newReview));

console.log("\n\n=== FEW-SHOT ===");
const examples = [
  { input: "This phone case is amazing, fits perfectly!", output: "positive" },
  { input: "Arrived broken and customer service never responded.", output: "negative" },
  { input: "It's fine, does what it says, nothing special.", output: "neutral" },
];
console.log(buildFewShotPrompt(task, examples, newReview));

console.log("\n\n=== CHAIN-OF-THOUGHT ===");
console.log(buildChainOfThoughtPrompt(task, newReview));

console.log("\n\n=== SYSTEM vs USER (structured, as a real API call would send it) ===");
const structured = buildSystemUserPrompt(
  "You are a sentiment classification assistant. Always respond with exactly one word: positive, negative, or neutral.",
  newReview
);
console.log(JSON.stringify(structured, null, 2));
```

**Run it:**
```bash
node prompt-templates.js
```

**Verified output:** see Section 4 — confirmed by actually running this code; every template produces a correctly-structured prompt string or object.

#### Line-by-line explanation

- **`buildZeroShotPrompt(task, input)`** — the simplest possible template: task description, then input, with a blank line separating them for readability (and, practically, to give the model a clear visual/structural break between instruction and content — a small thing, but it mirrors how real prompts are typically formatted).
- **`buildFewShotPrompt(task, examples, input)`** — note the structure: every example is formatted *identically* (`Input: ... \nOutput: ...`), and the new input is formatted the *same way*, just missing its output. This consistency matters — recall Day 7's lesson on emergent structure: the model is essentially being asked to recognize and continue a pattern, and a consistent format makes that pattern easier to recognize, exactly as consistent formatting made our toy model's embeddings cluster cleanly.
- **`buildChainOfThoughtPrompt(task, input)`** — appends a single, simple instruction. Note how little code this requires — the technique's power comes from how it changes the model's *generation process* (Section 2's explanation), not from any complexity in constructing the prompt itself.
- **`buildSystemUserPrompt(systemInstruction, userMessage)`** — returns a structured object, not a flat string, deliberately mirroring the actual shape of a real API request (recall Day 14's CLI tool, which sent a `messages` array) — this is meant to be a direct, drop-in template for real API calls, not just an illustrative string.
- **Why build these as reusable functions instead of hand-writing each prompt inline** — exactly as you wouldn't hand-write raw SQL strings scattered through application code without some abstraction, building prompts via tested, reusable functions reduces bugs (a typo in one example's format), makes prompts easier to update consistently across your codebase, and makes the *task* (what varies) cleanly separable from the *pattern* (the reusable scaffolding).

---

## 6. TEACH-IT-BACK — Script You Can Say Out Loud

> "Prompt engineering isn't a separate, mysterious skill bolted onto how language models work — every technique connects directly to mechanisms we've already covered. Zero-shot prompting just asks the model to do a task by describing it, relying on the model having generalized the pattern of following instructions from its instruction-tuning stage. Few-shot prompting includes a handful of worked examples directly in the prompt before the actual question — and the reason this helps is mechanical, not magical: those examples become tokens in the same sequence as your real input, and self-attention lets the model directly attend back to them, regardless of how far away they are in the prompt, when deciding how to respond to your new input. Chain-of-thought prompting asks the model to reason step by step before giving a final answer, which helps on complex problems because the model generates text one token at a time, left to right — if you force it to jump straight to a conclusion, it has to get a multi-step answer right in one shot with no intermediate scratch space, but if you let it reason first, each reasoning step becomes part of the context that the model can attend back to when producing its next step and its final answer. And separating system prompts from user prompts is a structural choice: the system prompt sets persistent behavior or role for an entire conversation, while user messages carry the specific request for each turn, which avoids having to repeat your instructions every single time and gives the model a clearer signal about what's configuration versus what's the actual task at hand."

---

## Key Terms Glossary (Day 15)

| Term | Meaning |
|---|---|
| **Zero-shot prompting** | Asking for a task with no examples, relying on generalized instruction-following |
| **Few-shot prompting** | Including worked examples in the prompt before the actual task |
| **Chain-of-thought prompting** | Explicitly requesting step-by-step reasoning before a final answer |
| **System prompt** | Persistent instruction setting behavior/role across a conversation |
| **User prompt** | The specific request/message for a given conversational turn |

---

## Quick Self-Check (ask yourself before moving to Day 16)

1. Can I explain, mechanically (using attention, not just "it works"), why few-shot prompting helps?
2. Can I explain why chain-of-thought prompting connects to how models generate text token by token?
3. Can I explain the practical reason for separating system and user prompts?
4. Can I build a reusable prompt template function for a task of my choosing?

If yes to all four — you're ready for **Day 16: Advanced Prompting** (ReAct, self-consistency, prompt templates in more depth, implemented in Node.js).
