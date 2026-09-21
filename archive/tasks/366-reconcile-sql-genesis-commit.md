# Task 366 — reconcile ambiguous SQL genesis commits

Component: **Manager and SQL Store genesis result**.

Type: **Bug fix**.

Dependencies: **[Task 329][t329] and [Task 467][t467]**.

## Archive decision

Superseded on 2026-09-21 without replacement. The accepted decision uses
Task 329's single genesis transaction and Task 358's conservative
ErrIndeterminateCommit mapping, without automatic readback or retry.
This unimplemented reconciliation task is not complete.

## Rationale

The SQL runtime reports an indeterminate result when a commit response cannot
prove whether the transaction committed. Returning that transport error from
wallet genesis can hide a complete wallet or let callers treat partial or
conflicting state as an ordinary retryable failure.

## Details

- Handle `runtime.ErrAmbiguousTxCommit` only for the atomic SQL genesis
  operation introduced by [Task 329][t329].
- Reopen and read the exact wallet, encrypted secrets, sync state, key scopes,
  default or initial accounts, account secrets, `NoChainSync` values, and checked
  cursors described by the attempted genesis plan.
- Return success only when every persisted fact exactly matches the committed
  plan. Complete runtime assembly before publishing the Wallet.
- Return ordinary failure when no artifact from the attempted genesis exists.
- For every partial, conflicting, or otherwise unprovable state, fix Task
  467's wallet-owned `ErrIndeterminateCommit` and enter or observe
  [Task 331][t331]'s existing terminal Create path. Privately stop and join
  every unpublished candidate, and initiate or join Manager shutdown before
  returning. Never publish an indeterminate Wallet.
- Use one bounded Manager-owned reconciliation context so later public caller
  cancellation cannot interrupt determination. [Task 358][t358] owns the
  public cancellation cutoff and contextual API. Reconciliation must remain
  usable after the commit operation's context expires; Stop still joins it
  before Store closure. The current non-contextual Create can already own
  bounded rereads, so Task 358 is not a prerequisite.
- Keep definite-failure rollback in [Task 329][t329] and duplicate-create
  policy in [Task 330][t330].
- Do not retry genesis from the transport error or add a manifest, replay
  marker, generic reconciliation framework, write lock, or cross-process
  Manager fence.

## Suggested fix

Add an operation-specific SQL genesis reader that compares the persisted state
with the attempted ciphertext/public plan after an ambiguous commit response.
Map its exact, absent, and conflicting outcomes once at the Manager boundary.
Assemble and publish only an exact committed result, preserve clean absence as
ordinary failure, and route every unprovable result through Task 331's existing
terminal cleanup before returning the fixed error.

## Verification

- SQLite and PostgreSQL fault injection force an ambiguous response after an
  exact commit, before any commit, and after representative partial or
  conflicting state.
- Exact committed state returns success and publishes one usable Wallet;
  absent state returns ordinary failure and publishes none. Neither definitive
  result enters terminal cleanup.
- Missing or conflicting wallet, secret, sync, scope, account,
  `NoChainSync`, and cursor facts each return the wallet-owned indeterminate
  error, satisfy `errors.Is(err, wallet.ErrIndeterminateCommit)`, publish
  nothing, and leave Manager stopped after private candidate cleanup. A queued
  Create returns `ErrManagerStopped` before backend or cache work.
- Reuse Task 331's cleanup barrier to prove an indeterminate Create does not
  return before terminal cleanup and a concurrent Create cannot reach backend
  or cache work once Manager shutdown begins.
- Cancellation after the mutation cutoff cannot interrupt reconciliation or
  change the definitive result.
- Tests prove the implementation adds no retry, manifest, replay, generic
  lock, or duplicate-create classification.

## Definition of done

- Every ambiguous SQL genesis response resolves to exact committed, absent, or
  wallet-owned indeterminate state on both SQL dialects.
- Only exact committed state can assemble and publish a Wallet; every residual
  indeterminate result returns after Task 331's terminal cleanup.
- Reconciliation compares the complete Task 329 genesis plan without a new
  durable marker or generic framework.
- The PR contains no definite-failure transaction rewrite, duplicate policy,
  or public contextual API.

[t329]: ../../tasks/329-atomic-wallet-account-creation.md
[t330]: ../../tasks/330-duplicate-wallet-create.md
[t331]: 331-manager-wallet-lifecycle.md
[t358]: ../../tasks/358-contextual-manager-operations.md
[t467]: 467-indeterminate-commit-identity.md
