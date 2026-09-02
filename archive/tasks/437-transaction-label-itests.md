# Task 437 — cover transaction label contracts

Component: **TxWriter integration tests**.

Type: **Test**.

Dependencies: **None**.

## Rationale

[Issue #1250][issue1250] assigns Abdullahi the TxWriter integration-test lane.
`LabelTx` mutates durable public transaction metadata, but its validation,
missing-transaction, replacement, and reload contracts lack focused
coverage through real kvdb, SQLite, and PostgreSQL Wallets.

## Details

- Exercise `LabelTx` through a real Store-backed Wallet.
- Register the same public contract for kvdb, SQLite, and PostgreSQL. Do not
  skip kvdb inside shared cases. Restrict backend-specific code to setup and
  teardown; require identical public assertions and stable error identities.
- Cover a valid label, replacement according to the maintained public
  contract, empty and boundary label inputs, and an unknown transaction.
- Read the label back through maintained public TxReader APIs and reopen the
  same wallet to prove the accepted value is durable.
- Verify rejected writes do not change the prior public label.
- Exclude transaction invalidation, creation, publication, unrelated reader
  coverage, direct Store assertions, RPC, deprecated helpers, and production
  changes unrelated to the required cases.

## Suggested fix

Add one TxWriter itest source with a deterministic wallet transaction and a
durable reopen helper. Establish preconditions and postconditions through
public TxReader and TxWriter calls rather than database rows.

## Verification

- Run the focused TxWriter cases on kvdb, SQLite, and PostgreSQL.
- Verify each rejection preserves the previous label and every path releases
  Wallet, Manager, miner, and backend resources.

## Definition of done

- `LabelTx` has representative success, replacement, validation, absence, and
  durable-reload integration coverage.
- Accepted labels are observable through public reads after a fresh reopen,
  while rejected labels cause no mutation.
- Shared cases use real kvdb, SQLite, and PostgreSQL Wallets without a backend
  skip branch.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
