# Active Tasks

Claim early, own the development, and ship; archive finished tasks and remove
them here. Dependencies set merge order; stacked PRs are encouraged.

## Active Tasks

All open Phase 1 and 2A work, including stacked tasks with open dependencies.

| Task | Description | Type | Phase / track | Claimant | PR |
| --- | --- | --- | --- | --- | --- |
| [Task 487][t487] | Compile lnd with the maintained Wallet | Gap fix | 1 | YY | — |
| [Task 356][t356] | Defer change-script allocation | Bug fix | 2A | Gustavo | [#1453][pr1453] |
| [Task 370][t370] | Fan out ordered Manager chain events | Refactor | 2A | YY | [#1449][pr1449] |
| [Task 372][t372] | Coordinate multi-Wallet rollback | Bug fix | 2A | YY | [#1435][pr1435] |
| [Task 406][t406] | Clean up failed PSBT funding leases | Gap fix | 2A | Gustavo | — |
| [Task 407][t407] | Authorize PSBT auto-finalization | Bug fix | 2A | Abdullahi | — |
| [Task 410][t410] | Coordinate transaction graph mutations | Bug fix | 2A | YY | [#1442][pr1442] |
| [Task 423][t423] | Cut over Manager chain notifications | Refactor | 2A | YY | — |
| [Task 469][t469] | Sign PSBT inputs with effective keys | Bug fix | 2A | Abdullahi | [#1437][pr1437] |
| [Task 470][t470] | Complete lnd Wallet runtime parity | Gap fix | 2A | YY | — |
| [Task 482][t482] | Fund imported inputs without known origin | Gap fix | 2A | — | — |

## Upcoming Phase 2 Tasks

All open Phase 2B verification work, including claimed and in-progress tasks.

| Task | Description | Type | Phase / track | Claimant | PR |
| --- | --- | --- | --- | --- | --- |
| [Task 369][t369] | Test repeated unlocks | Test | 2B | — | — |
| [Task 417][t417] | Benchmark startup synchronization | Benchmark | 2B | YY | [#1455][pr1455] |
| [Task 419][t419] | Benchmark address reads | Benchmark | 2B | YY | [#1433][pr1433] |
| [Task 420][t420] | Benchmark transaction reads | Benchmark | 2B | YY | [#1433][pr1433] |
| [Task 428][t428] | Test resync and rescan | Test | 2B | Gustavo | — |
| [Task 432][t432] | Test address imports | Test | 2B | Gustavo | — |
| [Task 438][t438] | Test PSBT decoration and combination | Test | 2B | — | — |
| [Task 439][t439] | Test PSBT funding | Test | 2B | — | — |
| [Task 440][t440] | Test PSBT signing and finalization | Test | 2B | — | — |
| [Task 444][t444] | Test receive and reopen | Test | 2B | Abdullahi | [#1457][pr1457] |
| [Task 445][t445] | Test account signing identity | Test | 2B | — | — |
| [Task 447][t447] | Test the complete PSBT workflow | Test | 2B | — | — |
| [Task 448][t448] | Test sync and rollback | Test | 2B | Abdullahi | [#1458][pr1458] |
| [Task 449][t449] | Test multi-wallet SQL isolation | Test | 2B | — | — |
| [Task 450][t450] | Compare wallet state with bitcoind | Test | 2B | — | — |
| [Task 451][t451] | Test supported daemon releases | Test | 2B | — | — |
| [Task 462][t462] | Test account mutation errors | Test | 2B | Abdullahi | — |
| [Task 463][t463] | Test path-bound UTXO identity | Test | 2B | — | — |
| [Task 471][t471] | Benchmark ListUnspent across backends | Benchmark | 2B | YY | [#1448][pr1448], [#1433][pr1433] |
| [Task 472][t472] | Test isolated wallet rescans | Test | 2B | — | — |
| [Task 490][t490] | Benchmark GetUtxo | Benchmark | 2B | YY | [#1433][pr1433] |
| [Task 491][t491] | Benchmark GetAccount with balances | Benchmark | 2B | YY | [#1433][pr1433] |
| [Task 492][t492] | Benchmark ListAccounts with balances | Benchmark | 2B | YY | [#1433][pr1433] |

## Claimed Tasks

Parked claims from Phase 3, Phase 4, and Hardening.

| Task | Description | Type | Phase / track | Claimant | PR |
| --- | --- | --- | --- | --- | --- |
| [Task 329][t329] | Commit SQL genesis atomically | Bug fix | 3 | YY | — |
| [Task 350][t350] | Use native PostgreSQL driver | Refactor | 4 | YY | [#1321][pr1321] |
| [Task 397][t397] | Define semantic-account recovery targets | Design | 4 | YY | — |
| [Task 398][t398] | Accept semantic account recovery targets | Gap fix | 4 | YY | — |
| [Task 457][t457] | Fence process-local Managers | Feature | Hardening | YY | — |

[pr1321]: https://github.com/btcsuite/btcwallet/pull/1321
[t329]: tasks/329-atomic-wallet-account-creation.md
[t350]: tasks/350-native-pgx-postgres.md
[t356]: tasks/356-defer-change-script-allocation.md
[t369]: tasks/369-repeated-wallet-unlock.md
[t370]: tasks/370-chain-source-delivery.md
[t372]: tasks/372-multi-wallet-rollback.md
[t397]: tasks/397-semantic-recovery-targets-adr.md
[t398]: tasks/398-semantic-account-recovery-targets.md
[t406]: tasks/406-psbt-funding-lease-release.md
[t407]: tasks/407-psbt-finalization-authorization.md
[t410]: tasks/410-transaction-graph-admission.md
[t417]: tasks/417-start-to-synced-benchmark.md
[t419]: tasks/419-address-read-benchmark.md
[t420]: tasks/420-transaction-read-benchmark.md
[t423]: tasks/423-manager-chain-source-cutover.md
[t428]: tasks/428-controller-sync-info-itests.md
[t432]: tasks/432-address-import-script-itests.md
[t438]: tasks/438-psbt-decoration-combination-itests.md
[t439]: tasks/439-psbt-funding-itests.md
[t440]: tasks/440-psbt-signing-finalization-itests.md
[t444]: tasks/444-wallet-receive-reload-scenario.md
[t445]: tasks/445-account-signing-scenario.md
[t447]: tasks/447-psbt-workflow-scenario.md
[t448]: tasks/448-sync-rollback-scenario.md
[t449]: tasks/449-multi-wallet-sql-isolation-scenario.md
[t450]: tasks/450-bitcoind-compatibility-scenario.md
[t451]: tasks/451-chain-version-itest-matrix.md
[t457]: tasks/457-process-local-manager-lease.md
[t462]: tasks/462-account-mutation-error-itests.md
[t463]: tasks/463-path-bound-utxo-identity-itests.md
[t469]: tasks/469-sign-psbt-with-effective-keys.md
[t470]: tasks/470-lnd-wallet-port.md
[t471]: tasks/471-three-backend-list-unspent-benchmark.md
[t472]: tasks/472-multi-wallet-rescan-isolation-scenario.md
[t490]: tasks/490-get-utxo-benchmark.md
[t491]: tasks/491-get-account-benchmark.md
[t492]: tasks/492-list-accounts-benchmark.md

[t476]: tasks/476-read-legacy-sql-walletdb-source.md
[t477]: tasks/477-import-legacy-snapshot-postgres.md
[t482]: tasks/482-fund-imported-inputs-without-origin.md
[t485]: tasks/485-convert-legacy-sql-wallets-through-manager.md

[t487]: tasks/487-compile-lnd-with-managed-wallet.md

[pr1435]: https://github.com/btcsuite/btcwallet/pull/1435
[pr1434]: https://github.com/btcsuite/btcwallet/pull/1434
[pr1433]: https://github.com/btcsuite/btcwallet/pull/1433
[pr1437]: https://github.com/btcsuite/btcwallet/pull/1437
[pr1442]: https://github.com/btcsuite/btcwallet/pull/1442
[pr1448]: https://github.com/btcsuite/btcwallet/pull/1448
[pr1449]: https://github.com/btcsuite/btcwallet/pull/1449
[pr1453]: https://github.com/btcsuite/btcwallet/pull/1453
[pr1455]: https://github.com/btcsuite/btcwallet/pull/1455
[pr1458]: https://github.com/btcsuite/btcwallet/pull/1458
[pr1457]: https://github.com/btcsuite/btcwallet/pull/1457
