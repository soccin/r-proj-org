# Attic

Documents this repo used to carry and no longer does. Nothing here is lost — git has it.
This file exists so you don't have to remember that.

**The rule:** when a document is superseded, **delete it and add a row below.** Do not create
an `originals/`, `archive/`, `old/`, or `attic/` directory. A repository about organization
does not get to hoard files, and a directory of dead documents is indistinguishable from a
live one at a glance.

To read a retired file:

```sh
git show <ref>:<path>                      # print it
git show <ref>:<path> > /tmp/recovered.md  # save it
git log --all -- <path>                    # every commit that touched it
```

## Retired documents

| Path (as it was) | What it was | Recover from |
| --- | --- | --- |
| `docs/originals/r-project-organization.md` | The original provenance-based, numbered-stage convention for plain `.R` scripts. The backbone of both canonical docs traces to it. Superseded by the 2026-08-10 revision below. | `764fd8f` (on `origin/master`) |
| `docs/originals/R Working Analysis Directory Tree Template.txt` | Notes excerpted from the external [`workflowr` docs](https://jdblischak.github.io/workflowr/articles/wflow-01-getting-started.html) — the `analysis/` + `docs/` literate-website scheme. Absorbed into both canonical docs as the optional reporting layer. | `764fd8f` (on `origin/master`), or the upstream URL |
| `docs/rProjectOrganization.md` | The 2026-08-10 revision of the base convention doc: manifest-based raw-data tracking, one-directional `data/` rule, `.here` anchor guidance, `run_all.R`, `fs::path()` joins, `compress = "gz"`. Content folded into both canonical docs. | tag `attic-2026-08-10` |

## Why these went away

The base convention doc had drifted into seven copies across Dropbox, a backup, and two
working directories — six of them byte-identical to a stale ancestor, one carrying real
corrections that existed nowhere else. Keeping a frozen `originals/` copy in the repo was
part of what made that feel normal.

The two canonical convention docs are now **self-contained**. They do not cite source
documents, because there are no source documents to cite — the reconciliation is finished and
the inputs are history. That is what git is for.
