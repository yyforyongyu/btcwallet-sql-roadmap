# Active Tasks

Claim early, own the development, and ship; archive finished tasks and remove
them here. Dependencies set merge order; stacked PRs are encouraged.

## Active Tasks

All open Phase 1 work, including stacked tasks with open dependencies.

| Task | Description | Type | Phase / track | Claimant | PR |
| --- | --- | --- | --- | --- | --- |
| [Task 388][t388] | Reject duplicate account XPubs | Bug fix | 1 | YY | [#1394][pr1394] |
| [Task 391][t391] | Create exact path-bound XPub accounts | Gap fix | 1 | YY | [#1408][pr1408] |
| [Task 403][t403] | Watch raw imports before success | Bug fix | 1 | YY | [#1410][pr1410] |
| [Task 431][t431] | Test address allocation | Test | 1 | Gustavo | [#1407][pr1407] |
| [Task 475][t475] | Deliver wallet transaction subscriptions | Gap fix | 1 | Abdullahi | [#1401][pr1401] |
| [Task 478][t478] | Retain spend-confirmation leases | Gap fix | 1 | Abdullahi | — |
| [Task 479][t479] | Mix imported XPub and signing accounts | Gap fix | 1 | YY | [#1411][pr1411] |
| [Task 480][t480] | Own raw imports in one signing wallet | Gap fix | 1 | YY | — |
| [Task 481][t481] | Preserve imported account address schema | Bug fix | 1 | YY | [#1409][pr1409] |
| [Task 486][t486] | Consolidate initial SQL schema | Migration | 1 | YY | — |
| [Task 487][t487] | Compile lnd with the maintained Wallet | Gap fix | 1 | YY | — |

## Upcoming Phase 2 Tasks

All open Phase 2A and 2B work, including claimed and in-progress tasks.

| Task | Description | Type | Phase / track | Claimant | PR |
| --- | --- | --- | --- | --- | --- |
| [Task 330][t330] | Reject duplicate wallet creation | Bug fix | 2A | YY | — |
| [Task 338][t338] | Latch passphrase results | Bug fix | 2A | — | — |
| [Task 347][t347] | Authorize existing PSBT signatures | Bug fix | 2A | Abdullahi | [#1400][pr1400] |
| [Task 356][t356] | Defer change-script allocation | Bug fix | 2A | Gustavo | — |
| [Task 365][t365] | Validate wallet genesis key identities | Bug fix | 2A | YY | — |
| [Task 367][t367] | Bound SQL genesis secrets | Bug fix | 2A | YY | — |
| [Task 370][t370] | Fan out ordered Manager chain events | Refactor | 2A | YY | — |
| [Task 372][t372] | Coordinate multi-Wallet rollback | Bug fix | 2A | YY | — |
| [Task 389][t389] | Report uncertain account commits | Bug fix | 2A | — | — |
| [Task 405][t405] | Reserve PSBT funding inputs | Gap fix | 2A | Gustavo | — |
| [Task 406][t406] | Clean up failed PSBT funding leases | Gap fix | 2A | Gustavo | — |
| [Task 407][t407] | Authorize PSBT auto-finalization | Bug fix | 2A | Abdullahi | — |
| [Task 410][t410] | Coordinate transaction graph mutations | Bug fix | 2A | — | — |
| [Task 423][t423] | Cut over Manager chain notifications | Refactor | 2A | YY | — |
| [Task 424][t424] | Define transaction integrity ownership | Design | 2A | YY | [#1343][pr1343] |
| [Task 454][t454] | Make PostgreSQL isolation explicit | Bug fix | 2A | YY | — |
| [Task 469][t469] | Sign PSBT inputs with effective keys | Bug fix | 2A | Abdullahi | — |
| [Task 470][t470] | Complete lnd Wallet runtime parity | Gap fix | 2A | YY | — |
| [Task 474][t474] | Watch authored change before return | Bug fix | 2A | Gustavo | — |
| [Task 482][t482] | Fund imported inputs without known origin | Gap fix | 2A | — | — |
| [Task 483][t483] | Preserve labels on chain replay | Bug fix | 2A | — | — |
| [Task 484][t484] | Align authoring confirmation height | Bug fix | 2A | — | — |
| [Task 369][t369] | Test repeated unlocks | Test | 2B | — | — |
| [Task 417][t417] | Benchmark startup synchronization | Benchmark | 2B | — | — |
| [Task 419][t419] | Benchmark address reads | Benchmark | 2B | — | — |
| [Task 420][t420] | Benchmark transaction reads | Benchmark | 2B | — | — |
| [Task 428][t428] | Test resync and rescan | Test | 2B | Gustavo | — |
| [Task 432][t432] | Test address imports | Test | 2B | Gustavo | — |
| [Task 438][t438] | Test PSBT decoration and combination | Test | 2B | — | — |
| [Task 439][t439] | Test PSBT funding | Test | 2B | — | — |
| [Task 440][t440] | Test PSBT signing and finalization | Test | 2B | — | — |
| [Task 444][t444] | Test receive and reopen | Test | 2B | — | — |
| [Task 445][t445] | Test account signing identity | Test | 2B | — | — |
| [Task 446][t446] | Test spend and publish | Test | 2B | — | — |
| [Task 447][t447] | Test the complete PSBT workflow | Test | 2B | — | — |
| [Task 448][t448] | Test sync and rollback | Test | 2B | — | — |
| [Task 449][t449] | Test multi-wallet SQL isolation | Test | 2B | — | — |
| [Task 450][t450] | Compare wallet state with bitcoind | Test | 2B | — | — |
| [Task 451][t451] | Test supported daemon releases | Test | 2B | — | — |
| [Task 462][t462] | Test account mutation errors | Test | 2B | — | — |
| [Task 463][t463] | Test path-bound UTXO identity | Test | 2B | — | — |
| [Task 464][t464] | Test NoChainSync account queries | Test | 2B | — | — |
| [Task 466][t466] | Verify final oldest-unused allocation | Test | 2B | — | — |
| [Task 471][t471] | Benchmark ListUnspent across backends | Benchmark | 2B | — | — |
| [Task 472][t472] | Test isolated wallet rescans | Test | 2B | — | — |

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
[pr1343]: https://github.com/btcsuite/btcwallet/pull/1343
[t329]: tasks/329-atomic-wallet-account-creation.md
[t330]: tasks/330-duplicate-wallet-create.md
[t338]: tasks/338-passphrase-result.md
[t347]: tasks/347-psbt-signing-authorization.md
[t350]: tasks/350-native-pgx-postgres.md
[t356]: tasks/356-defer-change-script-allocation.md
[t365]: tasks/365-validate-wallet-genesis-keys.md
[t367]: tasks/367-bound-sql-genesis-secrets.md
[t369]: tasks/369-repeated-wallet-unlock.md
[t370]: tasks/370-chain-source-delivery.md
[t372]: tasks/372-multi-wallet-rollback.md
[t388]: tasks/388-runtime-account-xpub-collisions.md
[t389]: tasks/389-runtime-account-commit-reconciliation.md
[t391]: tasks/391-path-bound-watch-only-xpub-accounts.md
[t397]: tasks/397-semantic-recovery-targets-adr.md
[t398]: tasks/398-semantic-account-recovery-targets.md
[t403]: tasks/403-raw-import-watch-obligations.md
[t405]: tasks/405-psbt-funding-lease-admission.md
[t406]: tasks/406-psbt-funding-lease-release.md
[t407]: tasks/407-psbt-finalization-authorization.md
[t410]: tasks/410-transaction-graph-admission.md
[t417]: tasks/417-start-to-synced-benchmark.md
[t419]: tasks/419-address-read-benchmark.md
[t420]: tasks/420-transaction-read-benchmark.md
[t423]: tasks/423-manager-chain-source-cutover.md
[t424]: tasks/424-transaction-utxo-integrity-adr.md
[t428]: tasks/428-controller-sync-info-itests.md
[t431]: tasks/431-address-allocation-lookup-itests.md
[t432]: tasks/432-address-import-script-itests.md
[t438]: tasks/438-psbt-decoration-combination-itests.md
[t439]: tasks/439-psbt-funding-itests.md
[t440]: tasks/440-psbt-signing-finalization-itests.md
[t444]: tasks/444-wallet-receive-reload-scenario.md
[t445]: tasks/445-account-signing-scenario.md
[t446]: tasks/446-transaction-spend-publish-scenario.md
[t447]: tasks/447-psbt-workflow-scenario.md
[t448]: tasks/448-sync-rollback-scenario.md
[t449]: tasks/449-multi-wallet-sql-isolation-scenario.md
[t450]: tasks/450-bitcoind-compatibility-scenario.md
[t451]: tasks/451-chain-version-itest-matrix.md
[t454]: tasks/454-postgres-transaction-isolation.md
[t457]: tasks/457-process-local-manager-lease.md
[t462]: tasks/462-account-mutation-error-itests.md
[t463]: tasks/463-path-bound-utxo-identity-itests.md
[t464]: tasks/464-no-chain-sync-account-query-itests.md
[t466]: tasks/466-oldest-unused-address-itests.md
[t469]: tasks/469-sign-psbt-with-effective-keys.md
[t470]: tasks/470-lnd-wallet-port.md
[t471]: tasks/471-three-backend-list-unspent-benchmark.md
[t472]: tasks/472-multi-wallet-rescan-isolation-scenario.md
[t474]: tasks/474-authored-change-watch.md
[pr1394]: https://github.com/btcsuite/btcwallet/pull/1394

[t475]: tasks/475-wallet-transaction-subscription.md
[t476]: tasks/476-read-legacy-sql-walletdb-source.md
[t477]: tasks/477-import-legacy-snapshot-postgres.md
[t478]: tasks/478-retain-spend-confirmation-leases.md
[t479]: tasks/479-mix-imported-xpub-accounts.md
[t480]: tasks/480-own-raw-imports-in-one-wallet.md
[t481]: tasks/481-persist-imported-account-address-schema.md
[t482]: tasks/482-fund-imported-inputs-without-origin.md
[t483]: tasks/483-preserve-labels-on-chain-replay.md
[t484]: tasks/484-align-authoring-confirmation-height.md
[t485]: tasks/485-convert-legacy-sql-wallets-through-manager.md

[t486]: tasks/486-freeze-native-sql-schema.md
[t487]: tasks/487-compile-lnd-with-managed-wallet.md
[pr1400]: https://github.com/btcsuite/btcwallet/pull/1400
[pr1407]: https://github.com/btcsuite/btcwallet/pull/1407
[pr1401]: https://github.com/btcsuite/btcwallet/pull/1401
[pr1408]: https://github.com/btcsuite/btcwallet/pull/1408
[pr1411]: https://github.com/btcsuite/btcwallet/pull/1411
[pr1410]: https://github.com/btcsuite/btcwallet/pull/1410
[pr1409]: https://github.com/btcsuite/btcwallet/pull/1409
