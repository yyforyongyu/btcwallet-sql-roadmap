# Task 338 — latch accepted SQL passphrase results

Component: **Controller**.

Type: **Bug fix**.

Dependencies: **[Task 331][t331] and [Task 421][t421]**.

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
- Register one ChangePassphrase-specific completion before mutation. The
  private Wallet stop owned by [Task 458][t458] and integrated by Task 331 is
  the sole owner that joins it before locking the Vault. Task 331's Manager Stop
  joins private Wallet stop before closing Store and does not join this
  completion separately.
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
- Barriers immediately after handoff prove caller cancellation and Manager Stop
  cannot replace or suppress the handler result.
- Private Wallet stop waits for the accepted handler before locking the Vault;
  Manager Stop reaches Store close only after joining that Wallet stop.
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

[t331]: 331-manager-wallet-lifecycle.md
[t421]: ../archive/tasks/421-sql-passphrase-commit-reconciliation.md
[t458]: 458-manager-operation-admission.md
