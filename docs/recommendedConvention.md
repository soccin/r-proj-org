# Recommended Analysis Project Convention (R + Python)

For an **R-only** project (pipeline entirely in R; Python, if any, reached in-process via
`reticulate`), use the sibling document **`recommendedConventionROnly.md`** instead.

This reconciles the source documents in this repo into one recommended layout for
projects that mix **R and Python** code:

- `originals/r-project-organization.md` — provenance-based, numbered-stage pipeline for plain `.R` scripts.
- `originals/R Working Analysis Directory Tree Template.txt` — the `workflowr` convention for
  reproducible, publishable research websites built from `.Rmd`.

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
├── .Rprofile               # runs on open; load R libs, set RETICULATE_PYTHON if used
├── .gitignore
├── README.md               # what this is; how to rebuild it; which languages do what
│
│  ── Environment / dependencies (one set per language; see note) ──
├── renv.lock               # R package versions (renv)
├── pyproject.toml          # Python package + dependency spec
│   (or requirements.txt / environment.yml)
│
│  ── DATA: shared, language-neutral, named by provenance ──
├── data/                   # IMMUTABLE inputs — never written by scripts, never gitignored
│   ├── raw/                #   exactly as received
│   ├── external/           #   reference data (annotations, gene lists, ...)
│   └── README.md           #   provenance: where each file came from, when, from whom
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
- `data/` — came from *outside*; scripts (R or Python) only ever **read** it. Never gitignored.
- `cache/` — *derived*; scripts write it, you can delete and regenerate it. Gitignored.
- `results/` — *final* deliverables you actually report.

The names carry intent: anyone sees `cache/` and knows it's disposable. (Cookiecutter Data
Science encodes the same idea as `data/{raw,external,interim,processed}`; `interim` ≈ our
staged `cache/`, `processed` ≈ analysis-ready output.)

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
of working directory.

**6. Document provenance with a `data/README.md`.** Recording where each raw file came from
is the single highest-value reproducibility habit, and it's language-independent.

---

## Handing Data Between Stages

**Default: stay native within a language.** If a stage's output is consumed only by another
stage in the *same* language, use that language's native serialization — it's lossless and
needs nothing extra:

- R → R: `.rds` (`readr::write_rds()` / `read_rds()`) — preserves factors, dates, list cols.
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

# Language environments
renv/             # R project library (keep renv.lock, ignore the library)
.venv/            # Python virtualenv
__pycache__/
*.pyc

# NOT ignored:
#   data/           <- precious; version-control or document provenance in data/README.md
#   results/tables/ <- usually small; useful to track changes over time
#   renv.lock, pyproject.toml  <- the reproducibility record
```

---

## Reconciling the Two Source Docs

The custom doc and `workflowr` disagree in one place: `workflowr` puts all processed data
in a single flat `output/`; the custom doc uses staged, numbered `cache/`. **Prefer staged
`cache/`** — the custom doc's own argument holds: flat `output/` doesn't scale past two
stages and is ambiguous about immutability, whereas numbered stages scale to any depth and
stay self-documenting. If you adopt `workflowr` for its `.Rmd` → website tooling, redirect
its processed-data writes into `cache/NN_stage/` and keep `docs/` strictly for rendered HTML.

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
  — the `analysis/` + `docs/` literate-website convention (also captured in this repo's `.txt`).
- [Reproducible environments for R and Python](https://occasionaldivergences.com/posts/rep-env/)
  — `renv` + `reticulate` for joint R/Python dependency management.
