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
| [Task 334][t334] | Define aggregate Manager ownership | Design | YY | [#1323][pr1323] |
| [Task 340][t340] | Persist database identity | Bug fix | YY | [#1345][pr1345] |
| [Task 357][t357] | Validate transaction amounts | Bug fix | Abdullahi | [#1348][pr1348] |
| [Task 378][t378] | Translate AccountManager errors | Bug fix | Gustavo | — |
| [Task 387][t387] | Define account-creation identity | Design | YY | [#1342][pr1342] |
| [Task 401][t401] | Add a live-tip-only chain watch | Gap fix | YY | [#1344][pr1344] |
| [Task 421][t421] | Reconcile SQL passphrase commits | Bug fix | YY | [#1341][pr1341] |
| [Task 427][t427] | Test Controller credentials | Test | Gustavo | [#1349][pr1349] |
| [Task 430][t430] | Test account queries | Test | Gustavo | — |
| [Task 431][t431] | Test address allocation | Test | Gustavo | — |
| [Task 432][t432] | Test address imports | Test | Gustavo | — |
| [Task 436][t436] | Test transaction reads | Test | Abdullahi | [#1346][pr1346] |
| [Task 437][t437] | Test transaction labels | Test | Abdullahi | [#1347][pr1347] |
| [Task 455][t455] | Report exact wallet sync status | Bug fix | Gustavo | — |
| [Task 458][t458] | Fence Wallet operations | Bug fix | YY | — |

## Blocked Phase 1 Tasks

These Phase 1 tasks have open hard dependencies. They remain outside the
current focus until every hard dependency completes, while claimant and PR
columns preserve any existing ownership. Task files own all dependency edges,
and `QUEUE.md` owns their delivery phase.

| Task | Description | Type | Claimant | PR |
| --- | --- | --- | --- | --- |
| [Task 329][t329] | Commit SQL genesis atomically | Bug fix | YY | — |
| [Task 331][t331] | Make Manager lifecycle terminal | Bug fix | YY | — |
| [Task 343][t343] | Create next-or-exact root-derived accounts | Gap fix | — | — |
| [Task 344][t344] | Allocate receiving-address batches atomically | Gap fix | — | — |
| [Task 345][t345] | Persist account discovery mode | Gap fix | YY | — |
| [Task 346][t346] | Preserve PSBT funding metadata | Bug fix | Abdullahi | — |
| [Task 347][t347] | Authorize explicit PSBT signatures | Bug fix | Abdullahi | — |
| [Task 348][t348] | Watch PSBT change before return | Bug fix | Gustavo | — |
| [Task 356][t356] | Defer change-script allocation | Bug fix | Gustavo | — |
| [Task 358][t358] | Make Manager creation cancellable | Gap fix | YY | — |
| [Task 364][t364] | Persist live-watch obligations | Gap fix | — | — |
| [Task 366][t366] | Reconcile SQL genesis commits | Bug fix | YY | — |
| [Task 367][t367] | Bound SQL genesis secrets | Bug fix | YY | — |
| [Task 388][t388] | Reject colliding account XPub identities | Bug fix | — | — |
| [Task 389][t389] | Reconcile runtime account commits | Bug fix | — | — |
| [Task 390][t390] | Create accounts in custom scopes | Gap fix | — | — |
| [Task 391][t391] | Create path-bound watch-only accounts | Gap fix | — | — |
| [Task 392][t392] | Consume account discovery mode | Gap fix | — | — |
| [Task 393][t393] | Reject key-only receiving allocation | Gap fix | — | — |
| [Task 394][t394] | Exclude key-only ownership locators | Gap fix | — | — |
| [Task 395][t395] | Allocate durable key-only locators | Gap fix | — | — |
| [Task 396][t396] | Return the oldest unused address | Bug fix | Gustavo | — |
| [Task 399][t399] | Isolate persisted recovery targets | Bug fix | — | — |
| [Task 402][t402] | Replay live-watch obligations | Gap fix | YY | — |
| [Task 404][t404] | Validate transaction input amounts | Bug fix | Abdullahi | — |
| [Task 405][t405] | Reserve PSBT funding inputs | Gap fix | Gustavo | — |
| [Task 406][t406] | Release PSBT funding leases | Gap fix | Gustavo | — |
| [Task 407][t407] | Authorize PSBT auto-finalization | Bug fix | Abdullahi | — |
| [Task 408][t408] | Watch direct transaction change | Bug fix | Gustavo | — |
| [Task 368][t368] | Remove individual Wallet lifecycle | Bug fix | YY | [#1328][pr1328] |
| [Task 452][t452] | Exclude key-only recovery derivation | Gap fix | — | — |
| [Task 456][t456] | Centralize Wallet runtime config | Refactor | YY | — |
| [Task 459][t459] | Assemble startup Wallets | Gap fix | YY | — |

## Claimed Tasks

These claimed tasks sit outside the current Phase 1 delivery phase and are not
listed under `Active Tasks` by the placement rule above. A claim does not
authorize implementation before every dependency and gate completes.

| Task | Description | Type | Phase | Claimant | PR |
| --- | --- | --- | ---: | --- | --- |
| [Task 330][t330] | Reject duplicate wallet creation | Bug fix | 3 | YY | — |
| [Task 338][t338] | Latch accepted SQL passphrase results | Bug fix | 3 | YY | — |
| [Task 350][t350] | Use native PostgreSQL driver | Refactor | 3 | YY | [#1321][pr1321] |
| [Task 362][t362] | Return a definitive kvdb passphrase result | Bug fix | 3 | YY | — |
| [Task 365][t365] | Validate wallet genesis key identities | Bug fix | 3 | YY | — |
| [Task 370][t370] | Prepare ordered Manager chain delivery | Refactor | 3 | YY | — |
| [Task 372][t372] | Prepare atomic multi-Wallet rollback | Bug fix | 3 | YY | — |
| [Task 397][t397] | Define semantic-account recovery targets | Design | 3 | YY | — |
| [Task 398][t398] | Accept semantic account recovery targets | Gap fix | 3 | YY | — |
| [Task 416][t416] | Build database benchmark fixtures | Benchmark | 2 | YY | — |
| [Task 423][t423] | Cut over SQL Manager chain notification ownership | Refactor | 3 | YY | — |
| [Task 424][t424] | Define transaction integrity ownership | Design | 3 | YY | [#1343][pr1343] |
| [Task 428][t428] | Test resync and rescan | Test | 2 | Gustavo | — |
| [Task 454][t454] | Make PostgreSQL isolation explicit | Bug fix | 3 | YY | — |
| [Task 457][t457] | Fence process-local Managers | Gap fix | 3 | YY | — |

## Coordination

Task 334 and PR #1323 must adopt the aggregate Manager lifecycle, active-Wallet
set, and process-local ownership contract before merge.

Task 368 and draft PR #1328 are paused. The current PR implements the removed
per-Wallet lifecycle coordinator and must be rewritten for Task 368's final
individual-lifecycle removal after Tasks 331 and 358 complete. It remains
unmergeable while those hard dependencies are open; no stacked implementation
exception is approved.

Task 350 and PR #1321 are paused behind the new hard dependency on Task 454.
No stacked implementation exception is approved: implementation must not
resume and PR #1321 must remain unmergeable until Task 454 completes.

Task 424 and PR #1343 are paused after the transaction-integrity branch moved
to non-blocking Phase 3. Resume them when Phase 3 becomes current or through
an explicit owner-approved priority exception.

[t329]: tasks/329-atomic-wallet-account-creation.md
[t330]: tasks/330-duplicate-wallet-create.md
[t331]: tasks/331-manager-wallet-lifecycle.md
[t334]: tasks/334-shared-chain-state.md
[t338]: tasks/338-passphrase-result.md
[t340]: tasks/340-durable-database-identity.md
[t343]: tasks/343-exact-account-creation.md
[t344]: tasks/344-bulk-address-allocation.md
[t345]: tasks/345-durable-watch-registration.md
[t346]: tasks/346-psbt-funding-metadata.md
[t347]: tasks/347-psbt-signing-authorization.md
[t348]: tasks/348-watched-change-delivery.md
[t349]: tasks/349-transaction-invalidation.md
[t350]: tasks/350-native-pgx-postgres.md
[t356]: tasks/356-defer-change-script-allocation.md
[t357]: tasks/357-transaction-amount-validation.md
[t358]: tasks/358-contextual-manager-operations.md
[t362]: tasks/362-kvdb-definitive-passphrase-result.md
[t364]: tasks/364-durable-watch-registration.md
[t365]: tasks/365-validate-wallet-genesis-keys.md
[t366]: tasks/366-reconcile-sql-genesis-commit.md
[t367]: tasks/367-bound-sql-genesis-secrets.md
[t368]: tasks/368-wallet-startup-shutdown.md
[t370]: tasks/370-chain-source-delivery.md
[t372]: tasks/372-multi-wallet-rollback.md
[t378]: tasks/378-account-manager-error-boundary.md
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
[t397]: tasks/397-semantic-recovery-targets-adr.md
[t398]: tasks/398-semantic-account-recovery-targets.md
[t399]: tasks/399-zero-lookahead-persisted-targets.md
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
[t423]: tasks/423-manager-chain-source-cutover.md
[t424]: tasks/424-transaction-utxo-integrity-adr.md
[t427]: tasks/427-controller-lifecycle-credential-itests.md
[t428]: tasks/428-controller-sync-info-itests.md
[t430]: tasks/430-account-query-itests.md
[t431]: tasks/431-address-allocation-lookup-itests.md
[t432]: tasks/432-address-import-script-itests.md
[t436]: tasks/436-transaction-read-itests.md
[t437]: tasks/437-transaction-label-itests.md
[t452]: tasks/452-key-only-recovery-derivation.md
[t454]: tasks/454-postgres-transaction-isolation.md
[t455]: tasks/455-exact-wallet-sync-status.md
[t456]: tasks/456-manager-runtime-configuration.md
[t457]: tasks/457-process-local-manager-lease.md
[t458]: tasks/458-manager-operation-admission.md
[t459]: tasks/459-manager-startup-wallet-assembly.md
[pr1321]: https://github.com/btcsuite/btcwallet/pull/1321
[pr1323]: https://github.com/btcsuite/btcwallet/pull/1323
[pr1328]: https://github.com/btcsuite/btcwallet/pull/1328
[pr1344]: https://github.com/btcsuite/btcwallet/pull/1344
[pr1343]: https://github.com/btcsuite/btcwallet/pull/1343
[pr1342]: https://github.com/btcsuite/btcwallet/pull/1342
[pr1341]: https://github.com/btcsuite/btcwallet/pull/1341
[pr1345]: https://github.com/btcsuite/btcwallet/pull/1345
[pr1347]: https://github.com/btcsuite/btcwallet/pull/1347
[pr1346]: https://github.com/btcsuite/btcwallet/pull/1346
[pr1348]: https://github.com/btcsuite/btcwallet/pull/1348
[pr1349]: https://github.com/btcsuite/btcwallet/pull/1349
