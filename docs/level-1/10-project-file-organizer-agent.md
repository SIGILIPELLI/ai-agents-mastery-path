---
description: "Level 1 project: a file-organizer agent confined to a sandbox folder, with path checks, a dry-run plan, step limits, tracing and a structured report."
---

# 10 · Project — File-Organizer Agent

This project combines everything from Level 1 into one small but honest agent: it tidies
a messy folder by moving files into subfolders. It is a good first project because it
*changes things* — so you have to think about confinement, dry runs and reporting, not
just about getting an answer.

## Requirements

1. The agent may only touch files **inside one sandbox folder**. Any path that escapes
   it is rejected by code, whatever the model asks.
2. It supports a **dry run**: it produces the full plan of moves without performing
   them, so a human can review first.
3. It never **overwrites** an existing file.
4. It stops on a step limit, a repeated call, or three consecutive errors.
5. Every run is **traced** to JSONL.
6. It ends with a **structured report**: files moved, files skipped, and why.

## Tools

| Tool | Effect | Notes |
|---|---|---|
| `list_files()` | read-only | names and sizes in the sandbox root only |
| `move_file(name, folder)` | **write** | creates `folder` if needed; refuses overwrites and escapes; in dry-run mode records the plan instead |
| `finish(moved, skipped, notes)` | ends the run | the finish-tool pattern from lesson 08 |

Deliberately missing: delete, rename, and anything that reads file *contents*. The agent
can't do what it has no tool for — the cheapest safety feature there is.

## The code

```python title="organizer.py"
"""File-organizer agent confined to a sandbox folder (Level 1 project)."""
import json
from pathlib import Path
from tools import tool, registry
from mini_agent import run_agent
from guards import repeated_call, too_many_errors

class Sandbox:
    def __init__(self, root, dry_run=True):
        self.root = Path(root).resolve()
        self.dry_run = dry_run
        self.plan = []            # moves performed (or planned, in dry-run mode)

    def inside(self, *parts):
        """Resolve a path and refuse anything outside the sandbox root."""
        p = self.root.joinpath(*parts).resolve()
        if p != self.root and self.root not in p.parents:
            raise PermissionError(f"path '{'/'.join(parts)}' is outside the sandbox; "
                                  "use plain file and folder names only")
        return p

    def tools(self):
        box = self

        @tool
        def list_files():
            """List files (not folders) in the sandbox root with their sizes in bytes."""
            return {"files": sorted(
                [{"name": p.name, "bytes": p.stat().st_size}
                 for p in box.root.iterdir() if p.is_file()],
                key=lambda f: f["name"])}

        @tool
        def move_file(name: str, folder: str):
            """Move one file from the sandbox root into a subfolder (created if missing).
            Never overwrites. In dry-run mode, records the move without doing it.

            Args:
                name: File name exactly as returned by list_files
                folder: Destination subfolder name, e.g. 'images'
            """
            src, dest_dir = box.inside(name), box.inside(folder)
            if not src.is_file():
                raise FileNotFoundError(f"'{name}' is not a file in the sandbox root; "
                                        "call list_files to see valid names")
            dest = dest_dir / src.name
            if dest.exists():
                raise FileExistsError(f"'{folder}/{name}' already exists; skip this file")
            if not box.dry_run:
                dest_dir.mkdir(exist_ok=True)
                src.rename(dest)
            box.plan.append({"file": name, "to": folder})
            return {"moved": name, "to": folder, "dry_run": box.dry_run}

        @tool
        def finish(moved: int, skipped: str, notes: str):
            """Call once when done. Reports the number of files moved, a comma-separated
            list of skipped files, and a one-sentence note.

            Args:
                moved: Number of files moved (or planned in dry run)
                skipped: Comma-separated skipped file names, or '' if none
                notes: One sentence for the user
            """
            return {"report": {"moved": moved, "skipped": skipped, "notes": notes}}

        return registry(list_files, move_file, finish)

def ended_with_finish(state):
    """Guard: stop right after the finish tool has returned its report."""
    last = state["messages"][-1]
    if last["role"] == "tool" and '"report"' in last["content"]:
        return "finished"

def organize(model, root, dry_run=True, on_event=None, max_steps=15):
    box = Sandbox(root, dry_run)
    kwargs = {"on_event": on_event} if on_event else {}
    result = run_agent(model, box.tools(),
                       "Tidy this folder: group files into subfolders by type.",
                       max_steps=max_steps, guards=[ended_with_finish,
                       repeated_call(2), too_many_errors(3)], **kwargs)
    report = None
    for m in result["messages"]:
        if m["role"] == "tool" and '"report"' in m["content"]:
            report = json.loads(m["content"])["report"]
    return {"stopped": result["stopped"], "plan": box.plan, "report": report,
            "tool_calls": result["tool_calls"], "errors": result["errors"]}
```

A few decisions to notice:

- **Confinement is enforced in `inside()`**, by resolving the path (which collapses `..`
  and follows symlinks) and checking it is under the root. The tool description asks
  for plain names, but the *code* is what guarantees it.
- **`finish` is a real tool**, so its arguments are validated like any other, and the
  `ended_with_finish` guard turns it into a clean stop.
- **Dry run lives in the tool**, not in the prompt. The model doesn't need to know or
  behave differently.

## A mock model that behaves like a real one — including mistakes

The mock groups by extension, like a sensible model would. It also makes two
realistic errors: it tries a path outside the sandbox (as a model might after seeing a
`..` in a user's request), and it tries to move a file into a folder where a file of
the same name already exists.

```python title="run_organizer.py"
import json
import tempfile
from pathlib import Path
from mini_agent import call, tool_results
from organizer import organize
from tracer import Tracer, replay

FOLDERS = {".jpg": "images", ".png": "images", ".pdf": "documents",
           ".docx": "documents", ".csv": "data"}

def organizer_model(messages, schemas):
    results = tool_results(messages)
    if not results:
        return call("list_files")
    files = results[0]["files"]
    step = len(results)                          # 1 = just listed
    if step == 1:
        return call("move_file", "c1", name="../outside.txt", folder="junk")
    todo = [f["name"] for f in files if Path(f["name"]).suffix in FOLDERS]
    done = step - 2                              # moves attempted so far
    if done < len(todo):
        name = todo[done]
        return call("move_file", f"m{done}", name=name, folder=FOLDERS[Path(name).suffix])
    moved = sum(1 for r in results if "moved" in r)
    skipped = [f["name"] for f in files if Path(f["name"]).suffix not in FOLDERS]
    skipped += [todo[i] for i, r in enumerate(results[2:2 + len(todo)]) if "error" in r]
    return call("finish", "fin", moved=moved, skipped=", ".join(skipped),
                notes="Grouped files by type; unknown types and conflicts left in place.")

def make_mess(root):
    for name in ["holiday.jpg", "logo.png", "invoice-0925.pdf", "cv.docx",
                 "sales.csv", "notes.xyz"]:
        (root / name).write_text("x" * 10)
    (root / "images").mkdir()
    (root / "images" / "logo.png").write_text("older logo")   # forces a conflict

with tempfile.TemporaryDirectory() as tmp:
    root = Path(tmp)
    make_mess(root)

    print("=== DRY RUN ===")
    out = organize(organizer_model, root, dry_run=True)
    print(json.dumps(out["report"], indent=1))
    print("planned:", [f"{p['file']} -> {p['to']}" for p in out["plan"]])
    print("files still in root after dry run:",
          sorted(p.name for p in root.iterdir() if p.is_file()))

    print("\n=== REAL RUN (traced) ===")
    tracer = Tracer(str(root.parent / "organizer_trace.jsonl"), task="tidy",
                    prompt_version="organizer-v1")
    out = organize(organizer_model, root, dry_run=False, on_event=tracer)
    replay(str(root.parent / "organizer_trace.jsonl"), tracer.run_id)
    print("stopped:", out["stopped"], "| errors:", out["errors"])
    for p in sorted(root.rglob("*")):
        if p.is_file():
            print("  ", p.relative_to(root))
    Path(root.parent / "organizer_trace.jsonl").unlink()
```

```text
=== DRY RUN ===
  step 1: list_files({})
      -> {"files": [{"name": "cv.docx", "bytes": 10}, {"name": "holiday.jpg", "bytes": 10}, {"name": "invoice...
  step 2: move_file({"name": "../outside.txt", "folder": "junk"})
      -> {"error": "PermissionError: path '../outside.txt' is outside the sandbox; use plain file and folder ...
  step 3: move_file({"name": "cv.docx", "folder": "documents"})
      -> {"moved": "cv.docx", "to": "documents", "dry_run": true}
  step 4: move_file({"name": "holiday.jpg", "folder": "images"})
      -> {"moved": "holiday.jpg", "to": "images", "dry_run": true}
  step 5: move_file({"name": "invoice-0925.pdf", "folder": "documents"})
      -> {"moved": "invoice-0925.pdf", "to": "documents", "dry_run": true}
  step 6: move_file({"name": "logo.png", "folder": "images"})
      -> {"error": "FileExistsError: 'images/logo.png' already exists; skip this file"}
  step 7: move_file({"name": "sales.csv", "folder": "data"})
      -> {"moved": "sales.csv", "to": "data", "dry_run": true}
  step 8: finish({"moved": 4, "skipped": "notes.xyz, logo.png", "notes": "Grouped files by type; unknown types and conflicts left in place."})
      -> {"report": {"moved": 4, "skipped": "notes.xyz, logo.png", "notes": "Grouped files by type; unknown t...
  stopped: finished
{
 "moved": 4,
 "skipped": "notes.xyz, logo.png",
 "notes": "Grouped files by type; unknown types and conflicts left in place."
}
planned: ['cv.docx -> documents', 'holiday.jpg -> images', 'invoice-0925.pdf -> documents', 'sales.csv -> data']
files still in root after dry run: ['cv.docx', 'holiday.jpg', 'invoice-0925.pdf', 'logo.png', 'notes.xyz', 'sales.csv']

=== REAL RUN (traced) ===
RUN  task='tidy' prompt=organizer-v1
  s1 CALL   list_files {}
  s1 result {"files": [{"name": "cv.docx", "bytes": 10}, {"name": "holiday.jpg", "
  s2 CALL   move_file {"name": "../outside.txt", "folder": "junk"}
  s2 ERROR  {"error": "PermissionError: path '../outside.txt' is outside the sandb
  s3 CALL   move_file {"name": "cv.docx", "folder": "documents"}
  s3 result {"moved": "cv.docx", "to": "documents", "dry_run": false}
  s4 CALL   move_file {"name": "holiday.jpg", "folder": "images"}
  s4 result {"moved": "holiday.jpg", "to": "images", "dry_run": false}
  s5 CALL   move_file {"name": "invoice-0925.pdf", "folder": "documents"}
  s5 result {"moved": "invoice-0925.pdf", "to": "documents", "dry_run": false}
  s6 CALL   move_file {"name": "logo.png", "folder": "images"}
  s6 ERROR  {"error": "FileExistsError: 'images/logo.png' already exists; skip thi
  s7 CALL   move_file {"name": "sales.csv", "folder": "data"}
  s7 result {"moved": "sales.csv", "to": "data", "dry_run": false}
  s8 CALL   finish {"moved": 4, "skipped": "notes.xyz, logo.png", "notes": "Grouped files by type; unknown types and conflicts left in place."}
  s8 result {"report": {"moved": 4, "skipped": "notes.xyz, logo.png", "notes": "Gr
  STOP   finished
stopped: finished | errors: 2
   data/sales.csv
   documents/cv.docx
   documents/invoice-0925.pdf
   images/holiday.jpg
   images/logo.png
   logo.png
   notes.xyz
```

Read the output carefully: the escape attempt was refused by code; the conflicting
`logo.png` was skipped instead of overwriting the older file; the dry run changed
nothing on disk; and the real run produced the same plan, then carried it out.

## How It Actually Works

The safety of this agent does not depend on the model being well-behaved, and that is
the core lesson of the project. Three layers make it so:

1. **Capability limitation** — the tool set has no delete, no content reading, no
   network. The worst a confused model can do is move a file into a silly folder.
2. **Mechanical enforcement** — `Path.resolve()` normalizes `..` segments and symlinks
   into an absolute path, and the parent check rejects anything outside the root. The
   check happens in the tool, after the model has spoken, so no wording can bypass it.
3. **Reversibility and review** — dry-run first, never overwrite, and a trace of every
   move mean a bad run can be understood and undone.

You will see the same three layers — limit capabilities, enforce in code, keep actions
reviewable and reversible — in every serious agent design in Levels 3 and 4.

## Common mistakes

- **Checking paths with string operations** (`startswith(root)`) instead of resolving
  them; `/sandbox-evil` starts with `/sandbox`.
- **Letting the model choose the sandbox root** as a parameter.
- **Dry run implemented as a prompt instruction** ("don't actually move anything").
- **Overwriting on conflict**, which destroys data silently.
- **Reporting what the model *said* it did** instead of what your code recorded in
  `plan`.

## Exercise

1. Run the project. Then add a `.txt` → `documents` mapping to the mock and a
   `readme.txt` to the mess; confirm the report changes.
2. Add an **undo** feature: from the recorded `plan`, write a function that moves every
   file back. Test it after a real run.
3. Move the repeated-call check *inside* the act step so a duplicate `move_file` is
   refused before execution. Write a mock that tries the same move twice to test it.
4. Replace the mock with a real model using the adapter shape from lesson 05. Compare
   its plan with the mock's on the same folder, in dry-run mode only, and note any
   differences.
