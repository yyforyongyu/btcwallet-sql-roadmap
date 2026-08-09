# Task 439 — cover PSBT funding contracts

Component: **PsbtManager funding integration tests**.

Dependencies: **None**.

## Rationale

[Issue #1250][issue1250] calls out `FundPsbt` as a public contract requiring
real-SQL happy-path, error, boundary, and durable-state coverage. Funding joins
coin selection, fee and change calculation, input leases, Wallet state, and
packet ownership, but it has no focused public integration-test task.

## Details

- Exercise `FundPsbt` through a funded real SQL-backed Wallet.
- Require SQLite and keep one backend-neutral registration for PostgreSQL after
  Task 333.
- Cover automatic selection, caller-selected inputs, with-change and no-change
  results, representative fee boundaries, and deterministic change indexing.
- Cover insufficient funds, locked or foreign inputs, invalid packets and
  outputs, stopped, locked, and watch-only Wallet states where defined.
- Verify selected inputs become unavailable through maintained public UTXO
  views and that observable durable funding state remains consistent after a
  full Wallet and Manager reopen.
- Verify every rejection leaves the caller packet, leases, and address
  allocation state unchanged.
- Keep the PR test-only. Do not anticipate Tasks 346, 348, 405, or 406, and do
  not fix a production defect inside the test task.

## Suggested fix

Add one funding-focused itest source with deterministic UTXOs and packet
fixtures. Establish preconditions and postconditions through PsbtManager,
UtxoManager, and AddressManager public calls rather than Store inspection.

## Verification

- Run the focused SQLite `FundPsbt` cases.
- After Task 333 merges, run the identical cases on PostgreSQL.
- Verify rejected calls cause no partial mutation and every case releases all
  Wallet, Manager, miner, chain, and backend resources.

## Definition of done

- `FundPsbt` has representative selection, manual-input, change, fee,
  lifecycle, watch-only, and stable-error coverage.
- Publicly observable accepted state survives a durable reopen.
- The backend-neutral PR is test-only and changes no funding behavior.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
