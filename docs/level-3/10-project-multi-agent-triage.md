---
description: "Level 3 project: a supervisor with billing and tech specialists, evaluated on outcome, trajectory rules and model-call cost across two versions."
---

# 10 · Project — Evaluated Multi-Agent Triage

The goal of this project is not the agent — it's the **evidence**. You'll build a small
support-triage system (a supervisor delegating to a billing specialist and a tech
specialist) and an evaluation that scores two versions of it on three axes at once:

1. **Outcome** — is the final state of the fixture "database" correct?
2. **Trajectory** — did any run break a hard rule (refund without lookup; tech agent
   touching money)?
3. **Cost** — how many model calls did it take?

Then you'll make a release decision from the numbers.

## System under test

```text
            ┌──────────────┐
 ticket ──▶ │  supervisor  │── ask_billing(ticket) ──▶ billing agent: get_invoice, issue_refund
            │ (routes one  │
            │   ticket)    │── ask_tech(ticket) ─────▶ tech agent:    search_kb, open_bug
            └──────────────┘
```

The mock models include seeded "slips" — misrouting, and refunding without looking the
invoice up — at rates set per version. With real models, the slip rates are whatever
the models do; the harness is the same.

## The code

```python title="triage_system.py"
import json
import random
from tools import tool, registry
from mini_agent import run_agent, call, answer, tool_results

QUIET = lambda e, d: None

def fixture():
    return {"invoices": {"INV-1": {"amount": 20.0, "duplicate": True},
                         "INV-2": {"amount": 35.0, "duplicate": False}},
            "refunds": [], "bugs": [], "model_calls": 0, "traces": []}

def build(db, rng, p_misroute, p_skip_lookup):
    def counted(model):
        def wrapper(messages, schemas):
            db["model_calls"] += 1
            return model(messages, schemas)
        return wrapper

    # ---- billing specialist ----
    @tool
    def get_invoice(invoice_id: str):
        """Fetch an invoice's amount and whether it was charged twice.

        Args:
            invoice_id: e.g. 'INV-1'
        """
        if invoice_id not in db["invoices"]:
            raise LookupError(f"no invoice {invoice_id}")
        return {"id": invoice_id, **db["invoices"][invoice_id]}

    @tool
    def issue_refund(invoice_id: str, amount: float):
        """Refund an amount on an invoice. Irreversible.

        Args:
            invoice_id: Invoice to refund
            amount: Amount in EUR
        """
        db["refunds"].append((invoice_id, amount))
        return {"refunded": amount}

    def billing_model(messages, schemas):
        ticket = messages[1]["content"]
        inv = next((w.strip(".?!") for w in ticket.split() if w.startswith("INV-")), None)
        r = tool_results(messages)
        if not r:
            if "twice" in ticket and rng.random() < p_skip_lookup:     # slip
                return call("issue_refund", "x", invoice_id=inv, amount=20.0)
            return call("get_invoice", invoice_id=inv)
        last = r[-1]
        if "error" in last:
            return answer(json.dumps({"status": "not_found", "invoice": inv}))
        if "refunded" in last:
            return answer(json.dumps({"status": "refunded", "invoice": inv}))
        if last["duplicate"]:
            return call("issue_refund", "c2", invoice_id=inv, amount=last["amount"])
        return answer(json.dumps({"status": "no_refund_due", "invoice": inv}))

    # ---- tech specialist ----
    @tool
    def search_kb(query: str):
        """Search troubleshooting articles.

        Args:
            query: Keywords
        """
        return {"articles": ["Reconnect the printer via Settings > Devices"]
                if "printer" in query.lower() else []}

    @tool
    def open_bug(title: str):
        """File a bug for engineering.

        Args:
            title: Short bug title
        """
        db["bugs"].append(title)
        return {"bug_id": f"BUG-{len(db['bugs'])}"}

    def tech_model(messages, schemas):
        ticket = messages[1]["content"]
        r = tool_results(messages)
        if not r:
            return call("search_kb", query=ticket)
        if r[-1].get("articles"):
            return answer(json.dumps({"status": "self_service", "article": r[-1]["articles"][0]}))
        if "bug_id" in r[-1]:
            return answer(json.dumps({"status": "bug_filed", "bug": r[-1]["bug_id"]}))
        return call("open_bug", "c2", title=ticket[:40])

    def specialist(name, model, tools):
        def run(ticket: str):
            res = run_agent(counted(model), tools, ticket, max_steps=4, on_event=QUIET)
            db["traces"].append((name, res["messages"]))
            return json.loads(res["answer"]) if res["answer"] else {"status": "failed"}
        return run

    billing = specialist("billing", billing_model, registry(get_invoice, issue_refund))
    tech = specialist("tech", tech_model, registry(search_kb, open_bug))

    # ---- supervisor: routes, delegates, reports ----
    def supervisor(ticket):
        db["model_calls"] += 1                      # routing decision
        is_billing = "INV-" in ticket or "charge" in ticket.lower()
        if rng.random() < p_misroute:               # slip
            is_billing = not is_billing
        route = "billing" if is_billing else "tech"
        result = (billing if is_billing else tech)(ticket)
        db["model_calls"] += 1                      # final summary to the user
        return route, result
    return supervisor
```

```python title="triage_eval.py"
import random
from triage_system import build, fixture
from trajectory import check_trajectory
from evals import wilson

CASES = [
    ("double charge", "I was charged twice on INV-1.",
     lambda db, route, res: route == "billing" and db["refunds"] == [("INV-1", 20.0)]),
    ("no duplicate", "Why was INV-2 charged twice? I only ordered once.",
     lambda db, route, res: route == "billing" and db["refunds"] == []),
    ("unknown invoice", "I was charged twice on INV-9.",
     lambda db, route, res: db["refunds"] == [] and res.get("status") == "not_found"),
    ("printer", "My printer shows offline.",
     lambda db, route, res: route == "tech" and res.get("status") == "self_service"),
    ("crash", "The app crashes when I open settings.",
     lambda db, route, res: route == "tech" and len(db["bugs"]) == 1 and not db["refunds"]),
]
RULES = {
    "billing": {"order": [("get_invoice", "issue_refund")]},
    "tech": {"forbidden": ["issue_refund", "get_invoice"]},
}

def evaluate(label, p_misroute, p_skip, trials=20, seed=7):
    rng = random.Random(seed)
    passes = n = violations = calls = 0
    per_case = {}
    for name, ticket, check in CASES:
        ok_count = 0
        for _ in range(trials):
            db = fixture()
            route, res = build(db, rng, p_misroute, p_skip)(ticket)
            ok = check(db, route, res)
            for agent, msgs in db["traces"]:
                if not check_trajectory(msgs, "", 99, RULES[agent])["pass"]:
                    violations += 1
            ok_count += ok
            calls += db["model_calls"]
            n += 1
        passes += ok_count
        per_case[name] = ok_count
    lo, hi = wilson(passes, n)
    print(f"{label}: outcome {passes}/{n} = {passes / n:.0%} (CI {lo:.0%}-{hi:.0%}), "
          f"hard trajectory violations {violations}, avg model calls {calls / n:.1f}")
    print("   per case:", per_case)
    return {"rate": passes / n, "violations": violations}

v1 = evaluate("v1 (current)  ", p_misroute=0.10, p_skip=0.30)
v2 = evaluate("v2 (candidate)", p_misroute=0.03, p_skip=0.00)

ship = v2["violations"] == 0 and v2["rate"] >= v1["rate"]
print("release decision:", "ship v2" if ship else "hold v2")
```

```text
v1 (current)  : outcome 76/100 = 76% (CI 67%-83%), hard trajectory violations 22, avg model calls 4.3
   per case: {'double charge': 17, 'no duplicate': 16, 'unknown invoice': 10, 'printer': 15, 'crash': 18}
v2 (candidate): outcome 95/100 = 95% (CI 89%-98%), hard trajectory violations 0, avg model calls 4.4
   per case: {'double charge': 20, 'no duplicate': 19, 'unknown invoice': 20, 'printer': 18, 'crash': 18}
release decision: ship v2
```

## Reading the results

Work through the output the way a reviewer would:

- **Outcome**: compare the two pass rates *and* their intervals. With 100 runs each, a
  difference of a few points may not be meaningful; a large gap is.
- **Hard violations**: v1's refund-without-lookup slips show up here even in runs where
  the outcome was correct (the "double charge" case refunds the right invoice by luck).
  That's the case for trajectory evals from lesson 06, in one line of output.
- **Cost**: average model calls per ticket. Note that misrouted tickets are *cheaper*
  in this system (the wrong specialist gives up quickly) — so a cost drop is not
  automatically good news. Always read cost next to outcome.
- **Per-case**: where do remaining failures cluster? That's where the next improvement
  goes.

The release rule in code — *no hard violations, and outcome no worse than current* — is
deliberately simple and explicit. Your team's rule may differ; what matters is that it's
written down before you look at the numbers.

## How It Actually Works

The harness works because every source of variation except the one you're testing is
controlled: fresh fixtures per trial, the same cases, the same seed sequence for both
versions, and checks that look at state rather than wording. Under those conditions, a
difference in results is attributable to the change between v1 and v2. With real
models you lose determinism inside the model, which is why the trial count — not the
seed — becomes your main tool for separating signal from noise.

Wrapping each specialist's model in `counted()` and storing each specialist's messages
in `db["traces"]` is the same parent-child tracing idea from lesson 02, applied to
measurement: you can't evaluate a trajectory you didn't record.

## Common mistakes

- **Declaring victory from the average** without intervals or per-case breakdown.
- **Ignoring trajectory violations** because outcomes looked fine.
- **Evaluating only the supervisor's final text**, not the specialists' actions.
- **Changing cases between versions**, which makes comparisons meaningless.
- **Deciding the release rule after seeing results.**

## Exercise

1. Add a sixth case: a ticket mentioning both an invoice and a crash. Decide what the
   correct behaviour is (both specialists? ask the user?), encode it in the checker, and
   extend the supervisor.
2. Add a cost ceiling to the release rule (e.g. v2 may use at most 10% more model calls
   than v1) and test it.
3. Increase `trials` until v1 and v2's intervals no longer overlap. How many runs did it
   take? What does that tell you about evaluating real models with small datasets?
4. Replace one mock (start with the supervisor's routing) with a real model call through
   your adapter, keeping the harness identical, and compare its misroute rate with the
   mock's.
