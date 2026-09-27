---
description: "Keep a long-running agent inside its context window: trimming by turn groups, preserving tool-call pairs, summarizing old steps, shrinking tool results."
---

# 03 · Short-Term Memory & Context

In Level 1 the message list grew by two or more messages per step and was sent in full
every time. That works for a five-step run. A thirty-step run eventually hits the
context limit, gets slower and more expensive with every step, and — well before the
limit — tends to lose track of the original goal as it drifts further back in a sea of
tool output. **Short-term memory management** is deciding what stays in the context.

## Three levers

1. **Shrink what goes in.** Tool results are usually the biggest items. Truncate, select
   fields, or summarize *in the tool* (lesson L1-04). This is the cheapest lever.
2. **Drop old turns.** Keep the system prompt and the task, drop the oldest steps.
3. **Summarize old turns.** Replace dropped steps with a short note of what was learned
   ("Checked logs for 02:00–03:00: timeout on db step. Metrics: lock wait high.").

Most production agents combine all three.

## The rule you must not break: keep tool pairs together

An assistant message with tool calls and the tool-result messages that answer it form
one **unit**. If you drop the call but keep a result (or the reverse), most provider APIs
reject the request, and even when they don't, the model sees an answer to a question it
never asked. Trim by **turn groups**, never by individual messages.

## Worked example: trimming and summarizing safely

```python title="memory.py"
"""Short-term memory: trim a message list by turn groups, optionally summarizing."""
import json

def turn_groups(messages):
    """Split messages after [system, user] into groups: an assistant message plus
    the tool results that answer it (or a lone user/assistant message)."""
    head, rest = messages[:2], messages[2:]
    groups, current = [], []
    for m in rest:
        if m["role"] == "tool":
            current.append(m)
        else:
            if current:
                groups.append(current)
            current = [m]
    if current:
        groups.append(current)
    return head, groups

def size(msgs):
    return sum(len(json.dumps(m)) for m in msgs)

def trim(messages, max_chars, summarize=None):
    """Drop the oldest turn groups until under max_chars. If `summarize` is given,
    replace the dropped groups with one summary message."""
    head, groups = turn_groups(messages)
    dropped = []
    while groups and size(head) + size(sum(groups, [])) > max_chars:
        dropped.append(groups.pop(0))
    kept = sum(groups, [])
    if dropped and summarize:
        note = summarize(sum(dropped, []))
        head = head + [{"role": "user", "content": f"[Summary of earlier steps] {note}"}]
    return head + kept, len(dropped)

def pairs_ok(messages):
    """Every tool result must answer a tool call that appears earlier."""
    asked = set()
    for m in messages:
        for c in m.get("tool_calls") or []:
            asked.add(c["id"])
        if m["role"] == "tool" and m["tool_call_id"] not in asked:
            return False
    return True
```

Build a realistic long history — twelve log-search steps, each with a bulky result —
then trim it with and without a summarizer. The summarizer here is a plain function;
in production it would be a cheap model call with an instruction like "list the facts
learned and the open questions, in under 80 words".

```python title="memory_demo.py"
import json
from memory import trim, pairs_ok, size

messages = [{"role": "system", "content": "You are an incident investigator."},
            {"role": "user", "content": "Why did the export job fail last night?"}]
for i in range(12):
    cid = f"c{i}"
    messages.append({"role": "assistant", "content": "", "tool_calls": [
        {"id": cid, "name": "search_logs",
         "arguments": json.dumps({"hour": f"{i:02d}:00"})}]})
    finding = "timeout on db step" if i == 2 else "no errors"
    messages.append({"role": "tool", "tool_call_id": cid,
                     "content": json.dumps({"hour": f"{i:02d}:00", "finding": finding,
                                            "raw": "log line " * 40})})

def facts_only(dropped):
    found = [json.loads(m["content"]) for m in dropped if m["role"] == "tool"]
    hits = [f"{f['hour']}: {f['finding']}" for f in found if f["finding"] != "no errors"]
    return f"searched {len(found)} hours; findings: {'; '.join(hits) or 'none'}"

print("original:", len(messages), "messages,", size(messages), "chars")

plain, n = trim(messages, max_chars=2500)
print("trimmed :", len(plain), "messages,", size(plain), "chars, dropped", n,
      "groups, pairs ok:", pairs_ok(plain))
print("  the 02:00 finding survived?", any("timeout" in m["content"] for m in plain))

summ, n = trim(messages, max_chars=2500, summarize=facts_only)
print("summary :", len(summ), "messages,", size(summ), "chars, pairs ok:", pairs_ok(summ))
print("  ", summ[2]["content"])

naive = messages[:2] + messages[-5:]          # cut by message count instead of groups
print("naive cut by message count, pairs ok:", pairs_ok(naive))
```

```text
original: 26 messages, 7397 chars
trimmed : 8 messages, 1952 chars, dropped 9 groups, pairs ok: True
  the 02:00 finding survived? False
summary : 9 messages, 2063 chars, pairs ok: True
   [Summary of earlier steps] searched 9 hours; findings: 02:00: timeout on db step
naive cut by message count, pairs ok: False
```

Plain trimming stays under budget but **loses the one important finding** from 02:00.
Summarizing keeps the fact in a few dozen characters. And the naive "keep the last five
messages" cut produces an orphaned tool result — exactly the malformed history that
providers reject.

## When to trim

- **Before every model call**, compare the estimated size to a budget well below the
  model's context limit (leave room for the response and for tool schemas, which also
  count).
- Prefer a **high-water / low-water** scheme: when you exceed 80% of budget, trim down
  to 50%. Trimming a little on every step changes the prompt prefix every time, which
  defeats prompt caching (Level 3 lesson 07); trimming in larger, rarer chunks keeps the
  prefix stable for longer.

## How It Actually Works

A model has no memory besides the tokens you send, and it attends over all of them on
every call. Two costs grow with history length: **compute and price** (roughly linear in
input tokens per call, so a long run's total cost grows faster than linearly), and
**attention dilution** — models are measurably worse at using information buried in the
middle of long contexts than information near the start or the end. The system prompt
and task sit at the start; the latest step sits at the end; old tool output accumulates
in the weakest position.

Summarizing works because it converts *raw observations* into *conclusions*. The model
needed forty lines of logs to notice the timeout; it needs only "timeout at 02:00 on the
db step" to reason about what to do next. The risk is lossy compression: if the
summarizer discards something that later turns out to matter, the agent can't get it
back. That's why good summaries keep **facts, decisions and open questions**, and why
long-term stores (next lesson) exist for anything that might be needed verbatim.

## Common mistakes

- **Cutting by message count** and orphaning tool results.
- **Dropping the original task** along with old turns. The first user message should
  almost never be trimmed.
- **Summaries that editorialize** ("the investigation is going well") instead of listing
  facts and open questions.
- **Trimming on every step**, destroying cache hits.
- **Ignoring tool-schema size** in the budget. Twenty verbose tools can be thousands of
  tokens.

## Exercise

1. Change `facts_only` to also list which hours were searched as a compact range
   ("00:00–08:00") and measure the character saving.
2. Implement high-water/low-water trimming: trim only when size exceeds 80% of a budget,
   and then down to 50%. Count how many times trimming happens over 40 simulated steps.
3. Add a `pin` flag to messages (e.g. `{"pinned": True}`) and make `trim` never drop a
   group containing a pinned message. What should be pinned in a real agent?
