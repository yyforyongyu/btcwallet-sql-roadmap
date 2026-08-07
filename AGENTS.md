# Roadmap instructions

This repository contains planning documents, not btcwallet source code.

- Read [README.md](README.md) and [ACTIVE.md](ACTIVE.md) before claiming work.
- Treat [README.md](README.md) as the canonical priority and dependency graph.
- Treat [tasks/](tasks/) as the canonical executable task specifications.
- Treat [ACTIVE.md](ACTIVE.md) as the ownership ledger. Claim one task at a
  time before starting implementation and clear the claim after its PR merges.
- Do not combine tasks merely because they touch the same component.
- Preserve task numbers and update every affected link when moving a task.
- Fold accepted review findings into the relevant task instead of storing raw
  review reports.
- Treat [archive/](archive/) as historical evidence, not current instruction.
- Keep changes concise, local, and wrapped near 80 columns.

## Task format

Start from [TASK_TEMPLATE.md](TASK_TEMPLATE.md). Create every task as
`NNN-kebab-case.md` using the next free number recorded in
[README.md](README.md).

Admit one small, atomic outcome per task and PR. The task must be independently
reviewable and mergeable. Split it before assignment when it contains unrelated
fixes, multiple independently useful behaviors, or more than one implementation
owner. Dependencies establish ordering only; they do not import another task's
scope.

Keep this section order:

1. `# Task NNN — imperative title`
2. `Component: **specific owner**.`
3. `Dependencies: **linked tasks or None**.`
4. `## Rationale`
5. `## Details`
6. `## Suggested fix`
7. `## Verification`
8. `## Definition of done`

Use `Rationale` for the concrete problem, `Details` for the minimal scope and
explicit exclusions, `Suggested fix` for the smallest likely implementation
shape, `Verification` for focused evidence, and `Definition of done` for
measurable completion. Link every dependency and task milestone to its task
file. Add the task to the canonical backlog and dependency order in
[README.md](README.md).

Before a task is newly claimed or materially revised, bring it to this format
and split it if it fails the atomicity rule. Do not mechanically expand an
unchanged task merely to fill every section.

Do not add a task status field. Current ownership and PR state belong only in
[ACTIVE.md](ACTIVE.md).
