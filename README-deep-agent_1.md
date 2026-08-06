# Sketch LangChain Deep Agent

**Layer L3** · work type · v0.1.0 · `status: alpha`

Work type for LangChain Deep Agent applications on the skill-based harness. It
does two things stock Spec Kit cannot: it layers a constitution, and it extends
the plan phase without forking anything.

```bash
specify bundle install sketch-langchain-deep-agent
```

- [The problem this solves](#the-problem-this-solves)
- [How it works](#how-it-works)
- [What ends up in your plan](#what-ends-up-in-your-plan)
- [The constitution layer](#the-constitution-layer)
- [Files and where they go](#files-and-where-they-go)
- [Adding another work type](#adding-another-work-type)
- [Verified and unverified](#verified-and-unverified)
- [Troubleshooting](#troubleshooting)

---

## The problem this solves

Spec Kit resolves **one file per name** and never merges. It walks the priority
stack, takes the first match, and stops. Two presets both providing
`memory/constitution.md` do not layer: one wins and the other is invisible.

That is fine for most things and fatal for two:

**A constitution cannot be layered by presets.** The bank-wide principles and a
work type's principles are both constitutions. Shipping them as two presets means
picking one. Copying the base into each work type means the same control exists in
triplicate and drifts within a year.

**A plan template cannot be extended by presets either.** Same reason. And
unlike prose, a plan template has an expected section order, so even
concatenation would not work: you would get two Technical Contexts and a
Structure Decision in the middle of the file.

So the composition happens in a `before_plan` hook instead.

## How it works

The ordering is the whole trick, and it comes from the plan command's own
instructions. Its Pre-Execution Checks say: run the `before_plan` hooks and
**wait for the result before proceeding to the Outline**. Outline step 1 then
runs `setup-plan`, which is what copies `plan-template.md` into the feature
folder and resolves the constitution path.

So a hook that writes those two files finishes before anything reads them.

```
/speckit.plan
  │
  ├─ 1. before_plan hooks ─── speckit.skt-deepagent.context
  │                             │
  │                             ├─ constitution.base.md  +  constitution.d/10-deep-agent.md
  │                             │       └──────────────► .specify/memory/constitution.md
  │                             │
  │                             └─ plan-template.base.md +  plan.d/*.md into SKETCH-SLOT markers
  │                                     └──────────────► .specify/templates/plan-template.md
  │
  ├─ 2. setup-plan  ───────── copies plan-template.md into specs/<feature>/plan.md
  │                            and resolves the constitution path
  │
  ├─ 3. reads spec.md and the merged constitution
  ├─ 4. Constitution Check gate, against all 16 principles
  ├─ 5. Phase 0 research, Phase 1 design
  │
  └─ after_plan hooks
```

Nothing in Spec Kit is modified. The plan command stays stock, `setup-plan` stays
stock, and upgrades cost nothing.

### Why it is idempotent

The hook runs on **every** plan run, so naive appending would compound. Both
outputs are rebuilt from pristine snapshots instead:

| Generated, do not edit | Pristine snapshot, captured on first run |
|------------------------|------------------------------------------|
| `.specify/memory/constitution.md` | `.specify/memory/constitution.base.md` |
| `.specify/templates/plan-template.md` | `.specify/templates/plan-template.base.md` |

Every run reads the `.base.md` file, composes, and overwrites the output. Four
consecutive runs produce a byte-identical file. If a snapshot is deleted, the
composer recovers it by stripping its own marker block rather than baking a
merged file in as the new base.

Both facts are asserted by the selftest.

```bash
bash extensions/skt-deepagent/scripts/bash/selftest.sh
# passed 39, failed 0
```

## What ends up in your plan

The `sketch-plan` preset supplies the plan template with three sections the stock
one lacks, plus six inert `SKETCH-SLOT` markers. This bundle fills four of them.

| Slot | Fragment | What appears in the plan |
|------|----------|--------------------------|
| `governance` | `plan.d/10-governance.md` | Harness configuration table, one row per `create_deep_agent()` argument. Skills inventory with resolution order. Interrupt policy per tool |
| `evaluation` | `plan.d/20-evaluation.md` | Task cases, adversarial cases per interrupted tool, injection cases, trajectory scoring, replay basis |
| `technical-context` | `plan.d/30-bounds.md` | Bounds table with the config location for each, and the subagent table |
| `structure` | `plan.d/40-structure.md` | A starting layout, and the decision folder names do not show |

Two slots are left unfilled on purpose, `constitution-check` and `coverage`, as
headroom for a later overlay. The selftest asserts an unmapped slot survives
untouched.

The point of the harness configuration table is that it is **one row per
argument**. The constitution states which values are permitted; the plan states
which were chosen. A blank row is a configuration decision nobody made, which for
an agent is the same as a capability nobody reviewed.

## The constitution layer

Principles I to VI come from `sketch-constitution`. This layer adds VII to XVI,
each naming the configuration argument or middleware it governs so conformance is
a reading of arguments rather than a review of control flow.

| | Principle | Governs |
|---|---|---|
| VII | Approved harness | `create_deep_agent()`, no bespoke loop |
| VIII | Confined backend | `backend`, `LocalShellBackend` prohibited |
| IX | Interrupt before consequence | `interrupt_on` |
| X | Durable state | `checkpointer`, `store` namespace |
| XI | Skills resolve to versions | `skills` sources and their order |
| XII | Bounded execution | recursion limit, timeout, cost, subagent depth |
| XIII | Full trace emission | collector, no sampling |
| XIV | Retrieved content is data | skill bodies, files, tool output |
| XV | Subagents inherit constraints | `subagents` |
| XVI | Task-level evidence | promotion gate |

**VIII, IX, XI and XIV are non-negotiable** and cannot be satisfied by a
Complexity Tracking entry.

Three of these exist because of specifics of the skill-based harness rather than
of agents in general. See [`docs/DEEP-AGENT-NOTES.md`](docs/DEEP-AGENT-NOTES.md)
for the source material:

- **XI on source order.** Skills load in source order with later sources
  overriding earlier ones, so order determines which instruction set actually
  runs. Two deployments with the same skill list in a different order are two
  different agents.
- **XI on inventory size.** Progressive disclosure defers skill *bodies*, not
  skill *metadata*. Every skill's description sits in the system prompt on every
  run, so the inventory is a bounded resource rather than a folder anyone may add
  to.
- **XIV on the skill path.** An agent with tools, a filesystem and progressive
  skill loading has an injection route to every tool, and the skill body is the
  least obvious of them.

The layer is inserted **before** the base's trailing `## Governance` section, so
principle numbering stays contiguous I to XVI and the amendment procedure stays
at the end. Asserted by the selftest.

## Files and where they go

```
sketch-langchain-deep-agent/
├── bundle.yml                          pins sketch, sketch-constitution,
│                                       sketch-plan, and skt-deepagent
├── README.md                           this file
├── CHANGELOG.md
├── .gitlab-ci.yml
├── docs/
│   ├── DEEP-AGENT-NOTES.md             harness research, with sources
│   └── VERIFY.md                       what to confirm before releasing
└── extensions/skt-deepagent/
    ├── extension.yml                   hooks.before_plan
    ├── commands/deep-agent-context.md   the hook command
    ├── constitution.d/
    │   └── 10-deep-agent.md            principles VII to XVI
    ├── plan.d/
    │   ├── 10-governance.md            harness, skills, interrupts
    │   ├── 20-evaluation.md            task-level evaluation
    │   ├── 30-bounds.md                bounds and subagents
    │   └── 40-structure.md             starting layout
    └── scripts/
        ├── bash/deep-agent-context.sh   the composer
        ├── bash/selftest.sh             39 assertions
        └── powershell/deep-agent-context.ps1
```

After installing, in a project:

```
.specify/
├── memory/
│   ├── constitution.md           GENERATED, 16 principles
│   └── constitution.base.md      pristine, principles I to VI
├── templates/
│   ├── plan-template.md          GENERATED, slots filled
│   └── plan-template.base.md     pristine, slots empty
└── extensions/skt-deepagent/     the layer files and the composer
```

**Gitignore the two generated files.** They are rebuilt on every plan run, and
committing them means a merge conflict every time anyone plans.

```bash
echo '.specify/memory/constitution.md'        >> .gitignore
echo '.specify/templates/plan-template.md'    >> .gitignore
```

**Do not use `/speckit.constitution` on a project using this bundle.** It writes
to `constitution.md`, which is generated output. Edits there are lost on the next
plan run. Edit `constitution.d/10-deep-agent.md`, or amend the base in
`sketch-base`.

## Adding another work type

Copy this bundle and change three things:

1. `constitution.d/10-<worktype>.md`, the principles
2. `plan.d/*.md`, the fragments and which slots they fill (the mapping is a
   dictionary near the end of the composer, in both variants)
3. The ids in `extension.yml` and `bundle.yml`

The composer itself is work-type agnostic. Nothing in it mentions LangGraph.

**Install one work type per project.** Two bundles both registering a
`before_plan` hook that writes `plan-template.md` means last hook wins, and hook
ordering is not something you control. Compose inside one hook rather than having
two hooks fight.

Extension ids are capped at 20 characters and Layer 3 ids take the `skt-` prefix,
per the catalog schema. Bundle ids have no length cap, which is why
`sketch-langchain-deep-agent` is legal at 27 characters, though
`sketch-deep-agent` would be kinder to anyone typing it.

## Verified and unverified

Verified by running it, in a throwaway project that mimics a real install:

- The full composition path: 2 constitution layers, 4 plan slots filled
- Idempotency: byte-identical output after four consecutive runs, exactly one
  marker pair
- Principle ordering: I to XVI contiguous, Governance last
- A layer edit reaching the next run
- Snapshot recovery after `constitution.base.md` is deleted
- Both degraded paths: no bank-wide constitution, and the stock plan template in
  place. Both warn and proceed rather than failing
- `--check` mode writing nothing
- No em-dash or en-dash in any shipped file, which the sketch-base pipeline
  rejects

Not verified, and listed in [`docs/VERIFY.md`](docs/VERIFY.md):

- **The `hooks.before_plan` entry shape.** The `extension.yml` field names are
  inferred from how `templates/commands/plan.md` consumes
  `.specify/extensions.yml`. Run `specify extension add --dev .` and read the
  generated `.specify/extensions.yml` to confirm.
- **The `provides.templates` type value in `sketch-plan`.** The
  `sketch-constitution` preset uses `type: memory`; `type: template` is the
  analogous value for a file under `templates/`, but it is inferred.
- **Where `setup-plan` reads the template from.** The composer writes to
  `.specify/templates/plan-template.md` because that is where a preset installs
  it. Confirm `setup-plan` reads that path rather than a separate override path.
- **The PowerShell composer.** It mirrors the bash logic and was not executed:
  no PowerShell in the build environment.

The first and third are the load-bearing ones. If the hook entry shape is wrong
the composer never runs; if `setup-plan` reads elsewhere, it runs and has no
effect. Both fail visibly on a first plan run rather than silently.

## Troubleshooting

**"slots filled: 0"** The stock Spec Kit plan template is installed rather than
the Sketch one, so this plan has no governance or evaluation sections. Check
`sketch-plan` is in the bundle and installed.

**"no bank-wide constitution"** The project is running on principles VII to XVI
alone, without I to VI. Install `sketch-base`.

**Sections appear twice** A snapshot was committed after being generated. Delete
both `.base.md` files and both generated files, then re-run the hook.

**The hook does not appear to run** Check `.specify/extensions.yml` has an entry
under `hooks.before_plan`. If the field names differ from `extension.yml`, that
is the inferred schema above, and the manifest needs correcting.

**Nothing in the plan reflects a layer edit** The layer is read on every run, so
this means the composer did not run at all rather than that it read stale input.
Run it by hand and read the output:

```bash
.specify/extensions/skt-deepagent/scripts/bash/deep-agent-context.sh
```
