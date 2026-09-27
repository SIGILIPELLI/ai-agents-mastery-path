---
description: "Decide when an agent is the wrong tool: signals, cheaper alternatives from plain code to single LLM calls, and a scorecard for reviewing agent proposals."
---

# 06 · When NOT to Use an Agent

A course on agents should say it plainly: **most problems that people propose solving
with an agent are better solved with something simpler.** Agents are the most flexible
option on the menu and the most expensive to build, test, secure and operate. Choosing
something simpler when it suffices is not a lack of ambition; it's engineering.

## The simpler options, in order

1. **Plain code / rules.** Deterministic, cheap, testable. If you can specify it, code it.
2. **Search or a better UI.** Many "assistant" requests are really "I can't find the
   thing". Improving search or a form may solve it without generation at all.
3. **A single LLM call.** Classification, extraction, summarization, rewriting, drafting.
   One call, fixed inputs, validated output.
4. **Retrieval + one call.** Question answering over documents (see the
   [RAG Mastery Path](https://sigilipelli.github.io/rag-mastery-path/)).
5. **A workflow with LLM steps.** Fixed sequence; the model fills in steps; code decides
   the path (L1-01).
6. **An agent** — only when the path genuinely depends on what's discovered along the way.

## Signals that an agent is the wrong choice

- **You can draw the flowchart.** If every path is known in advance, it's a workflow.
- **The task must be exactly right every time** with no human review (payroll
  calculations, regulatory filings, safety interlocks). Agents are probabilistic.
- **Latency budget is tight** (sub-second responses). Multiple model calls won't fit.
- **Volume is huge and margins are thin.** Per-run agent cost × volume may exceed the
  value.
- **There's no way to evaluate success.** If you can't write a checker or a rubric, you
  can't tell whether the agent works (Level 3).
- **The tools don't exist yet.** An agent is only as good as its tools; if the underlying
  APIs are missing or unreliable, fix those first — often that alone solves the problem.
- **Nobody will own it.** Agents need ongoing evaluation, monitoring and updates.

## Worked example: a proposal scorecard

Use a scorecard in design reviews to make the discussion concrete. Each question pushes
toward a simpler option or justifies autonomy.

```python title="agent_scorecard.py"
QUESTIONS = [
    ("path_depends_on_findings", "Does the next step genuinely depend on what earlier steps find?", +2),
    ("flowchart_drawable",       "Can the full flowchart be drawn in advance?", -3),
    ("needs_exactness",          "Must every output be exactly right with no human review?", -3),
    ("tight_latency",            "Is the latency budget under ~2 seconds?", -2),
    ("evals_possible",           "Can success be checked by code or a clear rubric?", +1),
    ("tools_ready",              "Do reliable APIs exist for every action needed?", +1),
    ("human_reviews",            "Does a human already review the outcome?", +1),
    ("owner_exists",             "Is a team committed to operating it?", +1),
]

def recommend(answers):
    score = sum(w for key, _, w in QUESTIONS if answers.get(key))
    if answers.get("flowchart_drawable") or score <= 0:
        return score, "workflow or single LLM call"
    if not (answers.get("evals_possible") and answers.get("tools_ready")):
        return score, "not yet: build tools and evals first"
    return score, "agent (start in assist mode)"

proposals = {
    "Password reset assistant": {"flowchart_drawable": True, "tools_ready": True,
                                 "evals_possible": True, "owner_exists": True},
    "Summarize weekly incident reports": {"flowchart_drawable": True, "evals_possible": True},
    "Investigate failed deploys": {"path_depends_on_findings": True, "evals_possible": True,
                                   "tools_ready": True, "human_reviews": True,
                                   "owner_exists": True},
    "Autonomous payroll corrections": {"path_depends_on_findings": True,
                                       "needs_exactness": True, "tools_ready": True},
    "Research competitor pricing": {"path_depends_on_findings": True, "human_reviews": True,
                                    "owner_exists": True},
}
for name, answers in proposals.items():
    score, rec = recommend(answers)
    print(f"{name:<36} score {score:+d} -> {rec}")
```

```text
Password reset assistant             score +0 -> workflow or single LLM call
Summarize weekly incident reports    score -2 -> workflow or single LLM call
Investigate failed deploys           score +6 -> agent (start in assist mode)
Autonomous payroll corrections       score +0 -> workflow or single LLM call
Research competitor pricing          score +4 -> not yet: build tools and evals first
```

The weights are illustrative — calibrate them for your organisation — but notice the
structure: some answers (a drawable flowchart) are close to disqualifying on their own,
and some prerequisites (evals, tools) turn a "yes" into "not yet".

## A worked comparison

Consider "summarize weekly incident reports". As an agent: the model decides to list
reports, read each, maybe search for related tickets, then write — perhaps 8–15 model
calls, variable output, a loop to test. As a workflow: code fetches the week's reports,
one call summarizes them against a fixed template, code posts the result — one model
call, predictable cost, trivially testable, and in practice at least as good. The agent
version adds nothing the task needs.

Now "investigate failed deploys": logs point to config, config to a secret, the secret to
an expired certificate — each step depends on the last, and the set of possible paths is
large. That's what agents are for.

## How It Actually Works

Every step up the autonomy ladder trades **predictability for flexibility**. The cost of
lost predictability is paid everywhere downstream: testing needs statistics instead of
assertions, security needs gates instead of fixed permissions, operations need behaviour
monitoring instead of just error rates, and cost becomes a distribution instead of a
number. Flexibility is worth that price only when the task actually requires runtime
decisions that you can't enumerate — which is exactly what the scorecard probes.

## Common mistakes

- **Starting from the technology** ("we should have an agent") instead of the problem.
- **Using an agent to paper over missing APIs** or bad data.
- **Comparing an agent with doing nothing** instead of with the simplest alternative.
- **Keeping an agent after evidence shows a workflow performs as well.**
- **Autonomy for autonomy's sake** when an assist-mode tool would capture the value.

## Exercise

1. Score three agent ideas from your own work with the scorecard. For any that come out
   as "workflow", sketch the workflow in five boxes.
2. Take the Level 2 research agent. Which parts could be a fixed workflow without losing
   quality? Rewrite it that way and compare model-call counts.
3. "Research competitor pricing" comes out as "not yet". Which prerequisites are
   missing, and what exactly would the team build to change the answer?
