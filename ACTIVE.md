# Active Tasks

Claim early, own the development, and ship. After a numbered task merges,
remove it here and archive its task file. Merged ad hoc PRs go directly to
[QUEUE.md](QUEUE.md), not these live-work lists.

## Active Tasks

Phase 1 is the current development focus. Every task in this list has no open
hard dependencies. It contains all dependency-ready Phase 1 tasks and
dependency-ready later-phase work assigned to developers other than YY. Put
all Phase 1 work with open hard dependencies under `Blocked Phase 1 Tasks`,
whether claimed or unclaimed. Park claimed later-phase work under `Claimed
Tasks` unless the active rule above places it under `Active Tasks`. Type guides
which ready work to select first; it does not change readiness or list
placement. Open dependencies block direct merge, not implementation in a
recorded PR stack.

| Task | Description | Type | Claimant | PR |
| --- | --- | --- | --- | --- |
| [Task 402][t402] | Establish and reconstruct live watches | Gap fix | YY | [#1380][pr1380] |
| [Task 430][t430] | Test account queries | Test | Gustavo | — |
| [Task 431][t431] | Test address allocation | Test | Gustavo | — |
| [Task 432][t432] | Test address imports | Test | Gustavo | — |
| [Task 442][t442] | Cover public signing contracts | Test | YY | [#1384][pr1384] |
| [Task 443][t443] | Cover UnsafeSigner extraction contracts | Test | YY | [#1383][pr1383] |

## Blocked Phase 1 Tasks

These Phase 1 tasks have open hard dependencies and therefore cannot merge
yet. They may still advance in recorded stacked PRs; claimant and PR columns
show that work without making the task dependency-ready. Task files own all
dependency edges, and `QUEUE.md` owns their delivery phase.

| Task | Description | Type | Claimant | PR |
| --- | --- | --- | --- | --- |
| [Task 344][t344] | Allocate receiving-address batches atomically | Gap fix | YY | [#1379][pr1379] |
| [Task 395][t395] | Add AllocateNextKey for durable key allocation | Gap fix | YY | [#1386][pr1386] |
| [Task 396][t396] | Return the oldest unused address | Bug fix | Gustavo | — |
| [Task 461][t461] | Cover public address batch contracts | Test | — | — |
| [Task 465][t465] | Verify durable key allocation results | Test | YY | — |
| [Task 466][t466] | Verify final oldest-unused allocation | Test | — | — |

## Claimed Tasks

These claimed tasks sit outside the current Phase 1 delivery phase and are not
listed under `Active Tasks` by the placement rule above. When phase priority
permits implementation, a claimed task may proceed as a stacked PR, but it
cannot merge before every hard dependency and gate completes. Sort this list
by ascending phase, then ascending task number.

| Task | Description | Type | Phase | Claimant | PR |
| --- | --- | --- | ---: | --- | --- |
| [Task 428][t428] | Test resync and rescan | Test | 2 | Gustavo | — |
| [Task 329][t329] | Commit SQL genesis atomically | Bug fix | 3 | YY | — |
| [Task 330][t330] | Reject duplicate wallet creation | Bug fix | 4 | YY | — |
| [Task 347][t347] | Authorize existing PSBT signatures | Bug fix | 4 | Abdullahi | — |
| [Task 348][t348] | Watch PSBT change before return | Bug fix | 4 | Gustavo | — |
| [Task 350][t350] | Use native PostgreSQL driver | Refactor | 4 | YY | [#1321][pr1321] |
| [Task 356][t356] | Defer change-script allocation | Bug fix | 4 | Gustavo | — |
| [Task 365][t365] | Validate wallet genesis key identities | Bug fix | 4 | YY | — |
| [Task 367][t367] | Bound SQL genesis secrets | Bug fix | 4 | YY | — |
| [Task 370][t370] | Fan out ordered Manager chain events | Refactor | 4 | YY | — |
| [Task 372][t372] | Coordinate multi-Wallet rollback | Bug fix | 4 | YY | — |
| [Task 397][t397] | Define semantic-account recovery targets | Design | 4 | YY | — |
| [Task 398][t398] | Accept semantic account recovery targets | Gap fix | 4 | YY | — |
| [Task 405][t405] | Reserve PSBT funding inputs | Gap fix | 4 | Gustavo | — |
| [Task 406][t406] | Clean up failed PSBT funding leases | Gap fix | 4 | Gustavo | — |
| [Task 407][t407] | Authorize PSBT auto-finalization | Bug fix | 4 | Abdullahi | — |
| [Task 408][t408] | Watch direct transaction change | Bug fix | 4 | Gustavo | — |
| [Task 423][t423] | Cut over Manager chain notifications | Refactor | 4 | YY | — |
| [Task 424][t424] | Define transaction integrity ownership | Design | 4 | YY | [#1343][pr1343] |
| [Task 454][t454] | Make PostgreSQL isolation explicit | Bug fix | 4 | YY | — |
| [Task 457][t457] | Fence process-local Managers | Gap fix | 4 | YY | — |
| [Task 469][t469] | Sign PSBT inputs with effective keys | Bug fix | 4 | Abdullahi | — |

[t329]: tasks/329-atomic-wallet-account-creation.md
[t330]: tasks/330-duplicate-wallet-create.md
[t344]: tasks/344-bulk-address-allocation.md
[t347]: tasks/347-psbt-signing-authorization.md
[t348]: tasks/348-watched-change-delivery.md
[t349]: archive/tasks/349-transaction-invalidation.md
[t350]: tasks/350-native-pgx-postgres.md
[t356]: tasks/356-defer-change-script-allocation.md
[t365]: tasks/365-validate-wallet-genesis-keys.md
[t366]: tasks/366-reconcile-sql-genesis-commit.md
[t367]: tasks/367-bound-sql-genesis-secrets.md
[t370]: tasks/370-chain-source-delivery.md
[t372]: tasks/372-multi-wallet-rollback.md
[t388]: tasks/388-runtime-account-xpub-collisions.md
[t389]: tasks/389-runtime-account-commit-reconciliation.md
[t391]: tasks/391-path-bound-watch-only-xpub-accounts.md
[t395]: tasks/395-key-only-locator-allocation.md
[t396]: tasks/396-oldest-unused-address.md
[t397]: tasks/397-semantic-recovery-targets-adr.md
[t398]: tasks/398-semantic-account-recovery-targets.md
[t402]: tasks/402-replay-live-watch-obligations.md
[t405]: tasks/405-psbt-funding-lease-admission.md
[t406]: tasks/406-psbt-funding-lease-release.md
[t407]: tasks/407-psbt-finalization-authorization.md
[t408]: tasks/408-direct-transaction-change-watch.md
[t409]: tasks/409-transaction-input-graph.md
[t410]: tasks/410-transaction-graph-admission.md
[t411]: tasks/411-terminal-parent-rejection.md
[t423]: tasks/423-manager-chain-source-cutover.md
[t424]: tasks/424-transaction-utxo-integrity-adr.md
[t428]: tasks/428-controller-sync-info-itests.md
[t430]: tasks/430-account-query-itests.md
[t431]: tasks/431-address-allocation-lookup-itests.md
[t432]: tasks/432-address-import-script-itests.md
[t442]: tasks/442-signer-signing-itests.md
[t443]: tasks/443-unsafe-signer-itests.md
[t454]: tasks/454-postgres-transaction-isolation.md
[t457]: tasks/457-process-local-manager-lease.md
[t460]: tasks/460-sequential-account-invalid-children.md
[t461]: tasks/461-address-batch-itests.md
[t465]: tasks/465-new-address-result-itests.md
[t466]: tasks/466-oldest-unused-address-itests.md
[t469]: tasks/469-sign-psbt-with-effective-keys.md
[pr1321]: https://github.com/btcsuite/btcwallet/pull/1321
[pr1343]: https://github.com/btcsuite/btcwallet/pull/1343
[pr1370]: https://github.com/btcsuite/btcwallet/pull/1370
[pr1379]: https://github.com/btcsuite/btcwallet/pull/1379
[pr1380]: https://github.com/btcsuite/btcwallet/pull/1380
[pr1384]: https://github.com/btcsuite/btcwallet/pull/1384
[pr1383]: https://github.com/btcsuite/btcwallet/pull/1383
[pr1386]: https://github.com/btcsuite/btcwallet/pull/1386
