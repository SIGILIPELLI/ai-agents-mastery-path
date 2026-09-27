---
description: "Run agent-generated code safely: process isolation, timeouts, resource limits, scrubbed environments, and why real untrusted code needs containers or VMs."
---

# 04 · Sandboxing Code-Execution Agents

Giving an agent a "run Python" tool is enormously powerful: it can compute instead of
guessing, analyse data files, test its own code. It is also the most dangerous tool you
can hand a model, because arbitrary code can do arbitrary things. The code comes from a
model that may have been steered by untrusted input (lesson 09), so treat it as
**untrusted code** — always.

## What can go wrong

| Risk | Example |
|---|---|
| Runaway resources | infinite loop, 50 GB allocation, fork bomb, filling the disk |
| Secret access | reading environment variables, `~/.ssh`, cloud credentials files |
| Data damage | deleting or overwriting files outside a work folder |
| Network abuse | exfiltrating data, calling internal services, scanning the network |
| Escape | exploiting the interpreter or kernel to leave the sandbox |

## Layers of isolation

Think in layers; each stops a different class of problem.

1. **Separate process** with a hard **timeout**. Stops hangs from hanging the agent.
2. **Resource limits** (CPU seconds, memory, file size, number of processes).
3. **Scrubbed environment** — no inherited secrets in env vars; a fresh, empty working
   directory; Python's isolated mode.
4. **Filesystem isolation** — only a scratch directory is writable; ideally nothing else
   is even visible.
5. **Network isolation** — no network by default; allowlist specific hosts if needed.
6. **Kernel-level isolation** — containers with seccomp/AppArmor profiles, user-space
   kernels (such as gVisor), or lightweight virtual machines (such as Firecracker).
   Hosted code-execution sandboxes offered by cloud and AI vendors provide these as a
   service.

**Layers 1–3 can be done with the standard library, and that's what the example below
does. They are *not* a security boundary against malicious code** — a determined
payload can still read your files and use your network. For code from untrusted inputs
in production, use layers 4–6: a container or microVM with no network and no mounted
secrets, destroyed after each run.

## Worked example: a guarded `run_python` tool

```python title="sandbox.py"
"""Run code in a separate process with a timeout, rlimits and a scrubbed environment.
Suitable for containing accidents. NOT a security boundary for hostile code."""
import os
import subprocess
import sys
import tempfile

try:
    import resource                           # Unix only
except ImportError:
    resource = None

def _limits(cpu_s, file_mb):
    def apply():
        resource.setrlimit(resource.RLIMIT_CPU, (cpu_s, cpu_s))
        resource.setrlimit(resource.RLIMIT_FSIZE, (file_mb * 2**20, file_mb * 2**20))
    return apply

def run_python(code, timeout_s=3, cpu_s=2, file_mb=5, max_output=2000):
    with tempfile.TemporaryDirectory() as work:
        path = os.path.join(work, "main.py")
        with open(path, "w") as f:
            f.write(code)
        try:
            p = subprocess.run(
                [sys.executable, "-I", "-S", path],   # isolated mode, no site-packages
                cwd=work, capture_output=True, text=True, timeout=timeout_s,
                env={"PATH": "/usr/bin:/bin"},        # no inherited secrets
                preexec_fn=_limits(cpu_s, file_mb) if resource else None)
        except subprocess.TimeoutExpired:
            return {"ok": False, "error": f"timed out after {timeout_s}s; simplify the "
                                          "computation or process less data"}
        out = (p.stdout + p.stderr)[-max_output:]
        return {"ok": p.returncode == 0, "exit_code": p.returncode, "output": out}
```

Test it against four kinds of code a model might produce:

```python title="sandbox_demo.py"
import os
from sandbox import run_python

os.environ["PAYMENTS_API_KEY"] = "sk-test-not-real"     # a secret in the parent process

cases = {
    "good": "import statistics\nprint(statistics.mean([3, 5, 8, 13]))",
    "bug": "print(10 / 0)",
    "hang": "while True:\n    pass",
    "peek at secrets": "import os\nprint(os.environ.get('PAYMENTS_API_KEY', 'not visible'))",
}
for name, code in cases.items():
    r = run_python(code)
    detail = r.get("output", r.get("error", "")).strip().splitlines()
    shown = detail[-1] if detail else f"(no output; exit code {r.get('exit_code')})"
    print(f"{name:<16} ok={r['ok']!s:<5} -> {shown}")
```

```text
good             ok=True  -> 7.25
bug              ok=False -> ZeroDivisionError: division by zero
hang             ok=False -> (no output; exit code -24)
peek at secrets  ok=True  -> not visible
```

The good code ran; the bug came back as an error message the model can read; the
infinite loop was stopped by the CPU limit before the 3-second timeout (a negative exit
code from `subprocess` means "killed by that signal number"; signal 24 is `SIGXCPU`, the
CPU-limit signal, on the machine that produced this output); and the
secret in the parent's environment was simply not there.

What this does **not** stop: the code could still read files your user can read (try
`open(os.path.expanduser("~/.ssh/config"))` in your own head, not in the tool), and it
could open network connections. That's the line between "containing accidents" and
"containing attackers".

## Designing the tool for the model

- **Return output and errors compactly**, with the tail of long output (errors are at
  the end).
- **Say what's available** in the description: Python version, which libraries,
  whether files persist between calls, no network.
- **Stateless or stateful?** Stateless (fresh process per call) is safer and simpler;
  stateful (a persistent kernel, like a notebook) is more capable for data analysis.
  If stateful, still isolate per user and per session and destroy at the end.
- **Put input files in, take output files out** explicitly, through the scratch
  directory — never mount real directories read-write.

## How It Actually Works

A separate process gets its own memory space, so a crash or an infinite loop in it
can't corrupt the agent. `timeout=` makes `subprocess.run` kill the child after a wall
clock deadline. `RLIMIT_CPU` asks the kernel to send the process a signal when it has
used that many CPU-seconds — which catches busy loops even if the wall-clock timeout
were long. `RLIMIT_FSIZE` makes writes beyond a size fail. `preexec_fn` runs in the
child between `fork` and `exec`, which is why the limits apply to the child only.
`env={...}` replaces rather than extends the environment, and `-I` stops Python from
reading `PYTHON*` variables and the user's site directory.

All of these rely on the same kernel and the same user account as the parent. That's
the fundamental limit: the child can do anything that *user* can do. Containers add
namespaces (their own view of the filesystem, processes and network) and syscall
filters; microVMs add a separate kernel. Each layer reduces what "anything that user can
do" means.

## Common mistakes

- **`exec()` in the agent's own process.** One bad line kills or compromises the agent.
- **Inheriting the environment**, including cloud credentials.
- **Mounting the project directory read-write** "for convenience".
- **Leaving network on** for a code tool that doesn't need it.
- **Believing a subprocess is a sandbox** for hostile code.
- **Unbounded output** flooding the model's context.

## Exercise

1. Add a memory limit with `RLIMIT_AS` and test code that allocates a very large list.
   Note that this limit behaves differently across operating systems — record what you
   observe on yours.
2. Make `run_python` accept a dict of input files, write them into the scratch
   directory, and return any new `.csv` files the code created.
3. Write the description you'd give the model for this tool, including every limit.
4. Research one container- or VM-based sandbox option and write a one-paragraph plan for
   moving `run_python` into it: what's mounted, what network is allowed, how it's cleaned
   up.
