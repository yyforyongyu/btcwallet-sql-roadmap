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
- Compare every exported transaction result field. Assert the public
  `TxStatus` from [Task 339][t339] for pending and published fixtures on all
  three backends; Task 339's focused conversion tests retain coverage of SQL-
  only terminal states that kvdb cannot represent.
- Reopen the same wallet and prove the public transaction view remains stable.
- Task 339 owns the production status model. This task owns its public itest
  assertions and must not encode backend status values directly.
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
- Assert every exported result field, including `TxStatus`, through the
  imported `wallet` package rather than a separate `package wallet_test` file.

## Definition of done

- Both TxReader methods have representative empty, missing, unmined, mined,
  boundary, lifecycle, and cross-method consistency coverage.
- The public transaction view remains correct after durable reopen.
- Shared cases use real kvdb, SQLite, and PostgreSQL Wallets without a backend
  skip branch.
- The public status values from Task 339 are covered without duplicating its
  production implementation or SQL-only conversion cases.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t339]: 339-transaction-visibility.md
