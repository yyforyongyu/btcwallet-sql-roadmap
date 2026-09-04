# Task NNN — imperative outcome

Copy this file to `tasks/NNN-kebab-case.md` using the next free task number in
[tasks/README.md](tasks/README.md).

A task is ready to enter the backlog only when it owns one small, independently
reviewable outcome that can be delivered in one PR. A dependent PR may be
stacked and becomes mergeable after its hard dependencies land. Split the task
before assignment when it contains unrelated fixes, multiple independently
useful behaviors, or more than one implementation owner. Dependencies
establish merge ordering; they do not add their scope to this task.

Component: **one primary component or tightly coupled allowed implementation
boundary**.

Type: **one allowed primary type**.

Dependencies: **linked tasks or None**.

## Rationale

State the concrete problem, its observed or directly reachable cause, why it
matters now, and why it belongs in this task. For a Bug fix, name the
maintained public Wallet API, a reachable input or interleaving, and the wrong
public result. Give enough causal detail that an implementer can evaluate more
than one valid fix. Do not use this section to collect adjacent cleanup.

## Details

- Describe the externally observable behavior and invariants in scope.
- For a direct production Bug fix or Gap fix, name the affected maintained
  public Wallet API and the allowed production boundary in `package wallet`
  or directly owned `wallet/internal` code.
- Identify any unchanged legacy or external types and APIs the implementation
  consumes without taking ownership of their packages.
- List explicit exclusions and follow-up work.
- Keep the result useful and mergeable as its own PR after dependencies land.
- Do not prescribe private helper or type names, synchronization primitives,
  file layout, query decomposition, or an internal state machine unless that
  exact shape is itself an accepted contract.

## Suggested fix

Describe one small likely implementation direction and the existing mechanisms
it may reuse. Treat it as non-binding guidance: another local solution is
valid when it satisfies the same public behavior, invariants, scope, and
verification. When production changes are in scope, keep them at the Wallet-
owned boundary. Implementation discoveries do not permit adjacent work.

## Verification

- For a Bug fix, name the narrowest public-contract test and demonstrated
  regression that prove the behavior.
- For a Gap fix, name the narrowest public-contract verification. For other
  task types, name the narrowest type-appropriate evidence.
- Name backend-specific or integration coverage only when the task requires it.
- State any manual inspection needed for generated or migration output.
- Do not use broad CI completion as the primary verification plan.
- Assert public results, persisted facts, and relevant interleavings rather
  than a suggested private implementation shape.

## Definition of done

- The scoped behavior and its error cases are covered.
- The required focused verification passes.
- Public contracts and documentation are updated when the task changes them.
- Unrelated findings are recorded as separate tasks rather than folded in.
- The PR contains only this task's outcome.
