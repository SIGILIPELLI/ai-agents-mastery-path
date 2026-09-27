---
description: "Reflection and self-critique for agents: generator-critic loops, grounding critiques in checks, bounded rounds, and when reflection helps or hurts."
---

# 06 · Reflection & Self-Critique

Reflection means the agent looks at its own output (or its own trajectory), critiques it,
and tries again. Done well, it catches mistakes a single pass misses. Done badly, it
doubles cost, polishes the wrong thing, or talks itself out of a correct answer. The
difference is almost entirely about **what the critique is grounded in**.

## Three flavours

1. **Self-refine.** The same model drafts, critiques its draft, and revises. Cheap to
   set up; the weakest form, because the critic shares the generator's blind spots.
2. **Separate critic.** A different prompt — sometimes a different model — with a
   rubric reviews the draft. Better separation of concerns; still subjective.
3. **Grounded critique.** The critique is driven by **external signals**: tests pass or
   fail, a linter complains, a validator rejects a field, a word count is over the
   limit, a cited source doesn't contain the quoted claim. This is by far the most
   reliable, because the feedback is true regardless of the model's opinion.

A related idea, sometimes called *Reflexion* after a 2023 paper, stores the lessons from
a failed attempt ("the API needs ISO dates") in memory so the next attempt — or the next
task — starts smarter.

## Worked example: a generator-critic loop with grounded checks

The task: write a customer-facing release note. Requirements that can be checked by code
— at most 45 words, must mention every ticket ID, must not mention internal codenames —
are checked by code. A requirement that can't — "is it clear to a non-technical
customer?" — would go to a model critic with a rubric.

```python title="reflect.py"
import re

TICKETS = ["PAY-112", "PAY-118"]
CODENAMES = ["Falcon", "Project Orca"]

def checks(text):
    """Grounded critique: objective problems found by code."""
    problems = []
    words = len(text.split())
    if words > 45:
        problems.append(f"too long: {words} words, limit 45")
    for t in TICKETS:
        if t not in text:
            problems.append(f"missing ticket {t}")
    for c in CODENAMES:
        if re.search(re.escape(c), text, re.I):
            problems.append(f"mentions internal codename '{c}'")
    return problems

# Mock generator: each revision fixes what the critique listed, like a model given
# precise feedback would. Draft 0 has three realistic problems.
DRAFTS = [
    "This release, built on the new Falcon payment engine, fixes the double-charge bug "
    "(PAY-112) that affected some customers paying by card during peak hours, and "
    "improves the reliability of payment confirmation emails, which previously could "
    "arrive late or not at all for a small number of orders (PAY-118).",
    "This release fixes a bug that could charge some card payments twice (PAY-112) and "
    "makes payment confirmation emails arrive reliably.",
    "This release fixes a bug that could charge some card payments twice (PAY-112) and "
    "makes payment confirmation emails arrive reliably and on time (PAY-118).",
]

def generate(round_no, feedback):
    return DRAFTS[min(round_no, len(DRAFTS) - 1)]

def reflect_loop(max_rounds=3):
    feedback = []
    for r in range(max_rounds + 1):
        draft = generate(r, feedback)
        feedback = checks(draft)
        print(f"round {r}: {len(draft.split())} words, problems: {feedback or 'none'}")
        if not feedback:
            return {"ok": True, "text": draft, "rounds": r}
    return {"ok": False, "text": draft, "problems": feedback, "rounds": max_rounds}

result = reflect_loop()
print(result["ok"], "after", result["rounds"], "revisions:\n ", result["text"])
```

```text
round 0: 48 words, problems: ['too long: 48 words, limit 45', "mentions internal codename 'Falcon'"]
round 1: 20 words, problems: ['missing ticket PAY-118']
round 2: 24 words, problems: none
True after 2 revisions:
  This release fixes a bug that could charge some card payments twice (PAY-112) and makes payment confirmation emails arrive reliably and on time (PAY-118).
```

Notice round 1: fixing length and the codename, the "model" dropped the second ticket —
a very common side effect of revision (fix one thing, break another). Because the
checks run on **every** round, the regression was caught immediately. A critic that only
looked at "what the last feedback asked for" would have missed it.

## Adding a model critic, carefully

For qualities code can't check, a model critic helps if you:

- Give it a **rubric** with specific, independent criteria ("uses no jargon a customer
  wouldn't know; states the benefit, not the implementation").
- Ask for **problems with evidence** (quote the offending phrase), not a score. Scores
  from models drift and cluster; quoted problems are actionable and verifiable.
- Allow **"no problems"** as an explicit, acceptable answer. A critic that must always
  find something will invent issues, and the generator will "fix" correct text.
- Keep the **grounded checks as a gate** after the model critic, so subjective revisions
  can't break objective requirements.

## When reflection doesn't help

- **No new information.** If the critic knows nothing the generator didn't, gains are
  small and inconsistent; published results on self-correction without external
  feedback are mixed, and sometimes negative for reasoning tasks.
- **The first answer was right.** Asking "are you sure?" can make models change correct
  answers. Only trigger reflection on a signal (a failed check, low confidence, a
  high-stakes action).
- **Latency-sensitive paths.** Each round is at least one more model call.

## How It Actually Works

A revision is conditioned on the draft *and the critique*. If the critique contains
information that wasn't available when the draft was produced — a failing test, the true
word count, a missing ID — the revision's most likely continuation shifts toward fixing
it: the model now "sees" the problem in its context. If the critique is itself just
another sample from the same distribution as the draft, it adds noise rather than
information, and the revision is as likely to move away from the right answer as
toward it.

That is why grounding matters so much. Code-based checks inject *facts*; a separate
critic with a rubric injects *a different perspective*; plain self-refinement injects
mostly *variance*. Rank your feedback sources in that order.

## Common mistakes

- **Unbounded "improve until good"** loops. Cap rounds (2–3) and report failure.
- **Only re-checking the last complaint**, letting earlier requirements regress.
- **Critics forced to find a problem**, generating churn.
- **Reflecting on everything**, including trivial or already-verified outputs.
- **Scores instead of evidence** ("7/10") that can't guide a revision.

## Exercise

1. Add a check that the note contains no URLs, and a fourth draft that introduces one.
   Watch the loop catch it.
2. Write the rubric and output format for a model critic that judges customer clarity.
   Include the "no problems" option and a requirement to quote evidence.
3. Implement Reflexion-style memory: when the loop fails, save the final problem list
   with the task type to the `ltm.py` store from lesson 04, and load relevant lessons at
   the start of the next run.
