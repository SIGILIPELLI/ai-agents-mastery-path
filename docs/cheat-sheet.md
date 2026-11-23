---
title: "AI Agents Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free AI Agents cheat sheet: 20 quick-reference tables from every level of the AI Agents Mastery Path, each linked to its full lesson."
---

# AI Agents Cheat Sheet

Quick-reference tables from **20 lessons** of the free AI Agents Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [01 · What an Agent Is (and Isn't)](level-1/01-what-is-an-agent.md)

*Three shapes of LLM application*

| | Chatbot | Workflow (pipeline) | Agent |
|---|---|---|---|
| Who decides the steps? | Nobody — one model call per turn | The developer, in code | The model, at run time |
| Number of model calls per request | 1 | Fixed (e.g. 3) | Variable (1 to N) |
| Uses tools? | Maybe one, fixed | Yes, in fixed places | Yes, chosen dynamically |
| Predictability | High | High | Lower |
| Handles unexpected inputs | Only in text | Poorly — falls off the path | Can adapt |
| Cost per request | Low, fixed | Fixed | Variable, sometimes large |

### [02 · The Agent Loop](level-1/02-the-agent-loop.md)

*Worked example: tracing a longer run by hand*

| Step | Model output | Your code does | Appended |
|---|---|---|---|
| 1 | `list_dir("/data")` | runs it | 40 entries with sizes |
| 2 | *(sorts mentally)* `draft_email(to=me, body=...)` | runs it (draft only) | draft id |
| 3 | `send_email(draft_id=...)` | **pauses for confirmation** | approval or refusal |
| 4 | final text | stops | — |

### [05 · A Minimal Agent in Plain Python](level-1/05-minimal-agent-from-scratch.md)

*Design decisions up front*

| Contract | Shape |
|---|---|
| **Model** | a callable `model(messages, tool_schemas) -> assistant_message` |
| **Assistant message** | `{"role": "assistant", "content": str, "tool_calls": [ {id, name, arguments} ]}` |
| **Tools** | a dict `name -> function`, each function carrying a `.schema` (from `tools.py`, lesson 04) |

### [06 · Handling Tool Errors](level-1/06-handling-tool-errors.md)

*Four kinds of failure*

| Kind | Example | Who should handle it | How |
|---|---|---|---|
| **Model-fixable** | bad date format, unknown ID, missing argument | the model | return a precise error message as the tool result |
| **Transient** | timeout, HTTP 503, rate limit | your code | retry with backoff *inside* the tool wrapper; the model never sees it unless retries run out |
| **Permanent but survivable** | record deleted, permission denied on one file | the model | return the error and let it choose another path or explain |
| **Fatal** | credentials revoked, disk full, sandbox gone | your code | stop the run with a clear reason; don't ask the model to work around it |

### [09 · Logging &amp; Tracing an Agent](level-1/09-logging-tracing.md)

*What to record*

| Event | Fields |
|---|---|
| run start | run id, task, model name/version, prompt version, tool list, limits |
| model call | step, input size (messages / tokens if available), output, latency |
| tool call | step, tool name, arguments, result (or error), latency |
| stop | reason (answer / guard name / error), totals |

### [10 · Project — File-Organizer Agent](level-1/10-project-file-organizer-agent.md)

*Tools*

| Tool | Effect | Notes |
|---|---|---|
| `list_files()` | read-only | names and sizes in the sandbox root only |
| `move_file(name, folder)` | **write** | creates `folder` if needed; refuses overwrites and escapes; in dry-run mode records the plan instead |
| `finish(moved, skipped, notes)` | ends the run | the finish-tool pattern from lesson 08 |

## Level 2 · Intermediate

### [01 · Frameworks: When and Which](level-2/01-frameworks-overview.md)

*What a framework actually gives you*

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

### [04 · Long-Term &amp; Vector Memory](level-2/04-long-term-memory.md)

*Kinds of long-term memory*

| Kind | Example | Typical storage | How it's retrieved |
|---|---|---|---|
| **Profile / preferences** | "Prefers metric units; reports go to #ops-weekly" | key-value or small table | loaded at start of each run |
| **Episodic** | "On 2026-09-14 the export failed due to lock waits" | log of summaries with dates | search by similarity or time |
| **Semantic / knowledge** | "Service X owns the billing tables" | documents + vector index | similarity search |
| **Procedural** | "For export failures, check lock waits first" | curated notes, prompts | loaded by task type, or searched |

### [05 · ReAct and Plan-and-Execute](level-2/05-react-plan-execute.md)

*Choosing*

| Situation | Prefer |
|---|---|
| Next step depends heavily on what you just found (debugging, exploration) | ReAct |
| Steps are knowable up front; many are independent | Plan-and-execute |
| A human should approve *what will happen* before anything runs | Plan-and-execute |
| Long tasks where drift is the main failure | Plan-and-execute, or ReAct with a written plan kept in state |
| Latency-sensitive, small tasks | ReAct with a tight step cap |

### [07 · Retrieval as a Tool](level-2/07-retrieval-as-a-tool.md)

*Always-retrieve vs. agentic retrieval*

| | Always-retrieve (pipeline) | Retrieval as a tool (agent) |
|---|---|---|
| Searches per question | exactly 1 | 0 to N |
| Query | the user's words | written by the model; can be reformulated |
| Good at | direct factual questions | multi-part questions, follow-ups, "search again with different terms" |
| Cost / latency | predictable | variable |
| Failure mode | irrelevant chunks for off-topic questions | not searching when it should; searching forever |

### [08 · Human-in-the-Loop Approvals](level-2/08-human-in-the-loop.md)

*What to gate*

| Class | Examples | Default |
|---|---|---|
| **Read-only** | search, get record, list files | run freely |
| **Reversible write, internal** | create draft, add internal note, write to scratch folder | run freely or log for review |
| **Externally visible** | send email/message, post comment, publish | **approval** |
| **Financial / irreversible / privileged** | refund, payment, delete, change permissions, deploy | **approval**, often with extra checks or two people |

### [10 · Project — Research Agent with Approvals](level-2/10-project-research-agent.md)

*How It Actually Works*

| Component | Guarantee |
|---|---|
| Graph with a revision counter | the draft/check cycle ends (at most 2 revisions) |
| Search results kept in state | the checker knows exactly what evidence existed |
| Grounded check | every sentence is cited, and every citation is to retrieved evidence |
| Interrupt before `save` | no side effect without a recorded human decision |
| Checkpoint file | approval can arrive later, in another process |

## Level 3 · Advanced

### [03 · Model Context Protocol (MCP) Concepts](level-3/03-mcp-concepts.md)

*What servers offer*

| Primitive | Controlled by | What it is | Example |
|---|---|---|---|
| **Tools** | the model (with host approval) | functions the model can call | `create_issue`, `run_query` |
| **Resources** | the application | readable context identified by URI | a file, a DB schema, a document |
| **Prompts** | the user | reusable prompt templates | "summarize this PR" |

### [04 · Sandboxing Code-Execution Agents](level-3/04-sandboxing-code-execution.md)

*What can go wrong*

| Risk | Example |
|---|---|
| Runaway resources | infinite loop, 50 GB allocation, fork bomb, filling the disk |
| Secret access | reading environment variables, `~/.ssh`, cloud credentials files |
| Data damage | deleting or overwriting files outside a work folder |
| Network abuse | exfiltrating data, calling internal services, scanning the network |
| Escape | exploiting the interpreter or kernel to leave the sandbox |

### [06 · Trajectory Evaluation](level-3/06-trajectory-evaluation.md)

*What to check in a trajectory*

| Check | Question | Example rule |
|---|---|---|
| **Required steps** | Did it do the things it must? | `get_order` called before any refund |
| **Forbidden steps** | Did it avoid what it must never do? | never `delete_customer`; never `send_email` in a read-only task |
| **Ordering** | Did it respect dependencies? | `verify_identity` before `change_email` |
| **Arguments** | Were critical arguments right? | refund `amount` ≤ order amount; `order_id` matches the task |
| **Redundancy** | Did it repeat itself? | same call with same arguments twice |
| **Efficiency** | How many steps vs. a good reference? | ≤ 1.5 × reference step count |
| **Grounding** | Were claims in the answer backed by tool results? | the amount in the answer appears in a tool result |

## Level 4 · Master

### [01 · Production Agent Architecture](level-4/01-production-architecture.md)

*Where each control lives*

| Control | Location | Lesson |
|---|---|---|
| Step, time, token limits | worker loop | L1-07 |
| Spend limits per tenant | model gateway | L3-07 |
| Authorization, scopes | tool gateway | L4-03 |
| Policy rules, approvals | tool gateway + run store | L4-04, L2-08 |
| Idempotency, retries, breakers | tool gateway | L3-08 |
| Tracing, metrics | everywhere, via event bus | L4-02 |
| Kill switch, versions | config service read by workers | L4-09 |

### [02 · Observability in Production](level-4/02-observability.md)

*What to measure*

| Question | Metrics |
|---|---|
| Is it working? | run success rate; stop-reason mix (answered / step limit / guard / error); approval rejection rate; user feedback rate |
| Is it fast enough? | run latency p50/p95/p99; time to first event; model latency vs tool latency |
| What does it cost? | tokens and cost per run (mean and p95); model calls per run; cost per tenant per day |
| Is it behaving? | steps per run; tool error rate per tool; repeated-call rate; hard trajectory violations (L3-06) |
| Is it safe? | blocked actions (policy, egress, taint); approval requests; injection-suspect events |

### [04 · Guardrails &amp; Policy as Code](level-4/04-guardrails-policy.md)

*Three checkpoints*

| Checkpoint | Sees | Typical checks | Possible decisions |
|---|---|---|---|
| **Input** | the user's request (and uploaded content) | out-of-scope topics, abusive content, obvious injection attempts, personal data that shouldn't be processed | allow, refuse, route to a human |
| **Action** | a proposed tool call with arguments, plus run context | spending limits, recipients, business hours, data classification, taint (L3-09) | allow, deny, require approval, modify (e.g. cap an amount) |
| **Output** | the final answer or outgoing message | leaked secrets or personal data, unsupported claims, promises the business can't make, tone | allow, redact, block, regenerate |

### [08 · Ethics &amp; Accountability](level-4/08-ethics-accountability.md)

*Principle: an agent is never the accountable party*

| Role | Responsibility |
|---|---|
| **Business owner** | decides the agent should exist, owns its outcomes and its budget |
| **Engineering owner** | builds and operates it; on call for incidents |
| **Policy owners** | own specific rules (refund limits, data handling) and approve changes |
| **Reviewers/approvers** | make the decisions the agent prepares, with authority to override |
| **Affected people's channel** | how a customer or employee can question or appeal an outcome |

### [10 · Capstone — A Production-Ready Agent](level-4/10-capstone.md)

*Requirements*

| # | Requirement | Course source |
|---|---|---|
| 1 | Framework-free loop with step limit and loop detection | L1-05, L1-07 |
| 2 | Tools declared once with schemas; actionable errors | L1-04, L1-06 |
| 3 | Every call checked against the user's permissions (tenant) | L4-03 |
| 4 | Every call evaluated by declarative policy: allow / approve / deny | L4-04 |
| 5 | Approval for money above a threshold; denials returned to the model | L2-08 |
| 6 | Idempotency keys on writes | L3-08 |
| 7 | Kill switch and writes-off switch, read on every call | L4-09 |
| 8 | JSONL trace per run including the version bundle ID; secrets redacted | L1-09, L4-09 |
| 9 | Offline eval gate on outcome and trajectory before release | L3-05, L3-06, L4-07 |

---

Go deeper with the full [AI Agents Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
