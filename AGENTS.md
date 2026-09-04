# Roadmap instructions

This repository contains planning documents, not btcwallet source code.

## Sources of truth

- [README.md](README.md) explains how to use the repository.
- [QUEUE.md](QUEUE.md) owns each roadmap task's delivery phase and status by
  placing open work in a phase table, completed work with its closing PR in
  `Done`, and obsolete unimplemented work with its replacement in
  `Superseded`.
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
  other than YY. Put every Phase 1 task with open hard dependencies under
  `Blocked Phase 1 Tasks`, preserving its claimant and PR when present. Park
  claimed later-phase work under `Claimed Tasks` unless the active rule above
  places it under `Active Tasks`.
- **Claimed** records the intended owner. A task may be claimed before its
  dependencies finish and therefore may be claimed without being active.
- **In progress** means implementation has started. Represent it with both a
  claimant and linked PR. A stacked dependent remains in progress even while
  an open dependency keeps it under `Blocked Phase 1 Tasks` or `Claimed
  Tasks`. Open draft PRs early so ACTIVE records every layer of the stack.
- **Complete** means the implementation PR is merged, or a required design or
  ADR task is accepted and merged.

When claiming a task with open hard dependencies, assigning every unclaimed
task in its full transitive open dependency chain at the same time is
recommended for clear ownership, but is not required. A claimed task may have
unclaimed prerequisites. When assigning a chain, use the dependent task's
claimant by default; an existing or explicitly coordinated different claimant
may remain. Chain assignment does not make a prerequisite active or in
progress. Hard dependencies control merge order and completion, not when
implementation may start. A claim, active status, review approval, passing CI,
or an open prerequisite PR is not completion.

Each developer may have one independent task or one coherent dependency stack
in progress at a time. Give every task in a stack its own branch, PR, claimant,
and ACTIVE entry. Unrelated parallel implementation still requires an explicit
temporary exception under ACTIVE's Coordination section. Archive each task
after its own implementation PR merges.

Prefer stacked implementation PRs when they let a claimed dependency chain
advance without combining task scopes. Base each dependent branch and PR on
its immediate prerequisite, state the stack order in every PR description,
and rebase downstream layers whenever a prerequisite changes. Keep one atomic
task per PR and merge the stack bottom-up; a dependent PR must not merge before
all of its hard dependencies. Stacking needs no special approval and does not
require the prerequisite contract to be frozen.

Refresh ACTIVE's three task lists whenever a claim, task dependency, task
phase, or task status changes. `Active Tasks` and `Blocked Phase 1 Tasks` must
partition every open Phase 1 task exactly once. `Claimed Tasks` contains parked
claims outside the current phase. No task appears in more than one list.

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

Admit direct btcwallet work to Phase 1 only when its main motivation is a Bug
fix or Gap fix required for lnd integration. Its transitive open dependency
closure stays in Phase 1 and keeps each prerequisite's honest primary type. A
Design or Refactor may remain in that closure only when the current boundary
cannot safely deliver the integration fix without it; otherwise remove the
hard edge and defer that follow-up.

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
be independently reviewable and land in its own PR. A stacked dependent
becomes mergeable after its hard dependencies land. Split it before claiming
when it contains unrelated fixes, multiple independently useful behaviors, or
more than one independently changeable implementation boundary. Dependencies
establish merge ordering only; they do not import another task's scope.

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

Use `Rationale` for the concrete issue, reachable cause, and impact. Use
`Details` for observable behavior, required invariants, minimal scope, and
explicit exclusions. Use `Suggested fix` for a non-binding local direction,
not a prescribed implementation. Use `Verification` for focused behavioral
evidence and `Definition of done` for measurable completion. Except where an
exact public or storage shape is the accepted outcome, do not require private
helper or type names, synchronization primitives, query decomposition, file
layout, or an internal state machine.

Link every dependency to its task file. Add the task to
[tasks/README.md](tasks/README.md) and add it exactly once to the delivery phase
map in [QUEUE.md](QUEUE.md).

Do not add status or claimant fields to a task. Before a task is claimed or
materially revised, bring it to this format and split it if it fails the
atomicity rule. Do not mechanically expand an unchanged task to fill sections.
Do not mechanically add comment requirements to unchanged tasks. A task may
require inline code-block or detailed Arrange/Act/Assert comments when that
acceptance constraint is stated in its task file. Otherwise require comments
only where they preserve a non-obvious contract, rationale, invariant,
constraint, or consequence.

For implementation unit tests, use `mock.Mock` for interface mocks. Set
`On(...).Return(...)` expectations during Arrange with the narrowest correct
cardinality, including `Once()` for one required call. During Assert, call
`AssertExpectations(t)` directly; do not duplicate the same call assertion or
wrap the result in `require.True`.

## Scope rules

- Focus on the maintained public APIs under `wallet/`. A roadmap Bug fix must
  name the affected maintained public component, a reachable input or
  interleaving, and the wrong public result. Tests, benchmarks, designs, and
  migrations retain their type-appropriate evidence rules.
- Treat the maintained wallet library as development-stage software. Public
  and private Go API source and binary compatibility are not requirements.
  Build directly toward the final request, result, interface, and method
  shapes, and update maintained callers, implementations, mocks, and tests in
  the same task. Do not add an adapter, wrapper, parallel interface, optional
  field, or retained method solely to preserve compatibility. This freedom
  does not relax accepted behavioral contracts, safety properties, persisted-
  data compatibility, maintained backend parity, daemon-version support, or
  task scope.
- Keep normal production fixes in `package wallet` and its directly owned
  `wallet/internal` storage, cryptographic, chain, and migration
  implementations. They may consume existing external or legacy types and
  APIs unchanged; importing or calling one does not transfer ownership of its
  package into the task.
- Do not modify `waddrmgr`, `wtxmgr`, `walletdb`, `wallet/txauthor`,
  `wallet/txrules`, `wallet/txsizes`, CLI, RPC, Loader, lnd, external-daemon
  production code, or deprecated Wallet methods as a normal Wallet fix. Do not
  add or update deprecated-method tests merely because a shared dependency
  changes. Tests may drive or provision daemons only through existing
  interfaces and must not change daemon or RPC contracts. Do not use a
  mechanical compile fix to broaden a Wallet task into one of these areas.
- Task 357's accepted authoring work and Task 401's accepted chain interface
  and adapter work are narrow historical exceptions. They are not precedent
  for another task or permission to preserve deprecated Wallet behavior.
- If the required public behavior cannot be implemented inside the allowed
  production boundary, narrow, defer, or supersede the task, or obtain an
  explicit task-specific exception before implementation. Do not silently
  broaden the task into an excluded package.
- Keep kvdb legacy and single-wallet behavior isolated from the SQL design.
- Do not add create manifests, generic write locks, replay state machines, or
  cross-process Manager fencing without a separately accepted task.
- An integration-test task owns the tested public contract. Test and CI files
  may live outside `wallet`, but production bugs revealed by required cases
  must still be fixed at the maintained Wallet API or its Wallet-owned
  implementation in the same task and implementation PR. Keep those cases as
  regression coverage and do not include unrelated production changes.
- A task that exports or reshapes a public type owns the production contract,
  conversion, and focused `package wallet` tests. It does not add or revise
  integration-test source merely to prove that the type or its fields are
  accessible. The owning public itest task asserts exported values through the
  imported `wallet` package and real maintained backends. Cross-link the tasks
  to record that coverage ownership without importing itest scope into the
  production task.
- Keep benchmark tasks measurement-only. Record a defect exposed only by a
  benchmark as its own implementation task.
- Register shared public integration-test and benchmark contracts for kvdb,
  SQLite, and PostgreSQL. Do not skip kvdb inside a shared contract. Restrict
  backend-specific code to setup and teardown; require identical public
  assertions and stable error identities. If a backend cannot represent the
  scenario, keep the task backend-specific and document the structural
  exclusion explicitly.
- Keep findings in the owning task instead of storing raw review reports.

## Superseding a task

Supersession is exceptional and is not completion. A task is eligible only
when no implementation PR for its outcome has merged and an accepted roadmap
decision makes that outcome obsolete or assigns the remaining outcome to a
freshly numbered replacement task. Do not supersede work merely because it is
blocked, difficult, partially implemented, or needs to be split. Account for
and stop any open implementation PR before recording supersession.

Use the roadmap update workflow to supersede an eligible task:

1. preserve its number, title, scope, and dependencies under `archive/tasks/`,
   adding only a short note with the decision, date, and replacement;
2. remove it from [tasks/README.md](tasks/README.md), every
   [ACTIVE.md](ACTIVE.md) list, and any obsolete coordination exception;
3. move its [QUEUE.md](QUEUE.md) row from its former phase to `Superseded`,
   record the former phase, and name the fresh replacement task that owns each
   remaining outcome or the accepted decision that eliminated it;
4. update remaining dependency edges to the replacement where ordering still
   applies, while pointing historical references at the archived task; and
5. update [archive/README.md](archive/README.md), the phase and roadmap counts,
   the superseded count, and the next task number when a replacement is
   allocated.

Moving a task to `Superseded` decreases the open and former-phase counts and
increases the superseded count; it does not change the total task count.
Allocating a replacement independently increases both total and open counts.
A superseded task has no closing PR, is never listed in `Done`, and its number
is never reused.

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
