# LLM Model Routing — Brainstorm

> Created: 2026-04-08

## Goal

Automatically switch between LLM models based on task type to improve **cost efficiency** and **token efficiency**.

---

## Key Insight: Cost vs Token Efficiency

Model routing primarily improves **cost per task**, not raw token count.

| What routing changes | What stays the same |
|---|---|
| Cost per token (cheaper model = cheaper rate) | Input/output token COUNT |
| Speed (faster model = faster response) | Prompt structure |
| Quality fit (right model for the job) | Risk: wrong model → retries → MORE tokens |

**True token count reduction** comes from: prompt caching, context trimming, output constraints, structured output.

---

## Task → Model Routing Map

| Task Type | Signal Keywords / Context | Model | Reason |
|---|---|---|---|
| Code writing / debugging | `write`, `fix bug`, `refactor`, `SQL`, `function` | Claude Sonnet 4.6 | Best instruction-following + structured output |
| Hard architecture / design | `design system`, `tradeoffs`, `architect`, `review PRD` | Claude Opus 4.6 | Deep reasoning, nuanced writing |
| Math / logic | `prove`, `calculate`, `formula`, `complexity` | o3-mini | Chain-of-thought reasoning dominates |
| UI / Figma / visual | `mockup`, `wireframe`, `Figma`, `screenshot`, `image` | GPT-4o | Multimodal — reads/analyzes images |
| Huge doc processing | context > 200K tokens, `summarize PDF`, full codebase | Gemini 2.5 Pro | 1M+ token window |
| Simple extraction / tagging | `classify`, `extract`, `format`, `convert JSON` | Haiku 4.5 / Gemini Flash | Cheap, fast, no complex reasoning needed |
| Privacy / sensitive data | internal PII, on-prem flag | Llama 4 (self-hosted) | Zero data egress |
| Multilingual | non-English input, `translate`, `respond in Chinese` | Mistral Large or Claude | Strong multilingual capability |
| Real-time web / news | `what happened today`, `latest`, `current price` | Grok or GPT-4o w/ search | Live data access |
| RAG / retrieval grounding | `search our docs`, `find in knowledge base` | Cohere Command | Built for retrieval augmentation |

---

## Router Architecture Patterns

### Pattern 1: Classifier-Then-Dispatch
```
Prompt
  → Haiku (fast classifier, ~100 tokens)
      → returns: { task_type, complexity, has_image, context_size }
  → Routing table → Model A / B / C
```
Cost: ~$0.0001 per routing decision. Almost free.

### Pattern 2: Rule-Based Fast Path + LLM Fallback
```
Prompt
  → Regex/keyword rules (zero cost, 0ms)
      → HIGH CONFIDENCE → dispatch directly
      → LOW CONFIDENCE → call Haiku classifier
  → Model selected
```
Best for token efficiency — avoids even the classifier call for obvious cases.

### Pattern 3: Confidence-Gated Escalation
```
Prompt → Sonnet (default)
  → Sonnet says "uncertain" or hits complexity threshold
  → Escalate to Opus / o3
```
Saves Opus tokens for genuinely hard tasks. Sonnet handles ~80%, Opus handles the edge 20%.

---

## Token Efficiency Levers (priority order)

| Lever | Effort | Impact |
|---|---|---|
| **Prompt caching** for repeated context | Low | Very high — 90% discount on cache hits |
| **Output length instructions** | Very low | 30–50% reduction on output tokens |
| **Context window trimming** | Medium | 40–60% reduction on input tokens |
| **Structured output (JSON)** | Low | 50%+ reduction on output tokens vs prose |
| **Model routing** | Medium | Cost reduction, not token count |
| **Task decomposition** | High | Situational |

### Prompt Caching Example
```python
# Mark static context as cacheable (Anthropic SDK)
{"type": "text", "text": your_big_context, "cache_control": {"type": "ephemeral"}}
```

### Output Length Control
```
Bad:  "Explain how Kafka works"
Good: "Explain Kafka in 3 bullet points, max 100 words"
```

### Structured Output over Prose
```
Bad:  "Tell me the model name, cost, and best use case for each Claude model"
Good: "Return JSON: {name, cost_per_1M_tokens, best_for}"
```

---

## Full Token-Efficient Router Stack

```
User Prompt
  → Classifier        (task_type, complexity, output_format_needed)
  → Context Trimmer   (strip irrelevant history, summarize old turns)
  → Prompt Compressor (enforce length constraints, remove filler)
  → Model Selector    (route to right model)
  → Output Parser     (extract structured data, discard prose wrapper)
```

Realistic savings: **50–70% token reduction** vs naively calling Opus with full context every time.

---

## Config Sketch

```yaml
# task_router_config.yaml
routes:
  coding:
    signals: ["write code", "debug", "SQL", "function", "test"]
    model: claude-sonnet-4-6
    escalate_to: claude-opus-4-6
    escalate_if: complexity_score > 0.8

  visual_design:
    signals: ["figma", "wireframe", "mockup", "screenshot"]
    model: gpt-4o
    requires: [image_input]

  math_logic:
    signals: ["prove", "formula", "optimize", "complexity"]
    model: o3-mini

  bulk_extraction:
    signals: ["extract", "classify", "format", "tag"]
    model: claude-haiku-4-5
    max_context_tokens: 8000

  long_doc:
    signals: []
    trigger_if: context_tokens > 200000
    model: gemini-2.5-pro
```

---

## Next Steps

- [ ] Implement Python routing layer with Anthropic SDK
- [ ] Add context trimmer (sliding window or summarization)
- [ ] Wire up prompt caching for repeated system prompts
- [ ] Add observability: log `model_used`, `task_type`, `tokens_used` per call
- [ ] Use logged data to tune routing rules over time
