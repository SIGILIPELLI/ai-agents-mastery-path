---
description: "Control agent cost and latency: how tokens accumulate across steps, prompt caching, model routing, tool-result caching and parallel tool execution."
---

# 07 · Cost & Latency Control

An agent's cost is not "one model call". It is every model call in the run, each
re-sending a growing history, plus tool latency, plus retries. Teams are regularly
surprised by agent bills because the growth is faster than linear. This lesson shows
where the money and time go, and the levers that actually move them.

## Where cost comes from

For a run of *n* steps where each step adds roughly *d* tokens of history on top of a
fixed prefix *P* (system prompt + tool schemas + task):

```text
input tokens at step k  ≈  P + k·d
total input tokens      ≈  n·P + d·n(n+1)/2      ← grows with n²
```

Output tokens (the model's tool calls and answer) are usually far fewer than input
tokens for agents, but are typically priced higher per token. Your provider's pricing
page gives the per-token rates; they change often, so the calculator below takes them as
parameters rather than hard-coding any real prices.

## Worked example: an agent cost model

```python title="cost_model.py"
"""Estimate tokens and cost of an agent run under different strategies.
Rates are hypothetical inputs, not real prices: substitute your provider's."""

def run_cost(steps, prefix, per_step, out_per_step, in_rate, out_rate,
             cache_discount=0.0, trim_to=None):
    """Return (input_tokens, output_tokens, cost). cache_discount applies to the
    stable prefix after the first call; trim_to caps the history part."""
    total_in = total_out = cost = 0.0
    for k in range(1, steps + 1):
        history = (k - 1) * per_step
        if trim_to is not None:
            history = min(history, trim_to)
        cached = prefix if (cache_discount and k > 1) else 0
        fresh = prefix - cached + history
        total_in += prefix + history
        total_out += out_per_step
        cost += fresh * in_rate + cached * in_rate * (1 - cache_discount) \
                + out_per_step * out_rate
    return int(total_in), int(total_out), cost

IN_RATE, OUT_RATE = 1.0, 4.0            # hypothetical cost units per 1,000 tokens
per_k = lambda r: r / 1000

scenarios = {
    "baseline": {},
    "trim history to 4k": {"trim_to": 4000},
    "cache prefix (90% off)": {"cache_discount": 0.9},
    "both": {"trim_to": 4000, "cache_discount": 0.9},
}
for steps in (5, 20):
    print(f"--- {steps} steps (prefix 3,000 tok, +800 tok/step, 150 out/step) ---")
    base = None
    for name, kw in scenarios.items():
        i, o, c = run_cost(steps, 3000, 800, 150, per_k(IN_RATE), per_k(OUT_RATE), **kw)
        base = base or c
        print(f"  {name:<24} in={i:>7,} out={o:>5,} cost={c:7.1f} units ({c / base:.0%})")
```

```text
--- 5 steps (prefix 3,000 tok, +800 tok/step, 150 out/step) ---
  baseline                 in= 23,000 out=  750 cost=   26.0 units (100%)
  trim history to 4k       in= 23,000 out=  750 cost=   26.0 units (100%)
  cache prefix (90% off)   in= 23,000 out=  750 cost=   15.2 units (58%)
  both                     in= 23,000 out=  750 cost=   15.2 units (58%)
--- 20 steps (prefix 3,000 tok, +800 tok/step, 150 out/step) ---
  baseline                 in=212,000 out=3,000 cost=  224.0 units (100%)
  trim history to 4k       in=128,000 out=3,000 cost=  140.0 units (62%)
  cache prefix (90% off)   in=212,000 out=3,000 cost=  172.7 units (77%)
  both                     in=128,000 out=3,000 cost=   88.7 units (40%)
```

Two things stand out. Going from 5 to 20 steps multiplies cost by much more than 4,
because history is re-sent at every step. And the levers compound: caching the stable
prefix and bounding history attack different parts of the bill.

!!! note "About prompt caching"
    Several providers offer **prompt caching**: if the beginning of a request exactly
    matches a recent request's beginning, that prefix is billed at a discount and
    processed faster. The size of the discount, the minimum length, how long entries
    live and whether you must mark cache points explicitly all vary by provider — the
    90% above is a hypothetical input, not a quoted price. The design rule is universal:
    **keep the prefix byte-for-byte stable** (system prompt, tool schemas, then history
    appended at the end), and avoid putting timestamps or per-request IDs near the top.

## The levers, in rough order of impact

1. **Fewer steps.** Better tools (coarser, returning exactly what's needed) and better
   descriptions cut steps more than any pricing trick. Check your step counts per task
   in traces.
2. **Smaller context per step.** Compact tool results; trim or summarize history
   (L2-03); fewer tool schemas per agent (L3-01).
3. **Prompt caching** of the stable prefix.
4. **Model routing.** Use a smaller, cheaper model for easy decisions (classification,
   routing, simple lookups) and a larger model only for hard steps — or try the small
   model first and escalate on low confidence or failed validation. Measure with your
   evals: routing that lowers cost but also lowers pass rate may not be a win.
5. **Tool-result caching.** Identical read-only calls within a run (or across runs, with
   a TTL) can be served from a cache.
6. **Parallelism** — cuts latency, not cost.

## Latency: parallel tool calls

When a model requests several independent read-only tools in one turn (L1-03), run them
concurrently. The example uses `time.sleep` to stand in for network calls; the timings
printed are real measurements from running it, and will vary slightly on your machine.

```python title="parallel_tools.py"
import time
from concurrent.futures import ThreadPoolExecutor

def fetch_price(sku):
    time.sleep(0.3)                     # stands in for a 300 ms API call
    return {"sku": sku, "price": 10 + len(sku)}

calls = ["kbd-01", "mouse-7", "monitor-27", "dock-usb-c"]

t0 = time.perf_counter()
sequential = [fetch_price(s) for s in calls]
t_seq = time.perf_counter() - t0

t0 = time.perf_counter()
with ThreadPoolExecutor(max_workers=4) as pool:
    parallel = list(pool.map(fetch_price, calls))       # preserves input order
t_par = time.perf_counter() - t0

print(f"sequential: {t_seq:.2f}s  parallel: {t_par:.2f}s  same results: {sequential == parallel}")
```

```text
sequential: 1.23s  parallel: 0.31s  same results: True
```

Only parallelize calls that are **independent and read-only**. Two writes to the same
record, or a write whose arguments depend on a read, must stay sequential. Also respect
the upstream API's rate limits — cap `max_workers`.

## A cost budget guard

Enforce spend per run the same way as step limits (L1-07): track actual usage reported
by the provider on each response, and stop when a run's budget is exhausted. Put a
second budget per user or tenant per day **outside** the agent, so a bug can't exceed it
even in a loop that never checks.

## How It Actually Works

A transformer processes the whole input on every call; there is no memory between calls
except what the provider caches. Prompt caching works by storing the model's internal
attention state (the key/value tensors) for a prefix; a new request whose first N tokens
are identical can reuse that state instead of recomputing it, which is why a single
changed character early in the prompt invalidates everything after it. Output tokens are
generated one at a time, each requiring a forward pass, which is why they are slower and
typically priced higher than input tokens, and why terse tool-call arguments help
latency.

Thread-based parallelism helps here even in Python, because the waiting happens in I/O
(network calls), during which Python releases the global interpreter lock; CPU-bound
tools would need processes instead.

## Common mistakes

- **Estimating cost from one call** instead of a full run.
- **Timestamps or random IDs at the top of the system prompt**, defeating caching.
- **Routing to a cheap model without an eval** showing quality holds.
- **Parallelizing dependent or write calls.**
- **No per-run and per-tenant spend limits.**
- **Optimizing price per token while ignoring step count**, the bigger multiplier.

## Exercise

1. Using `run_cost`, find the step count at which the baseline costs ten times the
   5-step run. How does that change with trimming?
2. Add a `cached_tool` decorator that memoizes read-only tool results per run by
   `(name, sorted args)`, and count hits in the `wasteful` trajectory from lesson 06.
3. Design a routing rule for the Level 2 research agent: which nodes could use a smaller
   model, and which eval numbers would convince you it's safe?
