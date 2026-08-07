# Task 338 — return a definitive SQL passphrase-change result

Component: **Controller**.

Dependencies: **[Task 331][t331]**.

## Rationale

After the Controller accepts a passphrase request, caller cancellation can win
the response race even though the Vault commits the new credential. The caller
cannot know which passphrase is authoritative.

## Details

- Cancellation before the unbuffered request handoff remains supported.
- An accepted request must publish exactly one buffered result.
- Keep the accepted-request, result, shutdown-join, and ambiguous-commit
  behavior SQL-only. It contains no kvdb rotation or legacy waddrmgr change
  and is consumed by the lnd SQL password path.
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

## Suggested fix

Receive the handler response directly after `sendReq` succeeds, independent
of caller or wallet-lifetime cancellation.
Audit every accepted SQL handler exit so it always sends one response. On an
ambiguous SQL commit, reconcile the persisted credential facts and, if
necessary, their generation before selecting success, failure, or the public
indeterminate result. Give this handler one bounded context and completion that
shutdown joins; do not add a generic Controller operation registry or cross-
Manager fencing.

## Verification

SQL tests prove that acceptance-latched success and Vault errors remain
definitive under concurrent cancellation, while pre-acceptance cancellation
still returns the context error. A barrier-controlled wallet shutdown after
acceptance still joins the handler before Vault lock and returns the handler
result. SQL ambiguous-commit tests prove old, new, readable neither-state, and
unavailable outcomes without claiming an authoritative passphrase when it
cannot be known. Concurrent rotations on one published Wallet from the same
old credential to different new credentials produce one successful transition;
the loser has a definitive result and changes no durable or live key state. No
cross-Manager or multi-process guarantee is added.

## Definition of done

- An accepted SQL passphrase request produces exactly one definitive or typed
  indeterminate result that shutdown joins before Vault lock or Store close.
- Pre-acceptance cancellation remains supported, while post-acceptance caller
  or Wallet cancellation cannot replace the handler result.
- Modern-kvdb parity is owned separately by
  [Task 362](362-kvdb-definitive-passphrase-result.md), and no generic
  Controller operation registry is added.

[t331]: 331-manager-wallet-lifecycle.md
