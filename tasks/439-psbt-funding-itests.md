# Task 439 — cover PSBT funding contracts

Component: **PsbtManager funding integration tests**.

Type: **Test**.

Dependencies: **[Task 348][t348]**.

## Rationale

[Issue #1250][issue1250] calls out `FundPsbt` as a public contract requiring
real-backend happy-path, error, boundary, and durable-state coverage. Deferred
Tasks 348, 405, and 406 add final lease, cleanup, and watched-change guarantees;
this task proves those effects together after the production contracts land.

## Details

- Keep this final-contract suite in Phase 2's verification inventory, with its
  real Task 348 dependency. It does not gate legacy conversion that never
  invokes PSBT funding. Enabling the affected funding path still requires its
  fixes and this coverage; do not weaken assertions or skip missing behavior.
- Exercise `FundPsbt` through funded real kvdb, SQLite, and PostgreSQL Wallets.
- Register one shared public contract for all three database backends. Do not
  skip kvdb inside shared cases. Restrict backend-specific code to setup and
  teardown; require identical public assertions and stable error identities.
- Cover automatic selection, caller-selected inputs, with-change and no-change
  results, representative fee boundaries, and deterministic change indexing.
- Supply a fresh non-zero `FundIntent.LeaseID` for each packet. Prove the
  returned packet preserves [Task 346][t346]'s caller metadata, names the exact
  leased inputs, and returns only after any change is watched through Task
  348's backend-specific path.
- Cover insufficient funds, locked or foreign inputs, invalid packets and
  outputs, and locked or watch-only Wallet states where defined.
- Verify selected inputs become unavailable through maintained public UTXO
  views and that observable durable funding state remains consistent after a
  full Wallet and Manager reopen.
- After success, release every packet outpoint with its caller-owned ID through
  existing `ReleaseOutput` and prove the inputs become available without a new
  batch cleanup API.
- Add one test-only failure gate to the existing integration chain-source
  setup at watched-change registration, after caller-selected funding
  admission. Keep the gate local to setup/teardown, reuse an equivalent
  existing control if present, and add no production fault seam or generic
  fault framework. Inject one stable watch error and prove `FundPsbt` wraps it
  after definitive cleanup, returns no packet, leaves the caller packet
  byte-for-byte unchanged, and leaves no selected input leased under the
  supplied ID. Task 406's focused tests own cleanup-indeterminacy
  classification; do not accept `ErrIndeterminateCommit` as an alternative
  result for this watch-error case. Verify every other rejection leaves the
  caller packet, leases, and address-allocation state unchanged.
- Assert only the final public contract and demonstrated funding regressions.
  Keep private exact-set states, Store inspection, production fault seams, and
  cases already covered by the owning focused tests out of scope.
- Keep the funding cases in this task's planned itest source and place the
  narrow failure gate in the existing integration harness setup rather than a
  standalone helper file.
- Document only non-obvious test contracts, rationales, invariants, and
  consequences. Keep Arrange, Act, and Assert structure clear without
  mandatory section comments that restate the test code.

## Suggested fix

Add one funding-focused itest source with deterministic UTXOs and packet
fixtures. Establish final lease, cleanup, watched-change, and packet
preconditions and postconditions through PsbtManager, UtxoManager, and
AddressManager public calls rather than Store inspection.

## Verification

- Run the focused `FundPsbt` cases on kvdb, SQLite, and PostgreSQL.
- Prove successful packet outpoints remain owned by the supplied lease ID
  until released individually, watched change is visible through public
  Wallet state, and the test-only post-admission watched-change gate returns the
  wrapped injected error with an unchanged caller packet and no lease after
  definitive cleanup.
- Verify rejected calls cause no partial mutation and every case releases all
  Wallet, Manager, miner, chain, and backend resources.

## Definition of done

- Final `FundPsbt` selection, metadata, lease ownership, private failure
  cleanup, watched change, caller release, fee, watch-only, and stable-error
  behavior has representative public coverage.
- Publicly observable accepted state survives a durable reopen.
- Shared cases contain no backend skip branch and add no new funding behavior.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t346]: ../archive/tasks/346-psbt-funding-metadata.md
[t348]: 348-watched-change-delivery.md
