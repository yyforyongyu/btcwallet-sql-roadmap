# Task 439 — cover PSBT funding contracts

Component: **PsbtManager funding integration tests**.

Type: **Test**.

Dependencies: **[Task 356][t356], [Task 406][t406], and [Task 474][t474]**.

## Rationale

[Issue #1250][issue1250] requires real-backend `FundPsbt` coverage. At
`sql-wallet` revision `48e7f4b`, [funding tests][source] prove Task 346's packet
metadata contract. Tasks 405, 406, and 474 still own the final lease,
cleanup, and watched-change guarantees; this suite proves their public effects
together after those production contracts land.

## Details

- Retain final allocation, reservation, cleanup, and watch guarantees through
  Tasks 356, 405/406, and 474. This suite is not a blanket legacy-conversion
  gate; affected funding paths still need their fixes and passing evidence.
- Exercise `FundPsbt` through funded real kvdb, SQLite, and PostgreSQL Wallets.
- Register one shared public contract for all three database backends. Do not
  skip kvdb inside shared cases. Restrict backend-specific code to setup and
  teardown; require identical public assertions and stable error identities.
- Cover automatic selection, caller-selected inputs, with-change and no-change
  results, representative fee boundaries, and deterministic change indexing.
- Supply a fresh non-zero `FundIntent.LeaseID` for each packet. Prove the
  returned packet preserves [Task 346][t346]'s caller metadata, names the exact
  leased inputs, and returns only after any change is watched through Task
  474's existing backend watch path.
- Cover insufficient funds, locked or foreign inputs, invalid packets and
  outputs, and locked or watch-only Wallet states where defined.
- Verify selected inputs become unavailable through maintained public UTXO
  views and that observable durable funding state remains consistent after a
  full Wallet and Manager reopen.
- After success, release every packet outpoint with its caller-owned ID through
  existing `ReleaseOutput` and prove the inputs become available without a new
  batch cleanup API.
- Exercise one watched-change failure after caller-selected funding.
  Require the wrapped injected watch error, no returned packet, an
  unchanged caller packet, and no attempt lease under the supplied ID after
  any required definitive cleanup. Do not prescribe watch/reservation order:
  if registration precedes reservation, no lease is acquired. Task 406 owns
  acquired-set cleanup and indeterminate-cleanup classification;
  `ErrIndeterminateCommit` is not an alternative pass for this definitive case.
- Retain committed change-address facts after post-commit watch failure so
  Task 402 can reconstruct readiness. Pre-mutation rejection leaves packet,
  leases, and address allocation unchanged; later failures do not imply that
  committed address allocation rolled back.
- Reuse Task 346's metadata matrix and Task 406's private cleanup tests. Keep
  representative public assertions here, with no Store inspection, production
  fault seam, generic failure framework, or duplicate private-state matrix.

## Suggested fix

Reuse deterministic UTXOs and packets, observing outcomes through
PsbtManager, UtxoManager, and AddressManager. An existing test-only chain
control, or the smallest gate around watched-change registration in the real
chain fixture, can prove the watch failure. Keep the control in test
setup and teardown without prescribing a new file or production hook.

## Verification

- Run the focused `FundPsbt` cases on kvdb, SQLite, and PostgreSQL.
- Prove successful packet outpoints remain owned by the supplied lease ID
  until released individually or expired, watched change is visible through
  public Wallet state, and the test-only watched-change gate returns the
  wrapped injected error with an unchanged caller packet and no lease after
  definitive cleanup.
- Verify pre-mutation rejection has no mutation, post-commit watch failure
  retains its committed address facts, and every case releases all Wallet,
  Manager, miner, chain, and backend resources.

## Definition of done

- Final `FundPsbt` selection, metadata, lease ownership, private failure
  cleanup, watched change, caller release, fee, watch-only, and stable-error
  behavior has representative public coverage.
- Publicly observable accepted state survives a durable reopen.
- Shared cases contain no backend skip branch and add no new funding behavior.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t346]: ../archive/tasks/346-psbt-funding-metadata.md
[t356]: 356-defer-change-script-allocation.md
[t406]: 406-psbt-funding-lease-release.md
[t474]: 474-authored-change-watch.md
[source]: https://github.com/btcsuite/btcwallet/blob/48e7f4b7e0f3808b5022fd817d12c6a4361bb9c1/wallet/psbt_manager_test.go#L5061
