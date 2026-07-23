# Release notes

<!-- do not remove -->

## 0.0.1
First release.

paar is a live, lazy variable inspector for Python. It runs a small HTTP server inside your
process and shows a PyCharm-style variables pane for whatever is in the namespace, whether
that is a Jupyter kernel, a plain script, or a long training run.

What you get in this release:

- **Live view.** In IPython the pane refreshes after every cell. In any other process a
  background thread polls the namespace and pushes changes. You start the inspector once and
  leave it open.
- **Lazy expansion.** Containers load one level at a time, on click. Opening a million-row
  DataFrame costs nothing until you ask for it, and then only one page is read. numpy arrays
  and pandas frames get a scrollable, paged grid instead of a tree.
- **Exec, edit, filter.** Run an expression or statement against the kernel (with an isolated
  mode that throws away side effects), click any scalar to edit it in place, and filter the
  variable list by name or type. The web exec bar and the terminal both offer completion.
- **Three frontends, one server.** An inline panel in Jupyter, a full page in the browser,
  and `paar-tui`, a Textual terminal inspector. `paar-serve script.py` runs a script under a
  live inspector and keeps the server up afterwards so you can look at the final state.
- **Many environments at once.** Each server registers itself by repo or package name in a
  local registry. Any frontend can discover the running servers and switch between them, so
  several projects can run paar side by side.
- **A restricted surface for AI agents.** `serve()` also publishes an agent API. Agent code
  can read your variables and create new ones, but it writes into its own overlay layer, so
  it can never change or delete what you made. An AST check refuses in-place mutation,
  `exec`/`eval` escapes, and destructive file or shell calls. Owner-only endpoints are gated
  by a per-server token. `paar-mcp` exposes this surface to Claude Code and Codex over stdio.

Install with `pip install paar`erver.
