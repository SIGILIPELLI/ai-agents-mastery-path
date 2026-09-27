---
description: "Record what an agent did and why: structured events, a JSONL trace per run, timings, and a replay view you can read when a run goes wrong."
---

# 09 · Logging & Tracing an Agent

When an ordinary function returns the wrong value, you read the code. When an agent
returns the wrong answer, the code is usually fine — the problem is in *what happened*
during that particular run: which tool it chose, what came back, what it did next. The
only way to answer "why did it do that?" is a **trace**: a faithful record of every
step.

## Logs vs. traces

- A **log** is a stream of lines: useful, but events from many concurrent runs
  interleave, and relationships between them are implicit.
- A **trace** groups everything that happened in one run under a **run ID**, orders it,
  times it, and nests it: a run contains steps, a step contains a model call and tool
  calls. Each timed unit is a **span**.

For agents you want traces. Industry tooling (OpenTelemetry and a number of
LLM-specific observability products) uses the same span model, so a home-made tracer
that follows it migrates easily later (Level 4 lesson 02).

## What to record

| Event | Fields |
|---|---|
| run start | run id, task, model name/version, prompt version, tool list, limits |
| model call | step, input size (messages / tokens if available), output, latency |
| tool call | step, tool name, arguments, result (or error), latency |
| stop | reason (answer / guard name / error), totals |

Record the **prompt version** and **tool definitions**: when behaviour changes next
month, you need to know whether it was the model, the prompt, or a tool.

Be deliberate about sensitive data. Tool arguments and results may contain personal
information or secrets. Redact before writing, restrict who can read traces, and set a
retention period.

## Worked example: a JSONL tracer

`run_agent` from lesson 05 already reports events through `on_event`. A tracer is just a
different event handler — the loop doesn't change.

```python title="tracer.py"
"""Write one JSON line per agent event, grouped by run id, with timings."""
import json
import re
import time
import uuid

# Matches key/value pairs like  api_key": "abc  or  password=abc, including the
# backslash-escaped quotes that appear when JSON is nested inside a JSON string.
SECRET = re.compile(r'(api[_-]?key|password|token)(\\?")?(\s*[:=]\s*)(\\?")?[^"\\,}\s]+',
                    re.I)

def redact(text):
    return SECRET.sub(r"\1\2\3\4[REDACTED]", text)

class Tracer:
    def __init__(self, path, task, **meta):
        self.path, self.run_id = path, uuid.uuid4().hex[:12]
        self.t0 = self.last = time.perf_counter()
        self._write("run_start", {"task": task, **meta})

    def _write(self, event, data):
        now = time.perf_counter()
        record = {"run_id": self.run_id, "event": event,
                  "t_ms": round((now - self.t0) * 1000, 1),
                  "since_prev_ms": round((now - self.last) * 1000, 1), **data}
        self.last = now
        with open(self.path, "a") as f:
            f.write(redact(json.dumps(record)) + "\n")

    def __call__(self, event, data):          # plugs into run_agent(on_event=...)
        self._write(event, data)

def replay(path, run_id=None):
    """Print a readable timeline of one run from a JSONL trace file."""
    rows = [json.loads(line) for line in open(path)]
    run_id = run_id or rows[-1]["run_id"]
    for r in (r for r in rows if r["run_id"] == run_id):
        if r["event"] == "run_start":
            print(f"RUN  task={r['task']!r} prompt={r.get('prompt_version')}")
        elif r["event"] == "tool_call":
            print(f"  s{r['step']} CALL   {r['name']} {r['arguments']}")
        elif r["event"] == "tool_result":
            flag = "ERROR " if r["is_error"] else "result"
            print(f"  s{r['step']} {flag} {r['content'][:70]}")
        elif r["event"] == "final":
            print(f"  s{r['step']} FINAL  {r['content'][:70]}")
        elif r["event"] == "stopped":
            print(f"  STOP   {r['reason']}")
```

Trace a run in which one tool leaks a secret in its result — the redactor catches it
before it reaches disk:

```python title="traced_run.py"
import json
import os
from tools import tool, registry
from mini_agent import run_agent, call, answer, tool_results
from tracer import Tracer, replay

@tool
def get_service_config(service: str):
    """Return deployment config for a service.

    Args:
        service: Service name, e.g. 'billing-api'
    """
    return {"service": service, "replicas": 3, "api_key": "sk-live-123456",
            "region": "eu-west"}

def model(messages, schemas):
    if not tool_results(messages):
        return call("get_service_config", service="billing-api")
    cfg = tool_results(messages)[-1]
    return answer(f"billing-api runs {cfg['replicas']} replicas in {cfg['region']}.")

path = "trace.jsonl"
if os.path.exists(path):
    os.remove(path)
tracer = Tracer(path, task="How is billing-api deployed?",
                prompt_version="ops-v3", model="mock")
result = run_agent(model, registry(get_service_config),
                   "How is billing-api deployed?", on_event=tracer)

replay(path)
print("secret on disk?", "sk-live" in open(path).read())
print("events recorded:", [json.loads(l)["event"] for l in open(path)])
```

```text
RUN  task='How is billing-api deployed?' prompt=ops-v3
  s1 CALL   get_service_config {"service": "billing-api"}
  s1 result {"service": "billing-api", "replicas": 3, "api_key": "[REDACTED]", "re
  s2 FINAL  billing-api runs 3 replicas in eu-west.
secret on disk? False
events recorded: ['run_start', 'tool_call', 'tool_result', 'final']
```

Every line in `trace.jsonl` also carries `t_ms` and `since_prev_ms`, so you can see
where time went — with a real model, almost always in the model calls.

!!! warning "Redaction by regex is a safety net, not a strategy"
    The pattern above catches obvious key names. It will miss secrets under unexpected
    field names and personal data in free text. The better fix is upstream: tools should
    not return secrets to the model at all. Nothing the model doesn't need should be in
    its context — or in your traces.

## Reading traces to debug

When a run misbehaves, read its trace top to bottom and ask at each step:

1. **Did the model have what it needed?** If the key fact wasn't in any tool result, the
   fix is a tool or retrieval problem, not a prompt problem.
2. **Was the choice reasonable given the context?** If yes, the problem is upstream
   (misleading tool output, ambiguous task). If no, look at the tool descriptions and
   instructions.
3. **Was the action executed as requested?** If the arguments were right but the effect
   wrong, it's an ordinary bug in the tool.

Collect traces of bad runs; they become test cases (Level 3 lesson 05, Level 4
lesson 07).

## How It Actually Works

A trace is useful because agent behaviour is **path-dependent**: the same task can take
different routes on different runs, and the route determines the result. Logging only
inputs and final outputs throws away the route. Recording each event with a shared run
ID and monotonic timings lets you reconstruct the exact sequence — effectively a
replayable record of the loop's state transitions.

Appending JSON lines (JSONL) is a deliberate choice: each line is written atomically in
practice for small records, a crash mid-run leaves every earlier line intact, and the
format streams easily into log pipelines and databases. `time.perf_counter()` is used
for durations because it is monotonic — unaffected by system clock changes — whereas
wall-clock time is what you would record for "when did this happen".

## Common mistakes

- **Printing to stdout and calling it tracing.** You need run IDs and structure to
  query later.
- **Logging secrets and personal data** in tool arguments/results.
- **Not recording versions** (prompt, model, tool schema), so regressions can't be
  attributed.
- **Tracing only failures.** You need successful runs to know what "normal" looks like.
- **Unbounded trace size** — store large tool results by reference or truncated, with
  the full size noted.

## Exercise

1. Add a `model_call` event to `run_agent` (emit it just before and after calling the
   model) and record the number of messages sent. Replay a run and check where the
   time went.
2. Extend `redact` to mask email addresses. Write a tool that returns one and confirm it
   never reaches `trace.jsonl`.
3. Run `umbrella_agent.py` (lesson 05) three times with the tracer, then write a
   ten-line script that reads the trace file and prints, per run, the number of steps,
   tool calls and total time.
