# Active Tasks

Claim early, own the development, and ship. After a task merges, remove it
here and archive its task file.

## Active Tasks

Phase 1 is the current development focus. This list contains every claimed
Phase 1 task, every open Phase 1 task that no other open task depends on, and
explicitly already-started work from later phases. A row may still be blocked;
its task file remains authoritative for implementation gates.

| Task | Description | Claimant | PR |
| --- | --- | --- | --- |
| [Task 329][t329] | Commit SQL genesis atomically | YY | — |
| [Task 330][t330] | Reject duplicate wallet creation | — | — |
| [Task 331][t331] | Fence Manager lifecycle | YY | — |
| [Task 332][t332] | Define watch-only policy | YY + Gustavo | [#1320][pr1320] |
| [Task 334][t334] | Define shared-chain ownership | YY | — |
| [Task 337][t337] | Restore imported-XPub derivation | YY | — |
| [Task 340][t340] | Persist database identity | YY | — |
| [Task 341][t341] | Define semantic account identity | YY | [#1313][pr1313] |
| [Task 345][t345] | Persist account discovery mode | YY | — |
| [Task 350][t350] | Use native PostgreSQL driver | YY | [#1321][pr1321] |
| [Task 358][t358] | Make Manager creation cancellable | — | — |
| [Task 359][t359] | Revalidate database identity | YY | [#1319][pr1319] |
| [Task 361][t361] | Support kvdb XPub derivation | — | — |
| [Task 363][t363] | Bound recovery derivation work | — | — |
| [Task 365][t365] | Validate wallet genesis keys | — | — |
| [Task 369][t369] | Reject repeated unlocks | — | — |
| [Task 378][t378] | Translate account errors | — | — |
| [Task 379][t379] | Translate load errors | — | — |
| [Task 380][t380] | Translate address lookup errors | — | — |
| [Task 381][t381] | Hide recovery adapter identity | — | — |
| [Task 400][t400] | Define live-watch semantics | YY | — |
| [Task 403][t403] | Persist raw import watches | — | — |
| [Task 413][t413] | Bind UTXOs to spending inputs | — | — |
| [Task 414][t414] | Bound persisted UTXO amounts | — | — |
| [Task 423][t423] | Cut over Manager chain source | — | — |
| [Task 426][t426] | Validate transaction output totals | — | — |
| [Task 427][t427] | Test Controller credentials | Gustavo | — |
| [Task 428][t428] | Test resync and rescan | Gustavo | — |
| [Task 429][t429] | Test account mutations | Gustavo | — |
| [Task 430][t430] | Test account queries | Gustavo | — |
| [Task 431][t431] | Test address allocation | Gustavo | — |
| [Task 432][t432] | Test address imports | Gustavo | — |
| [Task 433][t433] | Enforce sync test boundary | YY | [#1314][pr1314] |
| [Task 434][t434] | Test transaction creation | Abdullahi | [#1316][pr1316] |
| [Task 435][t435] | Test transaction publication | Abdullahi | — |
| [Task 436][t436] | Test transaction reads | Abdullahi | — |
| [Task 437][t437] | Test transaction labels | Abdullahi | — |

## Claimed Tasks

These tasks have owners but are parked outside the current focus. A claim does
not authorize implementation before every dependency and gate completes.

| Task | Description | Phase | Claimant | PR |
| --- | --- | ---: | --- | --- |
| [Task 416][t416] | Build SQLite benchmark fixtures | 2 | YY | — |

## Coordination

- Tasks 332, 341, 350, 359, and 433 are YY's grandfathered parallel
  in-progress work. Task 332 was shared with Gustavo; #1306 has merged and the
  remaining work is #1320. YY starts no additional task until only one remains
  in progress.
- Gustavo's Phase 2 series, Tasks 427–432, and Abdullahi's Phase 2 series,
  Tasks 434–437, already started and remain active as explicit series
  exceptions. Do not add work to either series before it clears.
- Task 416 is claimed by YY but remains parked with the rest of unstarted
  Phase 2 work.
- Task 329 waits for Task 332 and YY's two prerequisite chains:
  `360 → 341 → 337` and `359 → 340 → 345`.
- Outside the exceptions above, each developer may have only one task in
  progress at a time.

[pr1313]: https://github.com/btcsuite/btcwallet/pull/1313
[pr1314]: https://github.com/btcsuite/btcwallet/pull/1314

[t329]: tasks/329-atomic-wallet-account-creation.md
[t330]: tasks/330-duplicate-wallet-create.md
[t331]: tasks/331-manager-wallet-lifecycle.md
[t332]: tasks/332-watch-only-script-policy.md
[t334]: tasks/334-shared-chain-state.md
[t337]: tasks/337-imported-xpub-derivation.md
[t340]: tasks/340-durable-database-identity.md
[t341]: tasks/341-public-role-contract.md
[t345]: tasks/345-durable-watch-registration.md
[t350]: tasks/350-native-pgx-postgres.md
[t358]: tasks/358-contextual-manager-operations.md
[t359]: tasks/359-revalidate-database-identity-design.md
[t361]: tasks/361-kvdb-imported-xpub-derivation.md
[t363]: tasks/363-recovery-scan-work-bound.md
[t365]: tasks/365-validate-wallet-genesis-keys.md
[t369]: tasks/369-repeated-wallet-unlock.md
[t378]: tasks/378-account-manager-error-boundary.md
[t379]: tasks/379-manager-load-error-boundary.md
[t380]: tasks/380-address-lookup-error-boundary.md
[t381]: tasks/381-recovery-adapter-identity.md
[t400]: tasks/400-live-watch-semantics-adr.md
[t403]: tasks/403-raw-import-watch-obligations.md
[t413]: tasks/413-utxo-spend-identity.md
[t414]: tasks/414-utxo-amount-bounds.md
[t416]: tasks/416-sqlite-benchmark-fixtures.md
[t423]: tasks/423-manager-chain-source-cutover.md
[t426]: tasks/426-sql-transaction-output-aggregates.md
[t427]: tasks/427-controller-lifecycle-credential-itests.md
[t428]: tasks/428-controller-sync-info-itests.md
[t429]: tasks/429-account-mutation-itests.md
[t430]: tasks/430-account-query-itests.md
[t431]: tasks/431-address-allocation-lookup-itests.md
[t432]: tasks/432-address-import-script-itests.md
[t433]: tasks/433-wallet-sync-test-boundary.md
[t434]: tasks/434-transaction-creation-itests.md
[t435]: tasks/435-transaction-publication-itests.md
[t436]: tasks/436-transaction-read-itests.md
[t437]: tasks/437-transaction-label-itests.md
[pr1320]: https://github.com/btcsuite/btcwallet/pull/1320
[pr1321]: https://github.com/btcsuite/btcwallet/pull/1321
[pr1319]: https://github.com/btcsuite/btcwallet/pull/1319
[pr1316]: https://github.com/btcsuite/btcwallet/pull/1316
