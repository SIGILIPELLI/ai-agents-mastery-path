---
description: "Fit agents into enterprise processes: deterministic pipelines with agent-handled exceptions, systems of record, staged autonomy, audit and success metrics."
---

# 05 · Agentic Workflows in the Enterprise

Inside organisations, agents rarely replace a process end to end. The durable pattern is
narrower and more valuable: **a deterministic workflow handles the routine cases, and an
agent handles the exceptions** — the messy cases that currently land in someone's queue
and take twenty minutes of clicking through three systems. This lesson is about
designing that fit.

## Where agents fit in a process

Map the process first, then look for steps with these properties:

- **Judgement over messy inputs** (free-text emails, mismatched documents, ambiguous
  requests) — hard to code, easy for a person.
- **Investigation across systems** (look up the order, the contract, the shipment) — the
  sequence depends on what's found.
- **A human already reviews the outcome** — so the agent can *prepare* the decision
  rather than make it.
- **Measurable today** — you know current handling time and error rate, so you can show
  improvement.

Steps that are high-volume, rule-based and already automated should stay that way.

## Worked example: invoice exceptions

Accounts-payable teams commonly "three-way match" each supplier invoice against the
purchase order and the goods receipt. Matching invoices are paid automatically;
mismatches go to a person. The agent's job here is **only the mismatches**: investigate
and prepare a recommended resolution for the person, who still decides.

```python title="ap_exceptions.py"
from collections import Counter

INVOICES = [
    {"id": "I1", "po": "P1", "qty": 10, "price": 5.0},
    {"id": "I2", "po": "P2", "qty": 4, "price": 20.0},
    {"id": "I3", "po": "P3", "qty": 12, "price": 5.0},   # billed more than received
    {"id": "I4", "po": "P4", "qty": 3, "price": 110.0},  # price differs from PO
    {"id": "I5", "po": "P5", "qty": 7, "price": 9.0},
    {"id": "I6", "po": "P9", "qty": 1, "price": 50.0},   # unknown PO
    {"id": "I7", "po": "P6", "qty": 2, "price": 15.0},
    {"id": "I8", "po": "P7", "qty": 6, "price": 12.5},
]
POS = {"P1": (10, 5.0), "P2": (4, 20.0), "P3": (12, 5.0), "P4": (3, 100.0),
       "P5": (7, 9.0), "P6": (2, 15.0), "P7": (6, 12.5)}
RECEIVED = {"P1": 10, "P2": 4, "P3": 10, "P4": 3, "P5": 7, "P6": 2, "P7": 6}
PRICE_TOLERANCE = 0.02

def three_way_match(inv):
    """Deterministic rules: the routine path. Returns None if matched, else a reason."""
    if inv["po"] not in POS:
        return "unknown_po"
    po_qty, po_price = POS[inv["po"]]
    if inv["qty"] > RECEIVED.get(inv["po"], 0):
        return "qty_exceeds_receipt"
    if abs(inv["price"] - po_price) / po_price > PRICE_TOLERANCE:
        return "price_mismatch"
    return None

def exception_agent(inv, reason):
    """Stand-in for an agent run with read-only tools (PO, receipts, contracts, email
    history). It prepares a recommendation; it cannot pay, reject or email anyone."""
    if reason == "qty_exceeds_receipt":
        diff = inv["qty"] - RECEIVED[inv["po"]]
        return {"recommendation": f"short-pay: approve {RECEIVED[inv['po']]} units, "
                                  f"request credit note for {diff}",
                "evidence": [f"receipt {inv['po']}: {RECEIVED[inv['po']]} units"]}
    if reason == "price_mismatch":
        return {"recommendation": "hold: price 10% above PO; check contract amendment",
                "evidence": [f"PO {inv['po']} price {POS[inv['po']][1]}",
                             "no amendment found in contract folder"]}
    return {"recommendation": "route to purchasing: PO not found",
            "evidence": [f"searched POs for {inv['po']}: none"]}

auto_paid, review_queue = [], []
for inv in INVOICES:
    reason = three_way_match(inv)
    if reason is None:
        auto_paid.append(inv["id"])
    else:
        review_queue.append({"invoice": inv["id"], "reason": reason,
                             **exception_agent(inv, reason)})

print("auto-paid by rules:", auto_paid)
print("exceptions prepared by agent for human decision:")
for item in review_queue:
    print(f"  {item['invoice']} [{item['reason']}] -> {item['recommendation']}")
    print(f"      evidence: {item['evidence']}")
print("agent runs needed:", len(review_queue), "of", len(INVOICES),
      "| reasons:", dict(Counter(i["reason"] for i in review_queue)))
```

```text
auto-paid by rules: ['I1', 'I2', 'I5', 'I7', 'I8']
exceptions prepared by agent for human decision:
  I3 [qty_exceeds_receipt] -> short-pay: approve 10 units, request credit note for 2
      evidence: ['receipt P3: 10 units']
  I4 [price_mismatch] -> hold: price 10% above PO; check contract amendment
      evidence: ['PO P4 price 100.0', 'no amendment found in contract folder']
  I6 [unknown_po] -> route to purchasing: PO not found
      evidence: ['searched POs for P9: none']
agent runs needed: 3 of 8 | reasons: {'qty_exceeds_receipt': 1, 'price_mismatch': 1, 'unknown_po': 1}
```

The agent ran on three of eight invoices. It had **read-only** tools, produced a
recommendation **with evidence**, and the person kept the decision. That design is easy
to approve internally: nothing the agent does can pay, reject or email anything, and
the value — the investigation is done before a person opens the case — is measurable as
handling time.

## Staged autonomy

Roll out in stages, advancing only when the numbers justify it:

1. **Shadow** — the agent runs on real cases but its output is hidden; compare it with
   what people actually decided.
2. **Assist** — recommendations shown to people, who decide (the example above).
3. **Partial autonomy** — the agent acts on narrow, low-risk categories (e.g. short-pays
   under a threshold where it agreed with humans 99% of the time in the assist stage),
   with sampling review.
4. **Broader autonomy** — only with sustained evidence, policy approval and a rollback
   plan.

Many valuable deployments stop at stage 2 permanently — and that's fine.

## Integration realities

- **Systems of record stay authoritative.** The agent reads from and proposes changes to
  the ERP, CRM or ticketing system through their APIs; it doesn't keep its own copy of
  the truth.
- **Existing approval chains apply.** If a payment needs two approvers today, it does
  when an agent prepares it too.
- **Audit.** Every agent recommendation, the evidence, the human decision and any
  override are recorded where auditors already look.
- **Change management.** People whose work changes should help design the assist view,
  define what "good" looks like, and have a clear way to flag bad recommendations.

## Measuring success

Agree the metrics before building: handling time per exception, recommendation
acceptance rate, error rate of accepted recommendations (found later), backlog age, and
cost per case including model spend. Acceptance rate alone is misleading — people may
accept poor recommendations when busy — so sample accepted cases for quality.

## How It Actually Works

The exceptions-only pattern works because it matches each kind of work to the right
mechanism. Rules are cheap, exact and auditable, so they take the high-volume, well-
specified path. Agents are flexible but costly and probabilistic, so they take the
low-volume, poorly specified path where flexibility is actually needed. Keeping the human
as decision-maker converts the agent's errors from *incorrect actions* into *imperfect
recommendations*, which the existing process already knows how to catch.

## Common mistakes

- **Putting the agent on the happy path** that rules already handle.
- **Agent-owned copies of business data** drifting from the system of record.
- **Skipping the shadow stage**, so there's no baseline to compare with.
- **Measuring only acceptance rate.**
- **Designing without the people who do the work today.**

## Exercise

1. Add an invoice that is a duplicate of I2 (same PO, quantity and price, different ID)
   and a rule that detects it. Should duplicates go to the agent or be handled by rules?
2. Write the assist-view mockup for one exception: what the person sees, in what order,
   and which buttons they have.
3. For a process in your own organisation, identify one exception type that could use
   this pattern and the metric you'd use to prove value.
