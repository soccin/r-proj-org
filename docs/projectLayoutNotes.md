# R / R+Python Project-Layout Notes (carry-along)

A portable companion to the convention and recipe docs in this folder. Keep this file
*with* those docs when you copy them elsewhere. It records the standing rules behind them so
the guidance survives even though the original working directory was throwaway.

This is reference for a human or an agent. Nothing here auto-loads; point an agent at this
file (and the docs it names) when starting or wrangling a project.

## What's in this doc set

`../CLAUDE.md` (kept with these files) documents the full set. In short, it is two tiers:

1. **Canonical conventions** — `recommendedConvention.md` (R + Python) and
   `recommendedConventionROnly.md` (R-only). A pair; pick by whether there's standalone
   Python. Both are self-contained.
2. **Agent recipes** (at the repo root) — `../recipeNewProject.md` (greenfield) and
   `../recipeWrangleExisting.md` (brownfield; plan-then-migrate, gated).

Superseded documents are deleted rather than archived in place; `../ATTIC.md` records what
they were and which git ref holds them.

See `../CLAUDE.md` for the per-file detail; this note does not duplicate it.

## Standing constraints (the part worth carrying anywhere)

These are how I want R / data work organized — layout guidance, **not** a toolchain mandate.
They apply to these docs and to any project built or migrated from them:

- **Add no dependencies** to satisfy layout. The docs organize folders/files; they must not
  require new packages.
- **No proprietary/binary interchange formats** — explicitly **no Parquet, no Arrow/Feather**.
- **Intermediates:** R-native = `.rds`. Cross-language *file* handoff = **CSV or XLSX only**.
  (In an R-only project that calls Python via `reticulate`, data crosses in memory — there is
  no file seam at all.)
- **`reticulate` is fine** for an R-only project to borrow a Python library R lacks; record
  its Python env in `renv.lock` rather than standing up a separate Python toolchain.
- **Raw data: ignore the bytes, commit the provenance.** `data/raw/*` is gitignored with
  `!data/raw/MANIFEST.tsv` un-ignored, and `scripts/00_fetch_data.R` rebuilds the directory.
  Small inputs (sample sheets, gene lists) get an explicit un-ignore and go in git directly.

**Why:** these docs should organize a project, not dictate tools or pull in dependencies;
CSV/XLSX are universally readable from base R and base Python. And git history is
append-only — one accidental multi-gigabyte commit inflates every future clone forever, so
large inputs are reconstructed from a manifest rather than stored.

## Core layout idea (one-paragraph recall)

Provenance is the primary axis: `data/` is read-only external input (never written by
scripts), `cache/` is regenerable intermediate output (gitignored), `results/` is final
output. Number the pipeline stages so a script and its output dir share a prefix
(`02_process.R` → `cache/02_processed/`), and give the project a `run_all.R` / `run_all.sh`
entry point that runs them in order. Anchor the root so `here::here()` resolves (`.Rproj`, or
a bare `.here` from `here::set_here()`). Track `data/external/` and `results/tables/`; ignore
`cache/`, `results/figures/`, and the raw bytes under `data/raw/` — those are reconstructed
from `MANIFEST.tsv` plus `scripts/00_fetch_data.R`. Data is shared and language-neutral; only
the code layer splits by language. Staged `cache/` is preferred over both `data/processed/`
and a flat `output/`; `workflowr` is retained only as an optional `analysis/`+`docs/`
reporting layer.
