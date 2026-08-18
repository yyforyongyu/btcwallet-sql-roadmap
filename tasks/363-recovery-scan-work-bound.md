# Task 363 — bound aggregate recovery derivation work

Component: **recovery horizon derivation**.

Type: **Bug fix**.

Dependencies: **[Task 342][t342]**.

## Rationale

Accepting a bounded per-configuration recovery window does not
bound total work after multiplying lookahead across accounts and branches or
adding same-block expansion and invalid-child replacement.

## Details

- Enforce `wallet.MaxRecoveryWindow` as one aggregate attempt budget for a
  scan while retaining the caller's accepted recovery window as each branch's
  lookahead target.
- Debit every derivation attempt during initialization, same-block horizon
  extension, and replacement after an invalid child. No path replenishes the
  budget.
- Use deterministic account/branch ordering so the same persisted state fails
  or succeeds at the same attempt.
- Check `nextUnfound + window + invalid`, `index + 1`, horizon increment, and
  every child progression before overflow or entry into the hardened range.
- Return one wallet-owned work-exhausted/boundary result without persisting a
  partial horizon mutation.
- Keep configuration validation, public targets, persisted-target loading, and
  durable scan jobs out of scope.

## Suggested fix

Thread one checked attempt counter through recovery-state
construction and every horizon expansion path, and replace raw `uint32`
arithmetic with checked helpers.

## Verification

Many-account/branch initialization, same-block repeated
expansion, invalid-child replacement, exact-budget, budget-plus-one, and
near-hardened-boundary vectors prove no extra derivation or wraparound.

## Definition of done

Every recovery scan has one deterministic aggregate
attempt cap and checked child arithmetic, independent of target and persisted
watch loading.

[t342]: ../archive/tasks/342-recovery-window-validation.md
