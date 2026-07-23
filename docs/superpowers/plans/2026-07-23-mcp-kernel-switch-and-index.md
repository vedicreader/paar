# paar-mcp → isolated kernel + index refresh — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Repoint `paar-mcp`'s tools from the in-process overlay (`/agent/api/*`) to the process-isolated agent kernel (`/kernel/api/*`), and update `index.ipynb` to document the new isolated-kernel tier and reconcile the security model.

**Architecture:** nbdev project — edit notebooks in `nbs/`, `paar/*.py` is generated. Two agent surfaces exist on every paar server: the legacy in-process overlay (`/agent/api/*`, kept for other devs) and the new process-isolated kernel (`/kernel/api/*`, backed by `paar.kernel.AgentKernel` — a separate-pid clikernel worker). This change makes `paar-mcp` a fully-isolated client: exec runs in the worker, reads come from the read-only owner snapshot.

**Tech Stack:** nbdev, fastcore, mcp (FastMCP), httpx, paar.kernel, clikernel.

## Global Constraints

- Python `>=3.11`; build backend **hatchling** — never edit `MANIFEST.in`.
- Use `nbdev-prepare` (hyphen), never `nbdev_prepare`.
- Entry-point module names unchanged: `paar.mcp` (target of `paar-mcp`).
- The five MCP tool names must stay: `list_vars`, `expand`, `grid`, `execute`, `session_info` (the existing test asserts this set).
- `execute`'s tool description must still contain the substring `'blocked'` (existing test asserts it; `guarded_run` returns `blocked: …` on policy refusal, so this stays true).
- fastcore idioms where they shorten; short one-line docstrings.

### The nbdev notebook cycle ("run the cycle")

For each task: `uv run nbdev-prepare` runs `nbdev_export` (cells → `paar/*.py`), `nbdev_test` (executes every notebook; a raised exception = failing test), and `nbdev_clean`. Success = exit 0, no failures, regenerated `paar/mcp.py` + `paar/_modidx.py`. Fix the named cell and re-run on failure; do not advance until green.

---

## Key surface differences (reference for both tasks)

Old overlay (`/agent/api/*`), what mcp uses today:
- `GET /agent/api/rows?profile=` → merged view (owner ns + agent overlay layer), includes `agent_names`.
- `GET /agent/api/expand?accessor=&offset=` ; `GET /agent/api/grid?accessor=&roff=&coff=`
- `POST /agent/api/exec` with `code`, `scope` (`overlay`/`isolated`) → `{ok,stdout,error,scope,result}`
- `GET /agent/api/info` → `{env, mode, policy, agent_names}`

New isolated kernel (`/kernel/api/*`), the target:
- `GET /kernel/api/rows?profile=` → **read-only** snapshot of ear-marked owner vars (via `share()`); returns `{profile,profiles,groups,shared}` (no `agent_names`).
- `GET /kernel/api/expand?accessor=&offset=` ; `GET /kernel/api/grid?accessor=&roff=&coff=` → read-only, ear-marked owner values only.
- `POST /kernel/api/exec` with `code` **only** (no `scope`) → `{ok,stdout,error}`. Runs in a separate-pid clikernel worker; owner objects unreachable by construction.
- `GET /kernel/api/info` → `{alive, name, shared, policy}`.

Semantic change to communicate in `INSTRUCTIONS`: agent code no longer runs in the owner's interpreter. Names the agent creates still persist across `execute` calls (the worker is one persistent clikernel), but the agent **cannot read owner objects directly in exec** — it sees them read-only via `list_vars`/`expand`/`grid`, and can only compute on owner data the owner has `push()`-ed as copies.

---

## Task 1: Repoint `paar-mcp` to the isolated kernel (`nbs/11_mcp.ipynb`)

**Files:**
- Modify: `nbs/11_mcp.ipynb` — module docstring cell, `#11-c3` (INSTRUCTIONS + PaarClient docstring), `#11-c4` (build_mcp tool bodies + `main` description), and the test cell that exercises tool behavior.
- Generated: `paar/mcp.py`.

**Interfaces:**
- Consumes: paar server routes `/kernel/api/{rows,expand,grid,exec,info}` (already exist in `paar/fasthtml.py`).
- Produces (module `paar.mcp`, `__all__` unchanged): `INSTRUCTIONS`, `PaarClient`, `build_mcp`, `main`. Tool names unchanged: `list_vars`, `expand`, `grid`, `execute`, `session_info`.

- [ ] **Step 1: Rewrite `INSTRUCTIONS` for the isolated model**

In cell `#11-c3`, replace the `INSTRUCTIONS` string. New content (exact text to use):

```python
INSTRUCTIONS = (
    "You are connected to a live Python session owned by the user, through paar's "
    "process-isolated agent kernel. Your code runs in a SEPARATE process from the user's "
    "interpreter, so you can never mutate, rebind, or delete the user's variables -- they "
    "are unreachable by construction, and destructive file/shell escapes and exec/eval are "
    "blocked. A refused cell returns an error starting with 'blocked:'. "
    "Names you create persist across execute calls (your kernel is one long-lived worker). "
    "You can SEE the user's variables read-only via list_vars/expand/grid, but you cannot "
    "compute on the actual objects unless the user has shared copies into your kernel -- if "
    "you need to work on one of their objects, ask them to push it. "
    "Start with list_vars to see what's shared, expand/grid to drill in cheaply, and execute "
    "for your own computation.")
```

- [ ] **Step 2: Repoint `PaarClient` docstring**

In `#11-c3`, change the `PaarClient` docstring from "…/agent/api surface…" to "Thin HTTP client for a paar server's /kernel/api surface (process-isolated agent kernel); the target resolves per call, so late starts and restarts of the paar server are fine." Leave `_base`, `_req`, `get`, `post` logic unchanged (path-agnostic).

- [ ] **Step 3: Repoint the five tools in `build_mcp` (`#11-c4`)**

Change each tool body's path to `/kernel/api/*` and drop the overlay `scope` from `execute`:

```python
    @m.tool(structured_output=False)
    def list_vars(profile: str = 'standard') -> str:
        "The user's variables that are shared with you, read-only (rendered copies). profile: 'minimal' | 'standard' | 'full'. Every node carries an 'accessor' list -- pass it to expand/grid to look around cheaply instead of using execute."
        return c.get('/kernel/api/rows', profile=profile)

    @m.tool(structured_output=False)
    def expand(accessor: str, offset: int = 0) -> str:
        '''One level of children of a shared value at accessor -- a JSON list from list_vars/expand, e.g. '["data", 2]'. Read-only. Page with a trailing load-more node's more_offset as offset.'''
        return c.get('/kernel/api/expand', accessor=accessor, offset=offset)

    @m.tool(structured_output=False)
    def grid(accessor: str, roff: int = 0, coff: int = 0) -> str:
        "A paged table window of a shared DataFrame/ndarray at accessor (a JSON list); roff/coff scroll rows/columns. Read-only, far cheaper than printing with execute."
        return c.get('/kernel/api/grid', accessor=accessor, roff=roff, coff=coff)

    @m.tool(structured_output=False)
    def execute(code: str) -> str:
        "Run Python in your process-isolated kernel under paar's restriction policy. Your kernel is persistent -- names you create carry across calls. You cannot reach the user's objects here (separate process); an error starting with 'blocked:' means the cell hit a restricted escape. Use list_vars/expand/grid to read the user's shared state."
        return c.post('/kernel/api/exec', code=code)

    @m.tool(structured_output=False)
    def session_info() -> str:
        "Your kernel's status: whether it's alive, its env label, which owner variables are shared with you, and the policy in force."
        return c.get('/kernel/api/info')
```

- [ ] **Step 4: Update the module docstring + `main` description**

In the module docstring cell (top), replace "restricted `/agent/api` surface … overlay to create its own" with a description of the isolated kernel: "bridging an agent (Claude Code, Codex, …) to a live paar session's **process-isolated agent kernel** (`/kernel/api`). Agent code runs in a separate process; owner objects are unreachable, and the agent reads owner state read-only. Requires the `mcp` extra." In `main` (`#11-c4`), change the `description=` string's "restricted /agent/api surface" → "process-isolated agent kernel (/kernel/api)". Keep `--url`/`--env`/`--name` args unchanged.

- [ ] **Step 5: Update the behavior test cell**

The existing test at nb lines ~205-209 asserts the tool set and that `execute`'s description contains `'blocked'`. Keep it. Its exact assertions must still pass:

```python
#| hide
m = build_mcp(url='http://127.0.0.1:9')
tools = asyncio.run(m.list_tools())   # keep whatever call the current cell uses
assert {t.name for t in tools} == {'list_vars', 'expand', 'grid', 'execute', 'session_info'}
assert all(t.description for t in tools)
assert 'blocked' in next(t for t in tools if t.name == 'execute').description
```

Add one assertion that `execute` no longer advertises a scope param and that instructions mention isolation:

```python
    assert 'scope' not in next(t for t in tools if t.name == 'execute').description.lower()
    assert 'isolat' in INSTRUCTIONS.lower()
```

- [ ] **Step 6: Run the cycle**

Run: `uv run nbdev-prepare`
Expected: exit 0; `paar/mcp.py` shows all five tools posting/getting `/kernel/api/*`; `execute` takes only `code`.

- [ ] **Step 7: Smoke-check the built server**

```bash
uv run python -c "from paar.mcp import build_mcp, INSTRUCTIONS; m=build_mcp(url='http://127.0.0.1:9'); print('isolat' in INSTRUCTIONS.lower(), m.name)"
```
Expected: `True paar`.

- [ ] **Step 8: Commit**

```bash
git add nbs/11_mcp.ipynb paar/mcp.py paar/_modidx.py
git commit -m "feat(mcp): route paar-mcp tools to the process-isolated /kernel/api surface

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 2: Update `index.ipynb` — document the isolated kernel tier + reconcile security

**Files:**
- Modify: `nbs/index.ipynb` — the `idx-overview`, `idx-agents`/`idx-agents-modes`, `idx-mcp`, and `idx-security` markdown cells; add a new cell for the isolated kernel.

**Interfaces:** none (docs only). Keep all `#| eval: false` demo cells runnable/unchanged unless a demo is added.

- [ ] **Step 1: Add an "Isolated agent kernel" subsection after the overlay section**

Insert a new markdown cell after `idx-agents-modes` (before `idx-mcp`). Content covers:
- `paar.kernel.AgentKernel` — a persistent clikernel worker paar supervises as a **separate process**; agent code runs there, owner objects unreachable by construction.
- `serve()`/`inspector()` accept `agent_kernel=True` (default) which auto-starts the worker and gives it its own paar env (`<env>-agent`) so the owner can watch it live.
- Read-only view + push: `share(*names)` ear-marks which owner vars the agent may see read-only (`/kernel/api/rows`); `push(name)`/`push_all()` copy owner objects (pickle copies — originals never change) into the worker so the agent can compute on them; `unshare(name)` removes a pushed copy.
- New endpoints `/kernel/api/{exec,rows,expand,grid,info}` and the `paar-exec` CLI:
```sh
paar-exec --env myrepo "import numpy as np; np.arange(5).sum()"   # runs in the isolated worker
```

- [ ] **Step 2: Reframe the overview bullet**

In `idx-overview`, the third bullet currently says the MCP surface lets agents "work inside your live session without being able to break it." Extend it to note there are now **two tiers**: the convenient in-process overlay and a **process-isolated kernel** (separate process, read-only view + explicit push) for a hard boundary.

- [ ] **Step 3: Update the MCP connection section (`idx-mcp`)**

`paar-mcp` now exposes the **process-isolated kernel** surface. Update the prose: the tools (`list_vars`, `expand`, `grid`, `execute`, `session_info`) now read the owner's state read-only and run `execute` in a separate-pid worker; the agent computes on owner data only after the owner `push()`-es it. The `--env`/`--url` resolution is unchanged. Note the `execute` tool no longer takes a `scope` argument.

- [ ] **Step 4: Reconcile the security model (`idx-security`)**

Rewrite the closing paragraph. It currently says a hard boundary "would need a separate process on a copy of the data, which defeats the point of a live shared session." Replace with an honest two-tier statement:
- **Overlay tier** (`/agent/api`): structural name-binding guarantee + AST restriction list; convenient (agent reads owner objects directly) but **not a sandbox** — static analysis is evadable, and it shares the owner's interpreter.
- **Isolated tier** (`/kernel/api`, `paar-mcp`, `paar-exec`): agent code runs in a **separate process** with no reference to owner objects — the hard boundary now exists. The trade-off is exactly the one previously called out: the agent sees owner state only read-only and computes on **copies** the owner pushes, not the live objects. Keep the note that the owner token gates owner-only endpoints and that same-user local processes are outside the model.

- [ ] **Step 5: Run the cycle**

Run: `uv run nbdev-prepare`
Expected: exit 0 (index has `#| eval: false` on live cells, so no server needed). `README.md` regenerates from index.

- [ ] **Step 6: Verify README picked up the changes**

```bash
grep -c "kernel\|paar-exec\|isolated" README.md
```
Expected: ≥ 3 (new kernel/paar-exec/isolated mentions present).

- [ ] **Step 7: Commit**

```bash
git add nbs/index.ipynb README.md _docs/README.md
git commit -m "docs(index): document process-isolated agent kernel tier; reconcile security model

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Self-Review Notes

- **Coverage:** mcp exec full switch to `/kernel/api/exec` (T1 S3), reads matched to kernel surface (T1 S3), instructions/docstrings updated (T1 S1/S2/S4), tests preserved + extended (T1 S5); index gains isolated-kernel section (T2 S1), overview/MCP/security reconciled (T2 S2/S3/S4). Both user-requested items covered.
- **Type/name consistency:** tool set `{list_vars, expand, grid, execute, session_info}` unchanged; `execute(code)` loses `scope`; `__all__` unchanged; `_base`/`PaarClient` API unchanged (path-only edits).

## Open items flagged (not in this plan's scope — confirm if you want them folded in)

- **`/kernel/api/*` has no `_agent_gate`**: unlike `/agent/api`, the kernel routes are not gated by the server's `agent='off'/'readonly'` mode. Routing mcp there means mcp works even when the overlay is disabled. If you want `agent='off'` to also disable the kernel surface, that's a `paar/fasthtml.py` change — separate task.
- **`skills/paar-session/SKILL.md`** (referenced from `idx-mcp`) still documents the overlay etiquette. If agents should follow the isolated-kernel contract, that file needs the same update — say the word and I'll add a Task 3.
