# Task NNN — imperative outcome

Copy this file to `tasks/NNN-kebab-case.md` using the next free task number in
[tasks/README.md](tasks/README.md).

A task is ready to enter the backlog only when it owns one small, independently
reviewable outcome that can be delivered in one PR. Split it before assignment
when it contains unrelated fixes, multiple independently useful behaviors, or
more than one implementation owner. Dependencies establish ordering; they do
not add their scope to this task.

Component: **one primary component or tightly coupled boundary**.

Type: **one allowed primary type**.

Dependencies: **linked tasks or None**.

## Rationale

State the concrete problem, why it matters now, and why it belongs in this
task. Do not use this section to collect adjacent cleanup.

## Details

- Describe the exact behavior in scope.
- Name the relevant API or storage boundary.
- List explicit exclusions and follow-up work.
- Keep the result useful and independently mergeable.

## Suggested fix

Describe the smallest likely implementation shape, including the component
that should own the behavior. This is guidance, not permission to broaden the
task when implementation uncovers an adjacent issue.

## Verification

- Name the narrowest unit or package test that proves the behavior.
- Name backend-specific or integration coverage only when the task requires it.
- State any manual inspection needed for generated or migration output.
- Do not use broad CI completion as the primary verification plan.

## Definition of done

- The scoped behavior and its error cases are covered.
- The required focused verification passes.
- Public contracts and documentation are updated when the task changes them.
- Unrelated findings are recorded as separate tasks rather than folded in.
- The PR contains only this task's outcome.
