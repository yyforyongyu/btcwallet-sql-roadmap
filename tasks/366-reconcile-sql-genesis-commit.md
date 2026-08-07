# Task 366 — reconcile ambiguous SQL genesis commits

Component: **Manager and SQL Store genesis result**.

Dependencies: **[Task 329][t329]**.

## Rationale

The SQL runtime reports an indeterminate result when a commit response cannot
prove whether the transaction committed. Returning that transport error from
wallet genesis can hide a complete wallet or let callers treat partial or
conflicting state as an ordinary retryable failure.

## Details

- Handle `runtime.ErrAmbiguousTxCommit` only for the atomic SQL genesis
  operation introduced by [Task 329][t329].
- Reopen and read the exact wallet, encrypted secrets, sync state, key scopes,
  default or initial accounts, account secrets, discovery modes, and checked
  cursors described by the attempted genesis plan.
- Return success only when every persisted fact exactly matches the committed
  plan. Complete runtime assembly before publishing the Wallet.
- Return ordinary failure when no artifact from the attempted genesis exists.
- Return a wallet-owned indeterminate error for every partial, conflicting, or
  otherwise unprovable state. Never publish an indeterminate Wallet.
- Use one bounded Manager-owned reconciliation context so later public caller
  cancellation cannot interrupt determination. [Task 358][t358] owns the
  public cancellation cutoff and contextual API.
- Keep definite-failure rollback in [Task 329][t329] and duplicate-create
  policy in [Task 330][t330].
- Do not retry genesis from the transport error or add a manifest, replay
  marker, generic reconciliation framework, write lock, or cross-process
  Manager fence.

## Suggested fix

Add an operation-specific SQL genesis reader that compares the persisted state
with the attempted ciphertext/public plan after an ambiguous commit response.
Map its exact, absent, and conflicting outcomes once at the Manager boundary,
then assemble and publish only an exact committed result.

## Verification

- SQLite and PostgreSQL fault injection force an ambiguous response after an
  exact commit, before any commit, and after representative partial or
  conflicting state.
- Exact committed state returns success and publishes one usable Wallet;
  absent state returns ordinary failure and publishes none.
- Missing or conflicting wallet, secret, sync, scope, account,
  discovery-mode, and cursor facts each return the wallet-owned indeterminate
  error and publish nothing.
- Cancellation after the mutation cutoff cannot interrupt reconciliation or
  change the definitive result.
- Tests prove the implementation adds no retry, manifest, replay, generic
  lock, or duplicate-create classification.

## Definition of done

- Every ambiguous SQL genesis response resolves to exact committed, absent, or
  wallet-owned indeterminate state on both SQL dialects.
- Only exact committed state can assemble and publish a Wallet.
- Reconciliation compares the complete Task 329 genesis plan without a new
  durable marker or generic framework.
- The PR contains no definite-failure transaction rewrite, duplicate policy,
  or public contextual API.

[t329]: 329-atomic-wallet-account-creation.md
[t330]: 330-duplicate-wallet-create.md
[t358]: 358-contextual-manager-operations.md
