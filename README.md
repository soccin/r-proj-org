# R / R+Python Analysis Project Layout

Opinionated, reusable conventions — plus agent-runnable recipes — for laying out
**R** and **mixed R+Python** data-analysis projects. The organizing idea: structure a
project around the **provenance** of its files (immutable inputs vs. regenerable
intermediates vs. final outputs), with a numbered pipeline whose dependency graph is
readable from the filenames.

This repository is documentation only. There is no code to build or run here; the
deliverables are the Markdown documents below.

## Start here

**Pick a convention** (the *what* and *why*):

- [`docs/recommendedConvention.md`](docs/recommendedConvention.md) — for **mixed R + Python**
  projects. Code splits by language (`R/` + `python/`) over a shared, language-neutral data
  backbone; one numbered `scripts/` pipeline spans both.
- [`docs/recommendedConventionROnly.md`](docs/recommendedConventionROnly.md) — for **R-only**
  projects. The pipeline is all `.R`; Python, if needed, is reached in-process via
  `reticulate`.

**Run a recipe** (the *how* — written for an AI agent to execute):

- [`recipeNewProject.md`](recipeNewProject.md) — scaffold a new project to the convention.
- [`recipeWrangleExisting.md`](recipeWrangleExisting.md) — migrate an existing project onto
  it. Plan-then-migrate and gated: it analyzes, writes a migration plan, and **stops for your
  approval** before moving anything.

## Repository contents

| Path | What it is |
| --- | --- |
| `recipeNewProject.md` | Agent recipe — greenfield scaffold |
| `recipeWrangleExisting.md` | Agent recipe — brownfield migration |
| `docs/recommendedConvention.md` | Canonical layout — R + Python |
| `docs/recommendedConventionROnly.md` | Canonical layout — R-only |
| `docs/projectLayoutNotes.md` | Portable carry-along summary + the standing constraints |
| `CLAUDE.md` | Guidance for AI agents working *in this repo* |
| `ATTIC.md` | Retired documents and the refs they can be recovered from |

## The core idea, in brief

- `data/` — immutable inputs; scripts only read it, never write it.
- `cache/` — regenerable intermediates; scripts write it, safe to delete (gitignored).
- `results/` — final figures and tables.
- Number pipeline stages so a script and its output dir share a prefix
  (`02_process.R` → `cache/02_processed/`), and give the project an entry point
  (`run_all.R` / `run_all.sh`) that runs them in order.
- For large raw inputs, **ignore the bytes and commit the provenance**: `data/raw/*` is
  gitignored except `data/raw/MANIFEST.tsv`, and `scripts/00_fetch_data.R` rebuilds the
  directory from source.
- Data is shared and language-neutral; only the code layer differs between R-only and mixed.

See [`docs/projectLayoutNotes.md`](docs/projectLayoutNotes.md) for the standing constraints these
docs obey (notably: no added dependencies, and no proprietary interchange formats — `.rds`
within R, CSV/XLSX only at a cross-language boundary).
