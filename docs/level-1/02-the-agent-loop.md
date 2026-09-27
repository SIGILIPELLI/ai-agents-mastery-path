---
description: "The observe-think-act agent loop step by step, the message list that carries its state, and a runnable trace of one loop iteration."
---

# 02 · The Agent Loop

Every agent, from a thirty-line script to a commercial coding assistant, runs some
version of the same loop. Learn to see it and you can read any agent codebase.

## The three beats

```text
┌──────────────────────────────────────────────────────┐
│                                                      │
│   OBSERVE  →  what is in the context right now?      │
│      ↓                                               │
│   THINK    →  model proposes: call a tool, or answer │
│      ↓                                               │
│   ACT      →  your code runs the tool (or stops)     │
│      ↓                                               │
│   result appended to context ───────────────────────┘
```

- **Observe.** Assemble what the model will see: instructions, the user's goal, and
  every action and result so far.
- **Think.** Call the model. It returns either a final answer or one or more tool
  calls.
- **Act.** If there are tool calls, your code validates and executes them, then records
  the results as new messages. Loop.

The loop ends when the model answers without requesting a tool, or when your code
decides to stop it (step limit, budget, error, or a human says no).

## State lives in the message list

Beginners often look for the agent's "memory" somewhere clever. In the basic loop the
state is simply the **list of messages**. Each iteration appends to it:

```text
[system]    You are an ops assistant. Use tools; be concise.
[user]      Is the nightly backup healthy?
[assistant] → tool_call: get_job_status(job="backup")
[tool]      {"status": "failed", "last_run": "02:00", "error": "disk full"}
[assistant] → tool_call: get_disk_usage(host="backup-01")
[tool]      {"used_pct": 98}
[assistant] The backup failed at 02:00 because backup-01's disk is 98% full...
```

The model is stateless between calls. On every call you send the whole list (or a
trimmed version of it — Level 2 lesson 03). The model "remembers" that it already
checked the job status only because that exchange is in the list.

## A single iteration, runnable

Here is the loop body with the model replaced by a function that inspects the messages
and decides, by simple rules, what to do. A real model would make this decision itself;
the rules just make the example deterministic.

```python title="loop_once.py"
import json

def mock_model(messages):
    """Stand-in for an LLM: decide the next action from the conversation so far."""
    tool_results = [m for m in messages if m["role"] == "tool"]
    if not tool_results:
        return {"role": "assistant", "content": "",
                "tool_calls": [{"id": "c1", "name": "get_job_status",
                                "arguments": json.dumps({"job": "backup"})}]}
    status = json.loads(tool_results[-1]["content"])
    return {"role": "assistant", "tool_calls": [],
            "content": f"The backup job is {status['status']} ({status['error']})."}

def get_job_status(job):
    # A fake tool: in real life this would query a scheduler API.
    return {"job": job, "status": "failed", "error": "disk full"}

TOOLS = {"get_job_status": get_job_status}

messages = [{"role": "system", "content": "You are an ops assistant."},
            {"role": "user", "content": "Is the nightly backup healthy?"}]

for step in (1, 2):
    reply = mock_model(messages)                # THINK
    messages.append(reply)
    if not reply["tool_calls"]:
        print(f"step {step}: final -> {reply['content']}")
        break
    for call in reply["tool_calls"]:            # ACT
        args = json.loads(call["arguments"])
        result = TOOLS[call["name"]](**args)
        print(f"step {step}: {call['name']}({args}) -> {result}")
        messages.append({"role": "tool", "tool_call_id": call["id"],
                         "content": json.dumps(result)})   # OBSERVE (next turn)

print(f"messages in context: {len(messages)}")
```

```text
step 1: get_job_status({'job': 'backup'}) -> {'job': 'backup', 'status': 'failed', 'error': 'disk full'}
step 2: final -> The backup job is failed (disk full).
messages in context: 5
```

Four messages at the end of step 1 became five: system, user, assistant tool call, tool
result, final answer. That list *is* the agent's working memory.

## Worked example: tracing a longer run by hand

Consider a goal: *"Email me the three largest files in /data."* A reasonable agent run:

| Step | Model output | Your code does | Appended |
|---|---|---|---|
| 1 | `list_dir("/data")` | runs it | 40 entries with sizes |
| 2 | *(sorts mentally)* `draft_email(to=me, body=...)` | runs it (draft only) | draft id |
| 3 | `send_email(draft_id=...)` | **pauses for confirmation** | approval or refusal |
| 4 | final text | stops | — |

Notice two engineering decisions that are not the model's to make: step 2 sorts
"mentally" — a real design would give the model a tool that returns files already
sorted, because language models are unreliable at comparing forty numbers; and step 3
is gated, because sending email is externally visible. Both decisions live in the loop
and tool design, not in the prompt.

## How It Actually Works

Why does appending a tool result cause the model to "use" it? Because a chat model is
trained on transcripts in which a tool result is followed by an assistant turn that
refers to it. When your code sends the list, the provider's API renders it into a single
token sequence with special markers for each role — something like
`<system>…<user>…<assistant><tool_call>…<tool_result>…<assistant>` — and asks the model
to continue from the final assistant marker. The most probable continuation after a
tool result is one that incorporates it.

Two consequences follow:

1. **Cost grows with every step.** Each call re-sends the full history, so a ten-step
   run pays for the first messages ten times (prompt caching, Level 3 lesson 07,
   softens this).
2. **Position and formatting matter.** A huge, noisy tool result pushes the original
   goal far back in the context and can dilute it. Good tools return compact results.

## Common mistakes

- **Forgetting to append the assistant's tool-call message** before the tool result.
  Most APIs reject a tool result that does not follow a matching tool call, and models
  get confused when it is missing.
- **Mutating earlier messages.** Rewriting history mid-run makes traces impossible to
  debug. Append; trim only deliberately.
- **Putting control logic in the prompt.** "Never call more than five tools" belongs in
  code as a counter, not only as an instruction.
- **Returning huge tool outputs.** Truncate or summarize in the tool, and say that you
  did.

## Exercise

1. Run `loop_once.py`. Then change `get_job_status` to return `"status": "ok"` and
   `"error": None`. What does the final line print, and why is it awkward? Fix the mock
   model so it produces a sensible sentence for the healthy case.
2. Add a second tool, `get_disk_usage(host)`, and extend the mock so that after a
   "disk full" error it calls that tool before answering. Raise the loop range to 3.
3. Print `messages` as indented JSON at the end and label each entry with the beat
   (observe/think/act) that produced it.
