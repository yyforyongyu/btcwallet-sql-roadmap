# Task 338 — latch accepted SQL passphrase results

Component: **Controller**.

Dependencies: **[Task 331][t331], [Task 368][t368], and [Task 421][t421]**.

## Rationale

After the Controller accepts a passphrase request, caller cancellation or
Wallet shutdown can win the response wait even though the SQL Vault has
settled the durable credential. The caller then receives a transport result
instead of the authoritative operation result.

## Details

- Preserve cancellation before the unbuffered request handoff.
- Once the handler accepts a request, publish exactly one buffered result and
  have the caller wait for that result without selecting on caller or Wallet
  cancellation.
- Run the accepted SQL handler under one bounded handler-owned context that
  does not derive from `Wallet.lifetimeCtx`.
- Register one ChangePassphrase-specific completion before mutation. Wallet
  shutdown joins it through Task 368 before locking the Vault, and
  Task 331 joins Wallet shutdown before closing the Store.
- Consume Task 421's proven-new, proven-old, or typed
  indeterminate result and translate its internal errors once to a
  wallet-owned public contract.
- Serialize rotations through the one Wallet/Vault instance. Cross-Manager
  and cross-process credential fencing remain unsupported.
- Keep this SQL-only. Task 362 owns modern-kvdb Controller parity.
- Do not add a generic operation registry, redesign other Controller calls,
  or perform Store reconciliation in the Controller.

## Suggested fix

After `sendReq` succeeds, receive directly from the buffered response channel.
Give the passphrase handler one bounded context and one operation-specific
completion registered before the Vault call. Map the Vault's exact outcome to
the public result, send it once, and close the completion on every exit.

## Verification

- A cancellation barrier before handoff returns the caller context error.
- Barriers immediately after handoff prove caller cancellation and Wallet
  shutdown cannot replace or suppress the handler result.
- Wallet shutdown waits for the accepted handler before Vault lock, and
  Manager closure waits for that Wallet shutdown before Store close.
- Proven-new, proven-old, and indeterminate Vault outcomes each map to the
  exact public result without another Store read in the Controller.
- Concurrent rotations on one published Wallet serialize and every caller
  receives one operation result.

## Definition of done

- Every accepted SQL passphrase request returns exactly one authoritative
  success, failure, or typed indeterminate result.
- Shutdown joins accepted passphrase work before Vault lock and Store close.
- Pre-handoff cancellation remains supported, and the PR adds no generic
  operation registry or cross-Manager guarantee.

[t331]: ../archive/tasks/331-manager-wallet-lifecycle.md
[t368]: 368-wallet-startup-shutdown.md
[t421]: 421-sql-passphrase-commit-reconciliation.md
