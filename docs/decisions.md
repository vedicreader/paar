# Decisions

This page lists the decisions that a maintainer must know. Each entry gives the decision, the reason, the date and the place in the code.
How to use paar is in the [README](../README.md) and [nbs/index.ipynb](../nbs/index.ipynb). The release notes are in [CHANGELOG.md](../CHANGELOG.md).
The first designs and plans (20 to 24 July 2026) are in the git history under `docs/superpowers/`.

## Shape of the package

- **One core, thin frontends.** The core reads values and makes rows. The frontends (the FastHTML panel and page, `paar-tui`, `paar-mcp`) only show them. Imports go down only: frontend, then bridge, then serialize, then model.
  Why: one render path for every frontend. (20 Jul 2026) `nbs/00_core.ipynb` to `nbs/04_bridge.ipynb`.
- **The server runs in the same process as the namespace.** It reads the live namespace directly. There is no RPC.
  Why: pydev spends most of its effort on the wire. In process, the wire is a function call. (20 Jul 2026) `Bridge` in `nbs/04_bridge.ipynb`.
- **`Bridge` takes a client.** The default client is IPython. There is no global namespace setter. Tests use a real `InteractiveShell`, not fakes. (24 Jul 2026) `Bridge(client)` in `nbs/04_bridge.ipynb`.
- **Entry-point module names stay fixed.** `paar.fasthtml`, `paar.tui` and `paar.mcp` are `[project.scripts]` targets. Other import paths can change before 1.0. (23 Jul 2026) `pyproject.toml`.
- **The 13 to 7 notebook merge was planned but not done.** Two notebooks still share the `08` prefix (`08_runtime`, `08_tui`). The plan is in the git history (`2026-07-23-nbs-consolidation-fastcore.md`).

## Reading values

- **Lazy, one level at a time.** The top level never walks children. `expand` goes exactly one level. numpy arrays and pandas frames get a paged grid. A value that is not read yet shows `LAZY`.
  Why: a large frame costs nothing until the user opens it, and cycles cannot hang the pane. (20 Jul 2026) `nbs/03_snapshot.ipynb`, `nbs/06_grid.ipynb`, `core.LAZY`.
- **A path is readable.** A row carries a path like `df.columns[0]` or `d['k']`. (20 Jul 2026) `VarInfo.path`.
- **Refresh is nudge, then pull.** IPython's `post_run_cell` (or a poll thread in a plain process) sends a small signal. The client then gets the rows through HTMX. The server renders all rows.
  Why: one render path that the server controls. (20 Jul 2026) `Bridge.on_change`, `_poll_loop` in `nbs/05_fasthtml.ipynb`.
- **Value providers change the value string only.** A provider never changes `is_container`, the grid or the children. The match is `isinstance`, and the latest registration wins, so a user provider overrides a built-in. A provider that raises falls back to `safe_repr`. (20 Jul 2026) `nbs/07_providers.ipynb`.

## Writing values

- **Writes live in `runtime`, reads in `bridge`.** `run(code, scope)` and `set_value` are in `paar.runtime`. The name avoids the builtin `exec`. (21 Jul 2026) `nbs/08_runtime.ipynb`.
- **The exec box runs in the global namespace by default.** It uses IPython's `run_cell`, so a name it makes is also in the notebook. `scope='isolated'` runs on a scratch copy. The result row reads `_`, so the expand and grid routes work on it with no new code. ipythonng was not used because it draws to a terminal, not to HTML. (21 Jul 2026) `run` in `nbs/08_runtime.ipynb`.

## The agent surface

- **One agent mechanism: `AgentSession`.** The agent reads the owner's namespace. Its writes go to its own overlay layer. An AST restriction list runs before each cell and refuses changes to owner state, `exec`, `eval` and destructive file or shell calls. The HTTP API (`/agent/api`) and `paar-mcp` are two transports over it.
  Why: `AgentKernel` (a clikernel subprocess) showed the agent only pickled copies, made two registry entries per `serve()`, and added a dependency. It was removed. (23 Jul 2026) `nbs/10_agent.ipynb`, `nbs/11_mcp.ipynb`.
- **`scope='isolated'` runs in safepyrun.** It seeds a throwaway copy of the merged view, drops the writes and refuses calls that are not on the allowlist. A `readonly` server forces it. (24 Jul 2026) `AgentSession._run_isolated`, `safepyrun` in `pyproject.toml`.
- **These are guardrails, not a sandbox.** The overlay and the AST list stop mistakes. They do not stop a hostile agent. Say this in the docs. `nbs/index.ipynb`.
- **Owner endpoints need a token.** `serve()` and `inspector()` make a token per server and write it to `token-<port>` (mode 0600) in the registry folder, so `paar-tui` of the same user can read it. `agent='restricted'|'readonly'|'off'`. (Jul 2026) `_owner_token`, `_authed` in `nbs/05_fasthtml.ipynb`.
- **Every agent cell goes to a transcript.** Refused and failed cells are logged too, with the outcome first (`# blocked:`, `# error:`, `# scope: isolated`). The file is `paar_sessions/agent_<stamp>.ipynb`. It is for review, not for replay. (23 Jul 2026) `AgentSession._log`, `log_run`.
- **rishi is optional.** `agent_tools` gives plain functions for any tool-calling agent. `conversation_logger` imports rishi only when it is called. (23 Jul 2026) `nbs/10_agent.ipynb`.
- **The inspector file follows clikernel's contract** (`inspect(tree[, code])`, `inspectors`). This is an interface, not a dependency. `nbs/10_agent.ipynb`.

## Many environments

- **Each server registers by repo or package name.** Every frontend can list the live servers and switch between them (`paar-mcp --env`, the TUI switcher, the web dropdown). Each entry is one `serve()` that the user started.
  Why: running two projects side by side is a real use case, and it costs little code. (Jul 2026, kept on 23 Jul 2026) `nbs/09_registry.ipynb`.

## Build and release

- **nbdev with hatchling.** Edit the notebooks, then run `uv run nbdev-prepare`. hatchling builds the wheel from `paar/` (`[tool.hatch.build.targets.wheel]`); `MANIFEST.in` has no effect. 0.0.1 shipped on 23 Jul 2026.
