---
description: "Implement the supervisor and handoff multi-agent patterns in plain Python, with structured handoff contracts, budgets and linked traces."
---

# 02 · Supervisors & Handoffs

Two coordination patterns cover most multi-agent systems you'll need. A **supervisor**
keeps control and delegates sub-tasks; a **handoff** passes control — and the
conversation — to another agent. This lesson builds both on top of `mini_agent.py`.

## Pattern 1: supervisor with agents-as-tools

The cleanest way to build a supervisor is to expose each specialist **as a tool**. The
supervisor's loop is an ordinary agent loop; calling `ask_billing(question)` runs an
entire billing agent and returns its structured result. From the supervisor's point of
view, a specialist is just a slow, smart tool.

Advantages: reuses everything you have (validation, guards, tracing); the supervisor's
context holds only the specialists' *results*, not their internal steps; budgets are
easy to enforce per call.

## Pattern 2: handoffs

In a handoff, the active agent calls a tool such as `transfer_to(agent="billing",
summary=...)`. Your code then switches which agent (system prompt + toolset) handles the
**next** turns of the same conversation. The user keeps talking to "the assistant", but
the assistant's instructions and tools changed underneath.

Advantages: natural for customer-facing chat where the specialist should talk to the
user directly; no central bottleneck. Risks: ping-pong transfers, and context drift if
the whole history is passed along.

## Worked example: both patterns

```python title="multi_agent.py"
import json
from tools import tool, registry
from mini_agent import run_agent, call, answer, tool_results

quiet = lambda e, d: None
log = []                                   # (parent, child, summary) links for tracing

# ----- specialists: each is a complete agent with its own small toolset -----
@tool
def get_invoice(invoice_id: str):
    """Fetch an invoice.

    Args:
        invoice_id: e.g. 'INV-311'
    """
    return {"id": invoice_id, "amount": 49.0, "status": "paid twice"}

@tool
def track_parcel(order_id: str):
    """Tracking status for an order's parcel.

    Args:
        order_id: e.g. 'A-7'
    """
    return {"order_id": order_id, "status": "in transit", "eta": "2026-09-30"}

def billing_model(messages, schemas):
    if not tool_results(messages):
        return call("get_invoice", invoice_id="INV-311")
    inv = tool_results(messages)[-1]
    return answer(json.dumps({"finding": f"{inv['id']} was {inv['status']}",
                              "action_needed": "refund 49.00", "evidence": [inv["id"]]}))

def shipping_model(messages, schemas):
    if not tool_results(messages):
        return call("track_parcel", order_id="A-7")
    t = tool_results(messages)[-1]
    return answer(json.dumps({"finding": f"parcel {t['status']}, ETA {t['eta']}",
                              "action_needed": None, "evidence": [t["order_id"]]}))

SPECIALISTS = {"billing": (billing_model, registry(get_invoice)),
               "shipping": (shipping_model, registry(track_parcel))}

def run_specialist(name, question, max_steps=4):
    model, tools = SPECIALISTS[name]
    r = run_agent(model, tools, question, max_steps=max_steps, on_event=quiet)
    result = json.loads(r["answer"]) if r["answer"] else {"error": r["stopped"]}
    log.append(("supervisor", name, result.get("finding", result.get("error"))))
    return {**result, "steps_used": r["steps"]}

# ----- Pattern 1: supervisor, specialists exposed as tools -----
@tool
def ask_billing(question: str):
    """Delegate a billing question (invoices, charges, refunds) to the billing specialist.
    Returns finding, action_needed and evidence.

    Args:
        question: Self-contained question including any ids
    """
    return run_specialist("billing", question)

@tool
def ask_shipping(question: str):
    """Delegate a delivery question (tracking, addresses) to the shipping specialist.
    Returns finding, action_needed and evidence.

    Args:
        question: Self-contained question including any ids
    """
    return run_specialist("shipping", question)

def supervisor_model(messages, schemas):
    r = tool_results(messages)
    if len(r) == 0:
        return {"role": "assistant", "content": "Two independent questions; delegate both.",
                "tool_calls": [
                    {"id": "b", "name": "ask_billing",
                     "arguments": json.dumps({"question": "Was INV-311 charged twice?"})},
                    {"id": "s", "name": "ask_shipping",
                     "arguments": json.dumps({"question": "Where is order A-7?"})}]}
    b, s = r
    return answer(f"Billing: {b['finding']} (next: {b['action_needed']}). "
                  f"Shipping: {s['finding']}.")

print("PATTERN 1 — supervisor")
res = run_agent(supervisor_model, registry(ask_billing, ask_shipping),
                "I was charged twice on INV-311 and where is my order A-7?", on_event=quiet)
print(" ", res["answer"])
print("  supervisor context:", len(res["messages"]), "messages; links:", log)

# ----- Pattern 2: handoff, control moves to another agent -----
AGENTS = {
    "triage": "Route the user to the right specialist using transfer_to.",
    "billing": "You handle billing. Use billing tools. Speak to the user directly.",
}

@tool
def transfer_to(agent: str, summary: str):
    """Hand the conversation to another agent. Include a summary of what is known.

    Args:
        agent: 'billing' or 'shipping'
        summary: One or two sentences for the next agent
    """
    return {"transferred_to": agent}

def triage_model(messages, schemas):
    return call("transfer_to", agent="billing",
                summary="User reports a double charge on INV-311.")

def run_with_handoffs(user_msg, max_handoffs=2):
    active, history_note = "triage", ""
    for hop in range(max_handoffs + 1):
        if active == "triage":
            r = run_agent(triage_model, registry(transfer_to), user_msg,
                          system=AGENTS["triage"], max_steps=2, on_event=quiet)
            t = json.loads(r["messages"][2]["tool_calls"][0]["arguments"])
            print(f"  hop {hop}: triage -> {t['agent']} with summary: {t['summary']!r}")
            active, history_note = t["agent"], t["summary"]
            continue
        task = f"[Handoff note] {history_note}\n[User] {user_msg}"
        r = run_agent(billing_model, registry(get_invoice), task,
                      system=AGENTS[active], on_event=quiet)
        print(f"  hop {hop}: {active} answered: {json.loads(r['answer'])['finding']}")
        return
    print("  too many handoffs; escalating to a human")

print("PATTERN 2 — handoff")
run_with_handoffs("Hi, I think you charged me twice for INV-311?")
```

```text
PATTERN 1 — supervisor
  Billing: INV-311 was paid twice (next: refund 49.00). Shipping: parcel in transit, ETA 2026-09-30.
  supervisor context: 6 messages; links: [('supervisor', 'billing', 'INV-311 was paid twice'), ('supervisor', 'shipping', 'parcel in transit, ETA 2026-09-30')]
PATTERN 2 — handoff
  hop 0: triage -> billing with summary: 'User reports a double charge on INV-311.'
  hop 1: billing answered: INV-311 was paid twice
```

In pattern 1 the supervisor's context holds **six** messages — system, user, its
delegation (two calls in one message), the two structured results, and its answer — no
matter how many steps each specialist took internally. In pattern 2 the billing agent started fresh with a **handoff note**
rather than the entire prior conversation, and `max_handoffs` bounds ping-pong.

## Handoff contracts

Specialists above return `{"finding", "action_needed", "evidence"}`. Define this contract
explicitly for every boundary:

- **What was found**, in one or two sentences.
- **What action is recommended** (or `null`) — the specialist recommends; the supervisor
  or a gated tool decides.
- **Evidence** — IDs or citations the supervisor can check or show.
- **Status** — done / partial / failed, with a reason.

Validate it like any structured output (L1-08). A specialist that returns free text
invites the supervisor to misread it.

## Budgets across agents

Each specialist has its own `max_steps`, but you also need a **global** budget: total
model calls or tokens across the tree, and a maximum delegation depth (specialists
that can delegate further can recurse without bound). Pass the remaining budget down and
subtract what each child used — `steps_used` in the result makes that possible.

## How It Actually Works

Agents-as-tools works because a tool is anything that turns arguments into a result;
nothing requires it to be fast or deterministic. Wrapping a whole agent run in a tool
gives the supervisor **encapsulation**: it can't see (or be distracted by) the
specialist's internal steps, the same way a function caller can't see the callee's local
variables. That is context isolation implemented with an ordinary function boundary.

Handoffs are a different mechanism: a **state switch** in your code that changes which
system prompt and toolset are active for subsequent turns. The model that calls
`transfer_to` doesn't transfer anything itself; your loop reads the call and swaps
configuration. That's why handoff loops (A → B → A → B) are a code-level bug you prevent
with a counter, not a prompt instruction.

## Common mistakes

- **Passing entire histories between agents**, defeating context isolation.
- **Specialists taking irreversible actions** without the supervisor or a gate knowing.
- **No global budget**, so a tree of agents multiplies cost.
- **Parallel delegation of dependent tasks** (the second needs the first's result).
- **Traces that don't link** parent and child runs — keep `log`-style links with run IDs.

## Exercise

1. Make `ask_billing` fail (raise inside the specialist's tool three times) and have the
   supervisor answer the shipping half and report the billing half as unavailable.
2. Add a global budget: a shared counter of specialist steps, decremented by
   `steps_used`, that makes further delegation return an error once exhausted.
3. Add a `shipping` agent to the handoff example and a triage model that sometimes
   transfers to the wrong one. Implement a transfer-back and confirm `max_handoffs`
   stops a ping-pong.
