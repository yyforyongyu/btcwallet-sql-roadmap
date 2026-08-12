# Task 448 — cover synchronization and rollback

Component: **public Wallet reorganization scenario**.

Dependencies: **[Task 415][t415], [Task 428][t428], [Task 433][t433], and
[Task 436][t436]**.

## Rationale

[Issue #1250][issue1250] requires a broader sync-and-rollback scenario after
the public API contract layer. Focused Controller and sync-boundary tests do not
prove that transaction, UTXO, confirmation, and tip views move together through
a real chain reorganization and remain coherent after reload.

## Details

- Register the same receive-and-confirm flow for kvdb, SQLite, and PostgreSQL
  on a short deterministic chain, then assert the exact synchronized tip and
  public views. Do not skip kvdb inside the shared flow.
- Create a shallow competing branch that removes the transaction's confirming
  block and let the maintained chain backend deliver the reorganization.
- Cross only Task 433's exact synchronization boundary, then assert the public
  transaction status, confirmations, UTXO state, and synced tip once.
- Extend the winning branch, verify the next stable state, reload the Wallet,
  and repeat the final public assertions.
- Keep this to one Wallet and one bounded shallow reorg. Exclude multi-Wallet
  coordination, resumable rescans, bitcoind parity, randomized chains, and
  direct Store assertions.

## Suggested fix

Add one reorganization scenario using existing miner branch controls and the
completed sync helper. Use purpose-named waits only for backend events and no
generic polling over wallet-owned state.

## Verification

- Run the focused sync-and-rollback scenario on kvdb, SQLite, and PostgreSQL
  with each supported chain backend.
- Verify losing and winning chain state never appear mixed after
  synchronization.

## Definition of done

- Public tip, transaction, confirmation, and UTXO views remain mutually
  consistent before and after one reorg and after durable reload.
- The scenario obeys Task 433's one-shot wallet assertion boundary.
- The shared scenario contains no backend skip branch and no recovery redesign.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t415]: ../archive/tasks/415-utxo-public-integration-tests.md
[t428]: 428-controller-sync-info-itests.md
[t433]: 433-wallet-sync-test-boundary.md
[t436]: 436-transaction-read-itests.md
