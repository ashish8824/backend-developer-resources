# Day 8 — Self-Attention Mechanism Explained (Query, Key, Value)

> Part of the 30-Day Generative AI Mastery Roadmap (for a Node.js Backend Developer)

---

## 1. WHAT — Plain Definitions

Day 6 used a simplified attention mechanism: comparing raw word embeddings directly via dot product. Today we build the **real** mechanism every Transformer actually uses — where each word is projected into **three separate, learned representations** before any comparison happens.

| Term | Plain Definition |
|---|---|
| **Query (Q)** | A learned projection of a word representing "what am I looking for in other words?" |
| **Key (K)** | A learned projection of a word representing "what do I have to offer, if someone is looking for something?" |
| **Value (V)** | A learned projection of a word representing "what information do I actually pass along, once someone decides to pay attention to me?" |
| **Scaled dot-product attention** | The full formula: compare Queries against Keys (dot product), scale the result, apply softmax, then use those weights to blend Value vectors. |
| **Scaling factor (√dₖ)** | A division step that prevents attention scores from growing too large as embedding dimension increases — crucial for stable training. |

### Diagram: Why Three Separate Vectors Instead of One?

```
                    ┌─────────────┐
   word embedding ──┤  W_query    ├──► Query vector   "What am I searching for?"
        │           └─────────────┘
        │           ┌─────────────┐
        ├───────────┤  W_key      ├──► Key vector     "What do I advertise about myself?"
        │           └─────────────┘
        │           ┌─────────────┐
        └───────────┤  W_value    ├──► Value vector   "What do I actually hand over?"
                     └─────────────┘

  W_query, W_key, W_value are three SEPARATE learned weight matrices
  (learned via backprop, Day 3) -- the SAME word embedding gets
  multiplied by each one to produce three DIFFERENT vectors.
```

**The key insight Day 6 was missing:** in Day 6's simplified version, a word's "relevance-detector" and "content-it-shares" were the same vector (the raw embedding). Today, those become two genuinely different, independently-learnable things — plus a third, separate vector for what's actually passed along. This separation is what gives Transformers their real expressive power.

---

## 2. WHY — Why Split Into Three Roles Instead of One?

### Why not just use the embedding directly, like Day 6?

Think about what "relevance" really means in language. The word "bank" should attend strongly to "river" in one sentence, and to "money" in another — but the *information that should flow* in each case is different too. If you used the exact same vector for "what am I looking for," "what do I offer," and "what do I pass along," the model would be stuck using one fixed representation for all three jobs. Separating them lets the network learn, for example: "when deciding relevance, focus on these aspects of meaning (Query/Key), but when actually passing information along, emphasize different aspects entirely (Value)." This is strictly more expressive than Day 6's simplified version, where all three jobs were forced to share one representation.

### Why is this learned, not hand-designed?

Exactly like Day 3: `W_query`, `W_key`, and `W_value` start as random matrices and get adjusted via backpropagation and gradient descent, based on whatever task the Transformer is being trained on (e.g., predicting the next word, Day 7's task, but at a much larger scale). The network discovers on its own which aspects of a word's meaning are useful for "searching," which are useful for "being found," and which are useful for "being transmitted" — nobody designs this by hand.

### Why divide by √dₖ? (the part most tutorials just assert without proof)

Here's the concrete problem: dot products between random vectors tend to grow larger in magnitude as the vectors get longer (more dimensions). If you feed unusually large numbers into softmax, softmax becomes extremely "peaked" — it starts behaving like a hard maximum, assigning almost all its weight to just the single highest-scoring item and almost none to everything else. This is bad for two reasons: it makes attention overly rigid (ignoring potentially useful context from other words), and — echoing Day 5's vanishing gradient problem — when softmax saturates this way, its gradient becomes very close to zero almost everywhere, which slows or stalls learning. Dividing by √dₖ (the square root of the key vector's dimension) counteracts this growth, keeping the scores in a range where softmax behaves in a balanced, well-behaved, and trainable way — **regardless of how large the embedding dimension is.**

---

## 3. HOW — The Mechanism (Intuition + Light Math)

### The full scaled dot-product attention formula

```
1. Query  = embedding × W_query
2. Key    = embedding × W_key
3. Value  = embedding × W_value

4. raw_score(word_i, word_j) = dot_product(Query_i, Key_j)
5. scaled_score(word_i, word_j) = raw_score(word_i, word_j) / sqrt(dK)
6. attention_weights_i = softmax(scaled_score(word_i, word_j) for all j)
7. output_i = Σ (attention_weight_ij × Value_j)   for all j
```

This is the famous formula from the 2017 paper, usually written in matrix form as:

```
Attention(Q, K, V) = softmax( (Q · Kᵗ) / √dₖ ) · V
```

Every symbol in that compact formula maps directly to a step above: `Q · Kᵗ` is step 4 (every query compared against every key, all at once via matrix multiplication), `/ √dₖ` is step 5, `softmax(...)` is step 6, and the final `· V` is step 7.

### Worked example: tracing one word through the formula by hand

Take the word "cat" with embedding `[1.0, 0.1, 0.2]`. Suppose (for this teaching example) `W_query` and `W_key` are identity matrices (so Query = Key = the raw embedding, isolating the effect of scaling and softmax), while `W_value` scales dimension 2 by 2x and dimensions 0-1 by 0.5x:

```
cat's Query = cat's Key = [1.0, 0.1, 0.2]   (since W_query, W_key = identity here)
cat's Value = [0.5, 0.05, 0.4]               (since W_value = diag(0.5, 0.5, 2.0))

dot_product(cat.Query, cat.Key) = (1.0×1.0) + (0.1×0.1) + (0.2×0.2) = 1.0 + 0.01 + 0.04 = 1.05

scaled (÷ sqrt(3), since embedding dim = 3) = 1.05 / 1.732 ≈ 0.606
```

This scaled score (0.606) then competes against the scaled scores for every *other* word in the sentence through softmax, producing the final attention weight "cat" assigns to itself. **Notice that "cat"'s Value vector `[0.5, 0.05, 0.4]` looks meaningfully different from its raw embedding `[1.0, 0.1, 0.2]`** — this is the concrete proof that Value is doing real, separate work: the information that ultimately gets passed along is not the same as the vector used to determine relevance.

---

## 4. EXAMPLE — Real-World Analogy + Concrete Verified Trace

### Analogy: A conference networking event

Imagine a conference where everyone wears three different name badges:

- **Query badge**: "I'm looking for someone who knows about machine learning deployment." (What you're searching for.)
- **Key badge**: "I have experience in: cloud infrastructure, Kubernetes, cost optimization." (What you advertise about yourself, so others can find you.)
- **Value badge**: hidden in your pocket — it's the *actual* detailed knowledge you'd share once someone approaches you, which might include things you didn't even list on your Key badge (maybe you also happen to know about security best practices).

You scan the room (compute dot products between your Query and everyone's Key) to find the best matches, normalize your interest into percentages of attention you'll give each person (softmax), and then — critically — you don't just remember their *Key badge* text, you actually absorb whatever real information they share with you (their Value). **The thing that determines relevance (Key) and the thing that actually gets transmitted (Value) can be, and often are, different.**

### Concrete trace: verified scaled dot-product attention on "the cat sat on mat"

Using toy 3D embeddings where "cat" and "mat" are deliberately similar (both object-like nouns), with `W_query = W_key = identity` (to isolate the scaling/softmax behavior) and `W_value` deliberately transforming the representation (scaling dim 2 up 2x, dims 0-1 down 0.5x):

| Query Word | Raw QK Scores (vs the,cat,sat,on,mat) | Scaled (÷√3) | Attention Weights (softmax) |
|---|---|---|---|
| **cat** | 0.110, 1.050, 0.220, 0.165, 1.015 | 0.064, 0.606, 0.127, 0.095, 0.586 | 0.154, **0.265**, 0.164, 0.159, **0.259** |
| **sat** | 0.110, 0.220, 1.020, 0.165, 0.270 | 0.064, 0.127, 0.589, 0.095, 0.156 | 0.170, 0.181, **0.287**, 0.175, 0.186 |
| **mat** | 0.110, 1.015, 0.270, 0.165, 0.987 | 0.064, 0.586, 0.156, 0.095, 0.570 | 0.154, **0.260**, 0.169, 0.159, **0.256** |

**What to notice:** "cat" pays its strongest attention to itself (0.265) and to "mat" (0.259) — nearly identical proportions, confirming the semantic similarity we built into their embeddings — while "sat" (a verb, unrelated to the noun-similarity dimension) correctly attends most strongly to itself (0.287) since nothing else in this toy sentence resembles it.

**Final output for "cat"** (weighted sum of Value vectors, not raw embeddings): `[0.283, 0.134, 0.268]` — note dimension 2 (0.268) is proportionally larger relative to dimensions 0-1 than you'd see in a plain weighted-average of raw embeddings, directly reflecting the Value matrix's 2x amplification of that dimension. **This is the concrete, verified proof that "what determines relevance" and "what gets passed along" really are separate computations.**

### Concrete proof: why scaling matters, verified statistically across 200 trials per dimension

| Embedding Dimension | Avg. Raw Score Magnitude | Max Softmax Weight WITHOUT Scaling | Max Softmax Weight WITH Scaling |
|---|---|---|---|
| 4 | 0.91 | 0.4168 | 0.3332 |
| 16 | 2.00 | 0.6010 | 0.3458 |
| 64 | 3.93 | 0.7568 | 0.3401 |
| 256 | 7.73 | 0.8599 | 0.3419 |
| 1024 | 15.33 | **0.9320** | **0.3402** |

This is a real, statistically robust pattern (averaged over 200 random trials at each dimension, confirmed stable across repeated runs): **without scaling, as embedding dimension grows, softmax becomes increasingly winner-take-all** — at dimension 1024, the top match grabs over 93% of the attention weight, nearly ignoring the other 3 candidates entirely. **With scaling, the maximum weight stays stable around 0.34 regardless of dimension** — close to the "fair" baseline of 0.25 you'd expect with 4 equally-relevant candidates, meaning the model retains the ability to genuinely *blend* information from multiple words instead of being forced into near-total tunnel vision on just one. This single division operation is what keeps attention well-behaved as real Transformers scale embedding dimensions into the hundreds or thousands.

---

## 5. IMPLEMENTATION — Node.js

### Part A — Full Scaled Dot-Product Attention with Q, K, V

```javascript
// qkv-self-attention-demo.js
// THE REAL self-attention mechanism: Query, Key, Value projections.
// Day 6 used raw embeddings directly to compute relevance. Today, each word
// gets transformed into THREE different learned roles before any comparison
// happens -- this is what every real Transformer (GPT, Claude, BERT) does.

function dotProduct(a, b) {
  return a.reduce((sum, val, i) => sum + val * b[i], 0);
}

function matrixVectorMultiply(matrix, vector) {
  // matrix is rows x cols, vector is length cols, result is length rows
  return matrix.map(row => dotProduct(row, vector));
}

function softmax(scores) {
  const max = Math.max(...scores);
  const exp = scores.map(s => Math.exp(s - max));
  const sum = exp.reduce((a, b) => a + b, 0);
  return exp.map(e => e / sum);
}

// Toy word embeddings (3-dimensional this time)
const wordEmbeddings = {
  the: [0.1, 0.1, 0.0],
  cat: [1.0, 0.1, 0.2],
  sat: [0.1, 1.0, 0.1],
  on:  [0.15, 0.15, 0.0],
  mat: [0.95, 0.15, 0.25],
};

// LEARNED projection matrices (hand-picked here for a clean teaching example --
// in a real Transformer these are learned via backprop, exactly like Day 3).
// Each is 3x3 since our embedding dimension is 3.
const W_query = [
  [1.0, 0.0, 0.0],
  [0.0, 1.0, 0.0],
  [0.0, 0.0, 1.0],
]; // identity for Query -- "what am I looking for?" (kept simple to isolate Key's effect)

const W_key = [
  [1.0, 0.0, 0.0],
  [0.0, 1.0, 0.0],
  [0.0, 0.0, 1.0],
]; // identity for Key -- "what do I have to offer?" (kept simple, same reason)

const W_value = [
  [0.5, 0.0, 0.0],
  [0.0, 0.5, 0.0],
  [0.0, 0.0, 2.0],
]; // Value DOES transform meaningfully -- "what information do I actually pass along?"
   // here, it down-weights dims 0/1 and amplifies dim 2 -- a deliberate, visible transformation

const dK = 3; // dimension of the key vectors -- needed for the scaling factor

function selfAttentionWithQKV(sentence) {
  const embeddings = sentence.map(w => wordEmbeddings[w]);

  // Step 1: project every word into its Query, Key, and Value vectors
  const queries = embeddings.map(e => matrixVectorMultiply(W_query, e));
  const keys = embeddings.map(e => matrixVectorMultiply(W_key, e));
  const values = embeddings.map(e => matrixVectorMultiply(W_value, e));

  const results = [];

  for (let i = 0; i < sentence.length; i++) {
    const query = queries[i]; // "what THIS word is looking for"

    // Step 2: compare this word's Query against EVERY word's Key
    const rawScores = keys.map(key => dotProduct(query, key));

    // Step 3: SCALE by sqrt(dK) -- this is the "scaled" in "scaled dot-product attention"
    const scaledScores = rawScores.map(score => score / Math.sqrt(dK));

    // Step 4: softmax to get attention weights
    const attentionWeights = softmax(scaledScores);

    // Step 5: weighted sum of VALUE vectors (not the raw embeddings, not the keys!)
    const outputDim = values[0].length;
    const output = new Array(outputDim).fill(0);
    for (let j = 0; j < values.length; j++) {
      for (let d = 0; d < outputDim; d++) {
        output[d] += attentionWeights[j] * values[j][d];
      }
    }

    results.push({ word: sentence[i], rawScores, scaledScores, attentionWeights, output });
  }

  return results;
}

const sentence = ["the", "cat", "sat", "on", "mat"];
const results = selfAttentionWithQKV(sentence);

console.log(`Sentence: "${sentence.join(" ")}"\n`);
results.forEach(({ word, rawScores, scaledScores, attentionWeights, output }) => {
  console.log(`--- Word: "${word}" ---`);
  console.log("  raw QK scores:   ", rawScores.map(s => s.toFixed(3)));
  console.log("  scaled QK scores:", scaledScores.map(s => s.toFixed(3)));
  console.log("  attention weights:", attentionWeights.map(s => s.toFixed(3)));
  console.log("  output (weighted sum of VALUES):", output.map(s => s.toFixed(3)));
  console.log();
});
```

**Run it:**
```bash
node qkv-self-attention-demo.js
```

**Verified output (deterministic, since W_query/W_key/W_value are fixed, not random):**
```
Sentence: "the cat sat on mat"

--- Word: "cat" ---
  raw QK scores:    [ '0.110', '1.050', '0.220', '0.165', '1.015' ]
  scaled QK scores: [ '0.064', '0.606', '0.127', '0.095', '0.586' ]
  attention weights: [ '0.154', '0.265', '0.164', '0.159', '0.259' ]
  output (weighted sum of VALUES): [ '0.283', '0.134', '0.268' ]

--- Word: "sat" ---
  raw QK scores:    [ '0.110', '0.220', '1.020', '0.165', '0.270' ]
  scaled QK scores: [ '0.064', '0.127', '0.589', '0.095', '0.156' ]
  attention weights: [ '0.170', '0.181', '0.287', '0.175', '0.186' ]
  output (weighted sum of VALUES): [ '0.215', '0.188', '0.223' ]
```
*(full output includes "the", "on", "mat" as well — see Section 4's table)*

#### Line-by-line explanation

- **`matrixVectorMultiply(matrix, vector)`** — implements `embedding × W` as a series of dot products, one per output dimension. This is the exact operation that "projects" a word's raw embedding into its Query, Key, or Value representation.
- **`W_query`, `W_key` set to identity matrices** — a deliberate teaching simplification. An identity matrix, when multiplied by any vector, returns that vector unchanged. We use this so Query and Key equal the raw embedding, letting you focus entirely on the *scaling and softmax* mechanics without an extra layer of transformation to track. In a real Transformer, these would be learned, non-identity matrices.
- **`W_value` is NOT identity** — deliberately, so you can see Value doing real, independent work: scaling dimensions 0 and 1 down by half, and dimension 2 up by double. Look at the `output` arrays and notice dimension 2's values are proportionally larger than you'd expect from a simple average of the raw embeddings — direct, verified proof that Value transforms information independently of Query/Key.
- **`scaledScores = rawScores.map(score => score / Math.sqrt(dK))`** — this single line is "scaled" in "scaled dot-product attention." `dK = 3` here since our toy embeddings are 3-dimensional.
- **The final `output`** — a weighted blend of Value vectors (never the raw embeddings, never the Keys) — exactly matching step 7 of the formula in Section 3.

---

### Part B — Why the Scaling Factor Matters (Verified Statistically)

```javascript
// why-scaling-matters-demo.js
// Demonstrates WHY scaled dot-product attention divides by sqrt(dK).
// Without scaling, as the embedding dimension grows, dot products grow
// larger in magnitude, pushing softmax into a "saturated" regime where
// it behaves almost like a hard max (one weight near 1, rest near 0) --
// which makes gradients vanish during training (echoes Day 5's problem!).

function softmax(scores) {
  const max = Math.max(...scores);
  const exp = scores.map(s => Math.exp(s - max));
  const sum = exp.reduce((a, b) => a + b, 0);
  return exp.map(e => e / sum);
}

function randomVector(dim, scale = 1) {
  return Array.from({ length: dim }, () => (Math.random() * 2 - 1) * scale);
}

function dotProduct(a, b) {
  return a.reduce((sum, val, i) => sum + val * b[i], 0);
}

function demonstrateAtDimension(dim, trials = 200) {
  let totalMaxRaw = 0;
  let totalMaxScaled = 0;
  let totalRawScoreMagnitude = 0;

  for (let t = 0; t < trials; t++) {
    const query = randomVector(dim);
    const keys = [randomVector(dim), randomVector(dim), randomVector(dim), randomVector(dim)];

    const rawScores = keys.map(k => dotProduct(query, k));
    const scaledScores = rawScores.map(s => s / Math.sqrt(dim));

    const rawWeights = softmax(rawScores);
    const scaledWeights = softmax(scaledScores);

    totalMaxRaw += Math.max(...rawWeights);
    totalMaxScaled += Math.max(...scaledWeights);
    totalRawScoreMagnitude += Math.max(...rawScores.map(Math.abs));
  }

  console.log(`\n--- Embedding dimension = ${dim} (averaged over ${trials} random trials) ---`);
  console.log(`Average raw score magnitude: ${(totalRawScoreMagnitude / trials).toFixed(2)} (grows with dimension)`);
  console.log(`Average MAX softmax weight WITHOUT scaling: ${(totalMaxRaw / trials).toFixed(4)} (closer to 1.0 = winner-take-all)`);
  console.log(`Average MAX softmax weight WITH scaling:    ${(totalMaxScaled / trials).toFixed(4)} (closer to 0.25 = healthy/balanced, since there are 4 keys)`);
}

console.log("Comparing softmax behavior at increasing embedding dimensions, with vs without scaling:");
console.log("(Averaged over 200 random trials per dimension, to show the real statistical trend)");
[4, 16, 64, 256, 1024].forEach(dim => demonstrateAtDimension(dim));
```

**Run it:**
```bash
node why-scaling-matters-demo.js
```

**Verified output:**
```
Comparing softmax behavior at increasing embedding dimensions, with vs without scaling:
(Averaged over 200 random trials per dimension, to show the real statistical trend)

--- Embedding dimension = 4 (averaged over 200 random trials) ---
Average raw score magnitude: 0.91 (grows with dimension)
Average MAX softmax weight WITHOUT scaling: 0.4168
Average MAX softmax weight WITH scaling:    0.3332

--- Embedding dimension = 1024 (averaged over 200 random trials) ---
Average raw score magnitude: 15.33 (grows with dimension)
Average MAX softmax weight WITHOUT scaling: 0.9320
Average MAX softmax weight WITH scaling:    0.3402
```
*(full output includes dimensions 16, 64, 256 as well — see Section 4's table)*

#### Line-by-line explanation

- **`randomVector(dim)`** — simulates a realistic Query/Key vector with random values, since we want to observe a *general statistical trend*, not a single cherry-picked example.
- **Running 200 trials per dimension and averaging** — this is important methodologically: a single random sample can be noisy and might not show the real trend clearly. Averaging over many trials reveals the genuine underlying statistical pattern, which is far more convincing and far more honest than presenting one lucky (or unlucky) draw as if it were universal.
- **The result is unambiguous**: without scaling, the average "winner" weight climbs steadily from 0.42 (dim 4) to 0.93 (dim 1024) — increasingly winner-take-all as dimension grows. With scaling, it stays essentially flat around 0.33-0.35 regardless of dimension. This directly validates the formula's design: **the scaling factor specifically cancels out the dimension-dependent growth in dot-product magnitude**, keeping softmax's behavior consistent no matter how large the model's embedding dimension is.

---

## 6. TEACH-IT-BACK — Script You Can Say Out Loud

> "Yesterday's simplified attention compared raw word embeddings directly. Today's real mechanism, used in every actual Transformer, first projects each word's embedding into three separate vectors using three separate learned weight matrices: a Query, representing what this word is looking for; a Key, representing what this word advertises about itself so other words can find it; and a Value, representing the actual information this word passes along once attention lands on it. Splitting these into three separate, independently-learnable projections is more powerful than using one shared vector for all three jobs, because the network can learn different things for 'how should I judge relevance' versus 'what should I actually transmit.' To compute attention, you take a word's Query and dot-product it against every word's Key, which gives you raw relevance scores. Critically, before applying softmax, you divide those scores by the square root of the key dimension — this is the 'scaled' in scaled dot-product attention, and it solves a real, measurable problem: as embedding dimension grows, dot products between random vectors tend to grow larger in magnitude, and feeding large numbers into softmax makes it behave almost like a hard, winner-take-all maximum, which both limits the model's ability to blend information from multiple words and causes near-zero gradients that stall learning. Dividing by the square root of the dimension cancels out that growth, keeping softmax balanced and well-behaved no matter how large the embedding dimension gets. Finally, you use those softmax-normalized attention weights to compute a weighted blend — not of the raw embeddings, and not of the Keys, but of the Value vectors — and that blended result becomes the new, context-aware representation for that word."

---

## Key Terms Glossary (Day 8)

| Term | Meaning |
|---|---|
| **Query (Q)** | Learned projection representing "what this word is looking for" |
| **Key (K)** | Learned projection representing "what this word advertises about itself" |
| **Value (V)** | Learned projection representing "what information this word actually transmits" |
| **Scaled dot-product attention** | `softmax((Q·Kᵗ)/√dₖ)·V` — the full real attention formula |
| **dₖ** | The dimensionality of the Key vectors, used in the scaling denominator |
| **Softmax saturation** | When softmax behaves like a hard maximum, assigning nearly all weight to one item — caused by overly large input scores |

---

## Quick Self-Check (ask yourself before moving to Day 9)

1. Can I explain, without looking, why Query, Key, and Value need to be three separate learned projections instead of one shared vector?
2. Can I write out the scaled dot-product attention formula from memory, and explain what each symbol means?
3. Can I explain, with real numbers, why dividing by √dₖ matters as embedding dimension grows?
4. Can I point to the exact place in the code where Value (not Query, not Key) determines the final output?

If yes to all four — you're ready for **Day 9: Multi-head attention + positional encoding** (why order matters to a Transformer, and how running attention multiple times in parallel, with different learned projections, captures richer relationships).
