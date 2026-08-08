# Merge instructions for GitHub Copilot

You are applying one reviewed change to three existing repositories: **replacing a
per-work-type shell composer with a shared library driven by declaration.**

The change set is complete and verified. Your job is to transplant it accurately,
not to redesign it.

Read this whole file before editing. The "Why" section exists because two of these
choices look wrong at first glance and are not. Undoing one silently breaks
something that currently passes.

---

## Baseline

This change applies on top of:

| Repository | Current state |
|------------|---------------|
| `sketch-base` | bundle **0.4.0**, three components. `sketch` 0.3.0, `sketch-constitution` 0.1.0, `sketch-plan` 0.1.0 |
| `sketch-catalog` | `sketch-plan`, `skt-deepagent` 0.1.0 and the `sketch-langchain-deep-agent` bundle 0.1.0 already registered |
| `sketch-langchain-deep-agent` | bundle **0.1.0**, `skt-deepagent` 0.1.0, which ships its own bash and PowerShell composer |

If any of those differ in your target branch, stop and report before editing.

## What this change does

| | Before | After |
|---|--------|-------|
| Composer | 612 lines of bash and PowerShell inside `skt-deepagent` | One Python library in `skt-compose`, shipped from `sketch-base` |
| Work-type config | Two hand-written scripts kept behaviourally identical by hand | A 19 line `compose.yml` |
| Two work types installed together | Two hooks write the same file, last hook wins, silently | Every declaration applies in declared order, they compose |

New versions:

| Component | From | To |
|-----------|------|-----|
| `sketch-base` bundle | 0.4.0 | **0.5.0** |
| `skt-compose` | none | **0.1.0** |
| `skt-deepagent` | 0.1.0 | **0.2.0** |
| `sketch-langchain-deep-agent` bundle | 0.1.0 | **0.2.0** |
| `sketch` extension | 0.3.0 | **0.3.0, unchanged** |

Tags: `spec-kit-sketch-v0.5.0` and `spec-kit-sketch-deepagent-v0.2.0`.

---

## Layout

| Name | What it is |
|------|------------|
| **SOURCE** | The folder supplied alongside this file |
| **TARGET** | The working branch of each real repository |

SOURCE is the source of truth for every file listed below.

---

## Hard rules

1. **Do not touch `sketch-base/extensions/sketch/`.** Not one byte, not the
   version, not a comment. See Why 1.
2. **Do not touch `sketch-base/presets/`.** Both presets are unchanged.
3. **Do not copy `sketch-catalog/catalog.yml`.** It is generated. Run
   `tools/build_catalog.py`.
4. **Do not change any version** beyond the five named above.
5. **Do not use em-dash or en-dash characters** in anything you write or edit. The
   `house-style` CI job fails on both.
6. **Do not reformat, re-wrap, lint or tidy existing files.** Apply only the hunks
   described.
7. **Do not resolve an ambiguity by guessing.** Stop and report.
8. **Do not restore the deleted scripts** if a test fails. See Why 3.

---

## Task 1: sketch-base, add the `skt-compose` extension

New directory, copy whole:

```
SOURCE/sketch-base/extensions/skt-compose/  ->  TARGET/extensions/skt-compose/
```

Nine files:

```
extension.yml
compose.schema.yml
README.md
LICENSE
commands/compose.md
scripts/lib/sketch_compose.py          the library, 461 lines
scripts/lib/selftest.py                54 assertions
scripts/bash/sketch-compose.sh         thin wrapper
scripts/powershell/sketch-compose.ps1  thin wrapper
```

Do not copy any `__pycache__` directory if one is present in SOURCE.

Then:

```bash
chmod +x extensions/skt-compose/scripts/bash/sketch-compose.sh
```

**Acceptance**, no output:

```bash
diff -rq SOURCE/sketch-base/extensions/skt-compose TARGET/extensions/skt-compose
```

---

## Task 2: sketch-base, `bundle.yml`

15 changed lines. Two changes:

1. `bundle.version`: `0.4.0` becomes `0.5.0`
2. Add a fourth component after `sketch-plan`

```yaml
  - type: "extension"
    id: "skt-compose"
    version: "0.1.0"
    source: "extensions/skt-compose"
```

Copy the explanatory comment from SOURCE alongside it.

The `sketch` pin stays `0.3.0`.

Optionally update the bundle `description` to mention the composer, as SOURCE does.

---

## Task 3: sketch-base, `.gitlab-ci.yml`

28 changed lines, three hunks. Take them from
`SOURCE/sketch-base/.gitlab-ci.yml`.

| # | Job | Change |
|---|-----|--------|
| 1 | `build-assets` | After the existing `extension.zip` line, add a loop packaging every other directory under `extensions/`, **skipping `sketch`** |
| 2 | `create-release` | Add one `--assets-link` for `skt-compose.zip`, before the `preset-constitution.zip` link |
| 3 | new job | Add `compose-selftest` in the `test` stage, next to the existing `selftest` |

The preset packaging loop and the `upload-assets` loop are **already present** in
your 0.4.0 pipeline. Do not add them again.

Hunk 1 skips `sketch` on purpose so its asset keeps the name `extension.zip` and
existing install URLs keep resolving.

**Acceptance**: the file parses as YAML and the job list is exactly

```
manifests, tag-matches-manifest, selftest, compose-selftest, house-style,
build-assets, upload-assets, create-release, verify-release
```

---

## Task 4: sketch-base, three documentation files

Small. Take the exact text from SOURCE.

| File | Lines | Change |
|------|-------|--------|
| `README.md` | 1 | One row in the asset table: `skt-compose.zip` to `.specify/extensions/skt-compose/` |
| `docs/RELEASING.md` | 2 | One version-checklist line for the new manifest, one asset table row |
| `docs/REPO-MODEL.md` | 10 | `skt-compose/` in the tree, one sentence now naming three asset kinds |

---

## Task 5: deep agent, delete the shell

Delete the whole directory. Three files, 612 lines.

```
TARGET/extensions/skt-deepagent/scripts/
├── bash/deep-agent-context.sh
├── bash/selftest.sh
└── powershell/deep-agent-context.ps1
```

This is intentional and the CI enforces it: the `manifests` job fails if
`extensions/skt-deepagent/scripts` exists. See Why 3.

---

## Task 6: deep agent, add `compose.yml`

New file, copy whole:

```
SOURCE/sketch-langchain-deep-agent/extensions/skt-deepagent/compose.yml
```

This replaces everything Task 5 deleted. Nineteen non-comment lines:

```yaml
schema_version: "1.0"

compose:
  id: skt-deepagent
  order: 50

constitution:
  layers:
    - constitution.d/10-deep-agent.md
  insert_before: '^##\s+Governance\s*$'

templates:
  - target: .specify/templates/plan-template.md
    slots:
      governance: [plan.d/10-governance.md]
      evaluation: [plan.d/20-evaluation.md]
      technical-context: [plan.d/30-bounds.md]
      structure: [plan.d/40-structure.md]
```

**Do not change `insert_before`.** See Why 5.

The `constitution.d/` and `plan.d/` markdown files are **unchanged**. Do not
re-copy or edit them.

---

## Task 7: deep agent, four edited files

Take each from SOURCE.

| File | Lines | Change |
|------|-------|--------|
| `extensions/skt-deepagent/extension.yml` | 29 | Version 0.1.0 to 0.2.0. `requires` gains `skt-compose: ">=0.1.0"`, drops the `python3`/`git` tool block. Description and the `sketch:` metadata block updated |
| `extensions/skt-deepagent/commands/deep-agent-context.md` | 71 | Now runs `.specify/extensions/skt-compose/scripts/...` rather than its own script, and reports contributing extensions |
| `bundle.yml` | 11 | Bundle 0.1.0 to 0.2.0. Add a `skt-compose` 0.1.0 component. `skt-deepagent` pin to 0.2.0 |
| `.gitlab-ci.yml` | 86 | The `manifests` job now validates `compose.yml` instead of the shell. The `selftest` job is replaced by `compose-integration` |
| `CHANGELOG.md` and `extensions/skt-deepagent/CHANGELOG.md` | 12 each | New 0.2.0 entry. Both files are identical |
| `docs/VERIFY.md` | 28 | Item 4 rewritten: the extension no longer ships a PowerShell composer of its own |

The new `manifests` checks are worth knowing, because they encode the design:

- every path in `compose.yml` exists
- every shipped fragment under `plan.d/` and `constitution.d/` is declared
- no orphan markdown
- `extensions/skt-deepagent/scripts` does not exist
- `extension.yml` requires `skt-compose`

The `compose-integration` job needs a `SKT_COMPOSE_URL` CI variable pointing at the
released `skt-compose.zip`. It skips cleanly when unset, so it does not block the
first pipeline.

---

## Task 8: sketch-catalog, `catalog.d/platform.yml`

60 changed lines, four edits. Take each block verbatim from SOURCE.

| # | Entry | Change |
|---|-------|--------|
| 1 | `extensions:` | **Add `skt-compose`.** Layer `LX`, `category: process`, `effect: read-write`, `repo: platform/sketch-base`, tag and URL at `spec-kit-sketch-v0.5.0` |
| 2 | `extensions: skt-deepagent` | Version 0.1.0 to 0.2.0. `requires` gains `skt-compose: ">=0.1.0"`. Tag and URL to `spec-kit-sketch-deepagent-v0.2.0`. Description notes it is declaration only |
| 3 | `bundles: sketch-langchain-deep-agent` | Version 0.1.0 to 0.2.0. Add `skt-compose` 0.1.0 to `components`, bump `skt-deepagent` to 0.2.0. Tag and URL to v0.2.0 |
| 4 | `bundles: sketch-base` | Version 0.4.0 to 0.5.0. Add `skt-compose` 0.1.0 to `components`. Tag and URL to `spec-kit-sketch-v0.5.0` |

Three traps:

- **`skt-compose` must be `layer: LX`, not `L2`.** The schema invariant
  `reserved-prefix` requires an `skt-` id to be L3 or LX, and
  `validate_catalog.py` rejects `L2` outright. LX is right on the merits too:
  "cross-cutting utility, joins any bundle at any layer."
- **Do not change the `extensions: sketch` entry.** It stays `version: "0.3.0"`,
  `commands: 3`. See Why 1.
- **Add `skt-compose` to each `components` list exactly once.** A naive
  find-and-replace on the `sketch-plan` component line matches both bundles and
  produces a duplicate. `validate_catalog.py` catches it:
  `component 'skt-compose' listed twice`.

The `sketch-constitution` and `sketch-plan` preset entries are **unchanged**. They
did not change, so they keep pointing at the v0.4.0 release, where those versions
were published. Old tags keep their assets.

Then, in order:

```bash
python3 tools/validate_catalog.py      # must report 0 errors, 0 warnings
python3 tools/build_catalog.py         # regenerates catalog.yml
```

If validation reports errors, do not commit. Paste the output.

---

## Merge request order

The catalog goes last: doing it first fails validation because the pins would name
releases that do not exist yet.

1. `sketch-base`, tag `spec-kit-sketch-v0.5.0`. Confirm four assets on the Release:
   `extension.zip`, `skt-compose.zip`, `preset-constitution.zip`, `preset-plan.zip`
2. `sketch-langchain-deep-agent`, tag `spec-kit-sketch-deepagent-v0.2.0`
3. `sketch-catalog`

---

## Why this change was made

The whole point is **turning the composer into a library**. Everything else follows.

### Why 0: the composer had to exist at all

Spec Kit resolves one file per name and never merges. It walks the priority stack,
takes the first match, and stops.

So a work type cannot layer constitution principles or extend the plan template
through a second preset: it would replace the first rather than layer onto it. And
copying the bank-wide constitution into each work type means the same control
exists in triplicate and drifts within a year.

The composition happens in a `before_plan` hook instead, which the plan command
waits for before `setup-plan` reads the constitution or copies the template.
Nothing in Spec Kit is modified.

### Why the library, three reasons

**1. Two work types could not coexist.** This is the real defect, not the
duplication. With one composer per work type, installing two means two hooks
writing the same `plan-template.md`, and last hook wins, silently. Nobody sees a
warning; one work type's sections simply are not there. Hook ordering is not
something you control.

The library applies every installed `compose.yml` in declared `order`, each wrapped
in its own marker block, so they compose. Verified: a second synthetic work type
installed alongside the deep agent one produces both principle sets in order, two
marker blocks, and both fragments inside the one slot they share.

**2. Two scripts had to be kept behaviourally identical by hand.** 612 lines of
bash and PowerShell implementing the same logic twice. Any fix had to land in both
or they diverged silently. The library is one Python file; the two entry points are
short wrappers around it and cannot drift.

**3. A new work type meant writing shell.** Now it means writing 19 lines of YAML.
The composer is work-type agnostic: nothing in it mentions LangGraph or Deep
Agents.

### Why 1: the `sketch` extension stays at 0.3.0

The obvious placement puts the composer inside the existing `sketch` extension,
where the scripts already live. That was tried and rejected, by the validator
rather than by opinion.

Folding it in bumps `sketch` to 0.4.0, and **24 existing bundles across 16 catalog
fragments pin `sketch` 0.3.0 exactly**. `validate_catalog.py` reports one error per
bundle:

```
ERROR gwm.yml:   bundle 'sketch-advisor-reporting': pins 'sketch' at 0.3.0, but the catalog has 0.4.0
ERROR risk.yml:  bundle 'sketch-risk-reporting': ...
ERROR ib.yml:    bundle 'sketch-ib-pitch', 'sketch-counterparty', 'sketch-loan-agreement': ...
... 19 more, across am, caio, ceda, finance, gcore, gcto, gia, gic, hr,
    legal, platform, wma, wmpc
```

Those fragments are owned by division leads, not the platform team, so it means a
coordinated change across all of them for something none of those teams asked for.

A separate extension in the same repository costs one component and one catalog
entry and breaks nothing.

**If you find yourself wanting to bump `sketch`, stop.** That is the failure this
placement exists to avoid.

### Why 2: `skt-compose` is not its own repository

An earlier draft put it in a new `platform/sketch-shared` repository. That was
wrong: it adds a repository, a pipeline, a release-train concept and a catalog pin
to hold one file, and it makes users reason about two things instead of one.

It ships from `sketch-base`, which every bundle already pins, so it is not a second
install for anyone. One `specify bundle install` still installs everything.

### Why 3: `skt-deepagent` ships no shell, and the CI enforces it

Its `manifests` job fails if `extensions/skt-deepagent/scripts` exists. That is
deliberate: a work type that grows its own composer reintroduces both problems
above.

**If a test fails, do not restore the deleted scripts.** Report the failure.

### Why 4: the `.base` snapshots

The hook fires on every plan run, so appending would compound: the agent principles
would appear twice after two runs. Instead the composer captures a pristine
snapshot on first run (`constitution.base.md`, `plan-template.base.md`) and
rebuilds from it every time. Four consecutive runs give a byte-identical file. If a
snapshot is deleted, the composer recovers it by stripping its own marker block
rather than baking a merged file in as the new base.

### Why 5: `insert_before` in `compose.yml`

The regex `'^##\s+Governance\s*$'` puts the work-type layer above the bank-wide
constitution's trailing `## Governance` section. Without it, principle numbering
reads I to VI, Governance, VII to XVI. With it, I to XVI contiguous and the
amendment procedure at the end. It looks like a fragile detail; it is asserted by
the selftest.

---

## If something does not match

TARGET may have drifted. Do not improvise.

| Situation | What to do |
|-----------|------------|
| A version in TARGET differs from the Baseline table | Stop. Report both values. Do not pick one |
| A file listed for copy already exists | Stop that task. Report the path and diff the two |
| A hunk's anchor text is not found | Stop that task. Report the file and the text searched for |
| The preset packaging loop is missing from `.gitlab-ci.yml` | Your baseline is older than 0.4.0. Stop and report |
| `validate_catalog.py` reports errors | Do not commit. Paste the full output |
| A selftest fails | Do not modify the test, and do not restore deleted scripts. Paste the failing assertion |

A partial merge that is reported is recoverable. A merge that guessed is not.

---

## Verify, then report

Run all four and include the output verbatim:

```bash
# sketch-base
python3 extensions/skt-compose/scripts/lib/selftest.py    # expect: passed 54, failed 0
bash extensions/sketch/scripts/bash/selftest.sh           # expect: 87 passed, 0 failed

# sketch-catalog
python3 tools/validate_catalog.py                         # expect: 0 error(s), 0 warning(s)

# the untouched-extension guarantee
diff -rq SOURCE/sketch-base/extensions/sketch TARGET/extensions/sketch
                                                          # expect: no output
```

Confirm each of these explicitly:

- `extensions/skt-deepagent/scripts/` does not exist
- `extensions/skt-deepagent/compose.yml` exists
- `skt-compose` is `layer: LX` in the catalog
- The catalog `extensions: sketch` entry still reads `version: "0.3.0"`, `commands: 3`
- `skt-compose` appears exactly once in each bundle's `components` list
- `catalog.yml` was regenerated by the tool, not hand-copied
- No em-dash or en-dash was introduced

Then report per repository: files added, deleted, and edited with hunk counts;
anything you stopped on; and the verification output.

---

## One thing nobody has verified

Not your task to fix, and do not "correct" it.

The composer writes to `.specify/templates/plan-template.md`, which is where a
preset installs it. Whether `setup-plan` reads that exact path, rather than
resolving through a separate override directory, has not been confirmed against a
running Spec Kit.

If wrong, the hook runs, reports slots filled, and the plan uses a different file.
That is the only failure mode here that looks like success in the logs. One plan run
settles it: grep the generated `specs/<feature>/plan.md` for
"Harness configuration".

Details, plus two other inferred items, in
`sketch-langchain-deep-agent/docs/VERIFY.md`.
