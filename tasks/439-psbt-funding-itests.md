# Task 439 — cover PSBT funding contracts

Component: **PsbtManager funding integration tests**.

Type: **Test**.

Dependencies: **None**.

## Rationale

[Issue #1250][issue1250] calls out `FundPsbt` as a public contract requiring
real-backend happy-path, error, boundary, and durable-state coverage. Funding
joins coin selection, fee and change calculation, input leases, Wallet state,
and packet ownership, but it has no focused public integration-test task.

## Details

- Exercise `FundPsbt` through funded real kvdb, SQLite, and PostgreSQL Wallets.
- Register one shared public contract for all three database backends. Do not
  skip kvdb inside shared cases. Restrict backend-specific code to setup and
  teardown; require identical public assertions and stable error identities.
- Cover automatic selection, caller-selected inputs, with-change and no-change
  results, representative fee boundaries, and deterministic change indexing.
- Cover insufficient funds, locked or foreign inputs, invalid packets and
  outputs, stopped, locked, and watch-only Wallet states where defined.
- Verify selected inputs become unavailable through maintained public UTXO
  views and that observable durable funding state remains consistent after a
  full Wallet and Manager reopen.
- Verify every rejection leaves the caller packet, leases, and address
  allocation state unchanged.
- Do not anticipate behavior owned by Tasks 346, 348, 405, or 406.

## Suggested fix

Add one funding-focused itest source with deterministic UTXOs and packet
fixtures. Establish preconditions and postconditions through PsbtManager,
UtxoManager, and AddressManager public calls rather than Store inspection.

## Verification

- Run the focused `FundPsbt` cases on kvdb, SQLite, and PostgreSQL.
- Verify rejected calls cause no partial mutation and every case releases all
  Wallet, Manager, miner, chain, and backend resources.

## Definition of done

- `FundPsbt` has representative selection, manual-input, change, fee,
  lifecycle, watch-only, and stable-error coverage.
- Publicly observable accepted state survives a durable reopen.
- Shared cases contain no backend skip branch and add no new funding behavior.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
