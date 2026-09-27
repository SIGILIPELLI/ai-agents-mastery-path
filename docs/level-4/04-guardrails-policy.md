---
description: "Guardrails for agents as reviewable policy: input, action and output checks, allow/deny/approve decisions, rule ordering, testing and ownership."
---

# 04 · Guardrails & Policy as Code

"Guardrails" is used loosely for anything that keeps an AI system inside acceptable
bounds. For agents it's useful to be precise about **where** a check runs and **what
decision** it can make — and to write the rules as data that non-engineers can read,
review and own.

## Three checkpoints

| Checkpoint | Sees | Typical checks | Possible decisions |
|---|---|---|---|
| **Input** | the user's request (and uploaded content) | out-of-scope topics, abusive content, obvious injection attempts, personal data that shouldn't be processed | allow, refuse, route to a human |
| **Action** | a proposed tool call with arguments, plus run context | spending limits, recipients, business hours, data classification, taint (L3-09) | allow, deny, require approval, modify (e.g. cap an amount) |
| **Output** | the final answer or outgoing message | leaked secrets or personal data, unsupported claims, promises the business can't make, tone | allow, redact, block, regenerate |

The **action** checkpoint is the most important for agents, because it's where words
become effects. Input and output checks are valuable but probabilistic (they often use
classifiers or models); action checks can usually be exact.

## Policy as data

Write action rules as a list of declarative entries, evaluated in order, each with an ID,
a condition, a decision and a reason. Benefits: policy owners (finance, legal, support
leads) can review the rules; every decision cites the rule that made it; and the rules
can be tested like code. Dedicated policy engines exist for this (for example, general
purpose policy-as-code tools used in infrastructure); a small evaluator is enough to
learn the idea.

## Worked example: an action policy engine

```python title="policy.py"
"""Declarative action policy: first matching rule wins; default deny."""
from datetime import datetime

RULES = [
    {"id": "R1", "tool": "issue_refund", "when": lambda a, c: a["amount"] <= 0,
     "decision": "deny", "reason": "refund amount must be positive"},
    {"id": "R2", "tool": "issue_refund", "when": lambda a, c: a["amount"] <= 50,
     "decision": "allow", "reason": "small refunds are auto-approved"},
    {"id": "R3", "tool": "issue_refund", "when": lambda a, c: a["amount"] <= 1000,
     "decision": "approve", "reason": "refunds 50-1000 need a team lead"},
    {"id": "R4", "tool": "issue_refund", "when": lambda a, c: True,
     "decision": "deny", "reason": "refunds over 1000 go through finance, not the agent"},
    {"id": "R5", "tool": "send_email",
     "when": lambda a, c: not a["to"].endswith(c["customer_domain"]),
     "decision": "deny", "reason": "agent may only email the ticket's customer domain"},
    {"id": "R6", "tool": "send_email", "when": lambda a, c: c["tainted"],
     "decision": "approve", "reason": "run read untrusted content; human must review"},
    {"id": "R7", "tool": "send_email", "when": lambda a, c: not 8 <= c["hour"] < 20,
     "decision": "approve", "reason": "outside business hours; queue for review"},
    {"id": "R8", "tool": "send_email", "when": lambda a, c: True,
     "decision": "allow", "reason": "routine customer email"},
    {"id": "R9", "tool": "*", "when": lambda a, c: c.get("read_only_tool", False),
     "decision": "allow", "reason": "read-only tools are allowed"},
]

def evaluate(tool, args, ctx):
    for rule in RULES:
        if rule["tool"] in (tool, "*") and rule["when"](args, ctx):
            return rule["decision"], rule["id"], rule["reason"]
    return "deny", "DEFAULT", "no rule allows this action"
```

Policies are code, so they get tests. The table below is both documentation and a test
suite:

```python title="policy_tests.py"
from policy import evaluate

base = {"customer_domain": "@acme.example", "tainted": False, "hour": 14}
CASES = [
    ("issue_refund", {"amount": 20}, {}, "allow"),
    ("issue_refund", {"amount": 400}, {}, "approve"),
    ("issue_refund", {"amount": 5000}, {}, "deny"),
    ("issue_refund", {"amount": -5}, {}, "deny"),
    ("send_email", {"to": "bob@acme.example"}, {}, "allow"),
    ("send_email", {"to": "x@attacker.example"}, {}, "deny"),
    ("send_email", {"to": "bob@acme.example"}, {"tainted": True}, "approve"),
    ("send_email", {"to": "bob@acme.example"}, {"hour": 23}, "approve"),
    ("delete_account", {"id": "u1"}, {}, "deny"),
    ("search_kb", {"q": "x"}, {"read_only_tool": True}, "allow"),
]
failures = 0
for tool, args, extra, expected in CASES:
    decision, rule, reason = evaluate(tool, args, {**base, **extra})
    ok = decision == expected
    failures += not ok
    print(f"{'ok ' if ok else 'BAD'} {tool:<15} {str(args):<28} -> {decision:<8} [{rule}] {reason}")
print("failures:", failures)
```

```text
ok  issue_refund    {'amount': 20}               -> allow    [R2] small refunds are auto-approved
ok  issue_refund    {'amount': 400}              -> approve  [R3] refunds 50-1000 need a team lead
ok  issue_refund    {'amount': 5000}             -> deny     [R4] refunds over 1000 go through finance, not the agent
ok  issue_refund    {'amount': -5}               -> deny     [R1] refund amount must be positive
ok  send_email      {'to': 'bob@acme.example'}   -> allow    [R8] routine customer email
ok  send_email      {'to': 'x@attacker.example'} -> deny     [R5] agent may only email the ticket's customer domain
ok  send_email      {'to': 'bob@acme.example'}   -> approve  [R6] run read untrusted content; human must review
ok  send_email      {'to': 'bob@acme.example'}   -> approve  [R7] outside business hours; queue for review
ok  delete_account  {'id': 'u1'}                 -> deny     [DEFAULT] no rule allows this action
ok  search_kb       {'q': 'x'}                   -> allow    [R9] read-only tools are allowed
failures: 0
```

Three design choices are doing real work here:

- **Default deny.** `delete_account` has no rule, so it's denied. New tools are blocked
  until someone writes policy for them.
- **Order matters, and it's explicit.** The attacker-domain rule (R5) comes before the
  "routine email" rule (R8). Reviewers can read the list top to bottom.
- **Every decision cites a rule and a reason**, which goes into the audit log and — for
  denials — back to the model as the tool result, so it can explain to the user.

## Input and output guardrails

These typically combine cheap deterministic checks (regexes for card numbers or keys,
length limits, allowlists) with classifiers or a model-based check for fuzzier
properties (off-topic, unsafe content). Two practical rules:

- **Fail safe but visibly.** If a guardrail service is down, decide in advance whether the
  agent pauses or proceeds, per risk level, and alert either way.
- **Measure guardrails like models.** A classifier that blocks 10% of legitimate requests
  is a product problem; track false-positive and false-negative rates on a labelled set.

## How It Actually Works

A policy engine separates **decision** from **enforcement**. The gateway (lesson 03)
enforces; the policy function decides; the rules are data. That separation is what
lets you change who may approve refunds without redeploying the agent, test every rule
in isolation, and prove to an auditor which rule allowed a given action — because the
decision record includes the rule ID and the policy version.

First-match-wins with default deny is the same evaluation model as firewall rule lists,
and it has the same failure mode to watch for: a broad rule placed too early silently
shadows more specific rules below it. Tests that cover each rule's intended cases catch
that.

## Common mistakes

- **Guardrails only on text**, none on actions.
- **Default allow**, so new tools are unguarded.
- **Rules scattered through tool code**, invisible to policy owners.
- **No tests for policy**, so a reordering silently changes behaviour.
- **Blocking without telling the model why**, leading to retries or confusing answers.

## Exercise

1. Add a rule that caps refunds per customer per day at 100 total (the context will need
   a `refunded_today` value). Write two test cases for it.
2. Add a `"modify"` decision that caps `amount` at 50 instead of denying, and decide
   whether you'd ever want that in production. Why might silently modifying arguments be
   worse than asking?
3. Move the rules into a JSON or YAML file with conditions written as simple expressions
   (field, operator, value), so a non-programmer could edit them. What do you lose?
