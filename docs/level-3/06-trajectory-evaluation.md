---
description: "Evaluate how an agent reached its answer: required and forbidden tools, ordering rules, argument checks, redundant steps and efficiency vs a reference."
---

# 06 · Trajectory Evaluation

Two agents both answer "Refund issued for A-7". One looked up the order, confirmed the
duplicate charge and refunded 49.00. The other refunded 49.00 *without checking
anything* — it happened to guess right. An outcome eval scores them the same. A
**trajectory eval** — which inspects the sequence of steps — tells them apart, and the
second agent is the one that will eventually refund the wrong order.

## What to check in a trajectory

| Check | Question | Example rule |
|---|---|---|
| **Required steps** | Did it do the things it must? | `get_order` called before any refund |
| **Forbidden steps** | Did it avoid what it must never do? | never `delete_customer`; never `send_email` in a read-only task |
| **Ordering** | Did it respect dependencies? | `verify_identity` before `change_email` |
| **Arguments** | Were critical arguments right? | refund `amount` ≤ order amount; `order_id` matches the task |
| **Redundancy** | Did it repeat itself? | same call with same arguments twice |
| **Efficiency** | How many steps vs. a good reference? | ≤ 1.5 × reference step count |
| **Grounding** | Were claims in the answer backed by tool results? | the amount in the answer appears in a tool result |

Some checks are **hard** (a forbidden call is a failure regardless of outcome) and some
are **soft** (inefficiency is a warning, tracked as a metric).

## Exact-match vs. rule-based

You could compare each trajectory against one "golden" sequence of calls. That's usually
too strict: there are often several valid orders, and an extra harmless lookup shouldn't
fail a run. **Rule-based** trajectory checks — like the table above — express what
actually matters and tolerate harmless variation. Use exact matching only for short,
tightly specified procedures.

## Worked example: scoring three trajectories

The trajectories below are in the same shape `run_agent` returns (a list of messages),
so you can run these checks on your own traces.

```python title="trajectory.py"
"""Rule-based trajectory checks over a mini_agent message list."""
import json

def calls(messages):
    out = []
    for m in messages:
        for c in m.get("tool_calls") or []:
            out.append((c["name"], json.loads(c["arguments"] or "{}")))
    return out

def results(messages):
    return [m["content"] for m in messages if m["role"] == "tool"]

def check_trajectory(messages, answer, reference_steps, rules):
    cs = calls(messages)
    names = [n for n, _ in cs]
    hard, soft = [], []
    for tool in rules.get("required", []):
        if tool not in names:
            hard.append(f"missing required call {tool}")
    for tool in rules.get("forbidden", []):
        if tool in names:
            hard.append(f"forbidden call {tool}")
    for before, after in rules.get("order", []):
        if after in names and (before not in names or names.index(before) > names.index(after)):
            hard.append(f"{after} happened without a prior {before}")
    for name, check, msg in rules.get("arg_checks", []):
        for n, args in cs:
            if n == name and not check(args):
                hard.append(msg)
    seen = set()
    for n, args in cs:
        key = (n, json.dumps(args, sort_keys=True))
        if key in seen:
            soft.append(f"repeated call {n}")
        seen.add(key)
    if len(cs) > 1.5 * reference_steps:
        soft.append(f"{len(cs)} calls vs reference {reference_steps}")
    for number in rules.get("grounded_numbers", []):
        if number in answer and not any(number in r for r in results(messages)):
            soft.append(f"answer states {number} but no tool result contains it")
    return {"pass": not hard, "hard": hard, "soft": soft}
```

```python title="trajectory_demo.py"
import json
from trajectory import check_trajectory

def tc(name, **args):
    return {"role": "assistant", "content": "", "tool_calls": [
        {"id": name, "name": name, "arguments": json.dumps(args)}]}

def tr(name, payload):
    return {"role": "tool", "tool_call_id": name, "content": json.dumps(payload)}

ORDER = {"order_id": "A-7", "amount": 49.0, "charged_times": 2}
RULES = {
    "required": ["get_order"],
    "forbidden": ["delete_customer"],
    "order": [("get_order", "issue_refund")],
    "arg_checks": [("issue_refund", lambda a: a.get("order_id") == "A-7",
                    "refund targeted the wrong order"),
                   ("issue_refund", lambda a: a.get("amount", 0) <= 49.0,
                    "refund exceeds the order amount")],
    "grounded_numbers": ["49.00"],
}
ANSWER = "Refunded 49.00 EUR for the duplicate charge on A-7."

trajectories = {
    "careful": [tc("get_order", order_id="A-7"), tr("get_order", ORDER),
                tc("issue_refund", order_id="A-7", amount=49.0),
                tr("issue_refund", {"refunded": "49.00"})],
    "lucky guess": [tc("issue_refund", order_id="A-7", amount=49.0),
                    tr("issue_refund", {"refunded": "49.00"})],
    "wasteful": [tc("get_order", order_id="A-7"), tr("get_order", ORDER),
                 tc("get_order", order_id="A-7"), tr("get_order", ORDER),
                 tc("list_payments", order_id="A-7"), tr("list_payments", {"n": 2}),
                 tc("issue_refund", order_id="A-7", amount=49.0),
                 tr("issue_refund", {"refunded": "49.00"})],
}
for name, msgs in trajectories.items():
    r = check_trajectory(msgs, ANSWER, reference_steps=2, rules=RULES)
    print(f"{name:<12} pass={r['pass']!s:<5} hard={r['hard']} soft={r['soft']}")
```

```text
careful      pass=True  hard=[] soft=[]
lucky guess  pass=False hard=['missing required call get_order', 'issue_refund happened without a prior get_order'] soft=[]
wasteful     pass=True  hard=[] soft=['repeated call get_order', '4 calls vs reference 2']
```

All three produce the same answer, and an outcome check on the refund would pass all
three. The trajectory checks separate them: the lucky guess fails a hard rule, and the
wasteful run passes but gets flagged for review and counted in the efficiency metric.

## Using trajectory results

- **Gate releases on hard rules**: zero forbidden calls and zero ordering violations
  across the eval set is a reasonable bar for agents that take actions.
- **Track soft metrics over time**: average steps per task, repeated-call rate. They
  are early warnings of cost increases and prompt regressions.
- **Mine production traces** with the same checks (Level 4 lesson 02). A forbidden call
  in production is an incident, even if nothing bad happened.

## How It Actually Works

A trajectory eval treats the agent run as a **sequence of state transitions** and checks
properties of the sequence — much like runtime verification of a program, where you
check that "every `open` is followed by a `close`" rather than only checking the final
output. Outcome evals can't see these properties because many different sequences lead
to the same final state, and some of those sequences are safe only by luck.

The rule types map to temporal properties: *required* = "eventually X", *forbidden* =
"never X", *ordering* = "no Y before X". Keeping rules declarative (data, not code
branches) lets domain experts review them — a support lead can read `("get_order",
"issue_refund")` and confirm it's the policy.

## Common mistakes

- **Only outcome evals for action-taking agents**, missing unsafe-but-lucky runs.
- **Golden-sequence matching** so strict that every harmless variation fails.
- **Treating efficiency as a hard failure**, discouraging reasonable verification steps.
- **Rules nobody owns**: policies change; review rules with the people who own the
  policy.

## Exercise

1. Add a rule that `issue_refund` must only happen if a prior `get_order` result showed
   `charged_times >= 2`. You'll need to look at tool *results*, not just calls.
2. Write a trajectory where the agent calls `issue_refund` twice with the same
   arguments. Which check catches it, and should that be hard or soft for a refund?
3. Load a `trace.jsonl` produced by the Level 1 tracer, rebuild the message list from its
   events, and run `check_trajectory` on it.
