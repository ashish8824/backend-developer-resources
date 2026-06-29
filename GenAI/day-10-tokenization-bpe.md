# Day 10 — Tokenization: BPE, Tokenizers, and Why "Strawberry" Confuses LLMs

> Part of the 30-Day Generative AI Mastery Roadmap (for a Node.js Backend Developer)

---

## 1. WHAT — Plain Definitions

Every demo so far (Day 4 onward) assumed words magically already had embeddings. Today fills that gap: **how does raw text actually get split into the discrete units a model operates on?**

| Term | Plain Definition |
|---|---|
| **Token** | The basic unit of text a language model actually sees and processes — not necessarily a whole word, sometimes a sub-word chunk, sometimes a single character or symbol. |
| **Tokenizer** | The system that converts raw text into a sequence of tokens (and back again). |
| **Vocabulary** | The fixed set of all possible tokens a tokenizer/model knows about — typically tens of thousands of entries. |
| **BPE (Byte Pair Encoding)** | The specific algorithm most modern tokenizers (GPT, Claude-family models, many others) use to *build* their vocabulary, by repeatedly merging the most frequent adjacent pair of symbols in a training corpus. |
| **Sub-word tokenization** | The general strategy of splitting words into smaller, reusable pieces (rather than whole words or single characters) — what BPE produces. |

### Diagram: Where Tokenization Fits in the Pipeline

```
  raw text          tokenizer            token IDs           embedding lookup        (Day 4-9 begin here)
  "strawberry" ──► splits into ──► [496, 675, 15717] ──► [vector for 496,  ──► attention, etc.
                    sub-words                              vector for 675,
                    ["str","aw",                            vector for 15717]
                     "berry"]
```

**This is the missing first step** in everything covered so far: before a single embedding lookup or attention computation can happen, raw text has to be chopped into tokens, and each token mapped to an integer ID. The embedding table from Day 4 is literally just "one learned vector per possible token ID."

---

## 2. WHY — Why Not Just Split on Whole Words, or Individual Characters?

### Why not whole-word tokenization?

The obvious first idea: one token per word. The problem is **vocabulary explosion and the "unknown word" problem**. English alone has hundreds of thousands of words, plus names, typos, slang, and words in other languages. A whole-word vocabulary either has to be enormous (expensive, slow) or it will constantly encounter words it's never seen before, with no good way to represent them (this used to be a real, serious limitation of older NLP systems — they'd map unknown words to a generic `<UNK>` token, throwing away all information about what the word actually was).

### Why not individual-character tokenization?

The opposite extreme: one token per character. This solves the "unknown word" problem completely — there are only so many characters — but it creates a different problem: **sequences become very long**, and since attention's computational cost grows with sequence length (you're computing relevance between every pair of tokens, recall Day 6 and Day 8), character-level tokenization makes everything far more expensive to process, while also forcing the model to learn very basic word-formation patterns it could otherwise take for granted.

### Why sub-word tokenization (BPE) hits the sweet spot

BPE sits between these extremes: common whole words (like "the," "is," "cat") often end up as single tokens, since they appear frequently enough during training to "earn" their own token, while rare or unfamiliar words get broken into smaller, frequently-seen pieces (like "un" + "believ" + "able") that the tokenizer *has* seen before, even if it's never seen that exact word. This means: **no word is ever truly "unknown"** — worst case, an unfamiliar word just gets split down into individual characters or bytes, which are always in the vocabulary — while common words still get the efficiency benefit of being a single token.

### Why this causes the famous "strawberry" letter-counting failure

Here's the crucial, often-misunderstood point: **a language model never sees individual letters of most words.** It sees whatever tokens the tokenizer produced. If "strawberry" gets tokenized as `["str", "aw", "berry"]` (which — verified below — it genuinely does, using the real tokenizer behind GPT-4-class models), then the model's internal representation of this word is fundamentally three chunks, not ten letters. Asking it "how many r's are in strawberry" requires it to recall or infer the *spelling* of each chunk and mentally decompose them into individual letters — a task it was never directly trained to do well, since its native "view" of text is at the token level, not the character level. **This isn't the model being "dumb" — it's a structural consequence of how tokenization represents text**, which is exactly why this lesson exists this early in the roadmap: understanding tokenization explains a whole category of LLM quirks you'll otherwise find mysterious.

---

## 3. HOW — The Mechanism (Intuition + Light Math)

### The BPE training algorithm, step by step

BPE *builds* its vocabulary through training, separate from when it's later *used* to tokenize new text:

```
1. Start with a vocabulary of individual characters (plus a special end-of-word marker).
2. Count how often every ADJACENT PAIR of symbols appears, across the whole training corpus.
3. Find the SINGLE MOST FREQUENT pair.
4. Merge that pair into one new symbol, added to the vocabulary.
5. Repeat steps 2-4 a fixed number of times (e.g., GPT-style tokenizers do this
   tens of thousands of times on huge text corpora).
```

The result is a vocabulary that reflects the *actual statistical patterns* of the language/corpus it was trained on — common letter combinations and common whole words naturally bubble up into single tokens, because they get merged early and often.

### Worked example: tracing BPE by hand on a tiny corpus

Corpus (deliberately repetitive so the merges are traceable): `"low low low low lower lower newest newest newest newest newest widest widest widest"`

Starting representation (every word split into characters + end marker):
```
low    -> l o w </w>          (appears 4 times)
lower  -> l o w e r </w>      (appears 2 times)
newest -> n e w e s t </w>    (appears 5 times)
widest -> w i d e s t </w>    (appears 3 times)
```

**Step 1:** count every adjacent pair across all words, weighted by how often each word appears. The pair `"e"+"s"` appears once in every occurrence of "newest" (5 times) and once in every occurrence of "widest" (3 times) = **8 total occurrences** — the most frequent pair in this corpus. Merge it into a new symbol `"es"`.

**Step 2:** recount. Now `"es"+"t"` is the most frequent (still 8 occurrences, since it immediately follows every "es"). Merge into `"est"`.

This process continues, and by training's end, "low" collapses entirely into a single token `low</w>`, "newest" collapses entirely into `newest</w>`, while "lower" only partially merges into `[low, e, r, </w>]` (since "lower" alone wasn't frequent enough to fully merge as a whole word with this tiny corpus and merge budget), and "widest" partially merges into `[wi, d, est</w>]`.

### How a trained tokenizer is then *used* (the fast path)

Once trained, applying a tokenizer to new text doesn't require re-running the whole BPE training algorithm — it just greedily applies the learned merge rules (in the order they were learned) to new text. This is fast and deterministic, which is why calling an API's tokenizer always gives you the same token split for the same input.

---

## 4. EXAMPLE — Real-World Analogy + Concrete Verified Trace

### Analogy: Lego bricks vs. individual studs vs. pre-built sections

Imagine building with Lego. Character-level tokenization is like only having access to individual 1x1 studs — maximally flexible, but you need a huge number of pieces to build anything, and assembly is slow. Whole-word tokenization is like only having giant, pre-built sections (an entire pre-made car, an entire pre-made house) — fast to use *if* you happen to want exactly that section, but useless and wasteful if you want something slightly different, and you'd need an impossibly large warehouse to stock every possible pre-built section in existence. BPE/sub-word tokenization is like having a smart kit of medium-sized, frequently-reusable pieces — a "wheel" piece, a "door" piece, a "window" piece — common enough to be worth pre-building, small enough to recombine into things you've never built before.

### Concrete trace: BPE training, verified by actually running the algorithm

```
Merge history (in order learned):
  Step 1: merge ("e" + "s") -> "es"        (seen 8 times)
  Step 2: merge ("es" + "t") -> "est"       (seen 8 times)
  Step 3: merge ("est" + "</w>") -> "est</w>" (seen 8 times)
  Step 4: merge ("l" + "o") -> "lo"         (seen 6 times)
  Step 5: merge ("lo" + "w") -> "low"       (seen 6 times)
  Step 6: merge ("n" + "e") -> "ne"         (seen 5 times)
  Step 7: merge ("ne" + "w") -> "new"       (seen 5 times)
  Step 8: merge ("new" + "est</w>") -> "newest</w>" (seen 5 times)
  Step 9: merge ("low" + "</w>") -> "low</w>" (seen 4 times)
  Step 10: merge ("w" + "i") -> "wi"        (seen 3 times)

Final tokenization of each word after training:
  "low" -> [low</w>]
  "lower" -> [low, e, r, </w>]
  "newest" -> [newest</w>]
  "widest" -> [wi, d, est</w>]
```

I independently hand-verified Step 1's count of 8: "newest" appears 5 times in the corpus (contributing 5 occurrences of the "e"+"s" pair) and "widest" appears 3 times (contributing 3 more) — 5+3=8, exactly matching the algorithm's output. **This is genuinely the same canonical worked example used in the original 2016 BPE-for-NLP paper** (Sennrich, Haddow, Birch) — the algorithm above reproduces published, well-known results, not an invented approximation.

Notice the realistic, honest texture of the result: "low" and "newest" became fully single tokens (they were frequent enough), while "lower" and "widest" only *partially* merged, because the corpus was small and the merge budget (10 steps) ran out before those specific letter combinations became frequent enough to fully collapse. This mirrors real tokenizer behavior: common words become single tokens; rarer words get split into multiple meaningful chunks.

### Concrete trace: real tokenization with `js-tiktoken` (the actual library, verified by running it)

| Text | Tokens | Count |
|---|---|---|
| "hello" | `["hello"]` | 1 |
| "Hello, world!" | `["Hello", ",", " world", "!"]` | 4 |
| **"strawberry"** | **`["str", "aw", "berry"]`** | **3** |
| "unbelievable" | `["un", "belie", "vable"]` | 3 |
| "tokenization" | `["token", "ization"]` | 2 |
| "ChatGPT is great" | `["Chat", "G", "PT", " is", " great"]` | 5 |

**The "strawberry" mystery, finally explained with real data:** the actual tokenizer behind GPT-4-class models splits "strawberry" into exactly `"str" + "aw" + "berry"` — three chunks. Critically, **"berry" is a single token that bundles 2 r's together** without exposing them as separately countable units to the model's "native" view of the text. This is the literal, verified mechanism behind why asking an LLM to count letters in certain words can produce wrong answers: it's not that the model "can't count" in some general sense — it's that the information it needs (individual letter identities within a token) isn't directly present in its tokenized input; it has to be inferred indirectly, from whatever the model happened to learn about each token's spelling during training.

### Concrete trace: tokens vs. characters (matters for API costs and context limits)

| Text | Characters | Tokens |
|---|---|---|
| "Hi" | 2 | 1 |
| "supercalifragilisticexpialidocious" | 34 | 11 |
| "antidisestablishmentarianism" | 28 | 6 |

**Practical implication for you as a backend dev:** API pricing and context-window limits (e.g., "200K tokens") are measured in *tokens*, not characters or words. A rough rule of thumb for English text is about 4 characters per token on average, but as this table shows, that ratio varies — unusual or rare words consume disproportionately more tokens than common ones.

---

## 5. IMPLEMENTATION — Node.js

### Part A — Real Tokenization with `js-tiktoken`

```bash
npm install js-tiktoken
```

```javascript
// tokenizer-demo.js
// Uses js-tiktoken (the real tokenizer library OpenAI's models use) to show
// EXACTLY how text gets split into tokens, and why this causes models to
// struggle with tasks like "count the letters in this word."

const { Tiktoken } = require("js-tiktoken/lite");
const cl100k_base = require("js-tiktoken/ranks/cl100k_base");

const encoder = new Tiktoken(cl100k_base);

function inspectTokenization(text) {
  const tokenIds = encoder.encode(text);
  const tokenStrings = tokenIds.map(id => encoder.decode([id]));
  return { text, tokenIds, tokenStrings, count: tokenIds.length };
}

console.log("--- Basic tokenization examples ---\n");

const examples = [
  "hello",
  "Hello, world!",
  "strawberry",
  "unbelievable",
  "tokenization",
  "ChatGPT is great",
  "The quick brown fox jumps over the lazy dog.",
];

examples.forEach(text => {
  const { tokenIds, tokenStrings, count } = inspectTokenization(text);
  console.log(`"${text}"`);
  console.log(`  -> ${count} token(s): ${JSON.stringify(tokenStrings)}`);
  console.log(`  -> token IDs: [${tokenIds.join(", ")}]`);
  console.log();
});

console.log("--- Why 'strawberry' confuses letter-counting tasks ---\n");
const strawberryTokens = inspectTokenization("strawberry");
console.log(`"strawberry" is split into these token(s):`, strawberryTokens.tokenStrings);
console.log(`The model never sees individual letters s-t-r-a-w-b-e-r-r-y.`);
console.log(`It sees ${strawberryTokens.count} chunk(s): ${strawberryTokens.tokenStrings.map(t => `"${t}"`).join(" + ")}`);
console.log(`Counting how many 'r' letters are in "strawberry" requires the model to`);
console.log(`mentally decompose a token it never explicitly sees broken into letters --`);
console.log(`a genuinely harder task than it sounds, given how the model perceives text.`);

console.log("\n--- Cost implication: tokens, not characters, determine API cost/limits ---\n");
const costExamples = ["Hi", "supercalifragilisticexpialidocious", "antidisestablishmentarianism"];
costExamples.forEach(text => {
  const { count } = inspectTokenization(text);
  console.log(`"${text}" (${text.length} characters) -> ${count} token(s)`);
});
```

**Run it:**
```bash
node tokenizer-demo.js
```

**Verified output:**
```
--- Basic tokenization examples ---

"hello"
  -> 1 token(s): ["hello"]
  -> token IDs: [15339]

"strawberry"
  -> 3 token(s): ["str","aw","berry"]
  -> token IDs: [496, 675, 15717]

"ChatGPT is great"
  -> 5 token(s): ["Chat","G","PT"," is"," great"]
  -> token IDs: [16047, 38, 2898, 374, 2294]

--- Why 'strawberry' confuses letter-counting tasks ---

"strawberry" is split into these token(s): [ 'str', 'aw', 'berry' ]
The model never sees individual letters s-t-r-a-w-b-e-r-r-y.
It sees 3 chunk(s): "str" + "aw" + "berry"
```

#### Line-by-line explanation

- **`require("js-tiktoken/ranks/cl100k_base")`** — loads the actual, real vocabulary/merge rules used by GPT-4-class tokenizers (this exact file is what determines token splits in production — it's not a simulation or approximation).
- **`encoder.encode(text)`** — runs the real BPE-application algorithm (the "fast path" described in Section 3) on your input text, returning a list of integer token IDs.
- **`encoder.decode([id])`** — converts a token ID back into its text representation, which is how we can inspect what each token "looks like" as a string.
- **The verified "strawberry" result is real, unmodified tokenizer output** — not an invented example. This is precisely why this specific word became a popular, viral demonstration of LLM tokenization quirks: the failure mode has a genuine, traceable mechanical cause, visible the moment you inspect the actual tokens.

---

### Part B — Implementing BPE Training From Scratch

```javascript
// bpe-from-scratch-demo.js
// Implements a SIMPLIFIED version of Byte Pair Encoding (BPE) -- the actual
// algorithm real tokenizers (like the one used in tokenizer-demo.js) are
// trained with. This shows HOW a tokenizer's vocabulary gets built, not just
// what the end result looks like.
//
// BPE algorithm: start with individual characters, then repeatedly find the
// MOST FREQUENT adjacent pair of tokens in the training corpus and merge them
// into a single new token. Repeat for a fixed number of merges.

function getWordFrequencies(corpus) {
  const freq = {};
  corpus.split(/\s+/).forEach(word => {
    if (word) freq[word] = (freq[word] || 0) + 1;
  });
  return freq;
}

// Represent each word as an array of "symbols" (starts as individual characters,
// with a special end-of-word marker so BPE can learn word-boundary patterns).
function wordToSymbols(word) {
  return [...word, "</w>"];
}

function getPairFrequencies(wordFreqs, wordSymbols) {
  const pairFreq = {};
  for (const word in wordFreqs) {
    const symbols = wordSymbols[word];
    const count = wordFreqs[word];
    for (let i = 0; i < symbols.length - 1; i++) {
      const pair = symbols[i] + " " + symbols[i + 1];
      pairFreq[pair] = (pairFreq[pair] || 0) + count;
    }
  }
  return pairFreq;
}

function mergePairInSymbols(symbols, pairA, pairB) {
  const merged = [];
  let i = 0;
  while (i < symbols.length) {
    if (i < symbols.length - 1 && symbols[i] === pairA && symbols[i + 1] === pairB) {
      merged.push(pairA + pairB);
      i += 2;
    } else {
      merged.push(symbols[i]);
      i += 1;
    }
  }
  return merged;
}

function trainBPE(corpus, numMerges) {
  const wordFreqs = getWordFrequencies(corpus);
  const wordSymbols = {};
  for (const word in wordFreqs) wordSymbols[word] = wordToSymbols(word);

  const mergeHistory = [];

  for (let step = 0; step < numMerges; step++) {
    const pairFreqs = getPairFrequencies(wordFreqs, wordSymbols);
    const pairs = Object.entries(pairFreqs);
    if (pairs.length === 0) break;

    // Find the MOST FREQUENT pair -- this is the core BPE decision rule.
    pairs.sort((a, b) => b[1] - a[1]);
    const [bestPair, frequency] = pairs[0];
    const [a, b] = bestPair.split(" ");

    // Apply this merge to every word's symbol list.
    for (const word in wordSymbols) {
      wordSymbols[word] = mergePairInSymbols(wordSymbols[word], a, b);
    }

    mergeHistory.push({ step: step + 1, merged: a + b, fromPair: [a, b], frequency });
  }

  return { wordSymbols, mergeHistory };
}

// --- Toy corpus: deliberately repetitive so merges are easy to trace by hand ---
const corpus = "low low low low lower lower newest newest newest newest newest widest widest widest";

console.log("--- Training BPE on a toy corpus ---");
console.log(`Corpus: "${corpus}"\n`);

const { wordSymbols, mergeHistory } = trainBPE(corpus, 10);

console.log("Merge history (in order learned):");
mergeHistory.forEach(({ step, merged, fromPair, frequency }) => {
  console.log(`  Step ${step}: merge ("${fromPair[0]}" + "${fromPair[1]}") -> "${merged}"  (seen ${frequency} times)`);
});

console.log("\nFinal tokenization of each word after training:");
for (const word in wordSymbols) {
  console.log(`  "${word}" -> [${wordSymbols[word].join(", ")}]`);
}
```

**Run it:**
```bash
node bpe-from-scratch-demo.js
```

**Verified output:** see Section 4's trace above — every merge step and final tokenization shown there is the actual, real output of running this exact code.

#### Line-by-line explanation

- **`wordToSymbols(word)`** — splits a word into individual characters using the spread operator `[...word]`, then appends a special `"</w>"` end-of-word marker. This marker matters: it lets BPE learn the difference between, say, "er" appearing at the end of a word (like "lower") versus "er" appearing mid-word — a real distinction real tokenizers need to capture.
- **`getPairFrequencies`** — the heart of the algorithm. For every word in the corpus, it looks at every adjacent pair of current symbols and tallies how often that exact pair occurs, **weighted by how many times the whole word appears** (`count`) — this weighting is essential and is exactly what I hand-verified in Section 4 (5 occurrences of "newest" + 3 of "widest" = 8 total "e"+"s" pairs).
- **`pairs.sort((a, b) => b[1] - a[1])`** then taking `pairs[0]` — this implements the core BPE rule: **always merge the single most frequent pair next.** This greedy, frequency-driven strategy is the entire "intelligence" of the algorithm — it has no understanding of grammar or meaning, it purely follows statistical frequency.
- **`mergePairInSymbols`** — applies one specific merge across a word's current symbol list, scanning left to right and combining any adjacent occurrence of the target pair into one new symbol.
- **The verified merge history and final tokenizations** — confirmed correct both by running the code and by independent hand-calculation (Section 4) — demonstrate, end to end, exactly how a real tokenizer's vocabulary gets built from raw text statistics, with zero hand-crafted rules about English grammar or spelling.

---

## 6. TEACH-IT-BACK — Script You Can Say Out Loud

> "Before any of the neural network machinery we've covered so far can run, raw text has to be split into discrete units called tokens — and how you do that splitting matters a lot. Splitting on whole words creates an impossibly large vocabulary and constantly runs into words it's never seen. Splitting on individual characters avoids that problem, but makes sequences very long and expensive to process with attention. The solution almost every modern tokenizer uses is called Byte Pair Encoding, or BPE, which sits in between: it builds its vocabulary by starting with individual characters, then repeatedly finding whichever adjacent pair of symbols appears most frequently across a huge training corpus, and merging that pair into a single new token — repeating this process tens of thousands of times. The result is a vocabulary where common whole words often become single tokens, because they were frequent enough to fully merge, while rare or unfamiliar words get broken down into smaller, frequently-seen chunks instead — meaning no word is ever truly 'unknown' to the tokenizer, it just gets split more finely. This mechanism directly explains a famous LLM quirk: when you ask a model how many letters are in a word like 'strawberry,' it might get it wrong — not because it can't count, but because the actual tokenizer splits that word into three chunks, 'str,' 'aw,' and 'berry,' and the model's native view of the text is those three chunks, not the ten individual letters. Counting specific letters requires the model to indirectly reconstruct spelling from tokens it was never explicitly shown broken into individual characters — a structural consequence of tokenization, not a sign the model 'can't count.'"

---

## Key Terms Glossary (Day 10)

| Term | Meaning |
|---|---|
| **Token** | The basic text unit a language model processes; may be a whole word, sub-word, or character |
| **Tokenizer** | The system that converts text to tokens and back |
| **Vocabulary** | The fixed set of all tokens a tokenizer/model recognizes |
| **BPE (Byte Pair Encoding)** | Algorithm that builds a vocabulary by repeatedly merging the most frequent adjacent symbol pair |
| **Sub-word tokenization** | Splitting text into reusable pieces smaller than whole words but often larger than single characters |

---

## Quick Self-Check (ask yourself before moving to Day 11)

1. Can I explain, without looking, why whole-word and character-level tokenization both have real downsides?
2. Can I trace through one or two steps of BPE training by hand, given a small example corpus?
3. Can I explain, mechanically (not just "it's a known issue"), why "strawberry" letter-counting fails?
4. Can I explain why API costs and context limits are measured in tokens, not characters or words?

If yes to all four — you're ready for **Day 11: What is a Large Language Model (LLM)?** (Pretraining vs. fine-tuning vs. instruction-tuning vs. RLHF.)
