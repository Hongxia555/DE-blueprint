# Claude Token Efficiency: 10 Habits
**Source:** X/@kaize — shared habits for staying within Claude usage limits

---

## Core Insight

Claude limits **token count**, not message count. Because Claude re-reads the entire conversation history on every reply, costs grow quadratically:

```
Total tokens = S × N(N+1) / 2
S = average tokens per turn, N = number of messages
```

Example at ~500 tokens/turn:
| Messages | Total Tokens |
|---|---|
| 5 | 7,500 |
| 10 | 27,500 |
| 20 | 105,000 |
| 30 | 232,500 |

The 30th message costs **31x more** than the 1st. One study found 98.5% of tokens were spent re-reading history, only 1.5% on actual output.

---

## 10 Habits

### 1. Edit prompts instead of sending corrections
When Claude misunderstands, don't send a follow-up like "no, I meant...". Every new message stacks on top of history.

**Do this instead:** Click the edit button on the original message → revise → regenerate. The old exchange is replaced, not appended.

### 2. Start a new conversation every 15–20 messages
Long conversations burn tokens fast. When a chat gets long:
1. Ask Claude to summarize the conversation
2. Copy the summary
3. Open a new chat
4. Paste the summary as the first message

You keep the context, but reset the accumulation cost.

### 3. Batch questions into one message
Three separate prompts = three full context loads.
One prompt with three tasks = one context load.

Instead of:
> "Summarize this article" → "List the key points" → "Suggest a title"

Do this:
> "Summarize this article, list the key points, and suggest a title."

The answer is often better too — Claude sees the full requirement upfront.

### 4. Upload files to Projects, not individual chats
Uploading the same PDF to multiple chats re-tokenizes it each time. Use the Projects feature instead — files are cached and shared across all conversations in the project without re-consuming quota.

### 5. Save preferences in Memory settings
Repeating role/style setup at the start of every chat wastes tokens. Go to **Settings → Memory and user preferences** and save your role, writing style, and preferences once. Claude applies them automatically.

### 6. Disable unused features
Web search, connectors, and extended thinking consume tokens even when you don't need them.
- Turn off **Search & Tools** for writing-only tasks
- Keep **extended thinking** off by default — enable only when the first result isn't good enough

Rule: if you didn't actively turn it on, turn it off.

### 7. Match model to task complexity
| Model | Use for | Cost |
|---|---|---|
| Haiku | Grammar checks, quick translations, formatting, brainstorming | Lowest |
| Sonnet | Core daily work | Medium |
| Opus | Deep reasoning, complex tasks | Highest |

Simple tasks don't need a powerful model. Using Haiku for drafts saves 50–70% of budget for tasks that actually need it.

### 8. Spread work across the day
Claude uses a **rolling 5-hour window** — usage doesn't reset at midnight, it expires 5 hours after it was consumed. Messages sent at 9am expire by 2pm.

Split your day into 2–3 sessions (morning, afternoon, evening) so earlier usage expires before the next session begins.

### 9. Avoid peak hours
From March 26, 2026, Anthropic burns quota faster during peak load:
- **Weekdays 5:00–11:00am PT** (Beijing time: 8:00pm–2:00am)

Same conversation, same cost — but higher impact on your quota during peak. Schedule resource-intensive tasks in off-peak windows.

### 10. Enable overage as a safety net
Pro, Max 5x, and Max 20x subscribers can enable **Settings → Usage → Overage**. When session quota runs out, Claude falls back to pay-as-you-go API rates instead of cutting you off — useful for critical moments.

---

## Summary

Once these habits are consistent, you rarely hit the usage limit. Some users have downgraded from Max to Pro plans because token usage dropped enough to fit.

**Remember: Claude counts tokens, not messages.**
