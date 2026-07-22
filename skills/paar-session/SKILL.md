---
name: paar-session
description: Work with the user's live Python session through the paar MCP tools (list_vars, expand, grid, execute, session_info). Use whenever a paar MCP server is connected and the task involves inspecting the user's live variables, computing on them, or building new results alongside them.
---

# Working in a paar session

You are connected to a **live Python session owned by the user** — their real, running
namespace, inspected by [paar](https://github.com/vedicreader/paar). You see what they see.
Access is deliberately asymmetric:

- **Read anything.** Every variable, any depth.
- **Create anything new.** Names you make persist across `execute` calls in your own overlay
  layer — they never enter the owner's namespace, and the owner can watch them via
  `paar.fasthtml.agent_layer()`.
- **Never change what the owner made.** Deleting, rebinding-in-place, or mutating owner
  variables is blocked server-side, as are destructive file/shell escapes (`shutil.rmtree`,
  `os.remove`, `subprocess`, `exec`/`eval`, ...). This is enforced by an AST restriction list
  in the owner's server, not by convention.

## Tools

| tool | use for |
|---|---|
| `session_info` | env label, access mode (`restricted`/`readonly`), current policy, your overlay names |
| `list_vars` | the variable tree (`minimal`/`standard`/`full` profiles); each node has an `accessor` |
| `expand` | one lazy level of children at an accessor — cheap, prefer over printing |
| `grid` | paged table windows of DataFrames/arrays — never print a whole frame |
| `execute` | run Python; `scope='overlay'` (default) persists your names, `'isolated'` discards writes |

Accessors are JSON lists: `["data"]` is the variable, `["data", 2]` its third child as shown
by `expand`, and so on down.

## When a cell is blocked

An `execute` error starting with `blocked:` means the cell would have changed owner state.
Do not retry variations of the same mutation, and do not attempt to bypass the rule — the
policy is the owner's. Rewrite to bind results to **new names**:

- `df.drop(cols, inplace=True)` → `df2 = df.drop(cols)`
- `lst.append(x)` → `lst2 = [*lst, x]` (or copy once: `mine = list(lst)`, then mutate `mine`)
- `del big` → leave it; you cannot free the owner's objects

Once you shadow a name (`x = ...`), your `x` is yours to mutate; the owner's `x` is untouched
and still what *they* see.

## Etiquette

- Start with `session_info` + `list_vars`, not a dump-everything `execute`.
- In `readonly` mode every `execute` is isolated: reads and throwaway computation only.
- Keep your overlay tidy: meaningful names, since the owner reviews them live.
- The session may also be inspected/changed by the owner between your calls — re-read rather
  than assume when freshness matters.
