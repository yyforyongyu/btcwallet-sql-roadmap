# Task 343 — create next-or-exact root-derived accounts

Component: **AccountManager root-derived creation and SQL AccountStore**.

Type: **Gap fix**.

Dependencies: **[Task 332][t332], [Task 377][t377], [Task 378][t378],
[Task 392][t392], [Task 393][t393], [Task 452][t452], and
[Task 467][t467]**.

## Rationale

The maintained account API can create only the next sequential account. Lnd
maps each key family to an exact account number under purpose 1017, so filling
every lower account is wasteful and still does not provide a reliable
concurrent claim for the requested number.

An exact account also needs its `NoChainSync` policy committed with its key and
cursor state. Otherwise a purpose-1017 account can accidentally enter normal
address discovery or be only partly visible after a failure.

## Details

- Add an optional exact account number to Task 392's `NewAccountParams`. Nil
  retains sequential creation; a value requests that exact account.
- Limit this task to Wallet-root-derived accounts in existing canonical
  scopes. Task 390 adds custom scopes and Task 391 owns supplied XPub accounts.
- Validate the complete path before key preparation or Store mutation.
- Atomically persist the account, its public and encrypted facts,
  `NoChainSync`, and the scope's next-account position. Sparse lower numbers
  remain available for later exact creation.
- Concurrent exact and sequential requests must not claim the same account or
  move the cursor backward. Invalid children and definite failures leave no
  partial account.
- Preserve deterministic invalid-request, occupied-name, and occupied-number
  errors through Task 378's wallet-owned identities.
- Return Task 467's indeterminate-commit identity when durable success cannot
  be established. Exact reconciliation remains Task 389.
- Keep modern kvdb's existing sequential subset. Exact-number requests return
  `ErrAccountOperationUnsupported` before mutation.
- Do not add an lnd-specific method, account-kind enum, key-family registry,
  custom-scope behavior, supplied-XPub behavior, or watch obligation.

## Suggested fix

Extend the existing account request and SQL account-creation transaction with
an optional exact number. An expected-cursor update is one likely approach,
but any local Store implementation is valid if it preserves the atomicity,
uniqueness, and cursor invariants above.

## Verification

- On SQLite and PostgreSQL, create sequential and sparse exact accounts,
  reopen them, fill a lower hole, and verify the next sequential number.
- Race exact and sequential creation and prove unique accounts and monotonic
  cursor state.
- Cover invalid paths, occupied names and numbers, definite rollback, and
  indeterminate commit without partial public results.
- Verify both `NoChainSync` values and prove no watch obligation is created by
  account creation alone.
- Verify modern kvdb preserves sequential creation and rejects exact creation
  before secret or Store mutation.

## Definition of done

Canonical SQL scopes support one atomic next-or-exact root-derived account
operation through `NewAccountParams`. Its account identity, secret facts,
`NoChainSync` value, and cursor state remain consistent across concurrency,
failure, and reopen without an lnd-specific API.

[t332]: ../archive/tasks/332-watch-only-script-policy.md
[t377]: ../archive/tasks/377-account-manager-wallet-dto.md
[t378]: 378-account-manager-error-boundary.md
[t389]: 389-runtime-account-commit-reconciliation.md
[t390]: 390-custom-scope-account-creation.md
[t391]: 391-path-bound-watch-only-xpub-accounts.md
[t392]: 392-discovery-mode-account-creation.md
[t393]: 393-key-only-receiving-rejection.md
[t452]: 452-key-only-recovery-derivation.md
[t467]: ../archive/tasks/467-indeterminate-commit-identity.md
