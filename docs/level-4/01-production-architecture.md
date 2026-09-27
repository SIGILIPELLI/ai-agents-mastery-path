---
description: "Architect agents for production: runs as asynchronous jobs, a durable run store, workers, status lifecycles, approvals and where each control lives."
---

# 01 · Production Agent Architecture

A script that runs an agent in the foreground is fine for a demo. In production, agent
runs take seconds to hours, pause for approvals, survive deploys, and serve many users
at once. That changes the shape of the system: **an agent run becomes a job** with an
ID, a lifecycle and durable state, processed by workers.

## The reference architecture

```text
 Client ──▶ API ──▶ run store (DB)  ◀──────────────┐
  ▲  │        │         ▲                          │
  │  │        └─▶ queue ─┴─▶ worker(s) ──▶ model provider(s)
  │  │                        │  │
  │  │                        │  └──▶ tool gateway ──▶ internal APIs / MCP servers
  │  │                        │            (authz, policy, egress, audit)
  │  └── events (SSE/WebSocket) ◀── event bus ◀─┘
  └───── approvals UI ──▶ API ──▶ resume job
```

- **API** — accepts a task, creates a run record (`queued`), enqueues it, returns the run
  ID immediately. Also serves status, events, and approval decisions.
- **Run store** — the source of truth: run status, checkpoints (messages/graph state),
  pending approvals, results, costs. A relational database is a common, solid choice.
- **Queue + workers** — workers pull runs, execute steps, write checkpoints. Horizontal
  scaling is "add workers"; a crash is "another worker resumes from the last
  checkpoint".
- **Tool gateway** — the single place tool calls pass through: authorization, policy
  (lesson 04), rate limits, idempotency keys, audit logging. Agents never hold raw
  credentials to downstream systems.
- **Event bus** — progress events for streaming UIs (L2-09) and for observability.
- **Model gateway** (optional, often worthwhile) — one place for provider adapters,
  routing, retries, spend limits and usage accounting.

## The run lifecycle

```text
queued ──▶ running ──▶ succeeded
              │  ▲
              │  └────────── resumed
              ▼              │
      awaiting_approval ─────┘──▶ rejected / expired
              │
              ▼
           failed / stopped (budget, guard, error) / cancelled
```

Every transition is written to the run store with a timestamp and reason. The status is
what support staff, dashboards and users see; make the states meaningful and few.

## Worked example: a job-based agent service in miniature

The sketch below uses an in-memory queue, a thread as the worker and a dict as the run
store, so it runs anywhere. Swap them for a real queue and database and the structure
holds.

```python title="agent_service.py"
import json
import queue
import threading
import uuid
from tools import tool, registry
from mini_agent import run_agent, call, answer, tool_results
from approval import gated, with_approvals, Decision

RUNS = {}                                  # run store: id -> record
JOBS = queue.Queue()

def transition(run_id, status, **fields):
    RUNS[run_id].update(status=status, **fields)
    RUNS[run_id]["history"].append(status)

class NeedsApproval(BaseException):
    """Derives from BaseException on purpose: mini_agent.execute turns ordinary
    Exceptions into error results for the model, but a pause must escape the loop."""
    def __init__(self, tool, args):
        self.tool, self.tool_args = tool, args   # not .args: BaseException owns that

@tool
def lookup_account(email: str):
    """Look up an account by email.

    Args:
        email: Customer email
    """
    return {"email": email, "plan": "pro", "seats": 12}

@gated(lambda args: True)
@tool
def change_seats(email: str, seats: int):
    """Change the number of paid seats. Affects billing.

    Args:
        email: Customer email
        seats: New seat count
    """
    return {"email": email, "seats": seats}

def model(messages, schemas):
    r = tool_results(messages)
    if not r:
        return call("lookup_account", email="ana@example.com")
    if len(r) == 1:
        return call("change_seats", "c2", email="ana@example.com", seats=15)
    return answer(f"Seats updated to {r[-1]['seats']}.")

def approver_for(run_id):
    def approver(tool_name, args):
        decision = RUNS[run_id].get("decision")
        if decision is None:
            raise NeedsApproval(tool_name, args)            # pause the run
        return decision
    return approver

def worker():
    while True:
        run_id = JOBS.get()
        if run_id is None:
            break
        rec = RUNS[run_id]
        transition(run_id, "running")
        tools = with_approvals(registry(lookup_account, change_seats),
                               approver_for(run_id), rec["audit"])
        try:
            res = run_agent(model, tools, rec["task"], on_event=lambda e, d: None)
            transition(run_id, "succeeded", result=res["answer"])
        except NeedsApproval as p:
            transition(run_id, "awaiting_approval", pending={"tool": p.tool, "args": p.tool_args})
        JOBS.task_done()

def submit(task):                               # API: create + enqueue, return at once
    run_id = uuid.uuid4().hex[:8]
    RUNS[run_id] = {"task": task, "status": None, "history": [], "audit": []}
    transition(run_id, "queued")
    JOBS.put(run_id)
    return run_id

def decide(run_id, approve):                    # API: approval decision -> resume job
    RUNS[run_id]["decision"] = Decision("approve" if approve else "deny",
                                        reason="approved in UI" if approve else "declined")
    transition(run_id, "queued")
    JOBS.put(run_id)

t = threading.Thread(target=worker, daemon=True)
t.start()
rid = submit("Ana wants 15 seats instead of 12.")
JOBS.join()
print("after submit :", RUNS[rid]["status"], "| pending:", json.dumps(RUNS[rid]["pending"]))
decide(rid, approve=True)
JOBS.join()
print("after approve:", RUNS[rid]["status"], "|", RUNS[rid]["result"])
print("lifecycle    :", " -> ".join(RUNS[rid]["history"]))
JOBS.put(None)
```

```text
after submit : awaiting_approval | pending: {"tool": "change_seats", "args": {"email": "ana@example.com", "seats": 15}}
after approve: succeeded | Seats updated to 15.
lifecycle    : queued -> running -> awaiting_approval -> queued -> running -> succeeded
```

This miniature takes a shortcut worth naming: on resume it re-runs the agent from the
start, relying on the recorded decision to let the gated call through. That's
acceptable here because the only earlier step is a read. A real service resumes from
the **checkpoint** (L2-02) so earlier steps — especially writes — aren't repeated, and
protects any repeated write with an idempotency key (L3-08).

## Where each control lives

| Control | Location | Lesson |
|---|---|---|
| Step, time, token limits | worker loop | L1-07 |
| Spend limits per tenant | model gateway | L3-07 |
| Authorization, scopes | tool gateway | L4-03 |
| Policy rules, approvals | tool gateway + run store | L4-04, L2-08 |
| Idempotency, retries, breakers | tool gateway | L3-08 |
| Tracing, metrics | everywhere, via event bus | L4-02 |
| Kill switch, versions | config service read by workers | L4-09 |

A useful review question for any design: *if the model tried to do X, which component
would stop it?* If the answer is "the prompt", there's a gap.

## How It Actually Works

Turning runs into jobs decouples **accepting work** from **doing work**. The API only
writes a record and a queue message — both fast and durable — so it responds quickly
and can't be tied up by a slow model. Workers can crash, be redeployed or be scaled
without losing runs, because everything a run needs to continue is in the store, not in
a worker's memory. This is the same pattern as any background-job system; agents simply
make it mandatory, because their runs are long, stateful, and interrupted by humans.

Centralising tool calls in a gateway is the other key move. It gives one enforcement
point for security and policy, one place to log every action, and it means the agent's
code never holds downstream credentials — the gateway does, and checks every call
against the *user's* permissions (lesson 03).

## Common mistakes

- **Running agents inside the HTTP request**, so timeouts and deploys kill runs.
- **State only in worker memory.**
- **Tools calling downstream systems directly** with shared service credentials.
- **Too many statuses**, or statuses that don't say why a run stopped.
- **No resume semantics** after approvals or crashes.

## Exercise

1. Add a `cancel(run_id)` API that works in `queued` and `awaiting_approval` states, and
   decide what it should do for a `running` run.
2. Add an `expires_at` to pending approvals and a sweeper that moves stale runs to
   `expired`.
3. Draw your own planned agent on the reference architecture. Mark which components you
   would buy, which you'd build, and which you'd skip for a first release — with reasons.
