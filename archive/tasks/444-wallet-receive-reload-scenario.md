# Task 444 — cover new-account receipt and reload

Component: **public wallet receive scenario**.

Type: **Test**.

Dependencies: **[Task 331][t331], [Task 415][t415], [Task 429][t429],
[Task 433][t433], and [Task 436][t436]**.

## Rationale

[Issue #1250][issue1250] requires broader public scenarios. At `sql-wallet`
revision `48e7f4b`, [TxReader cases][source] already cover received transaction
fields and complete history across reopen; account mutation and UTXO cases
also exist. The remaining outcome joins those public views for one newly
created account and receiving address through receipt and fresh Manager
startup. It does not need another component field or error matrix.

## Details

- Register the same scenario for kvdb, SQLite, and PostgreSQL through public
  Manager and Controller APIs, without skipping kvdb inside the shared flow.
- Create an account and receiving address, fund it from the deterministic miner,
  mine a block, and cross the exact synchronization boundary from Task 433.
- Assert account and address identity, the received UTXO, transaction history,
  confirmation state, and balance as the sum of matching public UTXOs.
- Stop the Manager, construct and start a fresh Manager over the same database,
  select the Wallet from its returned active set, and repeat the public
  assertions without re-creating state.
- Keep this to one receive-and-reload path. Exclude spending, PSBT, reorg,
  bitcoind parity, multi-Wallet behavior, and duplicate negative matrices owned
  by the component contract tests.

## Suggested fix

Extend a suitable existing receive/reopen case with the new-account fixture
and the missing account/address/UTXO/transaction identity joins. Reuse existing
checks instead of adding a parallel receive suite. Do not wait for Tasks 430
or 431's test completion; use one-shot assertions after `AssertWalletSynced`
and no direct Store inspection.

## Verification

- Run the focused receive-and-reload scenario on kvdb, SQLite, and PostgreSQL.
- Verify the fresh Manager uses the same durable wallet identity and all
  resources stop on success and failure.

## Definition of done

- One public flow proves account and address creation, receipt, confirmation,
  consistent reads, and durable reload on real kvdb, SQLite, and PostgreSQL
  Wallets.
- The scenario reuses contract-layer behavior without duplicating its matrices.
- The shared scenario contains no backend skip branch.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t331]: ../archive/tasks/331-manager-wallet-lifecycle.md
[t415]: ../archive/tasks/415-utxo-public-integration-tests.md
[t429]: ../archive/tasks/429-account-mutation-itests.md
[t430]: ../archive/tasks/430-account-query-itests.md
[t431]: ../archive/tasks/431-address-allocation-lookup-itests.md
[t433]: ../archive/tasks/433-wallet-sync-test-boundary.md
[t436]: ../archive/tasks/436-transaction-read-itests.md
[source]: https://github.com/btcsuite/btcwallet/blob/48e7f4b7e0f3808b5022fd817d12c6a4361bb9c1/itest/tx_reader_test.go
