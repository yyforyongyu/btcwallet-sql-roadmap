# Task 436 — cover transaction read contracts

Component: **TxReader integration tests**.

Type: **Test**.

Dependencies: **None**.

## Rationale

[Issue #1250][issue1250] assigns Abdullahi the TxReader integration-test lane.
`GetTx` and `ListTxns` expose the same durable transaction view, but their
public lookup, ordering, filtering, lifecycle, and reload behavior lack focused
coverage through real kvdb, SQLite, and PostgreSQL Wallets.

## Details

- Exercise `GetTx` and `ListTxns` through a real Store-backed Wallet.
- Register the same public contract for kvdb, SQLite, and PostgreSQL. Do not
  skip kvdb inside shared cases. Restrict backend-specific code to setup and
  teardown; require identical public assertions and stable error identities.
- Cover empty history, an unknown transaction, unmined and mined transactions,
  confirmations, supported list boundaries, and agreement between point and
  list results.
- Reopen the same wallet and prove the public transaction view remains stable.
- Test the maintained contract available when the task is claimed. Do not
  define the future status model owned by [Task 339][t339] or hide a defect by
  encoding backend internals.
- Exclude creation, publication, labels, direct Store reads, RPC, deprecated
  transaction helpers, and unrelated production changes.

## Suggested fix

Add one TxReader itest source with deterministic unmined and mined history.
Derive expected transaction identities and chain facts from the fixture, then
compare the two public readers before and after a full durable reopen.

## Verification

- Run the focused TxReader cases on kvdb, SQLite, and PostgreSQL.
- Verify list boundaries are deterministic, point and list results agree, and
  every path releases Wallet, Manager, miner, and backend resources.

## Definition of done

- Both TxReader methods have representative empty, missing, unmined, mined,
  boundary, lifecycle, and cross-method consistency coverage.
- The public transaction view remains correct after durable reopen.
- Shared cases use real kvdb, SQLite, and PostgreSQL Wallets without a backend
  skip branch.
- Future status-contract work remains in Task 339.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t339]: 339-transaction-visibility.md
