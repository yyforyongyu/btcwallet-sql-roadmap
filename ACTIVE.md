# Active Tasks

Claim early, own the development, and ship. After a task merges, remove it
here and archive its task file.

## Active Tasks

These Phase 1 tasks are the current development focus. A claim records
ownership; only `In progress` means implementation has started.

| Task | Description | Claimant | State | PR |
| --- | --- | --- | --- | --- |
| [Task 332][t332] | Define watch-only policy | YY + Gustavo | In progress | [#1305][pr1305], [#1306][pr1306] |
| [Task 333][t333] | Add PostgreSQL test harness | YY | In progress | [#1303][pr1303] |
| [Task 334][t334] | Define shared-chain ownership | YY | Claimed | — |
| [Task 335][t335] | Validate legacy migration snapshot | — | Unclaimed | — |
| [Task 339][t339] | Define transaction visibility | — | Unclaimed | — |
| [Task 342][t342] | Bound recovery windows | — | Unclaimed | — |
| [Task 350][t350] | Use native PostgreSQL driver | YY | Claimed | — |
| [Task 352][t352] | Rotate kvdb passphrases atomically | — | Unclaimed | — |
| [Task 354][t354] | Define Vault secret ownership | — | Unclaimed | — |
| [Task 355][t355] | Correct transaction size accounting | — | Unclaimed | — |
| [Task 357][t357] | Validate transaction amounts | — | Unclaimed | — |
| [Task 359][t359] | Revalidate database identity | YY | Claimed | — |
| [Task 360][t360] | Document kvdb adapter identity | YY | Claimed | — |
| [Task 368][t368] | Serialize Wallet lifecycle | — | Unclaimed | — |
| [Task 369][t369] | Reject repeated unlocks | — | Unclaimed | — |
| [Task 378][t378] | Translate account errors | — | Unclaimed | — |
| [Task 379][t379] | Translate load errors | — | Unclaimed | — |
| [Task 380][t380] | Translate address lookup errors | — | Unclaimed | — |
| [Task 400][t400] | Define live-watch semantics | — | Unclaimed | — |
| [Task 415][t415] | Test public UTXO contracts | Abdullahi | In progress | [#1307][pr1307] |
| [Task 416][t416] | Build SQLite benchmark fixtures | — | Unclaimed | — |
| [Task 421][t421] | Reconcile passphrase commits | — | Unclaimed | — |
| [Task 424][t424] | Define transaction integrity | — | Unclaimed | — |
| [Task 425][t425] | Isolate transaction authoring | — | Unclaimed | — |
| [Task 427][t427] | Test Controller credentials | Gustavo | Claimed | — |
| [Task 428][t428] | Test resync and rescan | Gustavo | Claimed | — |
| [Task 429][t429] | Test account mutations | Gustavo | Claimed | — |
| [Task 430][t430] | Test account queries | Gustavo | Claimed | — |
| [Task 431][t431] | Test address allocation | Gustavo | Claimed | — |
| [Task 432][t432] | Test address imports | Gustavo | Claimed | — |
| [Task 433][t433] | Enforce sync test boundary | — | Unclaimed | — |
| [Task 434][t434] | Test transaction creation | Abdullahi | Claimed | — |
| [Task 435][t435] | Test transaction publication | Abdullahi | Claimed | — |
| [Task 436][t436] | Test transaction reads | Abdullahi | Claimed | — |
| [Task 437][t437] | Test transaction labels | Abdullahi | Claimed | — |

## Claimed Tasks

These tasks have owners but are not active yet. A claim does not authorize
implementation before every dependency and gate completes.

| Task | Description | Phase | Claimant |
| --- | --- | ---: | --- |
| [Task 331][t331] | Fence Manager lifecycle | 2 | YY |
| [Task 340][t340] | Persist database identity | 2 | YY |
| [Task 341][t341] | Define semantic account identity | 2 | YY |
| [Task 337][t337] | Restore imported-XPub derivation | 3 | YY |
| [Task 345][t345] | Persist account discovery mode | 3 | YY |
| [Task 329][t329] | Commit SQL genesis atomically | 4 | YY |

## Coordination

- Tasks 332 and 333 are YY's grandfathered parallel in-progress work. Task 332
  is shared with Gustavo because PR #1305 is stacked on PR #1306.
- Task 329 waits for Task 332 and YY's two prerequisite chains:
  `360 → 341 → 337` and `359 → 340 → 345`.
- After the grandfathered work clears, each developer may have only one task
  in progress at a time.

[pr1303]: https://github.com/btcsuite/btcwallet/pull/1303
[pr1305]: https://github.com/btcsuite/btcwallet/pull/1305
[pr1306]: https://github.com/btcsuite/btcwallet/pull/1306
[pr1307]: https://github.com/btcsuite/btcwallet/pull/1307

[t329]: tasks/329-atomic-wallet-account-creation.md
[t331]: tasks/331-manager-wallet-lifecycle.md
[t332]: tasks/332-watch-only-script-policy.md
[t333]: tasks/333-postgres-manager-harness.md
[t334]: tasks/334-shared-chain-state.md
[t335]: tasks/335-kvdb-to-sql-migration.md
[t337]: tasks/337-imported-xpub-derivation.md
[t339]: tasks/339-transaction-visibility.md
[t340]: tasks/340-durable-database-identity.md
[t341]: tasks/341-public-role-contract.md
[t342]: tasks/342-recovery-window-validation.md
[t345]: tasks/345-durable-watch-registration.md
[t350]: tasks/350-native-pgx-postgres.md
[t352]: tasks/352-kvdb-combined-passphrase-rotation.md
[t354]: tasks/354-vault-owned-secret-resolution.md
[t355]: tasks/355-transaction-size-accounting.md
[t357]: tasks/357-transaction-amount-validation.md
[t359]: tasks/359-revalidate-database-identity-design.md
[t360]: tasks/360-document-kvdb-adapter-identity.md
[t368]: tasks/368-wallet-startup-shutdown.md
[t369]: tasks/369-repeated-wallet-unlock.md
[t378]: tasks/378-account-manager-error-boundary.md
[t379]: tasks/379-manager-load-error-boundary.md
[t380]: tasks/380-address-lookup-error-boundary.md
[t400]: tasks/400-live-watch-semantics-adr.md
[t415]: tasks/415-utxo-public-integration-tests.md
[t416]: tasks/416-sqlite-benchmark-fixtures.md
[t421]: tasks/421-sql-passphrase-commit-reconciliation.md
[t424]: tasks/424-transaction-utxo-integrity-adr.md
[t425]: tasks/425-separate-transaction-authoring-wrappers.md
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
