---
description: "Make an agent's final answer machine-checkable: JSON results, the finish-tool pattern, validation, and a bounded repair loop, with runnable code."
---

# 08 · Structured Outputs

An agent's answer is often consumed by code, not a person: a ticket gets a category, a
dashboard gets a number, the next workflow step gets a list of IDs. Free-form prose is
the wrong format for that. This lesson makes the *final* step of the loop as disciplined
as the tool calls in the middle.

## Three ways to get structure

1. **Ask for JSON in the prompt.** Simple and portable; the model usually complies but
   sometimes wraps the JSON in Markdown fences, adds a sentence before it, or omits a
   field. You must parse and validate.
2. **Provider "structured output" / JSON mode.** Many APIs can constrain generation to
   valid JSON, and some to a specific JSON Schema. When available this removes
   *syntax* errors — but still check *values* (an enum can be right and the content
   wrong). Names and capabilities differ by provider, so check current docs.
3. **The finish tool.** Give the agent a tool such as `submit_result(...)` whose
   parameters *are* the output schema, and treat a call to it as the end of the run.
   This reuses the tool-calling machinery the model is already using, which makes it
   the most natural fit for agents.

## The finish-tool pattern

```text
tools: search_orders, get_order, ..., submit_result(status, order_ids, summary)

loop:
  model calls search_orders → result
  model calls get_order     → result
  model calls submit_result(status="refund_needed", order_ids=["A-7"], summary="...")
  → your code validates the arguments; if valid, the run ends with that object
```

Advantages: the output schema is visible to the model the whole time (it's in the tool
list), the arguments go through the same validator as every other tool, and "done" is an
explicit event rather than an inference from "no tool call this turn".

## Validate, then repair — a bounded number of times

Whatever the method, the recipe is the same: **parse, validate, and on failure send the
specific problem back and ask again — at most once or twice.**

```python title="structured.py"
"""Parse and validate a model's JSON answer, with a bounded repair loop."""
import json
import re

def extract_json(text):
    """Pull a JSON object out of a reply that may include fences or chatter."""
    fenced = re.search(r"```(?:json)?\s*(\{.*?\})\s*```", text, re.S)
    candidate = fenced.group(1) if fenced else text[text.find("{"): text.rfind("}") + 1]
    return json.loads(candidate)

def check(obj, spec):
    """spec: {field: (type, allowed_values_or_None)}. Returns a list of problems."""
    problems = []
    for field, (typ, allowed) in spec.items():
        if field not in obj:
            problems.append(f"missing field '{field}'")
        elif not isinstance(obj[field], typ):
            problems.append(f"'{field}' must be {typ.__name__}")
        elif allowed and obj[field] not in allowed:
            problems.append(f"'{field}' must be one of {sorted(allowed)}")
    extra = set(obj) - set(spec)
    if extra:
        problems.append(f"unexpected fields {sorted(extra)}")
    return problems

def ask_structured(model, messages, spec, max_repairs=2):
    """Call the model until its reply parses and validates, or repairs run out."""
    for attempt in range(max_repairs + 1):
        reply = model(messages, [])
        messages.append(reply)
        try:
            obj = extract_json(reply["content"])
            problems = check(obj, spec)
        except (json.JSONDecodeError, ValueError) as e:
            problems = [f"reply was not valid JSON ({e})"]
        if not problems:
            return {"ok": True, "value": obj, "attempts": attempt + 1}
        print(f"  attempt {attempt + 1} rejected: {'; '.join(problems)}")
        messages.append({"role": "user", "content":
            "Your previous reply could not be used: " + "; ".join(problems) +
            ". Reply with only the corrected JSON object."})
    return {"ok": False, "problems": problems, "attempts": max_repairs + 1}
```

Here is a mock that makes the two most common mistakes — Markdown fences plus a value
outside the allowed set — and then fixes them after feedback.

```python title="triage_structured.py"
from mini_agent import answer
from structured import ask_structured

SPEC = {"category": (str, {"billing", "bug", "how_to", "other"}),
        "urgent": (bool, None),
        "summary": (str, None)}

replies = iter([
    answer('Sure! Here is the triage:\n```json\n{"category": "payment", '
           '"urgent": true, "summary": "Charged twice for September"}\n```'),
    answer('{"category": "billing", "urgent": true, '
           '"summary": "Charged twice for September"}'),
])
mock = lambda messages, schemas: next(replies)

messages = [{"role": "user", "content":
             "Triage this ticket as JSON with fields category (billing|bug|how_to|other), "
             "urgent (bool), summary (<= 15 words): 'I was charged twice this month!!'"}]
result = ask_structured(mock, messages, SPEC)
print(result)
```

```text
  attempt 1 rejected: 'category' must be one of ['billing', 'bug', 'how_to', 'other']
{'ok': True, 'value': {'category': 'billing', 'urgent': True, 'summary': 'Charged twice for September'}, 'attempts': 2}
```

The fences were handled by `extract_json` without bothering the model; only the genuine
problem — an invalid category — cost a repair round.

## Worked example: when to *not* repair

Repair loops are for format mistakes. If the model says `"category": "other"` for a
ticket that is obviously billing, that is valid structure with a wrong judgement —
re-asking with "that's wrong" invites a coin-flip. Handle judgement errors with better
instructions, examples, or a review step (Level 2 lesson 06), and measure them with
evaluations (Level 3 lessons 05–06). Keep `max_repairs` small (1–2): if the model can't
produce valid output in three tries, a fourth rarely helps, and you want the failure
visible.

## How It Actually Works

Why do models wrap JSON in fences or add a friendly preface? Because in their training
data, JSON shown to a human in a chat is usually fenced and introduced. The instruction
"reply only with JSON" competes with that habit and usually wins — usually. Constrained
decoding removes the competition: at each step, the sampler only allows tokens that keep
the output a valid prefix of something matching the schema, so fences are literally
impossible to emit.

The finish-tool pattern works for a related reason: tool-call arguments are *already*
generated in the structured mode the model was trained on for tools, and many providers
apply schema constraints to tool arguments. You get structure from machinery the model
uses every step, rather than from an instruction it must remember at the end.

Validation matters even with constraints, because a schema says what is *well-formed*,
not what is *true*. `{"urgent": false}` for "my server is on fire" is perfectly valid.

## Common mistakes

- **`json.loads(reply)` with no fallback** — one stray sentence crashes the pipeline.
- **Unbounded repair loops** that silently multiply cost.
- **Vague repair messages** ("invalid output") instead of the exact problem list.
- **Giant schemas with deep nesting.** Every level is another place to go wrong; flatten
  where you can.
- **Trusting enum validity as correctness.** Evaluate judgement separately.
- **Asking for structure *and* long prose in the same field.** Put free text in a
  dedicated `summary` or `explanation` field.

## Exercise

1. Add a `"confidence": (float, None)` field to `SPEC` and a check that it lies between
   0 and 1. Make the mock return `1.5` first and see the repair message.
2. Convert the example to the finish-tool pattern: write `submit_triage(category,
   urgent, summary)` with the `@tool` decorator (use `Literal` for `category`) and a
   mock that calls it. Detect the call in your loop and end the run with its arguments.
3. Write down one field in a real task of yours where a *valid* value could still be
   dangerously wrong, and how you would detect it.
