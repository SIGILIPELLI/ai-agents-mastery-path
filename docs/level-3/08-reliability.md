---
description: "Make agent actions safe under failure: timeouts, retries with backoff and jitter, idempotency keys that prevent double effects, and circuit breakers."
---

# 08 · Reliability: Retries, Timeouts, Idempotency

Agents call networks, and networks fail in the most inconvenient way possible: a request
can **succeed on the server while the response is lost**. The agent sees a timeout,
retries, and now the customer has two refunds. Level 1 retried transient errors for
reads; this lesson makes *writes* safe to retry, and keeps a failing dependency from
dragging every run down with it.

## Timeouts everywhere

Every external call needs a timeout — model calls, tool calls, database queries. Without
one, a single hung connection holds the run (and its resources) forever, and step limits
never get a chance to fire because the step never finishes. Choose timeouts from
observed latency (a high percentile plus margin), not from hope.

## Retries done properly

- **Retry only retryable failures**: timeouts, connection resets, 429 and 5xx responses.
  Never retry validation errors or 4xx responses that will fail identically.
- **Exponential backoff with jitter**: wait `base × 2^attempt`, randomized, so many
  clients recovering at once don't retry in lockstep and knock the service over again.
- **Respect `Retry-After`** headers when the server provides them.
- **Cap attempts and total time**, then surface a clear error to the model (L1-06).
- **Retry at one layer.** If the HTTP client, the tool wrapper and the agent loop all
  retry three times, one failure becomes 27 attempts.

## Idempotency: the key idea

An operation is **idempotent** if doing it twice has the same effect as doing it once.
Reads are naturally idempotent; "refund 49" is not. You make it idempotent by attaching
an **idempotency key** — a unique ID for the *intended action* — and having the server
remember keys it has already processed, returning the original result for a repeat.
Many payment APIs support this pattern natively via a request header; for your own
services, implement it with a table of processed keys.

The key must identify the **action**, not the attempt: generate it once when the agent
decides to act, and reuse it on every retry. A good derivation for agents is a hash of
(run ID, tool name, canonical arguments), so an identical retry within a run maps to the
same key, while a genuinely new request later gets a new one.

## Worked example: the lost-response double refund

```python title="idempotency.py"
import hashlib
import json

class FlakyPaymentService:
    """Processes the refund, then 'loses' the response on the first request."""
    def __init__(self):
        self.refunds, self.seen_keys, self.requests = [], {}, 0

    def refund(self, order_id, amount, idempotency_key=None):
        self.requests += 1
        if idempotency_key and idempotency_key in self.seen_keys:
            return self.seen_keys[idempotency_key]           # replay original result
        self.refunds.append((order_id, amount))
        result = {"refund_id": f"R{len(self.refunds)}", "amount": amount}
        if idempotency_key:
            self.seen_keys[idempotency_key] = result
        if self.requests == 1:
            raise TimeoutError("response lost after processing")
        return result

def with_retries(fn, attempts=3):
    for i in range(attempts):
        try:
            return fn()
        except TimeoutError as e:
            print(f"    attempt {i + 1}: {e}; retrying")
    raise TimeoutError("gave up")

def action_key(run_id, tool, args):
    canonical = json.dumps(args, sort_keys=True)
    return hashlib.sha256(f"{run_id}|{tool}|{canonical}".encode()).hexdigest()[:16]

args = {"order_id": "A-7", "amount": 49.0}

print("naive retry:")
svc = FlakyPaymentService()
with_retries(lambda: svc.refund(**args))
print("  refunds recorded:", svc.refunds)

print("retry with idempotency key:")
svc = FlakyPaymentService()
key = action_key("run-42", "issue_refund", args)      # computed ONCE per action
r = with_retries(lambda: svc.refund(**args, idempotency_key=key))
print("  refunds recorded:", svc.refunds, "| result:", r, "| key:", key)
```

```text
naive retry:
    attempt 1: response lost after processing; retrying
  refunds recorded: [('A-7', 49.0), ('A-7', 49.0)]
retry with idempotency key:
    attempt 1: response lost after processing; retrying
  refunds recorded: [('A-7', 49.0)] | result: {'refund_id': 'R1', 'amount': 49.0} | key: 91329aaacb7155b3
```

The naive retry refunded twice. With the key, the second request was recognised as a
repeat and returned the original result — one refund, and the agent still received a
proper response.

## Circuit breakers

When a dependency is down, every run that calls it waits for timeouts and retries,
multiplying latency and load on the struggling service. A **circuit breaker** tracks
recent failures per dependency; after a threshold it "opens" and fails calls
immediately for a cool-down period, then lets a trial request through ("half-open").
For an agent, an open breaker should produce a fast, explicit tool error — "the billing
service is unavailable; do not retry, tell the user" — so the model stops trying and
reports honestly.

```python title="breaker.py"
import time

class CircuitBreaker:
    def __init__(self, threshold=3, cooldown_s=30):
        self.threshold, self.cooldown_s = threshold, cooldown_s
        self.failures, self.opened_at = 0, None

    def call(self, fn, *args, **kwargs):
        if self.opened_at and time.monotonic() - self.opened_at < self.cooldown_s:
            raise RuntimeError("service unavailable (circuit open); do not retry now")
        try:
            result = fn(*args, **kwargs)
        except Exception:
            self.failures += 1
            if self.failures >= self.threshold:
                self.opened_at = time.monotonic()
            raise
        self.failures, self.opened_at = 0, None          # success closes the circuit
        return result

if __name__ == "__main__":
    breaker = CircuitBreaker(threshold=2, cooldown_s=60)
    def down():
        raise ConnectionError("connection refused")
    for i in range(4):
        try:
            breaker.call(down)
        except Exception as e:
            print(f"call {i + 1}: {type(e).__name__}: {e}")
```

```text
call 1: ConnectionError: connection refused
call 2: ConnectionError: connection refused
call 3: RuntimeError: service unavailable (circuit open); do not retry now
call 4: RuntimeError: service unavailable (circuit open); do not retry now
```

After two real failures the breaker opens, and calls 3 and 4 fail instantly without
touching the dead service.

## Idempotency inside the agent loop

Two agent-specific cases need the same thinking:

- **Resumed graphs** (L2-02): a node that crashed after its side effect but before the
  checkpoint will run again on resume. Derive its idempotency key from the run ID and
  node name.
- **Model repeats itself**: models sometimes call the same write tool twice in one run.
  An action-derived key turns the duplicate into a harmless replay — though you should
  still flag it in trajectory evals (lesson 06).

## How It Actually Works

The underlying problem is that over a network you cannot distinguish "the request never
arrived" from "the request succeeded and the reply was lost" — both look like a timeout
to the caller. Exactly-once *delivery* is therefore impossible to guarantee in general;
what systems achieve instead is **at-least-once delivery plus idempotent processing**,
which yields exactly-once *effects*. The idempotency key is the piece of shared state
that lets the server recognise duplicates. It must be stored atomically with the effect
(same database transaction) — otherwise a crash between "refund done" and "key recorded"
reopens the window.

## Common mistakes

- **Generating a new idempotency key per attempt**, which defeats it entirely.
- **Retrying non-retryable errors** such as validation failures.
- **Retry storms** from retries at several layers, without jitter.
- **No timeouts**, so hangs outlast every other limit.
- **Letting the model handle outages** by retrying in the loop instead of a breaker
  telling it to stop.

## Exercise

1. Add jitter to a retry helper: sleep `random.uniform(0, base * 2**attempt)` and print
   the delays for five attempts.
2. Make `FlakyPaymentService` store keys only *after* a simulated crash point, and show
   how a double refund reappears. Then fix the ordering.
3. Wrap the `issue_refund` tool from L2-08 so it derives an idempotency key from the run
   ID and arguments. Write a mock model that calls it twice and confirm one refund.
