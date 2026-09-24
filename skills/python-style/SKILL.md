---
name: python-style
description: |
  How I want Python written — the readability choices that tooling does NOT enforce (blank-line
  spacing, class-vs-function, import shape). ALWAYS apply this skill automatically whenever you
  write or edit Python in my projects, without waiting to be asked. Also run it when I type
  /python-style, ask to "apply my style", "make this readable", "add blank lines", "rework this
  for readability", or "clean up the layout". It complements — never duplicates — Ruff and
  interrogate (see the check-quality skill); everything here must stay Ruff-compatible.
---

# Python style (readability)

Ruff and interrogate own formatting, import sorting, line length, quotes and docstring coverage
(see the `check-quality` skill and the global CLAUDE.md). **This skill owns only the subjective
readability choices Ruff does not enforce.**

> **Golden constraint:** everything here must stay **Ruff-compatible**. Pre-commit is the source
> of truth — if `ruff format`/`ruff check --fix` would rewrite it, don't do it. The rules below
> were chosen because Ruff leaves them intact.

## Blank lines — let the code breathe (≤ 1 blank line)

Use single blank lines to separate logical units. **Never 2+ consecutive blank lines inside a
function** — Ruff collapses them to one.

- Put a blank line **after a function's (or class's) docstring**, before the body — this is the
  Ruff-compatible way to separate the signature from the code. A blank line *before* the
  docstring (directly under `def:`) is stripped by Ruff, so it goes after the docstring:

  ```python
  def apply_rates(subtotal, rates):
      """Apply successive rates to a subtotal."""

      total = subtotal
      for rate in rates:
          total *= rate

      return total
  ```

- Put a blank line at the **end of an `if` block, before `else`/`elif`** (and after any block —
  `for`, `while`, `with`, `try/except` — before the next sibling statement):

  ```python
  if user.is_active:
      grant_access(user)

  else:
      deny_access(user)

  log_decision(user)
  ```

- Separate the logical phases of a function (setup → core → return) with one blank line:

  ```python
  def price(cart):
      items = load_items(cart)
      rates = load_rates(cart.currency)

      subtotal = sum(i.amount for i in items)
      total = apply_rates(subtotal, rates)

      return total
  ```

- Put a blank line **before the final `return`** (skip it in 1–2 line functions) — it sets the
  result apart from the work that produced it.

- Do **not** put a blank line right after a block opens (just under a `:`) or as the last line of
  a block — Ruff removes those. Keep blank lines *between* statements.

## Imports — one per line (do not group)

The project uses Ruff isort with `force-single-line = true`, so imports are **one per line**:

```python
from mypkg.file import alpha
from mypkg.file import beta
```

**Do not** combine them into `from mypkg.file import (alpha, beta)` — Ruff will split it back on
the next run. Let Ruff sort and lay out imports; don't hand-fight it.

## Prefer classes when they carry state + behaviour

- Reach for a **class** when related state and the operations on it belong together (cohesion),
  or for a data model — use a `pydantic` `BaseModel` for data, `BaseSettings` for config
  (per CLAUDE.md).
- **Don't** wrap stateless helpers in a class just to have one — a module-level function is
  simpler and clearer.
- One responsibility per class/function (SRP). If a class grows a second job, split it.

## Docstrings

Google-style, 100 % coverage — enforced by interrogate, owned by CLAUDE.md/`check-quality`. Just
comply:

```python
def apply_rates(subtotal: float, rates: list[float]) -> float:
    """Apply successive rates to a subtotal.

    Args:
        subtotal: Pre-tax amount.
        rates: Multiplicative rates to apply in order.

    Returns:
        The final amount after all rates.
    """
```

## Applying to existing code (rework mode)

When invoked to rework existing code (e.g. `/python-style`, "add blank lines", "make this
readable"):

1. **Scope** — the `.py` files in the current git diff (`git diff --name-only HEAD` +
   `--cached`), unless I name a specific path/file. Never sweep the whole repo unasked.
2. **Whitespace only** — apply the blank-line rules above. **Never** change logic, names, order of
   statements, or behaviour. This is a layout pass, not a refactor.
3. **Stay Ruff-compatible** — ≤ 1 blank line, single-line imports; nothing Ruff would revert.
4. **Verify** — run the `check-quality` skill (or at least `ruff format --check` + `pre-commit`)
   afterwards so the formatter confirms the layout survives.
5. **Report** — list the files touched as `file:line`; the diff must be whitespace-only.
