---
description: "Level 3 of AI agents: supervisors and handoffs, MCP, sandboxed code execution, task and trajectory evals, cost, reliability and injection defense."
---

# Level 3 · Advanced <span class="level-badge">Scale & Rigor</span>

By now you can build an agent that works on the examples you tried. Level 3 is about the
gap between "works on my examples" and "works, measurably, on the inputs we will actually
get, at a cost we can afford, without being turned against us".

The level has three threads:

- **Composition** — splitting work across several agents (lessons 01–02), and connecting
  agents to tools through a standard protocol, MCP (lesson 03). Plus the special case of
  agents that write and run code, which demands real isolation (lesson 04).
- **Measurement** — evaluating agents by outcome and by trajectory (lessons 05–06), then
  using those numbers to control cost and latency (lesson 07).
- **Robustness** — retries, timeouts and idempotency so that failures don't double-charge
  anyone (lesson 08), and defending against prompt injection arriving through tools and
  data (lesson 09).

The project (lesson 10) builds a small multi-agent triage system and, more importantly,
the evaluation harness that tells you whether it's any good.

## Modules

1. [Multi-Agent Systems: Why and When](01-multi-agent-systems.md) — what splitting buys you, what it costs, and a test for whether you need it
2. [Supervisors & Handoffs](02-supervisors-handoffs.md) — two coordination patterns, implemented and compared
3. [Model Context Protocol (MCP) Concepts](03-mcp-concepts.md) — hosts, clients, servers, primitives and the trust model
4. [Sandboxing Code-Execution Agents](04-sandboxing-code-execution.md) — timeouts, limits, isolation layers, and what a subprocess can't protect
5. [Evaluating Agents: Task Success](05-eval-task-success.md) — datasets, checkers, repeated trials and pass rates
6. [Trajectory Evaluation](06-trajectory-evaluation.md) — judging *how* the agent got there: required, forbidden and wasted steps
7. [Cost & Latency Control](07-cost-latency.md) — token accounting, routing, caching and parallel tool calls
8. [Reliability: Retries, Timeouts, Idempotency](08-reliability.md) — making actions safe to repeat
9. [Prompt Injection via Tools & Data](09-prompt-injection-defense.md) — how untrusted content steers agents, and layered defenses
10. [Project — Evaluated Multi-Agent Triage](10-project-multi-agent-triage.md) — a supervisor, two specialists and an eval report

## Before you start

- The code continues from Levels 1 and 2 (`tools.py`, `mini_agent.py`, `guards.py`,
  `tracer.py`, `approval.py`). Python 3.9+, standard library only.
- Lesson 04 runs code in a subprocess and uses the Unix-only `resource` module; on
  Windows those limits are skipped and the lesson explains the alternatives.
- Security material in this level is **defensive**: how attacks on agents work in
  principle and how to design so they fail. For broader application security, see the
  [Cybersecurity Mastery Path](https://sigilipelli.github.io/cybersecurity-mastery-path/);
  for LLM-level safety and red-teaming, see the
  [LLM Dev Mastery Path](https://sigilipelli.github.io/llm-dev-mastery-path/).
