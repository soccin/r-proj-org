# Recipe: Start a New Project With This Layout

**This is an instruction set for an AI agent.** Give it this file (paste it, or point the
agent at it) and tell the agent which project directory to set up. The agent should follow
the numbered steps and run each verification before moving on.

The *why* behind this structure lives in `RECOMMENDED-CONVENTION.md` (R + Python) and
`RECOMMENDED-CONVENTION-R-ONLY.md` (R-only). This recipe is the *how*. Read the matching
convention doc if a choice here is unclear; do not invent structure beyond it.

**Hard constraints (do not violate):**
- Add **no dependencies** and **no proprietary interchange formats** (no Parquet/Arrow).
  R-native intermediates are `.rds`; any cross-language *file* handoff is CSV or XLSX only.
- This is a layout task. Create folders, stub files, and config — do not write analysis logic.
- Make only the changes these steps call for.

---

## Step 0 — Determine the project shape → verify

Decide, by asking the user if unstated:

- **Target directory?** (the project root to create/populate)
- **R-only, or R + Python?**
  - *R-only* — pipeline is all `.R`; Python (if ever) via `reticulate`. Follow
    `RECOMMENDED-CONVENTION-R-ONLY.md`. **Do not create `python/`.**
  - *R + Python* — standalone `.py` pipeline stages alongside `.R`. Follow
    `RECOMMENDED-CONVENTION.md`. **Create `python/` and a Python dep file.**
- **Optional layers wanted?** `notebooks/` (exploration), `analysis/` + `docs/` (workflowr
  report/site). Default: skip both unless asked.

**Verify:** restate the chosen shape (dir, R-only vs mixed, which optional layers) in one
line before creating anything.

---

## Step 1 — Create the shared backbone → verify

In the target directory, create the language-neutral skeleton (same for R-only and mixed):

```
data/
  raw/
  external/
results/
  figures/
  tables/
cache/                # may stay empty; pipeline stages create cache/NN_* dirs as needed
R/
scripts/
```

Create placeholder files so empty dirs are tracked and intent is documented:

- `data/README.md` — a provenance template (table: file | source | date received | from whom).
- `cache/.gitkeep`, `results/figures/.gitkeep`, `results/tables/.gitkeep`.
- `README.md` at root — one paragraph: what the project is, how to rebuild it
  (`renv::restore()` then run `scripts/` in order), and which languages do what.

**Verify:** the tree matches the "Recommended Structure" block in the matching convention
doc (minus any optional layers). List the created paths.

---

## Step 2 — Add the code-layer stubs → verify

**If mixed (R + Python):**
- Create `python/` as an importable package: `python/<pkgname>/__init__.py`.
- The pipeline in `scripts/` is **one numbered sequence across both languages**
  (`01_*.R`, `02_*.py`, ...). Create no real stages yet; just confirm the convention in
  `README.md`.

**If R-only:**
- Do **not** create `python/`. (If reticulate is later needed, Python wrappers live in `R/`.)

In both cases, add a short comment header convention note to `README.md`: every script reads
from `data/`/`cache/`, writes only to `cache/`/`results/`, and shares a numeric prefix with
its `cache/NN_*` output dir.

**Verify:** mixed projects have `python/<pkg>/__init__.py` and no stray top-level `.py`;
R-only projects have no `python/`.

---

## Step 3 — Write `.gitignore` → verify

Create `.gitignore` at root:

```
# Derived — regenerate from scripts
cache/
results/figures/

# Rendered report (only if using analysis/)
analysis/docs/

# Environments
renv/
.venv/
__pycache__/
*.pyc

# Keep:
#   data/                     (precious — provenance in data/README.md)
#   results/tables/           (small, track changes)
#   renv.lock, pyproject.toml (reproducibility record)
```

Drop the `.venv/`/`__pycache__/` lines for an R-only project; drop the `analysis/docs/` line
if no `analysis/` layer.

**Verify:** `data/` and `results/tables/` are NOT ignored; `cache/` and `results/figures/`
ARE ignored.

---

## Step 4 (OPTIONAL) — Environment / reproducibility setup → verify

Skip unless the user wants a runnable, locked environment. This step adds tooling, so confirm
first.

**R (both shapes):**
- `renv::init()` in the project root → produces `renv.lock` + `renv/`.

**Python, mixed projects:**
- Create `pyproject.toml` (or `requirements.txt`) declaring the project's Python package + deps.

**Python via reticulate, R-only projects:**
- `renv::use_python()` to create a project-local Python env recorded *inside* `renv.lock`;
  pin the interpreter in `.Rprofile`. One lockfile, one `renv::restore()`.

Also create `project.Rproj` (root anchor) and `.Rprofile` (load libs; set `RETICULATE_PYTHON`
if applicable) — these are cheap and worth adding even when skipping the lockfile work.

**Verify:** if run, `renv.lock` exists; for mixed, the Python dep file exists; `.Rprofile`
pins the interpreter when reticulate is in use.

---

## Step 5 — Final check → report

- Re-list the full tree and confirm it matches the chosen convention doc.
- Confirm no analysis logic was written, no dependencies added beyond the optional Step 4,
  and no proprietary formats introduced.
- Report what was created and what was deliberately skipped (optional layers, env setup).

**Done when:** the skeleton matches the convention, `.gitignore` protects `data/` and ignores
`cache/`, and the report states exactly what exists.
