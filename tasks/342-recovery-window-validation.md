# Task 342 — accept bounded recovery windows

Component: **wallet configuration and Controller**.

Dependencies: **[Task 341][t341]**.

## Rationale

The current minimum rejects zero and small recovery windows even though the
recovery code and consumers use them. Configuration needs a public upper bound
without changing valid caller values.

## Details

- Define the public `wallet.MaxRecoveryWindow` as 100,000. Accept every value
  from zero through that maximum unchanged and reject maximum-plus-one before
  Store or chain access.
- Keep this validation change limited to the public maximum, unified
  configuration, stable wallet-owned error, and boundary behavior.
- Do not change derivation, target selection, scan budgeting, or RPC signed
  value handling. [Task 363][t363] owns recovery scan work and targets.

## Suggested fix

Remove the artificial minimum and accept `0..wallet.MaxRecoveryWindow`
exactly in unified configuration. Translate the first excess value to the
stable wallet-owned error before dependency access. Keep negative RPC
validation in the RPC consumer rather than adding signed wallet fields.

## Verification

Focused configuration tests prove 0, 1, 2, 19, 20, and
`wallet.MaxRecoveryWindow` are accepted unchanged. Maximum-plus-one returns
the stable public error before Store or chain access, and the RPC consumer
retains its separate negative-value validation.

## Definition of done

- Unified configuration accepts `0..wallet.MaxRecoveryWindow` unchanged and
  rejects the first excess value before Store or chain access.
- Aggregate scan budgeting and semantic-account target behavior are owned by
  [Task 363](363-recovery-scan-work-bound.md), not this validation task.

[t341]: 341-public-role-contract.md
[t363]: 363-recovery-scan-work-bound.md
