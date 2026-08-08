# Task 437 — cover transaction label contracts

Component: **TxWriter integration tests**.

Dependencies: **None**.

## Rationale

[Issue #1250][issue1250] assigns Abdullahi the TxWriter integration-test lane.
`LabelTx` mutates durable public transaction metadata, but its validation,
missing-transaction, replacement, lifecycle, and reload contracts lack focused
coverage through a real SQL-backed Wallet.

## Details

- Exercise `LabelTx` through a real Store-backed Wallet.
- Require SQLite and register the same cases for PostgreSQL after
  [Task 333][t333] enables that harness.
- Cover a valid label, replacement according to the maintained public
  contract, empty and boundary label inputs, an unknown transaction, and
  stopped Wallet state.
- Read the label back through maintained public TxReader APIs and reopen the
  same wallet to prove the accepted value is durable.
- Verify rejected writes do not change the prior public label.
- Exclude transaction invalidation, creation, publication, unrelated reader
  coverage, direct Store assertions, RPC, deprecated helpers, and production
  fixes.

## Suggested fix

Add one TxWriter itest source with a deterministic wallet transaction and a
durable reopen helper. Establish preconditions and postconditions through
public TxReader and TxWriter calls rather than database rows.

## Verification

- Run the focused SQLite TxWriter case selection.
- After Task 333 merges, run the same registered cases on PostgreSQL.
- Verify each rejection preserves the previous label and every path releases
  Wallet, Manager, miner, and backend resources.
- Verify the PR changes only integration-test or narrow harness files.

## Definition of done

- `LabelTx` has representative success, replacement, validation, absence,
  lifecycle, and durable-reload integration coverage.
- Accepted labels are observable through public reads after a fresh reopen,
  while rejected labels cause no mutation.
- The cases use a real SQLite Wallet and remain backend-neutral.
- Any exposed production defect is recorded separately.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t333]: 333-postgres-manager-harness.md
