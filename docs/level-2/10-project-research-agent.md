---
description: "Level 2 project: a graph-based research agent that plans queries, searches notes, drafts with citations, self-checks, and waits for approval before saving."
---

# 10 · Project — Research Agent with Approvals

This project assembles Level 2 into one agent: a **state graph** (lesson 02) that
**plans** its searches (05), uses **retrieval as a tool** (07), drafts a cited brief,
**critiques** it with grounded checks (06), and **interrupts for human approval**
(08) before saving anything — while **streaming** progress (09).

## The brief

> Given a research question and a folder of notes, produce a short brief (≤ 120 words)
> in which every sentence carries a citation to a note that was actually retrieved.
> Nothing is written to disk until a human approves the brief.

## The graph

```text
 plan ──▶ search ──▶ draft ──▶ check ──┬─(problems, revisions < 2)──▶ draft
                                        │
                                        └─(ok or out of revisions)──▶ [interrupt] save ──▶ END
```

- **plan** — turn the question into 2–3 search queries (a model call in real life).
- **search** — run each query against the notes; collect unique snippets.
- **draft** — write the brief from snippets only, citing `[id]` per sentence.
- **check** — grounded checks: word limit, every sentence cited, every citation
  retrieved. Failing drafts loop back once or twice.
- **save** — gated behind an interrupt; only runs after a human resumes the graph.

## The code

The "model" nodes are deterministic functions so the project runs offline. Each one is
marked where a real model call would go.

```python title="research_agent.py"
import json
import re
from pathlib import Path
from graph import Graph, END

NOTES = {
    "sqlite-1": "SQLite stores the whole database in a single file and runs inside the "
                "application process; there is no separate server to operate.",
    "sqlite-2": "SQLite allows many concurrent readers but only one writer at a time; "
                "WAL mode lets readers continue while a write is in progress.",
    "pg-1": "PostgreSQL is a client-server database; applications connect over the "
            "network, and it supports many concurrent writers using MVCC.",
    "pg-2": "PostgreSQL has roles and fine-grained privileges, which helps when several "
            "teams or services share one database.",
    "ops-1": "Backups for a single-file database can be as simple as copying the file "
             "safely, for example with SQLite's online backup API.",
}

def search(query, k=2):
    q = set(re.findall(r"[a-z]+", query.lower()))
    scored = sorted(((len(q & set(re.findall(r"[a-z]+", t.lower()))), i)
                     for i, t in NOTES.items()), reverse=True)
    return [i for s, i in scored[:k] if s >= 2]

g = Graph()

@g.node("plan")
def plan(s):          # real agent: model call returning JSON list of queries
    return {"queries": ["sqlite writer concurrency file server",
                        "postgresql concurrent writers roles privileges",
                        "backup single file database"]}

@g.node("search")
def do_search(s):     # real agent: retrieval tool, possibly chosen by the model
    found = []
    for q in s["queries"]:
        for i in search(q):
            if i not in found:
                found.append(i)
    return {"snippets": {i: NOTES[i] for i in found}}

DRAFTS = [  # real agent: model call with snippets in context; drafts vary by revision
    "SQLite keeps everything in one file with no server to run [sqlite-1]. It permits "
    "one writer at a time, so write-heavy multi-user tools may queue [sqlite-2]. "
    "PostgreSQL handles many concurrent writers [pg-1] and is the industry standard "
    "for every serious application. Backups of SQLite can be a safe file copy [ops-1].",
    "SQLite keeps everything in one file with no server to run [sqlite-1]. It permits "
    "one writer at a time, so write-heavy multi-user tools may queue [sqlite-2]. "
    "PostgreSQL handles many concurrent writers [pg-1] and offers roles for shared "
    "use [pg-2]. Backups of SQLite can be a safe file copy [ops-1].",
]

@g.node("draft")
def draft(s):
    n = s.get("revisions", 0)
    return {"draft": DRAFTS[min(n, len(DRAFTS) - 1)], "revisions": n + 1}

@g.node("check")
def check(s):
    problems = []
    sentences = [x for x in re.split(r"(?<=[.!?])\s+", s["draft"].strip()) if x]
    if len(s["draft"].split()) > 120:
        problems.append("over 120 words")
    for sent in sentences:
        cites = re.findall(r"\[([a-z]+-\d+)\]", sent)
        if not cites:
            problems.append(f"uncited sentence: {sent[:40]}...")
        for c in cites:
            if c not in s["snippets"]:
                problems.append(f"citation {c} was not retrieved")
    if "industry standard" in s["draft"]:
        problems.append("unsupported generalisation: 'industry standard'")
    return {"problems": problems}

@g.node("save")
def save(s):
    Path(s["out_path"]).write_text(s["draft"] + "\n")
    return {"saved_to": s["out_path"]}

g.edge("plan", "search")
g.edge("search", "draft")
g.edge("draft", "check")
g.route("check", lambda s: "draft" if s["problems"] and s["revisions"] < 2 else "save")
g.edge("save", END)
```

Run it: first until the interrupt, then show the human what they're approving, then
resume with approval.

```python title="run_research.py"
import os
from research_agent import g

CKPT, OUT = "research_ckpt.json", "brief.md"
for f in (CKPT, OUT):
    if os.path.exists(f):
        os.remove(f)

state = {"question": "SQLite or PostgreSQL for a small internal tool?", "out_path": OUT}
out = g.run(state, "plan", checkpoint=CKPT, interrupt_before={"save"},
            log=lambda line: print("stream:", line.strip()))

s = out["state"]
print("\nstatus:", out["status"], "| revisions:", s["revisions"],
      "| remaining problems:", s["problems"] or "none")
print("retrieved:", sorted(s["snippets"]))
print("APPROVAL REQUEST — save this brief?\n ", s["draft"])
print("file exists before approval?", os.path.exists(OUT))

nxt, state = g.resume(CKPT, approved_by="reviewer@example.com")
out = g.run(state, nxt, checkpoint=CKPT, interrupt_before={"save"},
            log=lambda line: print("stream:", line.strip()))
print("status:", out["status"], "| saved to:", out["state"]["saved_to"],
      "| approved by:", out["state"]["approved_by"])
```

```text
stream: [plan] -> ['queries']
stream: [search] -> ['snippets']
stream: [draft] -> ['draft', 'revisions']
stream: [check] -> ['problems']
stream: [draft] -> ['draft', 'revisions']
stream: [check] -> ['problems']

status: interrupted | revisions: 2 | remaining problems: none
retrieved: ['ops-1', 'pg-1', 'pg-2', 'sqlite-1', 'sqlite-2']
APPROVAL REQUEST — save this brief?
  SQLite keeps everything in one file with no server to run [sqlite-1]. It permits one writer at a time, so write-heavy multi-user tools may queue [sqlite-2]. PostgreSQL handles many concurrent writers [pg-1] and offers roles for shared use [pg-2]. Backups of SQLite can be a safe file copy [ops-1].
file exists before approval? False
stream: [save] -> ['saved_to']
status: done | saved to: brief.md | approved by: reviewer@example.com
```

Walk through what happened. The first draft contained a sweeping claim ("industry
standard for every serious application") that none of the notes support; the check
node caught it and the graph looped back to `draft`. The second draft passed. The graph
then **stopped before `save`**, with the draft visible for review and nothing on disk.
Only after `resume` — carrying the approver's identity into the state — did `save` run.

## How It Actually Works

Each Level 2 idea contributes one guarantee:

| Component | Guarantee |
|---|---|
| Graph with a revision counter | the draft/check cycle ends (at most 2 revisions) |
| Search results kept in state | the checker knows exactly what evidence existed |
| Grounded check | every sentence is cited, and every citation is to retrieved evidence |
| Interrupt before `save` | no side effect without a recorded human decision |
| Checkpoint file | approval can arrive later, in another process |

None of these guarantees depend on the model behaving well; they depend on the structure
around it. When you swap the deterministic nodes for real model calls, the *quality* of
briefs will vary with the model — but the guarantees won't.

One honest limitation: the check verifies that citations point to retrieved snippets,
not that each sentence is *faithful* to its snippet. A sentence could cite `[pg-1]` and
misstate it. Faithfulness checking (a model-based judge or claim-level matching) is an
evaluation topic in Level 3.

## Common mistakes

- **Checking only the final draft's format** and not its evidence.
- **Putting the approval inside `save`** as an `input()` call — it blocks a server
  process and can't survive a restart. Interrupt instead.
- **Letting the planner write unlimited queries.** Cap them in the node.
- **Losing the approver's identity.** Record who approved and when, in state and trace.

## Exercise

1. Add a note that contradicts another (e.g. an outdated claim about SQLite) and make the
   check flag briefs that cite both for opposite statements. What would a real model
   need in its prompt to handle conflicting sources?
2. Replace `plan` with a function that builds queries from the question's nouns, and
   observe how retrieval changes.
3. Add a `reject` path: resume with `approved_by=None, rejection="too technical"` and
   route back to `draft` with the rejection added as a problem.
4. Swap `draft` for a real model call using your Level 1 adapter. Keep everything else.
   Run it five times and record how often the check passes on the first try.
