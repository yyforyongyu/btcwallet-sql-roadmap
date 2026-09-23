# Task 403 — register raw imports before public success

Component: **AddressManager SQL raw-import mutation**.

Type: **Bug fix**.

Dependencies: **[Task 332][t332], [Task 402][t402], and
[Task 467][t467]**.

## Rationale

A committed SQL public-key or Taproot-script import can remain unmonitored
after registration fails. Public success must establish its live watch, not
just persist the import.

## Details

- Wire SQL `ImportPublicKey` and `ImportTaprootScript` into Task 402's
  `WatchAddrsFromTip` path on btcd/bitcoind, including polling, after commit
  and before public success. Leave kvdb and Neutrino behavior unchanged.
- Preserve Task 332's import policy and keep secret encryption outside the
  Store transaction. Successful duplicate imports register idempotently.
- On registration failure, return an error, preserve committed imports, and
  follow Task 402's Manager-wide failure policy. Startup reconstructs watches
  from those facts; add no watch metadata, completion flag, or retry backlog.
- Map an ambiguous SQL commit to `ErrIndeterminateCommit`, with no returned
  success or automatic retry. Add no exact-state readback, secret-provenance
  comparison, or classification-only serialization.
- Start no historical scan or mempool sweep; historical recovery remains an
  explicit caller request.

## Suggested fix

Replace direct `NotifyReceived` calls with Task 402's existing registration
path and use the shared conservative commit-error mapping.

## Verification

- SQLite and PostgreSQL cover commit-before-registration ordering for both
  imports, successful duplicates, and registration failure.
- Failure returns an error, retains committed facts, and follows Task 402's
  shutdown policy. Reuse its startup-reconstruction coverage.
- Ambiguous commit matches `ErrIndeterminateCommit` without success or retry.
- A payment after successful import is observed before mining on
  btcd/bitcoind. Registration starts no history scan; kvdb and Neutrino remain
  unchanged.

## Definition of done

SQL raw imports establish live monitoring before success on btcd/bitcoind,
without delivery persistence, automatic reconciliation, or history recovery.

[t332]: ../archive/tasks/332-watch-only-script-policy.md
[t402]: ../archive/tasks/402-replay-live-watch-obligations.md
[t467]: ../archive/tasks/467-indeterminate-commit-identity.md
