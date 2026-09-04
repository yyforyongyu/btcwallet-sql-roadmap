# Task 444 — cover the wallet receive and reload scenario

Component: **public wallet receive scenario**.

Type: **Test**.

Dependencies: **[Task 331][t331], [Task 415][t415], [Task 429][t429],
[Task 430][t430], [Task 431][t431], [Task 433][t433], and
[Task 436][t436]**.

## Rationale

[Issue #1250][issue1250] requires broader scenarios after the per-API contract
layer. The first foundation scenario should prove that one Wallet on each
maintained database can create an account and address, receive confirmed funds,
expose consistent account, UTXO, and transaction views, and preserve them
through a fresh Manager startup over the same database.

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

Add one scenario registration that composes the completed account, address,
UTXO, and transaction fixtures. Use one-shot wallet assertions after
`AssertWalletSynced` and no direct Store inspection.

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
[t331]: 331-manager-wallet-lifecycle.md
[t415]: ../archive/tasks/415-utxo-public-integration-tests.md
[t429]: ../archive/tasks/429-account-mutation-itests.md
[t430]: 430-account-query-itests.md
[t431]: 431-address-allocation-lookup-itests.md
[t433]: ../archive/tasks/433-wallet-sync-test-boundary.md
[t436]: ../archive/tasks/436-transaction-read-itests.md
