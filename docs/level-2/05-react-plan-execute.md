---
description: "Compare the ReAct pattern with plan-and-execute: how each works, model-call counts, replanning on failure, and when each planning style wins."
---

# 05 · ReAct and Plan-and-Execute

Two planning styles dominate agent design. **ReAct** decides one step at a time,
reasoning in between. **Plan-and-execute** writes the whole plan first, then carries it
out. Neither is "better"; they fail differently, cost differently, and suit different
tasks.

## ReAct: reason, act, observe, repeat

ReAct (from "Reasoning and Acting", a 2022 research paper) interleaves a short piece of
reasoning with each action:

```text
Thought: I need yesterday's error counts for each service. First, list services.
Action: list_services()
Observation: ["billing", "search", "auth"]
Thought: Now get error counts for each.
Action: error_count(service="billing")
Observation: 12
...
Thought: search has the most errors (41). Find its owner.
Action: owner(service="search")
Observation: "team-discovery"
Answer: search had the most errors (41); it is owned by team-discovery.
```

The Level 1 loop *is* a ReAct loop once the model writes a sentence of reasoning in the
`content` field alongside its tool call. Original ReAct parsed "Thought/Action" from
plain text; modern implementations use native tool calling for the action and keep the
thought as ordinary text (or rely on the model's built-in reasoning).

**Strengths:** adapts after every observation; simple; robust to surprises.
**Weaknesses:** one model call per step (slow, costly for long tasks); can lose the
thread and wander; no overall view of the task until it's done.

## Plan-and-execute

A **planner** call produces a list of steps up front. An **executor** runs them —
sometimes with plain code, sometimes with a smaller model per step. A final call
**synthesizes** the answer. If a step fails or reveals something unexpected, a
**replanner** revises the remaining steps.

**Strengths:** fewer calls to the expensive model; the plan is inspectable (and
approvable) *before* anything runs; independent steps can run in parallel.
**Weaknesses:** the plan is made with the least information; brittle when early results
change what later steps should be; replanning logic adds complexity.

## Worked example: the same task both ways

Task: *"Which service had the most errors yesterday, and who owns it?"*

```python title="planning.py"
import json
from tools import tool, registry
from mini_agent import run_agent, tool_results, execute

ERRORS = {"billing": 12, "search": 41, "auth": 3}
OWNERS = {"billing": "team-payments", "search": "team-discovery", "auth": "team-identity"}

@tool
def list_services():
    """List production services."""
    return sorted(ERRORS)

@tool
def error_count(service: str):
    """Number of errors a service logged yesterday.

    Args:
        service: Service name from list_services
    """
    return ERRORS[service]

@tool
def owner(service: str):
    """Team that owns a service.

    Args:
        service: Service name
    """
    return OWNERS[service]

TOOLS = registry(list_services, error_count, owner)
model_calls = {"react": 0, "plan": 0}

# ---------- ReAct: one model decision per step, with a visible thought ----------
def react_model(messages, schemas):
    model_calls["react"] += 1
    r = tool_results(messages)
    def act(thought, name, **args):
        return {"role": "assistant", "content": thought, "tool_calls": [
            {"id": f"c{len(r)}", "name": name, "arguments": json.dumps(args)}]}
    if not r:
        return act("First I need the list of services.", "list_services")
    services = r[0]
    if len(r) - 1 < len(services):
        s = services[len(r) - 1]
        return act(f"Checking errors for {s}.", "error_count", service=s)
    counts = dict(zip(services, r[1:1 + len(services)]))
    worst = max(counts, key=counts.get)
    if len(r) == 1 + len(services):
        return act(f"{worst} has the most errors; finding its owner.", "owner", service=worst)
    return {"role": "assistant", "tool_calls": [],
            "content": f"{worst} had the most errors ({counts[worst]}); owner: {r[-1]}."}

print("ReAct:")
res = run_agent(react_model, TOOLS, "Which service had the most errors yesterday, "
                "and who owns it?", on_event=lambda e, d: None)
for m in res["messages"]:
    if m["role"] == "assistant" and m["tool_calls"]:
        print("  thought:", m["content"])
print("  answer :", res["answer"])
print("  model calls:", model_calls["react"])

# ---------- Plan-and-execute: plan once, execute with code, synthesize once ----------
def planner(task):
    model_calls["plan"] += 1
    # A real planner is a model call that returns JSON like this.
    return [{"id": "s1", "tool": "list_services", "args": {}},
            {"id": "s2", "tool": "error_count", "for_each": "s1", "arg": "service"},
            {"id": "s3", "tool": "owner", "args_from": "argmax:s2", "arg": "service"}]

def executor(plan):
    out = {}
    for step in plan:
        if "for_each" in step:                     # fan out over an earlier result
            items = out[step["for_each"]]
            out[step["id"]] = {i: run_tool(step["tool"], {step["arg"]: i}) for i in items}
        elif "args_from" in step:                  # argument derived from earlier result
            src = out[step["args_from"].split(":")[1]]
            out[step["id"]] = run_tool(step["tool"], {step["arg"]: max(src, key=src.get)})
        else:
            out[step["id"]] = run_tool(step["tool"], step["args"])
        print(f"  {step['id']} {step['tool']} -> {out[step['id']]}")
    return out

def run_tool(name, args):
    result = execute(TOOLS, {"name": name, "arguments": json.dumps(args)})
    if "error" in result:
        raise RuntimeError(result["error"])       # a replanner would catch this
    return result["result"]

def synthesize(task, results):
    model_calls["plan"] += 1
    worst = max(results["s2"], key=results["s2"].get)
    return f"{worst} had the most errors ({results['s2'][worst]}); owner: {results['s3']}."

print("\nPlan-and-execute:")
plan = planner("Which service had the most errors yesterday, and who owns it?")
print("  plan:", [s["id"] + ":" + s["tool"] for s in plan])
answer = synthesize(None, executor(plan))
print("  answer :", answer)
print("  model calls:", model_calls["plan"])
```

```text
ReAct:
  thought: First I need the list of services.
  thought: Checking errors for auth.
  thought: Checking errors for billing.
  thought: Checking errors for search.
  thought: search has the most errors; finding its owner.
  answer : search had the most errors (41); owner: team-discovery.
  model calls: 6

Plan-and-execute:
  plan: ['s1:list_services', 's2:error_count', 's3:owner']
  s1 list_services -> ['auth', 'billing', 'search']
  s2 error_count -> {'auth': 3, 'billing': 12, 'search': 41}
  s3 owner -> team-discovery
  answer : search had the most errors (41); owner: team-discovery.
  model calls: 2
```

Both reach the same answer. ReAct made **one model call per step** (six here);
plan-and-execute made **two** (plan + synthesize), because the executor was plain code.
With three services that difference is small; with fifty, it is the difference between
fifty-odd sequential model calls and two — and step `s2` could run all fifty lookups
in parallel.

Notice what plan-and-execute needed to make that possible: a small **plan language**
(`for_each`, `args_from: argmax:...`). Real planners often emit steps in natural language
and let a model execute each; a structured plan language trades flexibility for speed
and checkability.

## Replanning

Plans break. When a step fails, the executor should hand the replanner **the original
task, the plan, the results so far, and the error**, and ask for a revised plan for the
remaining work. Limit replans (one or two) exactly like repair loops in L1-08, and
surface the failure if the budget runs out.

## Choosing

| Situation | Prefer |
|---|---|
| Next step depends heavily on what you just found (debugging, exploration) | ReAct |
| Steps are knowable up front; many are independent | Plan-and-execute |
| A human should approve *what will happen* before anything runs | Plan-and-execute |
| Long tasks where drift is the main failure | Plan-and-execute, or ReAct with a written plan kept in state |
| Latency-sensitive, small tasks | ReAct with a tight step cap |

A common hybrid: a plan is written and kept in state as a checklist, and a ReAct loop
executes it, ticking items off and allowed to amend the plan when observations demand.

## How It Actually Works

Why does writing a thought before acting help? Tokens the model generates become
context for the tokens that follow. A sentence like "search has the most errors; finding
its owner" puts the conclusion from the observations *right next to* the point where the
action is generated, so the action conditions on a summary rather than on scattered raw
results. That's the same mechanism as chain-of-thought prompting. Models with built-in
reasoning modes do this internally; explicit thoughts are still useful because they
appear in your traces.

Plan-and-execute works by moving decisions **earlier**, when they can be made once, in
one context, and checked. The cost is that the planner decides without observations —
it's forecasting. That's why plans are best for tasks whose structure is predictable
even when the data isn't.

## Common mistakes

- **ReAct without a step cap**, wandering through tangents.
- **Plans with no failure path** — the executor crashes on step 3 of 7 and all work is
  lost.
- **Over-detailed plans** that commit to specifics the planner can't know yet.
- **Plans nobody validates.** A plan is structured output; validate step tools and
  arguments before executing (L1-08).
- **Discarding thoughts from traces.** They're the best debugging signal you have.

## Exercise

1. Add a fourth service whose `error_count` raises an exception. Make the executor catch
   it and call a `replanner` that drops that service and records it as "unknown" in the
   answer.
2. Run the executor's `for_each` step in parallel with `concurrent.futures.ThreadPoolExecutor`
   and confirm the results are identical.
3. Write the planner prompt you would give a real model for this task, including the JSON
   format of the plan language, and one example plan.
