# Active Tasks

Claim early, own the development, and ship. After a task merges, remove it
here and archive its task file.

## Active Tasks

Dependency-complete Phase 1 wallet gaps and Phase 2 tests and benchmarks are
the current development focus and may proceed in parallel. No claimant means
unclaimed, a claimant without a PR means claimed, and a claimant with a PR
means in progress. Later delivery phases remain deferred unless recorded here
as already started or explicitly approved.

| Task | Description | Claimant | PR |
| --- | --- | --- | --- |
| [Task 332][t332] | Define watch-only policy | YY + Gustavo | [#1320][pr1320] |
| [Task 334][t334] | Define shared-chain ownership | YY | — |
| [Task 339][t339] | Define transaction visibility | — | — |
| [Task 341][t341] | Define semantic account identity | YY | [#1313][pr1313] |
| [Task 342][t342] | Bound recovery windows | — | — |
| [Task 350][t350] | Use native PostgreSQL driver | YY | [#1321][pr1321] |
| [Task 352][t352] | Rotate kvdb passphrases atomically | — | — |
| [Task 355][t355] | Correct transaction size accounting | — | — |
| [Task 357][t357] | Validate transaction amounts | — | — |
| [Task 359][t359] | Revalidate database identity | YY | [#1319][pr1319] |
| [Task 368][t368] | Serialize Wallet lifecycle | — | — |
| [Task 369][t369] | Reject repeated unlocks | — | — |
| [Task 378][t378] | Translate account errors | — | — |
| [Task 379][t379] | Translate load errors | — | — |
| [Task 380][t380] | Translate address lookup errors | — | — |
| [Task 381][t381] | Hide recovery adapter identity | — | — |
| [Task 400][t400] | Define live-watch semantics | YY | — |
| [Task 416][t416] | Build SQLite benchmark fixtures | YY | — |
| [Task 421][t421] | Reconcile passphrase commits | — | — |
| [Task 424][t424] | Define transaction integrity | — | — |
| [Task 425][t425] | Isolate transaction authoring | — | — |
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
| [Task 438][t438] | Test PSBT packet transforms | — | — |
| [Task 439][t439] | Test PSBT funding | — | — |
| [Task 440][t440] | Test PSBT signing | — | — |
| [Task 441][t441] | Test derivation and ECDH | — | — |
| [Task 442][t442] | Test public signing | — | — |
| [Task 443][t443] | Test unsafe key extraction | — | — |
| [Task 451][t451] | Test supported chain releases | — | — |

## Claimed Tasks

These tasks have owners but are not active yet. A claim does not authorize
implementation before every dependency and gate completes.

| Task | Description | Phase | Claimant | PR |
| --- | --- | ---: | --- | --- |
| [Task 331][t331] | Fence Manager lifecycle | 1 | YY | — |
| [Task 340][t340] | Persist database identity | 1 | YY | — |
| [Task 337][t337] | Restore imported-XPub derivation | 1 | YY | — |
| [Task 345][t345] | Persist account discovery mode | 1 | YY | — |
| [Task 329][t329] | Commit SQL genesis atomically | 1 | YY | — |

## Coordination

- Tasks 332, 341, 359, and 433 are YY's grandfathered parallel in-progress
  work. Task 332 was shared with Gustavo; #1306 has merged and the remaining
  work is #1320. YY starts no additional task until only one remains in
  progress.
- Task 329 waits for Task 332 and YY's two prerequisite chains:
  `360 → 341 → 337` and `359 → 340 → 345`.
- Every other developer may have only one task in progress at a time.

[pr1305]: https://github.com/btcsuite/btcwallet/pull/1305
[pr1313]: https://github.com/btcsuite/btcwallet/pull/1313
[pr1314]: https://github.com/btcsuite/btcwallet/pull/1314

[t329]: tasks/329-atomic-wallet-account-creation.md
[t331]: tasks/331-manager-wallet-lifecycle.md
[t332]: tasks/332-watch-only-script-policy.md
[t334]: tasks/334-shared-chain-state.md
[t337]: tasks/337-imported-xpub-derivation.md
[t339]: tasks/339-transaction-visibility.md
[t340]: tasks/340-durable-database-identity.md
[t341]: tasks/341-public-role-contract.md
[t342]: tasks/342-recovery-window-validation.md
[t345]: tasks/345-durable-watch-registration.md
[t350]: tasks/350-native-pgx-postgres.md
[t352]: tasks/352-kvdb-combined-passphrase-rotation.md
[t355]: tasks/355-transaction-size-accounting.md
[t357]: tasks/357-transaction-amount-validation.md
[t359]: tasks/359-revalidate-database-identity-design.md
[t368]: tasks/368-wallet-startup-shutdown.md
[t369]: tasks/369-repeated-wallet-unlock.md
[t378]: tasks/378-account-manager-error-boundary.md
[t379]: tasks/379-manager-load-error-boundary.md
[t380]: tasks/380-address-lookup-error-boundary.md
[t381]: tasks/381-recovery-adapter-identity.md
[t400]: tasks/400-live-watch-semantics-adr.md
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
[t438]: tasks/438-psbt-decoration-combination-itests.md
[t439]: tasks/439-psbt-funding-itests.md
[t440]: tasks/440-psbt-signing-finalization-itests.md
[t441]: tasks/441-signer-derivation-ecdh-itests.md
[t442]: tasks/442-signer-signing-itests.md
[t443]: tasks/443-unsafe-signer-itests.md
[t451]: tasks/451-chain-version-itest-matrix.md
[pr1320]: https://github.com/btcsuite/btcwallet/pull/1320
[pr1321]: https://github.com/btcsuite/btcwallet/pull/1321
[pr1319]: https://github.com/btcsuite/btcwallet/pull/1319
[pr1316]: https://github.com/btcsuite/btcwallet/pull/1316
