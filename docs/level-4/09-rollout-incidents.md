---
description: "Ship agent changes safely: version bundles of model, prompt, tools and policy, shadow and canary rollouts, kill switches, and an incident runbook."
---

# 09 · Versioning, Rollout & Incident Response

An agent's behaviour depends on more than its code: the model version, the system
prompt, every tool description, the policy rules, the limits. Change any one and
behaviour can shift — sometimes a lot. This lesson treats all of those as one versioned
artefact, rolls changes out gradually, and prepares for the day something goes wrong.

## The version bundle

Define an agent version as the combination of:

- model identifier (exact, pinned — not a "latest" alias),
- system prompt(s) and any templates,
- tool schemas (names, descriptions, parameters),
- policy rules and limits (steps, budgets),
- framework/library versions,
- the eval results that approved it.

Hash the behaviour-defining parts into a single ID, record that ID on every run's trace,
and you can always answer "which version did this?" — and compare versions directly in
your metrics.

## Rollout strategies

1. **Offline eval gate** (L4-07) — nothing ships without passing it.
2. **Shadow** — run the new version on real inputs alongside the current one, with its
   actions disabled (no side effects), and compare outputs and trajectories.
3. **Canary** — route a small, stable slice of traffic (say 5%) to the new version with
   actions enabled; watch the dashboards (L4-02) against the current version.
4. **Progressive rollout** — increase the slice in steps, with automatic rollback if
   alert conditions fire.

Route canaries **by a stable key** (user or tenant ID hash), not randomly per request,
so each user has a consistent experience and conversations don't switch versions
mid-way.

## Worked example: bundle IDs, canary routing and a kill switch

```python title="rollout.py"
import hashlib
import json

def bundle_id(bundle):
    """Stable short hash of everything that defines behaviour."""
    canonical = json.dumps(bundle, sort_keys=True)
    return hashlib.sha256(canonical.encode()).hexdigest()[:10]

CURRENT = {"model": "provider-model-2026-06-01", "prompt": "triage-prompt v7",
           "tools": ["get_invoice", "issue_refund", "search_kb", "open_bug"],
           "policy": "rules-2026-09-10", "max_steps": 8}
CANDIDATE = {**CURRENT, "prompt": "triage-prompt v8"}

FLAGS = {"agent_enabled": True, "canary_percent": 5, "writes_enabled": True}

def bucket(key):
    return int(hashlib.sha256(key.encode()).hexdigest(), 16) % 100

def choose_version(tenant_id):
    if not FLAGS["agent_enabled"]:
        return "disabled"                           # kill switch: fall back to humans
    return "candidate" if bucket(tenant_id) < FLAGS["canary_percent"] else "current"

print("current  :", bundle_id(CURRENT))
print("candidate:", bundle_id(CANDIDATE), "(prompt v7 -> v8 changes the id)")

tenants = [f"tenant-{i}" for i in range(1000)]
share = sum(choose_version(t) == "candidate" for t in tenants) / len(tenants)
print(f"canary share over 1000 tenants: {share:.1%}")
print("tenant-42 twice:", choose_version("tenant-42"), choose_version("tenant-42"))

FLAGS["canary_percent"] = 25
print("after ramp to 25%:", f"{sum(choose_version(t) == 'candidate' for t in tenants) / 10:.1f}%",
      "| tenants in the 5% slice stay in:",
      all(choose_version(t) == "candidate" for t in tenants if bucket(t) < 5))

FLAGS["agent_enabled"] = False
print("kill switch on:", {choose_version(t) for t in tenants})
```

```text
current  : 7f39a7c9e2
candidate: ffaa0defe4 (prompt v7 -> v8 changes the id)
canary share over 1000 tenants: 5.2%
tenant-42 twice: current current
after ramp to 25%: 25.6% | tenants in the 5% slice stay in: True
kill switch on: {'disabled'}
```

Two properties to notice: the same tenant always gets the same version, and ramping from
5% to 25% keeps everyone who was already in the canary — because the bucket is a
deterministic function of the tenant ID, and a larger threshold is a superset.

## Kill switches and degraded modes

Every production agent needs switches that operations staff can flip **without a
deploy**:

- **Agent off** — route everything to the non-agent path (a human queue, a simple form).
- **Writes off** — agent keeps running read-only, preparing recommendations only.
- **Per-tool off** — disable one tool (e.g. `send_email`) while an issue is investigated.
- **Per-tenant off** — for a customer-specific problem.

Test the switches regularly. A kill switch that has never been flipped is a hope.

## Incident response

Agent incidents look different from outages: the service is "up" but doing the wrong
thing — refunding too much, emailing the wrong people, looping expensively. A runbook
entry:

```text
INCIDENT: agent taking harmful or anomalous actions

1. Contain (minutes)   Flip writes_enabled=false (or agent_enabled=false). Confirm in
                       dashboards that side-effecting tool calls dropped to zero.
2. Scope              Query traces by version id and time window: which runs, which
                       tools, which tenants? Export the list of affected actions.
3. Remediate          Reverse actions where possible (refund reversals, message
                       retractions, restoring files) using the audit log and
                       idempotency records. Notify affected users per policy.
4. Diagnose           Diff the version bundle against the last good one. Replay
                       affected traces against both versions offline.
5. Prevent            Add the failing cases to the offline eval set and a trajectory
                       rule that would have caught it. Update the agent card.
6. Review             Blameless write-up within a week; track action items.
```

## How It Actually Works

Hashing the bundle turns a fuzzy question ("what changed?") into an exact one: two runs
with the same bundle ID had the same model, prompt, tools and policy, so behavioural
differences between them are the model's own variance; runs with different IDs can be
compared as an experiment. Hash-based bucketing works because a good hash spreads IDs
uniformly over 0–99, so "bucket < p" selects about p% of tenants, stably and without a
lookup table — the same technique used for feature flags and A/B tests in general.

Kill switches work because they sit **outside** the agent: the worker reads the flag
before each run (or each step), so no model output can override them, and no deploy is
needed to change them.

## Common mistakes

- **Model aliases** that silently change the underlying model.
- **Prompt edits outside version control**, often made "just to test" in production.
- **Random per-request canaries** that flip users between versions.
- **Kill switches that require a deploy** or were never tested.
- **Incidents closed without new eval cases**, so the same failure can return.

## Exercise

1. Add `writes_enabled` handling: when false, the tool gateway should convert every
   write into a "prepared, not executed" result. Test it with the Level 2 refund agent.
2. Implement automatic rollback: given the health dicts (L4-02) for current and
   candidate, set `canary_percent = 0` if the candidate's success rate is more than 3
   points lower or any hard violations appear.
3. Write the runbook entry for "agent costs doubled overnight", including which bundle
   fields you'd diff first.
