---
description: "Least privilege for agents: acting with the user's delegated authority, per-tool scopes, tenant isolation, server-side credentials and confirmation gates."
---

# 03 · Security & Least Privilege

The most important security question for any agent is not "can the model be tricked?"
— assume it can (L3-09) — but **"what is the worst thing it could do if it were?"**
Least privilege is the discipline of making that answer small: the agent can do only
what the current user may do, only what the current task needs, and nothing
irreversible without someone confirming it.

## Principle 1: the agent acts *as the user*, never as a superuser

A common and dangerous shortcut is to give the agent a service account that can read
and write everything, then ask the model to "only access the current user's data". That
turns every prompt injection or model mistake into a cross-user data breach.

Instead, every tool call carries the **principal** — the user (and tenant) on whose
behalf the agent is acting — and the tool gateway checks the call against that
principal's permissions, exactly as if the user had clicked the button themselves. Where
downstream systems support it, use **delegated, short-lived, narrowly scoped tokens**
(OAuth-style scopes) issued for this user and this agent, rather than long-lived keys.

## Principle 2: scope the toolset to the task

Even within the user's permissions, the agent should hold only the tools the task needs.
A "summarize my unread email" agent needs `list_unread` and `read_message` — not
`send_message`, `delete_message` or `set_forwarding_rule`, even though the user could do
all of those. Define **task profiles** (named toolsets) and choose the profile per
request.

## Principle 3: credentials never enter the context

The model should never see API keys, tokens or passwords. The gateway attaches
credentials when calling downstream systems. If a secret is never in the context, no
injection can exfiltrate it and no trace can leak it.

## Principle 4: confirmation gates for consequential actions

Some actions stay gated even when the user is authorized: sending messages on the user's
behalf, spending money, deleting data, changing permissions or security settings,
deploying. The user (or a designated approver) confirms the specific action with its
specific arguments (L2-08). Authorization answers *may this user do this?*;
confirmation answers *does this user want this, now, exactly like this?*

## Worked example: a permission-checking tool gateway

```python title="gateway.py"
"""Tool gateway: principal-based authorization, task profiles, tenant isolation,
confirmation requirements. The model never sees credentials or other tenants' data."""
from dataclasses import dataclass, field

@dataclass(frozen=True)
class Principal:
    user: str
    tenant: str
    scopes: frozenset = field(default_factory=frozenset)

TOOL_POLICY = {
    # tool:            (required scope,     confirm?)
    "read_ticket":     ("tickets:read",      False),
    "add_note":        ("tickets:write",     False),
    "email_customer":  ("messages:send",     True),
    "delete_ticket":   ("tickets:admin",     True),
}
PROFILES = {"triage": {"read_ticket", "add_note"},
            "customer_reply": {"read_ticket", "email_customer"}}

TICKETS = {"T-1": {"tenant": "acme", "subject": "Login fails"},
           "T-9": {"tenant": "globex", "subject": "Invoice question"}}

class Denied(Exception):
    pass

def authorize(principal, profile, tool, args, confirmed=False):
    if tool not in PROFILES[profile]:
        raise Denied(f"'{tool}' is not available in the '{profile}' task profile")
    scope, needs_confirm = TOOL_POLICY[tool]
    if scope not in principal.scopes:
        raise Denied(f"user lacks scope '{scope}'")
    ticket = TICKETS.get(args.get("ticket_id"))
    if ticket is None:
        raise Denied("no such ticket")                  # same message as cross-tenant:
    if ticket["tenant"] != principal.tenant:            # don't reveal existence
        raise Denied("no such ticket")
    if needs_confirm and not confirmed:
        return "needs_confirmation"
    return "allowed"

ana = Principal("ana", "acme", frozenset({"tickets:read", "tickets:write", "messages:send"}))
checks = [
    ("triage", "read_ticket", {"ticket_id": "T-1"}, False),
    ("triage", "read_ticket", {"ticket_id": "T-9"}, False),       # other tenant
    ("triage", "email_customer", {"ticket_id": "T-1"}, False),    # not in profile
    ("customer_reply", "email_customer", {"ticket_id": "T-1"}, False),
    ("customer_reply", "email_customer", {"ticket_id": "T-1"}, True),
    ("triage", "add_note", {"ticket_id": "T-404"}, False),
]
for profile, tool, args, confirmed in checks:
    try:
        outcome = authorize(ana, profile, tool, args, confirmed)
    except Denied as e:
        outcome = f"DENIED: {e}"
    print(f"{profile:<15} {tool:<15} {args['ticket_id']:<6} confirmed={confirmed!s:<5} -> {outcome}")
```

```text
triage          read_ticket     T-1    confirmed=False -> allowed
triage          read_ticket     T-9    confirmed=False -> DENIED: no such ticket
triage          email_customer  T-1    confirmed=False -> DENIED: 'email_customer' is not available in the 'triage' task profile
customer_reply  email_customer  T-1    confirmed=False -> needs_confirmation
customer_reply  email_customer  T-1    confirmed=True  -> allowed
triage          add_note        T-404  confirmed=False -> DENIED: no such ticket
```

Notice the cross-tenant ticket and the nonexistent ticket produce the **same** message.
Saying "that ticket belongs to another customer" would leak that it exists — a small
information disclosure that an attacker (or a curious model) could use to enumerate
other tenants' data.

## A threat-modelling checklist for agent tools

For each tool, write down:

1. **Who** can cause it to be called? (The user? Anyone who can put text in front of the
   agent?)
2. **What** can it read, write or send, at most?
3. **Whose** permissions does it run with?
4. **Can it be undone?** How?
5. **What's the blast radius** if it's called with attacker-chosen arguments?
6. **What stops that** — in code, not in the prompt?

If question 6 has no answer for a tool with a large blast radius, the tool isn't ready.

## How It Actually Works

Least privilege works because authorization is enforced at the boundary the model
can't cross: the tool gateway, which receives a structured request and checks it
against data the model doesn't control (the authenticated principal, the scope table,
the resource's actual owner). The model can put any string into `ticket_id`; it can't
change who the principal is or which tenant a ticket belongs to. This is the classic
"confused deputy" defence: the agent is a deputy acting for the user, and the system
makes sure it can never wield more authority than the user delegated — no matter who
wrote the words it's acting on.

## Common mistakes

- **One powerful service account for the agent.**
- **"Only access the current user's data" as a prompt instruction.**
- **Tool arguments that choose the tenant or user** — derive them from the session.
- **Credentials in environment variables of a code-execution sandbox** (L3-04).
- **Error messages that reveal existence** of other users' resources.
- **Confirmation fatigue**: gating reads, so users click through gates on writes too.

## Exercise

1. Add a `triage_admin` profile including `delete_ticket` and a principal with the admin
   scope. Confirm deletion still requires confirmation.
2. Add expiry to principals (a `valid_until` timestamp) and deny calls with an expired
   delegation.
3. Fill in the six-question threat model for three tools in an agent you plan to build.
