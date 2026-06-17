# Recommended Analysis Project Convention (R-only)

The **R-only** version of this repo's layout, for projects whose pipeline is written
entirely in R. Python, when needed, is reached *in-process from R* via
[`reticulate`](https://rstudio.github.io/reticulate/) — it is an embedded dependency, not a
second codebase.

> For projects with substantial standalone Python code (its own `.py` pipeline stages and
> `pyproject.toml`), use the sibling document **`RECOMMENDED-CONVENTION.md`** (R + Python,
> split by language) instead. This document is the simpler, single-language case.

It reconciles the two source docs in this repo:

- `originals/r-project-organization.md` — provenance-based, numbered-stage pipeline for plain `.R` scripts.
- `originals/R Working Analysis Directory Tree Template.txt` — the `workflowr` convention for
  reproducible, publishable research websites built from `.Rmd`.

> **Backbone:** provenance-based directories (`data/` → `cache/` → `results/`) with numbered
> `.R` pipeline stages. **One language to run, one entry point.** Python (if any) is invoked
> from R via `reticulate` and recorded in `renv.lock` — there is no parallel Python tree and
> no cross-language file handoff.

---

## "R-only" but with reticulate — what that means

The project is R-only in the ways that matter for day-to-day work:

- **One pipeline language.** Every script in `scripts/` is `.R`. There are no `.py` stages.
- **One entry point and mental model.** You run R; R drives everything.
- **One environment record.** `renv.lock` is the source of truth.

`reticulate` doesn't change that. It lets an R script borrow a Python library R lacks
(e.g. a scikit-learn model, a specific numpy/scipy routine) *inside* that R script. Data
crosses the boundary **in memory** via reticulate's automatic conversion
(`data.frame` ↔ `pandas`, vector ↔ `numpy`) — not as files. So:

- **Intermediates stay `.rds`.** Because Python never owns a pipeline stage, every file in
  `cache/` is written by R and read by R. The CSV/XLSX cross-language handoff rule from the
  R+Python document does **not** apply here — there is no language seam in the file layout.
- **Python deps live in `renv.lock`.** `renv::use_python()` records the Python environment
  alongside the R packages, so a single `renv::restore()` rebuilds both.

This keeps the project's reproducibility story singular: clone, `renv::restore()`, run.

---

## Recommended Structure

```
project/
├── project.Rproj           # project-root anchor (keep even outside RStudio)
├── .Rprofile               # runs on open; load R libs; pin RETICULATE_PYTHON if using reticulate
├── .gitignore
├── README.md               # what this is; how to rebuild it
│
├── renv.lock               # R packages — AND Python packages if reticulate is used
│
│  ── DATA: named by provenance ──
├── data/                   # IMMUTABLE inputs — never written by scripts, never gitignored
│   ├── raw/                #   exactly as received
│   ├── external/           #   reference data (annotations, gene lists, ...)
│   └── README.md           #   provenance: where each file came from, when, from whom
│
├── cache/                  # REGENERABLE intermediates (.rds) — safe to delete, gitignored
│   ├── 01_tidy/            #   output dir prefix matches the script that made it
│   ├── 02_processed/
│   └── 03_filtered/
│
├── results/                # FINAL outputs
│   ├── figures/            #   often large → gitignore
│   └── tables/             #   often small → track in git
│
│  ── CODE ──
├── R/                      # reusable R functions (sourced), NOT run as scripts
│                           #   reticulate wrappers (e.g. import + call a Python lib) live here
│
├── scripts/                # the ordered pipeline — ALL .R, numbered by stage
│   ├── 01_tidy_input.R     #   reads data/raw/      → writes cache/01_tidy/
│   ├── 02_process.R        #   reads cache/01_tidy/ → writes cache/02_processed/
│   └── 03_analyze.R        #   reads cache/02_...   → writes results/  (may call Python via reticulate)
│
├── notebooks/              # OPTIONAL exploratory .Rmd / .qmd
│                           #   name: NN_initials_topic, e.g. 01_ns_data-overview
│
└── analysis/               # OPTIONAL literate report layer (workflowr-style .Rmd)
    └── docs/               #   rendered HTML/figures (only if publishing a site)
```

The `notebooks/` and `analysis/` blocks are **optional**. The backbone
(`data/` → `scripts/` → `cache/` → `results/`, with `R/`) stands alone.

---

## The Principles

**1. Provenance is the primary axis — three write-tiers.**
- `data/` — came from *outside*; scripts only ever **read** it. Never gitignored.
- `cache/` — *derived*; scripts write it (as `.rds`), you can delete and regenerate it. Gitignored.
- `results/` — *final* deliverables you actually report.

The names carry intent: anyone sees `cache/` and knows it's disposable.

**2. Number the pipeline stages.** A stage's script and its output dir share a prefix
(`02_process.R` → `cache/02_processed/`). Six months later the dependency graph is readable
from filenames alone.

**3. Read-only vs. write-enabled stays strict.** Every script reads from `data/`/`cache/`
and writes to `cache/`/`results/`. Never write into `data/`.

**4. Functions in `R/`, pipeline in `scripts/`.** `R/` holds reusable functions you
`source()` — including any thin wrappers around reticulated Python calls. `scripts/` holds
the ordered, run-once `.R` pipeline. Keep heavy logic out of the numbered scripts.

**5. Anchor the project root.** A `.Rproj` plus `.Rprofile` means `here::here()` resolves
paths identically regardless of working directory. If using reticulate, `.Rprofile` is also
where you pin the interpreter (`Sys.setenv(RETICULATE_PYTHON = ...)` or
`reticulate::use_python()`).

**6. Document provenance with a `data/README.md`.** Recording where each raw file came from
is the single highest-value reproducibility habit.

---

## Intermediates & Handoff

- **Within the pipeline: `.rds`.** `readr::write_rds()` / `read_rds()` preserves types
  (factors, dates, list columns) exactly and is fast. Every `cache/` file is R-written,
  R-read.
- **reticulate boundary: in-memory, no files.** Python is called from inside an R stage;
  objects convert automatically. Nothing in `cache/` is in a Python-native format.
- **If you ever must exchange a *file* with non-reticulate tooling: CSV or XLSX only.** As in
  the sibling convention, do **not** introduce proprietary/binary interchange formats
  (Parquet, Arrow/Feather) and do not add dependencies for file exchange. This is a
  layout convention, not a toolchain mandate.

---

## Environment / Dependencies

- **R:** `renv` — `renv::init()`, `renv::snapshot()`, `renv::restore()`; produces `renv.lock`.
- **Python via reticulate (optional):** `renv::use_python()` creates a project-local Python
  environment and records its packages in `renv.lock`. After installing Python packages,
  `renv::snapshot()` captures them; `renv::restore()` rebuilds R **and** Python together.
  Pin the interpreter in `.Rprofile` so every session uses the same one.

The point: even with Python in play, there is **one** lockfile and **one** restore command.

---

## `.gitignore`

```
# Derived — regenerate from scripts
cache/
results/figures/

# Rendered report (only if using analysis/)
analysis/docs/

# R + reticulate environment
renv/             # project library (keep renv.lock, ignore the library)
# reticulate's Python env lives under renv/ when created via renv::use_python()

# NOT ignored:
#   data/        <- precious; version-control or document provenance in data/README.md
#   results/tables/ <- usually small; useful to track changes over time
#   renv.lock    <- the reproducibility record (R + Python)
```

---

## Reconciling the Two Source Docs

`workflowr` puts all processed data in a single flat `output/`; the custom doc uses staged,
numbered `cache/`. **Prefer staged `cache/`** — flat `output/` doesn't scale past two stages
and is ambiguous about immutability, whereas numbered stages scale to any depth and stay
self-documenting. If you adopt `workflowr` for its `.Rmd` → website tooling (a natural fit
for an R-only project), redirect its processed-data writes into `cache/NN_stage/` and keep
`docs/` strictly for rendered HTML.

---

## When to Use Which Layer

| You are doing… | Use |
| --- | --- |
| Pure scripted R processing, no report | backbone only (`data/`→`scripts/`→`cache/`→`results/`) |
| The above, plus exploration | add `notebooks/` (`.Rmd`/`.qmd`) |
| The above, plus a narrative report or published site | add the `analysis/` + `docs/` layer (`workflowr`) |
| Need a Python library R lacks | call it from an R stage via `reticulate`; record it in `renv.lock` |
| Substantial standalone Python code / `.py` pipeline stages | use the R+Python doc (`RECOMMENDED-CONVENTION.md`) instead |
| A one-off exploratory script | a single script reading `data/`, writing `results/` — don't over-build |

---

## Sources

- [workflowr getting started](https://jdblischak.github.io/workflowr/articles/wflow-01-getting-started.html)
  — the `analysis/` + `docs/` literate-website convention (also captured in this repo's `.txt`).
- [reticulate: R interface to Python](https://rstudio.github.io/reticulate/) — calling Python
  in-process from R.
- [Reproducible environments for R and Python](https://occasionaldivergences.com/posts/rep-env/)
  — using `renv` (with `renv::use_python()`) to capture R and reticulated-Python deps in one lockfile.
```
