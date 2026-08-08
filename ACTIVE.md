# Active Tasks

Claim a dependency-ready task early, own the development, and ship it. After
it merges, remove it here and archive its task file.

## Claimed Tasks

| Task | Description | Owner | PR |
| --- | --- | --- | --- |
| [Task 332][t332] | Define watch-only policy | YY + Gustavo | [#1305][pr1305], [#1306][pr1306] |
| [Task 333][t333] | Add PostgreSQL test harness | YY | [#1303][pr1303] |
| [Task 415][t415] | Test public UTXO contracts | Abdullahi | [#1307][pr1307] |

## Assigned Tasks

Assignments reserve future ownership. They are not active claims and do not
permit work before every dependency completes.

| Task | Description | Phase | Owner |
| --- | --- | ---: | --- |
| [Task 334][t334] | Define shared-chain ownership | 1 | YY |
| [Task 350][t350] | Use native PostgreSQL driver | 1 | YY |
| [Task 359][t359] | Revalidate database identity | 1 | YY |
| [Task 360][t360] | Document kvdb adapter identity | 1 | YY |
| [Task 427][t427] | Test Controller credentials | 1 | Gustavo |
| [Task 428][t428] | Test resync and rescan | 1 | Gustavo |
| [Task 429][t429] | Test account mutations | 1 | Gustavo |
| [Task 430][t430] | Test account queries | 1 | Gustavo |
| [Task 431][t431] | Test address allocation | 1 | Gustavo |
| [Task 432][t432] | Test address imports | 1 | Gustavo |
| [Task 434][t434] | Test transaction creation | 1 | Abdullahi |
| [Task 435][t435] | Test transaction publication | 1 | Abdullahi |
| [Task 436][t436] | Test transaction reads | 1 | Abdullahi |
| [Task 437][t437] | Test transaction labels | 1 | Abdullahi |
| [Task 331][t331] | Fence Manager lifecycle | 2 | YY |
| [Task 340][t340] | Persist database identity | 2 | YY |
| [Task 341][t341] | Define semantic account identity | 2 | YY |
| [Task 337][t337] | Restore imported-XPub derivation | 3 | YY |
| [Task 345][t345] | Persist account discovery mode | 3 | YY |
| [Task 329][t329] | Commit SQL genesis atomically | 4 | YY |

## Coordination

- Tasks 332 and 333 are YY's grandfathered parallel claims. Task 332 is shared
  with Gustavo because PR #1305 is stacked on PR #1306.
- Task 329 waits for Task 332 and YY's two prerequisite chains:
  `360 → 341 → 337` and `359 → 340 → 345`.
- After the grandfathered claims clear, each developer may claim only one task
  at a time.

[pr1303]: https://github.com/btcsuite/btcwallet/pull/1303
[pr1305]: https://github.com/btcsuite/btcwallet/pull/1305
[pr1306]: https://github.com/btcsuite/btcwallet/pull/1306
[pr1307]: https://github.com/btcsuite/btcwallet/pull/1307

[t329]: tasks/329-atomic-wallet-account-creation.md
[t331]: tasks/331-manager-wallet-lifecycle.md
[t332]: tasks/332-watch-only-script-policy.md
[t333]: tasks/333-postgres-manager-harness.md
[t334]: tasks/334-shared-chain-state.md
[t337]: tasks/337-imported-xpub-derivation.md
[t340]: tasks/340-durable-database-identity.md
[t341]: tasks/341-public-role-contract.md
[t345]: tasks/345-durable-watch-registration.md
[t350]: tasks/350-native-pgx-postgres.md
[t359]: tasks/359-revalidate-database-identity-design.md
[t360]: tasks/360-document-kvdb-adapter-identity.md
[t415]: tasks/415-utxo-public-integration-tests.md
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
