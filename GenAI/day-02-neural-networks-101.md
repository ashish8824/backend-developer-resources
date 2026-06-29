# Day 2 — Neural Networks 101

> Part of the 30-Day Generative AI Mastery Roadmap (for a Node.js Backend Developer)

---

## 1. WHAT — Plain Definitions

A **neural network** is a stack of simple math units, called **neurons**, arranged in **layers**, that work together to turn an input (numbers) into an output (numbers) — by repeatedly doing "multiply, add, squash."

| Term | Plain Definition |
|---|---|
| **Neuron** | A tiny function: takes some numbers in, multiplies each by an "importance" value, adds them up, and passes the result through a squashing function. |
| **Weight** | A number representing *how much* one specific input matters to a neuron. Bigger weight = more influence. |
| **Bias** | An extra number added to a neuron's sum — lets it shift its decision threshold up or down, independent of the inputs. |
| **Activation function** | A function that "squashes" or reshapes a neuron's raw sum, introducing non-linearity (explained in WHY below). |
| **Layer** | A group of neurons that all look at the same inputs, but each learns to notice something different. |
| **Neural Network** | Multiple layers chained together: input layer → one or more hidden layers → output layer. |

### Diagram: A Single Neuron

```
   input₁ (x₁) ───weight w₁──┐
                              │
   input₂ (x₂) ───weight w₂──┼──► [ Σ (sum) + bias ] ──► [ activation fn ] ──► output
                              │
   input₃ (x₃) ───weight w₃──┘

   Math:  z = (x₁·w₁ + x₂·w₂ + x₃·w₃) + bias
   Output = activation(z)
```

That's it. A neuron is just: **multiply, add, squash.** Everything else in deep learning — GPT-4, image generators, all of it — is this same operation, repeated billions of times across billions of neurons.

### Diagram: A Full Network (Layers)

```
   INPUT LAYER          HIDDEN LAYER (3 neurons)        OUTPUT LAYER
   (raw data,                                            (final answer)
    not "neurons")

   credit_score ──┬──► [Neuron H1] ──┐
        │         ├──► [Neuron H2] ──┼──► [Neuron O1] ──► APPROVE / REJECT
        │         └──► [Neuron H3] ──┘
   income ─────────┘

   Each arrow = a separate weight.
   Each neuron in the hidden layer looks at BOTH inputs,
   but each one can learn to weight them completely differently —
   e.g. H1 might mostly care about credit_score, H2 might mostly care about income.
```

**Why "hidden"?** Because you, the human, never directly see or label what H1, H2, H3 represent. The network decides on its own (during training, Day 3) what pattern each one should detect.

---

## 2. WHY — Why Do We Need This Structure?

### Why weights and bias?

Imagine you're deciding whether to approve a loan based on credit score and income. Not all factors are equally important — credit score probably matters more. **Weights are how the network encodes "how much each input matters."** A high weight on credit score means: "small changes in credit score should swing my decision a lot." Bias is a separate knob that shifts the *overall* tendency — e.g., a bank that's generally cautious would have a very negative bias, making it harder for ANY applicant to get approved unless the weighted inputs are strongly positive.

### Why an activation function? (This is the part people skip and later regret)

Without an activation function, a neuron is just `weighted sum + bias` — pure linear math. And here's the catch: **if you stack purely linear layers on top of each other, the whole network collapses into being equivalent to just ONE linear layer.** No matter how many layers you add, it can only ever learn straight-line relationships.

But the real world isn't straight lines. "Approve the loan if (credit score is high) OR (income is very high AND credit score is at least medium)" is not a straight-line rule — it's a curve, a combination, a *non-linear* relationship. The activation function injects a "bend" after every neuron, and stacking many bent layers lets the network approximate almost any complex pattern — this is called the **Universal Approximation Theorem**, and it's the mathematical reason deep learning works at all.

### Common activation functions (intuition only, no calculus)

| Function | What it does | Where it's used |
|---|---|---|
| **Sigmoid** | Squashes any number into a range between 0 and 1 — perfect for "probability" outputs. | Output layers for yes/no decisions |
| **ReLU** (Rectified Linear Unit) | If the number is negative, output 0. If positive, pass it through unchanged. Dead simple, but very effective. | Hidden layers in almost all modern networks |
| **Softmax** | Turns a list of numbers into a list of probabilities that all add up to 1. | Output layers for "pick one of N categories" |

> **Quick gut-check:** Sigmoid says "how confident am I, 0 to 100%?" ReLU says "ignore anything negative, pass through anything positive." Softmax says "distribute 100% of confidence across all my options."

---

## 3. HOW — The Mechanism (Intuition + Light Math)

### Step-by-step: what happens inside one neuron

Let's say a neuron receives 2 inputs: `x1 = 0.9` (normalized credit score) and `x2 = 0.8` (normalized income). It has learned weights `w1 = 0.7`, `w2 = 0.3`, and a bias `b = -0.5`.

**Step 1 — Weighted sum:**
```
z = (x1 × w1) + (x2 × w2) + b
z = (0.9 × 0.7) + (0.8 × 0.3) + (-0.5)
z = 0.63 + 0.24 - 0.5
z = 0.37
```

**Step 2 — Activation (sigmoid):**
```
sigmoid(z) = 1 / (1 + e^(-z))
sigmoid(0.37) = 1 / (1 + e^(-0.37)) ≈ 0.591
```

The neuron's final output is **≈0.591** — interpreted as "59.1% confidence this applicant should be approved."

### What happens across a full network (forward pass)

This entire process — data flowing from the input layer, through the hidden layer(s), to the output layer — is called the **forward pass**. Each layer's *output* becomes the *next* layer's *input*. That's the whole trick that lets networks build up complexity: layer 1 might detect simple patterns, layer 2 combines those into medium patterns, layer 3 combines those into complex patterns — like how your visual cortex detects edges first, then shapes, then whole objects.

> **Important honesty check:** in the diagram and math above, *I* (a human) chose the weight values to make a sensible example. In a real neural network, you **never** hand-pick the weights. They start as random numbers, and the network discovers good values for them through **training** — which is exactly what we cover tomorrow, Day 3 (backpropagation and gradient descent). Today is only about understanding the *structure*; tomorrow is about how it *learns*.

---

## 4. EXAMPLE — Real-World Analogy + Worked Example

### Analogy: A hiring committee

Picture a hiring committee deciding whether to extend a job offer:

- Each **committee member** is like a neuron.
- Each member cares about different signals — one cares mostly about technical test scores, another mostly about culture-fit interview notes, another blends both equally. That's the **weights**.
- Some members are just naturally harder to please ("I rarely vote yes unless I'm very impressed") — that's their **bias**.
- Each member doesn't just blurt out a raw number — they convert their gut feeling into "yes/no" or "1-10 confidence" — that's the **activation function**.
- The **hidden layer** is the committee members deliberating individually.
- The **output layer** is the final hiring manager, who takes all committee members' individual verdicts and combines THEM into one final decision.

This is structurally *identical* to the diagram above — multiple "opinions" (neurons) computed in parallel, combined into a final decision.

### Worked numeric example (3-applicant comparison)

| Applicant | Credit Score (x1) | Income (x2) | Network's Output | Decision |
|---|---|---|---|---|
| A | 0.9 (great) | 0.8 (good) | 0.711 | APPROVE |
| B | 0.2 (poor) | 0.3 (low) | 0.506 | APPROVE *(borderline — see note below)* |
| C | 0.5 (avg) | 0.9 (high) | 0.659 | APPROVE |

**Important teaching note:** Applicant B has poor credit and low income, yet the network still leans toward "approve" (barely, at 0.506). **This is not a mistake in the code — it's the correct lesson.** I deliberately chose arbitrary weights for this example to demonstrate the *structure* of a network. Since nobody has *trained* this network on real data yet, its weights don't actually encode good loan-approval judgment — they're just illustrative numbers. This is exactly why Day 3 (training) exists: **a network is only as good as the data and process used to adjust its weights.** An untrained or badly-trained network can produce confidently wrong answers — which, by the way, is also a preview of why GenAI models can "hallucinate" later in this course.

---

## 5. IMPLEMENTATION — Node.js (Built From Scratch, No Libraries)

We're building this with raw JavaScript — no TensorFlow.js, no libraries — specifically so every multiplication is visible to you. This is the single best way to kill the "black box" feeling.

### Part A — A Single Neuron

```javascript
// single-neuron.js
// A single neuron, built with ZERO ML libraries — just plain math.
// Task: predict "should I approve this loan?" from 2 inputs.

function sigmoid(x) {
  return 1 / (1 + Math.exp(-x));
}

function neuron(inputs, weights, bias) {
  // Step 1: weighted sum (each input multiplied by its importance/weight)
  let z = 0;
  for (let i = 0; i < inputs.length; i++) {
    z += inputs[i] * weights[i];
  }
  z += bias;

  // Step 2: activation function (squashes result into a 0-1 "probability")
  const output = sigmoid(z);
  return output;
}

// Inputs: [creditScoreNormalized, incomeNormalized] -- both scaled 0 to 1
const weights = [0.7, 0.3]; // credit score matters more than income here
const bias = -0.5;

const applicants = [
  { name: "Applicant A", inputs: [0.9, 0.8] }, // great credit, good income
  { name: "Applicant B", inputs: [0.2, 0.3] }, // poor credit, low income
  { name: "Applicant C", inputs: [0.5, 0.9] }, // average credit, high income
];

applicants.forEach(({ name, inputs }) => {
  const output = neuron(inputs, weights, bias);
  const decision = output > 0.5 ? "APPROVE" : "REJECT";
  console.log(`${name}: raw output = ${output.toFixed(3)} -> ${decision}`);
});
```

**Run it:**
```bash
node single-neuron.js
```

**Verified output:**
```
Applicant A: raw output = 0.591 -> APPROVE
Applicant B: raw output = 0.433 -> REJECT
Applicant C: raw output = 0.530 -> APPROVE
```

#### Line-by-line explanation

- **`sigmoid(x)`** — implements the math formula `1 / (1 + e^(-x))` exactly. `Math.exp(-x)` computes `e^(-x)`. As `x` gets very large, this approaches 1. As `x` gets very negative, this approaches 0. That's the "squashing."
- **`neuron(inputs, weights, bias)`** — the loop multiplies each input by its corresponding weight and accumulates the sum (`z`). This is literally implementing the dot product of two vectors — a concept that will reappear constantly (it's the same math as cosine similarity for embeddings, Day 12).
- **`z += bias`** — adds the bias after the weighted sum, exactly matching the math from Section 3.
- **The decision threshold (`> 0.5`)** — this is a choice *we* make on top of the neuron's raw output. The neuron itself just outputs a probability-like number; turning that into a binary decision is a separate, simple step.

---

### Part B — A Full Network with a Hidden Layer

```javascript
// tiny-network.js
// A tiny neural network with ONE hidden layer, built from scratch.
// No libraries. Just arrays, loops, and math — so every step is visible.

function sigmoid(x) {
  return 1 / (1 + Math.exp(-x));
}

function relu(x) {
  return Math.max(0, x);
}

// A single neuron: weighted sum of inputs + bias, then activation
function neuron(inputs, weights, bias, activationFn) {
  let z = 0;
  for (let i = 0; i < inputs.length; i++) {
    z += inputs[i] * weights[i];
  }
  z += bias;
  return activationFn(z);
}

// A layer = a collection of neurons, each with its own weights/bias,
// all looking at the SAME inputs but learning to detect different patterns.
function layer(inputs, neuronsConfig, activationFn) {
  return neuronsConfig.map(({ weights, bias }) =>
    neuron(inputs, weights, bias, activationFn)
  );
}

// --- Network architecture ---
// Input layer: 2 values (credit score, income) -- not "neurons", just raw data
// Hidden layer: 3 neurons, each using ReLU activation
// Output layer: 1 neuron, using Sigmoid (squashes to 0-1 probability)

const hiddenLayerConfig = [
  { weights: [0.8, 0.1], bias: 0.0 },   // hidden neuron 1: cares mostly about credit score
  { weights: [0.1, 0.9], bias: -0.2 },  // hidden neuron 2: cares mostly about income
  { weights: [0.5, 0.5], bias: 0.1 },   // hidden neuron 3: blends both equally
];

const outputLayerConfig = [
  { weights: [0.6, 0.4, 0.5], bias: -0.3 }, // combines all 3 hidden neuron outputs
];

function forwardPass(inputs) {
  // Step 1: run the hidden layer on the raw inputs
  const hiddenOutputs = layer(inputs, hiddenLayerConfig, relu);

  // Step 2: feed the hidden layer's OUTPUTS as inputs to the output layer
  const finalOutput = layer(hiddenOutputs, outputLayerConfig, sigmoid);

  return { hiddenOutputs, finalOutput: finalOutput[0] };
}

// --- Test on applicants ---
const applicants = [
  { name: "Applicant A", inputs: [0.9, 0.8] },
  { name: "Applicant B", inputs: [0.2, 0.3] },
  { name: "Applicant C", inputs: [0.5, 0.9] },
];

applicants.forEach(({ name, inputs }) => {
  const { hiddenOutputs, finalOutput } = forwardPass(inputs);
  const decision = finalOutput > 0.5 ? "APPROVE" : "REJECT";
  console.log(
    `${name}: hidden=[${hiddenOutputs.map(h => h.toFixed(2)).join(", ")}] ` +
    `final=${finalOutput.toFixed(3)} -> ${decision}`
  );
});
```

**Run it:**
```bash
node tiny-network.js
```

**Verified output:**
```
Applicant A: hidden=[0.80, 0.61, 0.95] final=0.711 -> APPROVE
Applicant B: hidden=[0.19, 0.09, 0.35] final=0.506 -> APPROVE
Applicant C: hidden=[0.49, 0.66, 0.80] final=0.659 -> APPROVE
```

#### Line-by-line explanation

- **`relu(x)`** — `Math.max(0, x)`. If `x` is negative, return 0. Otherwise, pass it through. This is the entire ReLU function — deceptively simple, but it's the most-used activation function in modern deep learning because it's cheap to compute and works well in practice.
- **`layer(inputs, neuronsConfig, activationFn)`** — this is the key structural idea of Day 2: a layer is just "run the same `neuron()` function multiple times, once per neuron config, all using the *same* `inputs`." Each neuron has its *own* independent weights/bias, so even though they see identical inputs, they can each learn to detect something different (that's what `hiddenLayerConfig` encodes — neuron 1 weights credit score heavily, neuron 2 weights income heavily, neuron 3 blends both).
- **`forwardPass(inputs)`** — this is the heart of "what does a neural network do." Notice `hiddenOutputs` (the result of the first layer) becomes the *input* to the second layer (`layer(hiddenOutputs, outputLayerConfig, sigmoid)`). Data literally flows forward, layer by layer — hence "forward pass."
- **Why the example's Applicant B result looks "wrong":** as discussed in Section 4, these weights were chosen by me for illustration, not learned from real data. This is intentional — it sets up tomorrow's lesson perfectly: a network's *structure* (today) is separate from its *intelligence* (tomorrow, via training).

---

## 6. TEACH-IT-BACK — Script You Can Say Out Loud

> "A neural network is built out of tiny units called neurons, and each neuron does exactly three things: multiply, add, squash. It multiplies each input by a 'weight' — a number representing how important that input is — adds those up along with one extra number called a bias, and then passes the result through an activation function that squashes it into a useful range, like a probability between 0 and 1. You stack these neurons into layers — a layer is just a bunch of neurons that all see the same inputs, but each one is allowed to weight those inputs completely differently, so collectively they can notice different patterns. Data flows forward through the network: the input layer's numbers go into the hidden layer, the hidden layer's outputs become the inputs to the next layer, and so on, until you reach the output layer with your final answer — this is called the forward pass. The activation function matters more than people realize: without it, stacking layers would be pointless math, because many linear layers stacked together collapse into being just one linear layer. The activation function is what lets the network learn bent, complex, real-world patterns instead of only straight lines. And critically — the weights and biases in a network aren't designed by a human. They start random, and they get adjusted automatically through a training process, which is tomorrow's topic."

---

## Key Terms Glossary (Day 2)

| Term | Meaning |
|---|---|
| **Neuron** | The basic computational unit: weighted sum + bias, then activation |
| **Weight** | A learned number representing the importance of one input |
| **Bias** | A learned number that shifts a neuron's decision threshold |
| **Activation function** | Introduces non-linearity (bending) so networks can learn complex patterns |
| **Sigmoid** | Activation function squashing output to range (0, 1) |
| **ReLU** | Activation function: `max(0, x)` — used in most modern hidden layers |
| **Softmax** | Activation function turning a list of numbers into probabilities summing to 1 |
| **Layer** | A group of neurons processing the same inputs in parallel |
| **Hidden layer** | A layer between input and output whose learned representations aren't directly labeled/interpreted by humans |
| **Forward pass** | The process of data flowing from input layer through hidden layers to the output layer |
| **Universal Approximation Theorem** | The mathematical guarantee that a network with enough neurons/layers and non-linear activations can approximate almost any function |

---

## Quick Self-Check (ask yourself before moving to Day 3)

1. Can I explain, without looking, what the three steps inside a single neuron are?
2. Can I explain *why* we need activation functions, not just *that* we need them?
3. Do I understand why this network's weights are illustrative, not "trained" — and what that implies about Applicant B's questionable approval?

If yes to all three — you're ready for **Day 3: How neural nets learn** (forward pass, loss function, backpropagation, gradient descent).
