# Roadmap instructions

This repository contains planning documents, not btcwallet source code.

## Sources of truth

- [README.md](README.md) explains how to use the repository.
- [QUEUE.md](QUEUE.md) owns the dependency phase of every open task.
- [ACTIVE.md](ACTIVE.md) owns the active-task focus, claims, in-progress PRs,
  and coordination.
- Individual files under [tasks/](tasks/) own scope and dependencies.
- [tasks/README.md](tasks/README.md) owns the open catalog and next number.
- [TASK_TEMPLATE.md](TASK_TEMPLATE.md) owns the required task shape.
- [archive/](archive/) is historical evidence, not current instruction.

Do not copy the complete dependency graph into README or ACTIVE. QUEUE may
show the complete derived phase map, but task files remain the only authority
for dependency edges and ACTIVE remains the only authority for live state.

## Activity, claiming, and dependencies

Use these terms precisely:

- **Active** means current development focus. Every Phase 1 task is active,
  whether claimed or unclaimed.
- **Claimed** records the intended owner. A task may be claimed before its
  dependencies finish and therefore may be claimed without being active.
- **In progress** means implementation has started. Mark an active task in
  progress before implementation and add its PR when opened.
- **Complete** means the implementation PR is merged, or a required design or
  ADR task is accepted and merged.

Claiming one task never claims its dependencies, directly or transitively.
Different developers may claim independent prerequisites and complete them in
parallel. Implementation may start only after every hard dependency and gate
is complete; a claim, active status, review approval, or passing CI is not
completion.

Each developer may have only one task in progress at a time. Archive the task
after its implementation PR merges. Additional claims do not authorize
parallel implementation by the same developer.

Do not use stacked implementation PRs to bypass dependency completion. YY must
approve an exception explicitly, the prerequisite contract must be frozen, and
the dependent PR must remain unmergeable until the prerequisite merges.

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
3. `Dependencies: **linked tasks or None**.`
4. `## Rationale`
5. `## Details`
6. `## Suggested fix`
7. `## Verification`
8. `## Definition of done`

Use `Rationale` for the concrete problem, `Details` for minimal scope and
explicit exclusions, `Suggested fix` for the smallest likely implementation,
`Verification` for focused evidence, and `Definition of done` for measurable
completion.

Link every dependency to its task file. Add the task to
[tasks/README.md](tasks/README.md) and add it exactly once to the phase map in
[QUEUE.md](QUEUE.md).

Do not add status or claimant fields to a task. Before a task is claimed or
materially revised, bring it to this format and split it if it fails the
atomicity rule. Do not mechanically expand an unchanged task to fill sections.

## Scope rules

- Focus on the maintained public APIs under `wallet/`.
- Avoid voluntary RPC, deprecated-wallet, and unrelated daemon changes.
- Keep kvdb legacy and single-wallet behavior isolated from the SQL design.
- Do not add create manifests, generic write locks, replay state machines, or
  cross-process Manager fencing without a separately accepted task.
- Keep integration-test tasks test-only and benchmark tasks measurement-only.
- Record an exposed defect as its own implementation task.
- Keep findings in the owning task instead of storing raw review reports.

## Completing a task

After the implementation PR merges, use the roadmap update workflow to:

1. remove its row from [ACTIVE.md](ACTIVE.md);
2. move its task file under `archive/tasks/`;
3. update links from remaining dependents to the archived task;
4. update [archive/README.md](archive/README.md) and the active
   [open task catalog](tasks/README.md); and
5. refresh the affected phase and downstream phases in
   [QUEUE.md](QUEUE.md).

Preserve task numbers. Never reuse an archived or superseded number.
