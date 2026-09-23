# Roadmap instructions

This repository contains planning documents, not btcwallet source code.

## Sources of truth

- [README.md](README.md) explains how to use the repository.
- [QUEUE.md](QUEUE.md) owns each roadmap task's delivery phase and status by
  placing open work in a phase table, completed work with its closing PR in
  `Done`, and obsolete unimplemented work with its replacement in
  `Superseded`, and approved future-cycle work in `Deferred`.
- [ACTIVE.md](ACTIVE.md) owns the active-task focus, claims, in-progress PRs,
  and coordination.
- Individual files under [tasks/](tasks/) own numbered-task type, scope, and
  dependencies. Merged ad hoc PRs own their own scope and evidence.
- [tasks/README.md](tasks/README.md) owns the open catalog and next number.
- [TASK_TEMPLATE.md](TASK_TEMPLATE.md) owns the required task shape.
- [archive/](archive/) is historical evidence, not current instruction.

Do not copy the complete dependency graph into README or ACTIVE. QUEUE shows
every numbered task's delivery phase and open dependencies, but task files
remain the only authority for type and all dependency edges, while ACTIVE
remains the only authority for live state.

## Activity, claiming, and dependencies

Use these terms precisely:

- **Active** means current-focus work listed under ACTIVE's `Active Tasks`.
  Include every open Phase 1 task and Phase 2 work selected for current
  development, including stacked work with open hard dependencies. Active does
  not mean merge-ready. Put remaining Phase 2A/2B work under
  `Upcoming Phase 2 Tasks`, claimed or unclaimed; a claim alone does not promote
  it. Park later-phase claims under `Claimed Tasks`. QUEUE's start gates still
  apply, including Hardening's gate.
- **Claimed** records the intended owner. A task may be claimed before its
  dependencies finish and therefore may be claimed without being active.
- **In progress** means implementation has started. Represent it with both a
  claimant and linked PR. Move Phase 2 work into `Active Tasks` when it starts;
  open dependencies do not remove a stacked task from current focus. Later-
  phase PRs may remain parked. Open draft PRs early to record every stack layer.
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

Refresh ACTIVE's three task lists whenever a claim, task phase, or task status
changes. List every open Phase 1 task in `Active Tasks`
and every open Phase 2 task exactly once in Active or Upcoming. `Claimed Tasks`
contains parked claims from Phase 3, Phase 4, and Hardening. Preserve claimants
and PRs, and sort each list by QUEUE phase/track then task number. No task
appears in more than one list. Keep section descriptions to one sentence;
dependency details stay in QUEUE and task files, not ACTIVE.

## Task types and priority

Give every numbered task exactly one primary type based on its main motivation,
not incidental implementation details:

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

A Bug fix or Gap fix owns focused regression or contract tests in its own
PR, not a hard dependency on standalone test coverage. If relevant itests
already exist, update their behavior-dependent assertions with the fix.
Otherwise add focused evidence with the fix; later itests test the implemented
contract without duplicating it. An itest that discovers a required-case bug
still owns that fix and regression in the same PR.

Test tasks depend on required production capabilities or genuinely necessary
shared infrastructure, not another task's coverage matrix or planned file.
Reuse available fixtures and keep coverage ownership as cross-links instead
of hard edges. Consumer adoption owns its required verification; equivalent
focused evidence may satisfy it without waiting for a named test task to
merge. Preserve production prerequisites when removing a test-only edge.

Phase 1 owns the final SQL schema, public APIs needed by the initial lnd
adapter, and a buildable lnd cutover. Keep their transitive open prerequisite
closure here, with honest primary types. Retain already claimed Phase 1 tasks
when regrouping; unclaimed standalone verification may follow in Phase 2B.
Keep a Design or Refactor prerequisite only when the current boundary cannot
safely deliver that contract without it; otherwise remove the hard edge and
defer the follow-up. Do not pull optional features forward to complete a test
matrix, or prescribe private architecture merely to stabilize a public API.

Before merging sql-wallet into master, Phase 1 must finish its planned schema
changes, initialize fresh SQL databases without versioned upgrade/downgrade
files, and prove that lnd compiles against the maintained Wallet. The pre-claim
gap check precedes lnd's compile cutover. Full lnd runtime and itest parity
remain Task 470's separate merge gate. After Phase 1, bugs/gaps in 2A and
verification in 2B run in parallel. Phase 3 migration and Phase 4 optional
improvements may overlap them as dependencies permit. Hardening starts only
after both 2A and 2B finish. Integration and migration retain their concrete
fixes and behavioral gates, not blanket test-task or benchmark completion.
Regrouping does not change task scope, dependencies, or requirements.

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

## Ad hoc PR accounting

One merged PR is one task. Work without a numbered entry in QUEUE needs no
retrospective task file. Count PRs merged into `btcsuite/btcwallet`'s
`sql-wallet` branch; there is no author filter. Verify the PR's merged state
and base branch on GitHub. Open and closed-unmerged PRs, and PRs merged into
other branches, do not qualify as completed ad hoc work.

Check QUEUE and ACTIVE for an existing numbered-task PR mapping before adding
an ad hoc entry. If it has one, complete that numbered task through the normal
workflow instead. Otherwise record one ad hoc row, including for a separately
merged follow-up to earlier work. A task or PR reference in historical notes
is not an additional counted task. Never count one PR as multiple tasks or
credit it in both the numbered and ad hoc lists.

Add eligible PRs directly to the ad hoc table under QUEUE's existing `Done`
section, with the PR link, short outcome, author, and UTC merge date. The PR
number is its identity; allocate no roadmap number, task file, catalog entry,
ACTIVE row, archive file, or retrospective phase. The numbered-task planning
and archival steps do not apply to these entries. Scope and verification
rules still apply to new implementation; historical accounting does not
authorize another change outside the current scope.

Derive counts from QUEUE rows. Report numbered and ad hoc totals separately,
then combine them for overall tracked and completed work. Each ad hoc row adds
one to total and done only; it does not reduce the numbered open backlog,
change phase counts, or establish completion of a numbered phase gate.
Superseded and deferred work remain separate from done.

## Task format

Start from [TASK_TEMPLATE.md](TASK_TEMPLATE.md). Create every numbered task as
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

Keep generic test-style and mocking instructions in the implementation
repository's guidance, not task scopes. Reuse existing mock setup; when it
registers `AssertExpectations` in `t.Cleanup`, do not call it again.

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
- [Tasks 470](tasks/470-lnd-wallet-port.md) and
  [487](tasks/487-compile-lnd-with-managed-wallet.md) are explicit exceptions
  for lnd runtime integration and compilation, respectively. Each merged lnd
  implementation PR closes its own task; ordinary Wallet fixes do not expand
  into lnd.
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

## Deferring a task beyond this roadmap

Use `Deferred` only for an explicit decision to archive still-valid work for
a future release cycle, not for ordinary later-phase work. Deferral is neither
completion nor supersession. Account for any implementation PR before moving
the task; do not silently abandon in-progress work.

Preserve the task under `archive/tasks/` with a short dated decision and
reactivation gate. Remove it from the open catalog and ACTIVE, move its QUEUE
row to `Deferred`, and update archive links and counts. No open task may retain
a hard dependency on deferred work. Preserve its number and original scope;
reactivate that same task through an explicit roadmap update when its gate is
met. Deferral reduces the open count, not the total, and adds no closing PR.

## Completing a task

For a numbered task, after the implementation PR merges, use the roadmap
update workflow to:

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

Preserve task numbers. Never reuse an archived number for a different task.
