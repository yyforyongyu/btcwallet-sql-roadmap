# Task 343 — create next-or-exact root-derived accounts

Component: **AccountManager root-derived creation and SQL AccountStore**.

Type: **Gap fix**.

Dependencies: **[Task 332][t332], [Task 377][t377], [Task 378][t378],
[Task 392][t392], and [Task 467][t467]**.

## Rationale

The maintained account API can create only the next sequential account. Lnd
maps each key family to an exact account number under purpose 1017, so filling
every lower account is wasteful and can race another request for the same
number.

## Details

- Add an optional exact account number to Task 392's existing
  `NewAccountParams`. Nil retains sequential creation; a value requests that
  exact account.
- Limit this task to Wallet-root-derived accounts in existing scopes with
  `NoChainSync=false`. [Task 468][t468] separately activates
  `NoChainSync=true` after its receiving and recovery guards are complete.
- Validate the complete path before secret preparation or Store mutation.
- Atomically persist the account, its key facts, and the scope's next-account
  position. Sparse lower numbers remain available for later exact creation.
- Concurrent exact and sequential requests must not claim the same account or
  move the cursor backward. Invalid children and definite failures leave no
  partial account.
- Preserve Task 378's wallet-owned errors and Task 467's indeterminate-commit
  identity. Exact ambiguous-commit reconciliation remains
  [Task 389][t389].
- Keep modern kvdb's sequential subset. Reject exact-number requests before
  mutation.
- Exclude custom scopes, supplied XPub accounts, `NoChainSync=true`
  activation, watch obligations, and lnd-specific APIs.

## Suggested fix

Extend the existing account request and SQL creation transaction with an
optional exact number. The internal concurrency mechanism is not part of the
contract as long as account uniqueness, atomicity, and cursor monotonicity
hold.

## Verification

- On SQLite and PostgreSQL, create sequential and sparse exact accounts,
  reopen them, fill a lower hole, and verify the next sequential number.
- Race exact and sequential creation and prove unique accounts and a monotonic
  cursor.
- Cover invalid paths, occupied names and numbers, definite rollback, and an
  indeterminate commit without a partial public result.
- Verify this task accepts only `NoChainSync=false` and creates no watch
  obligation by account creation alone.
- Verify modern kvdb preserves sequential creation and rejects exact creation
  before secret or Store mutation.

## Definition of done

Canonical SQL scopes support one atomic next-or-exact root-derived account
operation through `NewAccountParams`, without activating `NoChainSync=true` or
adding a parallel public API.

[t332]: ../archive/tasks/332-watch-only-script-policy.md
[t377]: ../archive/tasks/377-account-manager-wallet-dto.md
[t378]: 378-account-manager-error-boundary.md
[t389]: 389-runtime-account-commit-reconciliation.md
[t392]: 392-discovery-mode-account-creation.md
[t467]: ../archive/tasks/467-indeterminate-commit-identity.md
[t468]: 468-activate-no-chain-sync-account-creation.md
