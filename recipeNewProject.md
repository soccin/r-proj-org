# Recipe: Start a New Project With This Layout

**This is an instruction set for an AI agent.** Give it this file (paste it, or point the
agent at it) and tell the agent which project directory to set up. The agent should follow
the numbered steps and run each verification before moving on.

The *why* behind this structure lives in `docs/recommendedConvention.md` (R + Python) and
`docs/recommendedConventionROnly.md` (R-only). This recipe is the *how*. Read the matching
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
    `docs/recommendedConventionROnly.md`. **Do not create `python/`.**
  - *R + Python* — standalone `.py` pipeline stages alongside `.R`. Follow
    `docs/recommendedConvention.md`. **Create `python/` and a Python dep file.**
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
  run01/              # RUN axis: one dir per run, outermost
    figures/
    tables/
cache/                # may stay empty; stages create cache/<run>/NN_* dirs as needed
R/
scripts/
```

Create placeholder files so empty dirs are tracked and intent is documented:

- `00.PARAMS.yml` at root — the run file. Required content is one top-level key:
  ```yaml
  run: run01
  ```
  Add a `params:` block only if the user already knows the project's parameters; the
  namespace under `run:` is the project's own. The value is used **verbatim** as the run
  directory name, so it must match the `results/<run>/` dir created above.
- `data/raw/MANIFEST.tsv` — header row only: `file`, `bytes`, `md5`, `source`, `date`. This
  is the tracked record of raw inputs whose bytes are not committed (Step 3).
- `data/README.md` — the narrative a table can't hold: who provided the data, under what
  terms, known caveats.
- `cache/.gitkeep`, `results/run01/figures/.gitkeep`, `results/run01/tables/.gitkeep`.
- `README.md` at root — one paragraph: what the project is, how to rebuild it
  (`renv::restore()`, then `scripts/00_fetch_data.R` once, then the entry point), and which
  languages do what.
- An **entry point** at the root: `run_all.R` for R-only (`source(here("scripts", "NN_....R"))`
  per stage) or `run_all.sh` for mixed (`Rscript` / `python` per stage, `set -euo pipefail`).
  Create it with the stages commented out; uncomment as stages are written. Before the stages,
  it must read `run:` from `00.PARAMS.yml`, create `results/<run>/`, and copy `00.PARAMS.yml`
  into it — see "Running the Pipeline" in the matching convention doc for the exact snippet.
- A **root anchor**: `project.Rproj`, or a bare `.here` file if not using RStudio. Without
  one, `here::here()` silently resolves to the working directory.

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
from `data/`/`cache/`, writes only to `cache/`/`results/`, reads its run id from
`00.PARAMS.yml`, and shares a numeric prefix with its `cache/<run>/NN_*` output dir.

**Verify:** mixed projects have `python/<pkg>/__init__.py` and no stray top-level `.py`;
R-only projects have no `python/`.

---

## Step 3 — Write `.gitignore` → verify

Create `.gitignore` at root:

```
# Derived — regenerate from scripts
cache/
results/*/figures/

# Rendered report (only if using analysis/)
analysis/docs/

# Raw data: ignore the bytes, commit the provenance
data/raw/*
!data/raw/MANIFEST.tsv

# Environments
renv/
.venv/
__pycache__/
*.pyc

# Keep:
#   data/external/            (reference files, usually small)
#   results/*/tables/         (small, track changes)
#   results/*/00.PARAMS.yml   (what produced that run)
#   00.PARAMS.yml             (the current run)
#   renv.lock, pyproject.toml (reproducibility record)
```

Drop the `.venv/`/`__pycache__/` lines for an R-only project; drop the `analysis/docs/` line
if no `analysis/` layer.

The raw-data rule is **ignore the bytes, commit the provenance** — `MANIFEST.tsv` plus
`scripts/00_fetch_data.R` reconstruct `data/raw/` in a few kilobytes, without a
multi-gigabyte commit that would inflate every future clone forever. Two cautions:

- The pattern must be `data/raw/*`, not `data/raw/`. Git cannot re-include a file whose
  parent *directory* is excluded, so the `!` line would silently do nothing.
- The figures pattern must be `results/*/figures/`, not `results/figures/` — figures live one
  level down, under the run dir. The plain pattern matches nothing and every figure gets
  committed.
- Small raw inputs (sample sheet, gene list, config table) belong in git directly. Add an
  explicit un-ignore per file, e.g. `!data/raw/samplesheet.csv`. The rule is about size,
  not about the directory.

**Verify:** `data/external/`, `data/raw/MANIFEST.tsv`, `00.PARAMS.yml` and
`results/run01/tables/` are NOT ignored; `cache/`, `results/run01/figures/` and the raw bytes
ARE. Confirm with `git check-ignore -v data/raw/MANIFEST.tsv` (no match) and
`git check-ignore -v results/run01/figures/x.pdf` (matches).

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

Also create `.Rprofile` (load libs; set `RETICULATE_PYTHON` if applicable) — cheap and worth
adding even when skipping the lockfile work. The root anchor itself was already created in
Step 1.

**Verify:** if run, `renv.lock` exists; for mixed, the Python dep file exists; `.Rprofile`
pins the interpreter when reticulate is in use.

---

## Step 5 — Final check → report

- Re-list the full tree and confirm it matches the chosen convention doc.
- Confirm no analysis logic was written, no dependencies added beyond the optional Step 4,
  and no proprietary formats introduced.
- Report what was created and what was deliberately skipped (optional layers, env setup).

**Done when:** the skeleton matches the convention; `.gitignore` ignores `cache/`,
`results/*/figures/` and the raw bytes while keeping `MANIFEST.tsv` and `data/external/`;
a root anchor, a `00.PARAMS.yml` and an entry point exist, with the run id in `00.PARAMS.yml`
matching the `results/<run>/` dir; and the report states exactly what exists.
