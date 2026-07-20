# paar Live Inspector — P1 (core + flat rows + live refresh) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a working live variable inspector that renders every notebook binding as a flat row (name/type/qualifier/value/shape/dtype) in an inline FastHTML panel and refreshes after each cell execution.

**Architecture:** Frontend-agnostic serialize core (`namespace dict → list[VarInfo]`) with no notebook/web deps; an IPython bridge that reads `get_ipython().user_ns` in-process and fires on `post_run_cell`; a FastHTML app served in-kernel by `JupyUvi`, embedded via `HTMX()`, using the htmx websocket extension for nudge-then-pull refresh.

**Tech Stack:** Python ≥3.10, nbdev, fastcore, IPython, python-fasthtml (JupyUvi/HTMX + htmx ws ext), PicoCSS.

## Global Constraints

- **nbdev workflow.** All code is written in `nbs/*.ipynb` cells marked `#| export` and exported to `paar/*.py`. Each notebook starts with `#| default_exp <modname>`. Tests are plain (non-export) cells in the same notebook. After edits, run `nbdev-prepare` (hyphen — never `nbdev_prepare`) to export, test, and clean.
- **Commits: opt-out for this session.** The user has declined commits. Treat each "Commit" step as a checkpoint: run `nbdev-prepare` and confirm green; only run `git commit` if the user explicitly asks. Commit commands are shown for completeness.
- **Base dependencies stay minimal.** `IPython` is added at Task 4, `python-fasthtml` at Task 5, via `uv add`. numpy/pandas are NOT dependencies; tests use lightweight fakes.
- **Python ≥3.10**, Apache-2.0, package name `paar`.
- **Readable paths:** `VarInfo.path` is a human-readable accessor (P1: just the binding name).
- **Layer rule:** frontend → bridge → serialize → model; never import upward.

---

### Task 1: `VarInfo` model

**Files:**
- Modify: `nbs/00_core.ipynb` (replace the `foo` stub) → exports `paar/core.py`
- Test: cells in `nbs/00_core.ipynb`

**Interfaces:**
- Consumes: nothing.
- Produces: `VarInfo` dataclass with fields `name:str, type:str, qualifier:str='', value:str='', shape:str|None=None, dtype:str|None=None, is_container:bool=False, is_error:bool=False, path:str=''`; and `LAZY:str` sentinel.

- [ ] **Step 1: Write the failing test**

Add a test cell to `nbs/00_core.ipynb`:

```python
from paar.core import VarInfo, LAZY
def test_varinfo_defaults():
    v = VarInfo(name='x', type='int')
    assert v.name=='x' and v.type=='int'
    assert v.qualifier=='' and v.value=='' and v.shape is None and v.dtype is None
    assert v.is_container is False and v.is_error is False and v.path==''
test_varinfo_defaults()
```

- [ ] **Step 2: Run to verify it fails**

Run the cell (or `nbdev-prepare`). Expected: `ImportError: cannot import name 'VarInfo'`.

- [ ] **Step 3: Write minimal implementation**

Set the first cell to `#| default_exp core`. Replace the `foo` cell with an export cell:

```python
#| export
from __future__ import annotations
from dataclasses import dataclass

LAZY = '…'  # placeholder repr for values not yet evaluated (used from P1 onward)

@dataclass
class VarInfo:
    "One inspector row: a namespace binding or a container child."
    name:str
    type:str
    qualifier:str=''
    value:str=''
    shape:str|None=None
    dtype:str|None=None
    is_container:bool=False
    is_error:bool=False
    path:str=''
```

- [ ] **Step 4: Run to verify it passes**

Run: `nbdev-prepare`
Expected: export succeeds, `paar/core.py` regenerated, `test_varinfo_defaults` passes.

- [ ] **Step 5: Commit (checkpoint)**

```bash
# checkpoint only unless user asks to commit
git add nbs/00_core.ipynb paar/core.py && git commit -m "feat(core): VarInfo model + LAZY sentinel"
```

---

### Task 2: value rendering helpers (`reprs`)

**Files:**
- Create: `nbs/01_reprs.ipynb` → exports `paar/reprs.py`
- Test: cells in `nbs/01_reprs.ipynb`

**Interfaces:**
- Consumes: nothing.
- Produces: `safe_repr(v, n=300) -> str`; `get_shape(v) -> str|None`; `get_dtype(v) -> str|None`.

- [ ] **Step 1: Write the failing tests**

New notebook `nbs/01_reprs.ipynb`, first cell `#| default_exp reprs`, then a test cell:

```python
from paar.reprs import safe_repr, get_shape, get_dtype
class _Fake:
    def __init__(self, shape=None, dtype=None):
        if shape is not None: self.shape=shape
        if dtype is not None: self.dtype=dtype
class _Bad:
    def __repr__(self): raise ValueError('boom')

def test_safe_repr_ok():    assert safe_repr('hi')=="'hi'"
def test_safe_repr_trunc(): 
    r=safe_repr('a'*500, 10); assert len(r)==10 and r.endswith('…')
def test_safe_repr_bad():   assert 'repr-error' in safe_repr(_Bad())
def test_shape_attr():      assert get_shape(_Fake(shape=(3,4)))=='(3, 4)'
def test_shape_len():       assert get_shape([1,2,3])=='3'
def test_shape_str_none():  assert get_shape('hello') is None
def test_shape_scalar_none():assert get_shape(5) is None
def test_dtype_attr():      assert get_dtype(_Fake(dtype='float32'))=='float32'
def test_dtype_none():      assert get_dtype(5) is None
for t in [test_safe_repr_ok,test_safe_repr_trunc,test_safe_repr_bad,test_shape_attr,
          test_shape_len,test_shape_str_none,test_shape_scalar_none,test_dtype_attr,test_dtype_none]: t()
```

- [ ] **Step 2: Run to verify it fails**

Run the cell. Expected: `ImportError: cannot import name 'safe_repr'`.

- [ ] **Step 3: Write minimal implementation**

Add export cells:

```python
#| export
def safe_repr(v, n=300):
    "repr(v) truncated to n chars, never raising."
    try: r = repr(v)
    except Exception as e: return f'<repr-error {type(e).__name__}: {e}>'
    return r if len(r)<=n else r[:n-1]+'…'
```

```python
#| export
def get_shape(v):
    "tuple(v.shape) as a string; else len(v) for sized non-strings; else None."
    try:
        s = getattr(v, 'shape', None)
        if s is not None and not callable(s): return str(tuple(s))
        if hasattr(v, '__len__') and not isinstance(v, (str, bytes, bytearray)):
            return str(len(v))
    except Exception:
        return None
    return None
```

```python
#| export
def get_dtype(v):
    "str(v.dtype) when present, else None."
    try:
        d = getattr(v, 'dtype', None)
        return str(d) if d is not None else None
    except Exception:
        return None
```

- [ ] **Step 4: Run to verify it passes**

Run: `nbdev-prepare`
Expected: `paar/reprs.py` generated; all nine asserts pass.

- [ ] **Step 5: Commit (checkpoint)**

```bash
git add nbs/01_reprs.ipynb paar/reprs.py && git commit -m "feat(reprs): safe_repr, get_shape, get_dtype"
```

---

### Task 3: flat snapshot (`snapshot`)

**Files:**
- Create: `nbs/03_snapshot.ipynb` → exports `paar/snapshot.py`
- Test: cells in `nbs/03_snapshot.ipynb`

**Interfaces:**
- Consumes: `paar.core.VarInfo`; `paar.reprs.safe_repr/get_shape/get_dtype`.
- Produces: `snapshot(ns:dict, hidden=frozenset()) -> list[VarInfo]` (sorted by name, skips `hidden` names and dunder names, flat — `is_container` stays `False` in P1); helper `_var_info(name, v) -> VarInfo`.

- [ ] **Step 1: Write the failing tests**

New notebook `nbs/03_snapshot.ipynb`, first cell `#| default_exp snapshot`, then:

```python
from paar.snapshot import snapshot
def test_sorted_and_fields():
    rows = snapshot({'b':1, 'a':'hi'})
    assert [r.name for r in rows]==['a','b']
    assert rows[0].type=='str' and rows[0].value=="'hi'" and rows[0].path=='a'
    assert rows[1].type=='int' and rows[1].qualifier=='builtins'
def test_skips_hidden_and_dunder():
    rows = snapshot({'x':1, '_ih':2, '__y':3}, hidden=frozenset({'_ih'}))
    assert [r.name for r in rows]==['x']
def test_shape_populated():
    assert snapshot({'L':[1,2,3]})[0].shape=='3'
def test_flat_no_container():
    assert snapshot({'d':{'a':1}})[0].is_container is False
for t in [test_sorted_and_fields,test_skips_hidden_and_dunder,test_shape_populated,test_flat_no_container]: t()
```

- [ ] **Step 2: Run to verify it fails**

Run the cell. Expected: `ImportError: cannot import name 'snapshot'`.

- [ ] **Step 3: Write minimal implementation**

```python
#| export
from paar.core import VarInfo
from paar.reprs import safe_repr, get_shape, get_dtype

def _var_info(name, v):
    "Build a flat VarInfo for one binding (P1: no container resolution)."
    try:
        return VarInfo(name=name, type=type(v).__name__,
                       qualifier=getattr(type(v), '__module__', '') or '',
                       value=safe_repr(v), shape=get_shape(v), dtype=get_dtype(v),
                       path=name)
    except Exception as e:
        return VarInfo(name=name, type='?', value=f'<error {e}>', is_error=True, path=name)

def snapshot(ns, hidden=frozenset()):
    "namespace dict -> sorted list[VarInfo], skipping hidden and dunder names."
    keys = sorted(k for k in ns if k not in hidden and not k.startswith('__'))
    return [_var_info(k, ns[k]) for k in keys]
```

- [ ] **Step 4: Run to verify it passes**

Run: `nbdev-prepare`
Expected: `paar/snapshot.py` generated; all four asserts pass.

- [ ] **Step 5: Commit (checkpoint)**

```bash
git add nbs/03_snapshot.ipynb paar/snapshot.py && git commit -m "feat(snapshot): flat namespace snapshot"
```

---

### Task 4: IPython bridge (`bridge`)

**Files:**
- Create: `nbs/04_bridge.ipynb` → exports `paar/bridge.py`
- Modify: `pyproject.toml` (add `IPython` dep via `uv add ipython`)
- Test: cells in `nbs/04_bridge.ipynb`

**Interfaces:**
- Consumes: `paar.snapshot.snapshot`.
- Produces: `Bridge` class with `snapshot(self) -> list[VarInfo]`; module functions `_ns() -> dict`, `_hidden() -> frozenset`, `on_change(cb) -> bool` (registers `cb` on `post_run_cell`, returns whether an IPython shell was present). Module global `get_ipython` is monkeypatch-friendly (looked up at call time).

- [ ] **Step 1: Add the dependency**

Run: `uv add ipython`
Expected: `ipython` added to `pyproject.toml` `dependencies` and installed.

- [ ] **Step 2: Write the failing tests**

New notebook `nbs/04_bridge.ipynb`, first cell `#| default_exp bridge`, then:

```python
import paar.bridge as B
from paar.bridge import Bridge, on_change

class _Events:
    def __init__(self): self.cbs=[]
    def register(self, name, fn): self.cbs.append((name, fn))
class _IP:
    def __init__(self, ns): self.user_ns=ns; self.user_ns_hidden={'_ih'}; self.events=_Events()

def test_bridge_snapshot_uses_ns_and_hidden():
    B.get_ipython = lambda: _IP({'a':1, '_ih':2})
    assert [r.name for r in Bridge().snapshot()]==['a']
def test_on_change_registers_and_fires():
    ip = _IP({}); B.get_ipython = lambda: ip
    hits=[]; ok = on_change(lambda: hits.append(1))
    assert ok is True and ip.events.cbs[0][0]=='post_run_cell'
    ip.events.cbs[0][1](None); assert hits==[1]
def test_no_ipython_is_safe():
    B.get_ipython = lambda: None
    assert Bridge().snapshot()==[] and on_change(lambda: None) is False
test_bridge_snapshot_uses_ns_and_hidden(); test_on_change_registers_and_fires(); test_no_ipython_is_safe()
```

- [ ] **Step 3: Run to verify it fails**

Run the cell. Expected: `ImportError: cannot import name 'Bridge'`.

- [ ] **Step 4: Write minimal implementation**

```python
#| export
from paar.snapshot import snapshot
try: from IPython import get_ipython
except Exception:
    def get_ipython(): return None

def _ns():
    ip = get_ipython(); return ip.user_ns if ip else {}

def _hidden():
    ip = get_ipython()
    return frozenset(getattr(ip, 'user_ns_hidden', ())) if ip else frozenset()

class Bridge:
    "Frontend-agnostic access to the live kernel namespace."
    def snapshot(self): return snapshot(_ns(), _hidden())

def on_change(cb):
    "Register cb() to fire after each cell execution; returns False outside IPython."
    ip = get_ipython()
    if not ip: return False
    ip.events.register('post_run_cell', lambda res: cb())
    return True
```

- [ ] **Step 5: Run to verify it passes**

Run: `nbdev-prepare`
Expected: `paar/bridge.py` generated; three asserts pass.

- [ ] **Step 6: Commit (checkpoint)**

```bash
git add nbs/04_bridge.ipynb paar/bridge.py pyproject.toml uv.lock && git commit -m "feat(bridge): in-process IPython namespace bridge + post_run_cell hook"
```

---

### Task 5: FastHTML live panel (`fasthtml`)

**Files:**
- Create: `nbs/05_fasthtml.ipynb` → exports `paar/fasthtml.py`
- Modify: `pyproject.toml` (add `python-fasthtml` via `uv add python-fasthtml`)
- Test: cells in `nbs/05_fasthtml.ipynb`

**Interfaces:**
- Consumes: `paar.bridge.Bridge, on_change`; `paar.core.VarInfo`.
- Produces: module-level `app` (FastHTML), `bridge` (Bridge); `_row(v:VarInfo) -> FT`; `_rows_div() -> FT` (id=`rows`, the rendered snapshot); `_loader(oob=False) -> FT` (a `#rows` div that GET-loads `/rows`); `_broadcast(fragment) -> None` (cross-thread push to WS clients); `inspector(port=8000, height=520) -> HTMX` (starts JupyUvi once, wires `on_change`, returns the inline iframe). Routes: `GET /`, `GET /rows`, `WS /live`.

- [ ] **Step 1: Add the dependency**

Run: `uv add python-fasthtml`
Expected: `python-fasthtml` added and installed.

- [ ] **Step 2: Write the failing tests**

New notebook `nbs/05_fasthtml.ipynb`, first cell `#| default_exp fasthtml`, then:

```python
from starlette.testclient import TestClient
import paar.bridge as B, paar.fasthtml as F
from paar.fasthtml import app, _row, _broadcast, _clients
from paar.core import VarInfo

class _IP:
    user_ns={'alpha': 123}; user_ns_hidden=set()
    class events:
        @staticmethod
        def register(n, fn): pass

def test_row_renders_fields():
    html = str(_row(VarInfo(name='n', type='int', value='5', shape='3')))
    assert 'n' in html and 'int' in html and '5' in html and '3' in html
def test_rows_route_lists_vars():
    B.get_ipython = lambda: _IP()
    r = TestClient(app).get('/rows')
    assert r.status_code==200 and 'alpha' in r.text and '123' in r.text
    assert 'id="rows"' in r.text
def test_home_wires_ws_and_loader():
    r = TestClient(app).get('/')
    assert r.status_code==200
    assert 'ws-connect="/live"' in r.text and 'hx-ext="ws"' in r.text
    assert 'id="rows"' in r.text and '/rows' in r.text   # loader present
def test_broadcast_drops_dead_clients():
    _clients.clear()
    class _Loop:
        def __init__(self): self.n=0
    def _bad_send(_): raise RuntimeError('dead')
    # a client whose loop raises on schedule is dropped
    import paar.fasthtml as FF
    FF._clients.append(('not-a-loop', _bad_send))
    _broadcast('x')
    assert FF._clients==[]
test_row_renders_fields(); test_rows_route_lists_vars(); test_home_wires_ws_and_loader(); test_broadcast_drops_dead_clients()
```

- [ ] **Step 3: Run to verify it fails**

Run the cell. Expected: `ImportError: cannot import name 'app'`.

- [ ] **Step 4: Write minimal implementation**

Export cell — imports and app. Enable the ws extension via FastHTML's built-in `exts` (maps through `fasthtml.core.htmx_exts`, so the script + version are maintained by fasthtml); pico styling is on by default:

```python
#| export
import asyncio
from fasthtml.common import *
from fasthtml.jupyter import JupyUvi, HTMX
from paar.bridge import Bridge, on_change
from paar.core import VarInfo

bridge = Bridge()
app = FastHTML(exts='ws')   # bundles htmx-ext-ws from htmx_exts; pico on by default
_clients = []   # list[(loop, send)]
_server = None
```

Rendering helpers:

```python
#| export
def _row(v:VarInfo):
    "Render one VarInfo as a table row."
    badges = ' '.join(x for x in [v.shape and f'shape={v.shape}',
                                   v.dtype and f'dtype={v.dtype}'] if x)
    cls = 'error' if v.is_error else None
    return Tr(Td(Strong(v.name)), Td(Small(v.type, title=v.qualifier)),
              Td(Code(v.value)), Td(Small(badges)), cls=cls)

def _rows_div():
    "The rendered snapshot, wrapped in the #rows target."
    rows = [_row(v) for v in bridge.snapshot()]
    return Div(Table(Thead(Tr(Th('name'),Th('type'),Th('value'),Th('info'))),
                     Tbody(*rows)), id='rows')

def _loader(oob=False):
    "A #rows div that immediately GETs /rows (used initially and as the WS nudge)."
    kw = dict(hx_get='/rows', hx_trigger='load', hx_swap='outerHTML')
    if oob: kw['hx_swap_oob']='true'
    return Div(id='rows', **kw)
```

Routes:

```python
#| export
@app.get('/')
def home():
    return Titled('paar', Div(_loader(), hx_ext='ws', ws_connect='/live', id='paar'))

@app.get('/rows')
def rows(): return _rows_div()

async def _conn(send): _clients.append((asyncio.get_running_loop(), send))
async def _disconn(send): 
    global _clients
    _clients = [(l,s) for (l,s) in _clients if s is not send]

@app.ws('/live', conn=_conn, disconn=_disconn)
async def live(send): pass
```

Cross-thread broadcast + entry point:

```python
#| export
def _broadcast(fragment):
    "Push fragment to every WS client from any thread; drop dead clients."
    global _clients
    alive=[]
    for loop, send in list(_clients):
        try:
            asyncio.run_coroutine_threadsafe(send(fragment), loop); alive.append((loop, send))
        except Exception:
            pass
    _clients = alive

def inspector(port=8000, height=520):
    "Start the in-kernel live inspector panel and return the inline iframe."
    global _server
    if _server is None:
        _server = JupyUvi(app, port=port, daemon=True)
        on_change(lambda: _broadcast(to_xml(_loader(oob=True))))
    return HTMX(port=port, height=height, link=True)
```

- [ ] **Step 5: Run to verify it passes**

Run: `nbdev-prepare`
Expected: `paar/fasthtml.py` generated; three asserts pass. (`test_broadcast_drops_dead_clients` passes because `run_coroutine_threadsafe('not-a-loop', ...)` raises and the client is dropped.)

- [ ] **Step 6: Commit (checkpoint)**

```bash
git add nbs/05_fasthtml.ipynb paar/fasthtml.py pyproject.toml uv.lock && git commit -m "feat(fasthtml): in-kernel live inspector panel"
```

---

### Task 6: Live integration demo + index

**Files:**
- Modify: `nbs/index.ipynb` (add a live-demo section that doubles as the acceptance test)

**Interfaces:**
- Consumes: `paar.fasthtml.inspector`.
- Produces: no code exports; a runnable demo.

- [ ] **Step 1: Add the demo cells**

In `nbs/index.ipynb`, after the intro, add markdown "## Live demo" and these cells:

```python
from paar.fasthtml import inspector
inspector()          # panel renders inline; keep this cell's output visible
```

```python
import math
data = {'a': 1, 'b': [1,2,3], 'c': 'hello', 'pi': math.pi}
```

```python
data['d'] = list(range(1000))   # run this; the panel above updates without re-running inspector()
```

Add a markdown note: *"For a docked side panel in JupyterLab: right-click the `inspector()` output → Create New View for Cell Output → drag into a split pane. For a full window: open http://localhost:8000/ or click the 'open in browser' link."*

- [ ] **Step 2: Manual acceptance check**

Run the notebook top-to-bottom in JupyterLab. Expected:
- the `inspector()` cell shows a table with `a`, `b`, `c`, `pi` (with `b` showing `shape=3`);
- running the `data['d']=...` cell makes the panel refresh in place to include `d` (`shape=1000`) — **without** re-running `inspector()`. This is the P1 acceptance criterion (live refresh over WS).

- [ ] **Step 3: Prepare & checkpoint**

Run: `nbdev-prepare`
Expected: notebooks clean, all module tests still green.

```bash
git add nbs/index.ipynb && git commit -m "docs: live inspector demo in index"
```

---

## Self-Review

**Spec coverage (P1 scope):**
- VarInfo model → Task 1. ✓
- reprs (truncation, shape, dtype) → Task 2. ✓
- flat snapshot with hidden/dunder skip → Task 3. ✓
- bridge (`_ns`, `_hidden`, `user_ns_hidden`, `post_run_cell`) → Task 4. ✓
- FastHTML frontend (JupyUvi, HTMX, `/rows`, WS `/live`, nudge-then-pull) → Task 5. ✓
- viewing modes (inline / docked / browser tab) → Task 6 note. ✓
- Deferred by phase (correctly absent from P1): resolvers/expand (P2), array grid (P3), str-providers (P4), JupyterLab (P5). ✓

**Placeholder scan:** none — all steps carry runnable code/commands.

**Type consistency:** `VarInfo` field names identical across Tasks 1/3/5; `snapshot(ns, hidden)` signature identical in Tasks 3/4; `Bridge.snapshot`, `on_change`, `_broadcast`, `_loader(oob=)`, `inspector(port, height)` names consistent between Tasks 4/5/6.

**Note on `is_container`:** P1 always leaves it `False` (no resolvers yet); the field exists so P2 can populate it without a model change. `_row` ignores it in P1 — the expand arrow arrives in P2.
