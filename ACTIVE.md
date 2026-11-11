# Active Tasks

Claim early, own the development, and ship; archive finished tasks and remove
them here. Dependencies set merge order; stacked PRs are encouraged.

## Active Tasks

All open Phase 1 and 2A work, including stacked tasks with open dependencies.

| Task | Description | Type | Phase / track | Claimant | PR |
| --- | --- | --- | --- | --- | --- |
| [Task 407][t407] | Authorize PSBT auto-finalization | Bug fix | 2A | Abdullahi | — |
| [Task 423][t423] | Cut over Manager chain notifications | Refactor | 2A | YY | [#1468][pr1468] |
| [Task 470][t470] | Complete lnd Wallet runtime parity | Gap fix | 2A | YY | [lnd #77][lnd77] |

## Upcoming Phase 2 Tasks

All open Phase 2B verification work, including claimed and in-progress tasks.

| Task | Description | Type | Phase / track | Claimant | PR |
| --- | --- | --- | --- | --- | --- |
| [Task 417][t417] | Benchmark startup synchronization | Benchmark | 2B | YY | [#1455][pr1455] |
| [Task 419][t419] | Benchmark address reads | Benchmark | 2B | YY | [#1433][pr1433] |
| [Task 420][t420] | Benchmark transaction reads | Benchmark | 2B | YY | [#1433][pr1433] |
| [Task 432][t432] | Test address imports | Test | 2B | YY | — |
| [Task 438][t438] | Test PSBT decoration and combination | Test | 2B | Abdullahi | [#1480][pr1480] |
| [Task 439][t439] | Test PSBT funding | Test | 2B | YY | [#1472][pr1472] |
| [Task 440][t440] | Test PSBT signing and finalization | Test | 2B | Abdullahi | [#1481][pr1481] |
| [Task 447][t447] | Test the complete PSBT workflow | Test | 2B | Abdullahi | — |
| [Task 449][t449] | Test multi-wallet SQL isolation | Test | 2B | YY | — |
| [Task 451][t451] | Test supported daemon releases | Test | 2B | YY | — |
| [Task 471][t471] | Benchmark ListUnspent across backends | Benchmark | 2B | YY | [#1448][pr1448], [#1433][pr1433] |
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
| [Task 457][t457] | Fence process-local Managers | Feature | Hardening | YY | — |
| [Task 354][t354] | define Vault secret-operation ownership | 4 | Gustavo |
| [Task 383][t383] | resolve output scripts through the Vault | 4 | Gustavo |
| [Task 384][t384] | route safe signing through the Vault | 4 | Gustavo |
| [Task 385][t385] | isolate UnsafeSigner key extraction | 4 | Gustavo |

[t329]: tasks/329-atomic-wallet-account-creation.md
[t397]: tasks/397-semantic-recovery-targets-adr.md
[t398]: tasks/398-semantic-account-recovery-targets.md
[t407]: tasks/407-psbt-finalization-authorization.md
[t417]: tasks/417-start-to-synced-benchmark.md
[t419]: tasks/419-address-read-benchmark.md
[t420]: tasks/420-transaction-read-benchmark.md
[t423]: tasks/423-manager-chain-source-cutover.md
[t432]: tasks/432-address-import-script-itests.md
[t438]: tasks/438-psbt-decoration-combination-itests.md
[t439]: tasks/439-psbt-funding-itests.md
[t440]: tasks/440-psbt-signing-finalization-itests.md
[t447]: tasks/447-psbt-workflow-scenario.md
[t449]: tasks/449-multi-wallet-sql-isolation-scenario.md
[t451]: tasks/451-chain-version-itest-matrix.md
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

[pr1434]: https://github.com/btcsuite/btcwallet/pull/1434
[pr1433]: https://github.com/btcsuite/btcwallet/pull/1433
[pr1448]: https://github.com/btcsuite/btcwallet/pull/1448
[pr1455]: https://github.com/btcsuite/btcwallet/pull/1455
[pr1468]: https://github.com/btcsuite/btcwallet/pull/1468
[pr1472]: https://github.com/btcsuite/btcwallet/pull/1472
[t354]: tasks/354-define-vault-secret-operation-ownership.md
[t383]: tasks/383-resolve-output-scripts-through-the-vault.md
[t384]: tasks/384-route-safe-signing-through-the-vault.md
[t385]: tasks/385-isolate-unsafesigner-key-extraction.md
[pr1481]: https://github.com/btcsuite/btcwallet/pull/1481
[pr1480]: https://github.com/btcsuite/btcwallet/pull/1480
