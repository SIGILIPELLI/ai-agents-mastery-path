---
description: "Design how an agent stops: final answers, step and time budgets, repeated-call loop detection, error limits and a graceful partial answer."
---

# 07 · Stopping Conditions & Step Limits

An agent that never stops is worse than one that fails: it keeps spending money, keeps
holding resources, and may keep acting on the world. Every agent needs several
independent ways to end, and a sensible thing to say when it ends early.

## The ways a run can end

1. **Success** — the model returns an answer without a tool call (or calls an explicit
   `finish` tool).
2. **Budget exhausted** — steps, tool calls, wall-clock time, tokens or money.
3. **Stuck** — the model repeats the same call, or errors pile up.
4. **Refused** — a policy check or a human rejects the next action (Level 2 lesson 08).
5. **Fatal error** — an environment problem the model cannot work around.

Only the first is decided by the model. All the others are decided by your code, which
is what makes them reliable.

## Choosing limits

There is no universal number. Measure how many steps successful runs of your task take
(lesson 09 shows how to record that), then set the cap comfortably above the typical
case — and well below the point where cost becomes a problem. A simple lookup agent
might need 3–5 steps; a research agent might need 20. If many *successful* runs hit the
cap, the cap is too low or the tools are too fine-grained.

Budgets should be **layered**: a per-run step limit, a per-run time limit, and a
per-user or per-day spending limit enforced outside the agent entirely.

## Worked example: a guards module

Each guard is a function that looks at the run state and returns a reason string to
stop, or `None` to continue. They plug into the `guards` hook of `run_agent`.

```python title="guards.py"
"""Reusable stop conditions for mini_agent.run_agent(guards=[...])."""
import json
import time


def max_tool_calls(limit):
    def guard(state):
        if state["tool_calls"] >= limit:
            return f"tool-call budget of {limit} used up"
    return guard


def too_many_errors(limit=3):
    def guard(state):
        recent = [m for m in state["messages"] if m["role"] == "tool"][-limit:]
        if len(recent) == limit and all('"error"' in m["content"] for m in recent):
            return f"{limit} tool errors in a row"
    return guard


def repeated_call(limit=2):
    """Stop if the exact same tool call (name + arguments) was made `limit` times."""
    def guard(state):
        seen = {}
        for m in state["messages"]:
            for c in m.get("tool_calls") or []:
                key = (c["name"], json.dumps(json.loads(c["arguments"] or "{}"),
                                             sort_keys=True))
                seen[key] = seen.get(key, 0) + 1
                if seen[key] >= limit:
                    return f"repeated call {c['name']}{key[1]} {seen[key]} times"
    return guard


def deadline(seconds):
    start = time.monotonic()
    def guard(state):
        if time.monotonic() - start > seconds:
            return f"time limit of {seconds}s exceeded"
    return guard


def rough_token_budget(max_tokens):
    """Very rough: ~4 characters per token for English text. Use your provider's
    reported usage numbers in production instead of this estimate."""
    def guard(state):
        chars = sum(len(json.dumps(m)) for m in state["messages"])
        if chars / 4 > max_tokens:
            return f"context grew past ~{max_tokens} tokens"
    return guard
```

Now a model that gets stuck. It searches, finds nothing useful, and searches again with
the same query — a real pattern, especially when a tool returns empty results without
guidance.

```python title="stuck_agent.py"
from tools import tool, registry
from mini_agent import run_agent, call, answer
from guards import repeated_call, max_tool_calls, too_many_errors

@tool
def search_docs(query: str):
    """Search the internal wiki. Returns matching page titles.

    Args:
        query: Search keywords
    """
    return {"matches": []}

def stubborn_model(messages, schemas):
    return call("search_docs", f"c{len(messages)}", query="vpn setup linux")

tools = registry(search_docs)

print("Without loop detection:")
r1 = run_agent(stubborn_model, tools, "How do I set up the VPN on Linux?", max_steps=4)
print("  ->", r1["stopped"], "| tool calls:", r1["tool_calls"])

print("With guards:")
r2 = run_agent(stubborn_model, tools, "How do I set up the VPN on Linux?", max_steps=20,
               guards=[repeated_call(2), max_tool_calls(10), too_many_errors(3)])
print("  ->", r2["stopped"], "| tool calls:", r2["tool_calls"])
```

```text
Without loop detection:
  step 1: search_docs({"query": "vpn setup linux"})
      -> {"matches": []}
  step 2: search_docs({"query": "vpn setup linux"})
      -> {"matches": []}
  step 3: search_docs({"query": "vpn setup linux"})
      -> {"matches": []}
  step 4: search_docs({"query": "vpn setup linux"})
      -> {"matches": []}
  stopped: step limit (4) reached
  -> step limit (4) reached | tool calls: 4
With guards:
  step 1: search_docs({"query": "vpn setup linux"})
      -> {"matches": []}
  step 2: search_docs({"query": "vpn setup linux"})
      -> {"matches": []}
  stopped: repeated call search_docs{"query": "vpn setup linux"} 2 times
  -> repeated call search_docs{"query": "vpn setup linux"} 2 times | tool calls: 2
```

With only a step cap, the stuck run burns all four steps. With `repeated_call(2)`, it
stops as soon as the second identical call is recorded — before a third is executed —
even though the step cap was raised to 20.

Notice the guard checks the messages *before* each model call, so the second identical
call was still executed. If a repeated call is expensive or has side effects, check
inside the act step instead, before executing. That is a small change you will make in
the Level 1 project.

## Ending gracefully

"Stopped: step limit reached" is honest, but unhelpful to a user. Two better options:

- **Final summary call.** When a guard fires, call the model once more **with tools
  disabled** and an instruction like: *"You have run out of steps. Summarize what you
  found, what you could not determine, and what the user could try next."* The run
  still ends; the user gets the partial value.
- **Structured failure.** Return `{"status": "incomplete", "reason": ..., "evidence":
  [...]}` so the calling application can decide (retry with a bigger budget, escalate to
  a human, show a message).

Either way, never present a partial result as a complete one.

## How It Actually Works

The model's "decision" to finish is just the probability that its next continuation is
plain text rather than a tool call. Several things push that probability the wrong way:
tool results that look incomplete (empty lists, "truncated"), instructions that
over-emphasize thoroughness ("be exhaustive"), and a long history of tool calls, which
makes another tool call the pattern to continue. So the model's own stop signal is
*useful but unreliable*, and it fails in the expensive direction.

Guards convert that soft signal into hard properties you can state in a design review:
*this run cannot make more than 10 tool calls, cannot run longer than 60 seconds, and
cannot repeat an identical call.* These are invariants of your code, not hopes about
the model.

Loop detection by exact match is deliberately simple. Models often vary a query slightly
("vpn setup linux", "linux vpn setup"), which exact matching misses. Production systems
add a looser check — for example, stopping when N consecutive results add no new
information — but even exact matching catches a surprising share of stuck runs.

## Common mistakes

- **One limit only.** A step cap doesn't bound time if one tool call can hang for ten
  minutes; add timeouts inside tools too.
- **Caps set by guesswork** and never revisited against real traces.
- **Silent truncation.** Returning the last assistant message as if it were the answer
  when the run was actually cut off.
- **Retrying the whole run** automatically on a budget stop, doubling cost for a task
  that is simply too big.
- **Telling the model the limit and trusting it.** "You have 5 steps" in the prompt is a
  helpful hint, not an enforcement mechanism.

## Exercise

1. Change `stubborn_model` so it alternates between two queries. Which guard stops it
   now, and after how many calls? Write a `no_new_results(n)` guard that stops when the
   last `n` tool results were all identical.
2. Implement the **final summary call**: when `run_agent` returns with `stopped` set,
   call the model once more with an empty tool list and a "summarize what you found"
   instruction appended as a user message.
3. For an agent you might build at work, write down the limits you would set (steps,
   seconds, tool calls, daily spend) and how you would measure whether they are right.
