---
description: "What agent frameworks such as LangChain and LangGraph provide, what they cost, how their concepts map to a hand-written loop, and how to choose."
---

# 01 · Frameworks: When and Which

You have written an agent in sixty lines. So why do agent frameworks exist, and when is
adopting one the right call? This lesson maps framework concepts onto the loop you
already know, so you can evaluate any library on its merits rather than its marketing.

## What a framework actually gives you

Strip away the branding and agent frameworks offer some combination of:

| Capability | Your Level 1 equivalent | Why a framework might do it better |
|---|---|---|
| Provider adapters | `real_model` adapter (lesson 05) | many providers kept up to date by others |
| Tool schemas from code | `@tool` decorator (lesson 04) | richer types, nested models, validation |
| The loop | `run_agent` | tested edge cases: parallel calls, streaming, cancellation |
| Explicit control flow | `guards` and `if` statements | graphs with branches, cycles and sub-graphs (lesson 02) |
| Persistence / checkpoints | none yet | pause, resume, time-travel debugging |
| Memory helpers | none yet | trimming, summarizing, stores (lessons 03–04) |
| Human-in-the-loop | none yet | interrupt-and-resume primitives (lesson 08) |
| Streaming | none yet | token and event streams (lesson 09) |
| Tracing integrations | `tracer.py` | hooks for observability products |
| Prebuilt agents | none | a working tool-calling agent in a few lines |

## The landscape, briefly

Rather than a ranking — which would be out of date quickly — here are the *kinds* of
frameworks you will meet:

- **General LLM application toolkits** (LangChain is the best known). Broad component
  libraries: model wrappers, prompt templates, retrievers, tool abstractions, plus
  higher-level agent constructors.
- **Graph / state-machine orchestrators** (LangGraph is the best known, from the same
  ecosystem as LangChain). You define nodes, edges and a typed state; the runtime
  executes the graph, persists state between steps and supports interrupts. Good fit
  for agents that need explicit control flow and pauses.
- **Multi-agent / role-based frameworks.** Abstractions for teams of agents with roles
  and conversations between them (Level 3 covers the underlying patterns).
- **Provider SDK agent kits.** Model vendors increasingly ship their own lightweight
  agent loops and tool runners. Convenient if you are committed to that vendor; less
  so if you want to stay portable.
- **Workflow engines with LLM steps.** Durable-execution and workflow tools (general
  purpose, not LLM-specific) that treat model calls as activities. Strong on
  reliability and retries; you write more of the agent logic yourself.

## A hedged taste of LangGraph

The snippet below shows the *shape* of a prebuilt tool-calling agent in LangGraph. It is
**not executed in this course** (it needs the library and an API key), and names have
changed across versions — treat it as orientation, and copy exact imports from the
current docs.

```python
# Shape only — check current LangGraph docs for exact imports and signatures.
from langgraph.prebuilt import create_react_agent   # name may differ by version

def get_weather(city: str) -> str:
    """Return a short weather summary for a city."""
    return "heavy showers"

agent = create_react_agent(model=my_chat_model, tools=[get_weather])
result = agent.invoke({"messages": [("user", "Do I need an umbrella in Pune?")]})
```

Everything in it maps to Level 1: the function's signature and docstring become the tool
schema; `create_react_agent` builds the observe-think-act loop; `invoke` runs it until
the model answers; the returned state contains the message list.

## Worked example: a decision checklist

A team wants a support agent that looks up orders, drafts replies and waits for a human
to approve refunds. Walk the checklist:

1. **Do we need pause/resume across processes?** Yes — approvals may take hours. →
   persistence and interrupts are valuable. *Points toward a graph framework or a
   workflow engine.*
2. **How many providers?** One now, maybe two later. → a thin adapter would do.
   *Neutral.*
3. **How custom is the control flow?** Moderate: lookup → draft → (refund? approval) →
   send. *A graph expresses this clearly.*
4. **Who maintains it?** A small team, new to agents. → fewer moving parts matter.
   *Points toward the simplest option that covers #1.*
5. **Lock-in tolerance?** Keep business logic (tools, policies, prompts) in plain
   modules the framework imports, not the other way round. *Makes switching later
   feasible whichever option is chosen.*

Outcome: a graph orchestrator with persistence is justified by requirement 1 alone —
but tools and policies stay framework-free. For a stateless "answer a question with two
lookups" agent, the answer would be the sixty-line loop.

## How It Actually Works

Every framework, however elaborate, compiles down to the same runtime behaviour: call a
model with messages and tool schemas, parse tool calls, execute functions, append
results, decide whether to continue. What differs is **where the control flow is
declared** and **where state is stored**:

- In a simple loop, control flow is Python `if`/`for`, and state is a local list —
  gone if the process dies.
- In a graph framework, control flow is data (a graph of nodes and edges the runtime
  interprets), and state is a structured object the runtime can serialize after each
  node — which is precisely what makes checkpointing, resuming and replaying possible.

That's the real trade: frameworks move control flow from code into a data structure the
runtime understands. You gain persistence and introspection; you pay with indirection,
a learning curve, and exposure to the framework's release cadence.

## Common mistakes

- **Adopting a framework before understanding the loop**, then being unable to debug it.
  (You've avoided this one.)
- **Letting framework types leak into business logic** — tools that only work when
  called by one library.
- **Pinning nothing.** Agent libraries change often; pin versions and upgrade
  deliberately with your evaluation suite (Level 3) as the safety net.
- **Using a multi-agent framework for a single-agent problem** because it's fashionable.
- **Assuming "prebuilt agent" means "production-ready"**: limits, permissions and
  approvals are still your job.

## Exercise

1. List the capabilities from the first table that your planned agent actually needs.
   If the list is only "the loop" and "tool schemas", justify (or reject) using a
   framework in three sentences.
2. Pick one framework and read its quickstart. For each concept it introduces, write the
   Level 1 equivalent (or "none — new capability").
3. Rewrite `umbrella_agent.py` so the tools live in `umbrella_tools.py` with no imports
   from `mini_agent`. That separation is what keeps you portable.
