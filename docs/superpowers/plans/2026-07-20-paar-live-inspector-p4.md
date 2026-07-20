# paar Live Inspector — P4 (str-providers / custom value renderers) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let the inspector show a nicer value string for a type than its raw `repr`, via a registry of type→string providers with built-ins for common types and a public API for user types.

**Architecture:** A new serialize-layer leaf `providers.py` holds an ordered registry (`isinstance` match, latest-registered wins), a `str_provider` decorator + `register_str_provider`, built-in providers (stdlib + guarded numpy/pandas), and `value_str(v)` (a registered provider else `safe_repr`). `snapshot._var_info` swaps its one `safe_repr(v)` call for `value_str(v)`. No frontend changes.

**Tech Stack:** Python ≥3.10, nbdev, fastcore, numpy/pandas (optional, guarded). Builds on merged P1+P2+P3.

## Global Constraints

- **nbdev workflow.** Code in `nbs/*.ipynb` cells marked `#| export`, exported to `paar/*.py`. Each notebook starts `#| default_exp <mod>`. Tests are plain (non-export) cells. Use ABSOLUTE imports in the notebook (`from paar.reprs import safe_repr`); nbdev emits relative. Run `uv run --directory /Users/71293/code/personal/orgs/paar nbdev-prepare` (hyphen — NEVER `nbdev_prepare`). nbdev is in the venv (Python 3.13).
- **Commit with `--no-verify`.** The pre-commit hook re-runs the full `nbdev-prepare` and times out; each task runs `nbdev-prepare` explicitly, then commits with `git commit --no-verify`.
- **Always stage `paar/_modidx.py`**; after committing run `git status --short` and amend in any nbdev reformat so the tree is clean (aside from untracked `.kosha/`, `.superpowers/`).
- **Commits enabled** on branch `feat/live-inspector-p4`. End commit messages with:
  `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>`
- Python ≥3.10, package name `paar`.
- **Layer rule:** frontend → bridge → serialize(`snapshot`→`{resolvers, grid, providers}`→`reprs`) → model. `providers` imports ONLY `paar.reprs` (+ numpy/pandas, guarded). Only `snapshot` consumes `providers`.
- **Registry semantics:** `isinstance` match; iterate registrations latest-first (`reversed`), first match wins; a provider that raises → fall back to `safe_repr`; missing numpy/pandas → skip those (guarded) registrations.
- fastcore/fastai style: one-line docstrings, no decorative comments.

---

### Task 1: Provider registry + built-ins + top-level re-export (`providers`)

**Files:**
- Create: `nbs/07_providers.ipynb` → exports `paar/providers.py`
- Modify: `paar/__init__.py` (add the top-level re-export)
- Test: cells in `nbs/07_providers.ipynb`

**Interfaces:**
- Consumes: `paar.reprs.safe_repr`.
- Produces: `register_str_provider(typ, fn) -> None`; `str_provider(typ)` (decorator returning the fn); `provided_str(v) -> str|None`; `value_str(v) -> str`. Built-in providers registered at import for `pathlib.Path`, `datetime.datetime/date/time`, `enum.Enum`, `uuid.UUID`, `bytes`, `bytearray`, and (guarded) `numpy.ndarray`, `pandas.DataFrame`, `pandas.Series`. Top-level: `paar.str_provider`, `paar.register_str_provider`.

- [ ] **Step 1: Write the failing tests**

New notebook `nbs/07_providers.ipynb`, first cell `#| default_exp providers`, then a test cell:

```python
import pathlib, datetime, enum, uuid
import numpy as np, pandas as pd
from paar.providers import str_provider, register_str_provider, provided_str, value_str

class Color(enum.Enum): RED=1; GREEN=2
class MyInt(enum.IntEnum): A=1

def test_path():        assert value_str(pathlib.Path('/tmp/x'))==str(pathlib.Path('/tmp/x'))
def test_datetime():    assert value_str(datetime.datetime(2020,1,2,3,4,5))=='2020-01-02T03:04:05'
def test_date():        assert value_str(datetime.date(2020,1,2))=='2020-01-02'
def test_enum():        assert value_str(Color.RED)=='Color.RED'
def test_intenum_sub(): assert value_str(MyInt.A)=='MyInt.A'
def test_uuid():
    u=uuid.UUID('12345678-1234-5678-1234-567812345678'); assert value_str(u)==str(u)
def test_bytes_short(): assert value_str(b'abc')=='616263 (3 bytes)'
def test_bytes_long():
    s=value_str(bytes(range(20))); assert s.endswith('(20 bytes)') and '…' in s
def test_ndarray():     assert value_str(np.arange(6).reshape(2,3))=='ndarray shape=(2, 3) int64'
def test_dataframe():   assert value_str(pd.DataFrame({'a':[1],'b':[2]}))=='DataFrame [1×2]'
def test_series():      assert value_str(pd.Series([1,2,3]))=='Series [3] int64'
def test_unregistered_falls_back(): assert value_str(123)=='123'
def test_latest_registered_wins():
    class Widget: pass
    register_str_provider(Widget, lambda w: 'A')
    register_str_provider(Widget, lambda w: 'B')
    assert value_str(Widget())=='B'
def test_throwing_provider_falls_back():
    class Boom: pass
    register_str_provider(Boom, lambda x: 1/0)
    assert value_str(Boom()).startswith('<')   # safe_repr of the Boom instance
for t in [test_path,test_datetime,test_date,test_enum,test_intenum_sub,test_uuid,
          test_bytes_short,test_bytes_long,test_ndarray,test_dataframe,test_series,
          test_unregistered_falls_back,test_latest_registered_wins,test_throwing_provider_falls_back]: t()
```

- [ ] **Step 2: Run to verify it fails**

Run the cell. Expected: `ImportError: cannot import name 'str_provider'`.

- [ ] **Step 3: Write minimal implementation**

Export cell (absolute import in the notebook; nbdev emits `.reprs`):

```python
#| export
import pathlib, datetime, enum, uuid
from paar.reprs import safe_repr
try: import numpy as np
except Exception: np = None
try: import pandas as pd
except Exception: pd = None

_PROVIDERS = []   # list[(type, fn)]

def register_str_provider(typ, fn):
    "Register fn(value)->str as the display string for instances of typ (and subclasses)."
    _PROVIDERS.append((typ, fn))

def str_provider(typ):
    "Decorator: @str_provider(T) registers the decorated fn as T's value-string provider."
    def deco(fn): register_str_provider(typ, fn); return fn
    return deco

def provided_str(v):
    "Custom display string for v (latest-registered isinstance match), or None."
    for typ, fn in reversed(_PROVIDERS):
        try:
            if isinstance(v, typ): return fn(v)
        except Exception:
            return None
    return None

def value_str(v):
    "Display string for v: a registered provider if any, else safe_repr(v)."
    s = provided_str(v)
    return s if s is not None else safe_repr(v)

def _bytes_str(b):
    h = bytes(b[:8]).hex()
    return f'{h}… ({len(b)} bytes)' if len(b) > 8 else f'{h} ({len(b)} bytes)'

register_str_provider(pathlib.Path, lambda p: str(p))
register_str_provider(datetime.date, lambda d: d.isoformat())
register_str_provider(datetime.time, lambda t: t.isoformat())
register_str_provider(datetime.datetime, lambda d: d.isoformat())
register_str_provider(enum.Enum, lambda e: f'{type(e).__name__}.{e.name}')
register_str_provider(uuid.UUID, lambda u: str(u))
register_str_provider(bytes, _bytes_str)
register_str_provider(bytearray, _bytes_str)
if np is not None:
    register_str_provider(np.ndarray, lambda a: f'ndarray shape={tuple(a.shape)} {a.dtype}')
if pd is not None:
    register_str_provider(pd.DataFrame, lambda d: f'DataFrame [{d.shape[0]}×{d.shape[1]}]')
    register_str_provider(pd.Series, lambda s: f'Series [{s.shape[0]}] {s.dtype}')
```

- [ ] **Step 4: Run to verify it passes**

Run: `uv run --directory /Users/71293/code/personal/orgs/paar nbdev-prepare`
Expected: `paar/providers.py` generated; all 14 asserts pass.

- [ ] **Step 5: Add the top-level re-export**

Edit `paar/__init__.py` to append the re-export after the version line:

```python
__version__ = "0.0.1"
from .providers import str_provider, register_str_provider
```

- [ ] **Step 6: Verify the re-export survives export + works**

Run: `uv run --directory /Users/71293/code/personal/orgs/paar nbdev-prepare`
Then: `uv run --directory /Users/71293/code/personal/orgs/paar python -c "import paar; print(paar.str_provider, paar.register_str_provider)"`
Expected: prints the two callables; nbdev-prepare did NOT strip the `from .providers import ...` line from `paar/__init__.py` (confirm it's still there — nbdev only manages `__version__`).

- [ ] **Step 7: Commit**

```bash
git add nbs/07_providers.ipynb paar/providers.py paar/__init__.py paar/_modidx.py && git commit --no-verify -m "feat(providers): value-string registry, built-ins, top-level API"
```

---

### Task 2: `snapshot` uses `value_str`

**Files:**
- Modify: `nbs/03_snapshot.ipynb` (the `#| export` cell + the test cell) → re-exports `paar/snapshot.py`
- Test: cells in `nbs/03_snapshot.ipynb`

**Interfaces:**
- Consumes: `paar.providers.value_str`.
- Produces: `_var_info` sets `value=value_str(v)` (was `safe_repr(v)`); `safe_repr` no longer imported by `snapshot`.

- [ ] **Step 1: Add the failing test**

In `nbs/03_snapshot.ipynb`, add `import pathlib` to the test cell's imports and add this test (include it in the runner list):

```python
import pathlib
def test_value_uses_provider():
    by = {r.name:r for r in snapshot({'p': pathlib.Path('/tmp/x'), 'n': 5})}
    assert by['p'].value==str(pathlib.Path('/tmp/x'))
    assert by['n'].value=='5'
```

- [ ] **Step 2: Run to verify it fails**

Run the cell. Expected: `AssertionError` on `by['p'].value` (currently the truncated `repr`, e.g. `PosixPath('/tmp/x')`, not `/tmp/x`).

- [ ] **Step 3: Modify the `#| export` cell**

Change the reprs import line and the value expression. The import line becomes:
```python
from paar.reprs import get_shape, get_dtype
from paar.providers import value_str
```
(Remove `safe_repr` from the reprs import — it is no longer used in this module.) And in `_var_info`, change the value expression:
```python
                       value=value_str(v), shape=get_shape(v), dtype=get_dtype(v),
```
Leave the rest of `_var_info`, `snapshot`, `_walk`, `expand`, `grid_page`, and `MAX_CHILDREN` unchanged. (The `is_error` path keeps its `f'<error {e}>'` value.)

- [ ] **Step 4: Run to verify it passes**

Run: `uv run --directory /Users/71293/code/personal/orgs/paar nbdev-prepare`
Expected: `paar/snapshot.py` regenerated; `test_value_uses_provider` and all existing snapshot tests pass.

- [ ] **Step 5: Commit**

```bash
git add nbs/03_snapshot.ipynb paar/snapshot.py paar/_modidx.py && git commit --no-verify -m "feat(snapshot): use value_str for the display value"
```

---

### Task 3: Demo a custom provider

**Files:**
- Modify: `nbs/index.ipynb` (the `#| eval: false` demo cell + a markdown note)

**Interfaces:**
- Consumes: `paar.str_provider`.
- Produces: no exports; a demo registering a custom provider.

- [ ] **Step 1: Extend the demo data cell**

In `nbs/index.ipynb`, extend the existing `#| eval: false` data cell (keep `#| eval: false` first) to register a custom provider and add a value that uses it:

```python
#| eval: false
import math, numpy as np, pandas as pd
from paar import str_provider
data = {'a': 1, 'b': [1, 2, {'deep': [10, 20]}], 'c': 'hello', 'pi': math.pi}
arr = np.arange(12).reshape(3, 4)
df = pd.DataFrame({'x': range(5), 'y': list('abcde')})

class Money:
    def __init__(self, cents): self.cents = cents
@str_provider(Money)
def _(m): return f'${m.cents/100:,.2f}'
wallet = Money(129950)
```

- [ ] **Step 2: Add a markdown note after the demo cells:**

```
Register a **custom value renderer** for your own types with `@str_provider(T)` — e.g. `wallet`
above shows `$1,299.50` instead of a `Money` repr. Built-in providers already prettify `Path`,
`datetime`, `Enum`, `UUID`, `bytes`, and numpy/pandas summaries.
```

- [ ] **Step 3: Prepare & verify**

Run: `uv run --directory /Users/71293/code/personal/orgs/paar nbdev-prepare`
Expected: `Success.`, all module tests pass, demo cells SKIPPED (no server start / no hang). `git status --short` shows no `.sesskey` files.

- [ ] **Step 4: Commit**

```bash
git add nbs/index.ipynb paar/_modidx.py && git commit --no-verify -m "docs: demo custom str_provider"
```
(Also stage `README.md` if the hook regenerated it.)

---

## Self-Review

**Spec coverage:**
- registry (`isinstance`, latest-first) + `str_provider`/`register_str_provider`/`provided_str`/`value_str` → Task 1. ✓
- built-in providers (Path, datetime/date/time, Enum, UUID, bytes/bytearray, guarded numpy/pandas) → Task 1. ✓
- throwing-provider fallback + missing-numpy/pandas guard → Task 1 (`provided_str` try/except; `if np/pd is not None`). ✓
- top-level re-export (`paar.str_provider`) → Task 1 (Steps 5-6). ✓
- one integration hook (`value_str` in `_var_info`) → Task 2. ✓
- user-facing demo → Task 3. ✓
- Deferred (correctly absent): child/attribute customization, P5 JupyterLab.

**Placeholder scan:** none — every step carries runnable code/commands.

**Type consistency:** `value_str`/`provided_str`/`register_str_provider`/`str_provider` names identical across Tasks 1/2/3 and the spec; `value_str(v) -> str` signature matches its use in `snapshot._var_info`; the `_PROVIDERS` list + `reversed()` latest-first behavior is asserted by `test_latest_registered_wins`.

**Notes:**
- Task 2 removes `safe_repr` from `snapshot`'s imports (it becomes unused there) — this avoids an unused-import finding; `safe_repr` is still used by `providers` and `grid`.
- datetime/date/time providers all call `v.isoformat()`, which is polymorphic, so their registration order is irrelevant to output (a `datetime` value yields a full timestamp regardless of which of the three entries matches first).
- numpy `int64` dtype in `test_ndarray`/`test_series` assumes the macOS/Linux venv default (64-bit); correct for this project's environment.
