# Task 403 — register raw imports before public success

Component: **AddressManager SQL raw-import mutation**.

Type: **Bug fix**.

Dependencies: **[Task 332][t332], [Task 402][t402], and
[Task 467][t467]**.

## Rationale

A committed SQL public-key or Taproot-script import can remain unmonitored
after registration fails, and a duplicate insert can prevent retrying its
watch. Public success must establish live monitoring.

## Details

- Wire SQL `ImportPublicKey` and `ImportTaprootScript` into Task 402's
  `WatchAddrsFromTip` path on btcd/bitcoind, including polling, after commit
  and before public success. Use the Wallet lifetime context so caller
  cancellation cannot abandon committed registration. Leave kvdb and Neutrino
  behavior unchanged.
- Preserve Task 332's import policy and keep secret encryption outside the
  Store transaction. For an existing address in the same Wallet, preserve its
  row, ownership, and secrets and return stored metadata. Concurrent duplicates
  reuse one row; new and duplicate imports both attempt registration.
- On registration failure, return an error without a successful result and
  retain committed data. Keep the Manager running; repeating the import
  retries registration. Reuse Task 402's startup watch reconstruction.
- Map an ambiguous SQL commit to `ErrIndeterminateCommit`, with no returned
  success or automatic retry. Add no exact-state readback, secret-provenance
  comparison, or classification-only serialization.
- Add no public API, background retry mechanism, watch metadata, or readiness
  model. Start no historical scan or mempool sweep.

## Suggested fix

Reuse the existing wallet-scoped address lookup and SQL transaction boundary
for duplicate imports, then replace direct `NotifyReceived` calls with
`WatchAddrsFromTip`. Keep conservative commit-error mapping.

## Verification

- SQLite and PostgreSQL cover commit-before-registration ordering for both
  imports and concurrent duplicates without changing stored rows or secrets.
- Fail registration, then repeat the import successfully; the Manager stays
  running and one preserved address row remains. Reuse startup-reconstruction
  coverage.
- Caller cancellation cannot abandon committed registration; Wallet shutdown
  cancels and joins it.
- Ambiguous commit matches `ErrIndeterminateCommit` without success or retry.
- A payment after successful import is observed before mining on
  btcd/bitcoind. Registration starts no history scan; kvdb and Neutrino remain
  unchanged.

## Definition of done

SQL raw imports establish live monitoring before success on btcd/bitcoind.
Failed registration is safely retried through the same import API without
changing existing data or stopping the Manager.

[t332]: 332-watch-only-script-policy.md
[t402]: 402-replay-live-watch-obligations.md
[t467]: 467-indeterminate-commit-identity.md
