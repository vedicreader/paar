# paar — live lazy variable inspector — design

Date: 2026-07-20
Status: approved (pre-implementation)
Repo: `~/code/personal/orgs/paar` (nbdev, hatchling, Apache-2.0, Python ≥3.10)

## Summary

`paar` adds an inline, realtime variable inspector to notebooks — a reverse-engineering of
PyCharm's "live spatial inspector" (the Variables pane driven by pydev's
`frame_vars_to_struct`/`var_to_struct`), rebuilt as a clean, frontend-agnostic package in
fastcore style.

The panel shows every binding in the notebook namespace with type, module qualifier,
truncated repr, shape, and dtype; containers expand lazily on click; arrays/DataFrames get a
paged tabular viewer. It refreshes after every cell execution.

v1 target frontend is **FastHTML served in-kernel via `JupyUvi`**, viewed inline in a
notebook cell with `HTMX()`. A JupyterLab labextension is a later, separately-specced phase
that reuses the same core.

## Key insight vs pydev

pydev runs the console/debugger in a **separate process** and must serialize every variable
across a bidirectional Thrift/XML-RPC socket (`InterpreterInterface.getFrame` →
`frame_vars_to_struct` → wire → IDE). Its serialization logic is welded to that transport.

`JupyUvi` runs the FastHTML server **in the same process as the Jupyter kernel**, so the app
reads the live namespace directly (`get_ipython().user_ns`). The hard part of pydev — the
wire — collapses to an in-process function call. We therefore reverse-engineer only pydev's
*serialization logic* (fields, resolvers, shape/dtype, lazy evaluation), not its transport.
Cross-process serialization reappears only in the future JupyterLab frontend (over a Jupyter
Comm), and is isolated there.

## Decisions (locked)

- **Architecture:** shared frontend-agnostic core + thin per-frontend adapters.
- **Refresh trigger:** IPython `post_run_cell` hook → WebSocket nudge (after each cell).
- **Fidelity:** full pydev parity, built in layered phases.
- **Sourcing:** clean reimplementation in fastcore style (`store_attr`, small registries).
- **Namespace access:** in-process, direct (no RPC) for the FastHTML frontend.
- **Paths:** human-readable dotted/indexed access paths (`"df.columns[0]"`, `"d['k']"`).
- **WS strategy:** nudge-then-pull (server pushes a tiny `"changed"` signal; client pulls
  rows via HTMX — single server-authoritative render path).
- **Lazy/cycle safety:** top level never walks children; `expand` goes exactly one level.

## Architecture — layers and nbdev notebooks

Dependencies point downward only: frontends → bridge → serialize → model. Nothing imports
upward.

| Notebook | Module | Layer | Purpose |
|---|---|---|---|
| `00_core.ipynb` | `paar/core.py` | model | `VarInfo` dataclass + shared constants/sentinels |
| `01_reprs.ipynb` | `paar/reprs.py` | serialize | truncation, repr providers, shape/dtype extraction |
| `02_resolvers.ipynb` | `paar/resolvers.py` | serialize | type-keyed resolver registry → container children |
| `03_snapshot.ipynb` | `paar/snapshot.py` | serialize | `snapshot`, `expand`, `array_page` |
| `04_bridge.ipynb` | `paar/bridge.py` | bridge | IPython `post_run_cell`, namespace access, op API |
| `05_fasthtml.ipynb` | `paar/fasthtml.py` | frontend | JupyUvi app, WS push, HTMX rows + grid |
| `06_jupyterlab.ipynb` | `paar/jupyterlab.py` | frontend | *(later phase)* Comm + labextension |

**Dependencies:** base install keeps `dependencies = []`. `IPython` added with the bridge,
`python-fasthtml` with the frontend. numpy/pandas stay optional — their resolvers register
only if importable.

## Serialize core (the reverse-engineered pydev logic)

### `VarInfo` (`00_core`) — our `DebugValue`

```python
@dataclass
class VarInfo:
    name:str                 # binding name (or child key/index)
    type:str                 # type(v).__name__
    qualifier:str=''         # type(v).__module__
    value:str=''             # truncated repr, or LAZY sentinel if not evaluated
    shape:str|None=None      # str(tuple(v.shape)) or str(len(v))
    dtype:str|None=None      # str(v.dtype) for arrays/tensors
    is_container:bool=False   # a resolver exists -> render expand arrow
    is_error:bool=False       # repr/attr access raised (pydev isErrorOnEval)
    path:str=''              # readable access path for lazy expand
```

Uses `store_attr()`.

### `reprs.py` (`01`)

- `safe_repr(v, n=300)` — `repr` inside try/except, truncated with a marker.
- `get_shape(v)` — `str(tuple(v.shape))` for non-callable `.shape`, else `str(len(v))` for
  sized non-strings (pydev's exact fallback order).
- `get_dtype(v)` — `str(v.dtype)` when present.
- str-provider registry (`@str_provider(pd.DataFrame)`) for types whose raw repr is unusable;
  optional-import guarded (pydev's str-providers).

### `resolvers.py` (`02`) — the container hierarchy (heart of parity)

```python
class Resolver:
    def can(self, v) -> bool: ...
    def children(self, v) -> list[tuple[str,Any]]: ...   # [(key, child_obj), ...]
```

`resolve_for(v)` returns the first matching resolver or `None`. `None` ⇒ not a container ⇒
no expand arrow (mirrors pydev's `resolver is not None ⇒ isContainer`).

v1 resolvers, ordered specific→generic: `DictResolver`, `ListTupleResolver`, `SetResolver`,
`NumpyResolver`, `PandasResolver`, `ObjectResolver` (fallback: `vars(v)` / non-callable
attrs).

### `snapshot.py` (`03`) — the three operations the bridge calls

- `snapshot(ns, hidden=frozenset()) -> list[VarInfo]` — sorted keys, skip hidden/dunders,
  one `VarInfo` per binding. Top level lazy: `value` filled, children **not** walked
  (pydev's `should_evaluate_full_value` gate).
- `expand(ns, path) -> list[VarInfo]` — resolve `path` to an object, run its resolver,
  return one `VarInfo` per child (pydev `getVariable`).
- `array_page(ns, path, roff, coff, rows, cols) -> table` — numpy/pandas slice → headers,
  rows, dtypes for the tabular viewer (pydev `getArray`).

All three take plain dict/object input and return dataclasses — fully unit-testable with no
IPython and no browser. This is where full parity is proven.

## Bridge (`04_bridge`)

Only IPython-aware layer. Returns `VarInfo`/tables, never HTML.

```python
def _ns():
    ip = get_ipython(); return ip.user_ns if ip else {}

def _hidden():
    ip = get_ipython(); return frozenset(ip.user_ns_hidden) if ip else frozenset()

class Bridge:
    def snapshot(self):     return snapshot(_ns(), _hidden())
    def expand(self, path): return expand(_ns(), path)
    def array_page(self, path, roff, coff, rows, cols):
                            return array_page(_ns(), path, roff, coff, rows, cols)

def on_change(cb):
    "Register cb() to fire after every cell execution"
    get_ipython().events.register('post_run_cell', lambda res: cb())
```

`user_ns_hidden` is IPython's own injected-name set (`In`, `Out`, `_`, `exit`, …) — our
equivalent of pydev's `get_ipython_hidden_vars_dict`.

## Viewing — how the panel appears in a notebook

This is the primary user touchpoint, so it is specified in detail.

### The mechanism

`JupyUvi(app, port=..., daemon=True)` starts a uvicorn server on a background thread inside
the running kernel. `HTMX(path='/', port=..., height=...)` returns an `<iframe>` (as an
IPython display object) pointing at `http://localhost:<port>/`. Placed as a cell's output,
the iframe renders the FastHTML app inline. No notebook clone, no JS build, no separate
process.

Confirmed API (fasthtml.jupyter, present in the venv):
- `JupyUvi(app, log_level='error', host='0.0.0.0', port=8000, start=True, live=False, daemon=False, **kwargs)`
- `HTMX(path='/', host='localhost', app=None, port=8000, height='auto', link=False, iframe=True, client=None)`
- `ws_client(app, nm='', host='localhost', port=8000, ws_connect='/ws', frame=True, link=True, **kwargs)`

### The one-line entry point

```python
from paar.fasthtml import inspector
inspector()          # renders the panel in this cell's output; live from now on
```

`inspector(port=8000, height=520)`:
1. builds the `FastHTML` app + `Bridge`,
2. starts `JupyUvi(app, port=port, daemon=True)` (background, dies with the kernel),
3. registers `on_change` → WS broadcast,
4. returns `HTMX(port=port, height=height, link=True)`.

`height` is passed straight to `HTMX`; `link=True` also renders a click-out link so the same
panel can be opened as a full browser tab (`http://localhost:<port>/`) — useful for a large,
dedicated window beside the notebook.

### Viewing modes (all from the same running server)

1. **Inline cell (default):** the `HTMX()` iframe in the `inspector()` cell output. It stays
   live: because refresh is driven by `post_run_cell` + WS, running *other* cells updates
   this panel in place — you do not re-run the `inspector()` cell.
2. **Docked side panel (JupyterLab):** right-click the `inspector()` cell output →
   "Create New View for Cell Output", then drag that view into a split pane. Result: a
   persistent inspector docked beside your notebook that updates as you run cells — the
   closest match to PyCharm's Variables pane. (Classic Notebook: use the browser-tab mode.)
3. **Full browser tab:** open `http://localhost:<port>/` (or click the `link=True` link).
   Same app, full width, still live over WS. Good for a second monitor.

### Sizing, persistence, lifecycle

- Default `height=520`, scrollable rows; user-overridable.
- One server per kernel, tracked in a module-level handle. Calling `inspector()` again when
  the same kernel already has a server on that port reuses it (just returns a fresh `HTMX()`
  view). If the port is held by a *different* process, it errors clearly and suggests another
  port. The iframe can be re-emitted in any cell to get another view onto the same server.
- `daemon=True` ties the server's lifetime to the kernel; kernel shutdown stops it.
- WS reconnect: the client script retries the `/live` socket on drop so a kernel-busy period
  doesn't permanently detach the panel.

## FastHTML frontend (`05_fasthtml`)

```python
def inspector(port=8000, height=520):
    "Start the live inspector panel; call once in a notebook cell."
    app = FastHTML(hdrs=(picolink,))
    bridge = Bridge()

    @app.get('/')
    def home(): return Div(Div(id='rows', hx_get='/rows', hx_trigger='load'),
                           _ws_script(), id='paar')

    @app.get('/rows')
    def rows(): return _render(bridge.snapshot())

    @app.get('/expand')
    def expand(path:str): return _render(bridge.expand(path))

    @app.get('/grid')
    def grid(path:str, roff:int=0, coff:int=0):
        return _grid(bridge.array_page(path, roff, coff, 50, 50))

    @app.ws('/live')
    async def live(send): _clients.append(send)

    server = JupyUvi(app, port=port, daemon=True)
    on_change(lambda: _broadcast('changed'))
    return HTMX(port=port, height=height, link=True)
```

Rendering (`_render`, `_grid`) maps `VarInfo` → HTMX:
- one row per `VarInfo`: `name`, `type` (+ `qualifier` on hover), `value`, `shape`/`dtype`
  badges;
- `is_container` → expand arrow with `hx_get="/expand?path=..."`, `hx_target` a child div,
  `hx_swap="innerHTML"`; nested expansion recurses the same template;
- `is_error` → error-styled row (pydev `isErrorOnEval`);
- arrays/DataFrames → "view grid" link → `/grid` → paged table with prev/next on
  `roff`/`coff` (pydev `getArray` paging).

Styling via `picolink` (PicoCSS, bundled with FastHTML) — no extra assets in v1.

The WS nudge: `post_run_cell` → `cb()` → `_broadcast('changed')` → each registered client
socket receives `"changed"` → client fires an HTMX `GET /rows`. Server is the single render
authority.

## Error handling

- Every object touch (`repr`, `.shape`, `.dtype`, resolver `children`) is individually
  try/excepted → `is_error` row; one bad object never crashes the panel (pydev per-var
  recovery).
- WS `send` failure drops that client from `_clients`.
- `JupyUvi` port conflict → clear message naming the port and suggesting another.
- `get_ipython()` is `None` (non-IPython context) → `_ns()` returns `{}`, panel renders empty
  rather than raising.

## Testing strategy

- **`00–03` core:** nbdev test cells / pytest with literal dicts, e.g.
  `snapshot({'x': np.arange(9).reshape(3,3)})` asserts `shape=="(3, 3)"`, `dtype=="int64"`,
  `is_container`. No IPython, no browser. Parity verified here.
- **`04` bridge:** `TestClient` + fake `get_ipython` returning a stub ns; assert routes and
  that a simulated `post_run_cell` fires `cb`.
- **`05` frontend:** `TestClient` GETs `/rows`, `/expand?path=...`; assert HTML contains
  expected names/values; assert WS nudge enqueues to registered clients.
- **integration:** `index.ipynb` runs `inspector()`, then later cells mutate vars and the
  panel is observed to refresh (JupyUvi acceptance test + live docs).

## Build phasing (how full parity lands)

1. **P1 — core + flat rows:** `00–03` snapshot (no expand) + `04` + `05` rows + WS refresh.
   Usable end-to-end.
2. **P2 — lazy expansion:** resolvers + `/expand`.
3. **P3 — tabular viewer:** `array_page` + `/grid` paging.
4. **P4 — str-providers / custom renderers.**
5. **P5 (separate spec):** `06_jupyterlab` Comm + labextension.

Each phase is shippable; P1 alone is a working live inspector.

## Out of scope (v1)

- JupyterLab labextension (P5, separate spec).
- On-mutation / mid-cell live tracing (sys.monitoring) — rejected as invasive.
- Editing variable values from the panel (read-only inspector).
- Remote/out-of-process kernels for the FastHTML frontend (in-process assumption).
