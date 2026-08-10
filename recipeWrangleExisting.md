# Recipe: Wrangle an Existing Project Into This Layout

**This is an instruction set for an AI agent.** Give it this file and point it at an existing
project to reorganize. The agent **plans first, stops for human approval, then migrates** —
it must not move files before approval.

The *why* lives in `docs/recommendedConvention.md` (R + Python) and
`docs/recommendedConventionROnly.md` (R-only). This recipe is the *how*.

## Operating rules (read before doing anything)

1. **Plan-then-migrate, gated.** Phases A–C are read-only analysis ending in a written plan.
   **STOP and get explicit human approval** before Phase D (the moves). Do not move, rename,
   or delete anything during A–C.
2. **Git is the safety net.** Migration uses `git mv` on a dedicated branch with a clean tree.
   If the project is **not** a git repo, say so and recommend `git init` + an initial commit
   first; do not perform moves on an untracked tree unless the user insists, and if they do,
   make a full backup copy first.
3. **`data/` is sacred.** Never move files *into* `data/raw` or `data/external` unless they
   are genuinely external inputs, and never *modify* a data file. When unsure whether a file
   is input (`data/`) or derived (`cache/`/`results/`), **ask** — do not guess. Untracking a
   raw file from git is not modifying it; `git rm --cached` leaves the bytes on disk. Never
   use plain `git rm` on anything under `data/`.
4. **No dependencies, no proprietary formats.** Don't convert files to Parquet/Arrow or add
   packages. Reorganizing location only. (Optional env setup is Phase E, separately approved.)
5. **Surgical.** Touch only what the migration requires. Don't reformat scripts or "improve"
   logic; the one allowed code edit is fixing file paths broken by the moves (Phase D).

---

## Phase A — Inventory (read-only) → verify

Survey the project without changing it:

- List the tree (depth ~3), note total size and the largest files/dirs.
- Detect **shape**: are there standalone `.py` files doing pipeline work (→ treat as **mixed**,
  target `docs/recommendedConvention.md`), or is it all `.R`/`.Rmd` (→ **R-only**, target
  `docs/recommendedConventionROnly.md`)? If `.py` exists only via `reticulate` calls inside `.R`,
  it's still R-only.
- Detect git: is it a repo? Is the working tree clean? Current branch?
- Find existing analysis dirs and anti-patterns: `data/processed/` or `data/interim/`
  (conflates immutable with derived), outputs written next to scripts, results committed
  without a `cache/` distinction, unnumbered scripts, hardcoded absolute paths,
  `setwd()` calls, no root anchor (`.Rproj`/`.here`), no entry point that runs the stages in
  order, and large binaries committed to git (check with
  `git rev-list --objects --all | git cat-file --batch-check` or simply the largest tracked
  files).
- **Detect the existing run/version scheme(s).** Look for `_vN`, `_NN`, `_YYMMDD`, an output
  filename prefix set in a config file, `attic/` or `old/` dirs, and per-run notes
  (`RUN_NOTES_01.md`, `RESULTS_01.md`). Expect **several at once with none authoritative** —
  that is the normal failure, and consolidating them onto one axis is part of this migration.
  Record which files each scheme touches; do not pick a winner yet.

**Verify:** state the detected shape (R-only vs mixed), git status, every run/version scheme
found, and a bullet list of the top structural problems.

---

## Phase B — Classify every file by provenance (read-only) → verify

This is the core analysis. For each non-trivial file/dir, assign a target tier:

| Current file looks like… | Target |
| --- | --- |
| Raw/received input, never regenerated | `data/raw/` |
| Reference data (annotations, lookups) | `data/external/` |
| Intermediate computed by a script, regenerable | `cache/<run>/NN_stage/` |
| Final figure | `results/<run>/figures/` |
| Final table/report data | `results/<run>/tables/` |
| Reusable function definitions (sourced, not run) | `R/` (or `python/<pkg>/` if mixed) |
| An ordered pipeline script | `scripts/` (numbered) |
| Exploratory notebook | `notebooks/` |

Rules while classifying:
- **Provenance is decided by how a file is produced, not its extension.** A `.csv` can be raw
  input *or* derived output — check whether a script writes it.
- **Anything a script writes is NOT `data/`.** Map `data/processed/`-style dirs to `cache/`.
- **Number the pipeline.** Order the run-once scripts and assign `01_`, `02_`, … prefixes;
  give each a matching `cache/<run>/NN_*` output dir name. In mixed projects the numbering
  spans both languages in run order.
- **Collapse the version schemes onto the run axis.** Everything already produced is one run
  unless the user says otherwise — default to `run01` and put all existing derived output
  under it. Where two schemes are genuinely two runs, propose a run id per set and say which
  files land where. **Strip the version token from the filenames** as they move
  (`out/Proj_v1.03_cluster.rds` → `cache/run01/03_cluster/cluster.rds`): the run is the
  directory now, and leaving it in the name recreates the collision this axis exists to
  prevent. Ask before merging any two schemes you cannot tell apart.
- Flag any file you cannot confidently classify as **NEEDS HUMAN INPUT**.

**Verify:** produce a complete classification with no unclassified non-trivial files (besides
the explicitly flagged ones).

---

## Phase C — Write the migration plan, then STOP → verify

Write `MIGRATION-PLAN.md` in the project root containing:

1. **Detected shape** and which convention doc is the target.
2. **Run mapping:** every version/run scheme found in Phase A, the run id each maps to, and
   the `00.PARAMS.yml` you will write (at minimum `run:`; carry over any parameters the old
   config file held). State plainly which existing outputs are being declared a single run.
   If any output set was **sent to a collaborator**, name it here — that run is frozen and
   must not be regenerated or renamed under a different id.
3. **Move table:** every `git mv FROM → TO`, grouped by target tier.
4. **Script-path edits:** the list of read/write paths inside scripts that the moves will
   break, with the intended `here::here(...)`-based replacement for each — including reading
   `run:` from `00.PARAMS.yml` and inserting it as a path segment. (Plan only — no edits yet.)
5. **`.gitignore` changes** to add (`cache/`, `results/*/figures/`, env dirs, and
   `data/raw/*` + `!data/raw/MANIFEST.tsv`), plus the list of already-tracked files that will
   need `git rm --cached`.
6. **Raw-data provenance:** the `data/raw/MANIFEST.tsv` you will generate (file, bytes, md5,
   source, date) and what is known about where each file came from. Flag every file whose
   source you cannot determine — an untracked raw file with no recorded origin is
   unrecoverable, so those must be resolved before Phase D, or the file stays tracked.
7. **NEEDS HUMAN INPUT** section: every file you couldn't classify, with your best guess and
   the question.
8. **Risks**: large files that will change git status, anything ambiguous, anything
   irreversible.

Then **STOP.** Output: "Plan written to MIGRATION-PLAN.md. Review it; resolve the NEEDS HUMAN
INPUT items; approve to proceed to migration." **Do not continue to Phase D until the human
approves.**

**Verify:** `MIGRATION-PLAN.md` exists, the move table is complete, and you have explicitly
halted for approval.

---

## Phase D — Migrate (only after approval) → verify

Preconditions: human approved; git repo with a **clean** working tree (commit/stash first if
not).

1. Create a branch: `git checkout -b chore/project-layout` (short name, per repo style).
2. Create the target skeleton dirs (mirror `recipeNewProject.md` Step 1, language-aware).
3. Execute the move table with **`git mv`** (preserves history). Move data/inputs first, then
   cache/results, then code, then notebooks.
4. Apply the planned **script-path edits** so scripts read/write the new locations via
   `here::here(...)`. This is the only code change allowed.
5. Apply the `.gitignore` changes. If derived outputs were previously committed, untrack them
   with `git rm -r --cached cache/ results/*/figures/` (keeps files on disk, stops tracking).
6. **Write `data/raw/MANIFEST.tsv` before untracking any raw bytes** — file, bytes, md5,
   source, date, one row per file. Then untrack the large ones with
   `git rm --cached <file>` (never plain `git rm`). Leave small inputs tracked and give each
   an explicit `!data/raw/<name>` un-ignore. Untracking removes them from the *next* commit
   only; bytes already in history stay there, so say so rather than implying the repo shrinks.
7. Add `data/README.md` (narrative provenance) and a `README.md` rebuild note if absent.
8. Write `00.PARAMS.yml` at the root with the agreed `run:` value, and copy it into that
   run's `results/<run>/` dir. Retire the old version tokens: delete a config file's output
   prefix setting only if a script still referencing it was updated in step 4.
9. Add a root anchor (`project.Rproj` or `.here`) and an entry point (`run_all.R` /
   `run_all.sh`) listing the renumbered stages in order, if absent.

Commit in logical chunks with clear messages (no emoji). Do **not** push unless asked; draft
any commit message for the user to review per their workflow.

**Verify after the moves:**
- Tree matches the target convention doc.
- `git status` shows only intended changes; `git mv` preserved history (`git log --follow`
  on a sample moved file).
- **Nothing in `data/` was modified** (only additions like `data/README.md` and
  `data/raw/MANIFEST.tsv`). Untracked raw files are still on disk — verify with `ls`, and
  verify each has a manifest row before considering the step done.
- If the pipeline is runnable, run the first one or two stages (or a `targets`/source of the
  scripts) to confirm the path edits work; report any breakage rather than papering over it.

---

## Phase E (OPTIONAL) — Environment setup → verify

Only if the user wants reproducibility locked in (separate approval; adds tooling):

- **R:** `renv::init()` → `renv.lock`.
- **Mixed:** add `pyproject.toml`/`requirements.txt`.
- **R-only + reticulate:** `renv::use_python()` (records Python in `renv.lock`), pin
  `RETICULATE_PYTHON` in `.Rprofile`.

**Verify:** lockfile(s) created; `renv::status()` clean.

---

## Done when

- Files sit in the correct provenance tiers; the pipeline is numbered and has an entry point;
  `data/` is untouched on disk; derived outputs and large raw bytes are ignored/untracked,
  with every untracked raw file recorded in `data/raw/MANIFEST.tsv`.
- **One run axis, not four.** Derived output sits under `cache/<run>/` and `results/<run>/`,
  `00.PARAMS.yml` names the run, and no version token survives in a filename.
- History is preserved (used `git mv`), everything is on a branch, and the change is
  reversible.
- A short report states what moved, what script paths changed, what was skipped, and any
  remaining NEEDS HUMAN INPUT items.
