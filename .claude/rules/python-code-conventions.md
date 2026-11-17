---
paths:
  - "**/*.py"
  - "scripts/**/*.py"
  - "Figures/**/*.py"
  - "explorations/**/*.py"
---

# Python Code Standards

**Standard:** Senior Principal Data Engineer + PhD researcher quality

> **Scope:** These standards apply to **analysis scripts** — data work, simulations, figure
> generation (one global seed at top, imports at the top, relative output paths). For a
> distributable **package** (`src/`, `tests/`, `pyproject.toml`, `__init__.py`), the packaging
> rules differ (declared dependencies, no I/O at import time, `src/` layout); the numerical
> discipline in §8 applies to both. The R counterpart is
> [`r-code-conventions.md`](r-code-conventions.md) — same discipline, R idioms.

---

## 1. Reproducibility

- Set the seed **once** at the top: `rng = np.random.default_rng(20260220)` (a `YYYYMMDD`
  date reads as intent; `42` does not)
- Pass `rng` down as an argument — never call `np.random.seed()` or the legacy global
  `np.random.*` functions inside functions or loops
- All imports at the top of the file, standard library first, then third party, then local
- All paths relative to the repository root, built with `pathlib.Path` — never string
  concatenation and never a hardcoded absolute path
- Output directories created with `out_dir.mkdir(parents=True, exist_ok=True)`
- **Pin what you depend on.** `requirements.txt` or `environment.yml` in the repo, with
  versions. A script that only runs on your machine is not a script
- The script runs cleanly from `python3 scripts/py/foo.py` on a fresh clone

## 2. Function Design

- `snake_case` naming, verb-noun pattern (`run_simulation`, `generate_dgp`, `compute_att`)
- Type hints on every public function signature: `def compute_att(y: np.ndarray, d: np.ndarray) -> float:`
- A docstring on every non-trivial function — one line of intent, then Args / Returns /
  Raises only where the signature does not already say it
- Default parameters for every tuning value; no magic numbers in a function body
- Named returns — a `dict`, a `dataclass`, or a `NamedTuple` — not a bare tuple whose
  position the caller has to remember

## 3. Domain Correctness

<!-- Customize for your field's known pitfalls -->
- Verify estimator implementations match the formulas on the slides
- Check known library quirks (document them in §6) — the version where a default changed,
  the argument that silently does nothing
- Confirm the estimand: ATT is not ATE, and a weighted mean is not an unweighted one

## 4. Visual Identity

```python
# --- Institutional palette (CEIBS) ---
PRIMARY_BLUE   = "#012169"
PRIMARY_GOLD   = "#b9975b"
ACCENT_GRAY    = "#525252"
POSITIVE_GREEN = "#15803d"
NEGATIVE_RED   = "#b91c1c"
```

### Custom theme

```python
import matplotlib as mpl
import matplotlib.pyplot as plt

def theme_custom(base_size: int = 14) -> None:
    """Apply the project theme once, before any figure is drawn."""
    mpl.rcParams.update({
        "figure.dpi": 200,
        "savefig.dpi": 200,
        "font.size": base_size,
        "axes.titlesize": base_size + 2,
        "axes.titleweight": "bold",
        "axes.titlecolor": PRIMARY_BLUE,
        "axes.spines.top": False,
        "axes.spines.right": False,
        "axes.labelsize": base_size,
        "legend.frameon": False,
        "legend.loc": "lower center",
        "savefig.transparent": True,
    })
```

Palette names here must match `Preambles/header.tex` and `Quarto/theme-template.scss`;
`scripts/check-palette-sync.py` enforces the LaTeX ↔ SCSS pair, so add the Python side to
that check if you change a value.

### Figure dimensions for Beamer

```python
fig, ax = plt.subplots(figsize=(12, 5))   # inches, not pixels
fig.savefig(out_path, bbox_inches="tight", transparent=True)
```

`transparent=True` is the rule, not a preference — a white `facecolor` shows as a white box
on a dark Beamer theme.

## 5. Saved-artifact pattern

**Heavy computation saves its results; slide rendering loads them.** A deck must not
re-run a 40-minute simulation to draw a chart.

```python
import pickle
from pathlib import Path

def save_result(obj, name: str, out_dir: Path = Path("output")) -> Path:
    """Persist a computed object next to a human-readable summary."""
    out_dir.mkdir(parents=True, exist_ok=True)
    path = out_dir / f"{name}.pkl"
    with path.open("wb") as fh:
        pickle.dump(obj, fh, protocol=pickle.HIGHEST_PROTOCOL)
    return path
```

Prefer `parquet` (`df.to_parquet`) for tabular results and `.npz` for arrays — both survive a
language change and can be inspected without the code that wrote them. `pickle` is fine for a
complex object you will only reload in Python, but never load a pickle from an untrusted
source. Every object a slide references must have a saved counterpart; a missing save is a
**High** finding, because the deck cannot render without it.

## 6. Common pitfalls

<!-- Add your field-specific pitfalls here -->
| Pitfall | Impact | Prevention |
|---------|--------|------------|
| `df.groupby(...).mean()` skips NaN by default, `np.mean` does not | Two different answers from "the same" mean | Always pass `numeric_only=True` / `skipna=`; state the choice |
| Chained assignment `df[df.a > 0]["b"] = 1` | Silent no-op on a copy | `df.loc[df.a > 0, "b"] = 1` |
| Integer division `a / b` where counts were meant | Float index into an array | `a // b`, and `int()` at the boundary |
| Mutable default `def f(x, acc=[])` | State leaks between calls | `acc=None` then `acc = [] if acc is None else acc` |
| `plt.show()` in a script that also saves | Blocks a headless run | Save only; use `matplotlib.use("Agg")` |
| Hardcoded path with `~` or `/Users/` | Breaks on every other machine | `Path.home()` only for genuine user config, never for data |

## 7. Line length

**Standard:** 80 characters for code (PEP 8), **100** for docstrings and comments.

**Exception — mathematical formulas.** A line may exceed the limit **if and only if**:

1. Breaking it would harm readability of the math — influence functions, matrix algebra,
   finite-difference approximations, a line implementing a paper's equation
2. An inline comment names the mathematical operation:
   ```python
   # Sieve projection: inner product of residuals onto basis functions P_k
   alpha_k = (r_i * basis[:, k]).sum() / (basis[:, k] ** 2).sum()
   ```
3. The line is in a numerically intensive section (simulation loop, estimation routine,
   inference)

**Quality gate impact:** a long line in non-mathematical code is a reviewer-applied Style
violation (-1, see the Python Scripts table in [`quality-gates.md`](quality-gates.md));
a documented mathematical line carries no penalty.

## 8. Numerical discipline

The same seven rules as the R standards, in Python dress. They are the ones that produce
wrong numbers quietly.

- **No float equality.** `x == y` on floats is a coin flip. Use `np.isclose(x, y)` or
  `abs(x - y) < TOL`.
- **Clamp probabilities to an OPEN interval** before handing them to a quantile function.
  `scipy.stats.norm.ppf(0.0)` returns `-inf`, and one `-inf` poisons every downstream mean.
  ```python
  EPS = 1e-12
  p = np.clip(p, EPS, 1 - EPS)      # now safe for norm.ppf(p)
  ```
- **Integer types for counts.** `n = 1000` vs `np.int64(1000)` matters at an array index or
  in a loop bound; use `range(n)` and `int(...)` deliberately, and never index with a float.
- **Pre-allocate, don't grow.** `np.empty(n)` or a preallocated list, never
  `arr = np.append(arr, x)` in a loop (which reallocates every iteration).
- **Deterministic parallel seeding.** `rng.spawn(k)` (NumPy ≥ 1.25) or
  `SeedSequence(seed).spawn(k)` — one child stream per worker, so a 4-worker and an
  8-worker run give the same numbers.
- **Explicit NaN policy.** `np.nanmean` vs `np.mean`, `skipna=` in pandas, and
  `dropna=` everywhere it is a choice. Never rely on a default you did not read.
- **No `==`/`!=` on a whole DataFrame or array expecting a bool.** `df1 == df2` returns a
  frame; `df.equals()` is the comparison you meant.

## 9. Code quality checklist

```
[ ] Imports at top, grouped stdlib / third party / local
[ ] rng = np.random.default_rng(YYYYMMDD) once at top; passed down, never re-seeded
[ ] All paths relative, built with pathlib
[ ] Type hints on public signatures; docstrings on non-trivial functions
[ ] Figures: transparent background, explicit figsize, theme applied
[ ] Every object a slide uses is saved to output/
[ ] Comments explain WHY, not WHAT
[ ] Numerical discipline: np.isclose, clipped probabilities, preallocated arrays,
    spawned RNG streams, explicit NaN policy
[ ] No chained assignment, no mutable defaults, no `plt.show()` in a batch script
```
