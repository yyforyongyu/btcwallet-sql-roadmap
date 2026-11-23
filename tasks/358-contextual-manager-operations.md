# Task 358 — make Manager creation cancellable

Component: **Manager.Create**.

Type: **Gap fix**.

Dependencies: **[Task 329][t329], [Task 331][t331], [Task 456][t456],
and [Task 467][t467]**.

## Rationale

[PR #1496][pr1496] routes [`Manager.Create`][create] through the Manager's
runtime queue. Admission uses Manager lifetime, and accepted requests settle
without cancellation. The method accepts no caller context, so a caller
cannot cancel its own request while it is still safe to abort. Once a write
may have committed, returning a context error can misreport the result.

Wait for Task 329's atomic wallet-and-account genesis before adding public
cancellation, so cancellation does not introduce failures between separately
committed genesis steps. The initial lnd port uses Task 331's existing Create
signature and shutdown draining; contextual creation follows for migration.

## Details

- Replace the public method with `Create(ctx, params)`, with wallet identity
  carried by the existing `CreateWalletParams`. Retain no background-context
  compatibility wrapper.
- Honor caller cancellation while validation, preparation, Manager admission,
  and backend write admission remain safely abortable.
- Define one documented mutation cutoff. Cancellation before it returns the
  caller's context error without durable or cached state. At or after it,
  Manager completes classification, assembly, private Wallet start, and
  publication under Manager-owned lifetime. The caller waits for that result;
  finishing creation work cannot cancel the resulting Wallet runtime.
- Compose with Task 331 so Stop cannot close Store underneath an accepted
  Create, while a Create rejected after shutdown performs no backend work.
- Map an ambiguous SQL commit to Task 467's wallet-owned identity and do not
  retry, publish an uncertain Wallet, or assume rollback. Add no automatic
  genesis readback or reconciliation.
- Preserve the honest kvdb limitation where waiting inside its legacy write or
  key-derivation path cannot be cooperatively interrupted.
- Clean ordinary prepared secret material on cancellation and definite local
  failure. Exhaustive secret-lifetime hardening remains Task 367.
- Do not add a generic operation registry, create manifest, cross-process
  fence, or atomic-genesis work owned by Task 329.

## Suggested fix

Thread the caller context to the last safely abortable backend boundary, then
switch to a bounded Manager-owned context. The precise gate, claim marker, and
context wiring are implementation choices provided the observable cutoff is
single and race-safe.

## Verification

- Cancel while waiting for Manager and backend admission and prove no durable
  or cached state is created.
- Race cancellation immediately around the mutation cutoff on kvdb, SQLite,
  and PostgreSQL and verify the documented winner and result. On SQL, reopen
  to prove cancellation leaves either no genesis or the complete promised
  wallet-and-account state, never partial genesis.
- Race accepted Create with Stop and prove Store remains open until Create
  settles and a successful result is fully started and published.
- Inject an ambiguous SQL commit and verify no retry or Wallet publication and
  the shared indeterminate identity.
- Cancel the request after success and prove it does not control Wallet
  lifetime.

## Definition of done

`Manager.Create(ctx, params)` has one truthful cancellation cutoff and composes
with terminal Manager shutdown without exposing a partial or uncertain Wallet.
It does not absorb atomic genesis, reconciliation, secret-hardening, or generic
operation machinery.

[t329]: 329-atomic-wallet-account-creation.md
[t331]: ../archive/tasks/331-manager-wallet-lifecycle.md
[t367]: ../archive/tasks/367-bound-sql-genesis-secrets.md
[t456]: ../archive/tasks/456-manager-runtime-configuration.md
[t467]: ../archive/tasks/467-indeterminate-commit-identity.md
[pr1496]: https://github.com/btcsuite/btcwallet/pull/1496
[create]: https://github.com/btcsuite/btcwallet/blob/70f5629b6f5a75b29582eac9251c804d61bd7b20/wallet/manager.go#L624
