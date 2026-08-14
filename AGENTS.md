# Roadmap instructions

This repository contains planning documents, not btcwallet source code.

## Sources of truth

- [README.md](README.md) explains how to use the repository.
- [QUEUE.md](QUEUE.md) owns each roadmap task's delivery phase and status by
  placing open work in a phase table and completed work with its closing PR in
  `Done`.
- [ACTIVE.md](ACTIVE.md) owns the active-task focus, claims, in-progress PRs,
  and coordination.
- Individual files under [tasks/](tasks/) own type, scope, and dependencies.
- [tasks/README.md](tasks/README.md) owns the open catalog and next number.
- [TASK_TEMPLATE.md](TASK_TEMPLATE.md) owns the required task shape.
- [archive/](archive/) is historical evidence, not current instruction.

Do not copy the complete dependency graph into README or ACTIVE. QUEUE shows
every roadmap task's delivery phase and open dependencies, but task files
remain the only authority for type and all dependency edges, while ACTIVE
remains the only authority for live state.

## Activity, claiming, and dependencies

Use these terms precisely:

- **Active** means current-focus work listed under ACTIVE's `Active Tasks`.
  Every active task must have no open hard dependencies. Include every such
  Phase 1 task and dependency-ready later-phase work assigned to developers
  other than YY. Park every claimed task with open hard dependencies and YY's
  later-phase assignments under `Claimed Tasks`.
- **Claimed** records the intended owner. A task may be claimed before its
  dependencies finish and therefore may be claimed without being active.
- **In progress** means implementation has started. Represent it with both a
  claimant and linked PR. If a series starts before its individual PRs open,
  record the exception under ACTIVE's Coordination section and link each PR
  when opened.
- **Complete** means the implementation PR is merged, or a required design or
  ADR task is accepted and merged.

When claiming a task with open hard dependencies, assign every unclaimed task
in its full transitive open dependency chain at the same time. Use the
dependent task's claimant by default; an existing or explicitly coordinated
different claimant may remain. Chain assignment does not make a prerequisite
active or in progress. Implementation may start only after every hard
dependency and gate is complete; a claim, active status, review approval, or
passing CI is not completion.

Each developer may have only one task in progress at a time unless an explicit
temporary exception is recorded under ACTIVE's Coordination section. Archive
the task after its implementation PR merges. Additional claims do not
authorize parallel implementation by the same developer.

Do not use stacked implementation PRs to bypass dependency completion. YY must
approve an exception explicitly, the prerequisite contract must be frozen, and
the dependent PR must remain unmergeable until the prerequisite merges.

Refresh ACTIVE's two task lists whenever a claim, task dependency, or task
status changes. No task appears in both `Active Tasks` and `Claimed Tasks`.

## Task types and priority

Give every task exactly one primary type based on its main motivation, not
incidental implementation details:

- **Bug fix:** existing supported behavior is incorrect or unsafe.
- **Gap fix:** behavior required by an accepted public contract, backend
  parity, or consumer integration is missing.
- **Feature:** optional additive behavior not required by an accepted
  contract.
- **Refactor:** ownership or structure changes while intended public behavior
  remains unchanged.
- **Test:** standalone integration, scenario, conformance, or test-cleanup
  work.
- **Benchmark:** measurement-only work.
- **Design:** an ADR or contract decision required before implementation.
- **Migration:** legacy conversion, backend retirement, or migration work.

Among dependency-ready work, prioritize Bug fix and Gap fix tasks. Pull
forward Design or Test work when it directly enables or proves one of those
fixes. Refactors, features, benchmarks, and migrations normally follow.

A Refactor must not become a hard dependency merely to force a Bug fix or Gap
fix to use the desired final architecture. Keep that dependency only when the
current boundary cannot implement the fix safely. Type guides which ready work
to select first; it does not replace dependency readiness or the Active and
Claimed rules above.

## Roadmap update workflow

- Use a dedicated Git worktree under `.worktrees/` and a `roadmap/<topic>`
  branch.
- Fetch and rebase onto the latest `main` before allocating task numbers.
- Allocate a contiguous range from the next number in
  [tasks/README.md](tasks/README.md).
- Serialize number-allocating updates. If another update used the range first,
  rebase and fold the renumbering into the introducing commit.
- Sign every commit and keep each roadmap commit small and self-contained.
- Validate Markdown links, task numbering, line wrapping, and
  `git diff --check`. Do not run btcwallet CI for planning-only changes.
- Fast-forward local `main` to the validated roadmap commit and push `main`
  directly. Do not open a roadmap PR unless explicitly requested.
- Remove the worktree and its local branch after the push.

## Task format

Start from [TASK_TEMPLATE.md](TASK_TEMPLATE.md). Create every task as
`NNN-kebab-case.md` using the next free number in
[tasks/README.md](tasks/README.md).

Admit one small, atomic outcome per task and implementation PR. The task must
be independently reviewable and mergeable. Split it before claiming when it
contains unrelated fixes, multiple independently useful behaviors, or more
than one independently changeable implementation boundary. Dependencies
establish ordering only; they do not import another task's scope.

Keep this section order:

1. `# Task NNN — imperative title`
2. `Component: **specific owner**.`
3. `Type: **one allowed primary type**.`
4. `Dependencies: **linked tasks or None**.`
5. `## Rationale`
6. `## Details`
7. `## Suggested fix`
8. `## Verification`
9. `## Definition of done`

Use `Rationale` for the concrete problem, `Details` for minimal scope and
explicit exclusions, `Suggested fix` for the smallest likely implementation,
`Verification` for focused evidence, and `Definition of done` for measurable
completion.

Link every dependency to its task file. Add the task to
[tasks/README.md](tasks/README.md) and add it exactly once to the delivery phase
map in [QUEUE.md](QUEUE.md).

Do not add status or claimant fields to a task. Before a task is claimed or
materially revised, bring it to this format and split it if it fails the
atomicity rule. Do not mechanically expand an unchanged task to fill sections.

## Scope rules

- Focus on the maintained public APIs under `wallet/`.
- Avoid voluntary RPC, deprecated-wallet, and unrelated daemon changes.
- Keep kvdb legacy and single-wallet behavior isolated from the SQL design.
- Do not add create manifests, generic write locks, replay state machines, or
  cross-process Manager fencing without a separately accepted task.
- An integration-test task owns the tested public contract. Fix production bugs
  revealed by its required cases in the same task and implementation PR, and
  keep those cases as regression coverage. Do not include unrelated production
  changes.
- Keep benchmark tasks measurement-only. Record a defect exposed only by a
  benchmark as its own implementation task.
- Register shared public integration-test and benchmark contracts for kvdb,
  SQLite, and PostgreSQL. Do not skip kvdb inside a shared contract. Restrict
  backend-specific code to setup and teardown; require identical public
  assertions and stable error identities. If a backend cannot represent the
  scenario, keep the task backend-specific and document the structural
  exclusion explicitly.
- Keep findings in the owning task instead of storing raw review reports.

## Completing a task

After the implementation PR merges, use the roadmap update workflow to:

1. remove its row from [ACTIVE.md](ACTIVE.md) and refresh the remaining active
   and parked claimed rows;
2. move its task file under `archive/tasks/`;
3. update links from remaining dependents to the archived task;
4. update [archive/README.md](archive/README.md) and the active
   [open task catalog](tasks/README.md); and
5. in [QUEUE.md](QUEUE.md), move that task's row from its phase table to
   `Done`, preserve its phase and record its closing PR there, refresh the
   counts, and remove its number from remaining `Open dependencies` cells.

QUEUE.md keeps completed roadmap tasks in `Done`. Preserve a task's assigned
phase during ordinary completion; an explicit roadmap-priority decision may
rephase both open and completed rows so the table continues to reflect the
current thematic map. Preserve the completed edge in each owning task file.
Rephasing is never inferred from dependency completion.

A task is complete only when **every** PR linked to it has merged. If a
linked PR is closed unmerged, or cannot be accounted for, the task is not
complete.

Preserve task numbers. Never reuse an archived or superseded number.
