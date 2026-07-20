# paar Live Inspector — P2 (container resolvers + lazy expand) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make containers (dict/list/tuple/set/object) expandable in the inspector — click a row to lazily load its children one level deep, recursively.

**Architecture:** A type-keyed resolver registry (`resolve_for`) decides whether a value is a container and, if so, yields its ordered children. `VarInfo` gains a **positional accessor** `(root_name, *child_indices)` that `expand(ns, accessor)` walks step-by-step from the root binding (no `eval`, robust for any key type), rebuilding a readable `path` for display. The FastHTML frontend switches from a flat table to a node-tree: container rows get a ▸ toggle that `GET /expand`s their children into a nested div.

**Tech Stack:** Python ≥3.10, nbdev, fastcore, IPython, python-fasthtml (htmx). Builds on merged P1.

## Global Constraints

- **nbdev workflow.** Code in `nbs/*.ipynb` cells marked `#| export`, exported to `paar/*.py`. Each notebook starts `#| default_exp <mod>`. Tests are plain (non-export) cells in the same notebook. Run `uv run --directory /Users/71293/code/personal/orgs/paar nbdev-prepare` (hyphen — NEVER `nbdev_prepare`) after edits. nbdev is in the venv (Python 3.13).
- **Always stage `paar/_modidx.py`** in each commit; after committing run `git status --short` and amend in any nbdev notebook reformat so the tree is clean (aside from untracked `.kosha/`, `.superpowers/`).
- **Commits enabled** on the P2 branch. End commit messages with:
  `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>`
- Python ≥3.10, package name `paar`, Apache-2.0.
- **Layer rule:** frontend(`fasthtml`) → bridge → serialize(`snapshot`→`resolvers`→`reprs`) → model(`core`); never import upward. Only `bridge` imports IPython; only `fasthtml` imports fasthtml.
- **Accessor is positional and URL-safe:** a tuple `(root_name:str, *child_indices:int)`. `VarInfo.path` stays human-readable for display/copy-paste; it is NEVER used for resolution. No `eval`.
- **One-level lazy:** `expand` resolves exactly one level of children per call (recursion happens via repeated clicks). Cap children at `MAX_CHILDREN = 100`, appending one truncation-indicator row when exceeded (no silent truncation).
- fastcore/fastai style: one-line docstrings, no decorative comments.

---

### Task 1: Resolver registry (`resolvers`)

**Files:**
- Create: `nbs/02_resolvers.ipynb` → exports `paar/resolvers.py`
- Test: cells in `nbs/02_resolvers.ipynb`

**Interfaces:**
- Consumes: nothing internal.
- Produces: `Resolver` base; `DictResolver`, `ListTupleResolver`, `SetResolver`, `ObjectResolver`; `resolve_for(v) -> Resolver|None`. Each resolver's `children(v) -> list[(name:str, step:str, child:Any)]` in stable order, where `name` is the display label for the child's name column and `step` is the readable path fragment appended to the parent path (`"['k']"`, `"[0]"`, `".attr"`).

- [ ] **Step 1: Write the failing tests**

New notebook `nbs/02_resolvers.ipynb`, first cell `#| default_exp resolvers`, then a test cell:

```python
from paar.resolvers import (Resolver, DictResolver, ListTupleResolver, SetResolver,
                            ObjectResolver, resolve_for)
class _Obj:
    def __init__(self): self.x=1; self.y='hi'

def test_dict_resolver():
    r = resolve_for({'a':1})
    assert isinstance(r, DictResolver)
    assert r.children({'a':1, 'b':2}) == [("'a'", "['a']", 1), ("'b'", "['b']", 2)]
def test_list_resolver():
    r = resolve_for([10,20])
    assert isinstance(r, ListTupleResolver)
    assert r.children([10,20]) == [('0','[0]',10), ('1','[1]',20)]
def test_tuple_uses_list_resolver():
    assert isinstance(resolve_for((1,2)), ListTupleResolver)
def test_set_resolver():
    r = resolve_for({1})
    assert isinstance(r, SetResolver)
    kids = r.children({7})
    assert kids == [('0','[0]',7)]
def test_object_resolver():
    r = resolve_for(_Obj())
    assert isinstance(r, ObjectResolver)
    assert r.children(_Obj()) == [('x','.x',1), ('y','.y','hi')]
def test_scalars_not_containers():
    assert resolve_for(5) is None and resolve_for('hi') is None and resolve_for(None) is None
for t in [test_dict_resolver,test_list_resolver,test_tuple_uses_list_resolver,
          test_set_resolver,test_object_resolver,test_scalars_not_containers]: t()
```

- [ ] **Step 2: Run to verify it fails**

Run the cell. Expected: `ImportError: cannot import name 'Resolver'`.

- [ ] **Step 3: Write minimal implementation**

Export cell:

```python
#| export
class Resolver:
    "Decides if a value is a container and yields its ordered children."
    def can(self, v): raise NotImplementedError
    def children(self, v): raise NotImplementedError   # -> list[(name, step, child)]

class DictResolver(Resolver):
    def can(self, v): return isinstance(v, dict)
    def children(self, v): return [(repr(k), f'[{k!r}]', val) for k,val in v.items()]

class ListTupleResolver(Resolver):
    def can(self, v): return isinstance(v, (list, tuple))
    def children(self, v): return [(str(i), f'[{i}]', x) for i,x in enumerate(v)]

class SetResolver(Resolver):
    def can(self, v): return isinstance(v, (set, frozenset))
    def children(self, v): return [(str(i), f'[{i}]', x) for i,x in enumerate(v)]

class ObjectResolver(Resolver):
    def can(self, v): return hasattr(v, '__dict__') and bool(vars(v))
    def children(self, v): return [(k, f'.{k}', val) for k,val in sorted(vars(v).items())]

_RESOLVERS = [DictResolver(), ListTupleResolver(), SetResolver(), ObjectResolver()]

def resolve_for(v):
    "First resolver that handles v, else None (None => not a container)."
    for r in _RESOLVERS:
        try:
            if r.can(v): return r
        except Exception: pass
    return None
```

- [ ] **Step 4: Run to verify it passes**

Run: `uv run --directory /Users/71293/code/personal/orgs/paar nbdev-prepare`
Expected: `paar/resolvers.py` generated; all six asserts pass.

- [ ] **Step 5: Commit**

```bash
git add nbs/02_resolvers.ipynb paar/resolvers.py paar/_modidx.py && git commit -m "feat(resolvers): container resolver registry"
```

---

### Task 2: Add `accessor` field to `VarInfo`

**Files:**
- Modify: `nbs/00_core.ipynb` (the `#| export` cell defining `VarInfo`) → re-exports `paar/core.py`
- Test: cells in `nbs/00_core.ipynb`

**Interfaces:**
- Produces: `VarInfo` gains `accessor:tuple=()` (positional path `(root_name, *indices)`), placed after `path`.

- [ ] **Step 1: Add the failing test**

Add a new test cell to `nbs/00_core.ipynb` (leave the existing `test_varinfo_defaults` cell as-is):

```python
def test_varinfo_accessor():
    v = VarInfo(name='d', type='dict', accessor=('d',))
    assert v.accessor == ('d',)
    assert VarInfo(name='x', type='int').accessor == ()   # default empty
test_varinfo_accessor()
```

- [ ] **Step 2: Run to verify it fails**

Run the cell. Expected: `TypeError: __init__() got an unexpected keyword argument 'accessor'`.

- [ ] **Step 3: Modify the `VarInfo` export cell**

In the `#| export` cell in `nbs/00_core.ipynb`, add the `accessor` field immediately after `path`. The full class becomes:

```python
#| export
from __future__ import annotations
from dataclasses import dataclass, field

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
    accessor:tuple=()
```

- [ ] **Step 4: Run to verify it passes**

Run: `uv run --directory /Users/71293/code/personal/orgs/paar nbdev-prepare`
Expected: `paar/core.py` regenerated; `test_varinfo_accessor` and the existing `test_varinfo_defaults` both pass.

- [ ] **Step 5: Commit**

```bash
git add nbs/00_core.ipynb paar/core.py paar/_modidx.py && git commit -m "feat(core): add positional accessor to VarInfo"
```

---

### Task 3: Wire containers + `expand` into `snapshot`

**Files:**
- Modify: `nbs/03_snapshot.ipynb` (the `#| export` cell) → re-exports `paar/snapshot.py`
- Test: cells in `nbs/03_snapshot.ipynb`

**Interfaces:**
- Consumes: `paar.core.VarInfo`; `paar.reprs.safe_repr/get_shape/get_dtype`; `paar.resolvers.resolve_for`.
- Produces: `snapshot(ns, hidden=frozenset()) -> list[VarInfo]` now sets `is_container` and `accessor=(name,)`; `expand(ns, accessor) -> list[VarInfo]` returns one level of children (each with extended accessor + readable path), capped at `MAX_CHILDREN=100` with a trailing indicator row; helper `_var_info(name, v, accessor, path)`.

- [ ] **Step 1: Write the failing tests**

Replace the existing test cell in `nbs/03_snapshot.ipynb` with this (keeps the P1 assertions, adds P2 ones):

```python
from paar.snapshot import snapshot, expand
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
def test_container_flag_and_accessor():
    rows = snapshot({'d':{'a':1}, 'n':5})
    d, n = rows[0], rows[1]
    assert d.is_container is True and d.accessor==('d',)
    assert n.is_container is False and n.accessor==('n',)
def test_expand_dict_then_list():
    ns = {'d': {'x': 1, 'y': [10,20]}}
    kids = expand(ns, ('d',))
    assert [k.name for k in kids]==["'x'", "'y'"]
    assert kids[1].is_container and kids[1].accessor==('d',1) and kids[1].path=="d['y']"
    gk = expand(ns, ('d',1))
    assert [g.name for g in gk]==['0','1'] and gk[0].value=='10' and gk[0].path=="d['y'][0]"
def test_expand_truncates():
    kids = expand({'L': list(range(250))}, ('L',))
    assert len(kids)==101 and kids[-1].is_error is False and 'more' in kids[-1].value
def test_expand_non_container_empty():
    assert expand({'n':5}, ('n',))==[]
for t in [test_sorted_and_fields,test_skips_hidden_and_dunder,test_shape_populated,
          test_container_flag_and_accessor,test_expand_dict_then_list,
          test_expand_truncates,test_expand_non_container_empty]: t()
```

- [ ] **Step 2: Run to verify it fails**

Run the cell. Expected: `ImportError: cannot import name 'expand'`.

- [ ] **Step 3: Replace the `#| export` cell**

```python
#| export
from .core import VarInfo
from .reprs import safe_repr, get_shape, get_dtype
from .resolvers import resolve_for

MAX_CHILDREN = 100

def _var_info(name, v, accessor, path):
    "Build a VarInfo for one value at the given accessor/path."
    try:
        return VarInfo(name=name, type=type(v).__name__,
                       qualifier=getattr(type(v), '__module__', '') or '',
                       value=safe_repr(v), shape=get_shape(v), dtype=get_dtype(v),
                       is_container=resolve_for(v) is not None,
                       path=path, accessor=accessor)
    except Exception as e:
        return VarInfo(name=name, type='?', value=f'<error {e}>', is_error=True,
                       path=path, accessor=accessor)

def snapshot(ns, hidden=frozenset()):
    "namespace dict -> sorted list[VarInfo], skipping hidden and dunder names."
    keys = sorted(k for k in ns if k not in hidden and not k.startswith('__'))
    return [_var_info(k, ns[k], (k,), k) for k in keys]

def _walk(ns, accessor):
    "Resolve a positional accessor (name, *idxs) to (object, readable_path). Raises on bad path."
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
```

- [ ] **Step 4: Run to verify it passes**

Run: `uv run --directory /Users/71293/code/personal/orgs/paar nbdev-prepare`
Expected: `paar/snapshot.py` regenerated; all seven asserts pass.

- [ ] **Step 5: Commit**

```bash
git add nbs/03_snapshot.ipynb paar/snapshot.py paar/_modidx.py && git commit -m "feat(snapshot): container flags, accessors, and lazy expand"
```

---

### Task 4: Add `Bridge.expand`

**Files:**
- Modify: `nbs/04_bridge.ipynb` (the `#| export` cell) → re-exports `paar/bridge.py`
- Test: cells in `nbs/04_bridge.ipynb`

**Interfaces:**
- Consumes: `paar.snapshot.snapshot, expand`.
- Produces: `Bridge.expand(self, accessor) -> list[VarInfo]` (delegates to `expand(_ns(), accessor)`).

- [ ] **Step 1: Add the failing test**

Add a new test cell to `nbs/04_bridge.ipynb` (keep the existing tests). The existing test cell already defines `_IP`/`_Events` and sets `B.get_ipython`; this cell reuses that pattern:

```python
def test_bridge_expand():
    B.get_ipython = lambda: _IP({'d': {'x': 1, 'y': 2}})
    kids = Bridge().expand(('d',))
    assert [k.name for k in kids]==["'x'", "'y'"] and kids[0].value=='1'
test_bridge_expand()
```

- [ ] **Step 2: Run to verify it fails**

Run the cell. Expected: `AttributeError: 'Bridge' object has no attribute 'expand'`.

- [ ] **Step 3: Modify the `#| export` cell**

Update the imports line and add the `expand` method. The full cell becomes:

```python
#| export
from .snapshot import snapshot, expand
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

def on_change(cb):
    "Register cb() to fire after each cell execution; returns False outside IPython."
    ip = get_ipython()
    if not ip: return False
    ip.events.register('post_run_cell', lambda res: cb())
    return True
```

- [ ] **Step 4: Run to verify it passes**

Run: `uv run --directory /Users/71293/code/personal/orgs/paar nbdev-prepare`
Expected: `paar/bridge.py` regenerated; `test_bridge_expand` and all prior bridge tests pass.

- [ ] **Step 5: Commit**

```bash
git add nbs/04_bridge.ipynb paar/bridge.py paar/_modidx.py && git commit -m "feat(bridge): expose expand(accessor)"
```

---

### Task 5: Node-tree rendering + `/expand` route (`fasthtml`)

**Files:**
- Modify: `nbs/05_fasthtml.ipynb` (the export cells + the test cell) → re-exports `paar/fasthtml.py`
- Test: cells in `nbs/05_fasthtml.ipynb`

**Interfaces:**
- Consumes: `paar.bridge.Bridge, on_change`; `paar.core.VarInfo`.
- Produces: `_badges(v) -> str`; `_node(v:VarInfo) -> FT` (a node div; container nodes carry a ▸ toggle that `GET /expand`s into an adjacent `.paar-children` div); `_rows_div() -> FT` (id=`rows`, the snapshot as a node tree); `_acc(accessor) -> str` (URL-safe encoded accessor) and route `GET /expand?accessor=<enc>`. UNCHANGED external contract: `app`, `bridge`, `_loader`, `_broadcast`, `_clients`, `_clients_lock`, `_drop`, `_conn`, `_disconn`, `inspector`, routes `/` and `/rows` and WS `/live`. `_row`/table rendering is REPLACED by `_node`/tree.

- [ ] **Step 1: Replace the test cell**

Replace the existing test cell in `nbs/05_fasthtml.ipynb` with this (drops the table-specific `test_row_renders_fields`, keeps the rest, adds node/expand tests):

```python
import json
from urllib.parse import quote
from starlette.testclient import TestClient
import paar.bridge as B, paar.fasthtml as F
from paar.fasthtml import app, _node, _broadcast, _acc
from paar.core import VarInfo

class _IP:
    def __init__(self, ns): self.user_ns=ns; self.user_ns_hidden=set()
    class events:
        @staticmethod
        def register(n, fn): pass

def test_node_scalar_no_toggle():
    html = to_xml(_node(VarInfo(name='n', type='int', value='5', accessor=('n',), path='n')))
    assert 'n' in html and 'int' in html and '5' in html and '/expand' not in html
def test_node_container_has_toggle():
    html = to_xml(_node(VarInfo(name='d', type='dict', value='{...}',
                                is_container=True, accessor=('d',), path='d')))
    assert 'paar-children' in html and '/expand?accessor=' in html
def test_rows_route_lists_vars():
    B.get_ipython = lambda: _IP({'alpha': 123})
    r = TestClient(app).get('/rows')
    assert r.status_code==200 and 'alpha' in r.text and '123' in r.text and 'id="rows"' in r.text
def test_home_wires_ws_and_loader():
    r = TestClient(app).get('/')
    assert r.status_code==200
    assert 'ws-connect="/live"' in r.text and 'hx-ext="ws"' in r.text
    assert 'id="rows"' in r.text and '/rows' in r.text
def test_expand_route_returns_children():
    B.get_ipython = lambda: _IP({'d': {'x': 1}})
    r = TestClient(app).get('/expand?accessor=' + quote(json.dumps(['d'])))
    assert r.status_code==200 and 'x' in r.text and '1' in r.text
def test_broadcast_drops_dead_clients():
    F._clients.clear()
    def _bad_send(_): raise RuntimeError('dead')
    F._clients.append(('not-a-loop', _bad_send))
    _broadcast('x')
    assert F._clients==[]
for t in [test_node_scalar_no_toggle,test_node_container_has_toggle,test_rows_route_lists_vars,
          test_home_wires_ws_and_loader,test_expand_route_returns_children,test_broadcast_drops_dead_clients]: t()
```

- [ ] **Step 2: Run to verify it fails**

Run the cell. Expected: `ImportError: cannot import name '_node'` (or `_acc`).

- [ ] **Step 3: Replace the imports cell**

Replace the `#| export` imports cell so it adds `json`/`quote` and a CSS header for indentation:

```python
#| export
import asyncio, threading, json
from urllib.parse import quote
from fasthtml.common import *
from fasthtml.jupyter import JupyUvi, HTMX
from .bridge import Bridge, on_change
from .core import VarInfo

_css = Style('.paar-children{margin-left:1rem} .paar-toggle{cursor:pointer;user-select:none}')
bridge = Bridge()
app = FastHTML(exts='ws', hdrs=(_css,))   # ws ext + pico (default) + indentation css
_clients = []   # list[(loop, send)]
_clients_lock = threading.Lock()
_server = None
```

- [ ] **Step 4: Replace the rendering cell (was `_row`/`_rows_div`)**

Replace the cell that defined `_row`, `_rows_div`, `_loader` with:

```python
#| export
def _badges(v:VarInfo):
    "shape/dtype summary string for a row."
    return ' '.join(x for x in [v.shape and f'shape={v.shape}',
                                v.dtype and f'dtype={v.dtype}'] if x)

def _acc(accessor):
    "URL-safe encoding of a positional accessor."
    return quote(json.dumps(list(accessor)))

def _node(v:VarInfo):
    "Render one VarInfo as a tree node; containers get a ▸ toggle that lazily loads children."
    head = Span(Strong(v.name), ' ', Small(v.type, title=v.qualifier), ' ',
                Code(v.value), (' ' + _badges(v)) if _badges(v) else '',
                cls=('error' if v.is_error else None))
    if v.is_container:
        toggle = Span('▸', cls='paar-toggle', hx_get=f'/expand?accessor={_acc(v.accessor)}',
                      hx_target='next .paar-children', hx_swap='innerHTML')
        return Div(Div(toggle, ' ', head), Div(cls='paar-children'), cls='paar-node')
    return Div(head, cls='paar-node')

def _rows_div():
    "The rendered snapshot as a node tree, wrapped in the #rows target."
    return Div(*[_node(v) for v in bridge.snapshot()], id='rows')

def _loader(oob=False):
    "A #rows div that immediately GETs /rows (used initially and as the WS nudge)."
    kw = dict(hx_get='/rows', hx_trigger='load', hx_swap='outerHTML')
    if oob: kw['hx_swap_oob']='true'
    return Div(id='rows', **kw)
```

- [ ] **Step 5: Add the `/expand` route**

In the routes cell (the one with `@app.get('/')` and `@app.get('/rows')`), add the expand route directly after `rows`:

```python
@app.get('/expand')
def expand_route(accessor:str):
    return Div(*[_node(v) for v in bridge.expand(tuple(json.loads(accessor)))])
```

(Leave `home`, `rows`, `_drop`, `_conn`, `_disconn`, `live`, `_broadcast`, `inspector` unchanged.)

- [ ] **Step 6: Run to verify it passes**

Run: `uv run --directory /Users/71293/code/personal/orgs/paar nbdev-prepare`
Expected: `paar/fasthtml.py` regenerated; all six asserts pass. Confirm `nbdev-prepare` does not hang (index demo cells stay `#| eval: false`).

- [ ] **Step 7: Commit**

```bash
git add nbs/05_fasthtml.ipynb paar/fasthtml.py paar/_modidx.py && git commit -m "feat(fasthtml): node-tree rendering with lazy expand"
```

---

### Task 6: Update the live demo for expansion

**Files:**
- Modify: `nbs/index.ipynb` (the demo `data` cell + a note)

**Interfaces:**
- Consumes: `paar.fasthtml.inspector`.
- Produces: no exports; an updated demo showing nested containers.

- [ ] **Step 1: Update the demo data cell**

In `nbs/index.ipynb`, change the `#| eval: false` data cell so it includes a nested structure worth expanding:

```python
#| eval: false
import math
data = {'a': 1, 'b': [1, 2, {'deep': [10, 20]}], 'c': 'hello', 'pi': math.pi}
```

- [ ] **Step 2: Add an expansion note (markdown cell) after the demo**

```
Click the ▸ toggle on any container row (dict / list / tuple / set / object) to load its
children one level at a time. Nested containers keep their own toggles, so you can drill in
as deep as the data goes — e.g. `data` → `'b'` → `[2]` → `'deep'`.
```

- [ ] **Step 3: Prepare & verify**

Run: `uv run --directory /Users/71293/code/personal/orgs/paar nbdev-prepare`
Expected: `Success.`, all module tests pass, demo cells skipped (no server start / no hang). Confirm `git status --short` shows no `.sesskey` files (still ignored).

- [ ] **Step 4: Commit**

```bash
git add nbs/index.ipynb && git commit -m "docs: demo nested container expansion"
```

---

## Self-Review

**Spec coverage (P2 scope — spec §"resolvers.py (02)" and §"snapshot.py (03) — expand"):**
- resolver registry + protocol (`can`/`children`) → Task 1. ✓
- dict/list/tuple/set/object resolvers, ordered children → Task 1. ✓
- `resolve_for` returns None ⇒ not a container ⇒ `is_container` False → Tasks 1, 3. ✓
- lazy `expand(ns, path)` one level → Task 3 (`expand(ns, accessor)`). ✓
- readable path for display, structured (positional) accessor for resolution, no eval → Tasks 2, 3 (decided in this session's brainstorm; supersedes the spec's opaque-token wording). ✓
- no silent truncation (MAX_CHILDREN + indicator) → Task 3. ✓
- frontend click-to-drill (expand arrow, nested target) → Task 5. ✓
- bridge `expand` passthrough (spec §bridge `getVariable` equivalent) → Task 4. ✓
- Deferred to later phases (correctly absent): numpy/pandas resolvers + array grid (P3), str-providers/custom renderers (P4), JupyterLab (P5). numpy/pandas are NOT containers in P2 — they fall through `resolve_for` to `None` (no `__dict__` children of interest) and render as flat rows until P3.

**Placeholder scan:** none — every step carries runnable code/commands.

**Type consistency:** `accessor` is a `tuple` everywhere (`VarInfo.accessor`, `snapshot`/`expand` args, `_acc`, `/expand` route decodes via `tuple(json.loads(...))`); `resolver.children` returns `(name, step, child)` triples consistently across Tasks 1/3; `_node`/`_rows_div`/`_acc` names match between Task 5's export and test cells; `Bridge.expand(accessor)` signature matches its use in Task 5's `/expand` route.

**Note on empty containers:** an empty dict/list/set is still `is_container=True` (shows a ▸ that expands to nothing); an object with no instance attributes is NOT a container (`ObjectResolver.can` requires non-empty `vars(v)`). This is intentional and mirrors pydev's behavior for builtin containers.
