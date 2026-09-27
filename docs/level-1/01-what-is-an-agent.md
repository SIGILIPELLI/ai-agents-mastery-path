---
description: "What an AI agent actually is, how it differs from a chatbot and a fixed workflow, and where on the autonomy spectrum a task belongs."
---

# 01 · What an Agent Is (and Isn't)

"Agent" is one of the most overloaded words in software right now. Product pages call
a chatbot with a search box an agent; research papers use it for systems that run for
hours. Before building one, it pays to have a definition precise enough to make
engineering decisions with.

## A working definition

In this course:

> An **agent** is a program in which a language model **chooses the next action** —
> usually a tool call — **in a loop**, observing the result of each action before
> choosing the next, until it decides the task is done or a limit stops it.

Three parts of that sentence carry the weight:

- **Chooses the next action.** The sequence of steps is not written in advance. The
  model decides, at run time, whether to search, read a file, call an API, or answer.
- **In a loop, observing results.** The output of step *n* is input to the decision at
  step *n + 1*. This feedback is what lets an agent recover from surprises.
- **Until done or stopped.** Termination is part of the design, not an afterthought.

Anything missing one of these is something else — often something better for the job.

## Three shapes of LLM application

| | Chatbot | Workflow (pipeline) | Agent |
|---|---|---|---|
| Who decides the steps? | Nobody — one model call per turn | The developer, in code | The model, at run time |
| Number of model calls per request | 1 | Fixed (e.g. 3) | Variable (1 to N) |
| Uses tools? | Maybe one, fixed | Yes, in fixed places | Yes, chosen dynamically |
| Predictability | High | High | Lower |
| Handles unexpected inputs | Only in text | Poorly — falls off the path | Can adapt |
| Cost per request | Low, fixed | Fixed | Variable, sometimes large |

### A chatbot

A single call: messages in, text out. It may have a system prompt and conversation
history, but it takes no actions in the world. A customer-FAQ bot is a chatbot.

### A workflow

A fixed sequence written by a developer, where some steps happen to call a model:

```text
classify ticket  →  if "billing": fetch invoice  →  draft reply  →  send for review
```

The model fills in blanks (a category, a draft), but the *path* is code. Workflows are
predictable, cheap and easy to test. Many problems that are marketed as "agentic" are
better solved this way — Level 4 lesson 06 is devoted to that argument.

### An agent

The model holds the steering wheel inside boundaries you define:

```text
goal: "Find why last night's export job failed and suggest a fix"
  model → read_logs(job="export", since="yesterday")
  model → (sees a timeout on the database step) → query_metrics(db, window=...)
  model → (sees high lock wait) → search_runbook("lock wait export")
  model → final answer with evidence
```

No developer wrote "if timeout then query metrics". The model inferred it from the
observation. That flexibility is the whole value — and the whole risk.

## The autonomy spectrum

Rather than a yes/no label, think of a dial:

1. **Single call** — model answers from its own knowledge.
2. **Augmented call** — your code retrieves context, then one model call.
3. **Router** — the model picks one of several fixed paths.
4. **Tool-using loop** — the model picks tools repeatedly within a step budget.
5. **Planner + executor** — the model writes a multi-step plan, then carries it out.
6. **Long-running autonomous agent** — works for many minutes or hours, with memory,
   sub-agents and checkpoints.

Move up the dial only when the task needs it. Every level adds cost, latency, new
failure modes and more to test.

## Worked example: classifying three real requests

Suppose an internal IT team gets these requests. Where does each belong?

**"Reset my password."** The steps are always the same: verify identity, trigger reset,
confirm. A **workflow** — possibly with a model only to parse the request. An agent adds
risk (it might choose an unexpected action on an account) with no benefit.

**"Summarize this week's incident reports for the Monday meeting."** Fixed inputs, one
transformation. An **augmented call**: your code gathers the reports; one model call
summarizes.

**"The staging deploy is broken; figure out why."** The next step genuinely depends on
what you find: logs point to config, config points to a secret that expired, and so on.
This is where a **tool-using agent** earns its cost — ideally with read-only tools and a
human approving any fix.

The rule of thumb that falls out: *if you can draw the flowchart in advance, draw it and
write a workflow. If the flowchart depends on what you discover, consider an agent.*

## How It Actually Works

A language model does exactly one thing: given a sequence of tokens, it produces a
continuation. It cannot open a file, call an API or remember yesterday. Everything that
makes a system "agentic" lives **outside** the model, in ordinary code:

- The model is trained (and prompted) to emit a special structured continuation — a
  *tool call* — when it wants something done. That is still just text, formatted in a
  way your code can parse.
- Your code executes the tool, turns the result into text, appends it to the
  conversation, and calls the model again. From the model's point of view, the tool
  result is simply more context.
- The "decision" to stop is the model producing a normal text answer instead of a tool
  call, or your code refusing to continue.

So an agent is a **control loop with a stochastic policy**: deterministic code around a
component that proposes actions probabilistically. This framing explains most agent
engineering. Reliability comes from the deterministic parts — validation, limits,
permissions, retries — because you cannot make the policy itself deterministic.

## Common mistakes

- **Calling any LLM feature an agent.** Precise language drives good design reviews:
  "this is a router with two paths" invites different tests than "this is an agent".
- **Starting at the top of the autonomy dial.** Begin with the simplest shape that could
  work, measure where it fails, and add autonomy only at the failure point.
- **Assuming the model "knows" its tools' effects.** It knows only the descriptions you
  give it. Effects in the world are your responsibility.
- **Treating the agent as the product.** Users care about the outcome. An agent is an
  implementation detail with a cost profile.

## Exercise

Pick three tasks from your own work or studies. For each:

1. Write the steps you would take by hand.
2. Mark every step whose *choice* depends on the result of an earlier step.
3. Place the task on the autonomy dial (1–6) and justify it in two sentences.
4. For any task you placed at 4 or above, list which tools it would need and which of
   those tools could change something in the world (write, send, delete, pay). Those
   are the ones that will need confirmation gates later in the course.
