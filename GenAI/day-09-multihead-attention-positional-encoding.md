# Day 9 — Multi-Head Attention + Positional Encoding

> Part of the 30-Day Generative AI Mastery Roadmap (for a Node.js Backend Developer)

---

## 1. WHAT — Plain Definitions

Day 8 built one complete attention computation (one Query/Key/Value projection set). Today covers two refinements every real Transformer needs on top of that: running attention **multiple times in parallel** with different learned projections (multi-head attention), and giving the model a sense of **word order** (positional encoding) — something pure attention, as built so far, completely lacks.

| Term | Plain Definition |
|---|---|
| **Multi-head attention** | Running several independent attention computations ("heads") in parallel on the same input, each with its own learned Q/K/V projection matrices, then combining their results. |
| **Head** | One complete, independent instance of the Day 8 attention mechanism — its own W_query, W_key, W_value. |
| **Concatenation** | Joining each head's output vectors side-by-side into one longer combined vector, after all heads finish. |
| **Positional encoding** | A vector added to each word's embedding that encodes *where* that word sits in the sequence, since attention alone has no concept of order. |
| **Sinusoidal positional encoding** | The specific formula from the original Transformer paper: alternating sine and cosine waves of different frequencies, one pair of frequencies per pair of embedding dimensions. |

### Diagram: Multi-Head Attention Structure

```
                         word embeddings (whole sentence)
                                    │
              ┌─────────────────────┼─────────────────────┐
              ▼                     ▼                     ▼
        ┌──────────┐          ┌──────────┐          ┌──────────┐
        │  HEAD 1  │          │  HEAD 2  │   ...    │  HEAD N  │
        │ own Q,K,V│          │ own Q,K,V│          │ own Q,K,V│
        │  matrices │          │ matrices │          │ matrices │
        └────┬─────┘          └────┬─────┘          └────┬─────┘
             │                     │                     │
             ▼                     ▼                     ▼
        head1 output          head2 output          headN output
              │                     │                     │
              └─────────────────────┼─────────────────────┘
                                    ▼
                       CONCATENATE all head outputs
                                    │
                                    ▼
                          combined multi-head output
```

Each head has its **own independently-learned** W_query, W_key, W_value matrices — meaning each head is free to specialize in noticing a completely different kind of relationship in the data, all from the exact same input.

---

## 2. WHY — Why Multiple Heads, and Why Positional Encoding?

### Why not just use one (bigger) attention head?

A single attention head computes one specific kind of relevance for every word pair — but language has many *simultaneous* kinds of relationships worth tracking. In a sentence, you might want one mechanism to track "which words refer to the same entity" (coreference), another to track "which words are grammatically related" (subject-verb agreement), another to track "which words are semantically similar topics." A single head, with one shared Q/K/V projection, is forced to compromise across all of these uses at once. **Multiple heads let the network learn several different "lenses" simultaneously, each focusing on a different aspect of the relationships between words**, and then combine all those perspectives together.

### Why does this actually work — won't the heads just learn the same thing?

Because each head's W_query, W_key, W_value matrices are **initialized differently and trained with their own independent gradients** (recall Day 3 — gradient descent finds *a* good solution, not *the* unique solution, and different starting points lead to different specializations). In practice, when researchers inspect trained Transformers, different heads really do specialize — some attend to nearby words, some to syntactically related words far away, some to punctuation, some to specific recurring patterns relevant to the training data. We'll prove this concretely below by hand-designing two heads that genuinely capture different relationships from the same sentence.

### Why does attention need positional encoding at all?

Here's the crucial gap in everything built so far (Day 6, Day 8): self-attention computes relevance using **dot products between content vectors** — and dot products don't know or care about *where* in the sequence a word sits. This means, mathematically, "dog bites man" and "man bites dog" would produce **identical** attention scores between "dog" and "bites," because attention only ever looked at *what* each word's vector contains, never *where* it appeared. But word order obviously changes meaning in language — "dog bites man" and "man bites dog" describe very different events! Positional encoding solves this by injecting *position-specific* information directly into each word's vector, before attention ever runs, so that the *same word* gets a *different* representation depending on where it appears.

### Why this specific sinusoidal formula, rather than just numbering positions 1, 2, 3...?

You might think: why not just add the position number (1, 2, 3...) directly to the embedding? The problem is that raw position numbers grow unboundedly (position 1000 vs position 1 is a huge, unnormalized difference) and don't naturally encode *relative* distance in a way a network can easily learn to exploit. The sinusoidal approach instead uses waves of different frequencies — some oscillating fast (capturing fine differences between nearby positions) and some oscillating slowly (capturing broad differences between distant positions) — all while keeping every value bounded between -1 and 1, just like the embeddings they're added to. This also has a nice mathematical property (relative positions correspond to consistent rotations in the encoding space) that helps the network generalize to sequence lengths it didn't see during training.

---

## 3. HOW — The Mechanism (Intuition + Light Math)

### Multi-head attention, mechanically

```
1. For each head h (1 to N):
     Q_h = embedding × W_query_h
     K_h = embedding × W_key_h
     V_h = embedding × W_value_h
     head_output_h = scaled_dot_product_attention(Q_h, K_h, V_h)   [exactly Day 8's formula]

2. concatenated = concatenate(head_output_1, head_output_2, ..., head_output_N)

3. final_output = concatenated × W_output   [one more learned matrix, mixing the heads together]
```

Every head runs the *exact same formula* from Day 8 — the only difference is each head has its own private W_query, W_key, W_value. Step 3's final mixing matrix (`W_output`) is also learned, and lets the model combine information across heads in a flexible way, rather than just leaving them concatenated side by side.

### The sinusoidal positional encoding formula

```
PE(position, 2i)   = sin(position / 10000^(2i / d_model))
PE(position, 2i+1) = cos(position / 10000^(2i / d_model))
```

Here, `position` is the word's index in the sequence (0, 1, 2, ...), `i` indexes pairs of dimensions, and `d_model` is the total embedding dimension. Each pair of dimensions `(2i, 2i+1)` shares one frequency, alternating between a sine and a cosine of that frequency. As `i` increases, the denominator `10000^(2i/d_model)` grows, making the angle change *more slowly* as position increases — which is why **low dimensions oscillate quickly (sensitive to small position changes) while high dimensions oscillate slowly (sensitive only to large position changes)**.

### Combining positional encoding with the word embedding

```
final_input_vector = word_embedding + positional_encoding(position)
```

Just simple element-wise addition. This single sum is what gets fed into the attention mechanism — meaning attention now operates on vectors that encode *both* "what this word means" and "where it sits," entangled together, even though they started as two conceptually separate pieces.

---

## 4. EXAMPLE — Real-World Analogy + Concrete Verified Trace

### Analogy for multi-head attention: a panel of specialist reviewers

Imagine submitting a manuscript for review, and instead of one editor reading it once, you get a panel: one reviewer focuses purely on grammar, another purely on factual accuracy, another purely on narrative structure. Each reviewer reads the *exact same manuscript*, but because they're looking through a different "lens" (a different set of priorities, like each head's different W matrices), they each surface different issues. At the end, you combine all their separate feedback into one comprehensive review. That combination is exactly what concatenating multi-head attention outputs achieves.

### Analogy for positional encoding: assigning seat numbers at a conference

Imagine a name-badge system (the word embeddings) that tells you *who* someone is, but says nothing about *where they're sitting*. If you wanted to know "who is sitting next to whom" or "who spoke first," the name badge alone can't tell you. Positional encoding is like additionally stamping a seat number onto each badge — now the *combination* of identity and seat number tells a richer story, and you can finally answer order-dependent questions like "who spoke before whom."

### Concrete trace: multi-head attention with genuinely different specializations (verified by running the code)

Using two intentionally different heads on the sentence "the cat sat on mat": **Head 1** mirrors Day 8's setup (sensitive to noun/content similarity), while **Head 2** is deliberately tuned (via its W matrices) to amplify only the "verb-ness" dimension of the toy embeddings.

| Word | Head 1 weights (content similarity) | Head 2 weights (verb-dimension signal) |
|---|---|---|
| the | 0.194, 0.204, 0.204, 0.195, 0.204 | 0.128, 0.128, **0.469**, 0.138, 0.138 |
| cat | 0.154, **0.265**, 0.164, 0.159, **0.259** | 0.128, 0.128, **0.469**, 0.138, 0.138 |
| sat | 0.170, 0.181, **0.287**, 0.175, 0.186 | 0.000, 0.000, **1.000**, 0.000, 0.000 |
| mat | 0.154, **0.260**, 0.169, 0.159, **0.256** | 0.089, 0.089, **0.624**, 0.099, 0.099 |

**What this proves:** Head 1 correctly reproduces Day 8's pattern — "cat" and "mat" attend most strongly to each other (content similarity). Head 2 produces a **completely different pattern** — every word's strongest attention target is "sat" (ranging from 0.469 to a full 1.000), because Head 2's W matrices were designed to amplify exactly the dimension where "sat" has by far the strongest signal. **The same input sentence, processed by two heads with different learned projections, yields genuinely different relevance patterns** — concrete proof of why multiple heads can capture multiple distinct relationships simultaneously.

I double-checked the dramatic 1.000 self-attention for "sat" in Head 2 wasn't a numerical fluke: tracing the raw dot products by hand, "sat"'s self-comparison scores 25 (5×5, since its embedding's relevant dimension is amplified 5x by the head's query/key matrices), while its comparison to every other word scores only 2.5–3.75 — an enormous, genuine gap that softmax correctly converts into near-total self-focus.

### Concrete trace: proving attention alone is position-blind, then fixing it (verified by running the code)

```
score(dog, bites) in sentence A ("dog bites man"): 0.180
score(dog, bites) in sentence B ("man bites dog"): 0.180
```

**Identical.** This confirms the problem stated in Section 2: pure content-based attention cannot distinguish where in the sentence a word appears — only what it means.

**Positional encoding vectors for the first 5 positions** (6-dimensional, verified output):

| Position | Encoding Vector |
|---|---|
| 0 | [0.000, 1.000, 0.000, 1.000, 0.000, 1.000] |
| 1 | [0.841, 0.540, 0.046, 0.999, 0.002, 1.000] |
| 2 | [0.909, -0.416, 0.093, 0.996, 0.004, 1.000] |
| 4 | [-0.757, -0.654, 0.185, 0.983, 0.009, 1.000] |

**Notice the pattern:** dimensions 0-1 swing widely and quickly between positions (0.000 → 0.841 → 0.909 → ... → -0.757), while dimensions 4-5 barely change at all (0.000 → 0.002 → 0.004 → ... → 0.009) — exactly the "fast-oscillating low dimensions, slow-oscillating high dimensions" design described in Section 3.

**Proof that adding positional encoding breaks the symmetry from before:**

```
"dog" at position 0 (sentence A, "dog bites man"): [0.900, 1.100, 0.000, 1.000, 0.000, 1.000]
"dog" at position 2 (sentence B, "man bites dog"): [1.809, -0.316, 0.093, 0.996, 0.004, 1.000]
```

These are now genuinely **different vectors for the exact same word**, purely because of where it sits in its sentence. This is the concrete fix to the problem demonstrated above — once positional encoding is added, attention computations involving "dog" will produce different results in sentence A versus sentence B, finally allowing the model to distinguish "dog bites man" from "man bites dog."

---

## 5. IMPLEMENTATION — Node.js

### Part A — Multi-Head Attention with Two Genuinely Different Heads

```javascript
// multi-head-attention-demo.js
// Demonstrates MULTI-HEAD attention: running several independent
// Query/Key/Value attention computations IN PARALLEL on the same input,
// each with its OWN learned projection matrices, so each "head" can
// specialize in noticing a different kind of relationship.
//
// Today's toy example: Head 1 specializes in noun-noun similarity
// (like Day 8's cat/mat relationship). Head 2 specializes in a
// DIFFERENT relationship -- adjacent-word position -- to prove the
// two heads genuinely learn different things from the same input.

function dotProduct(a, b) {
  return a.reduce((sum, val, i) => sum + val * b[i], 0);
}

function matrixVectorMultiply(matrix, vector) {
  return matrix.map(row => dotProduct(row, vector));
}

function softmax(scores) {
  const max = Math.max(...scores);
  const exp = scores.map(s => Math.exp(s - max));
  const sum = exp.reduce((a, b) => a + b, 0);
  return exp.map(e => e / sum);
}

// Same toy embeddings as Day 8 (3-dimensional)
const wordEmbeddings = {
  the: [0.1, 0.1, 0.0],
  cat: [1.0, 0.1, 0.2],
  sat: [0.1, 1.0, 0.1],
  on:  [0.15, 0.15, 0.0],
  mat: [0.95, 0.15, 0.25],
};

const dK = 3;

// --- HEAD 1: tuned (via its W matrices) to pick up on noun-noun (content) similarity ---
const head1 = {
  W_query: [[1,0,0],[0,1,0],[0,0,1]], // identity -- pass content through directly
  W_key:   [[1,0,0],[0,1,0],[0,0,1]],
  W_value: [[1,0,0],[0,1,0],[0,0,1]],
};

// --- HEAD 2: tuned (via its W matrices) to amplify dimension 1 specifically --
// imagine dimension 1 happens to encode "verb-ness" -- this head specializes
// in detecting verb-related signal, a DIFFERENT relationship than Head 1.
const head2 = {
  W_query: [[0,0,0],[0,5,0],[0,0,0]], // only look at dimension 1 (verb-ness), zero out the rest
  W_key:   [[0,0,0],[0,5,0],[0,0,0]],
  W_value: [[0.2,0,0],[0,1,0],[0,0,0.2]],
};

function runAttentionHead(sentence, head) {
  const embeddings = sentence.map(w => wordEmbeddings[w]);
  const queries = embeddings.map(e => matrixVectorMultiply(head.W_query, e));
  const keys = embeddings.map(e => matrixVectorMultiply(head.W_key, e));
  const values = embeddings.map(e => matrixVectorMultiply(head.W_value, e));

  return sentence.map((word, i) => {
    const rawScores = keys.map(k => dotProduct(queries[i], k));
    const scaledScores = rawScores.map(s => s / Math.sqrt(dK));
    const weights = softmax(scaledScores);
    const outputDim = values[0].length;
    const output = new Array(outputDim).fill(0);
    for (let j = 0; j < values.length; j++) {
      for (let d = 0; d < outputDim; d++) output[d] += weights[j] * values[j][d];
    }
    return { word, weights, output };
  });
}

function multiHeadAttention(sentence, heads) {
  // Run EVERY head independently and in parallel on the SAME input.
  const headOutputs = heads.map(head => runAttentionHead(sentence, head));

  // Concatenate each word's outputs across all heads into one combined vector.
  return sentence.map((word, i) => {
    const concatenated = headOutputs.flatMap(headResult => headResult[i].output);
    const perHeadWeights = headOutputs.map(headResult => headResult[i].weights);
    return { word, perHeadWeights, concatenated };
  });
}

const sentence = ["the", "cat", "sat", "on", "mat"];
const results = multiHeadAttention(sentence, [head1, head2]);

console.log(`Sentence: "${sentence.join(" ")}"\n`);
results.forEach(({ word, perHeadWeights, concatenated }) => {
  console.log(`--- Word: "${word}" ---`);
  console.log(`  Head 1 attention weights (content/noun similarity):`, perHeadWeights[0].map(w => w.toFixed(3)));
  console.log(`  Head 2 attention weights (verb-dimension signal):  `, perHeadWeights[1].map(w => w.toFixed(3)));
  console.log(`  Concatenated multi-head output:`, concatenated.map(v => v.toFixed(3)));
  console.log();
});
```

**Run it:**
```bash
node multi-head-attention-demo.js
```

**Verified output (excerpt — full output covers all 5 words):**
```
--- Word: "cat" ---
  Head 1 attention weights (content/noun similarity): [ '0.154', '0.265', '0.164', '0.159', '0.259' ]
  Head 2 attention weights (verb-dimension signal):   [ '0.128', '0.128', '0.469', '0.138', '0.138' ]
  Concatenated multi-head output: [ '0.566', '0.268', '0.134', '0.068', '0.536', '0.021' ]

--- Word: "sat" ---
  Head 1 attention weights (content/noun similarity): [ '0.170', '0.181', '0.287', '0.175', '0.186' ]
  Head 2 attention weights (verb-dimension signal):   [ '0.000', '0.000', '1.000', '0.000', '0.000' ]
  Concatenated multi-head output: [ '0.430', '0.377', '0.112', '0.020', '1.000', '0.020' ]
```

#### Line-by-line explanation

- **`head1` and `head2` objects** — each contains its own independent `W_query`, `W_key`, `W_value`. This is the entire definition of "multi-head": just running Day 8's exact mechanism multiple times with different matrices.
- **`runAttentionHead(sentence, head)`** — literally Day 8's `selfAttentionWithQKV` function, just parameterized so it can accept *any* head's matrices instead of one fixed set.
- **`multiHeadAttention`** — calls `runAttentionHead` once per head (this loop *could* run in parallel across heads, since each head's computation is fully independent — another layer of parallelism on top of Day 6's "every word at once" parallelism), then uses `flatMap` to concatenate each word's per-head outputs into one combined vector.
- **The verified results prove genuine specialization**: Head 1's weights for "cat" closely mirror Day 8's results (cat attends to cat/mat). Head 2's weights are dramatically different — every word attends almost entirely to "sat" (up to 1.000 for "sat" itself), because Head 2's matrices were specifically built to amplify the one embedding dimension where "sat" stands out. **Same sentence, same words, completely different attention pattern — purely due to different learned (here, hand-set for clarity) projection matrices.**

---

### Part B — Positional Encoding: The Problem and the Fix

```javascript
// positional-encoding-demo.js
// Demonstrates WHY positional encoding is needed, and HOW the sinusoidal
// formula from the original Transformer paper works.
//
// KEY PROBLEM this solves: self-attention (Day 6, Day 8) computes relevance
// based purely on CONTENT similarity -- it has NO built-in sense of word
// ORDER. "dog bites man" and "man bites dog" would produce IDENTICAL
// attention patterns for "dog" and "man" if we only used content embeddings,
// because attention doesn't know which position anything is in.

function dotProduct(a, b) {
  return a.reduce((sum, val, i) => sum + val * b[i], 0);
}

// --- PART 1: prove attention alone is position-blind ---

const wordEmbeddings = {
  dog: [0.9, 0.1],
  bites: [0.1, 0.9],
  man: [0.85, 0.15],
};

function plainAttentionScore(wordA, wordB) {
  return dotProduct(wordEmbeddings[wordA], wordEmbeddings[wordB]);
}

console.log("--- Proof: plain content-based attention score doesn't care about ORDER ---");
console.log('Sentence A: "dog bites man"');
console.log('Sentence B: "man bites dog"');
console.log();
console.log("score(dog, bites) in sentence A:", plainAttentionScore("dog", "bites").toFixed(3));
console.log("score(dog, bites) in sentence B:", plainAttentionScore("dog", "bites").toFixed(3));
console.log("^ IDENTICAL, because content-only attention has no idea which word came first.\n");

// --- PART 2: the sinusoidal positional encoding formula ---

function positionalEncoding(position, dim, totalDims) {
  // Even dimensions use sine, odd dimensions use cosine.
  // The "frequency" of the wave changes across dimensions -- low dimensions
  // oscillate fast (capture fine-grained position differences), high
  // dimensions oscillate slowly (capture coarse, long-range position differences).
  const exponent = (2 * Math.floor(dim / 2)) / totalDims;
  const denominator = Math.pow(10000, exponent);
  const angle = position / denominator;
  return dim % 2 === 0 ? Math.sin(angle) : Math.cos(angle);
}

function getPositionalEncodingVector(position, totalDims) {
  return Array.from({ length: totalDims }, (_, dim) => positionalEncoding(position, dim, totalDims));
}

const encodingDim = 6;
console.log("--- Positional encoding vectors for positions 0 through 4 ---");
for (let pos = 0; pos < 5; pos++) {
  const vec = getPositionalEncodingVector(pos, encodingDim);
  console.log(`position ${pos}:`, vec.map(v => v.toFixed(3)));
}

// --- PART 3: prove WORD + POSITION encoding breaks the symmetry from Part 1 ---

function addVectors(a, b) {
  return a.map((val, i) => val + b[i]);
}

// Embed "dog" and "man" with a 2D content embedding PADDED to match positional dims,
// then add positional encoding for their respective positions in each sentence.
const dogContentPadded = [0.9, 0.1, 0, 0, 0, 0];
const manContentPadded = [0.85, 0.15, 0, 0, 0, 0];

const posEncoding0 = getPositionalEncodingVector(0, encodingDim); // first word in sentence
const posEncoding2 = getPositionalEncodingVector(2, encodingDim); // third word in sentence

const dogAtPosition0 = addVectors(dogContentPadded, posEncoding0); // "dog bites man" -> dog is position 0
const dogAtPosition2 = addVectors(dogContentPadded, posEncoding2); // "man bites dog" -> dog is position 2

console.log("\n--- Proof: word + position encoding breaks the symmetry ---");
console.log('"dog" at position 0 (sentence A, "dog bites man"):', dogAtPosition0.map(v => v.toFixed(3)));
console.log('"dog" at position 2 (sentence B, "man bites dog"):', dogAtPosition2.map(v => v.toFixed(3)));
console.log("^ These are now DIFFERENT vectors for the SAME word, purely because of position.");
console.log("This is exactly what lets a Transformer tell these two sentences apart.");
```

**Run it:**
```bash
node positional-encoding-demo.js
```

**Verified output:**
```
--- Proof: plain content-based attention score doesn't care about ORDER ---
Sentence A: "dog bites man"
Sentence B: "man bites dog"

score(dog, bites) in sentence A: 0.180
score(dog, bites) in sentence B: 0.180
^ IDENTICAL, because content-only attention has no idea which word came first.

--- Positional encoding vectors for positions 0 through 4 ---
position 0: [ '0.000', '1.000', '0.000', '1.000', '0.000', '1.000' ]
position 1: [ '0.841', '0.540', '0.046', '0.999', '0.002', '1.000' ]
position 2: [ '0.909', '-0.416', '0.093', '0.996', '0.004', '1.000' ]
position 3: [ '0.141', '-0.990', '0.139', '0.990', '0.006', '1.000' ]
position 4: [ '-0.757', '-0.654', '0.185', '0.983', '0.009', '1.000' ]

--- Proof: word + position encoding breaks the symmetry ---
"dog" at position 0 (sentence A, "dog bites man"): [ '0.900', '1.100', '0.000', '1.000', '0.000', '1.000' ]
"dog" at position 2 (sentence B, "man bites dog"): [ '1.809', '-0.316', '0.093', '0.996', '0.004', '1.000' ]
^ These are now DIFFERENT vectors for the SAME word, purely because of position.
This is exactly what lets a Transformer tell these two sentences apart.
```

#### Line-by-line explanation

- **`plainAttentionScore`** — computes a relevance score using only content embeddings, with zero position information — proving the gap that positional encoding needs to fill.
- **`positionalEncoding(position, dim, totalDims)`** — implements the exact formula from Section 3. `Math.floor(dim / 2)` groups dimensions into pairs (0,1 share `i=0`; 2,3 share `i=1`; etc.), `Math.pow(10000, exponent)` computes the frequency-controlling denominator, and the final `dim % 2 === 0 ? sin : cos` alternates between sine and cosine within each pair — verified to match the paper's formula structure exactly (confirmed by checking dim=0,1 both use exponent 0, denominator 1, so both directly use `position` as the angle; dim=2,3 use a larger denominator, slowing the wave; and so on).
- **`addVectors(dogContentPadded, posEncoding0)`** — this single addition is the entire mechanism: word meaning and position get fused into one vector via simple element-wise sum, exactly as described in Section 3.
- **The final proof** — "dog"'s vector is genuinely different depending on its position (`[0.900, 1.100, ...]` at position 0 vs. `[1.809, -0.316, ...]` at position 2), which means any downstream attention computation involving "dog" will now behave differently depending on where it sits in the sentence — finally giving the Transformer the order-sensitivity that pure content-based attention was missing.

---

## 6. TEACH-IT-BACK — Script You Can Say Out Loud

> "Yesterday we built one complete attention computation, with one Query, Key, and Value projection. Today's first idea, multi-head attention, just runs that exact same computation multiple times in parallel, on the same input, but each 'head' has its own independently learned projection matrices. Because each head starts from different random weights and gets its own gradients during training, different heads end up specializing in noticing different kinds of relationships in the data — one might focus on nearby words, another on semantically similar words far away, another on some pattern specific to the training data. After all heads finish, you concatenate their outputs together into one longer vector, optionally mixing them with one more learned matrix. Today's second idea solves a problem that's been hiding in everything we've built so far: self-attention computes relevance using dot products between content vectors, and dot products genuinely don't know or care about word order — 'dog bites man' and 'man bites dog' would produce identical attention scores between 'dog' and 'bites,' because attention only ever looks at what a word means, never where it sits. Positional encoding fixes this by adding a special vector to each word's embedding before attention runs — a vector built from sine and cosine waves of different frequencies, one frequency pair per pair of embedding dimensions, with low dimensions oscillating quickly to capture fine position differences and high dimensions oscillating slowly to capture broad ones. Once you add this positional vector to a word's content embedding, the exact same word ends up with a genuinely different combined vector depending on where it appears in the sequence — which is exactly what lets a Transformer finally distinguish sentences that use the same words in different orders."

---

## Key Terms Glossary (Day 9)

| Term | Meaning |
|---|---|
| **Multi-head attention** | Running several independent attention computations in parallel, each with its own Q/K/V matrices |
| **Head** | One independent instance of the attention mechanism within a multi-head setup |
| **Concatenation** | Joining multiple heads' output vectors side by side into one combined vector |
| **Positional encoding** | A vector encoding sequence position, added to word embeddings before attention |
| **Sinusoidal positional encoding** | The sine/cosine, multi-frequency formula used in the original Transformer paper |
| **d_model** | The total embedding dimension, used in the positional encoding frequency formula |

---

## Quick Self-Check (ask yourself before moving to Day 10)

1. Can I explain, without looking, why running multiple attention heads can capture more than a single head could?
2. Can I demonstrate, with a concrete example, why plain self-attention is "position-blind"?
3. Can I explain why the sinusoidal formula uses different frequencies across dimensions, rather than just adding the raw position number?
4. Can I explain how word embedding and positional encoding get combined into one vector?

If yes to all four — you're ready for **Day 10: Tokenization** (BPE, tokenizers, why "strawberry" confuses LLMs, with implementation using `js-tiktoken`).
