# Claude Cookbooks — What It Is and How It Works

## What Is It?

A collection of Python code examples and Jupyter notebooks for **building applications with the Claude API**. Think of it as a recipe book — you find the pattern you need, read it, run it, and adapt it to your own project.

**Target user:** Developers writing Python code who want to integrate Claude into products, pipelines, or automation tools.

---

## The Core Mechanic

Everything reduces to a single API call:

```python
import anthropic

client = anthropic.Anthropic(api_key="your-key")

response = client.messages.create(
    model="claude-opus-4-6",
    messages=[{"role": "user", "content": "Your prompt here"}]
)
print(response.content[0].text)
```

All the patterns in the cookbook are just different ways to **structure what you send and how you handle what comes back**.

---

## Key Patterns

### 1. Basic RAG (Retrieval-Augmented Generation)
**Problem:** Claude doesn't know your private data or recent info.
**Solution:** Fetch relevant documents first, inject them into the prompt, then ask Claude.

```
[Your data source] → fetch relevant docs → stuff into prompt → Claude answers
```

Use cases: internal knowledge bases, document Q&A, up-to-date news summarization.

---

### 2. Tool Use (Function Calling)
**Problem:** Claude can't run code, query databases, or call APIs on its own.
**Solution:** You define tools Claude can "request." Claude returns a structured call, you execute it, feed the result back.

```
You define tools → Claude picks one → You run it → Return result → Claude uses it to answer
```

Example flow:
1. You define a `run_sql(query)` tool
2. Claude says: "I need to run `SELECT * FROM orders WHERE date > '2024-01-01'`"
3. Your code executes the query
4. You send the result back to Claude
5. Claude gives a final answer using that data

Use cases: SQL agents, calculator, web search, file operations.

---

### 3. Agents (Agentic Loop)
**Problem:** Some tasks require multiple steps and decisions, not a single prompt.
**Solution:** Run Claude in a loop — it thinks, picks a tool, gets the result, thinks again, until it decides it's done.

```
Task → Claude thinks → picks tool → run tool → Claude thinks → picks tool → ... → final answer
```

The "agent" is just your Python code managing this loop. Claude is the brain; your code is the body.

Use cases: research agents, automated workflows, coding assistants.

---

### 4. Multi-Agent Systems
**Problem:** One agent trying to do everything gets messy and slow.
**Solution:** Multiple Claude instances with different roles, coordinated by an orchestrator.

```
Orchestrator Claude
    ├── Worker Claude A (specialist: web search)
    ├── Worker Claude B (specialist: data analysis)
    └── Worker Claude C (specialist: writing)
```

The orchestrator breaks down the task and delegates. Workers focus on their piece. Results are assembled back up.

Use cases: complex research pipelines, code review + fix + test cycles.

---

### 5. Prompt Caching
**Problem:** Sending the same large context (docs, instructions) on every call is expensive and slow.
**Solution:** Cache the static part of your prompt so Claude reuses it across calls.

Use cases: any app with a large system prompt or reference document.

---

## Folder Map

| Folder | What's in it |
|---|---|
| `capabilities/` | Classification, summarization, RAG basics |
| `tool_use/` | Function calling examples |
| `multimodal/` | Images, charts, PDFs |
| `claude_agent_sdk/` | Full agent tutorials using the Agent SDK |
| `patterns/` | Reusable agent patterns and skills |
| `third_party/` | Integrations: Pinecone, Wikipedia, Voyage AI |
| `extended_thinking/` | Using Claude's extended reasoning mode |
| `observability/` | Logging and monitoring agent runs |
| `finetuning/` | Fine-tuning workflows |

---

## How to Actually Use It

1. Find the folder that matches your use case
2. Open the `.ipynb` notebook in Jupyter
3. Set your `ANTHROPIC_API_KEY` in a `.env` file
4. Run through the notebook
5. Copy the relevant pattern into your own project

You don't "install" or "run" the cookbook itself — it's reference material you read and adapt.
