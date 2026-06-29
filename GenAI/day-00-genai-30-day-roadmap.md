# 30-Day Generative AI Mastery Roadmap
### For a Node.js Backend Developer — Teach-It-Back Edition

> Format for every day: **What → Why → How (intuition + light math) → Example → Implementation (Node.js) → Teach-it-back script**

---

## Week 1 — Foundations: How AI "Thinks" Before Generative AI

**Goal:** Understand the machinery under the hood so nothing later feels like magic.

- [ ] **Day 1** — What is AI / ML / DL / GenAI (the nested-doll relationship). History: rule-based systems → Machine Learning → Deep Learning → Generative AI.
- [ ] **Day 2** — Neural Networks 101: neurons, weights, biases, activation functions (intuition, no calculus grind).
- [ ] **Day 3** — How neural nets learn: forward pass, loss function, backpropagation, gradient descent (intuition + light math).
- [ ] **Day 4** — Data representations: tensors, embeddings — turning words/images into numbers.
- [ ] **Day 5** — The sequential data problem: RNNs/LSTMs and why they struggled (sets up "why Transformers won").
- [ ] **Day 6** — The Transformer architecture (high level): encoder/decoder, why it replaced RNNs.
- [ ] **Day 7** — **Review + Mini Project**: Build a tiny "predict next word" toy model conceptually + visualize embeddings in Node.js.

---

## Week 2 — The Core of Modern GenAI: Attention, LLMs, and Tokens

**Goal:** Deeply understand what an LLM actually is.

- [ ] **Day 8** — Self-Attention mechanism explained (Query, Key, Value — the famous "QKV").
- [ ] **Day 9** — Multi-head attention + positional encoding (why order matters to a Transformer).
- [ ] **Day 10** — Tokenization: BPE, tokenizers, why "strawberry" confuses LLMs. Implementation with `tiktoken`/`js-tiktoken`.
- [ ] **Day 11** — What is a Large Language Model (LLM)? Pretraining vs fine-tuning vs instruction-tuning vs RLHF.
- [ ] **Day 12** — Embeddings deep dive: semantic meaning as vectors, cosine similarity. Implementation: generate embeddings via API in Node.js.
- [ ] **Day 13** — Decoding strategies: temperature, top-k, top-p, greedy decoding — why the same prompt gives different answers.
- [ ] **Day 14** — **Review + Mini Project**: Node.js CLI tool that calls an LLM API, shows token usage, and lets you experiment with temperature/top-p live.

---

## Week 3 — Building with GenAI: Prompting, RAG, and Agents

**Goal:** Go from "understanding" to "building real products."

- [ ] **Day 15** — Prompt Engineering fundamentals: zero-shot, few-shot, chain-of-thought, system vs user prompts.
- [ ] **Day 16** — Advanced prompting: ReAct, self-consistency, prompt templates. Implementation with a prompt-template approach in Node.js.
- [ ] **Day 17** — What is RAG (Retrieval-Augmented Generation) and why it solves hallucination/knowledge-cutoff issues.
- [ ] **Day 18** — Vector databases: how they work (ANN search, HNSW intuition). Implementation: Pinecone/Chroma/pgvector from Node.js.
- [ ] **Day 19** — Building a full RAG pipeline: chunking, embedding, retrieval, augmentation. Implementation: end-to-end Node.js RAG app.
- [ ] **Day 20** — Function calling / Tool use: how LLMs call APIs (very natural for a backend dev).
- [ ] **Day 21** — **Review + Mini Project**: RAG-powered Q&A API over your own docs (Express + vector DB + LLM).

---

## Week 4 — Agents, Fine-Tuning, Multi-Modal & Production

**Goal:** Advanced concepts + how to ship GenAI products safely and cheaply.

- [ ] **Day 22** — AI Agents: what makes something "agentic" (planning, memory, tool-use loops). LangChain.js/LangGraph basics.
- [ ] **Day 23** — Multi-agent systems & orchestration patterns (supervisor, pipeline, debate patterns).
- [ ] **Day 24** — Fine-tuning vs RAG vs prompting — when to use which (decision framework). LoRA/QLoRA intuition.
- [ ] **Day 25** — Multi-modal models: how image/audio/video generation works (diffusion intuition, CLIP-style image-text linking).
- [ ] **Day 26** — Image generation deep dive: diffusion models (denoising intuition) + implementation calling an image-gen API.
- [ ] **Day 27** — Evaluation of GenAI apps: hallucination detection, RAGAS, LLM-as-judge, guardrails.
- [ ] **Day 28** — Production concerns: caching, rate limits, streaming (SSE), cost optimization, prompt injection/security.
- [ ] **Day 29** — MCP (Model Context Protocol) & the modern agent-tool ecosystem.
- [ ] **Day 30** — **Capstone Project**: Full-stack GenAI agent (Node.js backend + RAG + tool calling + streaming) + a "teaching pack" you can present like a course.

---

## Daily Template (what you'll get every day)

1. **What** — plain-language definition
2. **Why** — the problem it solves / why it exists
3. **How** — the mechanism, with intuition + light math (no heavy derivations)
4. **Example** — real-world analogy + worked numeric/text example
5. **Implementation** — runnable Node.js code, explained line by line
6. **Teach-it-back** — a script you could say out loud to teach someone else

---
*Tip: Check off each day's box as you complete it. Re-export or ask Claude to update this file as you progress.*
