# Task 446 — cover authored-spend publication and reload

Component: **public transaction spend scenario**.

Type: **Test**.

Dependencies: **[Task 331][t331], [Task 415][t415], [Task 433][t433],
[Task 434][t434], [Task 435][t435], and [Task 436][t436]**.

## Rationale

[Issue #1250][issue1250] requires a public spend scenario. At `sql-wallet`
revision `48e7f4b`, [publisher cases][source] already prove acceptance without
publication and unmined/mined UTXO changes. TxReader also proves durable
history. Their `SignSpend` fixtures bypass `CreateTransaction`, leaving the
authored transaction and change handoff into signing, publication, and reopen
unproved. This task connects that path without another publisher/read matrix.

## Details

- Register the same funded-Wallet scenario for kvdb, SQLite, and PostgreSQL and
  create one deterministic spend through `TxCreator.CreateTransaction` with
  automatic selection and change. Do not skip kvdb inside the shared flow.
- Independently validate the authored transaction and its input scripts, then
  call `CheckMempoolAcceptance` and prove it has not yet published.
- Broadcast through `TxPublisher`, observe the unmined transaction and changed
  UTXO view, mine it, cross the exact sync boundary, and verify confirmation and
  change through TxReader and UtxoManager.
- Stop the Manager, start a fresh Manager over the same database, and repeat
  the final public transaction and UTXO assertions with the returned Wallet.
- Keep this to one successful spend. Exclude PSBT, rejection matrices, reorg,
  multi-Wallet behavior, and Store inspection.

## Suggested fix

Extend suitable existing spend coverage to consume `CreateTransaction` with
automatic selection and change, then verify the signing and durable change
handoff. Reuse acceptance/no-publication, transaction, and UTXO assertions
instead of repeating component matrices. Use bounded waits only for named
external mempool and mining events.

## Verification

- Run the focused transaction spend scenario on kvdb, SQLite, and PostgreSQL.
- Verify acceptance alone does not publish and every resource is joined.

## Definition of done

- One authored transaction is valid, accepted, broadcast, observed unmined,
  confirmed, and durable after reload through maintained public APIs.
- Wallet UTXO and transaction views agree at every stage.
- The shared scenario contains no backend skip branch and does not duplicate
  component error matrices.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t331]: ../archive/tasks/331-manager-wallet-lifecycle.md
[t415]: ../archive/tasks/415-utxo-public-integration-tests.md
[t433]: ../archive/tasks/433-wallet-sync-test-boundary.md
[t434]: ../archive/tasks/434-transaction-creation-itests.md
[t435]: ../archive/tasks/435-transaction-publication-itests.md
[t436]: ../archive/tasks/436-transaction-read-itests.md
[source]: https://github.com/btcsuite/btcwallet/blob/48e7f4b7e0f3808b5022fd817d12c6a4361bb9c1/itest/tx_publisher_test.go
