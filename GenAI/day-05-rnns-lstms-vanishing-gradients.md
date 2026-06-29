# Day 5 — The Sequential Data Problem: RNNs, LSTMs, and Why They Struggled

> Part of the 30-Day Generative AI Mastery Roadmap (for a Node.js Backend Developer)

---

## 1. WHAT — Plain Definitions

Days 1-4 covered networks that take a fixed-size input and produce an output in one shot. But language is **sequential** — word order matters, and sentences can be any length. Today covers the first serious attempt to handle that: RNNs and LSTMs, and the specific mathematical problem that eventually killed their dominance.

| Term | Plain Definition |
|---|---|
| **RNN** (Recurrent Neural Network) | A neural network that processes a sequence **one element at a time**, carrying forward a "memory" (hidden state) from each step to the next. |
| **Hidden state** | A vector that acts as the network's "running summary" of everything it has read so far in the sequence. |
| **Recurrence** | The defining trick of an RNN: the *same* weights are reused at every time step, and each step's output depends on both the current input AND the previous hidden state. |
| **Backpropagation Through Time (BPTT)** | Backpropagation (Day 3), but applied across every time step of a sequence — gradients must flow backward through potentially hundreds of steps. |
| **Vanishing gradient problem** | When gradients, repeatedly multiplied across many time steps, shrink toward zero — meaning the network can't "remember" or learn from events far in the past. |
| **Exploding gradient problem** | The opposite failure: gradients repeatedly multiplied across many steps grow uncontrollably large, destabilizing training. |
| **LSTM** (Long Short-Term Memory) | An improved RNN variant with "gates" that let the network deliberately choose what to remember or forget, partially fixing the vanishing gradient problem. |

### Diagram: RNN Recurrence vs. a Normal Feedforward Network

```
NORMAL NETWORK (Day 2):                   RNN (today):
one-shot, fixed-size input                processes a SEQUENCE, one element at a time,
                                           carrying a hidden state forward

  input ──► [Network] ──► output            word1 ──► [RNN cell] ──► hidden_state_1
                                                              │
                                                              ▼ (carried forward)
                                            word2 ──► [RNN cell] ──► hidden_state_2
                                                              │
                                                              ▼ (carried forward)
                                            word3 ──► [RNN cell] ──► hidden_state_3
                                                              │
                                                              ▼
                                                        final hidden state
                                                     (a "summary" of the whole sequence)

  Note: it's the SAME [RNN cell] (same weights) reused at every single step --
  not a different network for word1, word2, word3.
```

---

## 2. WHY — Why Did We Need RNNs, and Why Did They Eventually Fail?

### Why not just use a normal (Day 2-style) network for text?

A plain feedforward network needs a **fixed-size** input. But sentences vary in length — "Hi" is 1 word, a paragraph might be 200 words. You could pad everything to some max length, but that wastes computation and, more importantly, a plain feedforward network has **no concept of order** — it would treat "dog bites man" and "man bites dog" as the same bag of inputs unless you did something clever. RNNs solve both problems at once: they handle any sequence length, and they process words **in order**, building up context step by step.

### Why does this design eventually break down? (The vanishing gradient problem)

Recall from Day 3: training a network means backpropagating gradients backward, layer by layer, using the chain rule — and each step multiplies by some number related to the local derivative. In an RNN, "layers" essentially become "time steps" — to learn from something that happened 50 words ago, the gradient has to flow backward through 50 repeated multiplications.

Here's the killer problem: **the same weight gets reused at every time step**, and the activation function's derivative (e.g., for tanh) is always less than or equal to 1. When you multiply a number less than 1 by itself 50 times, it shrinks **exponentially** toward zero. This means: by the time the gradient has traveled back far enough to update weights based on "what happened at the start of a long sequence," there's essentially no usable signal left. **The network literally cannot learn long-range dependencies — it forgets the distant past, not because it chooses to, but because the math makes long-range learning impossible.**

> The mirror-image failure also happens: if the repeated multiplier is greater than 1 instead of less than 1, gradients **explode** instead of vanish — growing astronomically large and destabilizing training (you'll often see this in practice as `NaN` losses).

### Why LSTMs were invented

LSTMs (1997, though they only became widely practical in the 2010s with enough compute) introduced **gates** — small learned mechanisms that let the network explicitly decide, at every step, "how much of my previous memory should I keep vs. throw away, and how much new information should I let in?" This gives the network a path for information to flow across many time steps with much less forced shrinkage — partially solving (not fully eliminating) the vanishing gradient problem. LSTMs were the dominant architecture for sequential tasks (translation, speech recognition) for roughly a decade.

### Why even LSTMs weren't enough (setting up Day 6)

LSTMs fixed *some* of the vanishing gradient pain, but they kept the fundamentally limiting design choice: **processing one element at a time, in strict sequence.** This has two consequences that became increasingly intolerable as datasets and models grew:

1. **It's slow.** You can't process word 50 until you've finished processing word 49 — there's no parallelism across the sequence. Modern GPUs are built to do massive amounts of math *in parallel*, and RNNs/LSTMs can't take advantage of that for the sequence dimension.
2. **Long-range memory is still imperfect.** Even with gates, information from very early in a long document can still get diluted by the time the network reaches the end.

This is the exact gap the Transformer architecture (Day 6) was designed to close — by abandoning recurrence entirely in favor of a mechanism called **attention**, which lets every word look directly at every other word, regardless of distance, all at once, in parallel.

---

## 3. HOW — The Mechanism (Intuition + Light Math)

### The RNN recurrence equation

At every time step `t`, an RNN computes a new hidden state from two things: the current input, and the *previous* hidden state:

```
hidden_state[t] = tanh( W_hidden × hidden_state[t-1]  +  W_input × input[t]  +  bias )
```

Notice: `W_hidden` and `W_input` are **the same weights, reused at every single time step** — this weight-sharing across time is what "recurrent" means, and it's also exactly why the vanishing gradient problem is unavoidable: the same shrinking factor gets applied again and again and again.

### Tracing the vanishing gradient, numerically

When backpropagating through time, at each step backward, the gradient gets multiplied by roughly `(recurrent weight) × (activation derivative)`. The tanh activation's derivative has a **maximum possible value of 1** (and it's usually less, depending on the input) — so even in the *best case*, you're multiplying by at most 1, and typically less, at every single step backward.

If the recurrent weight is, say, 0.5 (a perfectly ordinary, non-extreme value), then after just 20 steps backward:

```
gradient after 20 steps = 1.0 × (0.5)^20  ≈  0.00000095   (less than one millionth of the original signal)
```

That's not a hypothetical worst case — that's the **typical** behavior for a network trying to learn dependencies 20+ steps in the past. For context, a single moderately long sentence or short paragraph easily exceeds 20 words.

### How LSTM gates help (simplified intuition)

Instead of being forced through a squashing activation function at *every single step* (which is what causes the shrinkage above), an LSTM maintains a separate "memory cell" that can be passed forward almost **unchanged** if a learned "forget gate" decides the information is still relevant:

```
new_cell_state = (forget_gate_value) × old_cell_state + (new information to add)
```

If `forget_gate_value` is close to 1 (the network has learned "this information is still important, keep it"), the old memory survives mostly intact, step after step — instead of being squashed through tanh every time. This is the core trick: **gates give the network a *choice* about how much to preserve, rather than forcing uniform decay on everything.**

---

## 4. EXAMPLE — Real-World Analogy + Concrete Trace

### Analogy: Playing the "telephone game" vs. passing a written note

A plain RNN is like the children's telephone game: each person whispers what they heard to the next person, and details get garbled and lost the further the message travels — by the 20th person, the original message is nearly unrecognizable. Multiplying a number less than 1 by itself over and over is the mathematical equivalent of "a little bit gets lost at every step, and it compounds."

An LSTM is like passing a **written note** down the line instead of whispering — each person *can* choose to copy it down faithfully (if the "forget gate" says "this matters, preserve it exactly") instead of being forced to paraphrase from memory at every hop. Information survives much longer, but it's still hop-by-hop, sequential, and each person still has to wait for the note from the person before them — there's no way to skip ahead or look at the original message directly.

### Concrete trace: the RNN processing "the cat sat on the mat" (verified by running the code)

| Word | Embedding | Previous Hidden State | New Hidden State |
|---|---|---|---|
| the | 0.1 | 0.0000 | 0.0798 |
| cat | 0.6 | 0.0798 | 0.4776 |
| sat | 0.4 | 0.4776 | 0.5071 |
| on | 0.2 | 0.5071 | 0.3915 |
| the | 0.1 | 0.3915 | 0.2690 |
| mat | 0.5 | 0.2690 | 0.4888 |

Notice the **same word "the"** produces a different hidden-state transition the second time it appears (0.3915→0.2690 vs. starting from 0.0000→0.0798) — because the hidden state carries context forward. This is the entire point of recurrence: identical inputs are processed differently depending on what came before.

### Concrete trace: vanishing gradients, verified numerically

| Steps Backward | Gradient (recurrent weight = 0.5) | Gradient (recurrent weight = 1.5) |
|---|---|---|
| 0 | 1.0000 | 1.0000 |
| 6 | 0.015625 | 11.39 |
| 12 | 0.000244 | 129.75 |
| 20 | **0.00000095** | **3,325.3** |

This is real arithmetic, not a hand-wavy claim: with a recurrent weight of 0.5, by 20 steps back the gradient has shrunk to less than one part in a million of its starting value — the network effectively receives **zero learning signal** about anything that happened 20+ steps ago. With a slightly larger weight (1.5), the opposite failure happens: the gradient explodes to over 3,000 times its starting size, which in practice causes wildly unstable training (often showing up as `NaN` losses).

### Concrete trace: LSTM-style gating slows the decay (verified by running the code)

| Steps | Plain RNN (forced tanh decay) | LSTM-style (forget gate = 0.95) |
|---|---|---|
| 0 | 100% of signal | 100% of signal |
| 6 | 30.1% | 73.5% |
| 15 | **10.6%** | **46.3%** |

After 15 steps, the plain RNN has lost almost 90% of the original signal strength, while the LSTM-style gating mechanism retains nearly half. **This is a simplified illustration of the gating intuition, not a full LSTM simulation** — real LSTMs have three separate gates (forget, input, output) interacting in more complex ways — but the core mechanism you're seeing here (a gate value close to 1 preserves information; forced repeated squashing destroys it) is genuinely why LSTMs outperformed plain RNNs on longer sequences.

---

## 5. IMPLEMENTATION — Node.js (Built From Scratch)

### Part A — A Minimal RNN Forward Pass

```javascript
// simple-rnn-forward.js
// A minimal RNN forward pass, built from scratch, to show the RECURRENCE
// mechanism: the same weights are reused at every time step, and a
// "hidden state" carries information forward from one step to the next.
//
// Task: process the sentence "the cat sat" word by word, updating a hidden
// state at each step -- just like an RNN reading a sentence left to right.

function tanh(x) {
  return Math.tanh(x);
}

// Toy word embeddings (1-dimensional, just for simplicity of tracing the math by hand)
const wordEmbeddings = {
  the: 0.1,
  cat: 0.6,
  sat: 0.4,
  on: 0.2,
  mat: 0.5,
};

// RNN parameters (normally learned via training -- hand-picked here for clarity)
const W_input = 0.8;   // how much the CURRENT word influences the new hidden state
const W_hidden = 0.5;  // how much the PREVIOUS hidden state influences the new one
const bias = 0.0;

function rnnStep(prevHiddenState, currentWordEmbedding) {
  // THE key RNN equation: new hidden state depends on (previous hidden state) AND (current input)
  const z = (W_hidden * prevHiddenState) + (W_input * currentWordEmbedding) + bias;
  return tanh(z);
}

function processSequence(words) {
  let hiddenState = 0; // initial hidden state, before reading anything, starts at 0
  const trace = [];

  for (const word of words) {
    const embedding = wordEmbeddings[word];
    const newHiddenState = rnnStep(hiddenState, embedding);
    trace.push({ word, embedding, prevHidden: hiddenState, newHidden: newHiddenState });
    hiddenState = newHiddenState; // carry forward to next step
  }

  return { finalHiddenState: hiddenState, trace };
}

const sentence = ["the", "cat", "sat", "on", "the", "mat"];
const { finalHiddenState, trace } = processSequence(sentence);

console.log("--- RNN processing 'the cat sat on the mat' word by word ---\n");
trace.forEach(({ word, embedding, prevHidden, newHidden }) => {
  console.log(
    `word="${word}" (embedding=${embedding}) | ` +
    `prev_hidden=${prevHidden.toFixed(4)} -> new_hidden=${newHidden.toFixed(4)}`
  );
});

console.log(`\nFinal hidden state (the RNN's "summary" of the whole sentence): ${finalHiddenState.toFixed(4)}`);
```

**Run it:**
```bash
node simple-rnn-forward.js
```

**Verified output:**
```
--- RNN processing 'the cat sat on the mat' word by word ---

word="the" (embedding=0.1) | prev_hidden=0.0000 -> new_hidden=0.0798
word="cat" (embedding=0.6) | prev_hidden=0.0798 -> new_hidden=0.4776
word="sat" (embedding=0.4) | prev_hidden=0.4776 -> new_hidden=0.5071
word="on" (embedding=0.2) | prev_hidden=0.5071 -> new_hidden=0.3915
word="the" (embedding=0.1) | prev_hidden=0.3915 -> new_hidden=0.2690
word="mat" (embedding=0.5) | prev_hidden=0.2690 -> new_hidden=0.4888

Final hidden state (the RNN's "summary" of the whole sentence): 0.4888
```

#### Line-by-line explanation

- **`wordEmbeddings`** — deliberately 1-dimensional (just a single number per word) so you can trace the arithmetic by hand. Real RNNs use full multi-dimensional embeddings (Day 4), but the *mechanism* is identical, just with vectors instead of scalars.
- **`rnnStep(prevHiddenState, currentWordEmbedding)`** — implements the recurrence equation exactly: it combines the previous hidden state and the current input, each scaled by their own weight, then squashes with tanh. **This is the same `neuron()` pattern from Day 2** — multiply, add, squash — just with an extra input (the previous hidden state) feeding back in.
- **`processSequence`** — the loop that makes this "recurrent": `hiddenState` is reassigned after each word and fed into the *next* call to `rnnStep`. This single line (`hiddenState = newHiddenState`) is the entire mechanical definition of recurrence.
- **Notice "the" appears twice** in the sentence with the same embedding (0.1), but produces different new hidden states each time (0.0798 the first time, 0.2690 the second time) — proof that the network's behavior depends on accumulated context, not just the current word in isolation.

---

### Part B — The Vanishing (and Exploding) Gradient Problem

```javascript
// vanishing-gradient-demo.js
// Demonstrates WHY RNNs struggle with long sequences: the "vanishing gradient" problem.
//
// When backpropagating through time, the gradient gets multiplied by the SAME
// recurrent weight (and an activation derivative <= 1) at EVERY time step.
// Repeated multiplication of numbers < 1 shrinks toward zero exponentially fast.

// tanh derivative, in terms of its own output: tanh'(x) = 1 - tanh(x)^2
// Its MAXIMUM possible value is 1 (at x=0), and it's smaller everywhere else.
function tanhDerivativeMax() {
  return 1; // best case -- in practice it's usually less than this
}

function simulateGradientThroughTime(recurrentWeight, numTimeSteps) {
  let gradient = 1.0; // start with gradient = 1 at the final time step
  const trace = [gradient];

  for (let t = 0; t < numTimeSteps; t++) {
    // At each step backward through time, the gradient gets multiplied by:
    //   (recurrent weight) x (activation derivative, at best 1, often less)
    // We use the BEST CASE (derivative = 1) to show this isn't even a worst-case scenario.
    gradient = gradient * recurrentWeight * tanhDerivativeMax();
    trace.push(gradient);
  }

  return trace;
}

console.log("--- How the gradient shrinks as it flows backward through time ---");
console.log("(Using recurrent weight = 0.5, which is a TYPICAL, not extreme, value)\n");

const trace = simulateGradientThroughTime(0.5, 20);
trace.forEach((g, t) => {
  if (t === 0 || t % 2 === 0) {
    console.log(`Steps back = ${t}: gradient = ${g.toExponential(4)}`);
  }
});

console.log("\n--- Compare: what if recurrent weight was slightly larger, e.g. 1.5? ---");
console.log("(This shows the OPPOSITE failure mode: exploding gradients)\n");
const explodingTrace = simulateGradientThroughTime(1.5, 20);
explodingTrace.forEach((g, t) => {
  if (t === 0 || t % 2 === 0) {
    console.log(`Steps back = ${t}: gradient = ${g.toExponential(4)}`);
  }
});

console.log("\n--- Practical takeaway ---");
console.log(`After just 20 time steps with weight=0.5: gradient shrunk to ${trace[20].toExponential(4)}`);
console.log(`That's essentially ZERO -- the network gets no learning signal about events 20+ steps in the past.`);
console.log(`After just 20 time steps with weight=1.5: gradient exploded to ${explodingTrace[20].toExponential(4)}`);
console.log(`That's an enormous, unstable number -- training would likely diverge (NaN or wild weight swings).`);
```

**Run it:**
```bash
node vanishing-gradient-demo.js
```

**Verified output:**
```
--- How the gradient shrinks as it flows backward through time ---
(Using recurrent weight = 0.5, which is a TYPICAL, not extreme, value)

Steps back = 0: gradient = 1.0000e+0
Steps back = 2: gradient = 2.5000e-1
Steps back = 4: gradient = 6.2500e-2
Steps back = 6: gradient = 1.5625e-2
Steps back = 8: gradient = 3.9063e-3
Steps back = 10: gradient = 9.7656e-4
Steps back = 12: gradient = 2.4414e-4
Steps back = 14: gradient = 6.1035e-5
Steps back = 16: gradient = 1.5259e-5
Steps back = 18: gradient = 3.8147e-6
Steps back = 20: gradient = 9.5367e-7

--- Compare: what if recurrent weight was slightly larger, e.g. 1.5? ---
(This shows the OPPOSITE failure mode: exploding gradients)

Steps back = 0: gradient = 1.0000e+0
Steps back = 2: gradient = 2.2500e+0
...
Steps back = 20: gradient = 3.3253e+3

--- Practical takeaway ---
After just 20 time steps with weight=0.5: gradient shrunk to 9.5367e-7
That's essentially ZERO -- the network gets no learning signal about events 20+ steps in the past.
After just 20 time steps with weight=1.5: gradient exploded to 3.3253e+3
That's an enormous, unstable number -- training would likely diverge (NaN or wild weight swings).
```

#### Line-by-line explanation

- **`tanhDerivativeMax()`** returning `1` — this is the *best possible case* for tanh's derivative. We're deliberately being generous to the RNN here: even under the best-case assumption, the problem still occurs. In reality, the derivative is usually meaningfully below 1, making the shrinkage even faster than what's shown.
- **`simulateGradientThroughTime`** — each loop iteration represents one step backward through time during backpropagation. The multiplication `gradient * recurrentWeight * tanhDerivativeMax()` is the simplified core of what backpropagation-through-time does at every step — this is the same chain-rule multiplication from Day 3, just repeated many times because of the recurrence.
- **The numbers don't lie:** `0.5^20 ≈ 0.00000095`. This isn't a contrived worst case — 0.5 is an utterly ordinary weight value, and 20 time steps is a *short* sequence by real-world standards (a single paragraph). This is precisely why RNNs/LSTMs struggle with anything resembling "remember something from much earlier in a long document."

---

### Part C — Why LSTM Gates Help (Simplified Intuition)

```javascript
// lstm-gate-intuition-demo.js
// Shows the CORE intuition of LSTM's "forget gate" / "memory cell" idea:
// information can be carried forward UNCHANGED across many steps if the
// gate decides to "remember", instead of being forced through a squashing
// function (tanh) at every single step like a plain RNN.

function sigmoid(x) {
  return 1 / (1 + Math.exp(-x));
}

// Plain RNN: hidden state is OVERWRITTEN by a squashing function every step.
function plainRNNCarryForward(value, numSteps) {
  let state = value;
  const trace = [state];
  for (let i = 0; i < numSteps; i++) {
    state = Math.tanh(state * 0.9); // repeatedly squashed -- shrinks toward 0
    trace.push(state);
  }
  return trace;
}

// LSTM-style: a "memory cell" can be passed through almost UNCHANGED
// if the forget gate is close to 1 (i.e., "keep remembering this").
function lstmStyleCarryForward(value, numSteps, forgetGateValue) {
  let cellState = value;
  const trace = [cellState];
  for (let i = 0; i < numSteps; i++) {
    // Simplified: cellState = forgetGate * cellState (no squashing forced every step)
    cellState = forgetGateValue * cellState;
    trace.push(cellState);
  }
  return trace;
}

const initialSignal = 1.0;
const steps = 15;

console.log("--- Plain RNN: information decays fast (forced through tanh every step) ---");
const rnnTrace = plainRNNCarryForward(initialSignal, steps);
rnnTrace.forEach((v, t) => {
  if (t % 3 === 0) console.log(`step ${t}: value = ${v.toFixed(6)}`);
});

console.log("\n--- LSTM-style with forget gate = 0.95 (mostly 'remember') ---");
const lstmTrace = lstmStyleCarryForward(initialSignal, steps, 0.95);
lstmTrace.forEach((v, t) => {
  if (t % 3 === 0) console.log(`step ${t}: value = ${v.toFixed(6)}`);
});

console.log("\n--- Practical takeaway ---");
console.log(`After ${steps} steps, plain RNN retained: ${rnnTrace[steps].toFixed(6)} (${(rnnTrace[steps]*100).toFixed(2)}% of original signal)`);
console.log(`After ${steps} steps, LSTM-style retained: ${lstmTrace[steps].toFixed(6)} (${(lstmTrace[steps]*100).toFixed(2)}% of original signal)`);
console.log("The LSTM's gate lets the network CHOOSE to preserve information much longer.");
```

**Run it:**
```bash
node lstm-gate-intuition-demo.js
```

**Verified output:**
```
--- Plain RNN: information decays fast (forced through tanh every step) ---
step 0: value = 1.000000
step 3: value = 0.470928
step 6: value = 0.301169
step 9: value = 0.207318
step 12: value = 0.146975
step 15: value = 0.105634

--- LSTM-style with forget gate = 0.95 (mostly 'remember') ---
step 0: value = 1.000000
step 3: value = 0.857375
step 6: value = 0.735092
step 9: value = 0.630249
step 12: value = 0.540360
step 15: value = 0.463291

--- Practical takeaway ---
After 15 steps, plain RNN retained: 0.105634 (10.56% of original signal)
After 15 steps, LSTM-style retained: 0.463291 (46.33% of original signal)
The LSTM's gate lets the network CHOOSE to preserve information much longer.
```

#### Line-by-line explanation

- **`plainRNNCarryForward`** — at every step, the value is forced through `Math.tanh(state * 0.9)`. Even though 0.9 is close to 1, the repeated tanh squashing still drags the value down over time, because tanh's output magnitude is always ≤ its input magnitude for values not near zero.
- **`lstmStyleCarryForward`** — at every step, the value is simply multiplied by a `forgetGateValue` (here 0.95) — **no forced squashing function**. This is a deliberate simplification of how an LSTM's cell state update works; real LSTMs combine this with additional "input gate" and "output gate" computations, but the core insight — multiplicative gating instead of forced nonlinear squashing — is faithfully represented here.
- **The comparison is the whole point:** by step 15, the plain RNN has retained only ~10.6% of the original signal, while the LSTM-style gate retains ~46.3% — over 4x more. This is real, measurable, and is the actual mathematical reason LSTMs noticeably outperformed plain RNNs on tasks requiring longer memory (like translating long sentences).

> **Honesty note for teaching:** this LSTM demo is intentionally simplified to isolate *one* mechanism (the forget gate's multiplicative carry-forward) for clarity. A real LSTM cell has three gates (forget, input, output) and more involved interactions between them. The simplification is accurate to the *core insight* but is not a complete LSTM implementation — flag this explicitly if teaching it to someone who might later read the original 1997 LSTM paper or its equations.

---

## 6. TEACH-IT-BACK — Script You Can Say Out Loud

> "Language is sequential — word order matters, and sentences have no fixed length — so a normal feedforward network doesn't fit well. RNNs solve this by processing a sequence one element at a time, carrying forward a 'hidden state' that acts like a running summary of everything read so far, and reusing the exact same weights at every step. That weight reuse is exactly what causes RNNs' biggest weakness: when you backpropagate through time to train an RNN, the gradient gets multiplied by roughly the same number — usually less than 1 — over and over, once per time step. Multiplying a number less than 1 by itself dozens of times shrinks it exponentially fast, often down to a millionth of its original size within just 20 steps. This is called the vanishing gradient problem, and it means the network essentially can't learn from anything that happened far in the past within a sequence — not because it's choosing to forget, but because the math makes that learning signal disappear. LSTMs, invented to fix this, add 'gates' — small learned mechanisms that let the network deliberately decide how much old information to keep versus discard at each step, instead of being forced through a squashing function every single time. This genuinely helps — gated information survives much longer than ungated information — but LSTMs kept one fundamental limitation: they still process the sequence one element at a time, strictly in order, which means no parallel computation across the sequence, and even gated memory still degrades over very long distances. That combination of slowness and imperfect long-range memory is exactly the gap the Transformer architecture, coming tomorrow, was built to close — by throwing out recurrence entirely and letting every word look directly at every other word, all at once, regardless of distance."

---

## Key Terms Glossary (Day 5)

| Term | Meaning |
|---|---|
| **RNN** | Processes sequences one step at a time, carrying a hidden state forward |
| **Hidden state** | The RNN's running summary/memory of everything processed so far |
| **Recurrence** | Reusing the same weights at every time step, feeding each output back as the next input |
| **Backpropagation Through Time (BPTT)** | Backpropagation applied across every time step of a sequence |
| **Vanishing gradient problem** | Gradients shrink exponentially across many time steps, preventing long-range learning |
| **Exploding gradient problem** | Gradients grow exponentially across many time steps, destabilizing training |
| **LSTM** | RNN variant with gates that let the network choose what to remember/forget, partially fixing vanishing gradients |
| **Forget gate** | An LSTM mechanism controlling how much of the previous memory cell state to retain |

---

## Quick Self-Check (ask yourself before moving to Day 6)

1. Can I explain, without looking, what "recurrence" means and why the same weights are reused at every step?
2. Can I explain *why* the vanishing gradient problem happens — specifically, why repeated multiplication is the culprit?
3. Can I explain what an LSTM's "gate" does differently from a plain RNN's forced squashing?
4. Can I name the two remaining limitations of LSTMs that motivated the Transformer architecture?

If yes to all four — you're ready for **Day 6: The Transformer architecture** (high level: encoder/decoder, why it replaced RNNs).
