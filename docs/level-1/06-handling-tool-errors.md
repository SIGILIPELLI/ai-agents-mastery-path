---
description: "Classify tool failures, write error messages a model can act on, retry transient faults in code, and watch an agent recover from its own bad call."
---

# 06 · Handling Tool Errors

In a normal program, an exception is a signal to stop. In an agent, most tool errors are
**information**: the model asked for something slightly wrong, and if you tell it clearly
what was wrong, it can usually fix the request on the next step. The skill is deciding
which errors go back to the model, which your code handles itself, and which end the run.

## Four kinds of failure

| Kind | Example | Who should handle it | How |
|---|---|---|---|
| **Model-fixable** | bad date format, unknown ID, missing argument | the model | return a precise error message as the tool result |
| **Transient** | timeout, HTTP 503, rate limit | your code | retry with backoff *inside* the tool wrapper; the model never sees it unless retries run out |
| **Permanent but survivable** | record deleted, permission denied on one file | the model | return the error and let it choose another path or explain |
| **Fatal** | credentials revoked, disk full, sandbox gone | your code | stop the run with a clear reason; don't ask the model to work around it |

Mixing these up causes the classic bad behaviours: an agent that retries a permission
error ten times, or a run that aborts because a network blip surfaced as an exception.

## Writing error messages for a model

A good tool error message has three parts: **what went wrong, why, and what to do
instead.**

```text
Weak:    "ValueError"
Better:  "invalid date"
Good:    "date '28/09/2026' is not in YYYY-MM-DD format; resend as '2026-09-28'"
```

When the valid options are a short list, include them: `"unknown project 'web'; valid
projects: web-frontend, web-api, mobile"`. When they are a long list, suggest the tool
that finds them: `"unknown customer id; use search_customers to find the id first"`.

Keep internal details out. Stack traces, SQL text and hostnames waste tokens, can leak
information, and give the model irrelevant things to fixate on.

## Worked example: an agent that corrects itself

The mock model below deliberately makes the most common real-world mistake — a date in
the wrong format — then reads the error and retries correctly. The tool raises
`ValueError` with a message designed for the model; `execute` in `mini_agent.py`
(lesson 05) turns it into an `{"error": ...}` result.

```python title="self_correct.py"
import re
from tools import tool, registry
from mini_agent import run_agent, call, answer, tool_results

BOOKINGS = {"2026-10-02": ["09:00", "14:00"]}

@tool
def free_slots(date: str):
    """List free one-hour meeting-room slots on a date.

    Args:
        date: Date as YYYY-MM-DD
    """
    if not re.fullmatch(r"\d{4}-\d{2}-\d{2}", date):
        raise ValueError(f"date '{date}' is not in YYYY-MM-DD format; "
                         "resend it in that format, e.g. '2026-10-02'")
    taken = BOOKINGS.get(date, [])
    return {"date": date,
            "free": [f"{h:02d}:00" for h in range(9, 17) if f"{h:02d}:00" not in taken]}

def sloppy_model(messages, schemas):
    results = tool_results(messages)
    if not results:
        return call("free_slots", date="02/10/2026")        # the realistic mistake
    last = results[-1]
    if "error" in last:
        # A real model reads the message and fixes the argument; the mock imitates that.
        return call("free_slots", "c2", date="2026-10-02")
    return answer(f"On {last['date']} the room is free at: {', '.join(last['free'][:4])} "
                  f"(and {len(last['free']) - 4} more slots).")

result = run_agent(sloppy_model, registry(free_slots),
                   "When is the meeting room free on 2 October?")
print("ANSWER:", result["answer"])
print("errors during run:", result["errors"])
```

```text
  step 1: free_slots({"date": "02/10/2026"})
      -> {"error": "ValueError: date '02/10/2026' is not in YYYY-MM-DD format; resend it in that format, e.g....
  step 2: free_slots({"date": "2026-10-02"})
      -> {"date": "2026-10-02", "free": ["10:00", "11:00", "12:00", "13:00", "15:00", "16:00"]}
  step 3: final answer
ANSWER: On 2026-10-02 the room is free at: 10:00, 11:00, 12:00, 13:00 (and 2 more slots).
errors during run: 1
```

The error cost one extra step, not the whole run. Now imagine the tool had raised a bare
`ValueError()` — the model would see `"ValueError: "` and have to guess.

## Retrying transient failures in code

The model should not spend its reasoning on network flakiness. Wrap unreliable tools so
transient errors are retried before the loop ever sees them:

```python title="retrying.py"
import functools
import time

class TransientError(Exception):
    """Raise this (or map HTTP 429/503/timeouts to it) for retry-worthy failures."""

def retry_transient(attempts=3, base_delay=0.2):
    def wrap(fn):
        @functools.wraps(fn)                      # keeps .schema and the name
        def inner(**kwargs):
            for i in range(1, attempts + 1):
                try:
                    return fn(**kwargs)
                except TransientError as e:
                    if i == attempts:
                        raise TransientError(f"still failing after {attempts} attempts: {e}; "
                                             "try again later or use another source")
                    time.sleep(base_delay * 2 ** (i - 1))   # exponential backoff
        return inner
    return wrap

if __name__ == "__main__":
    calls = {"n": 0}

    @retry_transient(attempts=3, base_delay=0.01)
    def flaky_lookup(key):
        calls["n"] += 1
        if calls["n"] < 3:
            raise TransientError("upstream returned 503")
        return {"key": key, "value": 42}

    print(flaky_lookup(key="answer"), "after", calls["n"], "attempts")
```

```text
{'key': 'answer', 'value': 42} after 3 attempts
```

Only retry what is safe to repeat. Retrying a read is harmless; retrying "charge the
card" after a timeout can charge it twice. Level 3 lesson 08 covers idempotency keys,
which make retries of writes safe.

## Stopping on repeated failure

Sometimes the model cannot fix its mistake — it keeps sending the same bad call. Put a
limit on consecutive errors so a confused run ends quickly and cheaply. With the
`guards` hook from lesson 05 this is a few lines:

```python
def too_many_errors(limit=3):
    def guard(state):
        recent = [m for m in state["messages"] if m["role"] == "tool"][-limit:]
        if len(recent) == limit and all('"error"' in m["content"] for m in recent):
            return f"{limit} tool errors in a row"
    return guard

# run_agent(model, tools, task, guards=[too_many_errors(3)])
```

Lesson 07 collects this and the other stop conditions into one reusable module.

## How It Actually Works

The self-correction you just saw is not a special feature. It happens because, after the
error message is appended, the most probable continuation of the conversation is a
corrected call: in training transcripts, an error followed by a fix is far more common
than an error followed by the identical request. The quality of the correction therefore
tracks the **quality of the evidence** in the error text. A message that contains the
expected format and an example gives the model everything it needs to produce the right
tokens; a bare exception name gives it nothing.

The same mechanism explains a failure mode: if the error message is misleading (says
"not found" when the real problem is permissions), the model will "fix" the wrong thing,
confidently. Error text is part of your prompt — review it with the same care.

## Common mistakes

- **Raising through the loop** so a single bad argument kills the run.
- **Swallowing errors silently** — returning `[]` on failure, which the model reads as
  "no results" and reports as fact.
- **Letting the model retry transient errors.** It burns steps and tokens; do it in code.
- **Retrying non-idempotent actions** blindly after a timeout.
- **Leaking stack traces, credentials or internal hostnames** in error text.
- **No consecutive-error limit**, so a stuck model loops until the step cap.

## Exercise

1. In `self_correct.py`, change the tool's error message to just `"bad date"`. The mock
   will still "fix" it (it's scripted) — write two sentences on why a real model might
   not.
2. Add a `book_slot(date, time)` tool that raises `PermissionError("booking rooms
   requires manager approval; ask the user to book it themselves")`. Extend the mock to
   read that error and answer the user with the explanation instead of retrying.
3. Wrap a tool with `retry_transient` and make it fail four times. Confirm that the
   final message the model would see tells it what to do next.
