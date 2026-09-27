---
description: "Evaluate agents by outcome: build a task dataset with programmatic checkers, run repeated trials, and report pass rates, consistency and uncertainty."
---

# 05 · Evaluating Agents: Task Success

"It worked when I tried it" is not an evaluation. Agents are stochastic, multi-step and
sensitive to small changes in prompts, tools and models, so you need a way to answer
questions like *did this prompt change make things better?* and *is the new model safe to
switch to?* with numbers. This lesson builds the outcome side of that: **did the agent
accomplish the task?** Lesson 06 adds the process side.

## Anatomy of an agent eval

1. **Cases** — realistic tasks, each with a known correct outcome. Include easy cases,
   hard cases, and cases where the right behaviour is to refuse or ask.
2. **Environment** — the tools the agent uses, backed by **fixtures**: fake databases,
   recorded API responses, a temp folder. Evals must not touch production.
3. **Checker** — code that decides pass/fail by inspecting the *final state* and the
   answer: was the refund issued with the right amount? Is the file in the right folder?
   Does the answer contain the correct number?
4. **Trials** — each case run several times, because one run tells you little.
5. **Report** — pass rates with uncertainty, broken down by case and category.

## Check the state, not just the words

Where possible, check **effects**, not phrasing. For "cancel order A-7", check that order
A-7's status in the fixture database is `cancelled` and nothing else changed — don't
check that the answer contains the word "cancelled". State-based checks are objective,
robust to wording, and catch the dangerous failure where the agent *says* it did
something it didn't.

When the output is text that must be judged (a summary, an explanation), use a rubric —
and, if you use a model as the judge, validate it against human labels on a sample first.
LLM-as-judge methods and their biases are covered in the
[Prompt Engineering Mastery Path](https://sigilipelli.github.io/prompt-engineering-mastery-path/).

## Worked example: an eval harness with repeated trials

The agent under test is a mock with deliberate, seeded randomness: it sometimes picks
the wrong order ID or forgets a step, like a real model occasionally does. Seeding makes
this page's output reproducible; with a real model you'd see different numbers every
time you run it — which is exactly why you run many trials.

```python title="evals.py"
"""Outcome-based agent eval harness: cases, fixtures, checkers, trials, report."""
import math
import random

def wilson(passes, n, z=1.96):
    """95% Wilson score interval for a pass rate — honest error bars for small n."""
    if n == 0:
        return (0.0, 0.0)
    p = passes / n
    centre = (p + z * z / (2 * n)) / (1 + z * z / n)
    half = z * math.sqrt(p * (1 - p) / n + z * z / (4 * n * n)) / (1 + z * z / n)
    return (max(0.0, centre - half), min(1.0, centre + half))

def run_eval(agent, cases, trials=5, seed=0):
    rng = random.Random(seed)
    rows = []
    for case in cases:
        outcomes = []
        for t in range(trials):
            db = case["fixture"]()                          # fresh state every trial
            answer = agent(case["task"], db, rng)
            outcomes.append(case["check"](db, answer))
        rows.append({"id": case["id"], "passes": sum(outcomes), "trials": trials,
                     "all_passed": all(outcomes)})
    total = sum(r["passes"] for r in rows)
    n = sum(r["trials"] for r in rows)
    return {"rows": rows, "pass_rate": total / n, "ci95": wilson(total, n),
            "all_k": sum(r["all_passed"] for r in rows) / len(rows)}

def print_report(name, rep):
    lo, hi = rep["ci95"]
    print(f"{name}: pass rate {rep['pass_rate']:.0%} (95% CI {lo:.0%}-{hi:.0%}), "
          f"cases passing all trials {rep['all_k']:.0%}")
    for r in rep["rows"]:
        bar = "#" * r["passes"] + "." * (r["trials"] - r["passes"])
        print(f"   {r['id']:<18} {bar}")
```

```python title="eval_demo.py"
from evals import run_eval, print_report

def orders():
    return {"A-7": {"status": "open", "amount": 49.0},
            "B-9": {"status": "open", "amount": 640.0},
            "C-2": {"status": "shipped", "amount": 15.0}}

CASES = [
    {"id": "cancel-open", "task": "Cancel order A-7", "fixture": orders,
     "check": lambda db, a: db["A-7"]["status"] == "cancelled"
                            and all(db[k]["status"] != "cancelled" for k in ("B-9", "C-2"))},
    {"id": "cancel-shipped", "task": "Cancel order C-2", "fixture": orders,
     # correct behaviour: refuse — shipped orders can't be cancelled
     "check": lambda db, a: db["C-2"]["status"] == "shipped" and "cannot" in a.lower()},
    {"id": "amount-lookup", "task": "How much was order B-9?", "fixture": orders,
     "check": lambda db, a: "640" in a},
]

def make_agent(p_wrong_id, p_ignore_rule):
    """Mock agent. A real one would be run_agent(model, tools, task) against fixtures."""
    def agent(task, db, rng):
        oid = task.split()[-1].rstrip("?")
        if rng.random() < p_wrong_id:                 # realistic slip: wrong record
            oid = "B-9"
        if task.startswith("Cancel"):
            if db[oid]["status"] == "shipped" and rng.random() >= p_ignore_rule:
                return f"Order {oid} has shipped, so I cannot cancel it."
            db[oid]["status"] = "cancelled"
            return f"Cancelled {oid}."
        return f"Order {oid} was {db[oid]['amount']:.2f} EUR."
    return agent

print_report("prompt v1", run_eval(make_agent(0.15, 0.4), CASES, trials=10, seed=1))
print_report("prompt v2", run_eval(make_agent(0.05, 0.1), CASES, trials=10, seed=1))
```

```text
prompt v1: pass rate 73% (95% CI 56%-86%), cases passing all trials 33%
   cancel-open        #######...
   cancel-shipped     #####.....
   amount-lookup      ##########
prompt v2: pass rate 83% (95% CI 66%-93%), cases passing all trials 33%
   cancel-open        #########.
   cancel-shipped     ######....
   amount-lookup      ##########
```

Read the report the way you'd read any experiment:

- **The confidence intervals are wide** at 30 trials per version. If they overlap
  heavily, you haven't shown a difference yet — run more trials or add cases.
- **"Cases passing all trials"** is a consistency measure. An agent that succeeds 80% of
  the time on every case is very different from one that always succeeds on some cases
  and always fails on others, even at the same average. (Some benchmarks call the
  probability of succeeding on all *k* attempts "pass^k".)
- **Per-case rows** show *where* it fails. `cancel-shipped` is a policy case — the
  important one to get right, and the one v1 most often got wrong.

## Building the dataset

- **Start from real traffic** (anonymised) and from your traces of failures (L1-09).
- **Cover the categories**: happy paths, edge cases, cases requiring refusal or
  clarification, adversarial inputs (lesson 09), multi-step tasks.
- **Aim for tens of cases early, hundreds later.** Twenty good cases beat none; a few
  hundred make regressions visible.
- **Keep a held-out set** you don't tune prompts against, to detect overfitting to your
  eval.
- **Version the dataset** alongside prompts and code.

## How It Actually Works

Each trial is a draw from the agent's distribution of behaviours on that case. The pass
rate you measure is an **estimate** of the true success probability, and its precision
grows only with the square root of the number of trials: to halve the error bar you need
four times as many runs. The Wilson interval is used instead of the simple
"p ± 1.96·√(p(1−p)/n)" formula because the simple one behaves badly near 0% and 100% and
for small n — exactly the regime of agent evals.

Fresh fixtures per trial matter because agents *change state*. If trial 1 cancels order
A-7 and trial 2 reuses the same database, trial 2 is testing a different situation. The
`fixture()` call returning a new object is the eval equivalent of a test's setup method.

## Common mistakes

- **One run per case.** You're measuring noise.
- **Checking wording instead of effects.**
- **Evaluating against production systems**, with real side effects.
- **Only happy-path cases**, so the eval can't see policy violations.
- **Tuning prompts on the whole dataset** until it passes, then being surprised in
  production.
- **Reporting a single number** without per-case breakdown or uncertainty.

## Exercise

1. Add a case "Cancel order Z-1" (doesn't exist) whose correct outcome is an answer
   saying the order wasn't found and no state changes. Extend the mock so it sometimes
   crashes on it, and make the harness count a crash as a failure rather than stopping.
2. Run v1 and v2 with 50 trials each. Do the intervals still overlap?
3. Replace the mock with `run_agent` and real tools over the fixture database (mock
   model first, then — if you have one — a real model). Keep the checkers unchanged.
