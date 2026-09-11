# Task 468 — activate NoChainSync account creation

Component: **AccountManager NoChainSync creation admission**.

Type: **Gap fix**.

Dependencies: **[Task 343][t343], [Task 393][t393], and [Task 452][t452]**.

## Rationale

Task 392 lets callers request `NoChainSync`, but rejects true until account
creation, receiving APIs, and recovery all honor the policy. Once those guards
exist, lnd needs the existing account API to create purpose-1017 accounts with
chain synchronization disabled.

## Details

- Accept `NoChainSync=true` through the existing `NewAccountParams` path for
  Task 343's exact root-derived SQL accounts.
- Persist and return the requested value in the same atomic account creation
  operation.
- Do not derive receiving addresses, install watches, or admit the account to
  recovery as a side effect of activation.
- Preserve [Task 378][t378]'s public errors and modern kvdb's unsupported
  result.
- Exclude custom-scope creation, supplied XPub accounts, a new public method or
  request type, and changes to the receiving or recovery guards.

## Suggested fix

Remove the temporary true-value rejection after the three dependencies land
and pass the existing request value through Task 343's creation path. No
particular private helper or transaction layout is required.

## Verification

- On SQLite and PostgreSQL, create and reopen an exact
  `NoChainSync=true` account and verify the returned and persisted value.
- Prove receiving and positive-lookahead recovery remain rejected through the
  dependency-owned public errors, with no address, watch, or horizon mutation.
- Prove ordinary accounts and modern kvdb behavior remain unchanged.

## Definition of done

The existing account-creation request can activate one exact root-derived SQL
account with `NoChainSync=true` only after every required safety guard exists.

[t343]: 343-exact-account-creation.md
[t378]: ../archive/tasks/378-account-manager-error-boundary.md
[t393]: ../archive/tasks/393-key-only-receiving-rejection.md
[t452]: 452-key-only-recovery-derivation.md
