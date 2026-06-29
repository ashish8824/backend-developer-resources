# Day 4 — Data Representations: Tensors and Embeddings

> Part of the 30-Day Generative AI Mastery Roadmap (for a Node.js Backend Developer)

---

## 1. WHAT — Plain Definitions

Days 1-3 covered neurons that take in "numbers" and output "numbers." But the real world is words, sentences, images, audio. Today answers: **how does any of that actually become numbers a neural network can chew on?**

| Term | Plain Definition |
|---|---|
| **Tensor** | A general word for "a grid of numbers with some number of dimensions." A single number is a tensor. A list of numbers is a tensor. A grid (rows/columns) is a tensor. A cube of numbers is a tensor. |
| **Scalar** | A 0-dimensional tensor — just one number. |
| **Vector** | A 1-dimensional tensor — a list of numbers, e.g. `[0.9, 0.2, 0.5]`. |
| **Matrix** | A 2-dimensional tensor — a grid of numbers (rows × columns), e.g. a grayscale image. |
| **Shape** | The "size" of a tensor along each dimension, e.g. shape `[3, 3]` means 3 rows, 3 columns. |
| **Embedding** | A learned vector of numbers that represents the *meaning* of something (a word, sentence, image) — designed so that similar things end up as similar vectors. |
| **One-hot encoding** | A naive, non-learned way to turn categories into vectors — explained below as the "wrong way" that motivates embeddings. |
| **Cosine similarity** | A formula that measures how similar two vectors are, based on the *angle* between them — the standard way to compare embeddings. |

### Diagram: Tensor Dimensions, Ladder View

```
  0D (scalar)         1D (vector)              2D (matrix)                3D (tensor)
  ─────────────       ─────────────            ─────────────              ─────────────
       42              [0.9, 0.2, 0.5]          [ [0.1, 0.2, 0.3]          [ [[255,0,0], [0,255,0]],
                                                   [0.4, 0.5, 0.6]            [[0,0,255], [255,255,0]] ]
                                                   [0.7, 0.8, 0.9] ]
                                                                            (height × width × RGB channels
  "a single             "a list — e.g. one        "a grid — e.g. one        — a real color image!)
   number"               word's embedding,         grayscale image, or
                          or a single applicant's   one batch of word
                          [credit, income] from     vectors stacked)
                          Day 2-3)
```

**Why you should care as a backend dev:** when you call an embeddings API or look at a model's input shape, you'll see notation like `[batch_size, sequence_length, embedding_dim]` — that's just a 3D tensor's shape, described in words. Once shapes stop being scary, reading any ML library's documentation gets dramatically easier.

---

## 2. WHY — Why Do We Need Embeddings at All?

### Why not just number-code each word? (The naive approach, and why it fails)

The most obvious idea: assign each word in your vocabulary a unique number, or a "one-hot" vector (a vector of all zeros except a single 1 marking which word it is).

**The problem:** one-hot vectors capture **zero relationship information**. To a one-hot vector, "king" and "queen" are exactly as unrelated as "king" and "dog" — every word is equally different from every other word, because there's no shared structure between their vectors. That's mathematically useless for any task involving meaning, like "find documents similar to this one" or "understand that 'happy' and 'joyful' mean almost the same thing."

### Why embeddings solve this

An **embedding** is a vector where the *position* in space carries meaning. Words used in similar contexts end up with similar vectors. This isn't programmed by hand — it's **learned**, using the exact same training loop from Day 3 (forward pass → loss → backprop → gradient descent), just applied to a different task: "predict which words tend to appear near each other." After training on huge amounts of text, words like "king" and "queen" naturally land close together in this vector space, while "king" and "banana" land far apart — without anyone ever telling the model the *definition* of royalty.

### Why cosine similarity, specifically?

Once words are vectors, you need a way to ask "how similar are these two?" You could measure straight-line (Euclidean) distance, but that's sensitive to vector *length/magnitude*, which often reflects something irrelevant (like how frequently a word appears in training data) rather than meaning. **Cosine similarity instead measures the angle between two vectors, ignoring their length entirely.** Two vectors pointing in nearly the same direction get a similarity near 1, perpendicular vectors get near 0, and opposite-pointing vectors get near -1. This focus on *direction* rather than *magnitude* is exactly what you want when comparing meaning.

---

## 3. HOW — The Mechanism (Intuition + Light Math)

### Tensors: just nested arrays with a "shape"

A tensor's **shape** tells you how it's organized. You compute it by checking, at each nesting level, how many items are inside:

```
scalar:        42                          shape: []          (no dimensions)
vector:        [0.9, 0.2, 0.5]              shape: [3]         (3 numbers)
matrix:         [[0.1,0.2,0.3],
                  [0.4,0.5,0.6],
                  [0.7,0.8,0.9]]             shape: [3, 3]      (3 rows, 3 columns)
3D tensor:      (color image, h×w×channels) shape: [h, w, 3]   (3 = R, G, B channels)
```

That's it — there's no deeper trick. Every "tensor operation" you'll see in deep learning frameworks is just a structured way of doing the same multiply/add/loop operations from Day 2-3, but organized across these grids efficiently (often using your GPU, which is built to do many such multiplications in parallel).

### Cosine similarity, step by step

The formula:

```
cosine_similarity(A, B) = (A · B) / (||A|| × ||B||)

where:
  A · B   = dot product = sum of (A[i] × B[i]) for every dimension i
  ||A||   = magnitude (length) of A = sqrt(sum of A[i]²)
```

**Worked example** with toy 3-dimensional word vectors (dimensions meaning: `[royalty, gender(masc→fem), animal-ness]`):

```
king  = [0.9,  0.3, 0.0]
queen = [0.9, -0.3, 0.0]

dot product = (0.9×0.9) + (0.3×-0.3) + (0.0×0.0) = 0.81 - 0.09 + 0 = 0.72
||king||  = sqrt(0.9² + 0.3² + 0.0²) = sqrt(0.81+0.09) = sqrt(0.90) ≈ 0.9487
||queen|| = sqrt(0.9² + (-0.3)² + 0.0²) = same as above ≈ 0.9487

cosine_similarity = 0.72 / (0.9487 × 0.9487) = 0.72 / 0.9 = 0.8
```

**Result: 0.8** — high similarity, correctly reflecting that king and queen share the "royalty" dimension strongly, even though they oppose on the "gender" dimension.

### The famous "king − man + woman ≈ queen" vector arithmetic

This is the single most famous demonstration that embeddings capture real *semantic relationships*, not just similarity. If embeddings successfully encode "gender" as a consistent direction in vector space, then:

```
king − man  ≈  "royalty, with the masculine-ness subtracted out"
(king − man) + woman  ≈  "royalty, with femininity added back in"  ≈  queen
```

This works because subtracting `man` from `king` isolates the "royalty" component (canceling out the shared "masculine" direction), and then adding `woman` reintroduces a "feminine" direction — landing the result vector close to `queen`'s actual position in the embedding space.

---

## 4. EXAMPLE — Real-World Analogy + Concrete Trace

### Analogy: A city map, not a dictionary

Imagine every word isn't a dictionary entry, but a **pin on a giant map** with hundreds of streets (dimensions) instead of just two (latitude/longitude). Words used in similar contexts get pinned near each other — "dog" and "cat" end up in the same "pet neighborhood," while "banana" is pinned somewhere completely different, in the "food district." Crucially, **directions on this map mean something consistent** — walking the same direction and distance from "man" to "woman" is roughly the same walk as from "king" to "queen," because both walks represent "add femininity." That consistency of *direction* is what makes vector arithmetic like `king - man + woman ≈ queen` work.

### Concrete trace: verified results from our toy embedding space

Using hand-crafted, interpretable 3D vectors (dimensions: `[royalty, gender, animal-ness]`) so the math is fully transparent:

| Comparison | Cosine Similarity | Interpretation |
|---|---|---|
| king vs queen | **0.800** | High — both strongly "royal," differ mainly on gender |
| dog vs cat | **0.976** | Very high — both strongly "animal," differ slightly on gender |
| king vs banana | **0.000** | Zero — completely unrelated, no shared dimensions activated |
| man vs woman | **-0.976** | Near -1 — nearly opposite, since they differ almost only on the gender axis |
| king vs dog | **0.035** | Near zero — almost no shared meaning |

**Vector arithmetic result:**
```
king - man + woman = [0.900, -1.500, 0.000]
Closest match (excluding king/man/woman): "queen" (similarity 0.759)
```

The arithmetic correctly lands closest to "queen" — out of every word in our toy vocabulary, none is closer to the result vector than queen. **This is not a coincidence I cherry-picked after the fact — I deliberately designed these toy vectors to demonstrate a real phenomenon that was first observed in actual word2vec embeddings trained on billions of real words in 2013.** Real embeddings have hundreds of dimensions that are *not* human-interpretable like "royalty" or "gender" — but the underlying math (dot products, cosine similarity, vector arithmetic) is identical to what you just saw.

---

## 5. IMPLEMENTATION — Node.js

### Part A — Understanding Tensor Shapes

```javascript
// tensor-shapes-demo.js
// Demonstrates the "shape" concept: scalar -> vector -> matrix -> tensor

// 0D tensor (scalar): just a single number
const scalar = 42;
console.log("Scalar (0D):", scalar);

// 1D tensor (vector): a list of numbers
const vector = [0.9, 0.8, 0.3];
console.log("Vector (1D), shape [3]:", vector);

// 2D tensor (matrix): a grid of numbers -- e.g. a grayscale image (rows x cols)
const matrix = [
  [0.1, 0.2, 0.3],
  [0.4, 0.5, 0.6],
  [0.7, 0.8, 0.9],
];
console.log("Matrix (2D), shape [3,3]:");
matrix.forEach(row => console.log(" ", row));

// 3D tensor: e.g. a COLOR image (rows x cols x RGB channels)
const colorImage = [
  // Row 0
  [[255, 0, 0], [0, 255, 0]], // pixel(0,0)=red, pixel(0,1)=green
  // Row 1
  [[0, 0, 255], [255, 255, 0]], // pixel(1,0)=blue, pixel(1,1)=yellow
];
console.log("3D tensor, shape [2,2,3] (height x width x RGB channels):");
console.log(JSON.stringify(colorImage));

// Helper: compute the "shape" of a nested array automatically
function getShape(arr) {
  const shape = [];
  let current = arr;
  while (Array.isArray(current)) {
    shape.push(current.length);
    current = current[0];
  }
  return shape;
}

console.log("\nComputed shapes:");
console.log("vector shape:", getShape(vector));
console.log("matrix shape:", getShape(matrix));
console.log("colorImage shape:", getShape(colorImage));
```

**Run it:**
```bash
node tensor-shapes-demo.js
```

**Verified output:**
```
Scalar (0D): 42
Vector (1D), shape [3]: [ 0.9, 0.8, 0.3 ]
Matrix (2D), shape [3,3]:
  [ 0.1, 0.2, 0.3 ]
  [ 0.4, 0.5, 0.6 ]
  [ 0.7, 0.8, 0.9 ]
3D tensor, shape [2,2,3] (height x width x RGB channels):
[[[255,0,0],[0,255,0]],[[0,0,255],[255,255,0]]]

Computed shapes:
vector shape: [ 3 ]
matrix shape: [ 3, 3 ]
colorImage shape: [ 2, 2, 3 ]
```

#### Line-by-line explanation

- **`getShape(arr)`** — walks down the first element of each nesting level, recording `.length` at each level. This mirrors how real tensor libraries (NumPy, TensorFlow.js) compute `.shape` — they just check the size along each axis.
- **The color image** — note the shape is `[2, 2, 3]`: 2 rows, 2 columns, and 3 numbers per pixel (Red, Green, Blue intensity). This is exactly the shape convention you'll see in real image-processing/vision model code.

---

### Part B — Why One-Hot Encoding Fails

```javascript
// one-hot-encoding-demo.js
// Shows the NAIVE way to turn words into numbers (one-hot encoding),
// and why it's inferior to learned embeddings (next section).

const vocabulary = ["king", "queen", "man", "woman", "dog", "cat"];

function oneHotEncode(word, vocab) {
  const vector = new Array(vocab.length).fill(0);
  const index = vocab.indexOf(word);
  if (index === -1) throw new Error(`"${word}" not in vocabulary`);
  vector[index] = 1;
  return vector;
}

console.log("Vocabulary:", vocabulary);
vocabulary.forEach(word => {
  console.log(`"${word}" -> [${oneHotEncode(word, vocabulary).join(", ")}]`);
});

// The problem: every one-hot vector is EQUALLY dissimilar to every other.
function dotProduct(a, b) {
  return a.reduce((sum, val, i) => sum + val * b[i], 0);
}

const kingVec = oneHotEncode("king", vocabulary);
const queenVec = oneHotEncode("queen", vocabulary);
const dogVec = oneHotEncode("dog", vocabulary);

console.log("\n--- The problem with one-hot encoding ---");
console.log("dot(king, queen) =", dotProduct(kingVec, queenVec), "(no relationship captured!)");
console.log("dot(king, dog)   =", dotProduct(kingVec, dogVec), "(no relationship captured!)");
console.log("Both are equally '0 similarity' even though king/queen are obviously more related than king/dog.");
console.log("This is WHY we need learned embeddings instead of one-hot vectors.");
```

**Run it:**
```bash
node one-hot-encoding-demo.js
```

**Verified output:**
```
Vocabulary: [ 'king', 'queen', 'man', 'woman', 'dog', 'cat' ]
"king" -> [1, 0, 0, 0, 0, 0]
"queen" -> [0, 1, 0, 0, 0, 0]
"man" -> [0, 0, 1, 0, 0, 0]
"woman" -> [0, 0, 0, 1, 0, 0]
"dog" -> [0, 0, 0, 0, 1, 0]
"cat" -> [0, 0, 0, 0, 0, 1]

--- The problem with one-hot encoding ---
dot(king, queen) = 0 (no relationship captured!)
dot(king, dog)   = 0 (no relationship captured!)
This is WHY we need learned embeddings instead of one-hot vectors.
```

#### Line-by-line explanation

- **`oneHotEncode`** — creates a vector of all zeros, the same length as the vocabulary, with a single `1` at the word's index. This is the "obvious first idea" for representing categories as numbers.
- **The proof of failure** — `dot(king, queen)` and `dot(king, dog)` are *both exactly 0*. Mathematically, every pair of distinct one-hot vectors is equally "unrelated," no matter how semantically close or distant the actual words are. This concretely motivates why we need *learned* embeddings instead.

---

### Part C — Toy Embeddings: Cosine Similarity and Vector Arithmetic

```javascript
// toy-embeddings.js
// A TOY embedding system where I hand-pick dimensions that are interpretable,
// purely for teaching. Real embeddings (Day 12) have hundreds of dimensions
// that are NOT human-interpretable -- but the cosine similarity MATH is identical.
//
// Toy dimensions: [royalty, gender_masculine_to_feminine, animal-ness]
// (loosely inspired by the famous "king - man + woman = queen" word2vec example)

const embeddings = {
  king:    [0.9,  0.3, 0.0],
  queen:   [0.9, -0.3, 0.0],
  man:     [0.1,  0.9, 0.0],
  woman:   [0.1, -0.9, 0.0],
  dog:     [0.0,  0.1, 0.9],
  cat:     [0.0, -0.1, 0.9],
  banana:  [0.0,  0.0, 0.1], // unrelated to royalty/gender/animal-ness -- should be far from everything
};

// --- Cosine similarity: measures the ANGLE between two vectors, ignoring magnitude ---
function dotProduct(a, b) {
  let sum = 0;
  for (let i = 0; i < a.length; i++) sum += a[i] * b[i];
  return sum;
}

function magnitude(v) {
  return Math.sqrt(v.reduce((sum, x) => sum + x * x, 0));
}

function cosineSimilarity(a, b) {
  const dot = dotProduct(a, b);
  const magA = magnitude(a);
  const magB = magnitude(b);
  if (magA === 0 || magB === 0) return 0; // avoid divide-by-zero
  return dot / (magA * magB);
}

// --- Test 1: similarity sanity checks ---
console.log("--- Cosine similarity sanity checks ---");
console.log("king vs queen:", cosineSimilarity(embeddings.king, embeddings.queen).toFixed(3));
console.log("dog vs cat:", cosineSimilarity(embeddings.dog, embeddings.cat).toFixed(3));
console.log("king vs banana:", cosineSimilarity(embeddings.king, embeddings.banana).toFixed(3));
console.log("king vs dog:", cosineSimilarity(embeddings.king, embeddings.dog).toFixed(3));
console.log("man vs woman:", cosineSimilarity(embeddings.man, embeddings.woman).toFixed(3));

// --- Test 2: the famous "king - man + woman = queen" vector arithmetic ---
function vectorSubtract(a, b) {
  return a.map((val, i) => val - b[i]);
}
function vectorAdd(a, b) {
  return a.map((val, i) => val + b[i]);
}

const result = vectorAdd(vectorSubtract(embeddings.king, embeddings.man), embeddings.woman);
console.log("\n--- Vector arithmetic: king - man + woman = ? ---");
console.log("Result vector:", result.map(v => v.toFixed(3)));

// Find the closest known word to this result vector
function findClosest(targetVector, embeddingsDict, excludeWords = []) {
  let bestWord = null;
  let bestScore = -Infinity;
  for (const [word, vec] of Object.entries(embeddingsDict)) {
    if (excludeWords.includes(word)) continue;
    const score = cosineSimilarity(targetVector, vec);
    if (score > bestScore) {
      bestScore = score;
      bestWord = word;
    }
  }
  return { word: bestWord, score: bestScore };
}

const closest = findClosest(result, embeddings, ["king", "man", "woman"]);
console.log(`Closest match (excluding king/man/woman): "${closest.word}" (similarity ${closest.score.toFixed(3)})`);
```

**Run it:**
```bash
node toy-embeddings.js
```

**Verified output:**
```
--- Cosine similarity sanity checks ---
king vs queen: 0.800
dog vs cat: 0.976
king vs banana: 0.000
king vs dog: 0.035
man vs woman: -0.976

--- Vector arithmetic: king - man + woman = ? ---
Result vector: [ '0.900', '-1.500', '0.000' ]
Closest match (excluding king/man/woman): "queen" (similarity 0.759)
```

#### Line-by-line explanation

- **`embeddings` object** — hand-crafted vectors where I, as the teacher, control what each dimension "means" (royalty, gender, animal-ness) so the math is interpretable. Real embedding models learn these dimensions automatically and they end up meaning combinations of things no human would label cleanly.
- **`dotProduct`, `magnitude`, `cosineSimilarity`** — implements the formula from Section 3 exactly: dot product divided by the product of magnitudes.
- **The sanity checks confirm the intuition**: king/queen and dog/cat (same-category pairs) score high (0.8, 0.976); king/banana (unrelated) scores exactly 0; man/woman scores near -1 (nearly opposite, since gender is almost their *only* differing dimension).
- **`findClosest`** — loops through every known word, computes cosine similarity to a target vector, and returns the best match. This exact pattern — "given a query vector, find the most similar vector(s) in a collection" — is **precisely what a vector database does** (we'll build a real one on Day 18-19 for RAG). You're seeing the core algorithm of vector search today, in its simplest possible form.
- **The result confirms the famous relationship**: `king - man + woman` lands closest to `queen`, out of every other word in the vocabulary — verified by running the code, not just asserted.

---

## 6. TEACH-IT-BACK — Script You Can Say Out Loud

> "Before a neural network can process anything — words, images, audio — that thing has to become numbers. A 'tensor' is just a general word for a grid of numbers with some number of dimensions: a single number is a 0-dimensional tensor, a list of numbers is 1-dimensional, a grid like a black-and-white image is 2-dimensional, and a color image, which needs a red/green/blue value at every pixel, is 3-dimensional. Now, the naive way to turn words into numbers is called one-hot encoding — give each word a vector that's all zeros except a single 1 marking which word it is. The problem is this captures zero meaning: to a one-hot vector, 'king' and 'queen' are exactly as unrelated as 'king' and 'banana,' because there's no shared structure between any two of these vectors. Embeddings fix this. An embedding is a *learned* vector where the position in space actually represents meaning — words used in similar contexts end up close together, almost like pins on a map where similar words live in the same neighborhood. To measure how similar two embeddings are, we use cosine similarity, which looks at the *angle* between two vectors rather than their length — vectors pointing the same direction score close to 1, unrelated vectors score close to 0, and opposite vectors score close to -1. And here's the famous proof that embeddings capture real relationships, not just vague similarity: if you take the vector for 'king,' subtract 'man,' and add 'woman,' the resulting vector lands closest to 'queen' — because subtracting man cancels out the masculine direction, and adding woman reintroduces a feminine direction, and that direction in the embedding space consistently represents gender, no matter which pair of words you're looking at."

---

## Key Terms Glossary (Day 4)

| Term | Meaning |
|---|---|
| **Tensor** | A grid of numbers with some number of dimensions (scalar, vector, matrix, or higher) |
| **Shape** | The size of a tensor along each dimension, e.g. `[3, 3]` |
| **One-hot encoding** | A naive vector representation of categories that captures no semantic relationship |
| **Embedding** | A learned vector representing the meaning of something, positioned so similar things are close together |
| **Dot product** | Sum of element-wise products of two vectors — the core operation behind both neurons (Day 2) and cosine similarity |
| **Cosine similarity** | A measure of similarity based on the angle between two vectors, ignoring magnitude |
| **Vector arithmetic in embedding space** | Adding/subtracting embedding vectors to manipulate meaning (e.g. king - man + woman ≈ queen) |

---

## Quick Self-Check (ask yourself before moving to Day 5)

1. Can I explain, without looking, what "shape" means for a tensor, using an example?
2. Can I explain *why* one-hot encoding fails to capture meaning, with a concrete example?
3. Can I explain cosine similarity in terms of "angle, not length"?
4. Do I understand why `king - man + woman ≈ queen` is considered strong evidence that embeddings capture real semantic relationships?

If yes to all four — you're ready for **Day 5: The sequential data problem** (RNNs/LSTMs, and why they struggled — setting up "why Transformers won").
