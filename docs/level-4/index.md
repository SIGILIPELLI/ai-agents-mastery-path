---
description: "Level 4 of AI agents: production architecture, observability, least privilege, policy guardrails, enterprise workflows, testing, ethics, rollout, capstone."
---

# Level 4 · Master <span class="level-badge">Production</span>

Levels 1–3 taught you to build agents and measure them. Level 4 is about running them
for real people and real organisations: where the agent sits in a system, how you know
what it's doing right now, what it is and isn't allowed to do, who answers for it when it
gets something wrong, and how to change it without breaking it.

Much of this level is less about clever code and more about **engineering discipline**:
architecture, permissions, policy, testing, rollout and accountability. Those are the
things that decide whether an agent survives contact with production — and, just as
important, whether an agent should have been built at all (lesson 06).

## Modules

1. [Production Agent Architecture](01-production-architecture.md) — runs as jobs, state stores, workers and the request lifecycle
2. [Observability in Production](02-observability.md) — metrics from traces, dashboards, alerts and sampling
3. [Security & Least Privilege](03-security-permissions.md) — delegated authority, scopes, tenant isolation and confirmation gates
4. [Guardrails & Policy as Code](04-guardrails-policy.md) — input, action and output checks expressed as reviewable rules
5. [Agentic Workflows in the Enterprise](05-enterprise-workflows.md) — agents inside existing processes, exception handling and change management
6. [When NOT to Use an Agent](06-when-not-to-use-agents.md) — a decision framework and the cheaper alternatives
7. [Testing Strategies for Agents](07-testing-strategies.md) — a test pyramid from tool unit tests to eval gates
8. [Ethics & Accountability](08-ethics-accountability.md) — responsibility, transparency, consent and human oversight
9. [Versioning, Rollout & Incident Response](09-rollout-incidents.md) — version bundles, canaries, kill switches and runbooks
10. [Capstone — A Production-Ready Agent](10-capstone.md) — one agent with every control from this course, plus its launch dossier

## Before you start

- The runnable examples continue the same project folder (`tools.py`, `mini_agent.py`,
  `guards.py`, `tracer.py`, `approval.py`, `evals.py`, `trajectory.py`). Python 3.9+,
  standard library only.
- Where a lesson describes cloud services, managed queues or observability products,
  it describes them generically; they can't be run from this page, and specific product
  names are given only as examples of a category.
- Related reading: the [System Design Mastery Path](https://sigilipelli.github.io/system-design-mastery-path/)
  for queues, workers and scaling in general, and the
  [AI Manager Mastery Path](https://sigilipelli.github.io/ai-manager-mastery-path/) for
  the organisational side of adopting AI.
