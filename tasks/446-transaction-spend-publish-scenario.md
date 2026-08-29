# Task 446 — cover transaction creation and publication

Component: **public transaction spend scenario**.

Type: **Test**.

Dependencies: **[Task 368][t368], [Task 415][t415], [Task 433][t433],
[Task 434][t434], [Task 435][t435], and [Task 436][t436]**.

## Rationale

[Issue #1250][issue1250] requires a realistic transaction creation, signing,
and publication scenario after component contracts exist. No task currently
proves that a funded Wallet on every maintained database can author a valid
spend, pass mempool policy, broadcast it, confirm it, and expose one consistent
public transaction view.

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

Add one deterministic spend scenario that composes the completed funding,
creation, publication, read, and synchronization helpers. Use bounded waits
only for named external mempool and mining events.

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
[t368]: 368-wallet-startup-shutdown.md
[t415]: ../archive/tasks/415-utxo-public-integration-tests.md
[t433]: ../archive/tasks/433-wallet-sync-test-boundary.md
[t434]: ../archive/tasks/434-transaction-creation-itests.md
[t435]: ../archive/tasks/435-transaction-publication-itests.md
[t436]: ../archive/tasks/436-transaction-read-itests.md
