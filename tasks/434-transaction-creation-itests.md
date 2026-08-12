# Task 434 — cover transaction creation contracts

Component: **TxCreator integration tests**.

Dependencies: **None**.

## Rationale

[Issue #1250][issue1250] assigns Abdullahi the TxCreator integration-test
lane. `CreateTransaction` coordinates public input selection, fee and change
handling, wallet state, and returned transaction metadata, but those contracts
lack focused end-to-end coverage across real kvdb, SQLite, and PostgreSQL
Wallets.

## Details

- Exercise `CreateTransaction` through a real Store-backed Wallet.
- Register the same public contract for kvdb, SQLite, and PostgreSQL. Do not
  skip kvdb inside shared cases. Restrict backend-specific code to setup and
  teardown; require identical public assertions and stable error identities.
- Cover automatic coin selection, caller-selected inputs, transactions with
  and without change, and representative fee and output boundaries.
- Cover insufficient funds, locked or foreign inputs, invalid outputs, a bad
  change source, and stopped, locked, and watch-only Wallet states where the
  maintained API defines them.
- Assert only current public effects. Do not anticipate arithmetic, deferred
  change, watch-delivery, or wrapper behavior owned by Tasks 355–357, 404,
  408, and 425.
- Exclude UTXO contract tests, publication, transaction reads or labels, PSBT
  flows, direct Store assertions, RPC, and unrelated production changes.

## Suggested fix

Add one TxCreator itest source with deterministic funding, manual-input, and
change fixtures. Build expected values independently, invoke the public API,
and inspect only its public result and wallet-visible state.

## Verification

- Run the focused TxCreator cases on kvdb, SQLite, and PostgreSQL.
- Verify every rejection leaves wallet-visible inputs and change state
  unchanged and every path shuts down its Wallet, Manager, miner, and backend.

## Definition of done

- `CreateTransaction` has representative success, input-selection, change,
  lifecycle, and stable-error coverage on real kvdb, SQLite, and PostgreSQL
  Wallets.
- Shared cases contain no backend skip branch.
- The PR does not define future transaction behavior.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
