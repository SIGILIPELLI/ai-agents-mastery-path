---
description: "Level 2 of AI agents: frameworks, state graphs, short- and long-term memory, ReAct, plan-and-execute, reflection, retrieval tools, approvals and streaming."
---

# Level 2 · Intermediate <span class="level-badge">Patterns</span>

Level 1 gave you one loop. Real agents need more structure: explicit states and
transitions, memory that outlives a context window, planning that doesn't wander,
retrieval over your own documents, humans who can say "no" before something happens,
and a user interface that shows progress instead of a spinner.

This level covers those patterns in two ways at once. For each pattern you get a
**small framework-free implementation** that runs offline (continuing the
`mini_agent.py` project from Level 1), and a **description of how frameworks express the
same idea**, so you can recognise it in LangGraph, LangChain or any other library you
adopt.

!!! note "About framework code in this level"
    Agent frameworks evolve quickly and rename things between versions. Framework
    snippets here are **short, marked as not executed in this course, and describe the
    shape of the API rather than guaranteeing exact names.** Always check the current
    documentation of the version you install. The framework-free code is complete and
    its printed output is real.

## Modules

1. [Frameworks: When and Which](01-frameworks-overview.md) — what frameworks give you, what they cost, and a decision checklist
2. [Agents as State Graphs](02-state-graphs.md) — nodes, edges, shared state and checkpoints, with a 60-line graph runner
3. [Short-Term Memory & Context](03-short-term-memory.md) — trimming and summarizing history without breaking tool-call pairs
4. [Long-Term & Vector Memory](04-long-term-memory.md) — remembering across sessions with explicit memory tools
5. [ReAct and Plan-and-Execute](05-react-plan-execute.md) — two planning styles and when each wins
6. [Reflection & Self-Critique](06-reflection.md) — generator/critic loops that actually converge
7. [Retrieval as a Tool](07-retrieval-as-a-tool.md) — letting the agent decide when and what to search
8. [Human-in-the-Loop Approvals](08-human-in-the-loop.md) — pausing before risky actions, and resuming later
9. [Streaming Agent Progress](09-streaming.md) — events, partial output, and responsive interfaces
10. [Project — Research Agent with Approvals](10-project-research-agent.md) — a graph-based agent that searches, drafts, critiques and asks before saving

## Before you start

- Finish Level 1, or at least lessons 05, 07 and 09: this level imports `tools.py`,
  `mini_agent.py`, `guards.py` and `tracer.py`.
- Python 3.9+, standard library only. Installing a framework is optional and only needed
  if you want to try the framework snippets yourself.
- For the retrieval lesson, the [RAG Mastery Path](https://sigilipelli.github.io/rag-mastery-path/)
  covers embeddings, chunking and vector stores in depth. Here we focus on the agent
  side: exposing retrieval as a tool and using its results well.
