# Design: Simplify paar's agent story to one mechanism

Date: 2026-07-23
Status: Implemented

## Problem

paar shipped **three** overlapping ways for an agent to touch a session, and the
cost showed up as conceptual surface, a dependency, and self-inflicted "multiple
sessions":

1. `AgentSession` (`agent.py`) — the in-process overlay: agent reads the owner's
   live namespace, its writes land in a private layer, an AST restriction list runs
   before every cell. Served at `/agent/api/*`. This is the distinctive idea.
2. `AgentKernel` (`kernel.py`) — a supervised **clikernel subprocess**, served at
   `/kernel/api/*`, with its own `share`/`push`/`push_all` routes that *pickle-copy*
   owner variables into the worker. Started **by default** (`agent_kernel=True`), and
   `_ensure_kernel()` had the worker serve its *own* paar inspector on `port+1`.
3. The clikernel "mirror" recipe in the README (paar riding inside the agent's own
   worker via `startup.py`).

Mechanism 2 was the problem child. It could only show the agent *rendered read-only
copies* of owner state, which defeats the "live shared session" pitch; the README's
own security section already conceded these are guardrails, not a sandbox, so the
process-isolation story didn't buy the guarantee it implied. Worse, because it
auto-started and registered itself, one `serve()` produced two registry entries
(`fossick` and `fossick-agent`) — making the multi-environment registry look like
noise when it is actually a wanted feature.

## Decision

Collapse to **one** agent mechanism (`AgentSession`) with two transports over it
(the `/agent/api` HTTP surface and `paar-mcp`), plus direct in-process use.

### Removed

- `nbs/12_kernel.ipynb` + `paar/kernel.py` (`AgentKernel`, `guarded_run`, `paar-exec`).
- `/kernel/api/*` routes and the `share`/`push`/`unpush`/`push_all` + pickle block in
  `fasthtml`, the `_share_panel` UI, and the TUI `p`/`a` share bindings + `Client.share*`.
- `agent_kernel=` params on `inspector()`/`serve()`; the `paar-exec` entry point; the
  `clikernel` runtime dependency.

### Kept (deliberately)

- The **registry** and TUI env-switcher / web env dropdown / `paar-mcp --env`. Running
  `fossick` and `kosha` side by side and flipping between them is a real use case; it
  costs ~84 lines. With the auto-started agent worker gone, every registry entry is now
  an intentional `serve()`, one per project.
- `agent='restricted'|'readonly'|'off'`, the owner token, `agent_layer()`, `SKILL.md`.
- `AgentSession`'s clikernel-*compatible* inspector file contract (`inspect(tree[,code])`,
  `inspectors`, `RuleBlock`). This is an interface convention, not a runtime dep.

## Added

### Agent run transcript

`log_run` gained a `path`/`cell_type` parameter (it was hardcoded to the owner session).
`AgentSession` now transcribes **every attempted cell** to `paar_sessions/agent_<stamp>.ipynb`
(configurable via `log=`), outcome first:

- clean run → plain code cell
- refused → leading `# blocked: <reason>` comment
- raised → leading `# error: <Type>: <msg>` comment
- throwaway → leading `# scope: isolated`

Logging every *attempt* (not just successes) is the point: a refused cell is the most
informative line in an audit. Because `AgentSession` is now the single choke point, this
one hook covers all transports (MCP `execute`, `/agent/api/exec`, direct use). The path is
surfaced in `/agent/api/info` and the MCP `session_info`. `list_sessions`/`read_session`
pick up `agent_*` notebooks alongside owner `session_*` ones for free. The transcript is for
**review, not replay** — re-running it in the owner kernel would do the writes the overlay exists to prevent.

### rishi example + conversation logger

`agent_tools(session)` wraps an `AgentSession` as plain `list_vars`/`run_python` functions
for any tool-calling agent. `conversation_logger(session)` is a rishi `ChatCallback`
(imported lazily — **no** rishi dependency added) that mirrors each turn's prose into the
*same* transcript notebook as markdown cells, interleaved with the tools' code cells. The
result is one notebook that is the whole runnable history of the conversation. Documented
as a `#| eval: false` demo (rishi needs an on-device model); `agent_tools` is unit-tested
without rishi.

This composes cleanly: rishi's `hitl_policy` gates each call *before* it runs, and any
approved `run_python` cell still passes paar's AST restriction list.

## Sequencing note

The approved `2026-07-23-nbs-consolidation-fastcore-design.md` already omits
`12_kernel.ipynb` from its 13→7 target table. This cut should land first so that
consolidation folds a smaller, cleaner set (registry stays, kernel is simply gone).

## Success criteria (met)

- `nbdev-prepare` green end to end; entry points `paar-tui`/`paar-serve`/`paar-mcp` import.
- `clikernel` no longer in `pyproject.toml` or `uv.lock`.
- One agent mechanism; registry/env-switching retained.
- Agent runs transcribed and reviewable; rishi example runnable offline.
