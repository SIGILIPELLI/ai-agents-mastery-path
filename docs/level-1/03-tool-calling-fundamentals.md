---
description: "How tool (function) calling works on the wire: tool schemas, the model's structured call, who executes it, parallel calls and argument validation."
---

# 03 · Tool Calling Fundamentals

Tool calling — also called *function calling* — is the mechanism that turns a text
generator into something that can act. It is simpler than it sounds, and the single most
important fact about it is this: **the model never runs anything.** It writes a request.
Your code decides whether and how to fulfil it.

## The round trip

1. **You declare tools.** Along with the messages, you send a list of tool definitions:
   a name, a description, and a JSON Schema for the arguments.
2. **The model emits a call.** Instead of (or alongside) text, the response contains a
   structured object: the tool name, an ID, and the arguments as JSON.
3. **Your code executes.** You look up the function, validate the arguments, run it,
   and capture the result.
4. **You return the result.** You append a *tool result* message, referencing the call
   ID, and call the model again.

## What it looks like

Every major provider uses a variation of the same shape. Field names differ — some call
the schema `parameters`, others `input_schema`; some put calls in a `tool_calls` array,
others in content blocks of type `tool_use` — so treat the following as a
**vendor-neutral sketch**, and check your provider's current reference for exact names.

A tool definition you send:

```json
{
  "name": "get_order",
  "description": "Look up one customer order by its ID. Returns status, items and dates. Use when the user mentions an order number.",
  "parameters": {
    "type": "object",
    "properties": {
      "order_id": {"type": "string", "description": "Order ID such as 'A-10293'"}
    },
    "required": ["order_id"]
  }
}
```

A tool call the model returns:

```json
{
  "role": "assistant",
  "content": "",
  "tool_calls": [
    {"id": "call_7", "name": "get_order", "arguments": "{\"order_id\": \"A-10293\"}"}
  ]
}
```

The result you send back:

```json
{"role": "tool", "tool_call_id": "call_7",
 "content": "{\"status\": \"shipped\", \"shipped_on\": \"2026-09-20\"}"}
```

Notice that `arguments` is often a **string containing JSON**, not a JSON object. It was
generated token by token, so it can be malformed. Always parse defensively.

## Parallel calls

Many models can return several tool calls in one response — for example, fetching three
orders at once. Your loop should execute each (in parallel if they are independent and
read-only) and return **one result message per call ID**, in any order. If you only
answer the first call, most APIs will reject the next request.

## Controlling whether a tool is used

Providers usually offer a *tool choice* setting with options along the lines of:

- **auto** — the model decides whether to call a tool or answer.
- **required / any** — the model must call some tool.
- **a specific tool** — the model must call that tool (useful for forcing structured
  output, lesson 08).
- **none** — tools are visible but must not be called.

## Worked example: a dispatcher that validates before running

The dispatcher below is the heart of every agent's "act" step. It parses the arguments,
checks them against the declared schema (a deliberately tiny validator — real projects
use a library such as `jsonschema` or `pydantic`), and only then calls the function.

```python title="dispatch.py"
import json

def get_order(order_id):
    orders = {"A-10293": {"status": "shipped", "shipped_on": "2026-09-20"}}
    if order_id not in orders:
        raise LookupError(f"no order {order_id}")
    return orders[order_id]

TOOLS = {
    "get_order": {
        "fn": get_order,
        "schema": {"type": "object",
                   "properties": {"order_id": {"type": "string"}},
                   "required": ["order_id"]},
    }
}

PY_TYPES = {"string": str, "integer": int, "number": (int, float), "boolean": bool}

def validate(args, schema):
    """Return a list of problems (empty if valid). Handles flat objects only."""
    problems = []
    for name in schema.get("required", []):
        if name not in args:
            problems.append(f"missing required argument '{name}'")
    for name, value in args.items():
        spec = schema["properties"].get(name)
        if spec is None:
            problems.append(f"unexpected argument '{name}'")
        elif not isinstance(value, PY_TYPES[spec["type"]]):
            problems.append(f"'{name}' should be {spec['type']}")
    return problems

def dispatch(call):
    tool = TOOLS.get(call["name"])
    if tool is None:
        return {"error": f"unknown tool '{call['name']}'"}
    try:
        args = json.loads(call["arguments"])
    except json.JSONDecodeError as e:
        return {"error": f"arguments were not valid JSON: {e.msg}"}
    problems = validate(args, tool["schema"])
    if problems:
        return {"error": "; ".join(problems)}
    try:
        return {"result": tool["fn"](**args)}
    except Exception as e:
        return {"error": f"{type(e).__name__}: {e}"}

calls = [
    {"id": "1", "name": "get_order", "arguments": '{"order_id": "A-10293"}'},
    {"id": "2", "name": "get_order", "arguments": '{"order_id": 10293}'},
    {"id": "3", "name": "get_order", "arguments": '{"order": "A-1"}'},
    {"id": "4", "name": "get_order", "arguments": '{"order_id": "A-1"'},
    {"id": "5", "name": "cancel_order", "arguments": '{}'},
    {"id": "6", "name": "get_order", "arguments": '{"order_id": "Z-9"}'},
]
for c in calls:
    print(c["id"], json.dumps(dispatch(c)))
```

```text
1 {"result": {"status": "shipped", "shipped_on": "2026-09-20"}}
2 {"error": "'order_id' should be string"}
3 {"error": "missing required argument 'order_id'; unexpected argument 'order'"}
4 {"error": "arguments were not valid JSON: Expecting ',' delimiter"}
5 {"error": "unknown tool 'cancel_order'"}
6 {"error": "LookupError: no order Z-9"}
```

Six calls, six different outcomes, and **none of them crashed the program**. Each
problem became a short, specific message that can be sent back to the model as the
tool result — which is exactly what lesson 06 builds on.

## How It Actually Works

Models learn tool calling during fine-tuning on transcripts that contain tool
definitions, calls and results in a fixed internal format. At inference time, the
provider renders your tool list into the prompt (usually near the system prompt) in that
format. When the model's most likely continuation is a call, it emits the special tokens
that begin one, then the tool name and argument JSON.

Some providers add **constrained decoding**: while the model is writing arguments, the
sampler masks out tokens that would break the JSON Schema, so the output is guaranteed to
parse (sometimes offered as a "strict" mode). Without it, arguments are merely *likely*
to be valid. Either way, schema-valid is not the same as *correct*: `{"order_id":
"A-99999"}` parses perfectly and refers to an order that doesn't exist. Validation in
your code covers structure; the tool itself must handle meaning.

Because definitions are part of the prompt, **every tool costs tokens on every call**,
and a long tool list competes for the model's attention. That is a practical reason to
keep toolsets small and focused.

## Common mistakes

- **Executing arguments without validation** — especially when they reach a shell, SQL
  query or file path. The model's output is untrusted input.
- **Letting an exception escape the loop.** One bad argument should produce an error
  message to the model, not a stack trace to the user.
- **Dropping the call ID**, or answering only some of several parallel calls.
- **Assuming names are stable across providers.** Keep a thin adapter between your
  internal message format and each provider's wire format.
- **Relying on "tool choice: required" to end a loop.** If the model must always call a
  tool, it can never answer; you need an explicit "finish" tool in that mode.

## Exercise

1. Extend `validate` to support an `enum` list (e.g. `"status": {"type": "string",
   "enum": ["open", "closed"]}`) and test it with a bad value.
2. Add a second tool, `list_orders(customer_id, limit)`, where `limit` is an integer,
   and write three calls that should fail validation for different reasons.
3. Decide which of your dispatcher's error messages would help a model fix its call and
   which would not. Rewrite the unhelpful ones.
