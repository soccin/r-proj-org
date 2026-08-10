# Recommended Analysis Project Convention (R-only)

The **R-only** version of this repo's layout, for projects whose pipeline is written
entirely in R. Python, when needed, is reached *in-process from R* via
[`reticulate`](https://rstudio.github.io/reticulate/) — it is an embedded dependency, not a
second codebase.

> For projects with substantial standalone Python code (its own `.py` pipeline stages and
> `pyproject.toml`), use the sibling document **`recommendedConvention.md`** (R + Python,
> split by language) instead. This document is the simpler, single-language case.

This document is self-contained: it absorbs the provenance-based, numbered-stage pipeline
convention and the relevant parts of the `workflowr` scheme. The retired source documents
those came from are listed in `../ATTIC.md`.

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
│                           #   no RStudio? a bare .here file does the same job
├── .Rprofile               # runs on open; load R libs; pin RETICULATE_PYTHON if using reticulate
├── .gitignore
├── README.md               # what this is; how to rebuild it
├── run_all.R               # sources scripts in order — the pipeline entry point
│
├── renv.lock               # R packages — AND Python packages if reticulate is used
│
│  ── DATA: named by provenance ──
├── data/                   # IMMUTABLE inputs — never written by scripts
│   ├── raw/                #   exactly as received
│   │   └── MANIFEST.tsv    #     provenance: name, size, md5, source, date
│   ├── external/           #   reference data (annotations, gene lists, ...)
│   └── README.md           #   narrative: where the data came from, from whom, caveats
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
│   ├── 00_fetch_data.R     #   run once — rebuilds data/raw/ from its sources
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
- `data/` — came from *outside*; scripts only ever **read** it.
- `cache/` — *derived*; scripts write it (as `.rds`), you can delete and regenerate it. Gitignored.
- `results/` — *final* deliverables you actually report.

The names carry intent: anyone sees `cache/` and knows it's disposable. Read-only is a
property of `data/`, not of whether it is tracked — see the `.gitignore` section for what
gets committed.

**2. Number the pipeline stages.** A stage's script and its output dir share a prefix
(`02_process.R` → `cache/02_processed/`). Six months later the dependency graph is readable
from filenames alone.

**3. Read-only vs. write-enabled stays strict.** Every script reads from `data/`/`cache/`
and writes to `cache/`/`results/`. Never write into `data/`.

**4. Functions in `R/`, pipeline in `scripts/`.** `R/` holds reusable functions you
`source()` — including any thin wrappers around reticulated Python calls. `scripts/` holds
the ordered, run-once `.R` pipeline. Keep heavy logic out of the numbered scripts.

**5. Anchor the project root.** `here::here()` finds the root by searching upward for an
anchor file. A `.Rproj` is that anchor; outside RStudio, run `here::set_here()` once to write
an empty `.here` file instead. Without an anchor `here()` silently falls back to the working
directory and every path in the project resolves somewhere else — set this up first, since
everything below depends on it. If using reticulate, `.Rprofile` is also where you pin the
interpreter (`Sys.setenv(RETICULATE_PYTHON = ...)` or `reticulate::use_python()`).

**6. Document provenance.** `data/raw/MANIFEST.tsv` records one row per raw file — name,
size, md5, source (URL or accession), date received — and `data/README.md` carries the
narrative a table cannot. This is the single highest-value reproducibility habit, and once
raw bytes are untracked (below) the manifest is the only record that they were what you
think they were.

---

## In Practice

Define paths at the top of each script so they work regardless of working directory:

```r
library(here)

# Inputs
input_dir  <- here("data", "raw")
ref_dir    <- here("data", "external")

# Outputs
out_dir    <- here("cache", "01_tidy")
fs::dir_create(out_dir)   # safe no-op if it exists

# Read
raw <- read_csv(fs::path(input_dir, "samples.csv"), show_col_types = FALSE)

# ... transform ...

# Write
write_rds(tidy_data, fs::path(out_dir, "samples_tidy.rds"), compress = "gz")
```

Use `here()` once per path, to build the directory from the project root, then `fs::path()`
to join filenames onto it. Passing an already-absolute path back through `here()` happens to
work, but it reads as though the path is being re-anchored when it isn't.

---

## Running the Pipeline

The numbered filenames *document* the execution order. `run_all.R` *enforces* it:

```r
# run_all.R
library(here)

# source(here("scripts", "00_fetch_data.R"))   # run once — populates data/raw/

source(here("scripts", "01_tidy_input.R"))
source(here("scripts", "02_process.R"))
source(here("scripts", "03_analyze.R"))
```

Run it with `Rscript run_all.R` from anywhere in the project. Keeping this file working is
the cheapest reproducibility check available: if it does not run end to end in a fresh
session, the project is not reproducible, whatever the directory structure suggests.

---

## Intermediates & Handoff

- **Within the pipeline: `.rds`.** `readr::write_rds()` / `read_rds()` preserves types
  (factors, dates, list columns) exactly and is fast. Every `cache/` file is R-written,
  R-read. Note that `readr::write_rds()` defaults to `compress = "none"`, unlike base
  `saveRDS()`, which gzips by default — set it explicitly or intermediates land several
  times larger than expected.
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

# Raw data: ignore the bytes, commit the provenance
data/raw/*
!data/raw/MANIFEST.tsv

# R + reticulate environment
renv/             # project library (keep renv.lock, ignore the library)
# reticulate's Python env lives under renv/ when created via renv::use_python()

# NOT ignored:
#   data/external/  <- reference files are usually small; commit them
#   results/tables/ <- usually small; useful to track changes over time
#   renv.lock       <- the reproducibility record (R + Python)
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

**`data/processed/`** conflates the immutability guarantee of `data/` — a newcomer, or
future-you, no longer knows what is safe to delete. It also doesn't scale past two stages;
you end up with `data/processed_v2/` or timestamped directory names.

**`workflowr`'s flat `output/`** has the same scaling problem and is equally ambiguous about
immutability. Numbered `cache/` stages scale to any depth and stay self-documenting. If you
adopt `workflowr` for its `.Rmd` → website tooling (a natural fit for an R-only project),
redirect its processed-data writes into `cache/NN_stage/` and keep `docs/` strictly for
rendered HTML.

---

## When to Use Which Layer

| You are doing… | Use |
| --- | --- |
| Pure scripted R processing, no report | backbone only (`data/`→`scripts/`→`cache/`→`results/`) |
| The above, plus exploration | add `notebooks/` (`.Rmd`/`.qmd`) |
| The above, plus a narrative report or published site | add the `analysis/` + `docs/` layer (`workflowr`) |
| Need a Python library R lacks | call it from an R stage via `reticulate`; record it in `renv.lock` |
| Substantial standalone Python code / `.py` pipeline stages | use the R+Python doc (`recommendedConvention.md`) instead |
| A one-off exploratory script | a single script reading `data/`, writing `results/` — don't over-build |

---

## Sources

- [workflowr getting started](https://jdblischak.github.io/workflowr/articles/wflow-01-getting-started.html)
  — the `analysis/` + `docs/` literate-website convention (also captured in this repo's `.txt`).
- [reticulate: R interface to Python](https://rstudio.github.io/reticulate/) — calling Python
  in-process from R.
- [Reproducible environments for R and Python](https://occasionaldivergences.com/posts/rep-env/)
  — using `renv` (with `renv::use_python()`) to capture R and reticulated-Python deps in one lockfile.
