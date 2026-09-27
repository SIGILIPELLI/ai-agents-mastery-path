---
description: "Add human approval to an agent: gating risky tools, approve/deny/edit decisions, returning denials to the model, and async approvals that resume later."
---

# 08 · Human-in-the-Loop Approvals

Some actions should never happen just because a model decided they should: sending an
email to a customer, issuing a refund, deleting data, merging code, changing
permissions. **Human-in-the-loop (HITL)** means the agent proposes these actions and a
person approves, edits or rejects them before they execute. It is the single most
effective safety control for agents that act in the world.

## What to gate

Classify every tool by what it can affect:

| Class | Examples | Default |
|---|---|---|
| **Read-only** | search, get record, list files | run freely |
| **Reversible write, internal** | create draft, add internal note, write to scratch folder | run freely or log for review |
| **Externally visible** | send email/message, post comment, publish | **approval** |
| **Financial / irreversible / privileged** | refund, payment, delete, change permissions, deploy | **approval**, often with extra checks or two people |

Gate by **tool and arguments**, not only by tool: a refund of 5 might be auto-approved
under a policy, while a refund of 5,000 needs a manager. That policy lives in code.

## Approve, deny, or edit

An approval request should let the human do three things:

- **Approve** — execute exactly as proposed.
- **Deny with a reason** — the reason goes back to the model as the tool result, so it
  can adjust ("customer asked for store credit instead").
- **Edit** — change the arguments (fix the amount, soften the wording) and execute the
  edited version. Record that it was edited.

And it should show enough to decide well: the action and arguments in plain language,
**why** the agent wants to do it (its latest reasoning), what evidence it relied on, and
whether the action is reversible.

## Worked example: an approval gate around tools

```python title="approval.py"
"""Wrap tools so that gated ones ask an approver before executing."""
import functools
import json

class Decision:
    def __init__(self, action, reason="", edited_args=None):
        self.action, self.reason, self.edited_args = action, reason, edited_args

def gated(needs_approval):
    """Mark a tool: needs_approval(args) -> True if a human must approve these args."""
    def mark(fn):
        fn.needs_approval = needs_approval
        return fn
    return mark

def with_approvals(tools, approver, audit):
    wrapped = {}
    for name, fn in tools.items():
        check = getattr(fn, "needs_approval", None)
        if check is None:
            wrapped[name] = fn
            continue
        def make(fn=fn, name=name, check=check):
            @functools.wraps(fn)
            def inner(**args):
                if not check(args):
                    audit.append({"tool": name, "args": args, "decision": "auto"})
                    return fn(**args)
                d = approver(name, args)
                audit.append({"tool": name, "args": args, "decision": d.action,
                              "reason": d.reason, "edited_args": d.edited_args})
                if d.action == "deny":
                    return {"denied": True, "by": "human reviewer", "reason": d.reason,
                            "note": "do not retry this action; tell the user or choose "
                                    "another approach"}
                return fn(**(d.edited_args or args))
            return inner
        wrapped[name] = make()
    return wrapped
```

A refund agent. Refunds of 50 or less are auto-approved by policy; larger ones go to a
human. The approver here is scripted so the page's output is reproducible; in an app it
would be a UI prompt, a chat message with buttons, or a ticket.

```python title="refund_agent.py"
import json
from tools import tool, registry
from mini_agent import run_agent, call, answer, tool_results
from approval import gated, with_approvals, Decision

@tool
def get_order(order_id: str):
    """Look up an order's amount and payment status.

    Args:
        order_id: e.g. 'A-7'
    """
    return {"A-7": {"amount": 49.0, "charged_times": 2},
            "B-9": {"amount": 640.0, "charged_times": 2}}[order_id]

@gated(lambda args: args["amount"] > 50)
@tool
def issue_refund(order_id: str, amount: float, reason: str):
    """Refund money to the customer's original payment method. Irreversible.

    Args:
        order_id: Order to refund
        amount: Amount in EUR
        reason: Short reason shown on the customer's statement
    """
    return {"refunded": amount, "order_id": order_id}

def refund_model(messages, schemas):
    r = tool_results(messages)
    order_id = "A-7" if "A-7" in messages[1]["content"] else "B-9"
    if not r:
        return call("get_order", order_id=order_id)
    if len(r) == 1:
        o = r[0]
        return call("issue_refund", "c2", order_id=order_id, amount=o["amount"],
                    reason="duplicate charge")
    last = r[-1]
    if last.get("denied"):
        return answer(f"I could not refund {order_id}: a reviewer declined "
                      f"({last['reason']}). I've left the case open for the team.")
    return answer(f"Refunded {last['refunded']:.2f} EUR on {order_id}.")

def scripted_approver(tool_name, args):
    print(f"    APPROVAL NEEDED: {tool_name} {json.dumps(args)}")
    return Decision("deny", reason="amount over 500 needs finance sign-off")

audit = []
tools = with_approvals(registry(get_order, issue_refund), scripted_approver, audit)
quiet = lambda e, d: None

for task in ["Customer on order A-7 was charged twice.",
             "Customer on order B-9 was charged twice."]:
    res = run_agent(refund_model, tools, task, on_event=quiet)
    print(task, "\n  ->", res["answer"])
print("audit:", json.dumps(audit, indent=1))
```

```text
Customer on order A-7 was charged twice. 
  -> Refunded 49.00 EUR on A-7.
    APPROVAL NEEDED: issue_refund {"order_id": "B-9", "amount": 640.0, "reason": "duplicate charge"}
Customer on order B-9 was charged twice. 
  -> I could not refund B-9: a reviewer declined (amount over 500 needs finance sign-off). I've left the case open for the team.
audit: [
 {
  "tool": "issue_refund",
  "args": {
   "order_id": "A-7",
   "amount": 49.0,
   "reason": "duplicate charge"
  },
  "decision": "auto"
 },
 {
  "tool": "issue_refund",
  "args": {
   "order_id": "B-9",
   "amount": 640.0,
   "reason": "duplicate charge"
  },
  "decision": "deny",
  "reason": "amount over 500 needs finance sign-off",
  "edited_args": null
 }
]
```

The small refund went through under policy, and the audit log says so (`"auto"`). The
large one paused for a human, was denied with a reason, and the model received the
denial *as a tool result* — so it produced an honest answer instead of claiming
success or retrying.

## Asynchronous approvals

A human may take hours to respond. You can't keep a process blocked that long, so
approval becomes **interrupt and resume** (the checkpoint pattern from lesson 02):

1. The agent proposes a gated action. Your code saves the full state (messages, pending
   call) under an ID, creates an approval request, and ends the run with status
   `awaiting_approval`.
2. The reviewer decides in some UI. The decision is stored against the request ID.
3. A resume job loads the state, executes (or denies) the pending call, appends the
   result, and continues the loop.

Two details matter: **re-validate at resume time** (the order may have been refunded by
someone else in the meantime), and **expire stale requests** so a week-old approval
can't trigger an action on changed facts.

## How It Actually Works

The gate works because tool execution is the only path from the model's words to the
world, and your code owns that path. The model can *propose* anything; it cannot
*execute* anything your wrapper doesn't let through. That's why the gate must be in the
dispatch layer, not in the prompt: "ask before refunding" in a system prompt is a
request the model usually honours; `with_approvals` is a guarantee.

Feeding denials back matters for a subtler reason. If a gated call simply vanished, the
model's context would contain a call with no result, and its most likely continuation
would be to try again or to assume success. A clear result — denied, by whom, why, don't
retry — makes the correct next action (explain and stop) the most likely one.

## Common mistakes

- **Approval by prompt only.** Unenforceable.
- **Rubber-stamp fatigue.** If everything needs approval, humans approve without
  reading. Gate only what matters; auto-approve low-risk cases under explicit policy.
- **Showing raw JSON to approvers.** Render the action in plain language with the
  evidence.
- **Losing the audit trail** of who approved what, including edits.
- **Executing stale approvals** without re-checking current state.

## Exercise

1. Change `scripted_approver` to **edit** the refund to 49.0 when the amount is 640
   (perhaps only one line item was duplicated). Confirm the audit shows the edit and
   the model's answer reflects the edited amount.
2. Add a `send_customer_email(to, body)` tool that always needs approval, and make the
   approver show the first 80 characters of the body.
3. Sketch the data you'd store for an asynchronous approval request (fields, expiry,
   who may approve) and the checks your resume job runs before executing.
