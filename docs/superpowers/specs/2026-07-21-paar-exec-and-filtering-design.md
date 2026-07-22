# paar: exec box + inline edit + filtering — design

Date: 2026-07-21
Status: approved design, spec under review

## Goal

Add runtime-manipulation and filtering to the paar live inspector panel:

1. **Exec box** — run code in the live kernel namespace: add new variables and
   modify existing ones (e.g. `blah = arr`, `arr[0] = 9`). See the result and any
   output/errors; the variable tree updates. Runs in the **global** namespace by
   default, with an **isolated** toggle for side-effect-free evaluation.
2. **Inline value editing** — click a variable or child row's value, type a new
   value, and write it back into the namespace (pydev `changeVariable`).
3. **Filtering** — narrow the rendered variable list by name and/or type.

Non-goals (v1): PyCharm watch-expression toolbar (+/−/↑/↓), persistent exec
history, a main-thread exec queue, remote/multi-user execution, deep isolation of
in-place object mutation.

## Context

paar is an nbdev project. The inspector is a FastHTML app (`paar/fasthtml.py`)
served in-kernel via `JupyUvi` on localhost. Relevant existing pieces:

- `bridge.py` — `Bridge` gives frontend-agnostic access to the live namespace
  (`snapshot`, `view`, `expand`, `grid`); `on_change(cb)` registers `cb` on the
  IPython `post_run_cell` event.
- `snapshot.py` — `snapshot(ns, hidden)` → sorted `list[VarInfo]`;
  `_var_info(name, v, accessor, path)` builds one row; `_walk(ns, accessor)`
  resolves a positional accessor to `(obj, readable_path)`; `profile_view` groups
  by category; `expand`/`grid_page` walk accessors into the namespace.
- `fasthtml.py` — routes `/rows`, `/expand`, `/grid`; `_node(v)` renders a row;
  `_broadcast` pushes an OOB `#rows` reload to WS clients; `inspector()` starts
  the server and wires `on_change(lambda: _broadcast(...))` so the tree refreshes
  after each notebook cell.
- `core.py` — `VarInfo` dataclass.

ipythonng (already a dependency) was evaluated for execution: it is an IPython
*terminal* extension (Rich markdown, kitty-protocol PNG, matplotlib inline, shell
job control). It renders to a terminal, not to our HTML panel, so it does not fit.
Execution uses `IPython`'s `run_cell` instead.

## New module `paar/runtime.py`

Houses both namespace-write operations (execution and value edits), kept separate
from namespace-*reading* (`bridge`). Named `runtime` (not `exec`) to avoid
shadowing the builtin. nbdev source: `nbs/08_runtime.ipynb`.

```python
@dataclass
class ExecResult:
    ok: bool
    result: VarInfo | None   # last-expression value as an inspector row, or None
    stdout: str = ''
    error: str | None = None # formatted traceback / exception text

def run(code, scope='global') -> ExecResult
def set_value(accessor, expr) -> str | None   # returns error text, or None on success
```

### `run(code, scope='global')` — exec box

`scope='global'` (default):

```python
ip = get_ipython()
if ip is None: return ExecResult(ok=False, result=None, error='no IPython kernel')
with capture_output() as cap:
    res = ip.run_cell(code, store_history=True)
```

- Runs in the **live user namespace** — assignments persist and are visible in
  notebook cells too. `x = 5` **adds** a global; `arr = arr + 1` / `arr[0] = 9`
  **modifies** an existing one. True runtime manipulation.
- `res.result` is the last-expression value; `None` for assignments/statements
  (so `blah = arr` shows `result = {NoneType} None`, matching PyCharm).
- `stdout = cap.stdout`; `error` from `res.error_in_exec`/`error_before_exec`
  (formatted `Type: message` plus captured stderr); `ok = res.success and not error`.
- **Inspectable result row:** IPython binds `_` to the last result, so the result
  row is `_var_info('result', res.result, ('_',), '_')`. Its accessor `('_',)`
  means the existing `/expand` and `/grid` routes walk `user_ns['_']` and work on
  the result with no new code — it's expandable/gridable like any variable. (`_`
  starts with `_`, so it's already excluded from the normal snapshot.)
- **Auto-refresh:** `run_cell` fires `post_run_cell`; the existing `on_change`
  hook already handles it → the tree WS-refreshes after every global exec.

`scope='isolated'`:

- Executes in a scratch dict seeded from a shallow copy of the namespace, so the
  user can *reference* existing variables, but new/rebound names do **not** leak
  into globals:

  ```python
  ns = dict(ip.user_ns)                      # shallow copy: bindings isolated
  with capture_output() as cap:
      value = _exec_capture(code, ns)        # run statements, return last-expr value
  ```

  `_exec_capture` compiles the source, `exec`s all but a trailing expression, then
  `eval`s the trailing expression (if any) to get `value`.
- The result row is rendered **flat** (value string only, `is_container`/`is_grid`
  forced off, no accessor) so nothing needs to be reachable from `user_ns` — the
  isolated run leaves zero side effects and the tree does **not** refresh.
- Documented caveat: isolation is of name *bindings*, not deep object state. In-place
  mutation of a shared object (`arr[0] = 9`) still mutates the real object because
  the shallow copy shares references. Full deep isolation is out of scope.

### `set_value(accessor, expr)` — inline edit (changeVariable)

Reuses the readable lvalue path that `_walk` already builds. The `step` fragments
we generate (`['key']`, `[i]`, `.attr`) are valid Python assignment targets, so:

```python
ip = get_ipython()
obj, path = _walk(ip.user_ns, tuple(accessor))   # path is the lvalue, e.g. data['b'][2]
try:
    exec(f'{path} = ({expr})', ip.user_ns)         # writes into the global namespace
    return None
except Exception as e:
    return f'{type(e).__name__}: {e}'
```

- Works for top-level rebinds (`x`), dict items (`d['k']`), list/deque/ndarray
  elements (`[i]`), object attributes (`.attr`), and DataFrame columns (`[c]`).
- Not supported (returns an error, surfaced in the UI): set/frozenset elements
  (their `[i]` path is display-only, not an lvalue) and read-only targets
  (`.shape`, computed stat rows). Failures degrade to an inline error, not a crash.
- `expr` is evaluated in `user_ns` — same trust boundary as the exec box. The
  lvalue `path` is our own generated string, not user input.

## Frontend (`fasthtml.py`)

### Exec box UI + `POST /exec`

- An exec input above the tree: single-line text input (Enter or a Run button),
  plus an **"isolated"** checkbox (unchecked = global default). Issues
  `hx_post='/exec'` with `code` and `scope`, targeting an output container below.
- `/exec` → `runtime.run(code, scope)`; returns a fragment: the result node
  (`_node(r.result)`) when present, and an output panel showing `r.stdout` and, if
  `r.error`, the error styled with the existing `error` class.

### Inline edit UI + `POST /set`

- Each editable row's value is clickable; clicking swaps it for a small input
  prefilled with the current value string. Enter issues `hx_post='/set'` with the
  row's `accessor` and the typed `expr`.
- `/set` → `err = runtime.set_value(accessor, expr)`. On success: `_broadcast` the
  OOB `#rows` reload (the write did **not** go through `run_cell`, so
  `post_run_cell` did not fire — we refresh manually) and return the freshly
  rendered node; on error: return the error inline next to the row.
- `Bridge.set_value(accessor, expr)` forwards to `runtime.set_value`.

### Filtering — client-side hide

- Backend, separate name + type filters (pure data, independently testable, usable
  by any caller): `snapshot(ns, hidden=frozenset(), name=None, typ=None)`.
  - `name`: case-insensitive substring on the variable name.
  - `typ`: case-insensitive equality on the type name (`v.__class__.__name__`).
    Named `typ` (not `type`) to avoid shadowing the builtin in the function.
  - `None` means no constraint; both applied conjunctively. `Bridge.snapshot`
    forwards the same params.
- Frontend: every node carries `data-name` and `data-type`. A filter bar has a
  **name** text input (substring) and a **type** dropdown whose options are the
  distinct `data-type`s present, derived client-side from the DOM. A small JS
  function hides non-matching nodes, force-opens any group `<details>` containing a
  match, and re-applies after each WS refresh via the existing `htmx.onLoad` hook
  (so an active filter survives the post-cell refresh without losing the query).
- Filtering targets the top-level namespace (all nodes in the DOM, including
  collapsed group members). Lazily loaded container *children* are out of scope.

## Security / trust boundary

`/exec` and `/set` run arbitrary code in the kernel — this is the point (runtime
manipulation) and the same trust boundary as typing in the notebook. The server
binds to localhost via `JupyUvi`; no new remote surface beyond what the inspector
already exposes.

## Known limitation (documented, not fixed in v1)

The FastHTML app runs in JupyUvi's server thread, not the kernel main thread, so
`run_cell`/`exec` run on the server thread. Assignments and computations work;
display-magics / async that assume the main thread may not behave exactly as in a
real cell. Acceptable for v1; a main-thread exec-queue is a future option.

## Testing

nbdev test cells, mocking `get_ipython` as the existing bridge tests do.

- `runtime.py`:
  - global assignment → namespace mutated, `result is None`, `ok is True`.
  - global expression → `result` VarInfo populated; `_` bound; expandable accessor.
  - global modify-existing → existing var's value changes.
  - isolated run → globals unchanged, result rendered flat, `ok is True`.
  - error → `ok is False`, `error` contains the exception text.
  - stdout captured into `ExecResult.stdout`.
  - no-IPython → `ok is False` with a clear message.
  - `set_value` top-level rebind, dict item, list element, attribute → value
    changes; set element / read-only target → returns error string, no crash.
- `snapshot.py`: filter by name, by type, by both, no-match → empty.
- `fasthtml.py`:
  - `/exec` global assignment returns output panel; expression returns result node;
    isolated flag runs isolated.
  - `/set` writes and returns the updated node; bad target returns error inline.
  - nodes carry `data-name`/`data-type`; filter bar renders name input + type
    dropdown.

## Files touched

- `paar/core.py` — untouched; `VarInfo` already carries what the result row needs.
- `nbs/08_runtime.ipynb` (new) → `paar/runtime.py` — `run`, `set_value`, `ExecResult`.
- `nbs/03_snapshot.ipynb` — `name`/`typ` filter params on `snapshot`.
- `nbs/04_bridge.ipynb` — filter params on `Bridge.snapshot`; `Bridge.set_value`.
- `nbs/05_fasthtml.ipynb` — `/exec` + `/set` routes, exec input + isolated toggle,
  inline-edit UI, `data-*` attrs, filter bar + filter JS.
- `nbs/index.ipynb` — mention the new capabilities if the index documents features.

Run `nbdev_prepare` after changes; check `index.ipynb` for stale references.
