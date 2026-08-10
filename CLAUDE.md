# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

This is a **documentation/reference repository**, not a software project. It contains
no code, build, or test setup — only prose documents describing how to lay out
R/tidyverse analysis projects. Work here is editing Markdown, not running R.

The repo holds the **canonical conventions** and the **agent recipes** that operationalize
them. Nothing else. Superseded documents are deleted, not archived in place — `ATTIC.md`
lists them with the git ref each can be recovered from. Do not reintroduce an `originals/`
or `archive/` directory; if a document is retired, delete it and add a row to `ATTIC.md`.

**Canonical (the recommended output — edit these for layout guidance):**
- `docs/recommendedConvention.md` — the recommended layout for **mixed R + Python** projects.
  Code is split by language (`R/` + `python/`) over a shared, language-neutral data backbone;
  one numbered `scripts/` pipeline spans both languages.
- `docs/recommendedConventionROnly.md` — the recommended layout for **R-only** projects. The
  pipeline is all `.R`; Python, if needed, is reached in-process via `reticulate` (an embedded
  dependency, not a parallel codebase). Intermediates stay `.rds`; one `renv.lock`.

These two are a **pair**: pick by whether the project has standalone Python code. They
cross-reference each other in their intros — keep those pointers intact.

**Recipes (agent-executable — imperative procedures, a different genre from the canonical
reference docs; they link back to the canonical docs for the *why*):**
- `recipeNewProject.md` — instructions for an agent to scaffold a new project to the
  convention (greenfield).
- `recipeWrangleExisting.md` — instructions for an agent to migrate an existing project
  onto the convention (brownfield). It is **plan-then-migrate, gated**: read-only analysis →
  write `MIGRATION-PLAN.md` → STOP for human approval → migrate via `git mv` with git as the
  undo net. When editing it, preserve that approval gate and the "never modify `data/`" rule.

Both recipes branch internally on R-only vs. mixed (rather than splitting into four files)
and keep env/tooling setup as an explicitly optional step. When the convention docs change,
keep these recipes in sync with them.

**Retired sources:** the canonical docs were reconciled from a provenance-based
numbered-stage convention for plain `.R` scripts and from notes on the external `workflowr`
scheme. Both are now **fully absorbed** and deleted from the working tree; see `ATTIC.md`.
Treat the canonical docs as self-contained — do not cite `originals/`, and do not re-add
those files.

The earlier "competing conventions" tension is **resolved** in the canonical docs: the staged
`cache/` model wins over `workflowr`'s flat `output/`; `workflowr` is retained only as the
optional `analysis/`+`docs/` layer. Don't reintroduce that conflict.

## Core Conventions (shared by both canonical docs — preserve when editing)

- **Provenance split:** `data/` is read-only external input (never written by scripts);
  `cache/` is regenerable intermediate output; `results/` is final output. Read-only is about
  *writes*, not about *tracking* — see the gitignore rule below.
- **Numbered pipeline stages:** scripts and their output dirs share a prefix
  (`01_tidy_input.R` → `cache/run02/01_tidy/`), so the dependency graph is visible in
  filenames. In the mixed doc the numbering spans both languages. A `run_all.R` (R-only) or
  `run_all.sh` (mixed) at the root enforces the order the filenames only document.
- **The run axis is secondary to provenance, and separate from the stage axis.** A project
  runs more than once, so `cache/` and `results/` carry a run directory level — run outermost,
  stage one level down (`cache/run02/01_tidy/`, `results/run02/{figures,tables}`). **`data/`
  never gets a run level**, and a run is **never** a filename token: collapsing run and stage
  into one name (`proj_v1.01_raw.rds`) is the exact failure this axis exists to prevent. The
  current run is a top-level `run:` key in **`00.PARAMS.yml`** at the project root —
  deliberately language-neutral, deliberately standing alone rather than buried in an R config
  file, and required to be top-level so shell can `awk` it without a parser. Everything else
  in that file is the project's own namespace (`params:`, `args:`, `const:` — the convention
  does not care). The entry point copies `00.PARAMS.yml` into `results/<run>/` before stage
  01, which is the `MANIFEST.tsv` provenance argument applied to outputs. Bump `run:` when
  parameters change, not when fixing a bug; nothing enforces this and nothing should. Per-run
  `cache/` may be symlinked at a previous run's stage dir when that stage is genuinely
  unchanged.
- **Data is shared, code splits by language.** The data backbone is language-neutral; only
  code/environment dirs differ between the two docs.
- **No new dependencies / no proprietary interchange formats** (a hard constraint from the
  user). These are layout docs, not toolchain mandates. R-native intermediates use `.rds`;
  cross-language file handoff, when unavoidable, is **CSV or XLSX only** — never Parquet/Arrow.
  **Exactly one exception exists:** a YAML reader (`yaml` / `pyyaml`) for `00.PARAMS.yml`,
  taken knowingly because the alternative broke language neutrality. Do not treat it as
  precedent, and do not add a second.
- **R idioms in examples:** `here::here()` to build a directory from the project root, then
  `fs::path()` to join a filename onto it (never nest `here()` inside `here()`);
  `fs::dir_create()`; `read_csv(show_col_types = FALSE)`; `write_rds(..., compress = "gz")`.
  Match these and the tidyverse-first style in the user's global instructions.
- **The `.gitignore` split:** `cache/` and `results/*/figures/` ignored; `data/external/`,
  `results/*/tables/`, `results/*/00.PARAMS.yml`, the root `00.PARAMS.yml` and the lockfiles
  (`renv.lock`, `pyproject.toml`) tracked. The figures glob must keep its run wildcard — a
  bare `results/figures/` matches nothing now. Raw data is
  **ignore-the-bytes / commit-the-provenance**: `data/raw/*` ignored with
  `!data/raw/MANIFEST.tsv` un-ignored, plus `scripts/00_fetch_data.R` to rebuild it. This
  reversed an earlier "never gitignore `data/`" rule — do not flip it back.

## Working Notes

- This **is** a git repository (`origin git@github.com:soccin/r-proj-org.git`, default branch
  `master`). Work on a short-named branch; commit only when asked; never push unasked.
- Keep the four cross-referencing documents in sync. A convention change touches
  `README.md`, both `docs/recommendedConvention*.md`, `docs/projectLayoutNotes.md`, both
  recipes, and this file. Grep before declaring a change done.
- **Open question — how a released run is marked.** A run whose outputs went to a
  collaborator must never be overwritten; that rule is stated in both convention docs. The
  *mechanism* is deliberately unspecified pending the user's own experiments — candidates are
  a sub-folder inside the run dir, a git tag, and a `results/MANIFEST.tsv`. Do not invent one.
  Document-revision versioning (`METHODS_v1` → `v2`) is a third thing and needs no convention
  at all; resist formalizing it.
- The behavioral guidelines below still apply to any edits.

---

Behavioral guidelines to reduce common LLM coding mistakes. Merge with project-specific instructions as needed.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

