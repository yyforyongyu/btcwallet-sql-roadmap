# Active Tasks

Claim early, own the development, and ship; archive finished tasks and remove
them here. Dependencies set merge order; stacked PRs are encouraged.

## Active Tasks

All open Phase 1 and 2A work, including stacked tasks with open dependencies.

| Task | Description | Type | Phase / track | Claimant | PR |
| --- | --- | --- | --- | --- | --- |
| [Task 470][t470] | Complete lnd Wallet runtime parity | Gap fix | 2A | YY | Preparation [lnd #78][lnd78] → [#79][lnd79] → [#80][lnd80] → runtime parity [#77][lnd77] |

## Upcoming Phase 2 Tasks

All open Phase 2B verification work, including claimed and in-progress tasks.

| Task | Description | Type | Phase / track | Claimant | PR |
| --- | --- | --- | --- | --- | --- |
| [Task 417][t417] | Benchmark startup synchronization | Benchmark | 2B | YY | [#1455][pr1455] |
| [Task 419][t419] | Benchmark address reads | Benchmark | 2B | YY | [#1433][pr1433] |
| [Task 420][t420] | Benchmark transaction reads | Benchmark | 2B | YY | [#1433][pr1433] |
| [Task 449][t449] | Test multi-wallet SQL isolation | Test | 2B | YY | — |
| [Task 471][t471] | Benchmark ListUnspent across backends | Benchmark | 2B | YY | [#1448][pr1448] (merged setup), [#1433][pr1433] (open ListUnspent measurement) |
| [Task 472][t472] | Test isolated wallet rescans | Test | 2B | YY | — |
| [Task 490][t490] | Benchmark GetUtxo | Benchmark | 2B | YY | [#1433][pr1433] |
| [Task 491][t491] | Benchmark GetAccount with balances | Benchmark | 2B | YY | [#1433][pr1433] |
| [Task 492][t492] | Benchmark ListAccounts with balances | Benchmark | 2B | YY | [#1433][pr1433] |

## Claimed Tasks

Parked claims from Phase 3, Phase 4, and Hardening.

| Task | Description | Type | Phase / track | Claimant | PR |
| --- | --- | --- | --- | --- | --- |
| [Task 329][t329] | Commit SQL genesis atomically | Bug fix | 3 | YY | — |
| [Task 397][t397] | Define semantic-account recovery targets | Design | 4 | YY | — |
| [Task 398][t398] | Accept semantic account recovery targets | Gap fix | 4 | YY | — |
| [Task 384][t384] | Route safe signing through the Vault | Refactor | Hardening | Gustavo | [#1522][pr1522], [#1508][pr1508], [#1509][pr1509], [#1510][pr1510] |
| [Task 385][t385] | Isolate UnsafeSigner key extraction | Refactor | Hardening | Gustavo | — |
| [Task 457][t457] | Fence process-local Managers | Feature | Hardening | YY | — |

[t329]: tasks/329-atomic-wallet-account-creation.md
[t397]: tasks/397-semantic-recovery-targets-adr.md
[t398]: tasks/398-semantic-account-recovery-targets.md
[t417]: tasks/417-start-to-synced-benchmark.md
[t419]: tasks/419-address-read-benchmark.md
[t420]: tasks/420-transaction-read-benchmark.md
[t449]: tasks/449-multi-wallet-sql-isolation-scenario.md
[t457]: tasks/457-process-local-manager-lease.md
[t470]: tasks/470-lnd-wallet-port.md
[t471]: tasks/471-three-backend-list-unspent-benchmark.md
[t472]: tasks/472-multi-wallet-rescan-isolation-scenario.md
[t490]: tasks/490-get-utxo-benchmark.md
[t491]: tasks/491-get-account-benchmark.md
[t492]: tasks/492-list-accounts-benchmark.md

[t476]: tasks/476-read-legacy-sql-walletdb-source.md
[t477]: tasks/477-import-legacy-snapshot-postgres.md
[t485]: tasks/485-convert-legacy-sql-wallets-through-manager.md

[lnd77]: https://github.com/yyforyongyu/lnd/pull/77
[lnd78]: https://github.com/yyforyongyu/lnd/pull/78
[lnd79]: https://github.com/yyforyongyu/lnd/pull/79
[lnd80]: https://github.com/yyforyongyu/lnd/pull/80

[pr1434]: https://github.com/btcsuite/btcwallet/pull/1434
[pr1433]: https://github.com/btcsuite/btcwallet/pull/1433
[pr1448]: https://github.com/btcsuite/btcwallet/pull/1448
[pr1455]: https://github.com/btcsuite/btcwallet/pull/1455
[pr1468]: https://github.com/btcsuite/btcwallet/pull/1468
[t384]: tasks/384-vault-safe-signing.md
[t385]: tasks/385-unsafe-signer-key-extraction.md
[pr1508]: https://github.com/btcsuite/btcwallet/pull/1508
[pr1509]: https://github.com/btcsuite/btcwallet/pull/1509
[pr1510]: https://github.com/btcsuite/btcwallet/pull/1510
[pr1521]: https://github.com/btcsuite/btcwallet/pull/1521
[pr1522]: https://github.com/btcsuite/btcwallet/pull/1522
