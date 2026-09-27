---
description: "Observe agents in production: the metrics that matter, deriving them from traces, percentiles, alert rules, sampling, online evals and privacy."
---

# 02 · Observability in Production

In development you read individual traces (L1-09). In production there are thousands of
runs a day, and nobody reads them one by one. **Observability** means turning those
traces into a handful of numbers that tell you, at a glance, whether the agent is
healthy — and into alerts that tell you when it isn't — while keeping the ability to
drill down into any single run.

## What to measure

Group metrics by the question they answer:

| Question | Metrics |
|---|---|
| Is it working? | run success rate; stop-reason mix (answered / step limit / guard / error); approval rejection rate; user feedback rate |
| Is it fast enough? | run latency p50/p95/p99; time to first event; model latency vs tool latency |
| What does it cost? | tokens and cost per run (mean and p95); model calls per run; cost per tenant per day |
| Is it behaving? | steps per run; tool error rate per tool; repeated-call rate; hard trajectory violations (L3-06) |
| Is it safe? | blocked actions (policy, egress, taint); approval requests; injection-suspect events |

Percentiles matter more than averages. An agent with a 6-second mean latency may have a
p95 of 40 seconds because a few runs loop until the step limit — the average hides
exactly the runs that hurt users and budgets.

## Worked example: from run records to a health report

Each finished run emits a summary record (derived from its trace). The generator below
produces a day of **synthetic** records with a deterministic seed, including a small
cluster of looping runs, so the report has something to find.

```python title="observability.py"
import random
import statistics
from collections import Counter

def synthetic_runs(n=500, seed=3):
    rng = random.Random(seed)
    runs = []
    for i in range(n):
        looping = rng.random() < 0.04
        steps = 12 if looping else rng.randint(2, 6)
        tool_errors = {"search_orders": int(rng.random() < 0.05),
                       "issue_refund": int(rng.random() < 0.01)}
        runs.append({
            "run_id": i,
            "stop": "step_limit" if looping else rng.choices(
                ["answered", "guard", "error"], [0.95, 0.03, 0.02])[0],
            "steps": steps,
            "latency_s": round(steps * rng.uniform(0.8, 1.6), 2),
            "cost": round(steps * rng.uniform(0.9, 1.1) * (1.8 if looping else 1.0), 2),
            "tool_errors": tool_errors,
            "blocked_actions": int(rng.random() < 0.01),
        })
    return runs

def pct(values, p):
    values = sorted(values)
    k = max(0, min(len(values) - 1, round(p / 100 * len(values)) - 1))
    return values[k]

def health(runs):
    stops = Counter(r["stop"] for r in runs)
    lat = [r["latency_s"] for r in runs]
    cost = [r["cost"] for r in runs]
    tool_err = Counter()
    for r in runs:
        tool_err.update(r["tool_errors"])
    return {
        "runs": len(runs),
        "success_rate": stops["answered"] / len(runs),
        "stop_mix": dict(stops),
        "latency_p50": pct(lat, 50), "latency_p95": pct(lat, 95),
        "latency_mean": round(statistics.mean(lat), 2),
        "cost_mean": round(statistics.mean(cost), 2), "cost_p95": pct(cost, 95),
        "tool_error_rate": {t: round(c / len(runs), 3) for t, c in tool_err.items()},
        "blocked_actions": sum(r["blocked_actions"] for r in runs),
    }

ALERTS = [   # (name, condition on the health dict)
    ("success rate below 93%", lambda h: h["success_rate"] < 0.93),
    ("step-limit stops above 2%", lambda h: h["stop_mix"].get("step_limit", 0) / h["runs"] > 0.02),
    ("p95 latency above 12s", lambda h: h["latency_p95"] > 12),
    ("any tool error rate above 4%", lambda h: max(h["tool_error_rate"].values()) > 0.04),
]

if __name__ == "__main__":
    h = health(synthetic_runs())
    for k, v in h.items():
        print(f"{k:<16} {v}")
    fired = [name for name, cond in ALERTS if cond(h)]
    print("ALERTS:", fired or "none")
```

```text
runs             500
success_rate     0.914
stop_mix         {'answered': 457, 'error': 5, 'guard': 17, 'step_limit': 21}
latency_p50      4.55
latency_p95      9.4
latency_mean     5.06
cost_mean        4.66
cost_p95         6.53
tool_error_rate  {'search_orders': 0.042, 'issue_refund': 0.006}
blocked_actions  8
ALERTS: ['success rate below 93%', 'step-limit stops above 2%', 'any tool error rate above 4%']
```

The mean and even the p95 latency look acceptable, and that's the lesson: the looping
runs are only about 4% of traffic, so they sit *above* the 95th percentile and barely
move it. The step-limit share exposes them directly (a p99 would too). Pick metrics that
are sensitive to the failure you care about, not just the conventional ones. The
alert names are phrased as **conditions a human can act on**, and each should link to a
runbook (lesson 09) and to a filtered list of the offending runs' traces.

## Traces, metrics and logs together

- **Metrics** tell you *that* something is wrong (the step-limit rate doubled).
- **Traces** tell you *where* (these runs all loop on `search_orders` with empty results).
- **Logs/events** tell you *what exactly* (the tool started returning an error string the
  model misreads).

Link them with IDs: every metric data point should be traceable to run IDs, every run
to its trace. OpenTelemetry is the common open standard for traces and metrics, and it
has been developing semantic conventions specifically for generative-AI spans (model
calls, token counts, tool calls); many LLM observability products can ingest or export
it. Check the current state of those conventions before depending on specific attribute
names.

## Online evaluation

Offline evals (Level 3) test known cases before release. **Online evaluation** scores a
sample of live runs after the fact:

- Run the trajectory rules (L3-06) on every production trace — they're cheap.
- Run model-graded or human review on a **random sample** plus all flagged runs
  (negative feedback, guard stops, approvals rejected).
- Feed confirmed failures back into the offline dataset, so each incident becomes a
  regression test.

## Privacy and retention

Agent traces are full of user data: messages, documents, tool results. Treat the trace
store as a sensitive system: redact at write time (L1-09), restrict access, set
retention periods, honour deletion requests, and prefer storing metrics and structured
metadata long-term while expiring raw content sooner.

## How It Actually Works

Percentiles need the distribution, not a running average, which is why production
systems store latency in histograms (fixed buckets whose counts can be summed across
servers) and estimate percentiles from them; the exact sort-based calculation above is
fine for a report over one day's records but doesn't aggregate across machines. Alert
rules are just predicates over aggregated metrics evaluated on a schedule; the art is
choosing thresholds from observed baselines so alerts are rare, meaningful and
actionable, rather than firing on normal variation.

Agents add one twist: a large share of failures are **behavioural**, not technical — a
run that "succeeds" by every HTTP metric while quietly taking twelve steps instead of
three. That's why behaviour metrics (steps, repeated calls, trajectory violations) sit
alongside the classic latency/error/traffic signals.

## Common mistakes

- **Dashboards of averages** that hide tail behaviour.
- **Alerts without runbooks**, or on metrics nobody can act on.
- **No link from metric to trace.**
- **Logging everything forever**, creating a privacy liability.
- **Treating HTTP 200 as success** when the agent hit its step limit.

## Exercise

1. Add a per-tool latency field to `synthetic_runs` and an alert on p95 tool latency.
2. Change the seed and loop rate until the step-limit alert stops firing. What loop rate
   is the threshold equivalent to? Is 2% the right number for your use case?
3. Write the runbook entry for "step-limit stops above 2%": first three checks, likely
   causes, and the mitigation you'd try first.
