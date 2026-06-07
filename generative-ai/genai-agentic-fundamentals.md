# GenAI & Agentic AI — Interview Q&A

## Q1. Base model vs Instruction-tuned model

**Base model** — trained on raw internet text, predicts the next token. No instruction following, just text completion.

**Instruction-tuned model** — base model further trained using RLHF or SFT to follow instructions like "Explain X", "Write me Y". GPT-4, Claude, Gemini are all instruction-tuned.

**When to use base:** almost never in production. Fine-tuning your own task. Research.
**When to use instruction-tuned:** always. It's what you're already using.

---

## Q2. Temperature

Controls **randomness** in the output.

- `temperature=0` → deterministic. Same input = same output every time. Use for: code gen, structured data extraction, factual Q&A.
- `temperature=1` → default, balanced creativity.
- `temperature=2` → chaotic, creative but often incoherent.

**Interview one-liner:** "Temperature controls how the model samples from its probability distribution. Zero means always pick the highest probability token."

---

## Q3. Hallucination + How to reduce it

**Hallucination** = LLM confidently states something false. It doesn't "know" facts, it predicts plausible-sounding tokens.

**Two ways to reduce:**
1. **RAG** — don't rely on the model's memory, give it the actual source document in the prompt.
2. **Structured outputs + validation** — force JSON output, then validate fields programmatically.

Bonus: grounding with citations, lower temperature for factual tasks.

---

## Q4. Few-shot vs Zero-shot

**Zero-shot:** just give the instruction, no examples.
> "Classify this review as positive or negative: 'Great product!'"

**Few-shot:** give 2-5 examples before your actual input.
> "positive: 'Love it!' | negative: 'Terrible.' | positive: 'Amazing!' → classify: 'Could be better'"

**When zero-shot:** model is capable enough, simple task.
**When few-shot:** complex task, specific output format, consistent tone/structure needed.

---

## Q5. RAG Pipeline end-to-end

RAG = Retrieval-Augmented Generation. Solves: LLM doesn't know your private/fresh data.

```
1. INGEST (one-time setup)
   Your docs → chunk into pieces → embed each chunk (vector) → store in vector DB

2. QUERY TIME (every user request)
   User asks question
   → embed the question
   → search vector DB for similar chunks
   → retrieve top-K chunks
   → stuff chunks into LLM prompt
   → LLM answers from those chunks
```

Real world: "What's our refund policy?" → finds the refund policy chunk → LLM answers from it, not from hallucination.

---

## Q6. Chunking + Why chunk size matters

**Chunking** = splitting documents into smaller pieces before embedding.

**Why it matters:**
- Too big: chunk contains multiple topics, embedding is "averaged" across all, retrieval is fuzzy
- Too small: loses context, retrieved chunk doesn't have enough info to answer
- Sweet spot: ~512 tokens with ~50 token overlap between chunks

**Strategies:** fixed size, sentence-based, recursive (split by headers → paragraphs → sentences), semantic.

---

## Q7. Semantic vs Keyword search + Hybrid search

**Keyword search (BM25/Elasticsearch):** matches exact words. "JWT authentication" won't find "token-based login".

**Semantic search:** embed query + docs as vectors, find by meaning not words. "JWT authentication" WILL find "token-based login" — semantically close.

**Hybrid search:** run both, combine scores. Best of both worlds — exact match + semantic understanding. Most production RAG systems use hybrid.

---

## Q8. Vector embeddings + Cosine similarity

**Embedding** = a model converts text into a list of numbers (e.g. 1536 floats for OpenAI ada-002). Semantically similar text → numerically similar vectors.

**Cosine similarity** = measures the angle between two vectors, not their magnitude. Range: -1 (opposite) to 1 (identical direction = same meaning).

Why cosine vs Euclidean distance? You care about direction (meaning), not magnitude (word count). A short and long doc about the same topic should still match.

---

## Q9. What makes something an "agent"

An LLM call just takes input → returns output. Done.

An **agent** is an LLM in a loop that:
1. Perceives state (input + context + memory)
2. Decides what action to take (call a tool, search, write code)
3. Executes the action
4. Observes the result
5. Loops back until the goal is achieved

**Key difference:** agents have tools, memory, and can take multi-step actions autonomously.

---

## Q10. ReAct loop (Reasoning + Acting)

A prompting pattern where the agent alternates between Thought and Action:

```
Thought: I need to find the current weather in Delhi
Action: search("Delhi weather today")
Observation: 38°C, sunny
Thought: I have the answer now
Action: respond("Delhi is 38°C and sunny")
```

The model reasons before acting, which reduces errors vs blindly calling tools.

---

## Q11. Function calling / Tool use

You define functions (tools) with a name, description, and JSON schema for parameters. You send these to the LLM alongside the user message.

The LLM decides: "I need to call `get_weather` with `{city: 'Delhi'}`" and returns that as structured JSON instead of text. Your code executes the function, sends result back to the LLM, and the LLM uses it to answer.

Under the hood: trained to output structured JSON when a tool call is appropriate. OpenAI and Anthropic both support this natively.

---

## Q12. Multi-agent systems

**Single agent:** one LLM doing everything. Gets overwhelmed on complex tasks, context window fills up, quality degrades.

**Multi-agent:** split responsibility.
- **Orchestrator** — the planner. Breaks the big task into subtasks, delegates, collects results.
- **Subagents** — specialists. Each focused on one thing (Coder, Researcher, Designer).

**When to use multi-agent:**
- Task is too long for one context window
- Parallel work needed (research + design at same time)
- Specialized expertise matters (coding model for code, reasoning model for planning)

---

## Cheatsheet

| Concept | One-liner |
|---|---|
| Base vs Instruction-tuned | Raw predictor vs trained to follow instructions |
| Temperature | Randomness knob — 0 = deterministic |
| Hallucination | LLM predicts plausible but false tokens |
| RAG | Inject real docs into prompt so LLM doesn't guess |
| Chunking | Split docs into pieces; size affects retrieval quality |
| Semantic search | Match by meaning (vectors), not words |
| Embedding | Text → numbers; similar meaning → similar numbers |
| Agent | LLM + tools + memory + loop |
| ReAct | Thought → Action → Observation → repeat |
| Function calling | LLM returns structured JSON to trigger your code |
| Multi-agent | Orchestrator + specialists for complex parallel work |
