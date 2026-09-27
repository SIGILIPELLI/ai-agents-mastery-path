---
description: "When a multi-agent design beats a single agent: context isolation, specialization and parallelism versus coordination cost, with a decision test."
---

# 01 · Multi-Agent Systems: Why and When

A multi-agent system splits one job across several agents, each with its own
instructions, tools and context, coordinated by code or by another agent. It's one of
the most hyped patterns in the field and one of the most over-applied. This lesson gives
you a way to decide on evidence.

## What splitting actually buys you

1. **Context isolation.** Each agent sees only what its sub-task needs. A research
   sub-agent can read fifty pages and hand back a one-paragraph summary; the main agent's
   context never holds those fifty pages. This is often the biggest real benefit.
2. **Smaller toolsets.** Tool choice gets harder as the list grows. A billing specialist
   with 4 tools chooses better than a generalist with 30.
3. **Specialized instructions.** Each agent's system prompt can be focused, shorter and
   independently tested.
4. **Parallelism.** Independent sub-tasks (research three competitors) can run at the
   same time.
5. **Permission boundaries.** A sub-agent that reads untrusted web content can be given
   *no* write tools at all (lesson 09 builds on this).

## What it costs

1. **More model calls** — coordination itself takes calls: routing, delegating,
   summarizing results back.
2. **Information loss at boundaries.** Every handoff is a summary; details the receiver
   needed can be dropped.
3. **Harder debugging.** A bad outcome may be caused three agents upstream. Traces must
   link parent and child runs.
4. **Compounding errors.** If each agent succeeds 90% of the time on its part, a chain of
   three succeeds at most about 73% of the time (0.9³), unless later agents can detect
   and fix earlier mistakes.
5. **Conflicting actions** when two agents can write to the same resource.

## Common topologies

```text
Supervisor (hub-and-spoke)       Handoff (baton pass)         Pipeline
      ┌─────────┐                 A ──▶ B ──▶ C               A ─▶ B ─▶ C
      │  super  │                 (control moves;             (fixed order;
      └─┬──┬──┬─┘                  the current agent           really a workflow
        │  │  │                    talks to the user)          with agent steps)
        A  B  C
```

- **Supervisor**: one coordinator delegates to specialists and assembles results.
  Specialists never talk to each other directly.
- **Handoff**: the active agent transfers the whole conversation to another agent that
  is better suited (e.g. triage → billing), which then continues with the user.
- **Pipeline**: a fixed sequence — usually better described as a workflow (L1-01).

Lesson 02 implements the first two.

## Worked example: measuring the toolset argument

Before splitting, measure what a single agent would carry. Tool definitions are sent on
every call, so their size is a recurring cost, and their number affects choice accuracy.

```python title="toolset_size.py"
import json
from tools import tool, registry

def make_tool(name, doc):
    def fn(**kwargs):
        return {}
    fn.__name__ = name
    fn.__doc__ = doc + "\n\nArgs:\n    query: What to look up\n"
    fn.__annotations__ = {"query": str}
    import inspect
    fn.__signature__ = inspect.Signature(
        [inspect.Parameter("query", inspect.Parameter.KEYWORD_ONLY, annotation=str)])
    return tool(fn)

AREAS = {
    "billing": ["get_invoice", "list_payments", "issue_refund", "update_card"],
    "shipping": ["track_parcel", "change_address", "reschedule_delivery", "file_claim"],
    "accounts": ["reset_password", "change_email", "close_account", "export_data"],
    "tech": ["search_kb", "get_device_logs", "run_diagnostic", "open_bug"],
}
agents = {area: registry(*[make_tool(n, f"{n.replace('_', ' ').capitalize()} for the "
                                        f"customer. Part of the {area} toolset.")
                           for n in names])
          for area, names in AREAS.items()}
single = {k: v for tools in agents.values() for k, v in tools.items()}

def schema_chars(tools):
    return len(json.dumps([fn.schema for fn in tools.values()]))

print(f"single agent : {len(single):2d} tools, {schema_chars(single):5d} chars of schema per call")
for area, tools in agents.items():
    print(f"{area:<13}: {len(tools):2d} tools, {schema_chars(tools):5d} chars of schema per call")
router = registry(make_tool("route_to_specialist", "Hand the request to billing, shipping, "
                            "accounts or tech."))
print(f"router       : {len(router):2d} tool,  {schema_chars(router):5d} chars of schema per call")
```

```text
single agent : 16 tools,  3848 chars of schema per call
billing      :  4 tools,   958 chars of schema per call
shipping     :  4 tools,   978 chars of schema per call
accounts     :  4 tools,   968 chars of schema per call
tech         :  4 tools,   944 chars of schema per call
router       :  1 tool,    244 chars of schema per call
```

With sixteen short tools the numbers are modest; real tool definitions are often several
times longer. The more important effect is on **choice**: each specialist chooses among
four clearly related tools instead of sixteen, some with overlapping purposes. The price
is the extra routing call on every request.

## A decision test

Start with one agent. Split only when you can point to one of these in your traces or
evals:

- The agent **picks wrong tools** among a large toolset, and better descriptions haven't
  fixed it.
- The context fills with **sub-task detail** (raw pages, logs) that later steps don't
  need.
- Sub-tasks are **independent and slow**, and latency matters.
- Parts of the task need **different permissions** — especially reading untrusted content
  versus taking actions.

If none apply, a multi-agent design will probably make the system slower, costlier and
harder to debug for no measurable gain.

## How It Actually Works

Each agent is still one model conditioned on one context. "Multi-agent" means you are
running several such contexts and passing *text* between them. The benefits come from
what that does to each context: shorter, more focused, with fewer and more relevant
tools. The model's attention is spent on less irrelevant material, which tends to
improve decisions.

The costs come from the same place. Every boundary is a lossy compression step — the
sub-agent's findings become a summary — and nothing guarantees the summary keeps what
the receiver needs. That's why good multi-agent designs specify **handoff contracts**:
what a sub-agent must return (structured, with evidence), not just "report back".

## Common mistakes

- **Role-play teams** ("CEO agent", "CTO agent") with no measurable reason to exist.
- **Chatty agents** that converse freely with each other, multiplying calls without
  bound. Coordination should have a structure and a budget.
- **Unstructured handoffs** that lose key facts.
- **Shared write access** without locking or ownership rules.
- **Separate traces per agent** with no parent-child link.

## Exercise

1. For a system you know (or the Level 2 research agent), list the tools and the context
   each step needs. Would splitting reduce context or toolset size for any step? By how
   much?
2. Compute the upper bound on end-to-end success for a chain of four agents at 95% each,
   and for a supervisor that can retry a failed specialist once.
3. Write a handoff contract (JSON fields) for a research sub-agent that returns findings
   to a supervisor. Include evidence and confidence.
