---
description: "Model an agent as a state graph: nodes, conditional edges, shared state, cycles and checkpoints, with a small runnable graph runner and resume."
---

# 02 · Agents as State Graphs

The Level 1 loop has one shape: think, act, repeat. Many real agents have more: classify
the request first, take different paths per category, loop back to revise a draft, pause
for approval, then finish. Writing that as nested `if` statements inside a `for` loop
gets tangled fast. A **state graph** makes the shape explicit.

## The vocabulary

- **State** — one dictionary (or typed object) shared by the whole run: the messages,
  the category, the draft, the approval flag.
- **Node** — a function that receives the state and returns *updates* to it. A node may
  call a model, call a tool, or be plain code.
- **Edge** — "after node A, go to node B".
- **Conditional edge** — "after node A, call a *router* function on the state to choose
  the next node". This is where decisions live.
- **Cycle** — an edge back to an earlier node (e.g. revise → review → revise). The
  agent loop itself is a cycle: `model → tools → model`.
- **Checkpoint** — a saved copy of the state after a node, from which the run can
  resume.

In this view, the Level 1 agent is a two-node graph:

```text
      ┌──────────┐  has tool calls   ┌──────────┐
 ───▶ │  model   │ ────────────────▶ │  tools   │
      └──────────┘ ◀──────────────── └──────────┘
            │ no tool calls
            ▼
           END
```

## Worked example: a small graph runner

Here is a framework-free graph runner with conditional edges, a step cap, checkpoints
written after every node, and the ability to **interrupt** before a chosen node and
**resume** later.

```python title="graph.py"
"""A tiny state-graph runner: nodes, edges, routers, checkpoints, interrupts."""
import json

END = "__end__"

class Graph:
    def __init__(self):
        self.nodes, self.edges, self.routers = {}, {}, {}

    def node(self, name):
        def register(fn):
            self.nodes[name] = fn
            return fn
        return register

    def edge(self, a, b):
        self.edges[a] = b

    def route(self, a, router):
        self.routers[a] = router            # router(state) -> next node name

    def next_after(self, name, state):
        if name in self.routers:
            return self.routers[name](state)
        return self.edges.get(name, END)

    def run(self, state, start, checkpoint=None, interrupt_before=(), max_steps=25,
            log=print):
        current = start
        for _ in range(max_steps):
            if current == END:
                return {"status": "done", "state": state}
            if current in interrupt_before and not state.pop("_resume", False):
                self._save(checkpoint, current, state)
                return {"status": "interrupted", "at": current, "state": state}
            updates = self.nodes[current](state) or {}
            state = {**state, **updates}
            log(f"  [{current}] -> {sorted(updates)}")
            nxt = self.next_after(current, state)
            self._save(checkpoint, nxt, state)
            current = nxt
        return {"status": "step_limit", "at": current, "state": state}

    def resume(self, checkpoint, **changes):
        with open(checkpoint) as f:
            saved = json.load(f)
        state = {**saved["state"], **changes, "_resume": True}
        return saved["next"], state

    @staticmethod
    def _save(path, nxt, state):
        if path:
            with open(path, "w") as f:
                json.dump({"next": nxt, "state": state}, f)
```

Now a support-reply graph. `classify` routes billing questions through an invoice
lookup; `draft` writes a reply; `review` checks it and can send it back for one
revision; `send` is interrupted so a human can look first. The "model" parts are plain
functions so the example runs offline — in a real agent, `classify`, `draft` and
`review` would each call a model.

```python title="support_graph.py"
from graph import Graph, END

g = Graph()

@g.node("classify")
def classify(s):
    text = s["ticket"].lower()
    return {"category": "billing" if "charge" in text or "invoice" in text else "general"}

@g.node("lookup_invoice")
def lookup_invoice(s):
    return {"invoice": {"id": "INV-311", "amount": 49.0, "duplicate": True}}

@g.node("draft")
def draft(s):
    n = s.get("revisions", 0)
    if s["category"] == "billing" and s["invoice"]["duplicate"]:
        text = "We found a duplicate charge on INV-311 and will refund 49.00."
        if n == 0:
            text = "Thanks for writing. " + text          # first draft: too vague on timing
        else:
            text += " The refund reaches your account within 5 business days."
    else:
        text = "Thanks for reaching out; an agent will reply shortly."
    return {"draft": text, "revisions": n + 1}

@g.node("review")
def review(s):
    ok = "business days" in s["draft"] or s["category"] != "billing"
    return {"approved_by_reviewer": ok,
            "review_note": None if ok else "state when the refund arrives"}

@g.node("send")
def send(s):
    return {"sent": True}

g.edge("lookup_invoice", "draft")
g.edge("draft", "review")
g.edge("send", END)
g.route("classify", lambda s: "lookup_invoice" if s["category"] == "billing" else "draft")
g.route("review", lambda s: "send" if s["approved_by_reviewer"] or s["revisions"] >= 2
        else "draft")

print("first run:")
out = g.run({"ticket": "I was charged twice for my invoice!"}, "classify",
            checkpoint="ckpt.json", interrupt_before={"send"})
print(" status:", out["status"], "at", out["at"])
print(" draft awaiting human:", out["state"]["draft"])

print("resumed after human approval:")
nxt, state = g.resume("ckpt.json", human_ok=True)
out = g.run(state, nxt, checkpoint="ckpt.json", interrupt_before={"send"})
print(" status:", out["status"], "| sent:", out["state"].get("sent"),
      "| revisions:", out["state"]["revisions"])
```

```text
first run:
  [classify] -> ['category']
  [lookup_invoice] -> ['invoice']
  [draft] -> ['draft', 'revisions']
  [review] -> ['approved_by_reviewer', 'review_note']
  [draft] -> ['draft', 'revisions']
  [review] -> ['approved_by_reviewer', 'review_note']
 status: interrupted at send
 draft awaiting human: We found a duplicate charge on INV-311 and will refund 49.00. The refund reaches your account within 5 business days.
resumed after human approval:
  [send] -> ['sent']
 status: done | sent: True | revisions: 2
```

The first run stopped **before** `send`, with the entire state saved in `ckpt.json`. The
second run could have happened in a different process, hours later: it loaded the
checkpoint and continued from exactly that node. That property — interrupt, persist,
resume — is the main reason graph frameworks exist.

## The same idea in LangGraph (shape only)

LangGraph expresses this with a typed state, `add_node`, `add_edge`,
`add_conditional_edges`, a compile step that can take a checkpointer, and interrupt
options. A sketch — **not executed here; check current docs for exact names**:

```python
# Shape only.
from langgraph.graph import StateGraph, END

builder = StateGraph(SupportState)                  # SupportState: a TypedDict
builder.add_node("classify", classify)
builder.add_node("draft", draft)
builder.add_conditional_edges("classify", pick_path)  # router returns a node name
builder.add_edge("draft", "review")
graph = builder.compile(checkpointer=some_checkpointer, interrupt_before=["send"])
```

If you understood `graph.py`, you understand what each of those calls configures.

## How It Actually Works

A graph runner is an interpreter for a very small language whose programs are "which
function runs next". Because control flow is **data** rather than Python call stack,
the runner can stop between any two nodes, serialize everything that matters (the
state plus the name of the next node), and later reconstruct the exact point of
execution. An ordinary loop can't do that: its "where am I" lives in local variables and
the program counter, which vanish when the process exits.

Two constraints follow, and frameworks enforce versions of both:

1. **State must be serializable.** No open file handles, clients or lambdas in it —
   store IDs and recreate resources in nodes.
2. **Nodes should be safe to re-run.** A crash after a node's side effect but before
   the checkpoint write means the node runs again on resume. Keep side effects
   idempotent (Level 3 lesson 08), or put them behind an interrupt.

## Common mistakes

- **One giant node** that does everything — you lose the ability to route, interrupt
  and trace between steps.
- **Routers that call the model with no fallback.** If the router's output isn't a
  known node name, fail clearly.
- **Cycles without a counter** (like `revisions` above). Every cycle needs its own limit
  besides the global step cap.
- **Putting non-serializable objects in state.**
- **Side effects in nodes that may re-run** after a resume.

## Exercise

1. Add a `general` path that goes through a `search_faq` node before `draft`, and trace a
   non-billing ticket through the graph.
2. Make the human able to *reject* at the interrupt: resume with `human_ok=False` and
   route to a `handoff_to_agent` node instead of `send`.
3. Draw the Level 1 file-organizer project as a graph. Which node would you interrupt
   before, and what would the human see?
