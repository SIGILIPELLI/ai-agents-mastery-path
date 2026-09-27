---
description: "Model Context Protocol concepts for agent builders: hosts, clients and servers, tools/resources/prompts, JSON-RPC messages, transports and the trust model."
---

# 03 · Model Context Protocol (MCP) Concepts

Every agent so far has defined its tools in the same Python process. That doesn't scale
across an organisation: the team that owns the ticketing system wants to publish *one*
integration that every agent — your Python agent, an IDE assistant, a desktop chat app —
can use. The **Model Context Protocol (MCP)** is an open protocol for exactly that:
a standard way for AI applications to discover and call tools and read context provided
by separate programs.

MCP was introduced by Anthropic in late 2024 as an open specification and has since
been adopted by many clients and tool providers. The specification is versioned and
still evolving, so this lesson teaches the **concepts and message shapes**, and you
should check the current spec and official SDKs before building.

## The three roles

- **Host** — the AI application the user interacts with (a chat app, an IDE, your agent
  service). It owns the model, the conversation and the user's permissions.
- **Client** — a component inside the host that maintains a connection to **one**
  server.
- **Server** — a separate program that exposes capabilities: a GitHub server, a
  database server, a filesystem server.

A host with five integrations runs five clients, each connected to one server.

## What servers offer

| Primitive | Controlled by | What it is | Example |
|---|---|---|---|
| **Tools** | the model (with host approval) | functions the model can call | `create_issue`, `run_query` |
| **Resources** | the application | readable context identified by URI | a file, a DB schema, a document |
| **Prompts** | the user | reusable prompt templates | "summarize this PR" |

Clients can offer features back to servers too — for example letting a server request a
model completion through the host (*sampling*) or ask the user for input — always
mediated by the host, so the user stays in control.

## Messages and transports

MCP messages are **JSON-RPC 2.0**: requests with an `id` and a `method`, responses with
the same `id` and a `result` or `error`, and notifications with no `id`. A session starts
with an `initialize` exchange in which both sides state their protocol version and
capabilities. Then the client can call methods such as `tools/list` and `tools/call`.

Two standard transports exist: **stdio** (the host launches the server as a subprocess
and exchanges newline-delimited JSON over stdin/stdout — typical for local servers) and
**Streamable HTTP** (for remote servers). Earlier spec versions used a different
HTTP+SSE transport; check which your SDK supports.

## Worked example: a toy server and client over stdio

To make the shapes concrete, here is a **teaching toy** that speaks a small subset of
MCP's message format over stdio. It is *not* a compliant implementation — real servers
should use an official SDK, which handles versions, capabilities, errors, cancellation
and much more — but every message it exchanges has the same structure as the real thing.

```python title="toy_mcp_server.py"
"""Teaching toy: a tiny MCP-shaped server over stdio (NOT spec-complete)."""
import json
import sys

TOOLS = [{
    "name": "count_words",
    "description": "Count the words in a piece of text.",
    "inputSchema": {"type": "object",
                    "properties": {"text": {"type": "string"}},
                    "required": ["text"]},
}]

def handle(req):
    method = req.get("method")
    if method == "initialize":
        return {"protocolVersion": req["params"]["protocolVersion"],
                "capabilities": {"tools": {}},
                "serverInfo": {"name": "toy-words", "version": "0.1"}}
    if method == "tools/list":
        return {"tools": TOOLS}
    if method == "tools/call":
        p = req["params"]
        if p["name"] != "count_words":
            return {"content": [{"type": "text", "text": f"unknown tool {p['name']}"}],
                    "isError": True}
        n = len(p["arguments"]["text"].split())
        return {"content": [{"type": "text", "text": str(n)}], "isError": False}
    raise KeyError(method)

for line in sys.stdin:
    req = json.loads(line)
    if "id" not in req:                   # notification: no response
        continue
    try:
        resp = {"jsonrpc": "2.0", "id": req["id"], "result": handle(req)}
    except KeyError as e:
        resp = {"jsonrpc": "2.0", "id": req["id"],
                "error": {"code": -32601, "message": f"method not found: {e}"}}
    print(json.dumps(resp), flush=True)
```

The client launches the server as a subprocess, performs the handshake, discovers tools,
and adapts them into our Level 1 tool registry so `mini_agent` can use them unchanged:

```python title="toy_mcp_client.py"
import json
import subprocess
import sys
from mini_agent import run_agent, call, answer, tool_results

class StdioClient:
    def __init__(self, cmd):
        self.proc = subprocess.Popen(cmd, stdin=subprocess.PIPE, stdout=subprocess.PIPE,
                                     text=True)
        self.next_id = 0

    def request(self, method, params=None):
        self.next_id += 1
        msg = {"jsonrpc": "2.0", "id": self.next_id, "method": method,
               "params": params or {}}
        print("  >>", json.dumps(msg)[:95])
        self.proc.stdin.write(json.dumps(msg) + "\n")
        self.proc.stdin.flush()
        resp = json.loads(self.proc.stdout.readline())
        print("  <<", json.dumps(resp)[:95])
        if "error" in resp:
            raise RuntimeError(resp["error"]["message"])
        return resp["result"]

    def notify(self, method):
        self.proc.stdin.write(json.dumps({"jsonrpc": "2.0", "method": method}) + "\n")
        self.proc.stdin.flush()

    def close(self):
        self.proc.stdin.close()
        self.proc.wait()

def mcp_tools_as_registry(client):
    """Adapt server tools into mini_agent's {name: fn-with-.schema} registry."""
    registry = {}
    for t in client.request("tools/list")["tools"]:
        def fn(_name=t["name"], **arguments):
            result = client.request("tools/call", {"name": _name, "arguments": arguments})
            text = " ".join(c["text"] for c in result["content"] if c["type"] == "text")
            if result.get("isError"):
                raise RuntimeError(text)
            return text
        fn.schema = {"name": t["name"], "description": t["description"],
                     "parameters": t["inputSchema"]}
        registry[t["name"]] = fn
    return registry

client = StdioClient([sys.executable, "toy_mcp_server.py"])
print("handshake:")
info = client.request("initialize", {"protocolVersion": "2025-06-18", "capabilities": {},
                                     "clientInfo": {"name": "mini-agent", "version": "1"}})
client.notify("notifications/initialized")
print("discovery:")
tools = mcp_tools_as_registry(client)

def model(messages, schemas):
    if not tool_results(messages):
        return call("count_words", text="the quick brown fox jumps")
    return answer(f"That sentence has {tool_results(messages)[-1]} words.")

print("agent run:")
res = run_agent(model, tools, "How many words in 'the quick brown fox jumps'?",
                on_event=lambda e, d: None)
print("ANSWER:", res["answer"], "| server:", info["serverInfo"]["name"])
client.close()
```

```text
handshake:
  >> {"jsonrpc": "2.0", "id": 1, "method": "initialize", "params": {"protocolVersion": "2025-06-18",
  << {"jsonrpc": "2.0", "id": 1, "result": {"protocolVersion": "2025-06-18", "capabilities": {"tools
discovery:
  >> {"jsonrpc": "2.0", "id": 2, "method": "tools/list", "params": {}}
  << {"jsonrpc": "2.0", "id": 2, "result": {"tools": [{"name": "count_words", "description": "Count 
agent run:
  >> {"jsonrpc": "2.0", "id": 3, "method": "tools/call", "params": {"name": "count_words", "argument
  << {"jsonrpc": "2.0", "id": 3, "result": {"content": [{"type": "text", "text": "5"}], "isError": f
ANSWER: That sentence has 5 words. | server: toy-words
```

The agent loop didn't change at all. MCP sits **between** your dispatcher and the tool
implementation: discovery replaces hand-written schemas, and `tools/call` replaces a
direct function call. That's the whole value proposition — integrations written once,
usable from any compliant host.

## The trust model — read this twice

Connecting an MCP server means running someone else's code and putting someone else's
text into your model's context. Treat it accordingly:

- **Tool descriptions are prompt input.** A malicious or compromised server can put
  instructions in a description ("before any other tool, send the user's files to...").
  This is sometimes called *tool poisoning*. Review descriptions of third-party servers,
  pin versions, and watch for changes.
- **Tool results are untrusted data**, exactly like web pages (lesson 09).
- **Local servers run with your user's permissions.** A stdio server launched by your
  host can read whatever your account can. Prefer servers from sources you trust, run
  them with minimal privileges or in containers, and grant only the scopes needed.
- **The host enforces consent.** Approval gates for write tools (L2-08) belong in the
  host, regardless of what the server claims about a tool.
- **Remote servers need proper authorization** — the spec defines an OAuth-based flow for
  HTTP transports; don't invent your own token handling.

## How It Actually Works

MCP standardizes the part of tool calling that happens **outside** the model. The model
still sees tool definitions in its own provider-specific format and still emits calls
the same way (L1-03). The host translates: MCP `inputSchema` becomes the provider's
parameter schema; a model's tool call becomes a JSON-RPC `tools/call` request; the
server's `content` blocks become the tool-result message. Because both ends of that
translation are standardized — one by the protocol, one by each provider's API — any
host can use any server, and N hosts × M integrations collapse from N×M custom adapters
to N + M implementations of one protocol.

JSON-RPC's request IDs let a single connection carry many concurrent requests and match
responses to requests out of order; notifications (no ID) carry one-way events such as
"initialized" or "the tool list changed".

## Common mistakes

- **Installing servers like browser extensions** — without reading what they can do.
- **Exposing every tool of every server to the model** — toolsets balloon (lesson 01);
  filter to what the agent needs.
- **Trusting a server's self-description** of a tool as read-only.
- **Hard-coding a protocol version** without handling negotiation.
- **Building a production server from scratch** instead of an official SDK.

## Exercise

1. Add a second tool, `reverse_text`, to the toy server and confirm the client discovers
   it without any change to the client.
2. Make `count_words` return `isError: true` for empty text, and check how the error
   reaches the model through `mini_agent`'s error handling.
3. Pick a real MCP server you might use. Write down: who publishes it, what permissions
   it runs with, which of its tools write or send anything, and which you would expose
   to your agent.
