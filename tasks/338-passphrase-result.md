# Task 338 — latch accepted passphrase results

Component: **Controller**.

Type: **Bug fix**.

Dependencies: **[Task 331][t331] and [Task 421][t421]**.

## Rationale

At `sql-wallet` revision `2ae306dea8c5`, `Controller.ChangePassphrase` uses
`waitForReq`, which selects the buffered response or caller cancellation.
The shutdown race is separate: `handleChangePassphraseReq` passes
`w.lifetimeCtx` to the Vault, so shutdown can cancel accepted work before its
authoritative result settles. SQL and modern kvdb share the request handoff,
mainLoop handler, and Vault call; the same handler and response fix covers
both.

## Details

- Preserve cancellation before the unbuffered request handoff. After handler
  acceptance, return exactly its buffered success, failure, or indeterminate
  result without racing caller or Wallet cancellation.
- Give the accepted handler a bounded context independent of Wallet lifetime.
  Reuse mainLoop's existing serialization and Wallet stop's `w.wg` drain:
  accepted work must finish before Vault lock and Store close.
- Preserve Task 421's SQL result classification. Deferred Task 352 owns
  kvdb uncertain-rotation containment; this task forwards the Vault result
  without adding backend reconciliation or duplicating that fix.
- Keep cross-Manager and cross-process credential fencing unsupported. Add
  no generic operation registry or separate completion wait group.

## Suggested fix

After successful `sendReq`, receive the buffered operation result directly.
Keep rotation in the existing serialized control handler and joined mainLoop;
use a bounded handler context and translate internal errors only once.

## Verification

- Cancellation before handoff returns the caller error without mutation.
- After acceptance, caller cancellation and Manager Stop cannot replace or
  suppress the handler's success, failure, or indeterminate result on any
  maintained backend.
- Use the existing shutdown boundary to prove accepted work finishes before
  Vault lock and Store close. Concurrent rotations remain serialized.
- Reuse backend Vault classification tests; add no Controller Store rereads.

## Definition of done

Every accepted passphrase request returns its authoritative operation result.
Shutdown drains that work through the existing lifecycle, and pre-handoff
cancellation remains supported without another backend delivery layer.

[t331]: ../archive/tasks/331-manager-wallet-lifecycle.md
[t421]: ../archive/tasks/421-sql-passphrase-commit-reconciliation.md
