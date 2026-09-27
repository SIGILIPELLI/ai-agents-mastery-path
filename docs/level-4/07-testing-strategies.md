---
description: "A test pyramid for agents: tool unit tests, adapter contract tests on recorded responses, scripted-model loop tests, policy property tests, eval gates."
---

# 07 · Testing Strategies for Agents

"You can't unit-test an LLM" is true and beside the point. Most of an agent is ordinary
software — tools, dispatchers, adapters, policies, limits, state handling — and most
production incidents come from *that* code, not from the model. Test it the way you'd
test anything else, deterministically, and reserve statistical evaluation for the part
that's genuinely stochastic.

## The agent test pyramid

```text
                 ┌───────────────────────┐
                 │ online eval / canary  │  real traffic, sampled (L4-02, L4-09)
               ┌─┴───────────────────────┴─┐
               │  offline evals (real      │  datasets, trials, pass rates,
               │  model, fixtures)         │  trajectory rules (L3-05/06)
             ┌─┴───────────────────────────┴─┐
             │ loop tests with scripted       │  deterministic: the loop, guards,
             │ models                         │  approvals, error handling
           ┌─┴───────────────────────────────┴─┐
           │ contract tests for model adapters  │  recorded provider responses
         ┌─┴───────────────────────────────────┴─┐
         │ unit tests: tools, validators, policy  │  fast, many, deterministic
         └───────────────────────────────────────┘
```

The lower layers run on every commit in seconds, with no API keys. The upper layers run
before releases (offline evals) and continuously in production (online evals).

## Layer by layer

- **Tool unit tests.** Each tool with valid inputs, invalid inputs, empty results,
  upstream errors. Check error *messages* too — they're prompt content (L1-06).
- **Adapter contract tests.** Record a few real provider responses once (text answer,
  single tool call, parallel tool calls, error) as JSON fixtures, and test that your
  `from_provider_format` converts each correctly. When the provider changes its format,
  refresh fixtures and see exactly what broke.
- **Loop tests with scripted models.** The mock models used throughout this course are
  test doubles. Script the awkward cases: a malformed tool call, an unknown tool, a
  model that loops, a model that ignores a denial. Assert on what your *code* does.
- **Property tests for policy.** Generate many random inputs and assert invariants: "no
  refund above 1000 is ever allowed", "no email outside the customer domain is ever
  allowed". Libraries such as Hypothesis automate this; a seeded loop works too.
- **Offline evals** for the model's behaviour, with pass-rate thresholds as release
  gates.

## Worked example: a deterministic test suite

```python title="test_agent.py"
import json
import random
import sys
import unittest
from tools import tool, registry
from mini_agent import run_agent, call, answer, execute
from guards import repeated_call
from policy import evaluate

quiet = lambda e, d: None

@tool
def get_balance(account: str):
    """Balance of an account.

    Args:
        account: Account id like 'AC-1'
    """
    if not account.startswith("AC-"):
        raise ValueError("account must look like 'AC-<number>'")
    return {"account": account, "balance": 120}

TOOLS = registry(get_balance)

class ToolTests(unittest.TestCase):
    def test_valid(self):
        self.assertEqual(get_balance(account="AC-1")["balance"], 120)

    def test_error_message_is_actionable(self):
        out = execute(TOOLS, {"name": "get_balance", "arguments": '{"account": "1"}'})
        self.assertIn("AC-<number>", out["error"])

class LoopTests(unittest.TestCase):
    def test_malformed_arguments_do_not_crash(self):
        replies = iter([{"role": "assistant", "content": "", "tool_calls": [
            {"id": "x", "name": "get_balance", "arguments": '{"account": '}]},
            answer("sorry")])
        r = run_agent(lambda m, s: next(replies), TOOLS, "t", on_event=quiet)
        self.assertEqual(r["errors"], 1)
        self.assertEqual(r["answer"], "sorry")

    def test_unknown_tool_is_reported(self):
        replies = iter([call("transfer_money", amount=5), answer("done")])
        r = run_agent(lambda m, s: next(replies), TOOLS, "t", on_event=quiet)
        self.assertIn("unknown tool", r["messages"][3]["content"])

    def test_looping_model_is_stopped(self):
        r = run_agent(lambda m, s: call("get_balance", f"c{len(m)}", account="AC-1"),
                      TOOLS, "t", max_steps=50, guards=[repeated_call(3)], on_event=quiet)
        self.assertIsNone(r["answer"])
        self.assertEqual(r["tool_calls"], 3)

class AdapterContractTests(unittest.TestCase):
    RECORDED = {"type": "tool_use_response",        # shape of a stored fixture
                "blocks": [{"kind": "tool", "id": "t1", "name": "get_balance",
                            "input": {"account": "AC-9"}}]}

    @staticmethod
    def from_provider_format(resp):                  # your adapter under test
        calls = [{"id": b["id"], "name": b["name"], "arguments": json.dumps(b["input"])}
                 for b in resp["blocks"] if b["kind"] == "tool"]
        text = "".join(b.get("text", "") for b in resp["blocks"] if b["kind"] == "text")
        return {"role": "assistant", "content": text, "tool_calls": calls}

    def test_tool_call_conversion(self):
        msg = self.from_provider_format(self.RECORDED)
        self.assertEqual(msg["tool_calls"][0]["name"], "get_balance")
        self.assertEqual(json.loads(msg["tool_calls"][0]["arguments"]), {"account": "AC-9"})

class PolicyPropertyTests(unittest.TestCase):
    def test_large_refunds_never_allowed(self):
        rng = random.Random(0)
        ctx = {"customer_domain": "@acme.example", "tainted": False, "hour": 12}
        for _ in range(2000):
            amount = rng.uniform(1000.01, 1e7)
            decision, rule, _ = evaluate("issue_refund", {"amount": amount}, ctx)
            self.assertEqual(decision, "deny", (amount, rule))

    def test_foreign_domains_never_allowed(self):
        rng = random.Random(1)
        for _ in range(500):
            domain = "".join(rng.choice("abcxyz") for _ in range(6)) + ".example"
            ctx = {"customer_domain": "@acme.example", "tainted": rng.random() < 0.5,
                   "hour": rng.randint(0, 23)}
            decision, _, _ = evaluate("send_email", {"to": f"a@{domain}"}, ctx)
            self.assertEqual(decision, "deny")

if __name__ == "__main__":
    unittest.main(testRunner=unittest.TextTestRunner(stream=sys.stdout, verbosity=2),
                  exit=False)
```

```text
test_tool_call_conversion (__main__.AdapterContractTests.test_tool_call_conversion) ... ok
test_looping_model_is_stopped (__main__.LoopTests.test_looping_model_is_stopped) ... ok
test_malformed_arguments_do_not_crash (__main__.LoopTests.test_malformed_arguments_do_not_crash) ... ok
test_unknown_tool_is_reported (__main__.LoopTests.test_unknown_tool_is_reported) ... ok
test_foreign_domains_never_allowed (__main__.PolicyPropertyTests.test_foreign_domains_never_allowed) ... ok
test_large_refunds_never_allowed (__main__.PolicyPropertyTests.test_large_refunds_never_allowed) ... ok
test_error_message_is_actionable (__main__.ToolTests.test_error_message_is_actionable) ... ok
test_valid (__main__.ToolTests.test_valid) ... ok

----------------------------------------------------------------------
Ran 8 tests in 0.003s

OK
```

Eight tests, a fraction of a second, no network, no API key, and they pin down the
behaviours most likely to cause incidents: crashes on malformed output, silent handling
of unknown tools, unbounded loops, adapter drift and policy holes. (The policy tests use
`policy.py` from lesson 04.)

## Eval gates in CI

Offline evals with a real model are slower and cost money, so run them on a schedule and
before releases rather than on every commit. Make them **gates**, with thresholds
agreed in advance: for example, overall pass rate not lower than the current version's
confidence interval, zero hard trajectory violations, cost per task within 10%. Store
results with the version bundle (lesson 09) so you can see trends over time.

## How It Actually Works

The pyramid works because it isolates **sources of nondeterminism**. Everything below
the eval layers replaces the model with a scripted double, so any failure is a
reproducible bug in your code — the same inputs always produce the same result, and the
test can be debugged with a normal debugger. The eval layers deliberately reintroduce
the model and switch from assertions to statistics, because there the question is not
"is this code correct?" but "how often does this system achieve the goal?". Mixing the
two — asserting exact model outputs in unit tests — gives flaky tests that teams learn
to ignore.

## Common mistakes

- **No tests because "it's AI".** Most of the code is not AI.
- **Exact-output assertions on real model calls** — flaky by design.
- **Mocks that only ever behave well.** Script the failures.
- **Adapters without contract tests**, broken silently by provider changes.
- **Evals that aren't gates** — numbers nobody acts on.

## Exercise

1. Add a loop test proving that a denied approval (L2-08) is returned to the model as a
   tool result and that the tool function is never called.
2. Add a recorded fixture with **two** parallel tool calls and a text block, and extend
   the adapter test.
3. Write a property test for the file-organizer sandbox (L1-10): for random path strings
   including `..`, `/`, and symlink-like names, `Sandbox.inside` either raises
   `PermissionError` or returns a path under the root.
