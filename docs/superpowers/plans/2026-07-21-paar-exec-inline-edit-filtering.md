# paar exec box + inline edit + filtering — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add runtime code execution (global + isolated), inline value editing, and name/type filtering to the paar live inspector.

**Architecture:** A new `paar/runtime.py` holds the two namespace-write ops (`run`, `set_value`) built on `IPython.run_cell` and the existing `snapshot._walk` lvalue paths. `snapshot.snapshot` gains filter params; `Bridge` forwards them and exposes `set_value`. `fasthtml` adds `/exec`, `/edit`, `/set` routes, an exec bar, click-to-edit values, and a client-side filter bar.

**Tech Stack:** nbdev, FastHTML/HTMX, IPython, fastcore.

## Global Constraints

- nbdev project: edit `nbs/*.ipynb`, never the exported `paar/*.py`. Each notebook has a `#| default_exp` cell, `#| export` cells, and test cells that `assert`. After edits run `uv run nbdev-prepare` (hyphen). Check `nbs/index.ipynb` for stale references before the final commit.
- Per-task validation command (fast): `cd /Users/71293/code/personal/orgs/paar && uv run nbdev_export && uv run nbdev_test --path nbs/<nb>.ipynb`
- Test cells import from the exported package (e.g. `from paar.runtime import run`) — always `nbdev_export` before `nbdev_test` so the import sees fresh code. Mock IPython by monkeypatching the module's `get_ipython`, as existing bridge tests do.
- Code style: fastcore idioms, short one-line docstrings, no decorative comments/box-drawing/alignment padding. Keep diffs minimal.
- Env: `uv` only (`uv run ...`). Never edit `paar/*.py` by hand.
- Working dir for all commands: `/Users/71293/code/personal/orgs/paar`.

---

### Task 1: `runtime.run()` — global execution + `ExecResult`

**Files:**
- Create: `nbs/08_runtime.ipynb` → exports `paar/runtime.py`
- Test: test cell inside `nbs/08_runtime.ipynb`

**Interfaces:**
- Consumes: `paar.core.VarInfo`; `paar.snapshot._var_info`; `paar.providers.value_str`.
- Produces: `ExecResult(ok:bool, result:VarInfo|None=None, stdout:str='', error:str|None=None)`; `run(code, scope='global') -> ExecResult`.

- [ ] **Step 1: Create the notebook with default_exp + markdown title**

Create `nbs/08_runtime.ipynb` with these cells in order. First a markdown cell:

```markdown
# runtime

> Execute code and edit values in the live kernel namespace
```

Then a code cell:

```python
#| default_exp runtime
```

Then a hidden showdoc cell:

```python
#| hide
from nbdev.showdoc import *
```

- [ ] **Step 2: Write the failing test cell**

Add a code cell (this is the test cell; it goes above the export cell, matching the paar convention):

```python
import paar.runtime as R
from paar.runtime import run, ExecResult

class _Shell:
    "Minimal IPython stand-in whose run_cell execs into a shared user_ns."
    def __init__(self): self.user_ns = {}
    def run_cell(self, code, store_history=True):
        import types
        ns = self.user_ns
        err = None; result = None
        try:
            block = compile(code, '<t>', 'exec')
            exec(block, ns)
            # emulate IPython binding `_` for a trailing expression
            import ast
            body = ast.parse(code).body
            if body and isinstance(body[-1], ast.Expr):
                result = eval(compile(ast.Expression(body[-1].value), '<t>', 'eval'), ns)
                ns['_'] = result
        except Exception as e:
            err = e
        return types.SimpleNamespace(result=result, error_in_exec=err,
                                     error_before_exec=None, success=err is None)

def test_run_global_assignment_mutates_ns():
    sh = _Shell(); R.get_ipython = lambda: sh
    r = run('x = 5')
    assert isinstance(r, ExecResult) and r.ok is True
    assert r.result is None and sh.user_ns['x'] == 5   # assignment: no result row
def test_run_global_expression_makes_result_row():
    sh = _Shell(); R.get_ipython = lambda: sh
    r = run('3 + 4')
    assert r.ok is True and r.result is not None
    assert r.result.name == 'result' and r.result.value == '7' and r.result.accessor == ('_',)
def test_run_global_modify_existing():
    sh = _Shell(); sh.user_ns['n'] = 1; R.get_ipython = lambda: sh
    run('n = n + 41'); assert sh.user_ns['n'] == 42
def test_run_error_is_captured():
    sh = _Shell(); R.get_ipython = lambda: sh
    r = run('1/0')
    assert r.ok is False and 'ZeroDivisionError' in r.error
def test_run_no_ipython():
    R.get_ipython = lambda: None
    r = run('x=1'); assert r.ok is False and 'no IPython' in r.error
for t in [test_run_global_assignment_mutates_ns, test_run_global_expression_makes_result_row,
          test_run_global_modify_existing, test_run_error_is_captured, test_run_no_ipython]: t()
```

- [ ] **Step 3: Run to verify it fails**

Run: `uv run nbdev_export && uv run nbdev_test --path nbs/08_runtime.ipynb`
Expected: FAIL — `ModuleNotFoundError: No module named 'paar.runtime'` (no export cell yet).

- [ ] **Step 4: Write the export cell**

Add a code cell (below the test cell):

```python
#| export
from __future__ import annotations
from dataclasses import dataclass
from paar.core import VarInfo
from paar.snapshot import _var_info
from paar.providers import value_str
from IPython.utils.capture import capture_output
try: from IPython import get_ipython
except Exception:
    def get_ipython(): return None

@dataclass
class ExecResult:
    "Outcome of running code: last value as an inspector row, captured stdout, and error text."
    ok: bool
    result: VarInfo|None = None
    stdout: str = ''
    error: str|None = None

def _fmt_err(res):
    "Formatted exception text from a run_cell result, or None."
    e = getattr(res, 'error_in_exec', None) or getattr(res, 'error_before_exec', None)
    return f'{type(e).__name__}: {e}' if e is not None else None

def run(code, scope='global'):
    "Run `code` in the kernel; scope='global' mutates user_ns, 'isolated' uses a scratch copy."
    ip = get_ipython()
    if ip is None: return ExecResult(ok=False, error='no IPython kernel')
    with capture_output() as cap:
        res = ip.run_cell(code, store_history=True)
    err = _fmt_err(res)
    result = _var_info('result', res.result, ('_',), '_') if res.result is not None else None
    return ExecResult(ok=bool(res.success) and err is None, result=result, stdout=cap.stdout, error=err)
```

- [ ] **Step 5: Add the export trailer cell**

Add a final code cell:

```python
#| hide
import nbdev; nbdev.nbdev_export()
```

- [ ] **Step 6: Run to verify it passes**

Run: `uv run nbdev_export && uv run nbdev_test --path nbs/08_runtime.ipynb`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add nbs/08_runtime.ipynb paar/runtime.py paar/_modidx.py
git commit -m "feat(runtime): run() global exec with ExecResult"
```

---

### Task 2: `runtime.run()` — isolated scope

**Files:**
- Modify: `nbs/08_runtime.ipynb` (export cell + test cell)

**Interfaces:**
- Consumes: same as Task 1.
- Produces: `run(code, scope='isolated')` returns `ExecResult` with a **flat** (non-expandable) result row and no namespace mutation; adds `_exec_capture(code, ns)`.

- [ ] **Step 1: Add failing isolated tests to the test cell**

Append these functions to the Task 1 test cell (before the runner list) and add them to the runner list:

```python
def test_run_isolated_no_leak():
    sh = _Shell(); sh.user_ns['seed'] = 10; R.get_ipython = lambda: sh
    r = run('y = seed + 1\ny', scope='isolated')
    assert r.ok is True and r.result is not None and r.result.value == '11'
    assert 'y' not in sh.user_ns                       # isolated: no leak into globals
    assert r.result.is_container is False and r.result.accessor == ()   # flat, non-expandable
def test_run_isolated_statement_only():
    sh = _Shell(); R.get_ipython = lambda: sh
    r = run('z = 1', scope='isolated')
    assert r.ok is True and r.result is None and 'z' not in sh.user_ns
```

Update the runner list line to include them:

```python
for t in [test_run_global_assignment_mutates_ns, test_run_global_expression_makes_result_row,
          test_run_global_modify_existing, test_run_error_is_captured, test_run_no_ipython,
          test_run_isolated_no_leak, test_run_isolated_statement_only]: t()
```

- [ ] **Step 2: Run to verify it fails**

Run: `uv run nbdev_export && uv run nbdev_test --path nbs/08_runtime.ipynb`
Expected: FAIL — `run('...', scope='isolated')` currently ignores scope and calls `ip.run_cell`, mutating `sh.user_ns` (so `'y' in sh.user_ns`) — assertion fails.

- [ ] **Step 3: Add `_exec_capture` and the isolated branch**

In the export cell, add `import ast` to the imports, add `_exec_capture`, and add the isolated branch at the top of `run`. The export cell becomes:

```python
#| export
from __future__ import annotations
from dataclasses import dataclass
import ast
from paar.core import VarInfo
from paar.snapshot import _var_info
from paar.providers import value_str
from IPython.utils.capture import capture_output
try: from IPython import get_ipython
except Exception:
    def get_ipython(): return None

@dataclass
class ExecResult:
    "Outcome of running code: last value as an inspector row, captured stdout, and error text."
    ok: bool
    result: VarInfo|None = None
    stdout: str = ''
    error: str|None = None

def _fmt_err(res):
    "Formatted exception text from a run_cell result, or None."
    e = getattr(res, 'error_in_exec', None) or getattr(res, 'error_before_exec', None)
    return f'{type(e).__name__}: {e}' if e is not None else None

def _exec_capture(code, ns):
    "Exec statements in ns; return the value of a trailing expression, else None."
    block = ast.parse(code)
    value = None
    if block.body and isinstance(block.body[-1], ast.Expr):
        last = ast.Expression(block.body.pop().value)
        exec(compile(block, '<paar>', 'exec'), ns)
        value = eval(compile(last, '<paar>', 'eval'), ns)
    else:
        exec(compile(block, '<paar>', 'exec'), ns)
    return value

def _flat_result(val):
    "A non-expandable result row (isolated mode leaves nothing reachable in user_ns)."
    return VarInfo(name='result', type=type(val).__name__, value=value_str(val))

def run(code, scope='global'):
    "Run `code` in the kernel; scope='global' mutates user_ns, 'isolated' uses a scratch copy."
    ip = get_ipython()
    if ip is None: return ExecResult(ok=False, error='no IPython kernel')
    if scope == 'isolated':
        ns = dict(ip.user_ns)
        try:
            with capture_output() as cap:
                val = _exec_capture(code, ns)
            return ExecResult(ok=True, result=None if val is None else _flat_result(val), stdout=cap.stdout)
        except Exception as e:
            return ExecResult(ok=False, error=f'{type(e).__name__}: {e}')
    with capture_output() as cap:
        res = ip.run_cell(code, store_history=True)
    err = _fmt_err(res)
    result = _var_info('result', res.result, ('_',), '_') if res.result is not None else None
    return ExecResult(ok=bool(res.success) and err is None, result=result, stdout=cap.stdout, error=err)
```

- [ ] **Step 4: Run to verify it passes**

Run: `uv run nbdev_export && uv run nbdev_test --path nbs/08_runtime.ipynb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add nbs/08_runtime.ipynb paar/runtime.py
git commit -m "feat(runtime): isolated scope via scratch namespace"
```

---

### Task 3: `runtime.set_value()` — inline edit write-path

**Files:**
- Modify: `nbs/08_runtime.ipynb` (export cell + test cell)

**Interfaces:**
- Consumes: `paar.snapshot._walk`.
- Produces: `set_value(accessor, expr) -> str|None` (error text, or `None` on success).

- [ ] **Step 1: Add failing tests to the test cell**

Add `set_value` to the import line at the top of the test cell:

```python
from paar.runtime import run, ExecResult, set_value
```

Append these test functions and add them to the runner list:

```python
def test_set_value_toplevel():
    sh = _Shell(); sh.user_ns['x'] = 1; R.get_ipython = lambda: sh
    assert set_value(('x',), '99') is None and sh.user_ns['x'] == 99
def test_set_value_dict_item():
    sh = _Shell(); sh.user_ns['d'] = {'k': 1}; R.get_ipython = lambda: sh
    assert set_value(('d', 0), '2') is None and sh.user_ns['d']['k'] == 2   # accessor ('d',0)->d['k']
def test_set_value_list_element():
    sh = _Shell(); sh.user_ns['L'] = [10, 20]; R.get_ipython = lambda: sh
    assert set_value(('L', 1), '21') is None and sh.user_ns['L'][1] == 21
def test_set_value_bad_target_errors():
    sh = _Shell(); sh.user_ns['s'] = {1, 2}; R.get_ipython = lambda: sh
    err = set_value(('s', 0), '9')          # set element path is display-only, not an lvalue
    assert err is not None and 's' in sh.user_ns   # error string, no crash
```

Runner list:

```python
for t in [test_run_global_assignment_mutates_ns, test_run_global_expression_makes_result_row,
          test_run_global_modify_existing, test_run_error_is_captured, test_run_no_ipython,
          test_run_isolated_no_leak, test_run_isolated_statement_only,
          test_set_value_toplevel, test_set_value_dict_item, test_set_value_list_element,
          test_set_value_bad_target_errors]: t()
```

- [ ] **Step 2: Run to verify it fails**

Run: `uv run nbdev_export && uv run nbdev_test --path nbs/08_runtime.ipynb`
Expected: FAIL — `ImportError: cannot import name 'set_value'`.

- [ ] **Step 3: Add `set_value` to the export cell**

Add `_walk` to the snapshot import and append `set_value`. Change the import line and add the function:

```python
from paar.snapshot import _var_info, _walk
```

```python
def set_value(accessor, expr):
    "Assign `expr` (evaluated in user_ns) to the lvalue at `accessor`; return error text or None."
    ip = get_ipython()
    if ip is None: return 'no IPython kernel'
    try: _, path = _walk(ip.user_ns, tuple(accessor))
    except Exception as e: return f'bad accessor: {e}'
    try:
        exec(f'{path} = ({expr})', ip.user_ns)
        return None
    except Exception as e:
        return f'{type(e).__name__}: {e}'
```

- [ ] **Step 4: Run to verify it passes**

Run: `uv run nbdev_export && uv run nbdev_test --path nbs/08_runtime.ipynb`
Expected: PASS.

Note: `_Shell.user_ns` is a plain dict, so `exec(f'{path} = (...)', ns)` writes to it exactly like a real `user_ns`.

- [ ] **Step 5: Commit**

```bash
git add nbs/08_runtime.ipynb paar/runtime.py
git commit -m "feat(runtime): set_value inline-edit write-path"
```

---

### Task 4: snapshot name/type filters

**Files:**
- Modify: `nbs/03_snapshot.ipynb` — export cell `a0000020` (the `snapshot` function) and test cell `a0000010`.

**Interfaces:**
- Produces: `snapshot(ns, hidden=frozenset(), name=None, typ=None) -> list[VarInfo]`.

- [ ] **Step 1: Add failing tests**

In test cell `a0000010`, append these functions and add them to the runner list at the bottom:

```python
def test_snapshot_filter_name():
    rows = snapshot({'alpha':1, 'beta':2, 'gamma':3}, name='a')
    assert [r.name for r in rows] == ['alpha', 'beta', 'gamma']   # 'a' substring, case-insensitive
    rows = snapshot({'alpha':1, 'beta':2}, name='ALPH')
    assert [r.name for r in rows] == ['alpha']
def test_snapshot_filter_type():
    rows = snapshot({'a':1, 'b':'x', 'c':2}, typ='int')
    assert [r.name for r in rows] == ['a', 'c']
def test_snapshot_filter_both():
    rows = snapshot({'a1':1, 'a2':'x', 'b1':3}, name='a', typ='int')
    assert [r.name for r in rows] == ['a1']
def test_snapshot_filter_no_match():
    assert snapshot({'a':1}, name='zzz') == []
```

Add `test_snapshot_filter_name, test_snapshot_filter_type, test_snapshot_filter_both, test_snapshot_filter_no_match` to the final `for t in [...]: t()` runner list.

- [ ] **Step 2: Run to verify it fails**

Run: `uv run nbdev_export && uv run nbdev_test --path nbs/03_snapshot.ipynb`
Expected: FAIL — `snapshot()` got an unexpected keyword argument `name`.

- [ ] **Step 3: Update `snapshot` in export cell `a0000020`**

Replace the `snapshot` function (leave the rest of the cell unchanged):

```python
def snapshot(ns, hidden=frozenset(), name=None, typ=None):
    "namespace dict -> sorted list[VarInfo], skipping hidden/dunder; optional name-substring & type-name filters."
    keys = sorted(k for k in ns if k not in hidden and not k.startswith('__'))
    if name: keys = [k for k in keys if name.lower() in k.lower()]
    if typ:  keys = [k for k in keys if ns[k].__class__.__name__.lower() == typ.lower()]
    return [_var_info(k, ns[k], (k,), k) for k in keys]
```

- [ ] **Step 4: Run to verify it passes**

Run: `uv run nbdev_export && uv run nbdev_test --path nbs/03_snapshot.ipynb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add nbs/03_snapshot.ipynb paar/snapshot.py
git commit -m "feat(snapshot): optional name/type filters"
```

---

### Task 5: Bridge filter params + set_value

**Files:**
- Modify: `nbs/04_bridge.ipynb` — export cell `a0000020` and test cell `5a671188`.

**Interfaces:**
- Consumes: `paar.snapshot.snapshot` (with `name`/`typ`); `paar.runtime.set_value`.
- Produces: `Bridge.snapshot(name=None, typ=None)`; `Bridge.set_value(accessor, expr)`.

- [ ] **Step 1: Add failing tests**

In test cell `5a671188` (the `test_bridge_expand` cell), append:

```python
def test_bridge_snapshot_filters():
    B.get_ipython = lambda: _IP({'alpha':1, 'beta':'x'})
    assert [r.name for r in Bridge().snapshot(name='a')] == ['alpha', 'beta']
    assert [r.name for r in Bridge().snapshot(typ='int')] == ['alpha']
def test_bridge_set_value():
    ip = _IP({'x': 1}); B.get_ipython = lambda: ip
    import paar.runtime as R; R.get_ipython = lambda: ip
    assert Bridge().set_value(('x',), '7') is None and ip.user_ns['x'] == 7
test_bridge_snapshot_filters(); test_bridge_set_value()
```

Note: `_IP` (defined in cell `a0000010`) already exposes `.user_ns`; `set_value` reads `get_ipython().user_ns`, so point `paar.runtime.get_ipython` at the same `_IP`.

- [ ] **Step 2: Run to verify it fails**

Run: `uv run nbdev_export && uv run nbdev_test --path nbs/04_bridge.ipynb`
Expected: FAIL — `Bridge.snapshot()` got an unexpected keyword `name` / `Bridge` has no `set_value`.

- [ ] **Step 3: Update the Bridge export cell `a0000020`**

Add the runtime import and the two Bridge changes. The export cell becomes:

```python
#| export
from paar.snapshot import snapshot, expand, grid_page, profile_view
from paar.runtime import set_value
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
    def snapshot(self, name=None, typ=None): return snapshot(_ns(), _hidden(), name, typ)
    def view(self, profile='standard'): return profile_view(_ns(), _hidden(), profile)
    def expand(self, accessor, offset=0): return expand(_ns(), accessor, offset)
    def grid(self, accessor, roff=0, coff=0, rows=50, cols=50): return grid_page(_ns(), accessor, roff, coff, rows, cols)
    def set_value(self, accessor, expr): return set_value(accessor, expr)

def on_change(cb):
    "Register cb() to fire after each cell execution; returns False outside IPython."
    ip = get_ipython()
    if not ip: return False
    ip.events.register('post_run_cell', lambda res: cb())
    return True
```

- [ ] **Step 4: Run to verify it passes**

Run: `uv run nbdev_export && uv run nbdev_test --path nbs/04_bridge.ipynb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add nbs/04_bridge.ipynb paar/bridge.py
git commit -m "feat(bridge): filter params + set_value"
```

---

### Task 6: fasthtml `/exec` route + exec bar

**Files:**
- Modify: `nbs/05_fasthtml.ipynb` — imports cell `cell-export-imports`, rows cell `cell-export-rows` (add `_exec_bar`, `_exec_out`), routes cell `cell-export-routes` (add `/exec`), home route, and test cell `cell-tests`.

**Interfaces:**
- Consumes: `paar.runtime.run`; existing `_node`, `_broadcast`, `_loader`.
- Produces: `POST /exec` (fields `code`, `scope`); `_exec_bar()`; `_exec_out(r)`.

- [ ] **Step 1: Add failing tests to `cell-tests`**

Append and register in the runner list:

```python
def test_exec_route_assignment():
    import paar.runtime as R, types
    class _Sh:
        def __init__(s): s.user_ns={}
        def run_cell(s, code, store_history=True):
            exec(code, s.user_ns)
            return types.SimpleNamespace(result=None, error_in_exec=None, error_before_exec=None, success=True)
    sh = _Sh(); R.get_ipython = lambda: sh
    r = TestClient(app).post('/exec', data={'code':'q = 5'})
    assert r.status_code == 200 and sh.user_ns['q'] == 5 and 'exec-out' in r.text
def test_exec_bar_renders():
    r = TestClient(app).get('/')
    assert 'hx-post="/exec"' in r.text and 'name="code"' in r.text and 'isolated' in r.text
```

Add `test_exec_route_assignment, test_exec_bar_renders` to the `for t in [...]` runner list.

- [ ] **Step 2: Run to verify it fails**

Run: `uv run nbdev_export && uv run nbdev_test --path nbs/05_fasthtml.ipynb`
Expected: FAIL — no `/exec` route (404) / exec bar not in `/`.

- [ ] **Step 3: Add the runtime import**

In `cell-export-imports`, add after the bridge import:

```python
from .runtime import run
```

- [ ] **Step 4: Add `_exec_bar` and `_exec_out` to `cell-export-rows`**

Append these two functions to the end of `cell-export-rows`:

```python
def _exec_bar():
    "Input to run code in the kernel; Enter runs it, checkbox toggles isolated scope."
    return Form(Input(name='code', placeholder='run code…', cls='paar-exec-input'),
                Label(Input(type='checkbox', name='scope', value='isolated'), ' isolated'),
                Button('run', type='submit'),
                hx_post='/exec', hx_target='#exec-out', hx_swap='outerHTML', cls='paar-exec')

def _exec_out(r):
    "Render an ExecResult as the #exec-out container: result row (if any) + stdout/error panels."
    parts = []
    if r.result is not None: parts.append(_node(r.result))
    if r.stdout: parts.append(Pre(r.stdout, cls='paar-out'))
    if r.error:  parts.append(Pre(r.error, cls='paar-out error'))
    return Div(*parts, id='exec-out')
```

`_exec_out` always carries `id='exec-out'` and the bar swaps it via `outerHTML`, so it is self-replacing and matches the `/set` OOB error swap (Task 7), which targets the same id.

- [ ] **Step 5: Add the `/exec` route and update `home` in `cell-export-routes`**

Add the route after `rows`:

```python
@rt('/exec')
def exec_route(code:str, scope:str='global'):
    return _exec_out(run(code, scope if scope in ('global','isolated') else 'global'))
```

Replace `home` to include the exec bar and output container:

```python
@rt('/')
def home():
    return Titled('paar', _profile_select(), _exec_bar(), Div(id='exec-out'),
                  Div(_loader(), hx_ext='ws', ws_connect='/live', id='paar'))
```

- [ ] **Step 6: Add exec CSS to cell `374fe0c4676e9b8`**

In the `_css = Style(...)` string, add these rules before the closing `)` (append to the last string literal, keeping the trailing space style):

```python
    '.paar-exec{display:flex;gap:.4rem;align-items:center;margin:0 0 .4rem} '
    '.paar-exec-input{flex:1;font-size:.8rem;padding:.15rem .3rem;margin:0} '
    '.paar-out{font-size:.75rem;white-space:pre-wrap;margin:.2rem 0} '
```

- [ ] **Step 7: Run to verify it passes**

Run: `uv run nbdev_export && uv run nbdev_test --path nbs/05_fasthtml.ipynb`
Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add nbs/05_fasthtml.ipynb paar/fasthtml.py
git commit -m "feat(fasthtml): /exec route + exec bar"
```

---

### Task 7: fasthtml inline edit (`/edit`, `/set`, clickable value)

**Files:**
- Modify: `nbs/05_fasthtml.ipynb` — rows cell `cell-export-rows` (`_head` value wrap), routes cell `cell-export-routes` (`/edit`, `/set`), test cell `cell-tests`.

**Interfaces:**
- Consumes: `bridge.set_value`; existing `_acc`, `_node`, `_rows_div`, `_broadcast`, `_loader`.
- Produces: `POST /edit` (field `accessor`) → edit form; `POST /set` (fields `accessor`, `expr`).

- [ ] **Step 1: Add failing tests to `cell-tests`**

Append and register in the runner list:

```python
def test_edit_route_returns_form():
    r = TestClient(app).post('/edit', data={'accessor': quote(json.dumps(['x']))})
    assert r.status_code == 200 and 'hx-post="/set"' in r.text and 'name="expr"' in r.text
def test_set_route_writes_and_refreshes():
    import paar.runtime as R
    class _IP2:
        def __init__(s): s.user_ns={'x':1}; s.user_ns_hidden=set()
        class events:
            @staticmethod
            def register(n, fn): pass
    ip = _IP2(); B.get_ipython = lambda: ip; R.get_ipython = lambda: ip
    F._clients.clear()
    r = TestClient(app).post('/set', data={'accessor': quote(json.dumps(['x'])), 'expr':'42'})
    assert r.status_code == 200 and ip.user_ns['x'] == 42 and 'id="rows"' in r.text
def test_set_route_error_inline():
    import paar.runtime as R
    class _IP3:
        def __init__(s): s.user_ns={'s':{1,2}}; s.user_ns_hidden=set()
        class events:
            @staticmethod
            def register(n, fn): pass
    ip = _IP3(); B.get_ipython = lambda: ip; R.get_ipython = lambda: ip
    r = TestClient(app).post('/set', data={'accessor': quote(json.dumps(['s',0])), 'expr':'9'})
    assert r.status_code == 200 and 'exec-out' in r.text   # error surfaced OOB, tree intact
def test_node_value_is_editable():
    html = to_xml(_node(VarInfo(name='x', type='int', value='1', accessor=('x',), path='x')))
    assert '/edit?accessor=' in html and 'paar-val' in html
```

Add the four test names to the `for t in [...]` runner list.

- [ ] **Step 2: Run to verify it fails**

Run: `uv run nbdev_export && uv run nbdev_test --path nbs/05_fasthtml.ipynb`
Expected: FAIL — no `/edit` / `/set` routes; value not clickable.

- [ ] **Step 3: Make the value clickable in `_head`**

Replace `_head` in `cell-export-rows` with (value wrapped in a clickable `.paar-val` span when the row is editable):

```python
def _head(v:VarInfo):
    "PyCharm-style row: name = {type: shape} value; value is click-to-edit when it has an accessor."
    typ = f'{v.type}: {v.shape}' if v.shape else v.type
    val = Code(v.value, cls='pv language-python')
    if v.accessor and v.more_offset is None and not v.is_error:
        val = Span(val, cls='paar-val', title='click to edit',
                   hx_post=f'/edit?accessor={_acc(v.accessor)}',
                   hx_target='this', hx_swap='innerHTML', hx_trigger='click')
    return Span(Span(v.name, cls='paar-name'), Span(' = ', cls='paar-eq'),
                Span('{', typ, '}', cls='paar-type', title=v.qualifier), ' ',
                val,
                Small(f' {v.dtype}', cls='paar-type') if v.dtype else '',
                cls=('error' if v.is_error else None))
```

- [ ] **Step 4: Add `/edit` and `/set` routes to `cell-export-routes`**

Add after `expand_route`:

```python
@rt('/edit')
def edit_route(accessor:str):
    "Return an inline edit form for the value at `accessor`."
    return Form(Input(name='expr', placeholder='new value', cls='paar-edit-input', autofocus=True),
                Input(type='hidden', name='accessor', value=accessor),
                hx_post='/set', hx_target='#rows', hx_swap='outerHTML', cls='paar-edit')

@rt('/set')
def set_route(accessor:str, expr:str):
    "Write `expr` into the namespace at `accessor`; refresh the tree, or surface an inline error."
    err = bridge.set_value(tuple(json.loads(accessor)), expr)
    if err:
        return _rows_div(), Div(err, id='exec-out', cls='paar-out error', hx_swap_oob='true')
    _broadcast(to_xml(_loader(oob=True)))
    return _rows_div()
```

- [ ] **Step 5: Add edit CSS to cell `374fe0c4676e9b8`**

Append to the `_css` string:

```python
    '.paar-val{cursor:pointer;border-bottom:1px dotted var(--pico-muted-border-color)} '
    '.paar-edit-input{font-size:.8rem;padding:.1rem .25rem;margin:0} '
```

- [ ] **Step 6: Run to verify it passes**

Run: `uv run nbdev_export && uv run nbdev_test --path nbs/05_fasthtml.ipynb`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add nbs/05_fasthtml.ipynb paar/fasthtml.py
git commit -m "feat(fasthtml): inline value editing via /edit + /set"
```

---

### Task 8: fasthtml filter bar + data attributes + client-side JS

**Files:**
- Modify: `nbs/05_fasthtml.ipynb` — hljs/script cell `374fe0c4676e9b8` (filter JS + onLoad hook + CSS), rows cell `cell-export-rows` (`data-*` attrs in `_node`, `_filter_bar`), home route, test cell `cell-tests`.

**Interfaces:**
- Consumes: existing `_node`, `_head`, `home`.
- Produces: `_filter_bar()`; `data-name`/`data-type` on nodes; JS `paarFilter()` / `paarSyncTypes()`.

- [ ] **Step 1: Add failing tests to `cell-tests`**

Append and register:

```python
def test_nodes_have_data_attrs():
    html = to_xml(_node(VarInfo(name='arr', type='ndarray', value='x', accessor=('arr',), path='arr')))
    assert 'data-name="arr"' in html and 'data-type="ndarray"' in html
def test_filter_bar_renders():
    r = TestClient(app).get('/')
    assert 'paar-filter-name' in r.text and 'paar-filter-type' in r.text
```

Add `test_nodes_have_data_attrs, test_filter_bar_renders` to the runner list.

- [ ] **Step 2: Run to verify it fails**

Run: `uv run nbdev_export && uv run nbdev_test --path nbs/05_fasthtml.ipynb`
Expected: FAIL — no `data-name`/`data-type`; no filter bar.

- [ ] **Step 3: Add `data-*` attributes in `_node`**

In `cell-export-rows`, add `data_name=v.name, data_type=v.type` to the outermost element of each non-sentinel `_node` branch. The `_node` function becomes:

```python
def _node(v:VarInfo):
    "Render a tree node: load-more sentinel; containers expand; gridables get a grid; scalars plain."
    if v.more_offset is not None: return _more(v)
    head = _head(v)
    data = dict(data_name=v.name, data_type=v.type)
    if v.is_container:
        body = [_grid_toggle(v)] if v.is_grid else []
        return Details(
            Summary(head, hx_get=f'/expand?accessor={_acc(v.accessor)}',
                    hx_target='next .paar-children', hx_swap='innerHTML', hx_trigger='click once'),
            *body,
            Div(cls='paar-children'), cls='paar-node', **data)
    if v.is_grid:
        return Details(
            Summary(head, ' ', Span('▦ grid', cls='paar-grid-label'),
                    hx_get=f'/grid?accessor={_acc(v.accessor)}&roff=0&coff=0',
                    hx_target='next .paar-gridbox', hx_swap='innerHTML', hx_trigger='click once'),
            Div(cls='paar-gridbox'), cls='paar-node', **data)
    return Div(head, cls='paar-node paar-leaf', **data)
```

- [ ] **Step 4: Add `_filter_bar` to `cell-export-rows`**

Append:

```python
def _filter_bar():
    "Name substring input + type dropdown; both filter the tree client-side."
    return Div(Input(name='fname', placeholder='filter name…', cls='paar-filter-name', oninput='paarFilter()'),
               Select(Option('all types', value=''), name='ftype', cls='paar-filter-type', onchange='paarFilter()'),
               cls='paar-filter')
```

- [ ] **Step 5: Add filter JS + onLoad hook + CSS to cell `374fe0c4676e9b8`**

Add a new `Script` to the `_hljs` tuple (append before its closing `)`):

```python
    Script("""
function paarFilter(){
  const nq=(document.querySelector('.paar-filter-name')?.value||'').toLowerCase();
  const tq=document.querySelector('.paar-filter-type')?.value||'';
  document.querySelectorAll('#rows .paar-node[data-name]').forEach(n=>{
    const nm=(n.getAttribute('data-name')||'').toLowerCase();
    const ty=n.getAttribute('data-type')||'';
    const show=(!nq||nm.includes(nq))&&(!tq||ty===tq);
    n.style.display=show?'':'none';
    if(show){const g=n.closest('details.paar-group'); if(g)g.open=true;}
  });
}
function paarSyncTypes(){
  const sel=document.querySelector('.paar-filter-type'); if(!sel)return;
  const cur=sel.value;
  const types=[...new Set([...document.querySelectorAll('#rows .paar-node[data-type]')]
    .map(n=>n.getAttribute('data-type')).filter(Boolean))].sort();
  sel.innerHTML='<option value="">all types</option>'+types.map(t=>'<option value="'+t+'">'+t+'</option>').join('');
  sel.value=cur; paarFilter();
}
htmx.onLoad(()=>{ if(window.paarSyncTypes) paarSyncTypes(); });
""")
```

Append to the `_css` string:

```python
    '.paar-filter{display:flex;gap:.4rem;margin:0 0 .4rem} '
    '.paar-filter-name{flex:1;font-size:.8rem;padding:.15rem .3rem;margin:0} '
    '.paar-filter-type{font-size:.8rem;margin:0} '
```

- [ ] **Step 6: Add the filter bar to `home`**

In `cell-export-routes`, update `home` to include `_filter_bar()` (keep the exec bar from Task 6):

```python
@rt('/')
def home():
    return Titled('paar', _profile_select(), _filter_bar(), _exec_bar(), Div(id='exec-out'),
                  Div(_loader(), hx_ext='ws', ws_connect='/live', id='paar'))
```

- [ ] **Step 7: Run to verify it passes**

Run: `uv run nbdev_export && uv run nbdev_test --path nbs/05_fasthtml.ipynb`
Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add nbs/05_fasthtml.ipynb paar/fasthtml.py
git commit -m "feat(fasthtml): client-side name/type filter bar"
```

---

### Task 9: Full prepare, docs check, final commit

**Files:**
- Modify: `nbs/index.ipynb` (only if it documents feature lists).

- [ ] **Step 1: Run the full nbdev prepare (export + all tests + clean + readme)**

Run: `uv run nbdev-prepare`
Expected: ends with `Success.` and no test failures across all notebooks.

- [ ] **Step 2: Check `index.ipynb` for stale references**

Open `nbs/index.ipynb`. If it lists inspector capabilities (viewing modes, toggles), add one line each for: the exec box (global/isolated), inline value editing (click a value), and the name/type filter bar. If `index.ipynb` does not enumerate features, make no change.

- [ ] **Step 3: If index changed, re-run prepare**

Run: `uv run nbdev-prepare`
Expected: `Success.`

- [ ] **Step 4: Final commit**

```bash
git add -A
git commit -m "docs: note exec, inline edit, and filtering in index"
```

(If Step 2 made no change, skip Step 4 — Task 9 has no commit of its own.)

---

## Self-Review Notes

- **Spec coverage:** Task 1 = global exec + ExecResult + result row; Task 2 = isolated scope (flat result, no leak); Task 3 = set_value write-path (incl. bad-target error); Task 4 = name/typ filters; Task 5 = Bridge filter params + set_value; Task 6 = /exec + exec bar + isolated toggle; Task 7 = inline edit /edit+/set + manual refresh + inline error; Task 8 = data-* attrs + filter bar + client-side JS; Task 9 = prepare + index. All spec sections mapped.
- **Type consistency:** `ExecResult(ok, result, stdout, error)`, `run(code, scope)`, `set_value(accessor, expr)`, `snapshot(ns, hidden, name, typ)`, `Bridge.snapshot(name, typ)`, `Bridge.set_value(accessor, expr)` used identically across tasks. Routes `/exec`, `/edit`, `/set` and helpers `_exec_bar/_exec_out/_filter_bar` named consistently.
- **Known limitation carried from spec:** exec runs on the server thread, not the kernel main thread (documented; not addressed in v1).
