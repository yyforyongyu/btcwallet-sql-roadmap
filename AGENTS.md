# Roadmap instructions

This repository contains planning documents, not btcwallet source code.

## Sources of truth

- [README.md](README.md) explains how to use the repository.
- [QUEUE.md](QUEUE.md) owns the derived execution state of every active task.
- [ACTIVE.md](ACTIVE.md) owns assignments, active PRs, and coordination.
- Individual files under [tasks/](tasks/) own scope and dependencies.
- [tasks/README.md](tasks/README.md) owns the active catalog and next number.
- [TASK_TEMPLATE.md](TASK_TEMPLATE.md) owns the required task shape.
- [archive/](archive/) is historical evidence, not current instruction.

Do not copy the complete dependency graph into README or ACTIVE. QUEUE may
show the complete derived active-task state, but task files remain the only
authority for dependency edges.

## Assignment, claiming, and dependencies

Use these terms precisely:

- **Assigned** records the intended future owner. A blocked task may be
  assigned before its dependencies finish.
- **Claimed** means active implementation. Record its branch and PR under
  `Active branches` in [ACTIVE.md](ACTIVE.md).
- **Complete** means the implementation PR is merged, or a required design or
  ADR task is accepted and merged.

Claiming one task never claims its dependencies, directly or transitively.
Different developers may claim independent prerequisites and complete them in
parallel. A task may be claimed only after every hard dependency is complete;
assignment, an active claim, review approval, or passing CI is not completion.

Each developer may claim only one task at a time. Clear the active claim after
its implementation PR merges. A future assignment does not authorize parallel
implementation by the same developer.

Do not use stacked implementation PRs to bypass dependency completion. YY must
approve an exception explicitly, the prerequisite contract must be frozen, and
the dependent PR must remain unmergeable until the prerequisite merges.

## Roadmap PR workflow

- Use a dedicated Git worktree under `.worktrees/` and a `roadmap/<topic>`
  branch.
- Never push directly to `main`. Open a roadmap PR and stop for review.
- Rebase onto the latest `main` before allocating task numbers.
- Allocate a contiguous range from the next number in
  [tasks/README.md](tasks/README.md).
- Serialize number-allocating PR merges. If another PR used the range first,
  rebase and fold the renumbering into the introducing commit.
- Sign every commit and keep each roadmap commit small and self-contained.
- Fold accepted review corrections into the commit that introduced the task.
- Validate Markdown links, task numbering, line wrapping, and
  `git diff --check`. Do not run btcwallet CI for planning-only changes.
- Remove the worktree and its local branch after merge.

## Task format

Start from [TASK_TEMPLATE.md](TASK_TEMPLATE.md). Create every task as
`NNN-kebab-case.md` using the next free number in
[tasks/README.md](tasks/README.md).

Admit one small, atomic outcome per task and implementation PR. The task must
be independently reviewable and mergeable. Split it before assignment when it
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
[tasks/README.md](tasks/README.md) and add it exactly once to the primary map in
[QUEUE.md](QUEUE.md).

Do not add status or assignee fields to a task. Before a task is claimed or
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

After the implementation PR merges, use a roadmap PR to:

1. remove its active claim from [ACTIVE.md](ACTIVE.md);
2. move its task file under `archive/tasks/`;
3. update links from remaining dependents to the archived task;
4. update [archive/README.md](archive/README.md) and the active
   [task catalog](tasks/README.md); and
5. refresh the affected current state and downstream layers in
   [QUEUE.md](QUEUE.md).

Preserve task numbers. Never reuse an archived or superseded number.
