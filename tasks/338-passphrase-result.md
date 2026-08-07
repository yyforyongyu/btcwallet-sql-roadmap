# Task 338 — return a definitive passphrase-change result

Component: **Controller**.

Dependencies: **Complete task: [Task 331][t331] plus [Task 352][t352] while modern kvdb
remains; otherwise [Task 353][t353] removes the kvdb clause. [`338.sql-definitive`][t338]:
[Task 331][t331] only**.

## Rationale

After the Controller accepts a passphrase request, caller cancellation can win
the response race even though the Vault commits the new credential. The caller
cannot know which passphrase is authoritative.

## Details

- Cancellation before the unbuffered request handoff remains supported.
- An accepted request must publish exactly one buffered result.
- Name the SQL-only accepted-request, result, shutdown-join, and ambiguous-
  commit behavior [`338.sql-definitive`][t338]. It contains no kvdb rotation or legacy
  waddrmgr change and is the artifact consumed by the lnd SQL password path.
- A SQL ambiguous-commit error is not a definitive credential result. Re-read
  persisted Vault parameters and prove old or new credentials; if the Store
  remains unavailable or neither state is provable, return a wallet-owned
  indeterminate sentinel.
- Serialize accepted rotations through [Task 331][t331]'s single published Wallet.
  Multiple writable Managers and cross-process credential fencing remain
  unsupported. Retain a durable credential generation only if it is needed to
  reconcile an ambiguous commit, not as a cross-Manager lock.
- Once the handler accepts a request, run mutation and reconciliation under one
  bounded handler-owned context independent of `Wallet.lifetimeCtx`. Register
  one operation-specific completion before mutation; Wallet and Manager
  shutdown join it before locking the Vault or closing the Store.
- Wallet shutdown after request acceptance cannot cancel, replace, or suppress
  the one handler result; only cancellation before handoff may win.
- Limit this task to `ChangePassphrase`; do not redesign every Controller call.
- Do not add a one-line wait helper or an operation registry.

## Potential fix

Land [`338.sql-definitive`][t338] by receiving the handler response directly after
`sendReq` succeeds, independent of caller or wallet-lifetime cancellation.
Audit every accepted SQL handler exit so it always sends one response. On an
ambiguous SQL commit, reconcile the persisted credential facts and, if
necessary, their generation before selecting success, failure, or the public
indeterminate result. Give this handler one bounded context and completion that
shutdown joins; do not add a generic Controller operation registry or cross-
Manager fencing. The complete task later adds [Task 352][t352]'s coherent kvdb result.
If [Task 353][t353] retires modern kvdb first, remove that legacy clause instead.

## Done when

The [`338.sql-definitive`][t338] artifact proves on SQL that acceptance-latched success
and Vault errors remain definitive under concurrent cancellation, while pre-
acceptance cancellation still returns the context error. A barrier-controlled
wallet shutdown after acceptance still joins the handler before Vault lock and
returns the handler result. SQL ambiguous-commit tests prove old, new, readable
neither-state, and unavailable outcomes without claiming an authoritative
passphrase when it cannot be known. The complete task repeats the applicable
contract on kvdb after [Task 352][t352] while that backend remains. If [Task 353][t353] retires
modern kvdb first, the SQL artifact completes the surviving scope after the
kvdb clauses are removed. Concurrent rotations on one published Wallet from the
same old credential to different new credentials produce one successful
transition; the loser has a definitive result and changes no durable or live
key state. No cross-Manager or multi-process guarantee is added.

[t331]: 331-manager-wallet-lifecycle.md
[t338]: 338-passphrase-result.md
[t352]: 352-kvdb-combined-passphrase-rotation.md
[t353]: 353-retire-modern-kvdb-backend.md
