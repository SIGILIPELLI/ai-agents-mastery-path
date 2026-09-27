---
description: "Stream an agent's progress to users: event streams from a generator-based loop, partial text, server-sent events, and cancelling a run midway."
---

# 09 · Streaming Agent Progress

An agent run can take tens of seconds. A spinner for that long feels broken; a live view
— "Searching policies… found 2 results… drafting answer…" — feels fast even when it
isn't, and lets the user stop a run that's heading the wrong way. Streaming is a UX
feature with real architectural consequences.

## Two levels of streaming

1. **Token streaming** — the model's text arrives piece by piece as it's generated.
   Providers support this for plain text and, with varying detail, for tool-call
   arguments. Covered in depth in the
   [LLM Dev Mastery Path](https://sigilipelli.github.io/llm-dev-mastery-path/).
2. **Event streaming** — the *agent* emits structured events: step started, tool called,
   tool finished, approval needed, final answer. This is what a user interface for an
   agent is built on, and it's what this lesson covers.

Most agent UIs combine them: events for structure, tokens inside the "final answer"
event so the answer types itself out.

## From callbacks to a generator

`run_agent` in Level 1 already emits events through `on_event`. That's a **push**
design: the loop calls you. For streaming to a web client it's often nicer to **pull**:
make the loop a Python generator that `yield`s events, so the caller can forward each
one, and can stop iterating to cancel.

```python title="stream_agent.py"
"""Generator version of the agent loop: yields events, can be cancelled."""
import json
from mini_agent import execute

def stream_agent(model, tools, task, max_steps=8):
    messages = [{"role": "system", "content": "You are a helpful agent."},
                {"role": "user", "content": task}]
    schemas = [fn.schema for fn in tools.values()]
    for step in range(1, max_steps + 1):
        yield {"type": "step", "step": step}
        reply = model(messages, schemas)
        messages.append(reply)
        if not reply.get("tool_calls"):
            for chunk in chunks(reply["content"]):            # simulate token streaming
                yield {"type": "text", "delta": chunk}
            yield {"type": "done", "steps": step}
            return
        for c in reply["tool_calls"]:
            yield {"type": "tool_start", "name": c["name"], "args": json.loads(c["arguments"])}
            outcome = execute(tools, c)
            content = json.dumps(outcome.get("result", outcome))
            yield {"type": "tool_end", "name": c["name"], "ok": "error" not in outcome,
                   "preview": content[:60]}
            messages.append({"role": "tool", "tool_call_id": c["id"], "content": content})
    yield {"type": "stopped", "reason": "step limit"}

def chunks(text, size=3):
    """Split text into small word groups, standing in for streamed tokens."""
    w = text.split(" ")
    for i in range(0, len(w), size):
        yield " ".join(w[i:i + size]) + (" " if i + size < len(w) else "")

def to_sse(event):
    """Format one event as a server-sent-events message for a browser EventSource."""
    return f"event: {event['type']}\ndata: {json.dumps(event)}\n\n"
```

A consumer that renders a live status line and the answer as it streams, plus one that
**cancels** as soon as it sees a tool it doesn't like:

```python title="stream_demo.py"
from tools import tool, registry
from mini_agent import call, answer, tool_results
from stream_agent import stream_agent, to_sse

@tool
def search_flights(origin: str, dest: str):
    """Find flights between two airport codes.

    Args:
        origin: IATA code, e.g. 'BLR'
        dest: IATA code, e.g. 'DEL'
    """
    return {"flights": [{"no": "XY123", "dep": "07:10", "price": 5400},
                        {"no": "XY456", "dep": "18:40", "price": 4700}]}

@tool
def book_flight(flight_no: str):
    """Book a flight for the user. Charges their saved card.

    Args:
        flight_no: Flight number from search_flights
    """
    return {"booked": flight_no}

def model(messages, schemas):
    r = tool_results(messages)
    if not r:
        return call("search_flights", origin="BLR", dest="DEL")
    if "book" in messages[1]["content"] and len(r) == 1:
        return call("book_flight", "c2", flight_no="XY456")
    return answer("The cheapest option is XY456 at 18:40 for 4700; the morning flight "
                  "XY123 costs 5400.")

tools = registry(search_flights, book_flight)

print("--- live view ---")
for ev in stream_agent(model, tools, "Find me flights from BLR to DEL"):
    if ev["type"] == "tool_start":
        print(f"[working] {ev['name']} {ev['args']}")
    elif ev["type"] == "tool_end":
        print(f"[done]    {ev['name']} ok={ev['ok']}")
    elif ev["type"] == "text":
        print(ev["delta"], end="", flush=True)
    elif ev["type"] == "done":
        print(f"\n[finished in {ev['steps']} steps]")

print("--- cancel on a risky tool ---")
for ev in stream_agent(model, tools, "Find and book the cheapest BLR to DEL flight"):
    if ev["type"] == "tool_start" and ev["name"] == "book_flight":
        print("client cancelled before", ev["name"], "ran")
        break
    print(ev["type"], ev.get("name", ""))

print("--- first SSE frame ---")
print(to_sse({"type": "tool_start", "name": "search_flights", "args": {"origin": "BLR"}}),
      end="")
```

```text
--- live view ---
[working] search_flights {'origin': 'BLR', 'dest': 'DEL'}
[done]    search_flights ok=True
The cheapest option is XY456 at 18:40 for 4700; the morning flight XY123 costs 5400.
[finished in 2 steps]
--- cancel on a risky tool ---
step 
tool_start search_flights
tool_end search_flights
step 
client cancelled before book_flight ran
--- first SSE frame ---
event: tool_start
data: {"type": "tool_start", "name": "search_flights", "args": {"origin": "BLR"}}
```

Breaking out of the `for` loop stopped the generator **before** `book_flight` executed:
the `tool_start` event is yielded before `execute` is called, and a generator does no
work after its consumer stops asking for the next item. That ordering — announce, then
act — is what makes cancellation meaningful.

!!! warning "Cancellation is not approval"
    A user watching a stream and pressing Stop is a convenience, not a safety control:
    they may not be watching, and some actions happen too fast to stop. Risky actions
    still need the approval gate from lesson 08.

## Getting events to a browser

Common transports:

- **Server-sent events (SSE)** — one-way server → browser stream over HTTP; simple, and
  a good default for agent progress. `to_sse` above produces the wire format.
- **WebSockets** — two-way; useful when the user can interject (approve, cancel, add
  info) mid-run on the same connection.
- **Polling** a run-status endpoint — the fallback for long runs that outlive a
  connection; pair with the checkpointing from lesson 02.

## How It Actually Works

A Python generator is a paused function. Each `next()` resumes it until the next
`yield`, hands back the value, and freezes it again — including its local variables
(here, the whole message list). When the consumer stops calling `next()` (by `break`ing
out of a `for` loop), the generator is simply never resumed, and when it's garbage
collected Python raises `GeneratorExit` inside it at the paused `yield`, so any `finally`
blocks run. That makes a generator a natural fit for an agent loop: it produces a
sequence of events lazily, and backpressure and cancellation come for free.

In a web server the same shape appears as an async generator feeding an SSE response.
Each event is flushed to the socket as it's produced, so the browser renders progress
while the agent is still thinking.

## Common mistakes

- **Streaming raw internal data** (full tool results, secrets) to the browser. Send
  previews and user-meaningful labels.
- **Acting before announcing.** Emit `tool_start` before executing so UIs and
  cancellation stay accurate.
- **No terminal event.** Clients need `done` or `stopped` to know the run ended — and
  why.
- **Treating Stop as a safety mechanism.**
- **Losing progress on disconnect.** For long runs, persist state and let clients
  reconnect to a run ID.

## Exercise

1. Add a `finally:` block to `stream_agent` that prints "run closed" and confirm it runs
   both on normal completion and when the consumer breaks early.
2. Add an `approval_needed` event: when the model calls `book_flight`, yield the event
   and wait for the consumer to `send()` a decision back into the generator (look up
   `generator.send`).
3. Write a minimal HTTP endpoint (any framework you know) that streams `to_sse` frames,
   and a ten-line HTML page using `EventSource` to display them.
