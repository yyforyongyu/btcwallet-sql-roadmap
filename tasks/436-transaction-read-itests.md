# Task 436 — cover transaction read contracts

Component: **TxReader integration tests**.

Dependencies: **None**.

## Rationale

[Issue #1250][issue1250] assigns Abdullahi the TxReader integration-test lane.
`GetTx` and `ListTxns` expose the same durable transaction view, but their
public lookup, ordering, filtering, lifecycle, and reload behavior lack focused
coverage through a real SQL-backed Wallet.

## Details

- Exercise `GetTx` and `ListTxns` through a real Store-backed Wallet.
- Require SQLite and register the same cases for PostgreSQL after
  [Task 333][t333] enables that harness.
- Cover empty history, an unknown transaction, unmined and mined transactions,
  confirmations, supported list boundaries, and agreement between point and
  list results.
- Reopen the same wallet and prove the public transaction view remains stable.
- Test the maintained contract available when the task is claimed. Do not
  define the future status model owned by [Task 339][t339] or hide a defect by
  encoding backend internals.
- Exclude creation, publication, labels, direct Store reads, RPC, deprecated
  transaction helpers, and production fixes.

## Suggested fix

Add one TxReader itest source with deterministic unmined and mined history.
Derive expected transaction identities and chain facts from the fixture, then
compare the two public readers before and after a full durable reopen.

## Verification

- Run the focused SQLite TxReader case selection.
- After Task 333 merges, run the identical cases on PostgreSQL.
- Verify list boundaries are deterministic, point and list results agree, and
  every path releases Wallet, Manager, miner, and backend resources.
- Verify the PR remains test-only.

## Definition of done

- Both TxReader methods have representative empty, missing, unmined, mined,
  boundary, lifecycle, and cross-method consistency coverage.
- The public transaction view remains correct after durable reopen.
- The cases use a real SQLite Wallet and remain backend-neutral.
- Any status-contract gap or production defect remains a separate task.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t333]: 333-postgres-manager-harness.md
[t339]: 339-transaction-visibility.md
