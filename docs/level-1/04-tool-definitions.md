---
description: "How to design tool names, descriptions, parameters and return values a model can use correctly, plus a decorator that builds schemas from Python."
---

# 04 · Writing Good Tool Definitions

A model uses a tool the way a new colleague uses an unfamiliar internal API with only
the docstring to go on — except the model cannot ask a follow-up question. Most
"the agent picked the wrong tool" bugs are really documentation bugs.

## The four parts of a tool

### 1. Name

Use a verb-object name that says what happens: `search_tickets`, `get_invoice`,
`create_draft_reply`. Avoid names that differ only subtly (`get_user` and `fetch_user`)
or that hide an effect (`process_order` — process how?).

### 2. Description

Write it for the moment of choice. A good description answers:

- **What it does** and **what it returns**, in one or two sentences.
- **When to use it** — and, if a similar tool exists, **when not to**.
- **Limits** — result caps, date ranges, units, required formats.
- **Side effects** — "Creates a draft only; does not send."

```text
Bad:  "Searches tickets."

Good: "Full-text search over support tickets from the last 90 days. Returns up to 10
       matches with id, title, status and created date, newest first. Use for finding
       tickets by keywords; use get_ticket when you already have an id."
```

### 3. Parameters

- Prefer **few, well-typed** parameters. Every optional flag is another decision the
  model can get wrong.
- Use **enums** for closed sets: `"priority": {"enum": ["low", "normal", "high"]}`.
- Describe **formats** explicitly: dates as `YYYY-MM-DD`, money in cents, IDs with an
  example.
- Don't ask the model for things your code already knows — the current user's ID, the
  tenant, an auth token. Inject those server-side. This is both simpler and safer.

### 4. Return value

- Return **compact, structured** data — the fields needed for the next decision, not a
  raw API response with sixty keys.
- Include **identifiers** the model will need for follow-up calls.
- Say when results were **truncated** (`"truncated": true, "total": 412`) so the model
  knows to narrow its query instead of assuming it saw everything.
- Make "nothing found" explicit (`{"matches": []}`) rather than returning an empty
  string the model may misread.

## Granularity: how big should a tool be?

Too fine and the agent needs many steps (and many chances to err): `open_file`,
`read_line`, `close_file`. Too coarse and the tool becomes a workflow the model can't
steer: `do_everything(request)`. Aim for tools that match **one meaningful decision** a
human operator would make: "search", "read this record", "draft a reply", "request
approval".

A useful test: can you describe what the tool does in one sentence without the word
"and"? If not, consider splitting it.

## Worked example: generating schemas from Python functions

Keeping hand-written JSON schemas in sync with code is tedious and error-prone. Most
frameworks derive them from type hints and docstrings. Here is a small version of that
idea we will reuse throughout the course.

```python title="tools.py"
"""Tiny tool registry: build JSON-Schema tool definitions from Python functions."""
import inspect
import typing

_JSON_TYPES = {str: "string", int: "integer", float: "number", bool: "boolean"}

def tool(fn):
    """Decorator: attach a tool schema built from the signature and docstring.

    Docstring format: first paragraph is the description; lines like
    'name: text' under an 'Args:' header describe parameters.
    """
    doc = inspect.getdoc(fn) or ""
    description, _, args_block = doc.partition("Args:")
    arg_docs = {}
    for line in args_block.strip().splitlines():
        if ":" in line:
            k, v = line.split(":", 1)
            arg_docs[k.strip()] = v.strip()

    hints = typing.get_type_hints(fn)
    props, required = {}, []
    for name, param in inspect.signature(fn).parameters.items():
        hint = hints.get(name, str)
        spec = {}
        if typing.get_origin(hint) is typing.Literal:
            spec["type"] = "string"
            spec["enum"] = list(typing.get_args(hint))
        else:
            spec["type"] = _JSON_TYPES.get(hint, "string")
        if name in arg_docs:
            spec["description"] = arg_docs[name]
        props[name] = spec
        if param.default is inspect.Parameter.empty:
            required.append(name)

    fn.schema = {
        "name": fn.__name__,
        "description": " ".join(description.split()),
        "parameters": {"type": "object", "properties": props, "required": required},
    }
    return fn

def registry(*fns):
    """Map tool name -> function, for the dispatcher."""
    return {f.schema["name"]: f for f in fns}
```

Now define a tool the way you would any Python function:

```python title="demo_tools.py"
import json
from typing import Literal
from tools import tool

@tool
def search_tickets(query: str, status: Literal["open", "closed", "any"] = "any",
                   limit: int = 5):
    """Full-text search over support tickets from the last 90 days. Returns up to
    `limit` matches (max 10) with id, title and status, newest first. Use get_ticket
    when you already know an id.

    Args:
        query: Keywords to search for, e.g. 'refund delayed'
        status: Filter by ticket status
        limit: Number of results, 1-10
    """
    return {"matches": [], "truncated": False}

print(json.dumps(search_tickets.schema, indent=2))
```

```text
{
  "name": "search_tickets",
  "description": "Full-text search over support tickets from the last 90 days. Returns up to `limit` matches (max 10) with id, title and status, newest first. Use get_ticket when you already know an id.",
  "parameters": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "Keywords to search for, e.g. 'refund delayed'"
      },
      "status": {
        "type": "string",
        "enum": [
          "open",
          "closed",
          "any"
        ],
        "description": "Filter by ticket status"
      },
      "limit": {
        "type": "integer",
        "description": "Number of results, 1-10"
      }
    },
    "required": [
      "query"
    ]
  }
}
```

The function is now the single source of truth: change a parameter and the schema the
model sees changes with it.

## How It Actually Works

When the model chooses among tools it is, in effect, matching the user's goal and the
current context against the text of every tool definition in its prompt. Names and
descriptions are therefore *retrieval keys*: a tool called `lookup` with the description
"Looks things up" matches every situation weakly, and loses to a tool whose description
contains words that actually appear in the task.

Parameter descriptions act at a second moment — while the model generates argument
values. An example format (`e.g. 'A-10293'`) is especially effective because models
imitate nearby examples closely; the same effect you see with few-shot prompting (see
[Prompt Engineering Mastery Path](https://sigilipelli.github.io/prompt-engineering-mastery-path/)).

Finally, tool output becomes context for the *next* decision. A result that includes
`"truncated": true` changes what the most likely next action is (narrow the search),
whereas a silently truncated list leads the model to conclude that what it saw was
everything.

## Common mistakes

- **Descriptions written for humans who already know the system** ("Uses the v2 path").
- **Overlapping tools** with no guidance on which to prefer.
- **Asking for data the server knows** (user IDs, tokens) — wasteful and a security risk.
- **Free-text parameters for closed sets.** Use enums.
- **Returning raw upstream payloads** full of irrelevant fields and internal IDs.
- **Hiding side effects.** If a tool changes anything, its name and description must say
  so, and your loop should know it too (Level 4 lesson 03).

## Exercise

1. Using the `@tool` decorator, write `get_ticket(ticket_id: str)` and
   `add_internal_note(ticket_id: str, note: str)`. Make the second description state its
   side effect clearly.
2. Take one real API you use (weather, GitHub, your company's) and design a single tool
   around it. Write the return value you'd send the model — then cut it down to only the
   fields needed for the next decision.
3. Swap descriptions between two of your tools and imagine a model choosing between
   them. Which misuse would you expect? That's the bug your real descriptions prevent.
