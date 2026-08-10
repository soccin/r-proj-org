# Recommended Analysis Project Convention (R + Python)

For an **R-only** project (pipeline entirely in R; Python, if any, reached in-process via
`reticulate`), use the sibling document **`recommendedConventionROnly.md`** instead.

This is the recommended layout for projects that mix **R and Python** code. It is
self-contained: it absorbs the provenance-based, numbered-stage pipeline convention and the
relevant parts of the `workflowr` scheme. The retired source documents those came from are
listed in `../ATTIC.md`.

It is also informed by the widely used Python layout,
[Cookiecutter Data Science](https://cookiecutter-data-science.drivendata.org/), whose
`data/{raw, external, interim, processed}` split maps almost one-to-one onto the
provenance model below. That convergence is the whole point: **the data layer is
language-neutral, so it stays shared; only the code layer splits by language.**

> **Backbone:** provenance-based directories (`data/` → `cache/` → `results/`), shared by
> both languages, with numbered pipeline stages. **Code** lives in language-idiomatic dirs
> (`R/`, `python/`). **Handoff** between stages stays in each language's native format
> unless data must cross the R/Python boundary, in which case use **CSV or XLSX** — no
> proprietary formats, no new dependencies.

---

## The Insight: Data Is Shared, Code Is Not

A mixed-language project has two layers with opposite needs:

- **Data lifecycle** is *language-agnostic*. A `.csv` in `data/raw/` is the same file
  whether R or Python reads it. So the data directories are **shared** and named by
  *provenance*, not by language.
- **Code and environments** are *language-specific*. R wants `R/`, `DESCRIPTION`,
  `renv.lock`; Python wants a package dir and `pyproject.toml`. Forcing them together
  fights both ecosystems. So code is **split by language**.

Keeping these layers separate is what lets one project hold both languages without mess.

---

## Recommended Structure

```
project/
├── project.Rproj           # project-root anchor (keep even outside RStudio)
│                           #   no RStudio? a bare .here file does the same job for R
├── .Rprofile               # runs on open; load R libs, set RETICULATE_PYTHON if used
├── .gitignore
├── README.md               # what this is; how to rebuild it; which languages do what
├── run_all.sh              # runs the stages in order — the pipeline entry point
│
│  ── Environment / dependencies (one set per language; see note) ──
├── renv.lock               # R package versions (renv)
├── pyproject.toml          # Python package + dependency spec
│   (or requirements.txt / environment.yml)
│
│  ── DATA: shared, language-neutral, named by provenance ──
├── data/                   # IMMUTABLE inputs — never written by scripts
│   ├── raw/                #   exactly as received
│   │   └── MANIFEST.tsv    #     provenance: name, size, md5, source, date
│   ├── external/           #   reference data (annotations, gene lists, ...)
│   └── README.md           #   narrative: where the data came from, from whom, caveats
│
├── cache/                  # REGENERABLE intermediates — safe to delete, gitignored
│   ├── 01_tidy/            #   output dir prefix matches the script that made it
│   ├── 02_processed/
│   └── 03_features/
│
├── results/                # FINAL outputs
│   ├── figures/            #   often large → gitignore
│   └── tables/             #   often small → track in git
│
│  ── CODE: split by language ──
├── R/                      # reusable R functions (sourced), NOT run as scripts
├── python/                 # importable Python package / modules, NOT run as scripts
│
├── scripts/                # the ordered pipeline — numbered across BOTH languages
│   ├── 00_fetch_data.R     #   run once — rebuilds data/raw/ from its sources
│   ├── 01_tidy_input.R     #   reads data/raw/      → writes cache/01_tidy/
│   ├── 02_process.py       #   reads cache/01_tidy/ → writes cache/02_processed/
│   └── 03_features.py      #   reads cache/02_...   → writes cache/03_features/
│
├── notebooks/              # OPTIONAL exploratory work (.ipynb and/or .Rmd)
│                           #   name: NN_initials_topic, e.g. 01_ns_data-overview
│
└── analysis/               # OPTIONAL literate report layer (workflowr-style .Rmd)
    └── docs/               #   rendered HTML/figures (only if publishing a site)
```

The `notebooks/` and `analysis/` blocks are **optional** — include them only when you
need exploration scratch space or a published report. The backbone above
(`data/` → `scripts/` → `cache/` → `results/`, with `R/` + `python/`) stands alone.

---

## The Principles

**1. Provenance is the primary axis — three write-tiers, shared by both languages.**
- `data/` — came from *outside*; scripts (R or Python) only ever **read** it.
- `cache/` — *derived*; scripts write it, you can delete and regenerate it. Gitignored.
- `results/` — *final* deliverables you actually report.

The names carry intent: anyone sees `cache/` and knows it's disposable. Read-only is a
property of `data/`, not of whether it is tracked — see the `.gitignore` section for what
gets committed. (Cookiecutter Data Science encodes the same idea as
`data/{raw,external,interim,processed}`; `interim` ≈ our staged `cache/`, `processed` ≈
analysis-ready output.)

**2. Number the pipeline stages — across both languages.** A stage's script and its output
dir share a prefix (`02_process.py` → `cache/02_processed/`), and stages are numbered in
run order regardless of which language each uses. The dependency graph and the
language-handoff points are both visible in the filenames. A reader sees `01_*.R` then
`02_*.py` and knows R hands off to Python there.

**3. Read-only vs. write-enabled stays strict.** Every script reads from `data/`/`cache/`
and writes to `cache/`/`results/`. Never write into `data/`. This keeps the flow auditable
no matter the language.

**4. Code splits by language; the pipeline order does not.**
- `R/` — reusable R functions you `source()`. `python/` — importable modules you `import`.
- `scripts/` — the single ordered pipeline, mixing `.R` and `.py` files. One numbered
  sequence, not one-per-language, so the end-to-end order is unambiguous.
- Keep heavy reusable logic in `R/` or `python/`; keep `scripts/` thin and sequential.

**5. Anchor the project root.** A `.Rproj` (or equivalent root marker) plus `.Rprofile`
means `here::here()` (R) and project-relative paths (Python) resolve identically regardless
of working directory. On the R side `here()` finds the root by searching upward for that
anchor; outside RStudio, `here::set_here()` writes a bare `.here` file that serves the same
purpose. Without an anchor `here()` falls back to the working directory and every path
silently resolves somewhere else.

**6. Document provenance.** `data/raw/MANIFEST.tsv` records one row per raw file — name,
size, md5, source (URL or accession), date received — and `data/README.md` carries the
narrative a table cannot. Both are plain text and language-independent. This is the single
highest-value reproducibility habit, and once raw bytes are untracked (below) the manifest
is the only record that they were what you think they were.

---

## Handing Data Between Stages

**Default: stay native within a language.** If a stage's output is consumed only by another
stage in the *same* language, use that language's native serialization — it's lossless and
needs nothing extra:

- R → R: `.rds` (`readr::write_rds()` / `read_rds()`) — preserves factors, dates, list cols.
  Pass `compress = "gz"`: `readr::write_rds()` defaults to `compress = "none"`, unlike base
  `saveRDS()`, so intermediates otherwise land several times larger than expected.
- Python → Python: the project's normal in-language format (e.g. pickle, feather-if-already-present).

**Crossing the R ↔ Python boundary: use CSV or XLSX.** When an R stage feeds a Python stage
(or vice-versa), write the handoff file as **CSV** (preferred) or **XLSX**. Both are
universally readable from base R and base Python tooling.

- **Do not introduce proprietary or binary interchange formats** (e.g. Parquet, Arrow/Feather,
  `.rds` consumed from Python) just to move data across languages. This layout intentionally
  **adds no dependencies** — it organizes folders and files, it does not dictate a toolchain.
- CSV is the default; reach for XLSX only when the consumer genuinely needs sheets or the
  data is small and human-inspected. Be aware CSV/XLSX lose strict type fidelity (factors,
  precise numeric/date types) — re-assert types on read at the boundary.

So a typical seam looks like:

```
cache/01_tidy/samples_tidy.rds      # R wrote it, an R stage reads it     (native)
cache/01_tidy/samples_for_py.csv    # R wrote it, a Python stage reads it (handoff → CSV)
```

---

## Running the Pipeline

The numbered filenames *document* the execution order. An entry point *enforces* it. Because
the stages span two languages, a plain shell script is the honest choice — it adds nothing
and runs each stage with the right interpreter:

```sh
#!/usr/bin/env bash
# run_all.sh — run from the project root
set -euo pipefail

# The project's interpreter, not whatever `python` PATH happens to resolve to.
PYTHON="${PYTHON:-.venv/bin/python}"

# Rscript scripts/00_fetch_data.R      # run once — populates data/raw/

Rscript scripts/01_tidy_input.R
"$PYTHON" scripts/02_process.py
"$PYTHON" scripts/03_features.py
```

A bare `python` is the one thing worth being careful about here. It resolves against `PATH`,
which in a fresh shell is usually the system interpreter — none of the deps in
`pyproject.toml`, and on macOS possibly no `python` at all. Pointing at the project's venv
makes the stages run against the environment the lockfiles describe. If you use conda or put
the venv elsewhere, override it (`PYTHON=$(which python) ./run_all.sh`) or activate the
environment before running. `Rscript` needs no equivalent because `renv` activates the
project library from `.Rprofile` on startup.

Keeping this file working is the cheapest reproducibility check available: if it does not
run end to end in a fresh session, the project is not reproducible, whatever the directory
structure suggests. (In an R-only project this is a `run_all.R` that `source()`s each stage
— see the sibling document.)

---

## Environments / Dependencies

Each language manages its own dependencies; both lockfiles live at the project root.

- **R:** `renv` — `renv::init()`, `renv::snapshot()`, `renv::restore()`; produces `renv.lock`.
- **Python:** `pyproject.toml` (or `requirements.txt` / `environment.yml`) with your venv/conda tool.

If you call Python *from within R* via `reticulate`, `renv::use_python()` can record the
Python dependencies in `renv.lock` alongside the R ones, and `.Rprofile` can pin
`RETICULATE_PYTHON`. That is an optional integration, not a requirement — the split-by-language
layout above works with the two ecosystems kept entirely separate.

---

## `.gitignore`

```
# Derived — regenerate from scripts
cache/
results/figures/

# Rendered report (only if using analysis/)
analysis/docs/

# Raw data: ignore the bytes, commit the provenance
data/raw/*
!data/raw/MANIFEST.tsv

# Language environments
renv/             # R project library (keep renv.lock, ignore the library)
.venv/            # Python virtualenv
__pycache__/
*.pyc

# NOT ignored:
#   data/external/  <- reference files are usually small; commit them
#   results/tables/ <- usually small; useful to track changes over time
#   renv.lock, pyproject.toml  <- the reproducibility record
```

Large inputs — FASTQs, BAMs, count matrices — have to stay out of git. History is
append-only, so a single accidental multi-gigabyte commit permanently inflates every future
clone, and deleting the file later does not undo it.

Ignoring the bytes does not mean giving up the guarantee. Commit two small things instead:

- **`data/raw/MANIFEST.tsv`** — one row per file: name, size, md5, source (URL or
  accession), date received.
- **`scripts/00_fetch_data.R`** — the code that rebuilds `data/raw/` from those sources.

Together they run to a few kilobytes and reconstruct the input state exactly, which is what
committing the files was meant to buy in the first place.

Small inputs are the exception worth naming: a 200-row gene list, a sample sheet, a config
table all belong in git directly. Add an explicit un-ignore for each
(`!data/raw/samplesheet.csv`). The rule is about size, not about which directory a file
lives in.

One mechanical detail: the pattern must be `data/raw/*`, not `data/raw/`. Git cannot
re-include a file whose parent *directory* is excluded, so ignoring the directory itself
would silently make the `!data/raw/MANIFEST.tsv` line a no-op.

---

## Why Staged `cache/`, Not `data/processed` or a Flat `output/`

Two common alternatives, and why this layout rejects both.

**`data/processed/`** (and Cookiecutter's `data/interim` + `data/processed`) conflates the
immutability guarantee of `data/` — a newcomer, or future-you, no longer knows what is safe
to delete. It also doesn't scale past two stages; you end up with `data/processed_v2/` or
timestamped directory names.

**`workflowr`'s flat `output/`** has the same scaling problem and is equally ambiguous about
immutability. Numbered `cache/` stages scale to any depth and stay self-documenting. If you
adopt `workflowr` for its `.Rmd` → website tooling, redirect its processed-data writes into
`cache/NN_stage/` and keep `docs/` strictly for rendered HTML.

---

## When to Use Which Layer

| You are doing… | Use |
| --- | --- |
| Pure scripted processing (R and/or Python), no report | backbone only (`data/`→`scripts/`→`cache/`→`results/`) |
| The above, plus exploration | add `notebooks/` |
| The above, plus a narrative report or published site | add the `analysis/` + `docs/` layer (`workflowr`) |
| A one-off exploratory script | a single script reading `data/`, writing `results/` — don't over-build |

Don't adopt heavy machinery (full `workflowr`, `reticulate` in-process bridging, extra
interchange formats) unless the project actually needs it. Start from the shared data
backbone plus per-language code dirs; add layers only when a real need appears.

---

## Sources

- [Cookiecutter Data Science](https://cookiecutter-data-science.drivendata.org/) — the
  canonical Python data-science layout (`data/{raw,external,interim,processed}`, module dir,
  numbered notebooks, immutable raw data).
- [workflowr getting started](https://jdblischak.github.io/workflowr/articles/wflow-01-getting-started.html)
  — the `analysis/` + `docs/` literate-website convention.
- [Reproducible environments for R and Python](https://occasionaldivergences.com/posts/rep-env/)
  — `renv` + `reticulate` for joint R/Python dependency management.
