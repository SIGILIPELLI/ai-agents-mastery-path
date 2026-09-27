---
description: "Build a complete agent loop in plain Python with no framework or API key, using a mock model that emits real tool calls, then swap in a real model."
---

# 05 · A Minimal Agent in Plain Python

Time to write the whole thing. By the end of this page you will have `mini_agent.py`: a
complete agent loop in about sixty lines of standard-library Python. It runs offline,
because the "model" is a mock function — but the loop itself is the same one you would
use with a real model, and later lessons build on it without changes.

## Design decisions up front

Before writing code, decide the contracts. These are the only three things the loop
needs to know about the outside world:

| Contract | Shape |
|---|---|
| **Model** | a callable `model(messages, tool_schemas) -> assistant_message` |
| **Assistant message** | `{"role": "assistant", "content": str, "tool_calls": [ {id, name, arguments} ]}` |
| **Tools** | a dict `name -> function`, each function carrying a `.schema` (from `tools.py`, lesson 04) |

Keeping the model behind a plain callable is the most important decision. The loop never
imports a provider SDK; an *adapter* converts between our neutral message format and
whatever the provider expects. Swap the adapter, keep the agent.

## The loop

```python title="mini_agent.py"
"""mini_agent.py — a framework-free agent loop (AI Agents Mastery Path, Level 1)."""
import json


def call(tool_name, call_id="c1", /, **arguments):
    """Build an assistant message that requests one tool call (handy for mock models)."""
    return {"role": "assistant", "content": "",
            "tool_calls": [{"id": call_id, "name": tool_name,
                            "arguments": json.dumps(arguments)}]}


def answer(text):
    """Build an assistant message with a final answer and no tool calls."""
    return {"role": "assistant", "content": text, "tool_calls": []}


def tool_results(messages):
    """All tool results so far, decoded from JSON, oldest first."""
    return [json.loads(m["content"]) for m in messages if m["role"] == "tool"]


def print_event(event, data):
    """Default event handler: a compact, human-readable log."""
    if event == "tool_call":
        print(f"  step {data['step']}: {data['name']}({data['arguments']})")
    elif event == "tool_result":
        text = data["content"]
        print(f"      -> {text[:100] + '...' if len(text) > 100 else text}")
    elif event == "final":
        print(f"  step {data['step']}: final answer")
    elif event == "stopped":
        print(f"  stopped: {data['reason']}")


def execute(tools, call_):
    """Run one tool call. Never raises: problems come back as {'error': ...}."""
    fn = tools.get(call_["name"])
    if fn is None:
        return {"error": f"unknown tool '{call_['name']}'; available: {sorted(tools)}"}
    try:
        args = json.loads(call_["arguments"] or "{}")
    except json.JSONDecodeError as e:
        return {"error": f"arguments were not valid JSON ({e.msg}); resend the call"}
    try:
        return {"result": fn(**args)}
    except TypeError as e:          # missing / unexpected arguments
        return {"error": f"bad arguments: {e}"}
    except Exception as e:          # the tool itself failed
        return {"error": f"{type(e).__name__}: {e}"}


def run_agent(model, tools, task, system="You are a helpful agent. Use tools when needed.",
              max_steps=8, guards=(), on_event=print_event):
    """Run the observe-think-act loop until a final answer, a guard, or max_steps."""
    messages = [{"role": "system", "content": system},
                {"role": "user", "content": task}]
    schemas = [fn.schema for fn in tools.values()]
    state = {"messages": messages, "steps": 0, "tool_calls": 0, "errors": 0}

    for step in range(1, max_steps + 1):
        state["steps"] = step
        for guard in guards:                        # stop checks (lesson 07)
            reason = guard(state)
            if reason:
                on_event("stopped", {"step": step, "reason": reason})
                return {"answer": None, "stopped": reason, **state}

        reply = model(messages, schemas)            # THINK
        messages.append(reply)
        if not reply.get("tool_calls"):
            on_event("final", {"step": step, "content": reply["content"]})
            return {"answer": reply["content"], "stopped": None, **state}

        for c in reply["tool_calls"]:               # ACT
            state["tool_calls"] += 1
            on_event("tool_call", {"step": step, **c})
            outcome = execute(tools, c)
            if "error" in outcome:
                state["errors"] += 1
            content = json.dumps(outcome["result"] if "result" in outcome else outcome)
            on_event("tool_result", {"step": step, "id": c["id"], "content": content,
                                     "is_error": "error" in outcome})
            messages.append({"role": "tool", "tool_call_id": c["id"],
                             "content": content})   # OBSERVE on the next turn

    reason = f"step limit ({max_steps}) reached"
    on_event("stopped", {"step": max_steps, "reason": reason})
    return {"answer": None, "stopped": reason, **state}
```

A few details worth noticing:

- `execute` **never raises**. Unknown tool, broken JSON, wrong arguments and tool
  crashes all become a small `{"error": ...}` object the model can read.
- Events go through `on_event`, not `print`. Lesson 09 replaces the printer with a
  tracer without touching the loop.
- `guards` is an empty hook for now. Lesson 07 plugs stop conditions into it.
- The return value distinguishes "answered" from "stopped", so callers never mistake a
  timeout for an answer.

## Giving it a task

The task: *"Should I take an umbrella to my next meeting?"* Answering requires two
dependent tool calls — find the meeting, then check the weather **where the meeting
is**. The second call's argument comes from the first call's result, which is exactly
the kind of dependency that makes this an agent task rather than a fixed pipeline.

```python title="umbrella_agent.py"
from tools import tool, registry
from mini_agent import run_agent, call, answer, tool_results

@tool
def get_next_meeting():
    """Return the user's next calendar meeting: title, city and start time."""
    return {"title": "Quarterly review", "city": "Pune", "start": "2026-09-28 10:00"}

@tool
def get_weather(city: str, date: str):
    """Forecast for a city on a date. Returns rain probability (0-100) and summary.

    Args:
        city: City name, e.g. 'Pune'
        date: Date as YYYY-MM-DD
    """
    fake = {"Pune": {"rain_pct": 80, "summary": "heavy showers"}}
    return fake.get(city, {"rain_pct": 10, "summary": "clear"})

def mock_model(messages, schemas):
    """Rule-based stand-in for an LLM. It reads earlier tool results, like a model would."""
    results = tool_results(messages)
    if len(results) == 0:
        return call("get_next_meeting")
    if len(results) == 1:
        meeting = results[0]
        return call("get_weather", "c2", city=meeting["city"],
                    date=meeting["start"][:10])
    meeting, weather = results
    advice = "take an umbrella" if weather["rain_pct"] >= 50 else "no umbrella needed"
    return answer(f"Your next meeting ({meeting['title']}) is in {meeting['city']} with "
                  f"{weather['rain_pct']}% chance of rain ({weather['summary']}): {advice}.")

result = run_agent(mock_model, registry(get_next_meeting, get_weather),
                   "Should I take an umbrella to my next meeting?")
print("ANSWER:", result["answer"])
print("steps:", result["steps"], "| tool calls:", result["tool_calls"],
      "| messages:", len(result["messages"]))
```

```text
  step 1: get_next_meeting({})
      -> {"title": "Quarterly review", "city": "Pune", "start": "2026-09-28 10:00"}
  step 2: get_weather({"city": "Pune", "date": "2026-09-28"})
      -> {"rain_pct": 80, "summary": "heavy showers"}
  step 3: final answer
ANSWER: Your next meeting (Quarterly review) is in Pune with 80% chance of rain (heavy showers): take an umbrella.
steps: 3 | tool calls: 2 | messages: 7
```

Three model calls, two tool calls, seven messages. Change the city returned by
`get_next_meeting` to `"Jaipur"` and the second call's arguments and the final answer
change with it — the path is driven by observations, not hard-coded.

## Swapping in a real model

The only thing you replace is `mock_model`. A real adapter has this shape (pseudocode —
it is not runnable as written, because every provider's SDK and field names differ, and
they change between versions):

```python
def real_model(messages, schemas):
    # 1. Convert our neutral messages/schemas to the provider's request format.
    request = to_provider_format(messages, schemas)
    # 2. Call the provider's chat/messages endpoint with tools enabled.
    response = provider_client.create(model=MODEL_NAME, **request)
    # 3. Convert the response back to {"role", "content", "tool_calls": [...]}.
    return from_provider_format(response)
```

Write `to_provider_format` and `from_provider_format` from your provider's current
documentation, keep them in one file, and unit-test them with recorded responses. The
[LLM Dev Mastery Path](https://sigilipelli.github.io/llm-dev-mastery-path/) covers API
calls themselves in detail; this course stays on the loop around them.

## How It Actually Works

The loop turns a stateless next-token predictor into a stateful process by **externalizing
state into the message list** and **externalizing effects into your code**. Each
iteration is a pure function from "history" to "next action"; the loop provides the
side effects and grows the history.

That separation is why the mock is a faithful stand-in. The loop cannot tell whether the
next action came from rules or from a neural network — it only sees an assistant message.
It also tells you where bugs can live:

- **Policy bugs** (a wrong decision) — prompts, tool descriptions, model choice.
- **Mechanism bugs** (a right decision executed wrongly) — the dispatcher, validation,
  message formatting, limits. These are ordinary software bugs, and you test them
  exactly as you would any code — with deterministic mocks like this one.

## Common mistakes

- **Coupling the loop to one provider SDK.** You lose the ability to test offline and to
  switch models.
- **Returning `None` on failure.** Callers then treat "gave up" and "answered with
  nothing" the same way. Return an explicit reason.
- **Unbounded loops** (`while True`). Always have a step cap, even in prototypes.
- **Letting the mock drift from reality.** Mocks should occasionally produce the bad
  outputs real models produce — lesson 06 does exactly that.

## Exercise

1. Save `tools.py` (lesson 04), `mini_agent.py` and `umbrella_agent.py` in one folder and
   run it. Confirm the output matches.
2. Make `get_next_meeting` return `{"title": "1:1", "city": "Online", ...}` and update the
   mock so it answers without calling `get_weather` when the meeting is online. How
   many steps does the run take now?
3. Pass `max_steps=2` to `run_agent`. What does the result dictionary contain, and how
   should a user-facing application present it?
