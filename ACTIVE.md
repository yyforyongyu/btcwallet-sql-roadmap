# Active Tasks

Claim early, own the development, and ship. After a task merges, remove it
here and archive its task file.

## Active Tasks

Phase 1 is the current development focus. Every task in this list has no open
hard dependencies. It contains all dependency-ready Phase 1 tasks and
dependency-ready later-phase work assigned to developers other than YY. Park
blocked claims and YY's later-phase work under `Claimed Tasks`. Type guides
which ready work to select first; it does not change readiness or list
placement.

| Task | Description | Type | Claimant | PR |
| --- | --- | --- | --- | --- |
| [Task 337][t337] | Restore imported-XPub derivation | Bug fix | YY | [#1324][pr1324] |
| [Task 342][t342] | Bound recovery windows | Bug fix | — | — |
| [Task 355][t355] | Correct transaction size accounting | Bug fix | — | — |
| [Task 357][t357] | Validate transaction amounts | Bug fix | — | — |
| [Task 359][t359] | Revalidate database identity | Design | YY | [#1319][pr1319] |
| [Task 377][t377] | Return wallet-owned accounts | Refactor | YY | — |
| [Task 400][t400] | Define live-watch semantics | Design | YY | [#1322][pr1322] |
| [Task 425][t425] | Isolate transaction authoring | Refactor | YY | — |
| [Task 427][t427] | Test Controller credentials | Test | Gustavo | — |
| [Task 428][t428] | Test resync and rescan | Test | Gustavo | — |
| [Task 429][t429] | Test account mutations | Test | Gustavo | [#1326][pr1326] |
| [Task 430][t430] | Test account queries | Test | Gustavo | — |
| [Task 431][t431] | Test address allocation | Test | Gustavo | — |
| [Task 432][t432] | Test address imports | Test | Gustavo | — |
| [Task 434][t434] | Test transaction creation | Test | Abdullahi | [#1316][pr1316] |
| [Task 435][t435] | Test transaction publication | Test | Abdullahi | — |
| [Task 436][t436] | Test transaction reads | Test | Abdullahi | — |
| [Task 437][t437] | Test transaction labels | Test | Abdullahi | — |

## Claimed Tasks

These tasks have owners but are parked outside the current focus. A claim does
not authorize implementation before every dependency and gate completes.

| Task | Description | Type | Phase | Claimant | PR |
| --- | --- | --- | ---: | --- | --- |
| [Task 329][t329] | Commit SQL genesis atomically | Bug fix | 2 | YY | — |
| [Task 331][t331] | Fence Manager lifecycle | Bug fix | 2 | YY | — |
| [Task 334][t334] | Define shared-chain ownership | Design | 2 | YY | [#1323][pr1323] |
| [Task 340][t340] | Persist database identity | Bug fix | 1 | YY | — |
| [Task 345][t345] | Persist account discovery mode | Gap fix | 1 | YY | — |
| [Task 350][t350] | Use native PostgreSQL driver | Refactor | 2 | YY | [#1321][pr1321] |
| [Task 368][t368] | Serialize Wallet lifecycle | Bug fix | 2 | YY | — |
| [Task 416][t416] | Build database benchmark fixtures | Benchmark | 3 | YY | — |

## Coordination

- Tasks 332, 334, 337, 350, 359, and 400 are YY's grandfathered parallel
  in-progress work. Task 332 was shared with Gustavo; #1306 has merged and the
  remaining work is #1320. YY starts no additional task until only one remains
  in progress.
- Gustavo's Phase 3 series, Tasks 427–432, and Abdullahi's Phase 3 series,
  Tasks 434–437, already started and remain active as explicit series
  exceptions. Do not add work to either series before it clears.
- YY's blocked Phase 1 Tasks 340 and 345 remain claimed. YY's Phase 2 Tasks
  329, 331, 334, 350, and 368 and Phase 3 Task 416 remain parked from the
  current focus.
- Outside the exceptions above, each developer may have only one task in
  progress at a time.


[t329]: tasks/329-atomic-wallet-account-creation.md
[t331]: tasks/331-manager-wallet-lifecycle.md
[t334]: tasks/334-shared-chain-state.md
[t337]: tasks/337-imported-xpub-derivation.md
[t340]: tasks/340-durable-database-identity.md
[t342]: tasks/342-recovery-window-validation.md
[t345]: tasks/345-durable-watch-registration.md
[t350]: tasks/350-native-pgx-postgres.md
[t355]: tasks/355-transaction-size-accounting.md
[t357]: tasks/357-transaction-amount-validation.md
[t359]: tasks/359-revalidate-database-identity-design.md
[t368]: tasks/368-wallet-startup-shutdown.md
[t377]: tasks/377-account-manager-wallet-dto.md
[t400]: tasks/400-live-watch-semantics-adr.md
[t416]: tasks/416-database-benchmark-fixtures.md
[t425]: tasks/425-separate-transaction-authoring-wrappers.md
[t427]: tasks/427-controller-lifecycle-credential-itests.md
[t428]: tasks/428-controller-sync-info-itests.md
[t429]: tasks/429-account-mutation-itests.md
[t430]: tasks/430-account-query-itests.md
[t431]: tasks/431-address-allocation-lookup-itests.md
[t432]: tasks/432-address-import-script-itests.md
[t434]: tasks/434-transaction-creation-itests.md
[t435]: tasks/435-transaction-publication-itests.md
[t436]: tasks/436-transaction-read-itests.md
[t437]: tasks/437-transaction-label-itests.md
[pr1321]: https://github.com/btcsuite/btcwallet/pull/1321
[pr1319]: https://github.com/btcsuite/btcwallet/pull/1319
[pr1316]: https://github.com/btcsuite/btcwallet/pull/1316
[pr1322]: https://github.com/btcsuite/btcwallet/pull/1322
[pr1323]: https://github.com/btcsuite/btcwallet/pull/1323
[pr1324]: https://github.com/btcsuite/btcwallet/pull/1324
[pr1326]: https://github.com/btcsuite/btcwallet/pull/1326
