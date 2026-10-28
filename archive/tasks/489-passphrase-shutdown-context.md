# Task 489 — drain accepted passphrase rotation during shutdown

Component: **Controller.ChangePassphrase accepted-operation context**.

Type: **Bug fix**.

Dependencies: **[Task 331][t331]**.

## Rationale

The accepted passphrase handler passes Wallet lifetime to the Vault. Manager
Stop cancels that context before draining the handler, so shutdown can abort
an admitted rotation solely through runtime cancellation. The existing join
protects Store lifetime but does not keep the operation context usable.
A still-waiting `ChangePassphrase` caller can receive runtime cancellation
instead of the accepted operation's result.

## Details

- Give accepted rotation a cooperatively bounded operation context independent
  of caller and Wallet runtime cancellation. Preserve the existing serialized
  handler and drain it before Vault lock and Store closure.
- Keep caller-cancelable response waiting and forward the Vault's existing
  result. A context deadline is not a forced completion or response-time bound.
- Clarify ADR 0015's distinction between admitted-work ownership and caller
  observation without changing other lifecycle or admission contracts.
- Keep production changes in the existing `package wallet` Controller path.
  [Task 488][t488] owns request buffers; deferred [Task 352][t352] owns kvdb
  containment. Add no registry, worker, wait group, or backend reconciliation.

## Suggested fix

Use an independent operation context for the existing Vault call and retain
the current mainLoop join and shutdown order.

## Verification

- Hold an accepted Vault call, cancel its caller, and begin Manager Stop.
  The caller can return while the operation context remains live until its own
  deadline or completion; Stop waits before Vault lock and Store closure.
- Verify the operation context has a deadline without claiming that a backend
  ignoring cancellation is forcibly stopped. Reuse Vault result tests.

## Definition of done

Shutdown drains accepted rotation using a context independent of runtime
cancellation, while caller waiting remains cancelable and storage closes last.
Documentation distinguishes cooperative deadlines from hard completion bounds.

[t331]: ../archive/tasks/331-manager-wallet-lifecycle.md
[t352]: ../archive/tasks/352-kvdb-combined-passphrase-rotation.md
[t488]: 488-passphrase-request-ownership.md
