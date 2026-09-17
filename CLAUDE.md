# acx_product_management

Specification repo. A stakeholder ask enters; unit specs leave, and engineering repos
generate code from them. **This repo holds no application code.**

## First run in a brand-new repo (bootstrap)

**This repo does not vendor `.claude/skills`, `.claude/scripts`, `.claude/templates`,
`.claude/rules`, or `.claude/hooks` into git.** `acxworks-artifacts` is the single
source of truth for those five directories; this repo only ever holds a local,
gitignored working copy, fetched on demand. **On a truly fresh clone only this file
is present** — no `.gitignore`, no fetch script, nothing else — so the commands below
are self-contained and create everything they need.

**STRICT RULE: if `.claude/skills/` is missing on disk, run the bootstrap block below
immediately, automatically, without asking the user for confirmation first.** This
applies at the start of any session/task in this repo where the directory is absent —
treat it as a required setup step, not an optional offer. Tell the user afterward that
the bootstrap ran and what was fetched; do not ask beforehand. Run in bash/Git Bash,
not PowerShell.

```sh
git remote get-url acx-artifacts >/dev/null 2>&1 || \
  git remote add acx-artifacts https://github.com/acxworks/acxworks-artifacts.git

for pair in skills:claude-skills scripts:claude-scripts templates:claude-templates \
            rules:claude-rules hooks:claude-hooks; do
  name=${pair%%:*}
  branch="pm-export/${pair#*:}"
  git fetch -q acx-artifacts "$branch"
  rm -rf ".claude/$name"
  mkdir -p ".claude/$name"
  git archive FETCH_HEAD | tar -x -C ".claude/$name"
done

for dir in .claude/hooks .claude/rules .claude/scripts .claude/skills .claude/templates; do
  grep -qxF "$dir" .gitignore 2>/dev/null || echo "$dir" >> .gitignore
done
```

Re-run this block any time to refresh to the latest upstream — it always overwrites
each directory's contents outright, so never hand-edit files under these five
directories (the read-only guard hook also blocks this); changes belong in
`acxworks-artifacts`. No merge, no local git history, no duplicate commits in this
repo — just a plain working copy refreshed from `git archive`.

Full picture of the artifact-export side (branches, what publishes them, the
`subtree-export.js` tooling used _there_): `GIT-SUBTREE.md` at the
`acxworks-artifacts` repo root, and `product_management/.claude/templates/downstream/
README.md` there. Those docs describe `git subtree`-based consumption as one option;
this repo has deliberately opted for the plain-fetch/gitignore approach above instead,
since no downstream changes ever need to flow back from here into `acxworks-artifacts`.

## The pipeline

`project → capability → unit`. One skill per step, prefixed by the role that owns it.

| Step | Skill                       | Produces                                                                                                                                                    |
| ---- | --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1    | `pm-project-init`           | the project folder's `project.md` — `projects/<slug>/` under nested layout, the repo root under flat layout; see `shared-spec-conventions` § Project folder |
| 2    | `ba-requirements-intake`    | `intake/<date>-<slug>.md` + a READY verdict                                                                                                                 |
| 3    | `ba-capability-split`       | `CAP-<CODE>-NNNN-<slug>/capability.md`                                                                                                                      |
| 3a   | `architect-detailed-design` | `capability-design.md` — **before the split**                                                                                                               |
| 4    | `ba-unit-split`             | `UNIT-<CODE>-NNNN-<slug>/` scaffolded                                                                                                                       |
| 5    | `ba-unit-requirements`      | `requirements.md` → `framed`                                                                                                                                |
| 6    | `architect-unit-design`     | `design.md`                                                                                                                                                 |
| 7    | `architect-unit-interfaces` | `interfaces/*` → `designed`                                                                                                                                 |
| 7a   | `designer-unit-ux`          | `ux/*` — frontend units only                                                                                                                                |
| 7b   | `designer-ux-handoff`       | early `Ui-Development` (blocked) + `QA` test-case-creation tickets, both citing `ux/test-attributes.md` — frontend units only                               |
| 8    | `architect-unit-tasks`      | `tasks.md` → `ready`                                                                                                                                        |
| 9    | `ba-spec-validate`          | verdict + traceability matrix                                                                                                                               |
| 10   | `ba-unit-handoff`           | branch + PR staged in the project's staging repo — `handed-off` follows separately, once `pm-status-transition` confirms the PR is merged                   |

Also: `architect-adr-record`, `ba-change-request`, `ba-readiness-sweep`,
`pm-spec-review`, `pm-status-transition`, `pm-state-rollup`, `pm-capability-sweep`,
`architect-code-review` — the architect's review of a unit's engineering-repo code PR
against its spec, run after `handed-off` and cited as the conformance evidence for
`verified`.

**Capability design precedes the unit split.** `architect-detailed-design` owns the
shared schema, the unified contract family and the cross-unit flows, and it runs at
step 3a — _before_ `ba-unit-split`, not after. The design is what reveals where the
seams should fall; a seam corrected before unit IDs are minted costs nothing, and one
corrected afterwards costs a change request per unit. `capability.md` and
`capability-design.md` are reviewed and merged together as one PR, so the split's
existing gate covers both. Skip it where a capability projects exactly one unit.

## Trackers

One `tracker.md` at each level — the project folder, each `CAP-*/`, each `UNIT-*/`.
It answers _which pipeline steps have been done, and what is the next action_, as an
Overview table plus one section per step with derived checkboxes.

**Fully generated by `pm-state-rollup`, whole file.** Every box is a condition
evaluated against the tree, not a human tick — nobody can tick one by wishing. Six
values: `⬜ Not Started · 🟡 In Progress · ✅ Done · ⛔ Blocked · — n/a · ? Unknown`.

At project and capability level it is the **only** generated view — it carries the
status counts and the unit index too. There is no project or capability `state.md`.

Three things look adjacent and are not:

|                    | Answers                            | What it is                          |
| ------------------ | ---------------------------------- | ----------------------------------- |
| unit `state.md`    | who moved this unit, when, why     | the **source** — append-only ledger |
| `tracker.md`       | which steps are done, what is next | a **view** of that ledger           |
| `ba-spec-validate` | is what was done consistent        | a **verdict**, on demand            |

The ledger is not a duplicate of the tracker — it is the input the tracker is computed
from, and the only file in the tree that is immutable after write.

**A `✅` is not a PASS.** No gate reads a tracker in place of running
`ba-spec-validate`. Definition lives in `shared-spec-conventions`; shape lives in
`.claude/templates/*/tracker.md`.

**The child never writes the parent's files.** A unit-level rollup — triggered by
`pm-status-transition`, `designer-unit-ux`, `architect-adr-record`, or any future
unit-authoring skill — only ever regenerates that one unit's own `tracker.md`. It
never cascades up to touch its capability's `tracker.md`/`capability.md`, or the
project's `tracker.md`/`project.md`, even though the data to do so is right there.
That upward cascade is `pm-capability-sweep`'s job alone, run as a deliberate batch
over every unit in one capability — never automatically after a single unit's
transition. This exists because two unit-level flows running in parallel used to
each trigger a full bottom-up rollup and clobber each other's writes to the ~20
shared capability/project files; scoping unit rollups to one file removes that race.
It is expected, not a bug, for a capability or project tracker to read a step behind
until the next `pm-capability-sweep` runs.

## Hard rules

1. **One skill owns each file.** If the skill you are running does not own the file
   you are about to write, you are in the wrong skill.
2. **Unit `state.md` is append-only, and only units have one.** Only
   `pm-status-transition` writes it. A unit's current status _is_ its last row. Never
   rewrite, reorder, or delete a row — corrections are appended as same-status rows.
   It is the source every tracker is computed from, so it is never replaced by one.
3. **Generated content is never hand-edited** — anything between
   `<!-- GENERATED:… -->` markers, and all three `tracker.md` files. Only
   `pm-state-rollup` writes those.
4. **`tasks.md` is never edited after `ready`.** Changes arrive as
   `tasks_<YYYY-MM-DD>.md` from `ba-change-request`.
5. **After a unit is handed off, `requirements.md` is appended to, never edited**, and
   `capability.md`'s original ask is frozen once it leaves `draft`.
6. **A spec copied into an engineering repo is read-only there.** Changes come back
   here as a change request.

## Status

```
draft → framed → designed → ready → handed-off → building → verified → accepted
verified|accepted → amending → ready
any → blocked → (the status it was blocked from)      any → withdrawn
X → X   same-status correcting row, always legal
```

`accepted` is never derived. `pm-status-transition` sets it, and only against dated
evidence: every outcome measure with an actual figure from a named source, every
acceptance condition evidenced, and the PM's acceptance in writing.

**Capability `status: accepted` currently has no writing skill** — as do `in-progress`
and `verified`. `pm-state-rollup` derives all three and is forbidden from writing them.
Known gap; set by hand until it is assigned.

## Engineering surfaces — the second axis

The status above is spec-side, and it is one flag: it cannot say "the screen is done
but the service is not". Each unit therefore declares which engineering surfaces it
has, in `requirements.md` frontmatter, set by `ba-unit-split` from `kind`:

```yaml
engineering:
  frontend: { applicable: false }
  api: { applicable: true }
```

Two surfaces only. **Infra is tracked at service/repo level, not per unit.**
`applicable: false` drops the surface out of every rollup and renders `— n/a`, never
as a completed state.

Per-surface **status** is not a field. It is a same-status ledger row with an exact
prefix, appended by `pm-status-transition UNIT-XXX-NNNN api verified` — the same
mechanism as `readiness-sweep:`, and for the same reason: post-handoff facts belong
in the append-only ledger, not in a file that is appended to rather than edited.

```
surface:api in-progress → verified — contract tests green, PR #431
```

`not-started | in-progress | verified | blocked`, and deliberately nothing finer —
sub-steps belong to the engineering repo's tooling. **Engineering completion is
derived**, never stored: complete only when ≥1 surface is applicable and every
applicable surface is `verified`. `pm-state-rollup` writes it to the trackers.

The two axes are independent. A surface never moves the lifecycle status and the
lifecycle status never moves a surface; `ba-spec-validate` T9 catches a row that
tried. Definitions live in `shared-spec-conventions` § Engineering surfaces.

## Where things are

| What                                                                                                              | Where                                                                                          |
| ----------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| IDs, statuses, frontmatter, who-writes-what, and the nested-vs-flat project folder rule                           | `.claude/skills/shared-spec-conventions`                                                       |
| platform, compliance and format standards                                                                         | `.claude/rules/` — loaded automatically                                                        |
| blank artifacts to copy                                                                                           | `.claude/templates/`                                                                           |
| a complete worked example                                                                                         | `projects/subscription-billing` — `UNIT-SUB-0001` is fully specified                           |
| the command list                                                                                                  | `DEMO-PROMPTS.md`                                                                              |
| publishing skills/scripts/templates/rules/hooks to downstream repos, one artifact type at a time, via git subtree | `.claude/scripts/subtree-export.js` — full walkthrough in the monorepo root's `GIT-SUBTREE.md` |

## Asking for input

Skills ask for the facts they cannot derive. Answer briefly; they proceed on stated
assumptions rather than stalling. Never invent a fact to fill a gap — see
`.claude/rules/00-core.md`.
