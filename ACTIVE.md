# Active Tasks

Claim early, own the development, and ship. After a task merges, remove it
here and archive its task file.

## Active Tasks

Phase 1 is the current development focus. Every task in this list has no open
hard dependencies. It contains all dependency-ready Phase 1 tasks and
dependency-ready later-phase work assigned to developers other than YY. Put
all Phase 1 work with open hard dependencies under `Blocked Phase 1 Tasks`,
whether claimed or unclaimed. Park claimed later-phase work under `Claimed
Tasks` unless the active rule above places it under `Active Tasks`. Type guides
which ready work to select first; it does not change readiness or list
placement.

| Task | Description | Type | Claimant | PR |
| --- | --- | --- | --- | --- |
| [Task 334][t334] | Define shared-chain ownership | Design | YY | [#1323][pr1323] |
| [Task 337][t337] | Restore imported-XPub derivation | Bug fix | YY | [#1324][pr1324] |
| [Task 339][t339] | Define transaction visibility | Gap fix | YY | [#1330][pr1330] |
| [Task 355][t355] | Correct transaction size accounting | Bug fix | Gustavo | — |
| [Task 357][t357] | Validate transaction amounts | Bug fix | — | — |
| [Task 359][t359] | Revalidate database identity | Design | YY | [#1319][pr1319] |
| [Task 377][t377] | Return wallet-owned accounts | Gap fix | YY | [#1329][pr1329] |
| [Task 378][t378] | Translate AccountManager errors | Bug fix | — | — |
| [Task 379][t379] | Translate Manager load errors | Bug fix | YY | [#1331][pr1331] |
| [Task 380][t380] | Translate address lookup errors | Bug fix | YY | [#1332][pr1332] |
| [Task 387][t387] | Define account-creation identity | Design | YY | — |
| [Task 400][t400] | Define live-watch semantics | Design | YY | [#1322][pr1322] |
| [Task 421][t421] | Reconcile SQL passphrase commits | Bug fix | YY | — |
| [Task 424][t424] | Define transaction integrity ownership | Design | YY | — |
| [Task 425][t425] | Isolate transaction authoring | Refactor | YY | [#1327][pr1327] |
| [Task 427][t427] | Test Controller credentials | Test | Gustavo | — |
| [Task 428][t428] | Test resync and rescan | Test | Gustavo | — |
| [Task 429][t429] | Test account mutations | Test | Gustavo | [#1326][pr1326] |
| [Task 430][t430] | Test account queries | Test | Gustavo | — |
| [Task 431][t431] | Test address allocation | Test | Gustavo | — |
| [Task 432][t432] | Test address imports | Test | Gustavo | — |
| [Task 435][t435] | Test transaction publication | Test | Abdullahi | [#1334][pr1334] |
| [Task 436][t436] | Test transaction reads | Test | Abdullahi | — |
| [Task 437][t437] | Test transaction labels | Test | Abdullahi | — |
| [Task 453][t453] | Serialize Manager Wallet publication | Bug fix | YY | [#1336][pr1336] |

## Blocked Phase 1 Tasks

These Phase 1 tasks have open hard dependencies. They remain outside the
current focus until every hard dependency completes, while claimant and PR
columns preserve any existing ownership. Task files own all dependency edges,
and `QUEUE.md` owns their delivery phase.

| Task | Description | Type | Claimant | PR |
| --- | --- | --- | --- | --- |
| [Task 329][t329] | Commit SQL genesis atomically | Bug fix | YY | — |
| [Task 330][t330] | Reject duplicate wallet creation | Bug fix | — | — |
| [Task 331][t331] | Fence Manager lifecycle | Bug fix | YY | — |
| [Task 338][t338] | Latch accepted SQL passphrase results | Bug fix | YY | — |
| [Task 340][t340] | Persist database identity | Bug fix | YY | — |
| [Task 343][t343] | Create next-or-exact root-derived accounts | Gap fix | — | — |
| [Task 344][t344] | Allocate receiving-address batches atomically | Gap fix | — | — |
| [Task 345][t345] | Persist account discovery mode | Gap fix | YY | — |
| [Task 346][t346] | Preserve PSBT funding metadata | Bug fix | — | — |
| [Task 347][t347] | Authorize explicit PSBT signatures | Bug fix | — | — |
| [Task 348][t348] | Watch PSBT change before return | Bug fix | — | — |
| [Task 349][t349] | Implement transaction invalidation | Gap fix | — | — |
| [Task 356][t356] | Defer change-script allocation | Bug fix | — | — |
| [Task 358][t358] | Make Manager creation cancellable | Gap fix | YY | — |
| [Task 364][t364] | Persist live-watch obligations | Gap fix | — | — |
| [Task 366][t366] | Reconcile SQL genesis commits | Bug fix | — | — |
| [Task 367][t367] | Bound SQL genesis secrets | Bug fix | — | — |
| [Task 368][t368] | Manager-own Wallet lifecycle | Bug fix | YY | [#1328][pr1328] |
| [Task 388][t388] | Reject colliding account XPub identities | Bug fix | — | — |
| [Task 389][t389] | Reconcile runtime account commits | Bug fix | — | — |
| [Task 390][t390] | Create accounts in custom scopes | Gap fix | — | — |
| [Task 391][t391] | Create path-bound watch-only accounts | Gap fix | — | — |
| [Task 392][t392] | Consume account discovery mode | Gap fix | — | — |
| [Task 393][t393] | Reject key-only receiving allocation | Gap fix | — | — |
| [Task 394][t394] | Exclude key-only ownership locators | Gap fix | — | — |
| [Task 395][t395] | Allocate durable key-only locators | Gap fix | — | — |
| [Task 396][t396] | Return the oldest unused address | Bug fix | — | — |
| [Task 399][t399] | Isolate persisted recovery targets | Bug fix | — | — |
| [Task 401][t401] | Add a live-tip-only chain watch | Gap fix | — | — |
| [Task 402][t402] | Replay live-watch obligations | Gap fix | — | — |
| [Task 404][t404] | Validate transaction input amounts | Bug fix | — | — |
| [Task 405][t405] | Reserve PSBT funding inputs | Gap fix | — | — |
| [Task 406][t406] | Release PSBT funding leases | Gap fix | — | — |
| [Task 407][t407] | Authorize PSBT auto-finalization | Bug fix | — | — |
| [Task 408][t408] | Watch direct transaction change | Bug fix | — | — |
| [Task 409][t409] | Normalize transaction input edges | Gap fix | — | — |
| [Task 410][t410] | Coordinate transaction graph mutations | Bug fix | — | — |
| [Task 411][t411] | Reject children of terminal transactions | Bug fix | — | — |
| [Task 412][t412] | Expose invalidation with publication ordering | Gap fix | — | — |
| [Task 422][t422] | Make Manager loading cancellable | Gap fix | YY | — |
| [Task 452][t452] | Exclude key-only recovery derivation | Gap fix | — | — |

## Claimed Tasks

These claimed tasks sit outside the current Phase 1 delivery phase and are not
listed under `Active Tasks` by the placement rule above. A claim does not
authorize implementation before every dependency and gate completes.

| Task | Description | Type | Phase | Claimant | PR |
| --- | --- | --- | ---: | --- | --- |
| [Task 350][t350] | Use native PostgreSQL driver | Refactor | 2 | YY | [#1321][pr1321] |
| [Task 416][t416] | Build database benchmark fixtures | Benchmark | 3 | YY | — |

[t329]: tasks/329-atomic-wallet-account-creation.md
[t330]: tasks/330-duplicate-wallet-create.md
[t331]: tasks/331-manager-wallet-lifecycle.md
[t334]: tasks/334-shared-chain-state.md
[t337]: tasks/337-imported-xpub-derivation.md
[t338]: tasks/338-passphrase-result.md
[t339]: tasks/339-transaction-visibility.md
[t340]: tasks/340-durable-database-identity.md
[t343]: tasks/343-exact-account-creation.md
[t344]: tasks/344-bulk-address-allocation.md
[t345]: tasks/345-durable-watch-registration.md
[t346]: tasks/346-psbt-funding-metadata.md
[t347]: tasks/347-psbt-signing-authorization.md
[t348]: tasks/348-watched-change-delivery.md
[t349]: tasks/349-transaction-invalidation.md
[t350]: tasks/350-native-pgx-postgres.md
[t355]: tasks/355-transaction-size-accounting.md
[t356]: tasks/356-defer-change-script-allocation.md
[t357]: tasks/357-transaction-amount-validation.md
[t358]: tasks/358-contextual-manager-operations.md
[t359]: tasks/359-revalidate-database-identity-design.md
[t364]: tasks/364-durable-watch-registration.md
[t366]: tasks/366-reconcile-sql-genesis-commit.md
[t367]: tasks/367-bound-sql-genesis-secrets.md
[t368]: tasks/368-wallet-startup-shutdown.md
[t377]: tasks/377-account-manager-wallet-dto.md
[t378]: tasks/378-account-manager-error-boundary.md
[t379]: tasks/379-manager-load-error-boundary.md
[t380]: tasks/380-address-lookup-error-boundary.md
[t387]: tasks/387-account-creation-identity-adr.md
[t388]: tasks/388-runtime-account-xpub-collisions.md
[t389]: tasks/389-runtime-account-commit-reconciliation.md
[t390]: tasks/390-custom-scope-account-creation.md
[t391]: tasks/391-path-bound-watch-only-xpub-accounts.md
[t392]: tasks/392-discovery-mode-account-creation.md
[t393]: tasks/393-key-only-receiving-rejection.md
[t394]: tasks/394-key-only-transaction-ownership.md
[t395]: tasks/395-key-only-locator-allocation.md
[t396]: tasks/396-oldest-unused-address.md
[t399]: tasks/399-zero-lookahead-persisted-targets.md
[t400]: tasks/400-live-watch-semantics-adr.md
[t401]: tasks/401-live-tip-chain-watch.md
[t402]: tasks/402-replay-live-watch-obligations.md
[t404]: tasks/404-transaction-input-amounts.md
[t405]: tasks/405-psbt-funding-lease-admission.md
[t406]: tasks/406-psbt-funding-lease-release.md
[t407]: tasks/407-psbt-finalization-authorization.md
[t408]: tasks/408-direct-transaction-change-watch.md
[t409]: tasks/409-transaction-input-graph.md
[t410]: tasks/410-transaction-graph-admission.md
[t411]: tasks/411-terminal-parent-rejection.md
[t412]: tasks/412-transaction-publication-admission.md
[t416]: tasks/416-database-benchmark-fixtures.md
[t421]: tasks/421-sql-passphrase-commit-reconciliation.md
[t422]: tasks/422-manager-load-cancellation.md
[t424]: tasks/424-transaction-utxo-integrity-adr.md
[t425]: tasks/425-separate-transaction-authoring-wrappers.md
[t427]: tasks/427-controller-lifecycle-credential-itests.md
[t428]: tasks/428-controller-sync-info-itests.md
[t429]: tasks/429-account-mutation-itests.md
[t430]: tasks/430-account-query-itests.md
[t431]: tasks/431-address-allocation-lookup-itests.md
[t432]: tasks/432-address-import-script-itests.md
[t435]: tasks/435-transaction-publication-itests.md
[t436]: tasks/436-transaction-read-itests.md
[t437]: tasks/437-transaction-label-itests.md
[t452]: tasks/452-key-only-recovery-derivation.md
[t453]: tasks/453-serialize-manager-wallet-publication.md
[pr1321]: https://github.com/btcsuite/btcwallet/pull/1321
[pr1319]: https://github.com/btcsuite/btcwallet/pull/1319
[pr1322]: https://github.com/btcsuite/btcwallet/pull/1322
[pr1323]: https://github.com/btcsuite/btcwallet/pull/1323
[pr1324]: https://github.com/btcsuite/btcwallet/pull/1324
[pr1326]: https://github.com/btcsuite/btcwallet/pull/1326
[pr1329]: https://github.com/btcsuite/btcwallet/pull/1329
[pr1328]: https://github.com/btcsuite/btcwallet/pull/1328
[pr1327]: https://github.com/btcsuite/btcwallet/pull/1327
[pr1332]: https://github.com/btcsuite/btcwallet/pull/1332
[pr1331]: https://github.com/btcsuite/btcwallet/pull/1331
[pr1330]: https://github.com/btcsuite/btcwallet/pull/1330
[pr1334]: https://github.com/btcsuite/btcwallet/pull/1334
[pr1336]: https://github.com/btcsuite/btcwallet/pull/1336
