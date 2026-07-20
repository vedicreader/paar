# paar Live Inspector — P3 (numpy/pandas grid viewer) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give numpy arrays and pandas DataFrames/Series a dedicated paged grid viewer in the inspector, and enhance the panel's look with a small Pico-based stylesheet.

**Architecture:** A new serialize-layer leaf `grid.py` detects gridable values (`is_gridable`) and slices a page of one (`array_page`) — numpy/pandas are optional (import-guarded). `snapshot` flags gridable vars (`is_grid`, mutually exclusive with `is_container`) and adds `grid_page(ns, accessor, ...)` (walks the positional accessor, then pages). `bridge` exposes `Bridge.grid`. The FastHTML frontend renders a `▦ grid` affordance for gridable nodes that opens a scrollable, paged table via a `/grid` route, plus custom CSS layered on Pico.

**Tech Stack:** Python ≥3.10, nbdev, fastcore, IPython, python-fasthtml (htmx + Pico), numpy/pandas (optional at runtime, dev deps for tests). Builds on merged P1+P2.

## Global Constraints

- **nbdev workflow.** Code in `nbs/*.ipynb` cells marked `#| export`, exported to `paar/*.py`. Each notebook starts `#| default_exp <mod>`. Tests are plain (non-export) cells in the same notebook. In the notebook use ABSOLUTE imports (`from paar.x import y`); nbdev emits them as relative (`.x`). Run `uv run --directory /Users/71293/code/personal/orgs/paar nbdev-prepare` (hyphen — NEVER `nbdev_prepare`) after edits. nbdev is in the venv (Python 3.13).
- **Always stage `paar/_modidx.py`** in each commit; after committing run `git status --short` and amend in any nbdev reformat so the tree is clean (aside from untracked `.kosha/`, `.superpowers/`).
- **Commits enabled** on the P3 branch. End commit messages with:
  `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>`
- Python ≥3.10, package name `paar`.
- **Layer rule:** frontend(`fasthtml`) → bridge → serialize(`snapshot`→`{resolvers, grid}`→`reprs`) → model(`core`); never import upward. Only `bridge` imports IPython; only `fasthtml` imports fasthtml. `grid` is a serialize leaf importing only `paar.reprs` (+ numpy/pandas, import-guarded).
- **numpy/pandas are OPTIONAL at runtime:** `grid.py` guards imports (`np`/`pd` may be `None`); base install must not require them. They ARE added to the dev group so tests can run.
- **Gridable vs container are mutually exclusive:** in `_var_info`, `is_grid = is_gridable(v)` and `is_container = (not gridable) and resolve_for(v) is not None`. A gridable value is never tree-expanded.
- **Scope:** numpy ndarray of ndim 1 or 2, and pandas DataFrame/Series. 3-D+ arrays are NOT gridable (render as a flat row). Grid pages default to 50×50 with prev/next paging. No silent truncation — paging controls show the current window and totals.
- **Styling:** enhance the built-in Pico theme with a small custom `Style` (compact rows, monospace values, scrollable grid with sticky header). Do NOT add external CSS libraries.
- fastcore/fastai style: one-line docstrings, no decorative comments.

---

### Task 1: Grid module — `is_gridable` + `array_page` (`grid`)

**Files:**
- Create: `nbs/06_grid.ipynb` → exports `paar/grid.py`
- Test: cells in `nbs/06_grid.ipynb`

**Interfaces:**
- Consumes: `paar.reprs.safe_repr`.
- Produces: `is_gridable(v) -> bool`; `array_page(v, roff=0, coff=0, rows=50, cols=50) -> dict|None` with keys `headers:list[str]`, `index:list[str]`, `cells:list[list[str]]`, `nrows:int`, `ncols:int`, `roff:int`, `coff:int`. Returns `None` for non-gridable input.

Note: `numpy` and `pandas` are already declared as project dependencies and installed in the venv — do NOT run `uv add`. `grid.py` still import-guards them (`np`/`pd` may be `None`) so the module imports cleanly even if they were ever absent.

- [ ] **Step 1: Write the failing tests**

New notebook `nbs/06_grid.ipynb`, first cell `#| default_exp grid`, then a test cell:

```python
import numpy as np, pandas as pd
from paar.grid import is_gridable, array_page

def test_is_gridable():
    assert is_gridable(np.arange(6).reshape(2,3))
    assert is_gridable(np.arange(3))
    assert is_gridable(pd.DataFrame({'a':[1,2]}))
    assert is_gridable(pd.Series([1,2,3]))
    assert not is_gridable(np.zeros((2,2,2)))   # 3-D not gridable
    assert not is_gridable([1,2,3]) and not is_gridable(5)
def test_page_ndarray_2d():
    p = array_page(np.arange(6).reshape(2,3))
    assert p['nrows']==2 and p['ncols']==3
    assert p['headers']==['0','1','2'] and p['index']==['0','1']
    assert p['cells']==[['0','1','2'],['3','4','5']]
def test_page_ndarray_1d():
    p = array_page(np.arange(3))
    assert p['headers']==['0'] and p['index']==['0','1','2'] and p['cells']==[['0'],['1'],['2']]
    assert p['nrows']==3 and p['ncols']==1
def test_page_dataframe():
    p = array_page(pd.DataFrame({'x':[1,2],'y':[3,4]}))
    assert p['headers']==['x','y'] and p['index']==['0','1']
    assert p['cells']==[['1','3'],['2','4']] and p['nrows']==2 and p['ncols']==2
def test_page_series():
    p = array_page(pd.Series([10,20], name='s'))
    assert p['headers']==['s'] and p['cells']==[['10'],['20']] and p['nrows']==2 and p['ncols']==1
def test_page_paging_offsets():
    p = array_page(np.arange(10).reshape(10,1), roff=5, rows=3)
    assert p['index']==['5','6','7'] and p['cells']==[['5'],['6'],['7']] and p['nrows']==10 and p['roff']==5
def test_page_non_gridable_none():
    assert array_page([1,2,3]) is None
for t in [test_is_gridable,test_page_ndarray_2d,test_page_ndarray_1d,test_page_dataframe,
          test_page_series,test_page_paging_offsets,test_page_non_gridable_none]: t()
```

- [ ] **Step 2: Run to verify it fails**

Run the cell. Expected: `ImportError: cannot import name 'is_gridable'`.

- [ ] **Step 3: Write minimal implementation**

Export cell (use `from paar.reprs import safe_repr` in the notebook):

```python
#| export
from paar.reprs import safe_repr
try: import numpy as np
except Exception: np = None
try: import pandas as pd
except Exception: pd = None

def is_gridable(v):
    "True for a numpy ndarray (1-2D) or a pandas DataFrame/Series."
    if np is not None and isinstance(v, np.ndarray): return v.ndim in (1, 2)
    if pd is not None and isinstance(v, (pd.DataFrame, pd.Series)): return True
    return False

def _cell(x): return safe_repr(x, 40)

def array_page(v, roff=0, coff=0, rows=50, cols=50):
    "Page a gridable value -> dict(headers,index,cells,nrows,ncols,roff,coff), or None."
    if pd is not None and isinstance(v, pd.DataFrame):
        nrows, ncols = v.shape
        sub = v.iloc[roff:roff+rows, coff:coff+cols]
        headers = [str(c) for c in sub.columns]
        index = [str(i) for i in sub.index]
        cells = [[_cell(x) for x in row] for row in sub.values.tolist()]
    elif pd is not None and isinstance(v, pd.Series):
        nrows, ncols = v.shape[0], 1
        sub = v.iloc[roff:roff+rows]
        headers = [str(v.name) if v.name is not None else 'value']
        index = [str(i) for i in sub.index]
        cells = [[_cell(x)] for x in sub.tolist()]
    elif np is not None and isinstance(v, np.ndarray) and v.ndim == 2:
        nrows, ncols = v.shape
        sub = v[roff:roff+rows, coff:coff+cols]
        headers = [str(c) for c in range(coff, min(coff+cols, ncols))]
        index = [str(r) for r in range(roff, min(roff+rows, nrows))]
        cells = [[_cell(x) for x in row] for row in sub.tolist()]
    elif np is not None and isinstance(v, np.ndarray) and v.ndim == 1:
        nrows, ncols = v.shape[0], 1
        sub = v[roff:roff+rows]
        headers = ['0']
        index = [str(r) for r in range(roff, min(roff+rows, nrows))]
        cells = [[_cell(x)] for x in sub.tolist()]
    else:
        return None
    return {'headers': headers, 'index': index, 'cells': cells,
            'nrows': nrows, 'ncols': ncols, 'roff': roff, 'coff': coff}
```

- [ ] **Step 4: Run to verify it passes**

Run: `uv run --directory /Users/71293/code/personal/orgs/paar nbdev-prepare`
Expected: `paar/grid.py` generated; all seven asserts pass.

- [ ] **Step 5: Commit**

```bash
git add nbs/06_grid.ipynb paar/grid.py paar/_modidx.py && git commit -m "feat(grid): numpy/pandas gridable detection + paging"
```

---

### Task 2: Add `is_grid` field to `VarInfo`

**Files:**
- Modify: `nbs/00_core.ipynb` (the `#| export` cell) → re-exports `paar/core.py`
- Test: cells in `nbs/00_core.ipynb`

**Interfaces:**
- Produces: `VarInfo` gains `is_grid:bool=False`, placed immediately after `is_container`.

- [ ] **Step 1: Add the failing test**

Add a NEW test cell to `nbs/00_core.ipynb` (leave existing tests unchanged):

```python
def test_varinfo_is_grid():
    assert VarInfo(name='a', type='ndarray', is_grid=True).is_grid is True
    assert VarInfo(name='x', type='int').is_grid is False
test_varinfo_is_grid()
```

- [ ] **Step 2: Run to verify it fails**

Run the cell. Expected: `TypeError: __init__() got an unexpected keyword argument 'is_grid'`.

- [ ] **Step 3: Modify the `VarInfo` export cell** — add `is_grid` right after `is_container`. Full cell:

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
    is_grid:bool=False
    is_error:bool=False
    path:str=''
    accessor:tuple=()
```

- [ ] **Step 4: Run to verify it passes**

Run: `uv run --directory /Users/71293/code/personal/orgs/paar nbdev-prepare`
Expected: `paar/core.py` regenerated; `test_varinfo_is_grid` and all existing core tests pass.

- [ ] **Step 5: Commit**

```bash
git add nbs/00_core.ipynb paar/core.py paar/_modidx.py && git commit -m "feat(core): add is_grid flag to VarInfo"
```

---

### Task 3: Wire `is_grid` + `grid_page` into `snapshot`

**Files:**
- Modify: `nbs/03_snapshot.ipynb` (the `#| export` cell + the test cell) → re-exports `paar/snapshot.py`
- Test: cells in `nbs/03_snapshot.ipynb`

**Interfaces:**
- Consumes: `paar.grid.is_gridable, array_page`; existing `paar.resolvers.resolve_for`; `paar.core.VarInfo`; `paar.reprs.*`.
- Produces: `_var_info` now sets `is_grid` and gates `is_container` off it; `grid_page(ns, accessor, roff=0, coff=0, rows=50, cols=50) -> dict|None` (walks the accessor to a gridable object, returns `array_page(...)` or `None`).

- [ ] **Step 1: Update the test cell** — keep all existing P2 tests; add numpy/pandas import + three tests. Add `grid_page` to the import line:

Change the first import line of the test cell to:
```python
from paar.snapshot import snapshot, expand, grid_page
import numpy as np, pandas as pd
```
Add these test functions (and include them in the runner list at the bottom):
```python
def test_is_grid_and_not_container():
    rows = snapshot({'arr': np.arange(4), 'df': pd.DataFrame({'a':[1]}), 'lst':[1,2]})
    by = {r.name:r for r in rows}
    assert by['arr'].is_grid and not by['arr'].is_container
    assert by['df'].is_grid and not by['df'].is_container
    assert by['lst'].is_container and not by['lst'].is_grid
def test_grid_page_walk():
    p = grid_page({'arr': np.arange(6).reshape(2,3)}, ('arr',))
    assert p['nrows']==2 and p['ncols']==3 and p['cells'][1]==['3','4','5']
def test_grid_page_non_gridable_none():
    assert grid_page({'n':5}, ('n',)) is None
```
Update the runner line at the end of the cell to also call `test_is_grid_and_not_container`, `test_grid_page_walk`, `test_grid_page_non_gridable_none`.

- [ ] **Step 2: Run to verify it fails**

Run the cell. Expected: `ImportError: cannot import name 'grid_page'`.

- [ ] **Step 3: Modify the `#| export` cell** — add the grid import, gate `is_container`, set `is_grid`, and add `grid_page`. Full cell:

```python
#| export
from paar.core import VarInfo
from paar.reprs import safe_repr, get_shape, get_dtype
from paar.resolvers import resolve_for
from paar.grid import is_gridable, array_page

MAX_CHILDREN = 100

def _var_info(name, v, accessor, path):
    "Build a VarInfo for one value at the given accessor/path."
    try:
        gridable = is_gridable(v)
        return VarInfo(name=name, type=type(v).__name__,
                       qualifier=getattr(type(v), '__module__', '') or '',
                       value=safe_repr(v), shape=get_shape(v), dtype=get_dtype(v),
                       is_container=(not gridable) and resolve_for(v) is not None,
                       is_grid=gridable, path=path, accessor=accessor)
    except Exception as e:
        return VarInfo(name=name, type='?', value=f'<error {e}>', is_error=True,
                       path=path, accessor=accessor)

def snapshot(ns, hidden=frozenset()):
    "namespace dict -> sorted list[VarInfo], skipping hidden and dunder names."
    keys = sorted(k for k in ns if k not in hidden and not k.startswith('__'))
    return [_var_info(k, ns[k], (k,), k) for k in keys]

def _walk(ns, accessor):
    "Resolve a positional accessor (name, *idxs) to (object, readable_path). Raises KeyError/IndexError on bad path."
    name, *idxs = accessor
    obj, path = ns[name], name
    for i in idxs:
        r = resolve_for(obj)
        if r is None: raise KeyError(accessor)
        _, step, obj = r.children(obj)[i]
        path += step
    return obj, path

def expand(ns, accessor):
    "Return one level of children of the value at accessor, as VarInfo (capped at MAX_CHILDREN)."
    try:
        obj, path = _walk(ns, tuple(accessor))
    except Exception:
        return []
    r = resolve_for(obj)
    if r is None: return []
    kids = r.children(obj)
    out = [_var_info(nm, child, tuple(accessor)+(i,), path+step)
           for i,(nm, step, child) in enumerate(kids[:MAX_CHILDREN])]
    if len(kids) > MAX_CHILDREN:
        out.append(VarInfo(name='…', type='', value=f'{len(kids)-MAX_CHILDREN} more',
                           path=path, accessor=tuple(accessor)))
    return out

def grid_page(ns, accessor, roff=0, coff=0, rows=50, cols=50):
    "Walk accessor to a gridable object and return a page dict, or None."
    try:
        obj, _ = _walk(ns, tuple(accessor))
    except Exception:
        return None
    if not is_gridable(obj): return None
    return array_page(obj, roff, coff, rows, cols)
```

- [ ] **Step 4: Run to verify it passes**

Run: `uv run --directory /Users/71293/code/personal/orgs/paar nbdev-prepare`
Expected: `paar/snapshot.py` regenerated; all P2 tests + the three new tests pass.

- [ ] **Step 5: Commit**

```bash
git add nbs/03_snapshot.ipynb paar/snapshot.py paar/_modidx.py && git commit -m "feat(snapshot): flag gridable vars and add grid_page"
```

---

### Task 4: Add `Bridge.grid`

**Files:**
- Modify: `nbs/04_bridge.ipynb` (the `#| export` cell + add a test cell) → re-exports `paar/bridge.py`
- Test: cells in `nbs/04_bridge.ipynb`

**Interfaces:**
- Consumes: `paar.snapshot.snapshot, expand, grid_page`.
- Produces: `Bridge.grid(self, accessor, roff=0, coff=0, rows=50, cols=50) -> dict|None` (delegates to `grid_page(_ns(), ...)`).

- [ ] **Step 1: Add the failing test**

Add a NEW test cell to `nbs/04_bridge.ipynb` (keep existing tests; reuse the `_IP` fake that takes a namespace dict):

```python
def test_bridge_grid():
    import numpy as np
    B.get_ipython = lambda: _IP({'arr': np.arange(6).reshape(2,3)})
    p = Bridge().grid(('arr',))
    assert p['nrows']==2 and p['ncols']==3
test_bridge_grid()
```

- [ ] **Step 2: Run to verify it fails**

Run the cell. Expected: `AttributeError: 'Bridge' object has no attribute 'grid'`.

- [ ] **Step 3: Modify the `#| export` cell** — import `grid_page` and add the method. Full cell:

```python
#| export
from paar.snapshot import snapshot, expand, grid_page
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
    def expand(self, accessor): return expand(_ns(), accessor)
    def grid(self, accessor, roff=0, coff=0, rows=50, cols=50): return grid_page(_ns(), accessor, roff, coff, rows, cols)

def on_change(cb):
    "Register cb() to fire after each cell execution; returns False outside IPython."
    ip = get_ipython()
    if not ip: return False
    ip.events.register('post_run_cell', lambda res: cb())
    return True
```

- [ ] **Step 4: Run to verify it passes**

Run: `uv run --directory /Users/71293/code/personal/orgs/paar nbdev-prepare`
Expected: `paar/bridge.py` regenerated; `test_bridge_grid` and all prior bridge tests pass.

- [ ] **Step 5: Commit**

```bash
git add nbs/04_bridge.ipynb paar/bridge.py paar/_modidx.py && git commit -m "feat(bridge): expose grid(accessor, roff, coff)"
```

---

### Task 5: Grid affordance + `/grid` route + Pico-enhanced CSS (`fasthtml`)

**Files:**
- Modify: `nbs/05_fasthtml.ipynb` (imports/CSS cell, rendering cell, routes cell, test cell) → re-exports `paar/fasthtml.py`
- Test: cells in `nbs/05_fasthtml.ipynb`

**Interfaces:**
- Consumes: `paar.bridge.Bridge` (now with `.grid`); `paar.core.VarInfo`.
- Produces: `_node` gains a `v.is_grid` branch rendering a `▦ grid` `<details>` whose summary GETs `/grid`; `_grid(page, acc) -> FT` and `_grid_nav(page, acc) -> FT`; route `GET /grid?accessor=<enc>&roff=<int>&coff=<int>`; enhanced `_css`.
- UNCHANGED: `_badges`, `_acc`, `_rows_div`, `_loader`, `/` `/rows` `/expand` routes, WS/`_broadcast`/`inspector`. `_node` stays the single render path.

- [ ] **Step 1: Update the test cell** — keep the P2 tests; add two grid tests. Add these functions and include them in the runner list:

```python
def test_node_grid_has_grid_link():
    html = to_xml(_node(VarInfo(name='arr', type='ndarray', is_grid=True, accessor=('arr',), path='arr')))
    assert '<details' in html and '/grid?accessor=' in html and 'grid' in html
    assert '/expand' not in html   # gridables are not tree-expanded
def test_grid_route_renders_table():
    import numpy as np
    B.get_ipython = lambda: _IP({'arr': np.arange(6).reshape(2,3)})
    r = TestClient(app).get('/grid?accessor=' + quote(json.dumps(['arr'])) + '&roff=0&coff=0')
    assert r.status_code==200 and '<table' in r.text and '5' in r.text  # last cell value present
```
Update the runner line to also call `test_node_grid_has_grid_link` and `test_grid_route_renders_table`.

- [ ] **Step 2: Run to verify it fails**

Run the cell. Expected: `AssertionError` in `test_node_grid_has_grid_link` (current `_node` renders a gridable as a plain scalar `Div`, no `/grid`).

- [ ] **Step 3: Replace the imports/CSS `#| export` cell** — enhance `_css` (Pico stays on by default). Full cell:

```python
#| export
import asyncio, threading, json
from urllib.parse import quote
from fasthtml.common import *
from fasthtml.jupyter import JupyUvi, HTMX
from paar.bridge import Bridge, on_change
from paar.core import VarInfo

_css = Style(
    '.paar-children{margin-left:1rem} '
    'summary{cursor:pointer} '
    '.paar-node{font-size:.9rem} '
    'code{white-space:pre} '
    '.paar-grid{max-height:360px;overflow:auto} '
    '.paar-grid table{font-size:.8rem;margin:0} '
    '.paar-grid th{position:sticky;top:0;background:var(--pico-background-color)} '
    '.paar-gridnav{margin:.25rem 0} '
    '.paar-page{margin-right:.5rem;cursor:pointer} '
    '.paar-grid-label{opacity:.6}')
bridge = Bridge()
app = FastHTML(exts='ws', hdrs=(_css,))   # ws ext + pico (default) + inspector css
_clients = []   # list[(loop, send)]
_clients_lock = threading.Lock()
_server = None
```

- [ ] **Step 4: Replace the rendering `#| export` cell** — add the `is_grid` branch to `_node` and the `_grid`/`_grid_nav` renderers. Full cell:

```python
#| export
def _badges(v:VarInfo):
    "shape/dtype summary string for a row."
    return ' '.join(x for x in [v.shape and f'shape={v.shape}',
                                v.dtype and f'dtype={v.dtype}'] if x)

def _acc(accessor):
    "URL-safe encoding of a positional accessor."
    return quote(json.dumps(list(accessor)), safe='')

def _node(v:VarInfo):
    "Render a tree node; containers expand, gridables open a paged grid, scalars are plain."
    b = _badges(v)
    head = Span(Strong(v.name), ' ', Small(v.type, title=v.qualifier), ' ',
                Code(v.value), (' ' + b) if b else '',
                cls=('error' if v.is_error else None))
    if v.is_grid:
        return Details(
            Summary(head, ' ', Span('▦ grid', cls='paar-grid-label'),
                    hx_get=f'/grid?accessor={_acc(v.accessor)}&roff=0&coff=0',
                    hx_target='next .paar-children', hx_swap='innerHTML', hx_trigger='click once'),
            Div(cls='paar-children'), cls='paar-node')
    if v.is_container:
        return Details(
            Summary(head, hx_get=f'/expand?accessor={_acc(v.accessor)}',
                    hx_target='next .paar-children', hx_swap='innerHTML', hx_trigger='click once'),
            Div(cls='paar-children'), cls='paar-node')
    return Div(head, cls='paar-node')

def _grid_nav(page, acc):
    "Prev/next paging controls + window info for a grid page."
    roff, coff = page['roff'], page['coff']
    npr, npc = len(page['index']), len(page['headers'])
    nrows, ncols = page['nrows'], page['ncols']
    def lnk(label, ro, co):
        return A(label, hx_get=f'/grid?accessor={_acc(acc)}&roff={ro}&coff={co}',
                 hx_target='closest .paar-children', hx_swap='innerHTML', cls='paar-page')
    ctrls = []
    if roff > 0: ctrls.append(lnk('◀ rows', max(0, roff-npr), coff))
    if roff+npr < nrows: ctrls.append(lnk('rows ▶', roff+npr, coff))
    if coff > 0: ctrls.append(lnk('◀ cols', roff, max(0, coff-npc)))
    if coff+npc < ncols: ctrls.append(lnk('cols ▶', roff, coff+npc))
    info = Small(f'rows {roff}-{roff+npr} of {nrows}, cols {coff}-{coff+npc} of {ncols}')
    return Div(info, ' ', *ctrls, cls='paar-gridnav')

def _grid(page, acc):
    "Render a grid page as a scrollable table with paging controls."
    if page is None: return Div('not gridable')
    hdr = Tr(Th(''), *[Th(h) for h in page['headers']])
    body = [Tr(Th(ix), *[Td(c) for c in row]) for ix, row in zip(page['index'], page['cells'])]
    return Div(Div(Table(Thead(hdr), Tbody(*body)), cls='paar-grid'),
               _grid_nav(page, acc))

def _rows_div():
    "The rendered snapshot as a node tree, wrapped in the #rows target."
    return Div(*[_node(v) for v in bridge.snapshot()], id='rows')

def _loader(oob=False):
    "A #rows div that immediately GETs /rows (used initially and as the WS nudge)."
    kw = dict(hx_get='/rows', hx_trigger='load', hx_swap='outerHTML')
    if oob: kw['hx_swap_oob']='true'
    return Div(id='rows', **kw)
```

- [ ] **Step 5: Add the `/grid` route** — in the routes cell, directly after `expand_route`. Leave `home`, `rows`, `expand_route`, `_drop`, `_conn`, `_disconn`, `live` unchanged:

```python
@app.get('/grid')
def grid_route(accessor:str, roff:int=0, coff:int=0):
    acc = tuple(json.loads(accessor))
    return _grid(bridge.grid(acc, roff, coff), acc)
```

- [ ] **Step 6: Run to verify it passes**

Run: `uv run --directory /Users/71293/code/personal/orgs/paar nbdev-prepare`
Expected: `paar/fasthtml.py` regenerated; all P2 tests + the two grid tests pass; prepare does not hang (index demo cells stay `#| eval: false`).

- [ ] **Step 7: Commit**

```bash
git add nbs/05_fasthtml.ipynb paar/fasthtml.py paar/_modidx.py && git commit -m "feat(fasthtml): paged grid viewer + Pico-enhanced styling"
```

---

### Task 6: Demo numpy/pandas grids

**Files:**
- Modify: `nbs/index.ipynb` (the `#| eval: false` demo `data` cell + a markdown note)

**Interfaces:**
- Consumes: `paar.fasthtml.inspector`.
- Produces: no exports; an updated demo with a numpy array + DataFrame.

- [ ] **Step 1: Update the demo data cell** — extend the `#| eval: false` cell to add an array and a DataFrame (keep `#| eval: false` first line):

```python
#| eval: false
import math, numpy as np, pandas as pd
data = {'a': 1, 'b': [1, 2, {'deep': [10, 20]}], 'c': 'hello', 'pi': math.pi}
arr = np.arange(12).reshape(3, 4)
df = pd.DataFrame({'x': range(5), 'y': list('abcde')})
```

- [ ] **Step 2: Add a markdown note after the demo cells:**

```
`arr` (numpy) and `df` (pandas) show a **▦ grid** toggle instead of a tree — click it to open a
scrollable, paged table (use the row/col ◀ ▶ controls for data larger than one page). Plain
containers still expand as a tree.
```

- [ ] **Step 3: Prepare & verify**

Run: `uv run --directory /Users/71293/code/personal/orgs/paar nbdev-prepare`
Expected: `Success.`, all module tests pass, demo cells skipped (no server start / no hang). `git status --short` shows no `.sesskey` files.

- [ ] **Step 4: Commit**

```bash
git add nbs/index.ipynb paar/_modidx.py && git commit -m "docs: demo numpy/pandas grid viewer"
```

---

## Self-Review

**Spec coverage (spec §"snapshot.py (03) — array_page" / §P3 "tabular viewer"):**
- gridable detection for numpy/pandas → Task 1 (`is_gridable`). ✓
- paged slice with headers/index/cells/totals (pydev `getArray`) → Task 1 (`array_page`), Task 3 (`grid_page` walk). ✓
- `is_grid` flag, mutually exclusive with `is_container` → Tasks 2, 3. ✓
- bridge passthrough → Task 4 (`Bridge.grid`). ✓
- frontend grid affordance + paged table + prev/next paging → Task 5 (`_node` grid branch, `/grid`, `_grid`/`_grid_nav`). ✓
- Pico-enhanced styling (user request) → Task 5 (`_css`). ✓
- optional numpy/pandas (runtime import-guarded; dev deps for tests) → Task 1 (import guards + `uv add --dev`). ✓
- no silent truncation (paging window + totals shown) → Task 5 (`_grid_nav`). ✓
- Deferred to later phases (correctly absent): str-providers/custom renderers (P4), JupyterLab (P5). 3-D+ arrays intentionally out of scope (flat row).

**Placeholder scan:** none — every step carries runnable code/commands.

**Type consistency:** the page dict keys (`headers`, `index`, `cells`, `nrows`, `ncols`, `roff`, `coff`) are identical across `array_page` (Task 1), `grid_page` (Task 3), `Bridge.grid` (Task 4), and `_grid`/`_grid_nav`/`grid_route` (Task 5). `is_grid` field name consistent across Tasks 2/3/5. `Bridge.grid(accessor, roff, coff, rows, cols)` signature matches its call in `grid_route`. `_acc`/`json.loads` accessor round-trip matches the P2 pattern.

**Note on mutual exclusivity:** numpy arrays and pandas objects never satisfy `resolve_for` (ndarray/DataFrame/Series aren't dict/list/set and lack a usable `__dict__` for `ObjectResolver`), but `_var_info` also explicitly gates `is_container` off `is_grid`, so a gridable value is guaranteed to render as a grid, never a tree.
