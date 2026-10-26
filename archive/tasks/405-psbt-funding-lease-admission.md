# Task 405 — reserve PSBT funding inputs atomically

Component: **PsbtManager and UTXO Store funding admission**.

Type: **Gap fix**.

Dependencies: **[Task 346][t346], [Task 404][t404], and [Task 467][t467]**.

## Rationale

`FundIntent` and `handleFundPsbt` at `sql-wallet` revision `2ae306dea8c5`
have no funding lease identity or selected-set reservation. Concurrent funding
attempts can return packets spending the same inputs.

## Details

- Add one required `LeaseID wtxmgr.LockID` field to existing `FundIntent`.
  Reject zero before Store access; callers supply a fresh, never-reused ID
  for every funding attempt.
- Construct the packet first, then atomically recheck availability and reserve
  its complete automatic or caller-selected input set under that ID. A
  definite conflict acquires no partial set. Keep selection, authoring, and
  chain calls outside the reservation transaction.
- Use a fixed, package-private ten-minute duration from the Store clock.
  Reuse existing lease expiry: expired outputs become available without an
  expiry worker. Callers may release early through existing `ReleaseOutput`.
- Keep the packet and change-index result. On success, the attempt owns valid,
  unexpired leases for every selected input; expose no new result or duration.
- On ambiguous reservation commit, return `ErrIndeterminateCommit` and no
  packet, without readback or retry. Do not claim no leases exist.
  [Task 406][t406] attempts ownership-checked cleanup even in this case.
- Keep change-allocation timing in Task 356 and watch registration in
  [Task 474][t474]. Neither is a prerequisite for reserving an authored set;
  Task 406 owns cleanup after subsequent failure.
- Cover one Wallet and independent PostgreSQL Store transactions without
  claiming multi-Manager or cross-process support.
- Add no attempt manifest, fencing, or general UtxoManager lease redesign.

## Suggested fix

Add lease identity to the existing intent and a narrow selected-set
reservation transaction. Reuse existing lease availability and expiry rules,
without moving the whole funding operation into a database transaction.

## Verification

- Zero ID fails before Store access. Invalid authoring leaves no leases.
- On kvdb, SQLite, and PostgreSQL, a successful attempt owns the complete set;
  conflicting attempts cannot share an input or acquire a partial set.
- Verify the ten-minute expiration, early release, and availability after
  expiry using controlled time rather than sleeps.
- Ambiguous reservation returns no packet and the shared error identity,
  without retry or a false claim of rollback.
- [Task 439][t439] owns final public funding and caller-release itests.

## Definition of done

Funding reserves its complete input set atomically under a fresh caller ID,
with bounded lease lifetime and truthful uncertain results. No new public
cleanup method, reservation result, or reconciliation system is added.

[t346]: ../archive/tasks/346-psbt-funding-metadata.md
[t404]: ../archive/tasks/404-transaction-input-amounts.md
[t406]: 406-psbt-funding-lease-release.md
[t439]: 439-psbt-funding-itests.md
[t467]: ../archive/tasks/467-indeterminate-commit-identity.md
[t474]: ../archive/tasks/474-authored-change-watch.md
