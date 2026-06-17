# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

This is a **documentation/reference repository**, not a software project. It contains
no code, build, or test setup — only prose documents describing how to lay out
R/tidyverse analysis projects. Work here is editing Markdown, not running R.

The repo holds **source documents**, the **reconciled conventions** derived from them, and
**agent recipes** that operationalize the conventions.

**Canonical (the recommended output — edit these for layout guidance):**
- `RECOMMENDED-CONVENTION.md` — the recommended layout for **mixed R + Python** projects.
  Code is split by language (`R/` + `python/`) over a shared, language-neutral data backbone;
  one numbered `scripts/` pipeline spans both languages.
- `RECOMMENDED-CONVENTION-R-ONLY.md` — the recommended layout for **R-only** projects. The
  pipeline is all `.R`; Python, if needed, is reached in-process via `reticulate` (an embedded
  dependency, not a parallel codebase). Intermediates stay `.rds`; one `renv.lock`.

These two are a **pair**: pick by whether the project has standalone Python code. They
cross-reference each other in their intros — keep those pointers intact.

**Recipes (agent-executable — imperative procedures, a different genre from the canonical
reference docs; they link back to the canonical docs for the *why*):**
- `RECIPE-NEW-PROJECT.md` — instructions for an agent to scaffold a new project to the
  convention (greenfield).
- `RECIPE-WRANGLE-EXISTING.md` — instructions for an agent to migrate an existing project
  onto the convention (brownfield). It is **plan-then-migrate, gated**: read-only analysis →
  write `MIGRATION-PLAN.md` → STOP for human approval → migrate via `git mv` with git as the
  undo net. When editing it, preserve that approval gate and the "never modify `data/`" rule.

Both recipes branch internally on R-only vs. mixed (rather than splitting into four files)
and keep env/tooling setup as an explicitly optional step. When the convention docs change,
keep these recipes in sync with them.

**Sources (background — the inputs that were reconciled; do not present as competing anymore;
they live in `originals/`):**
- `originals/r-project-organization.md` — the original provenance-based, numbered-stage
  convention for plain `.R` scripts. The backbone of both canonical docs traces to this.
- `originals/R Working Analysis Directory Tree Template.txt` — notes excerpted from the external
  `workflowr` package docs (the `analysis/`+`docs/` literate-website scheme), folded into the
  canonical docs as an optional reporting layer.

The earlier "competing conventions" tension is **resolved** in the canonical docs: the staged
`cache/` model wins over `workflowr`'s flat `output/`; `workflowr` is retained only as the
optional `analysis/`+`docs/` layer. Don't reintroduce that conflict.

## Core Conventions (shared by both canonical docs — preserve when editing)

- **Provenance split:** `data/` is read-only external input (never written by scripts, never
  gitignored); `cache/` is regenerable intermediate output; `results/` is final output.
- **Numbered pipeline stages:** scripts and their output dirs share a prefix
  (`01_tidy_input.R` → `cache/01_tidy/`), so the dependency graph is visible in filenames.
  In the mixed doc the numbering spans both languages.
- **Data is shared, code splits by language.** The data backbone is language-neutral; only
  code/environment dirs differ between the two docs.
- **No new dependencies / no proprietary interchange formats** (a hard constraint from the
  user). These are layout docs, not toolchain mandates. R-native intermediates use `.rds`;
  cross-language file handoff, when unavoidable, is **CSV or XLSX only** — never Parquet/Arrow.
- **R idioms in examples:** `here::here()` for paths, `fs::dir_create()`,
  `read_csv(show_col_types = FALSE)`. Match these and the tidyverse-first style in the user's
  global instructions.
- **The `.gitignore` split:** `cache/` and `results/figures/` ignored; `data/` and
  `results/tables/` tracked; lockfiles (`renv.lock`, `pyproject.toml`) tracked.

## Working Notes

- There is no `git` repository here yet. Do not assume version control commands work.
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

