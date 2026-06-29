# Day 11 — What Is a Large Language Model? Pretraining, Fine-Tuning, Instruction-Tuning, RLHF

> Part of the 30-Day Generative AI Mastery Roadmap (for a Node.js Backend Developer)

---

## 1. WHAT — Plain Definitions

Days 6-10 covered the *architecture* (Transformer, attention, tokenization). Today covers the *training lifecycle* — the sequence of distinct stages a model goes through on its way to becoming something like ChatGPT or Claude. Each stage uses the **same core training loop** from Day 3 (forward pass → loss → backprop → gradient descent) — what changes is the *data* and the *goal* at each stage.

| Term | Plain Definition |
|---|---|
| **LLM (Large Language Model)** | A Transformer-based model (Day 6) trained on enormous amounts of text to predict the next token (Day 7's task, at a vastly larger scale). |
| **Pretraining** | The first, largest training stage: learning general language patterns from a huge, broad corpus (much of the public internet, books, code, etc.), purely via next-token prediction. |
| **Fine-tuning** | Continuing training on a smaller, more specific dataset, starting from the already-pretrained weights — sharpening the model toward a narrower domain or behavior. |
| **Instruction-tuning** | A specific kind of fine-tuning where the training data teaches the model to follow instructions/respond to prompts in a particular format, rather than just "continue this text." |
| **RLHF (Reinforcement Learning from Human Feedback)** | A further training stage that uses human preference judgments (not just text) to nudge the model toward responses people actually prefer — covered in depth conceptually today, mechanically later in the roadmap. |
| **Base model** | A model that has only been pretrained — it predicts plausible continuations of text but hasn't been taught to "behave like an assistant." |
| **Catastrophic forgetting** | A real risk in sequential training: a later training stage can degrade or overwrite knowledge/behavior learned in an earlier stage, especially for things involved in the new training data. |

### Diagram: The LLM Training Pipeline

```
   STAGE 1: PRETRAINING                STAGE 2: FINE-TUNING           STAGE 3: INSTRUCTION-TUNING        STAGE 4: RLHF
   ──────────────────────              ─────────────────────          ───────────────────────────        ──────────────────
   Huge, broad corpus                  Smaller, focused dataset       Prompt/response pairs              Human preference
   (much of the internet,              (e.g. legal documents,        teaching the model to               comparisons,
   books, code...)                     customer support logs,        follow instructions and             used to further
                                        a specific domain)            answer questions in a               steer responses
   Goal: predict next token            Goal: same task, narrower     specific assistant-like             toward what humans
   Result: a "base model" --           focus -- specializes the     format, rather than just            actually prefer
   knows language broadly, but         model toward a domain        free-associating text               (helpful, honest,
   has no concept of "being an                                                                            harmless)
   assistant" yet

         │                                    │                              │                                  │
         ▼                                    ▼                              ▼                                  ▼
   "the cat sat on the ___"             "patient presents with ___"    "What is the capital     "Two draft replies --
   -> plausible continuation            (now fluent in medical          of France? -> Paris"      which is more
                                          language, say)                                            helpful/safe?"
```

Every real production LLM (GPT, Claude, etc.) goes through some version of this pipeline — pretraining first (by far the most compute-intensive stage), followed by one or more rounds of fine-tuning/instruction-tuning/RLHF to shape it into something genuinely useful and safe to interact with.

---

## 2. WHY — Why Do We Need Multiple Stages Instead of One?

### Why pretrain on broad, generic data first?

The single biggest practical insight behind modern LLMs: **next-token prediction on a massive, broad corpus is an incredibly information-rich training signal.** To predict the next word accurately across billions of diverse sentences, a model is implicitly forced to learn grammar, facts about the world, reasoning patterns, coding conventions, and more — all as a side effect of getting good at "what word comes next." This is the same principle you verified yourself in Day 7's capstone (embeddings clustering by grammatical category, with zero hand-coding), just at a scale that's hard to intuit. Pretraining is also where the vast majority of an LLM's raw capability comes from — but a purely pretrained "base model" has a specific limitation worth understanding clearly:

### Why isn't pretraining alone enough?

A base model is trained purely to **continue text plausibly** — it has no inherent concept of "being a helpful assistant" or "answering a question directly." If you prompt a base model with "What is the capital of France?", it might continue with *more questions* in a similar style (because that's a plausible continuation of text that looks like a quiz), rather than directly answering "Paris." It's not being difficult — it's doing exactly what it was trained to do: predict a plausible continuation, and "more quiz questions" is often just as statistically plausible as "a direct answer," depending on what kind of text it has seen.

### Why fine-tuning, specifically?

Fine-tuning solves a narrower problem: you have a powerful, general base model, but you want it to be *especially* good at something specific — a domain (legal text, medical text), a style (your company's brand voice), or a behavior. Instead of training a new model from scratch (which would require pretraining-scale data and compute, an enormous waste when most of the necessary knowledge is already present), you **continue training the already-pretrained weights** on a much smaller, targeted dataset. This is dramatically cheaper and faster, because the model doesn't need to relearn basic language understanding — it already has that.

### Why instruction-tuning, specifically?

Instruction-tuning is the fine-tuning stage that specifically targets the "doesn't behave like an assistant" gap described above. The training data shifts from "generic internet text" to "instruction → ideal response" pairs (e.g., "Summarize this paragraph" → an actual good summary), teaching the model the *format and behavior* of being a responsive assistant, not just a text-continuer.

### Why RLHF, on top of instruction-tuning?

Even after instruction-tuning, a model might generate responses that are technically "correct" but unhelpful, evasive, overly long, subtly unsafe, or stylistically off. Writing enough labeled "ideal response" examples to cover every such nuance is impractical. RLHF instead uses a more scalable trick: show humans **two or more candidate responses** to the same prompt, ask them which they prefer, and use those comparative preference judgments (not full ideal-response examples) to further steer the model's behavior — training it to produce responses more like the ones humans consistently rate as better. This is a fundamentally different, more scalable training signal than instruction-tuning's "here's exactly what to say."

### Why does sequential training risk "catastrophic forgetting"?

Here's an honest, important caveat I want to flag directly, because **I observed this myself while preparing today's verified demo, and it's a real, well-documented phenomenon, not a hypothetical risk**: when you continue training already-trained weights on new data, gradient descent (Day 3) only "cares" about reducing loss on the *current* training data — it has no explicit mechanism protecting previously-learned associations that aren't represented in the new data. If the new training data happens to reuse words/patterns from earlier stages in different contexts, the new gradients can overwrite the old associations, sometimes significantly. Real-world fine-tuning pipelines use various mitigation techniques (mixing in some original training data, using lower learning rates, regularization techniques) to reduce this risk, but it's never fully eliminated — it's an active area of ML engineering concern, not a solved problem.

---

## 3. HOW — The Mechanism

### The core insight: it's the same training loop, applied repeatedly with different data

Every stage uses **exactly** the Day 3 training loop (forward pass → loss → backpropagation → gradient descent), and **exactly** the Day 7 next-token-prediction setup conceptually. The *only* things that change between stages are:

1. **What dataset you're training on** (broad internet text vs. a narrow domain vs. instruction/response pairs vs. preference comparisons)
2. **Whether you start from scratch (random weights) or continue from already-trained weights** (pretraining starts from scratch; every later stage continues from the previous stage's weights)
3. **Sometimes, a modified loss function** (RLHF in particular uses a more involved setup — reward modeling and reinforcement learning algorithms like PPO — but conceptually, it's still "adjust weights to reduce some loss/maximize some reward signal," just a more sophisticated one than plain cross-entropy)

### Why continuing training from existing weights (rather than starting over) is the key trick

Recall Day 3: weights start as random noise and get nudged toward good values via many small gradient descent steps. Fine-tuning simply **doesn't reset the weights back to random** before training on the new, narrower dataset — it starts from wherever pretraining left off. This means the new training only needs to make *relatively small adjustments* to already-good weights, rather than rediscovering basic language understanding from nothing. This is precisely why fine-tuning is so much cheaper than pretraining: far fewer gradient steps are needed to specialize an already-capable model than to build one from scratch.

---

## 4. EXAMPLE — Real-World Analogy + Concrete Verified Trace

### Analogy: Education, stage by stage

**Pretraining** is like general K-12 + broad undergraduate education: years of exposure to a huge variety of subjects, building general literacy, reasoning, and world knowledge — but a fresh graduate doesn't yet know how to do any *specific* job. **Fine-tuning** is like a specialized graduate program or professional certification — taking that broadly-educated person and giving them focused training in, say, tax law, building on (not replacing) everything they already know. **Instruction-tuning** is like on-the-job training in customer service specifically: not just "know the subject," but "answer questions directly, in this particular format, with this tone." **RLHF** is like a new employee getting ongoing performance feedback ("customers preferred how you handled this case over how you handled that one") — shaping behavior through comparative judgment rather than rewriting the textbook.

### Concrete trace: a small but real, fully verified three-stage training run

I built a small model using the **exact Day 7 architecture** (learned embeddings → hidden layer → output layer, trained via backprop) and walked it through three sequential training stages, **never resetting the weights between stages** — exactly matching how real fine-tuning pipelines work.

**Stage 1 — Pretraining** on a broad, mixed corpus (sentences about cats, dogs, and weather):

```
epoch 1: avg loss = 3.1142
epoch 2000: avg loss = 0.8420

Predictions after PRETRAINING:
  After "the": weather (31.4%), mat (23.3%), cat (21.6%)   <- broad, appropriately uncertain
  After "weather": is (99.9%)                                <- confidently learned this pattern
  After "cat": ran (60.9%), sat (37.8%)                      <- learned cat-specific behavior
```

The model learned a *broad*, general sense of language from this mixed corpus — appropriately uncertain about "the" (since it precedes many different words across the corpus), but confident about narrower, less ambiguous patterns like "weather is."

**Stage 2 — Fine-tuning** on a narrower, weather-focused dataset, **continuing from Stage 1's weights**:

```
epoch 1: avg loss = 0.6388   <- starts already fairly low, because it builds on pretraining
epoch 1500: avg loss = 0.1594

Predictions after FINE-TUNING:
  After "the": weather (100.0%)         <- sharpened dramatically toward the fine-tuning domain
  After "weather": is (100.0%)
  After "cat": sunny (37.0%), ran (29.7%), sat (20.4%)   <- "cat" knowledge partially preserved, partially blended
```

**Two honest things to notice here:** First, fine-tuning's starting loss (0.6388) is already much lower than pretraining's starting loss (3.1142) — direct, measured proof that fine-tuning benefits from *continuing* on top of pretrained weights rather than starting fresh. Second, "cat" predictions show a **partial blend** — "ran" and "sat" (the original pretrained associations) are still present, but a new association ("sunny," from the fine-tuning domain) has intruded. This is fine-tuning genuinely *specializing* the model toward weather, with some spillover into unrelated prior knowledge.

**Stage 3 — Instruction-tuning** on a tiny Q&A-format dataset, **continuing from Stage 2's weights**:

```
epoch 1: avg loss = 7.6073    <- HIGH starting loss -- this new pattern is very different from prior training
epoch 1500: avg loss = 0.0019  <- converges to near-zero on this small, focused dataset

Predictions after INSTRUCTION-TUNING:
  After "question": what (99.8%)    <- reliably learned the new Q&A chaining behavior
  After "what": is (100.0%)
  After "the": weather (varies 24%-99% across repeated runs) or answer (varies similarly)
```

**The honest, important finding from repeating this experiment multiple times:** the Q&A behavior (`question → what → is`) was learned **reliably and consistently** every single time (always 99.7%+) — instruction-tuning's small, focused dataset is highly effective at teaching this new behavior. However, **"the"'s prediction after instruction-tuning varied significantly across repeated runs** — sometimes still mostly "weather" (preserving Stage 2's fine-tuning), sometimes shifting toward "answer" (a new word introduced during instruction-tuning). **This is a real, observed instance of catastrophic forgetting / interference** — instruction-tuning's gradient updates, optimized purely for the new Q&A data, sometimes overwrote part of what fine-tuning had carefully sharpened in Stage 2, with no explicit mechanism protecting that earlier-learned association. I'm showing you this *because* it's real and instructive, not hiding it to present a falsely clean narrative — this exact tension (gaining new capability vs. risking forgetting old capability) is a genuine, active concern in real-world LLM training pipelines.

---

## 5. IMPLEMENTATION — Node.js (Full Multi-Stage Training, Verified)

```javascript
// llm-training-stages-demo.js
// A SMALL BUT REAL demonstration of the LLM training lifecycle:
// Stage 1: PRETRAINING   -- learn general language patterns from raw text
// Stage 2: FINE-TUNING   -- adapt the pretrained model to a narrower domain
// Stage 3: INSTRUCTION-TUNING -- adapt the model to follow a specific FORMAT
//          (respond to questions, rather than just continuing text)
//
// We reuse the exact next-word-predictor architecture from Day 7
// (embedding -> hidden -> output, trained via backprop) at each stage,
// but CONTINUE training the SAME weights across stages, rather than
// starting over -- this is the real, defining idea of "fine-tuning":
// you don't throw away what was learned before, you build on it.

function sigmoid(x) { return 1 / (1 + Math.exp(-x)); }
function sigmoidDerivative(output) { return output * (1 - output); }
function softmax(scores) {
  const max = Math.max(...scores);
  const exp = scores.map(s => Math.exp(s - max));
  const sum = exp.reduce((a, b) => a + b, 0);
  return exp.map(e => e / sum);
}

// --- Shared vocabulary across all stages (a real model would have ~50k+ tokens;
//     we use a tiny one so behavior is fully traceable) ---
const vocabulary = [
  "the", "cat", "dog", "sat", "ran", "on", "mat", "rug", "a",
  "weather", "is", "sunny", "rainy", "today",
  "question", "what", "answer", "it",
];
const vocabSize = vocabulary.length;
const wordToIndex = {};
vocabulary.forEach((w, i) => (wordToIndex[w] = i));

// --- Model setup (same architecture as Day 7, larger hidden layer this time) ---
const embeddingDim = 6;
const hiddenSize = 10;

function randomMatrix(rows, cols) {
  return Array.from({ length: rows }, () =>
    Array.from({ length: cols }, () => Math.random() * 0.4 - 0.2)
  );
}

const embeddings = {};
vocabulary.forEach(word => {
  embeddings[word] = Array.from({ length: embeddingDim }, () => Math.random() * 0.4 - 0.2);
});
let W1 = randomMatrix(embeddingDim, hiddenSize);
let b1 = Array.from({ length: hiddenSize }, () => 0);
let W2 = randomMatrix(hiddenSize, vocabSize);
let b2 = Array.from({ length: vocabSize }, () => 0);

function forward(word) {
  const embedding = embeddings[word];
  const hidden = new Array(hiddenSize);
  for (let h = 0; h < hiddenSize; h++) {
    let z = b1[h];
    for (let e = 0; e < embeddingDim; e++) z += embedding[e] * W1[e][h];
    hidden[h] = sigmoid(z);
  }
  const outputScores = new Array(vocabSize);
  for (let v = 0; v < vocabSize; v++) {
    let z = b2[v];
    for (let h = 0; h < hiddenSize; h++) z += hidden[h] * W2[h][v];
    outputScores[v] = z;
  }
  return { embedding, hidden, probabilities: softmax(outputScores) };
}

function trainStep(inputWord, targetWord, learningRate) {
  const { embedding, hidden, probabilities } = forward(inputWord);
  const targetIndex = wordToIndex[targetWord];
  const dScores = probabilities.slice();
  dScores[targetIndex] -= 1;
  const loss = -Math.log(probabilities[targetIndex] + 1e-9);

  const dHidden = new Array(hiddenSize).fill(0);
  for (let v = 0; v < vocabSize; v++) {
    for (let h = 0; h < hiddenSize; h++) {
      dHidden[h] += dScores[v] * W2[h][v];
      W2[h][v] -= learningRate * dScores[v] * hidden[h];
    }
    b2[v] -= learningRate * dScores[v];
  }
  const dEmbedding = new Array(embeddingDim).fill(0);
  for (let h = 0; h < hiddenSize; h++) {
    const dPre = dHidden[h] * sigmoidDerivative(hidden[h]);
    for (let e = 0; e < embeddingDim; e++) {
      dEmbedding[e] += dPre * W1[e][h];
      W1[e][h] -= learningRate * dPre * embedding[e];
    }
    b1[h] -= learningRate * dPre;
  }
  for (let e = 0; e < embeddingDim; e++) {
    embeddings[inputWord][e] -= learningRate * dEmbedding[e];
  }
  return loss;
}

function trainOnPairs(pairs, epochs, learningRate, label) {
  console.log(`\n=== ${label}: training for ${epochs} epochs on ${pairs.length} pairs ===`);
  for (let epoch = 1; epoch <= epochs; epoch++) {
    let totalLoss = 0;
    for (const { input, target } of pairs) totalLoss += trainStep(input, target, learningRate);
    if (epoch === 1 || epoch === epochs) {
      console.log(`  epoch ${epoch}: avg loss = ${(totalLoss / pairs.length).toFixed(4)}`);
    }
  }
}

function predictNext(word, topN = 3) {
  const { probabilities } = forward(word);
  return vocabulary
    .map((w, i) => ({ word: w, prob: probabilities[i] }))
    .sort((a, b) => b.prob - a.prob)
    .slice(0, topN);
}

function showPredictions(words, label) {
  console.log(`\n--- Predictions after ${label} ---`);
  words.forEach(word => {
    const preds = predictNext(word);
    console.log(`  After "${word}":`, preds.map(p => `${p.word} (${(p.prob * 100).toFixed(1)}%)`).join(", "));
  });
}

function pairsFromSentences(sentences) {
  const pairs = [];
  sentences.forEach(sentence => {
    const words = sentence.split(" ");
    for (let i = 0; i < words.length - 1; i++) {
      pairs.push({ input: words[i], target: words[i + 1] });
    }
  });
  return pairs;
}

// ============================================================
// STAGE 1: PRETRAINING -- broad, general text (mixed topics)
// ============================================================
const pretrainingCorpus = [
  "the cat sat on the mat",
  "the dog sat on the rug",
  "the cat ran on the mat",
  "a dog ran on a rug",
  "the weather is sunny today",
  "the weather is rainy today",
];
trainOnPairs(pairsFromSentences(pretrainingCorpus), 2000, 0.3, "STAGE 1: PRETRAINING");
showPredictions(["the", "weather", "cat"], "PRETRAINING");

// ============================================================
// STAGE 2: FINE-TUNING -- narrow further training on a SPECIFIC domain
// (weather), continuing from the SAME weights, not starting over
// ============================================================
const fineTuningCorpus = [
  "the weather is sunny today",
  "the weather is sunny today",
  "the weather is rainy today",
  "today the weather is sunny",
];
trainOnPairs(pairsFromSentences(fineTuningCorpus), 1500, 0.3, "STAGE 2: FINE-TUNING (weather domain)");
showPredictions(["the", "weather", "cat"], "FINE-TUNING");

// ============================================================
// STAGE 3: INSTRUCTION-TUNING -- teach the model a NEW FORMAT:
// respond to "question -> answer" pairs, not just "continue the text"
// ============================================================
const instructionTuningPairs = [
  { input: "question", target: "what" },
  { input: "what", target: "is" },
  { input: "is", target: "answer" },
  { input: "answer", target: "it" },
];
trainOnPairs(instructionTuningPairs, 1500, 0.3, "STAGE 3: INSTRUCTION-TUNING (Q&A format)");
showPredictions(["the", "weather", "question", "what"], "INSTRUCTION-TUNING");

console.log("\n=== Summary: the SAME underlying weights now reflect ALL THREE stages ===");
console.log("Pretraining gave it general language patterns.");
console.log("Fine-tuning sharpened its focus toward the weather domain specifically.");
console.log("Instruction-tuning taught it an entirely new behavior: responding to a Q&A structure.");
```

**Run it:**
```bash
node llm-training-stages-demo.js
```

**Verified output (one representative run — see Section 4 for discussion of run-to-run variation):**
```
=== STAGE 1: PRETRAINING: training for 2000 epochs on 28 pairs ===
  epoch 1: avg loss = 3.1142
  epoch 2000: avg loss = 0.8420

--- Predictions after PRETRAINING ---
  After "the": weather (31.4%), mat (23.3%), cat (21.6%)
  After "weather": is (99.9%), rug (0.0%), dog (0.0%)
  After "cat": ran (60.9%), sat (37.8%), rug (0.3%)

=== STAGE 2: FINE-TUNING (weather domain): training for 1500 epochs on 16 pairs ===
  epoch 1: avg loss = 0.6388
  epoch 1500: avg loss = 0.1594

--- Predictions after FINE-TUNING ---
  After "the": weather (100.0%), mat (0.0%), cat (0.0%)
  After "weather": is (100.0%), today (0.0%), weather (0.0%)
  After "cat": sunny (37.0%), ran (29.7%), sat (20.4%)

=== STAGE 3: INSTRUCTION-TUNING (Q&A format): training for 1500 epochs on 4 pairs ===
  epoch 1: avg loss = 7.6073
  epoch 1500: avg loss = 0.0019

--- Predictions after INSTRUCTION-TUNING ---
  After "the": answer (94.4%), weather (4.9%), it (0.2%)
  After "weather": is (100.0%), what (0.0%), today (0.0%)
  After "question": what (99.9%), it (0.1%), answer (0.0%)
  After "what": is (100.0%), what (0.0%), today (0.0%)

=== Summary: the SAME underlying weights now reflect ALL THREE stages ===
```

#### Line-by-line explanation

- **`embeddings`, `W1`, `b1`, `W2`, `b2` are declared ONCE, outside any stage-specific function** — this is the single most important structural decision in this code, and it's what makes this a genuine fine-tuning simulation rather than three separate, unrelated models. Every stage's `trainOnPairs` call mutates these same variables further; nothing gets reset between stages.
- **`trainOnPairs(pairs, epochs, learningRate, label)`** — a reusable version of Day 7's training loop, now parameterized so the *same* underlying mechanism can be invoked repeatedly with different datasets — exactly mirroring how a real fine-tuning pipeline reuses the same training code across pretraining and every subsequent fine-tuning stage, only the dataset changes.
- **Stage 2's starting loss (0.6388) vs. Stage 1's starting loss (3.1142)** — concrete, measured proof that fine-tuning starts from a meaningfully better position than training from scratch, because it inherits Stage 1's already-learned weights rather than starting from random noise.
- **Stage 3's starting loss (7.6073) — notably higher than Stage 2's starting loss** — this is realistic and worth explaining: the instruction-tuning data introduces a *new pattern* (`question → what → is → answer → it`) that the model's current weights have no strong prior association with, so its initial predictions for this new pattern are confidently wrong in places, producing a higher loss before it adapts.
- **The varying "the" predictions across repeated runs** — this is not a bug to be fixed, but the actual, honest behavior of the code, included deliberately to teach a real, important concept (catastrophic forgetting) rather than presenting an artificially tidy result.

---

## 6. TEACH-IT-BACK — Script You Can Say Out Loud

> "A large language model goes through several distinct training stages, but every single stage uses the exact same underlying mechanism: the forward pass, loss, backpropagation, gradient descent loop from earlier in this course — what changes between stages is the data and sometimes the precise loss signal, not the fundamental training algorithm. The first and by far most expensive stage is pretraining: training on a massive, broad corpus purely to predict the next token, which turns out to be such a rich learning signal that the model is forced to implicitly absorb grammar, facts, and reasoning patterns as a side effect. The result is called a base model — genuinely knowledgeable, but with no built-in concept of 'being a helpful assistant'; if you ask it a direct question, it might just continue the text in a quiz-like style rather than answering, because that's an equally plausible continuation given its training. Fine-tuning fixes specific gaps by continuing to train those same, already-pretrained weights on a smaller, more targeted dataset, which is dramatically cheaper than training from scratch since the model doesn't need to relearn basic language understanding. Instruction-tuning is a particular flavor of fine-tuning where the targeted dataset specifically teaches the format of responding to instructions and questions directly, rather than just continuing arbitrary text. And RLHF goes a step further, using human comparative preference judgments between candidate responses — rather than fixed ideal-response examples — to keep nudging the model toward outputs people actually find more helpful and appropriate. One important, honest caveat: because later stages continue training the same weights without any built-in protection for earlier-learned associations, there's a real, well-documented risk called catastrophic forgetting, where a later fine-tuning stage can degrade or overwrite something an earlier stage carefully taught — which is exactly why real fine-tuning pipelines use techniques like mixing in original training data or careful learning-rate tuning to manage that risk, rather than treating each stage as a free, side-effect-free upgrade."

---

## Key Terms Glossary (Day 11)

| Term | Meaning |
|---|---|
| **LLM** | A Transformer-based model trained at massive scale to predict the next token |
| **Pretraining** | The first, broadest, most expensive training stage on general text |
| **Base model** | A purely pretrained model, before any assistant-specific shaping |
| **Fine-tuning** | Continuing training on a smaller, targeted dataset from already-pretrained weights |
| **Instruction-tuning** | Fine-tuning specifically aimed at teaching instruction/question-response format |
| **RLHF** | Using human preference comparisons to further steer model behavior |
| **Catastrophic forgetting** | A later training stage degrading or overwriting earlier-learned associations |

---

## Quick Self-Check (ask yourself before moving to Day 12)

1. Can I explain, without looking, why next-token prediction on broad data turns out to teach so much more than "just predicting words"?
2. Can I explain why a purely pretrained base model might not directly answer a question?
3. Can I explain why fine-tuning is cheaper than pretraining, in terms of the weights' starting point?
4. Can I explain catastrophic forgetting, and why it's a real, unresolved engineering concern rather than a hypothetical edge case?

If yes to all four — you're ready for **Day 12: Embeddings Deep Dive** (semantic meaning as vectors, cosine similarity, with real embedding APIs called from Node.js).
