---
description: "Level 1 of AI agents: what an agent is, the observe-think-act loop, tool calling, tool design, a from-scratch agent, errors, limits and tracing."
---

# Level 1 · Entry <span class="level-badge">Foundations</span>

Goal: understand an agent well enough to build one with nothing but Python's standard
library — and to explain every line. By the end of this level you will have written a
working agent loop, given it tools, made it survive bad tool calls, stopped it from
running forever, and recorded a trace you can read after the fact.

We deliberately start **without a framework**. Frameworks are useful (Level 2 covers
them), but they hide the loop at the centre of every agent. If you have written the
loop yourself, framework documentation stops being magic: you can see which part of
your own forty lines each abstraction replaces.

Three ideas run through the whole level:

1. **An agent is a loop, not a model.** The model proposes the next action; *your code*
   decides whether to execute it, runs it, and feeds the result back. The control
   flow belongs to you.
2. **Tools are an interface you design.** The model can only use a tool as well as its
   name, description and parameters let it. A tool definition is a small piece of
   documentation written for a very literal reader.
3. **Every loop needs a way out.** Step limits, error budgets and a clear "done"
   signal are not optional extras; they are what separates an agent from a runaway
   process.

## Modules

1. [What an Agent Is (and Isn't)](01-what-is-an-agent.md) — chatbots, workflows and agents compared, and the autonomy spectrum
2. [The Agent Loop](02-the-agent-loop.md) — observe, think, act, and the message list that carries state
3. [Tool Calling Fundamentals](03-tool-calling-fundamentals.md) — what a tool call really is on the wire, and who executes it
4. [Writing Good Tool Definitions](04-tool-definitions.md) — names, descriptions, parameter schemas and return values
5. [A Minimal Agent in Plain Python](05-minimal-agent-from-scratch.md) — a complete, runnable loop with a mock model
6. [Handling Tool Errors](06-handling-tool-errors.md) — turning exceptions into information the model can act on
7. [Stopping Conditions & Step Limits](07-stopping-conditions.md) — budgets, loops-within-loops and graceful give-up
8. [Structured Outputs](08-structured-outputs.md) — making the final answer machine-checkable
9. [Logging & Tracing an Agent](09-logging-tracing.md) — spans, events and a trace you can replay
10. [Project — File-Organizer Agent](10-project-file-organizer-agent.md) — a small agent that plans moves in a sandbox folder, with a dry run

## What you need before starting

- **Python 3.9 or newer.** Every code example in this level uses only the standard
  library.
- **No API key.** The examples use a *mock model*: a plain Python function that returns
  tool calls the way a real model would. This keeps the code runnable offline and makes
  the output deterministic, so the printed output you see on each page is exactly what
  the code prints. Lesson 05 shows the one function you replace to use a real model.
- Comfort with Python functions, dictionaries and JSON. If you need a refresher, see
  [Python Mastery Path](https://sigilipelli.github.io/python-mastery-path/).

## How the code is organized

Code blocks with a file name in their title (for example `mini_agent.py`) are meant to
be saved into one working folder as you go. Later lessons import earlier files, just as
a real project grows. Blocks labelled `text` directly under a script show the output
produced by running it.

!!! note "About the mock model"
    A mock model is not intelligent — it follows rules we wrote. That is the point: it
    lets you study the *loop* in isolation. A real language model makes the same kind of
    decisions (which tool, with what arguments, or answer now) but with judgement and
    with mistakes. Several lessons deliberately make the mock misbehave so you can see
    how your loop copes.
