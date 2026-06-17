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

- [`RECOMMENDED-CONVENTION.md`](RECOMMENDED-CONVENTION.md) — for **mixed R + Python**
  projects. Code splits by language (`R/` + `python/`) over a shared, language-neutral data
  backbone; one numbered `scripts/` pipeline spans both.
- [`RECOMMENDED-CONVENTION-R-ONLY.md`](RECOMMENDED-CONVENTION-R-ONLY.md) — for **R-only**
  projects. The pipeline is all `.R`; Python, if needed, is reached in-process via
  `reticulate`.

**Run a recipe** (the *how* — written for an AI agent to execute):

- [`RECIPE-NEW-PROJECT.md`](RECIPE-NEW-PROJECT.md) — scaffold a new project to the convention.
- [`RECIPE-WRANGLE-EXISTING.md`](RECIPE-WRANGLE-EXISTING.md) — migrate an existing project onto
  it. Plan-then-migrate and gated: it analyzes, writes a migration plan, and **stops for your
  approval** before moving anything.

## Repository contents

| Path | What it is |
| --- | --- |
| `RECOMMENDED-CONVENTION.md` | Canonical layout — R + Python |
| `RECOMMENDED-CONVENTION-R-ONLY.md` | Canonical layout — R-only |
| `RECIPE-NEW-PROJECT.md` | Agent recipe — greenfield scaffold |
| `RECIPE-WRANGLE-EXISTING.md` | Agent recipe — brownfield migration |
| `PROJECT-LAYOUT-NOTES.md` | Portable carry-along summary + the standing constraints |
| `CLAUDE.md` | Guidance for AI agents working *in this repo* |
| `originals/` | The two source documents the conventions were reconciled from |

## The core idea, in brief

- `data/` — immutable inputs; scripts only read it, never write it (never gitignored).
- `cache/` — regenerable intermediates; scripts write it, safe to delete (gitignored).
- `results/` — final figures and tables.
- Number pipeline stages so a script and its output dir share a prefix
  (`02_process.R` → `cache/02_processed/`).
- Data is shared and language-neutral; only the code layer differs between R-only and mixed.

See [`PROJECT-LAYOUT-NOTES.md`](PROJECT-LAYOUT-NOTES.md) for the standing constraints these
docs obey (notably: no added dependencies, and no proprietary interchange formats — `.rds`
within R, CSV/XLSX only at a cross-language boundary).
