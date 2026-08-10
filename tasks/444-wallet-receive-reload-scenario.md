# Task 444 — cover the wallet receive and reload scenario

Component: **public wallet receive scenario**.

Dependencies: **[Task 415][t415], [Task 429][t429], [Task 430][t430],
[Task 431][t431], [Task 433][t433], and [Task 436][t436]**.

## Rationale

[Issue #1250][issue1250] requires broader scenarios after the per-API contract
layer. The first foundation scenario should prove that one SQL Wallet can
create an account and address, receive confirmed funds, expose consistent
account, UTXO, and transaction views, and preserve them through load and reload.

## Details

- Create and start one SQLite Wallet through public Manager and Controller APIs.
- Create an account and receiving address, fund it from the deterministic miner,
  mine a block, and cross the exact synchronization boundary from Task 433.
- Assert account and address identity, the received UTXO, transaction history,
  confirmation state, and balance as the sum of matching public UTXOs.
- Stop and close the Wallet and Manager, load the same durable Wallet through a
  fresh Manager, and repeat the public assertions without re-creating state.
- Keep this to one receive-and-reload path. Exclude spending, PSBT, reorg,
  bitcoind parity, multi-Wallet behavior, and duplicate negative matrices owned
  by the component contract tests.
- Keep the PR test-only and record exposed production defects separately.

## Suggested fix

Add one scenario registration that composes the completed account, address,
UTXO, and transaction fixtures. Use one-shot wallet assertions after
`AssertWalletSynced` and no direct Store inspection.

## Verification

- Run the focused SQLite receive-and-reload scenario.
- After Task 333 merges, run the same source on PostgreSQL.
- Verify the fresh Manager uses the same durable wallet identity and all
  resources stop on success and failure.

## Definition of done

- One public flow proves account and address creation, receipt, confirmation,
  consistent reads, and durable reload on a real SQL Wallet.
- The scenario reuses contract-layer behavior without duplicating its matrices.
- The PR is test-only and backend-neutral.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t415]: 415-utxo-public-integration-tests.md
[t429]: 429-account-mutation-itests.md
[t430]: 430-account-query-itests.md
[t431]: 431-address-allocation-lookup-itests.md
[t433]: 433-wallet-sync-test-boundary.md
[t436]: 436-transaction-read-itests.md
