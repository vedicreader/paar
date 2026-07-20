# paar P4 — str-providers / custom value renderers — design

Date: 2026-07-20
Status: approved (pre-implementation)
Repo: `~/code/personal/orgs/paar` (nbdev, hatchling, Apache-2.0, Python ≥3.10)
Builds on: P1 (flat rows + live refresh), P2 (container resolvers + lazy expand), P3 (numpy/pandas grid) — all merged.

## Summary

P4 lets the inspector show a nicer **value string** for a type than its raw `repr`. It adds a
small registry (`providers.py`) mapping types → value-string functions, ships built-in
providers for common stdlib types, and exposes a public API so users register providers for
their own types. Providers customize the **value string only** — container/grid detection and
children are unchanged.

This reverse-engineers pydev's "str-providers" (the `_str_from_providers` path in
`var_to_struct`), adapted to paar's in-process, fastcore-style architecture.

## Decisions (locked)

- **Scope:** built-in providers for common stdlib types AND a public registration API for
  user types.
- **Provider power:** value string only. A provider maps a value → a display string that
  replaces `safe_repr` for that value. It does NOT change `is_container`/`is_grid` or children.
- **Location:** a new serialize-layer leaf `providers.py` (sibling to `resolvers.py` /
  `grid.py`), consulted by `snapshot`. `reprs.py` stays the low-level primitive module.
- **Matching:** `isinstance` (subclasses match), **latest-registered-first** precedence, so a
  user registration overrides a built-in.
- **Safety:** a provider that raises falls back to `safe_repr`; missing numpy/pandas skip
  their (guarded) providers.
- **No frontend changes:** the value string flows through the existing `snapshot` → `_node`/
  `_grid` path.

## Architecture

Layer rule (unchanged, extended): frontend(`fasthtml`) → bridge → serialize(`snapshot` →
`{resolvers, grid, providers}` → `reprs`) → model(`core`). `providers` imports only
`paar.reprs` (for the `safe_repr` fallback). Only `snapshot` consumes `providers`.

New/changed units:
- `providers.py` (new, `07_providers`): the registry + API + built-ins + `value_str`.
- `snapshot.py` (change one line in `_var_info`): `value=value_str(v)` instead of
  `value=safe_repr(v)`.
- `paar/__init__.py` (add): re-export `str_provider`, `register_str_provider` for top-level
  ergonomics.
- `index.ipynb` (docs): demo a custom provider.

## `providers.py` (the registry)

```python
from paar.reprs import safe_repr
# guarded optional deps for built-in array/frame summaries
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
    "The custom display string for v (latest-registered isinstance match), or None."
    for typ, fn in reversed(_PROVIDERS):
        try:
            if isinstance(v, typ): return fn(v)
        except Exception:
            return None
    return None

def value_str(v):
    "The display string for v: a registered provider if any, else safe_repr(v)."
    s = provided_str(v)
    return s if s is not None else safe_repr(v)
```

### Built-in providers (registered at import)

Stdlib (always available):
- `pathlib.Path` → `str(p)`
- `datetime.datetime` / `datetime.date` / `datetime.time` → `.isoformat()`
- `enum.Enum` → `f'{type(v).__name__}.{v.name}'`
- `uuid.UUID` → `str(v)`
- `bytes` / `bytearray` → short hex preview + length, e.g. `f'{bytes(v[:8]).hex()}… ({len(v)} bytes)'`
  (no ellipsis when `len(v) <= 8`)

Optional (import-guarded; registered only if the lib is present):
- `numpy.ndarray` → `f'ndarray shape={tuple(v.shape)} {v.dtype}'`
- `pandas.DataFrame` → `f'DataFrame [{v.shape[0]}×{v.shape[1]}]'`
- `pandas.Series` → `f'Series [{v.shape[0]}] {v.dtype}'`

These improve grid rows (whose summary value was a truncated repr) without touching the grid
itself. Registration order puts stdlib and optional built-ins in `_PROVIDERS`; user
registrations appended later take precedence via `reversed()`.

## Integration

`snapshot._var_info` changes exactly one expression: `value=value_str(v)` (imported from
`paar.providers`) replaces `value=safe_repr(v)`. Everything else in `_var_info` (type,
qualifier, shape, dtype, is_container, is_grid, path, accessor) is unchanged. The error path
(`is_error`) keeps its `f'<error {e}>'` value.

`paar/__init__.py` adds:
```python
from .providers import str_provider, register_str_provider
```
so users can `import paar; paar.str_provider(MyType)(...)` as well as
`from paar.providers import str_provider`.

## Error handling

- A provider function that raises → `provided_str` returns `None` → `value_str` falls back to
  `safe_repr`. One bad provider never breaks a row.
- numpy/pandas absent → their providers are simply not registered (guarded).
- `_var_info` already wraps the whole build in try/except → `is_error` row on any failure.

## Testing

All pure (no kernel/browser):
- `07_providers` cells:
  - each stdlib built-in returns the expected string (`Path`, `datetime`, `date`, `Enum`,
    `UUID`, `bytes` short + long);
  - guarded numpy/pandas summaries (numpy/pandas are in the venv);
  - `isinstance` subclass match (a `Path` subclass, or a user `IntEnum`);
  - user registration overrides a built-in (latest-first precedence);
  - a throwing provider falls back to `safe_repr`;
  - an unregistered type returns `safe_repr`.
- `03_snapshot` cell: a `Path` value in a snapshot shows the provider string; a plain `int`
  still shows `repr`.
- `index.ipynb`: register a custom provider for a small demo class + a markdown note.

## Tasks (4)

1. **`providers.py`** (`07_providers`): registry, `str_provider`/`register_str_provider`,
   `provided_str`/`value_str`, stdlib + guarded numpy/pandas built-ins.
2. **`snapshot`**: swap `safe_repr(v)` → `value_str(v)` in `_var_info` (`03_snapshot`).
3. **`paar/__init__.py`**: re-export `str_provider`, `register_str_provider`.
4. **`index.ipynb`**: demo a custom provider + note.

## Out of scope (deferred)

- Child/attribute customization (custom "type renderers" that change children) — stays with
  the resolver registry; not part of P4.
- P5 JupyterLab labextension.
- Per-instance / conditional providers, format-string configuration, provider priority beyond
  latest-first — YAGNI for now.
