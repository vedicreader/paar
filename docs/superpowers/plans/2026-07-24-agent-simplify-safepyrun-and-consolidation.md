# Agent simplification (safepyrun hybrid) + docs reconcile + nbs consolidation — Master Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Bring the codebase in line with the simplified single-mechanism model (overlay `AgentSession` is the one runtime; MCP is that runtime with restrictions): tighten `policy_inspectors`, make `scope='isolated'` a real allowlist sandbox via safepyrun, reconcile `index.ipynb` to the overlay-only story, remove stale kernel/share remnants, then run the 13→7 notebook consolidation + fastcore/literate-test pass.

**Architecture:** After commit `cd805c7` there is ONE agent mechanism — `AgentSession` (nbs/10_agent.ipynb): an overlay on the owner's live namespace, guarded per-cell by an AST restriction list (`policy_inspectors`). The `/kernel` process-isolated runtime and `share`/`push` were removed. This plan keeps the overlay + AST policy as the owner-protection core (safepyrun cannot express per-object owner protection), and uses safepyrun ONLY for the throwaway `isolated` scope, where liveness isn't needed.

**Tech Stack:** nbdev, fastcore, safepyrun (new dep), mcp (FastMCP), IPython, ast.

## Global Constraints

- Python `>=3.11`; build backend **hatchling** — never edit `MANIFEST.in`; add deps via `uv add` (writes `pyproject.toml` + `uv.lock`).
- Use `nbdev-prepare` (hyphen), never `nbdev_prepare`.
- Entry-point module names unchanged where they are entry points: `paar.tui`, `paar.fasthtml`, `paar.mcp` (Phase 2 preserves these via `#| default_exp`).
- The AST policy is "steering for well-behaved agents, NOT a security sandbox" — do not oversell it in docs.
- Preserve test coverage: net `test_eq`/`assert` count per touched notebook must not drop.
- fastcore idioms where they genuinely shorten; short one-line docstrings.
- **Real tests only — no mocks, monkeypatching, or fake shells.** Replace every fake with a real object or real in-process fixture: `NsClient(dict)` for namespaces; a real `IPython.core.interactiveshell.InteractiveShell.instance()` where a live shell is genuinely needed (exec/completion/events); `starlette.testclient.TestClient(app)` for routes (already used); Textual `Pilot` for the TUI; a real in-process paar server for mcp HTTP round-trips. This governs both the tests touched in Phase 1 and the literate-test conversion in Phase 2. Banned patterns to grep for and eliminate: `get_ipython = lambda`, `class _IP`/`_Shell`/`_Sh`/`_CapShell`, `Mock`, `monkeypatch`, hard-coded dead URLs like `http://127.0.0.1:9`.

### The nbdev cycle ("run the cycle")

`uv run nbdev-prepare` = export cells → `paar/*.py`, run every notebook as tests (raised exception = failure), clean. Success = exit 0, no failures, regenerated modules. Fix the named cell, re-run; do not advance until green.

### This plan supersedes

- `docs/superpowers/plans/2026-07-23-mcp-kernel-switch-and-index.md` — **obsolete** (its `/kernel` target no longer exists). Task 6 retires it.
- `docs/superpowers/plans/2026-07-23-nbs-consolidation-fastcore.md` — **still valid**; Phase 2 (Task 7) runs it, with the amendments noted there (safepyrun/policy carry-through **and** the real-tests / Bridge-DI constraint).

---

# Phase 1 — content changes on the CURRENT 13-notebook layout

Done first so edits target stable notebook names (`10_agent`, `index`, `08_tui`, `11_mcp`); Phase 2 consolidation then absorbs them.

## Task 0: Spike — verify safepyrun capabilities (gates Task 2)

**Files:** none committed (throwaway exploration). Use the clikernel session (`! clikernel`) or `uv run python`.

**Interfaces:** Produces two facts that determine Task 2's exact shape: (a) how to call safepyrun synchronously from inside a running event loop (Jupyter), (b) whether owner variables can be seeded into the sandbox namespace so the agent can still *read* owner state under `isolated`.

- [ ] **Step 1: Install and probe the API**

```bash
uv add safepyrun
uv run python -c "import safepyrun, inspect; from safepyrun import RunPython; print([m for m in dir(RunPython) if not m.startswith('__')])"
```
Expected: prints the public methods (look for `allow`, `allow_write`, and any namespace/seed/`__call__`). Record the list.

- [ ] **Step 2: Confirm sync-from-async wrapping works**

Run this and confirm it prints `4` both outside and inside a running loop:

```python
import asyncio, concurrent.futures
from safepyrun import RunPython
def run_sync(coro):
    try: asyncio.get_running_loop()
    except RuntimeError: return asyncio.run(coro)
    with concurrent.futures.ThreadPoolExecutor(1) as ex:
        return ex.submit(lambda: asyncio.run(coro)).result()
r = RunPython()
print(run_sync(r("x_ = 2+2\nprint(x_)")).stdout)
```
Expected: `4` (and `\n`). If a fresh `RunPython()` is needed per isolated call to avoid state bleed, note it.

- [ ] **Step 3: Determine owner-var seeding**

Try to make an owner value readable inside the sandbox (try each until one works; record which):
```python
r = RunPython()
# (a) does RunPython accept an initial namespace / does .allow take non-callables?
# (b) can we pre-run a cell that binds names, then read them?
run_sync(r("owner_df_ = None"))   # placeholder to test the _-export convention
```
Record the verdict: **CAN seed** owner vars → `isolated` keeps read access (seed the merged view as `name_`); **CANNOT seed** → `isolated` becomes a pure sandbox with NO owner reads (a behavior change to document, and to confirm acceptable in Task 2).

- [ ] **Step 4: Write findings into Task 2's notes**

Update Task 2 Steps below with the confirmed `run_sync` helper and the seeding decision before implementing. No commit.

---

## Task 1: Simplify `policy_inspectors` (nbs/10_agent.ipynb, cell `10-c7`)

**Files:**
- Modify: `nbs/10_agent.ipynb` cell `10-c7` (the restriction list) and, if needed, the test cell `10-c11`.
- Generated: `paar/agent.py`.

**Interfaces:**
- Produces (unchanged public API): `policy_inspectors(owner_names)` returning `[guard]`; module names `_MUTATORS`, `_BLOCKED_CALLS`, `_DESTRUCTIVE`, `_DESTRUCTIVE_ATTRS`, `_root`, `_block`, `_check_del`, `_check_assign`, `_check_call` may be reorganized but keep the same blocking behavior (tests in `10-c11` are the contract).

- [ ] **Step 1: Run the existing policy tests to capture the behavior contract**

Run: `uv run nbdev-prepare`
Expected: green. The cells `test_del_and_mutation_blocked` and `test_escapes_blocked` in `10-c11` define exactly what must still be blocked — treat them as the spec. Do NOT weaken them.

- [ ] **Step 2: Apply low-risk simplifications only**

Targeted, behavior-preserving edits (the code is already tight — do not restructure for its own sake):
- Collapse the node-type dispatch in `guard` into a single walk with a small dispatch, using `fastcore.basics` only where it shortens. Keep the if/elif if a dict-dispatch would be less clear.
- Fold the destructive-escape data so `_check_call`'s escape branches read from one structure where possible (e.g. keep `_DESTRUCTIVE` as `(module, attr)` pairs; keep `_DESTRUCTIVE_ATTRS`/`_BLOCKED_CALLS` as-is).
- Use `L(...).filter(...)`/`first(...)` in place of manual loops in `_check_del`/`_check_assign` only where it stays readable.
- Keep `_root`, `_block`, and the `owner`-awareness exactly as-is (that logic is the point).

- [ ] **Step 3: Run the cycle**

Run: `uv run nbdev-prepare`
Expected: green, all `10-c11`/`10-c12`/`10-c13` assertions pass unchanged. If any block test regresses, revert that simplification.

- [ ] **Step 4: Commit**

```bash
git add nbs/10_agent.ipynb paar/agent.py paar/_modidx.py
git commit -m "refactor(agent): simplify policy_inspectors (behavior-preserving)

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 2: Hybrid safepyrun for `scope='isolated'` (nbs/10_agent.ipynb)

**Files:**
- Modify: `nbs/10_agent.ipynb` cell `10-c4` (imports + a `_run_sync` helper), cell `10-c9` (`AgentSession.run` isolated branch), test cell `10-c11` (`test_isolated_discards` + a new sandbox-escape test), `pyproject.toml`/`uv.lock` (dep, via Task 0 `uv add`).
- Generated: `paar/agent.py`.

**Interfaces:**
- Consumes: `safepyrun.RunPython`; the Task 0 `run_sync` helper + seeding verdict.
- Produces: `AgentSession.run(code, scope='overlay')` unchanged for `overlay`; for `scope='isolated'` the code runs in a fresh safepyrun sandbox (allowlist), writes are discarded (as today), and destructive escapes are blocked by safepyrun's allowlist in addition to the AST policy. `ExecResult` shape unchanged.

- [ ] **Step 1: Add the import + sync helper (cell `10-c4`)**

```python
#| export
import asyncio, concurrent.futures
try: from safepyrun import RunPython
except Exception: RunPython = None

def _run_sync(coro):
    "Run an async coroutine to completion whether or not an event loop is already running (Jupyter-safe)."
    try: asyncio.get_running_loop()
    except RuntimeError: return asyncio.run(coro)
    with concurrent.futures.ThreadPoolExecutor(1) as ex:
        return ex.submit(lambda: asyncio.run(coro)).result()
```

- [ ] **Step 2: Route the isolated branch through safepyrun (cell `10-c9`, `run`)**

Replace the isolated execution path. The AST policy still runs first (unchanged). Then, for `isolated`:

```python
    if scope == 'isolated' and RunPython is not None:
        sb = RunPython()
        # SEEDING per Task 0 verdict: if seeding is supported, seed the merged read view
        # as sandbox names so the agent can still READ owner state; else run bare (no owner reads).
        res = _run_sync(sb(code))
        self._log(code, 'scope: isolated')
        out = note + (res.stdout or '')
        if res.stderr: return ExecResult(ok=False, stdout=out, error=res.stderr.strip())
        return ExecResult(ok=True, stdout=out, result=None if res.result is None else _flat_result(res.result))
```
Keep the current `_exec_capture`-based path as the fallback when `RunPython is None` (safepyrun not installed) so the notebook still runs without the optional dep. Overlay scope is untouched.

- [ ] **Step 3: Decide + document the seeding behavior**

Apply Task 0's verdict:
- **CAN seed:** seed owner vars before running (`isolated` keeps read access); note it in the `run` docstring.
- **CANNOT seed:** update the `run` docstring and cell `10-c8`/markdown to say `isolated` is now a *sandboxed scratch scope with no owner reads* (overlay is the scope for reading owner state). Confirm this is the intended trade-off before finalizing.

- [ ] **Step 4: Update/extend tests (cell `10-c11`)**

Keep `test_isolated_discards` (writes must not persist). Add a sandbox test proving safepyrun blocks an escape in isolated scope even if the AST policy were bypassed:

```python
def test_isolated_is_sandboxed():
    r = s.run('import os; os.system("echo hi")', scope='isolated')
    assert not r.ok   # safepyrun allowlist refuses os.system
```
If seeding is disabled, adjust any isolated test that assumed owner reads.

- [ ] **Step 5: Run the cycle**

Run: `uv run nbdev-prepare`
Expected: green. `paar/agent.py` imports safepyrun guardedly; isolated path uses it.

- [ ] **Step 6: Add the dep as an extra (optional install)**

Since safepyrun is only needed for isolated sandboxing, prefer an optional extra over a hard dep. In `pyproject.toml`, add safepyrun under `[project.optional-dependencies]` (e.g. an existing/new `agent` extra) rather than core `dependencies`, matching the guarded import. Re-run `uv lock` if needed.

- [ ] **Step 7: Commit**

```bash
git add nbs/10_agent.ipynb paar/agent.py paar/_modidx.py pyproject.toml uv.lock
git commit -m "feat(agent): sandbox scope='isolated' via safepyrun (overlay unchanged)

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 2A: `Bridge` takes a client (dependency injection) + remove the namespace global

**Files:**
- Modify: `nbs/04_bridge.ipynb` — the export cell (add `IPythonClient`, `NsClient`, rewrite `Bridge`; delete `_ns`/`_hidden`/`_ns_override`/`set_namespace`) and all its test cells (rewrite real via `NsClient`).
- Modify: `nbs/05_fasthtml.ipynb` — drop `set_namespace` import; module `bridge = Bridge()`; `serve()`/`inspector()` build `Bridge(NsClient(ns))` for headless instead of `set_namespace(ns)`; convert route tests to inject `fasthtml.bridge = Bridge(NsClient({...}))` (remove `_IP`/`get_ipython` patch).
- Modify: `nbs/10_agent.ipynb` — `AgentSession` default owner switches from `from .bridge import _ns` to `IPythonClient().ns`.
- Generated: `paar/bridge.py`, `paar/fasthtml.py`, `paar/agent.py`.

**Interfaces:**
- Produces (module `paar.bridge`): `IPythonClient`, `NsClient`, `Bridge`, `on_change`. **Removed:** `set_namespace`, `_ns`, `_hidden`, `_ns_override`.
- Client protocol: `ns() -> dict`, `hidden() -> frozenset`, `set_value(accessor, expr) -> None`.
- `Bridge(client=None)` — defaults to `IPythonClient()`. Methods `snapshot/view/expand/grid/set_value` delegate to `self.client`.
- Consumes: `snapshot`, `profile_view`, `expand`, `grid_page` (snapshot), `set_value` (runtime), `get_ipython` (IPython).

- [ ] **Step 1: Write the failing real test first (cell in `04_bridge`)**

```python
#| hide
import numpy as np
from fastcore.test import test_eq
b = Bridge(NsClient({'a': 1, 'arr': np.arange(6).reshape(2,3)}, hidden={'_hidden'}))
test_eq([r.name for r in b.snapshot()], ['a', 'arr'])          # real namespace, no mock
test_eq(b.grid(('arr',))['nrows'], 2)
test_eq([k.name for k in b.expand(('a',))], [])               # scalar has no children
nb = NsClient({'x': 1}); bb = Bridge(nb); bb.set_value(('x',), '7'); test_eq(nb.ns()['x'], 7)
```

- [ ] **Step 2: Run it — expect failure**

Run: `uv run nbdev-prepare`. Expected: FAIL (`NsClient`/`Bridge(client)` not defined yet).

- [ ] **Step 3: Implement the clients + Bridge (rewrite the export cell)**

```python
#| export
try: from IPython import get_ipython
except Exception:
    def get_ipython(): return None

class IPythonClient:
    "Default namespace source: the live IPython kernel (empty when not in IPython)."
    def ns(self):     ip = get_ipython(); return ip.user_ns if ip else {}
    def hidden(self): ip = get_ipython(); return frozenset(getattr(ip, 'user_ns_hidden', ())) if ip else frozenset()
    def set_value(self, accessor, expr): return set_value(accessor, expr)

class NsClient:
    "A plain-dict namespace source for headless serve() and real (non-mock) tests."
    def __init__(self, ns, hidden=frozenset()): self.ns_, self.hidden_ = ns, frozenset(hidden)
    def ns(self):     return self.ns_
    def hidden(self): return self.hidden_
    def set_value(self, accessor, expr): return set_value(accessor, expr, ns=self.ns_)

class Bridge:
    "Frontend-agnostic access to a namespace via an injected client (defaults to the live IPython kernel)."
    def __init__(self, client=None): self.client = client or IPythonClient()
    def snapshot(self, name=None, typ=None): return snapshot(self.client.ns(), self.client.hidden(), name, typ)
    def view(self, profile='standard'):      return profile_view(self.client.ns(), self.client.hidden(), profile)
    def expand(self, accessor, offset=0):    return expand(self.client.ns(), accessor, offset)
    def grid(self, accessor, roff=0, coff=0, rows=50, cols=50): return grid_page(self.client.ns(), accessor, roff, coff, rows, cols)
    def set_value(self, accessor, expr):     return self.client.set_value(accessor, expr)

def on_change(cb):
    "Register cb() to fire after each cell execution; returns False outside IPython."
    ip = get_ipython()
    if not ip: return False
    ip.events.register('post_run_cell', lambda res: cb()); return True
```

- [ ] **Step 4: Make `runtime.set_value` accept an explicit ns**

In `nbs/08_runtime.ipynb`, extend `set_value(accessor, expr, ns=None)` to operate on the given dict (default: the IPython user_ns) so `NsClient.set_value` reuses it. Keep the IPython default path identical. Add a real test using a plain dict (no shell).

- [ ] **Step 5: Rewire fasthtml + agent**

`nbs/05_fasthtml.ipynb`: remove `set_namespace` from the import; keep `bridge = Bridge()` for the in-kernel default; in `serve()`/`inspector()` replace `set_namespace(ns)` with `bridge = Bridge(NsClient(ns))` (rebind the module global). `nbs/10_agent.ipynb`: change the default owner to `IPythonClient().ns` (import `IPythonClient` from `.bridge`).

- [ ] **Step 6: Convert fasthtml route tests to real namespaces**

Replace each `B.get_ipython = lambda: _IP({...})` with `import paar.fasthtml as F; F.bridge = Bridge(NsClient({...}))` before the `TestClient(app)` call; delete the `_IP` fake class. Routes now serve a real namespace over the real test client.

- [ ] **Step 7: Run the cycle**

Run: `uv run nbdev-prepare`. Expected: green; `paar/bridge.py` has `IPythonClient`/`NsClient`/`Bridge(client)` and no `set_namespace`.

- [ ] **Step 8: Verify the global is gone**

```bash
grep -rn "set_namespace\|_ns_override" nbs/*.ipynb paar/*.py
```
Expected: no matches (except perhaps a removed-in-this-task mention). Fix any straggler (e.g. index docs — handled in Task 3).

- [ ] **Step 9: Commit**

```bash
git add nbs/04_bridge.ipynb nbs/05_fasthtml.ipynb nbs/08_runtime.ipynb nbs/10_agent.ipynb paar/bridge.py paar/fasthtml.py paar/runtime.py paar/agent.py paar/_modidx.py
git commit -m "refactor(bridge): inject namespace client (Bridge(client)); drop set_namespace global; real tests

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 2B: Real-test sweep for runtime and remaining fakes

**Files:** `nbs/08_runtime.ipynb` (replace `_Shell`/`_CapShell`), plus any leftover fakes surfaced by the audit in `nbs/01_reprs.ipynb`, `nbs/02_resolvers.ipynb`, `nbs/08_tui.ipynb`.

**Interfaces:** tests only; no public API change.

- [ ] **Step 1: Replace runtime's fake shells with a real InteractiveShell**

In `nbs/08_runtime.ipynb`, delete `class _Shell`/`_CapShell` and the `R.get_ipython = lambda: sh` patches. Use a real shell:

```python
#| hide
from IPython.core.interactiveshell import InteractiveShell
import paar.runtime as R
_ip = InteractiveShell.instance()          # a real in-process shell
R.get_ipython = InteractiveShell.instance  # real, not a lambda-mock
def _fresh():
    _ip.user_ns.clear(); _ip.user_ns.update(_ip.user_ns_hidden or {}); return _ip
```
Rewrite each exec/complete/log/set_value test to seed `_ip.user_ns[...]` and assert against real `run`/`complete`/`set_value` results. `run('1+1')`, completion, and capture now exercise the real IPython machinery.

- [ ] **Step 2: Audit + convert remaining fakes**

```bash
grep -rnE "get_ipython = lambda|class _IP|_Shell|_Sh\b|_CapShell|Mock|monkeypatch|http://127.0.0.1:9" nbs/*.ipynb
```
Convert each remaining hit to a real object/fixture per the Global Constraint. For `nbs/08_tui.ipynb`, prefer a Textual `Pilot` (`async with app.run_test() as pilot`) over stubbed client methods where a stub still remains after Task 4.

- [ ] **Step 3: Run the cycle**

Run: `uv run nbdev-prepare`. Expected: green with real fixtures. If `InteractiveShell.instance()` state bleeds between tests, clear `user_ns` per test.

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "test: replace fake shells/mocks with real IPython + fixtures across notebooks

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 3: Reconcile `index.ipynb` to the single-mechanism story

**Files:** Modify `nbs/index.ipynb` (`idx-overview`, `idx-agents`/`idx-agents-modes`, `idx-mcp`, `idx-security`, and the `idx-clikernel` section); regenerates `README.md`.

**Interfaces:** docs only; keep `#| eval: false` demo cells runnable.

- [ ] **Step 1: Remove any two-tier / kernel / share prose**

Search and purge references to the removed features:
```bash
grep -niE "isolated kernel|/kernel|AgentKernel|process-isolated|share\(|push\(|two tier|second tier|set_namespace" nbs/index.ipynb
```
Delete/rewrite each hit. The story is now ONE mechanism: the overlay `AgentSession`. Any `set_namespace(...)` in prose/examples becomes `Bridge(NsClient(ns))` (Task 2A removed the global).

- [ ] **Step 2: Rewrite the agent section around the overlay only**

`idx-agents`/`idx-agents-modes`: describe the single contract — agent reads the owner's live variables, its writes land in a persistent overlay `layer`, owner names can be shadowed but never rebound/deleted (structural), and an AST restriction list blocks in-place mutation of owner objects + destructive escapes. Mention `scope='isolated'` now runs in a safepyrun allowlist sandbox for throwaway evals (per Task 2's seeding verdict). Keep the `AgentSession` demo cells.

- [ ] **Step 3: Update the MCP section (`idx-mcp`)**

`paar-mcp` exposes the overlay agent surface (`/agent/api`) as tools `list_vars/expand/grid/execute/session_info`. No kernel, no push. `--env`/`--url` resolution unchanged.

- [ ] **Step 4: Reconcile the security model (`idx-security`)**

Restore the honest single-tier statement: these are guardrails for well-behaved agents, not a sandbox; the overlay gives a structural name-binding guarantee; the AST list is evadable by determined code; `scope='isolated'` adds a safepyrun allowlist sandbox for throwaway code but the overlay path shares the owner's interpreter. Remove the now-false "a separate process would defeat the point" framing OR keep it as the rationale for *why* there's no isolated-process tier.

- [ ] **Step 5: Run the cycle + verify README**

```bash
uv run nbdev-prepare
grep -niE "/kernel|AgentKernel|push\(|two tier" README.md   # expect: no matches
```
Expected: nbdev green; no stale kernel/share terms in README.

- [ ] **Step 6: Commit**

```bash
git add nbs/index.ipynb README.md _docs/README.md
git commit -m "docs(index): reconcile to single overlay mechanism; note safepyrun isolated scope

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 4: Clean up stale kernel/share remnants

**Files:** `nbs/08_tui.ipynb` (test-mock `share` stubs), plus a tree-wide sweep.

- [ ] **Step 1: Tree-wide sweep for dead references**

```bash
grep -rniE "AgentKernel|/kernel/api|guarded_run|agent_kernel|def share|share_all|push_all|_shared_ns" nbs/*.ipynb paar/*.py
```
Every hit must be either a Textual API (`push_screen*` — leave) or dead code (remove).

- [ ] **Step 2: Remove the no-op share stubs in the tui test mock**

In `nbs/08_tui.ipynb` remove the `async def share(self, name): return 'ok'` / `async def share_all(self): return 'ok'` mock methods and any test line that calls them (the `p`/`a` share keybindings were removed with the feature). Leave `push_screen*` untouched.

- [ ] **Step 3: Run the cycle**

Run: `uv run nbdev-prepare`
Expected: green.

- [ ] **Step 4: Commit**

```bash
git add nbs/08_tui.ipynb paar/tui.py paar/_modidx.py
git commit -m "chore(tui): drop dead share/push test stubs from removed feature

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 5: Verify mcp doc wording matches overlay reality

**Files:** `nbs/11_mcp.ipynb` (module docstring + tool descriptions).

- [ ] **Step 1: Confirm mcp already targets `/agent/api`**

```bash
grep -nE "/agent/api|/kernel" nbs/11_mcp.ipynb
```
Expected: only `/agent/api` (kernel gone). The tools are correct as-is; this task is wording only.

- [ ] **Step 2: Tighten `execute`/INSTRUCTIONS wording if needed**

Ensure `execute`'s description still contains `'blocked'` (the test asserts it) and the overlay contract is accurately described. No path changes.

- [ ] **Step 3: Run the cycle + commit (skip if no change)**

```bash
uv run nbdev-prepare
git add nbs/11_mcp.ipynb paar/mcp.py paar/_modidx.py && git commit -m "docs(mcp): align tool wording with overlay surface" || echo "no change needed"
```

---

## Task 6: Retire the obsolete mcp-kernel plan doc

- [ ] **Step 1: Remove it**

```bash
git rm docs/superpowers/plans/2026-07-23-mcp-kernel-switch-and-index.md
git commit -m "docs: retire obsolete mcp-kernel-switch plan (kernel removed on main)

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

# Phase 2 — notebook consolidation (13 → 7) + fastcore + literate tests

## Task 7: Execute the existing consolidation plan

**Files:** per `docs/superpowers/plans/2026-07-23-nbs-consolidation-fastcore.md` (still valid — it targets exactly the current 13 notebooks; kernel was never in it).

- [ ] **Step 1: Re-baseline assertion counts**

The Phase 1 edits changed counts in `10_agent`, `08_tui`, `11_mcp`, `index`. Re-run the consolidation plan's Task 0 baseline `grep -c` and update its per-notebook minimums before merging.

- [ ] **Step 2: Run the consolidation plan Tasks 1–9**

Follow that plan as written (00_core = core+reprs+resolvers+providers; 01_snapshot = grid+snapshot; 02_runtime = runtime+bridge+registry; 03_agent; 04_fasthtml; 05_tui; 06_mcp), `nbdev-prepare`-gated per notebook.

- [ ] **Step 3: Amendments carried into the merge**

- **`03_agent`** now also carries: the guarded `safepyrun` import + `_run_sync` helper, the simplified `policy_inspectors`, and the safepyrun `isolated` branch. Carry all through unchanged; keep the `store_attr`/`first`/`L` fastcore pass. The default owner is now `IPythonClient().ns` (Task 2A) — after bridge merges into runtime, import `IPythonClient` from `.runtime`.
- **`02_runtime`** (absorbs bridge) now contains `IPythonClient`, `NsClient`, `Bridge(client)`, `on_change` — no `set_namespace`/`_ns`/`_hidden`. Update the consolidation plan's runtime task interfaces accordingly.
- **Literate tests must be real** (Global Constraint): the consolidation plan's "literate test conversion" steps use `NsClient`, a real `InteractiveShell.instance()`, `TestClient(app)`, and Textual `Pilot` — never re-introduce a fake shell while merging.

- [ ] **Step 4: Final verification**

Run the consolidation plan's Task 9 verification (7 notebooks + index, entry points import, no stale generated modules, assertion totals held).

---

## Self-Review Notes

- **Coverage vs request:** simplify policy (T1), safepyrun hybrid for isolated (T0 spike + T2), Bridge client DI + drop `set_namespace` global (T2A), full real-tests / ban-mocks sweep (T2A + T2B + Global Constraint, carried into T7), index reconcile (T3), stale-remnant cleanup (T4, +T6 retire obsolete plan), mcp wording (T5), consolidation still done (T7). All selected scopes mapped.
- **Bridge DI ordering:** T2A (bridge takes a client, global removed) precedes fasthtml test conversion and Phase 2, since consolidation merges bridge→runtime and inherits the new client API + real tests.
- **Ordering rationale:** content edits (Phase 1) on stable notebook names first; consolidation (Phase 2) last so it absorbs them once, avoiding rework.
- **Risk gates:** Task 0 spike de-risks the two safepyrun unknowns (sync wrap + var seeding) before any commit; guarded `try: import safepyrun` keeps the notebook runnable without the optional dep; existing policy tests are the contract for T1; assertion-count guards in T7.
- **Honesty:** safepyrun is NOT presented as replacing owner-object protection (it can't); it only sandboxes the throwaway `isolated` scope. Docs (T3) keep the "not a security sandbox" framing for the overlay path.
```
