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
placement. Open dependencies block direct merge, not implementation in a
recorded PR stack.

| Task | Description | Type | Claimant | PR |
| --- | --- | --- | --- | --- |
| [Task 340][t340] | Persist database identity | Bug fix | YY | [#1345][pr1345] |
| [Task 357][t357] | Validate transaction amounts | Bug fix | Abdullahi | [#1348][pr1348] |
| [Task 378][t378] | Translate AccountManager backend errors | Bug fix | Gustavo | [#1358][pr1358] |
| [Task 431][t431] | Test address allocation | Test | Gustavo | — |
| [Task 432][t432] | Test address imports | Test | Gustavo | — |
| [Task 455][t455] | Report exact wallet sync status | Bug fix | Gustavo | — |
| [Task 458][t458] | Drain Wallet calls during terminal stop | Bug fix | YY | [#1354][pr1354] |

## Blocked Phase 1 Tasks

These Phase 1 tasks have open hard dependencies and therefore cannot merge
yet. They may still advance in recorded stacked PRs; claimant and PR columns
show that work without making the task dependency-ready. Task files own all
dependency edges, and `QUEUE.md` owns their delivery phase.

| Task | Description | Type | Claimant | PR |
| --- | --- | --- | --- | --- |
| [Task 329][t329] | Commit SQL genesis atomically | Bug fix | YY | — |
| [Task 331][t331] | Make Manager lifecycle terminal | Bug fix | YY | — |
| [Task 343][t343] | Create next-or-exact root-derived accounts | Gap fix | — | — |
| [Task 344][t344] | Allocate receiving-address batches atomically | Gap fix | — | — |
| [Task 345][t345] | Persist account NoChainSync state | Gap fix | YY | — |
| [Task 346][t346] | Preserve PSBT funding metadata | Bug fix | Abdullahi | [#1359][pr1359] |
| [Task 347][t347] | Authorize explicit PSBT signatures | Bug fix | Abdullahi | — |
| [Task 348][t348] | Watch PSBT change before return | Bug fix | Gustavo | — |
| [Task 356][t356] | Defer change-script allocation | Bug fix | Gustavo | — |
| [Task 358][t358] | Make Manager creation cancellable | Gap fix | — | — |
| [Task 364][t364] | Persist live-watch obligations | Gap fix | — | — |
| [Task 390][t390] | Create accounts in custom scopes | Gap fix | — | — |
| [Task 392][t392] | Carry NoChainSync through account creation | Gap fix | YY | — |
| [Task 393][t393] | Guard NoChainSync receiving APIs | Gap fix | YY | — |
| [Task 395][t395] | Allocate a durable NoChainSync child | Gap fix | YY | — |
| [Task 396][t396] | Return the oldest unused address | Bug fix | Gustavo | — |
| [Task 399][t399] | Isolate persisted recovery targets | Bug fix | — | — |
| [Task 402][t402] | Replay live-watch obligations | Gap fix | YY | — |
| [Task 404][t404] | Validate transaction input amounts | Bug fix | Abdullahi | [#1357][pr1357] |
| [Task 405][t405] | Reserve PSBT funding inputs | Gap fix | Gustavo | — |
| [Task 406][t406] | Clean up failed PSBT funding leases | Gap fix | Gustavo | — |
| [Task 407][t407] | Authorize PSBT auto-finalization | Bug fix | Abdullahi | — |
| [Task 408][t408] | Watch direct transaction change | Bug fix | Gustavo | — |
| [Task 368][t368] | Make Wallet lifecycle Manager-internal | Bug fix | YY | [#1328][pr1328] |
| [Task 452][t452] | Exclude NoChainSync accounts from recovery | Gap fix | YY | — |
| [Task 459][t459] | Assemble startup Wallets | Gap fix | YY | [#1356][pr1356] |

## Claimed Tasks

These claimed tasks sit outside the current Phase 1 delivery phase and are not
listed under `Active Tasks` by the placement rule above. When phase priority
permits implementation, a claimed task may proceed as a stacked PR, but it
cannot merge before every hard dependency and gate completes.

| Task | Description | Type | Phase | Claimant | PR |
| --- | --- | --- | ---: | --- | --- |
| [Task 330][t330] | Reject duplicate wallet creation | Bug fix | 3 | YY | — |
| [Task 350][t350] | Use native PostgreSQL driver | Refactor | 3 | YY | [#1321][pr1321] |
| [Task 365][t365] | Validate wallet genesis key identities | Bug fix | 3 | YY | — |
| [Task 367][t367] | Bound SQL genesis secrets | Bug fix | 3 | YY | — |
| [Task 370][t370] | Prepare ordered Manager chain delivery | Refactor | 3 | YY | — |
| [Task 372][t372] | Prepare atomic multi-Wallet rollback | Bug fix | 3 | YY | — |
| [Task 397][t397] | Define semantic-account recovery targets | Design | 3 | YY | — |
| [Task 398][t398] | Accept semantic account recovery targets | Gap fix | 3 | YY | — |
| [Task 423][t423] | Cut over SQL Manager chain notification ownership | Refactor | 3 | YY | — |
| [Task 424][t424] | Define transaction integrity ownership | Design | 3 | YY | [#1343][pr1343] |
| [Task 427][t427] | Test Controller credentials | Test | 2 | Gustavo | [#1349][pr1349] |
| [Task 428][t428] | Test resync and rescan | Test | 2 | Gustavo | — |
| [Task 430][t430] | Test account queries | Test | 2 | Gustavo | — |
| [Task 454][t454] | Make PostgreSQL isolation explicit | Bug fix | 3 | YY | — |
| [Task 457][t457] | Fence process-local Managers | Gap fix | 3 | YY | — |

## Coordination

Task 357 and PR #1348 may retain their accepted `wallet/txauthor`,
`wallet/txrules`, root-module wiring, and Wallet changes as a narrow historical
exception. They must drop deprecated Wallet behavior and
`wallet/deprecated_test.go` unless a separate roadmap exception is approved.

Task 404 and draft PR #1357 own only the Wallet-local input-source layer. Drop
its `wallet/txauthor` changes, keep Task 357's arithmetic private, and restack
the PR on Task 357's final base and result before merge.

Task 334 and PR #1323 must adopt Manager public lifecycle authority, private
Wallet `start`/`stop` mechanics, Wallet-owned call draining, the complete
active-Wallet set, Store-close-last ordering, and separate live-sync and
historical-rescan state before merge.

Task 368 and draft PR #1328 must be rewritten around private Wallet
`start`/`stop`, Wallet-owned call draining, and Task 368's final public-surface
cutover. It may proceed at the top of the Task 331 -> Task 358 -> Task 368
stack. Keep it rebased on the prerequisite work and merge the stack bottom-up.

Task 427 and PR #1349 may proceed as a stacked dependent of Task 331 because
the durable-reopen case consumes final Manager Stop and fresh Manager Start
behavior. Keep it rebased on Task 331 and merge it only after Task 331.

Task 350 and PR #1321 may resume as a stacked dependent of Task 454 when Phase
3 work is prioritized. Keep it rebased on Task 454 and merge it only after
Task 454.

Task 424 and PR #1343 are paused after the transaction-integrity branch moved
to non-blocking Phase 3. Resume them when Phase 3 becomes current or through
an explicit owner-approved priority exception.

Task 421 and PR #1341 may finish as an already-started Phase 3 exception. Its
rephasing preserves YY's ownership and does not authorize another concurrent
implementation.

[t329]: tasks/329-atomic-wallet-account-creation.md
[t330]: tasks/330-duplicate-wallet-create.md
[t331]: tasks/331-manager-wallet-lifecycle.md
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
[t364]: tasks/364-durable-watch-registration.md
[t365]: tasks/365-validate-wallet-genesis-keys.md
[t366]: tasks/366-reconcile-sql-genesis-commit.md
[t367]: tasks/367-bound-sql-genesis-secrets.md
[t368]: tasks/368-wallet-startup-shutdown.md
[t370]: tasks/370-chain-source-delivery.md
[t372]: tasks/372-multi-wallet-rollback.md
[t378]: tasks/378-account-manager-error-boundary.md
[t388]: tasks/388-runtime-account-xpub-collisions.md
[t389]: tasks/389-runtime-account-commit-reconciliation.md
[t390]: tasks/390-custom-scope-account-creation.md
[t391]: tasks/391-path-bound-watch-only-xpub-accounts.md
[t392]: tasks/392-discovery-mode-account-creation.md
[t393]: tasks/393-key-only-receiving-rejection.md
[t395]: tasks/395-key-only-locator-allocation.md
[t396]: tasks/396-oldest-unused-address.md
[t397]: tasks/397-semantic-recovery-targets-adr.md
[t398]: tasks/398-semantic-account-recovery-targets.md
[t399]: tasks/399-zero-lookahead-persisted-targets.md
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
[t423]: tasks/423-manager-chain-source-cutover.md
[t424]: tasks/424-transaction-utxo-integrity-adr.md
[t427]: tasks/427-controller-lifecycle-credential-itests.md
[t428]: tasks/428-controller-sync-info-itests.md
[t430]: tasks/430-account-query-itests.md
[t431]: tasks/431-address-allocation-lookup-itests.md
[t432]: tasks/432-address-import-script-itests.md
[t452]: tasks/452-key-only-recovery-derivation.md
[t454]: tasks/454-postgres-transaction-isolation.md
[t455]: tasks/455-exact-wallet-sync-status.md
[t457]: tasks/457-process-local-manager-lease.md
[t458]: tasks/458-manager-operation-admission.md
[t459]: tasks/459-manager-startup-wallet-assembly.md
[t460]: tasks/460-sequential-account-invalid-children.md
[pr1321]: https://github.com/btcsuite/btcwallet/pull/1321
[pr1328]: https://github.com/btcsuite/btcwallet/pull/1328
[pr1343]: https://github.com/btcsuite/btcwallet/pull/1343
[pr1345]: https://github.com/btcsuite/btcwallet/pull/1345
[pr1348]: https://github.com/btcsuite/btcwallet/pull/1348
[pr1349]: https://github.com/btcsuite/btcwallet/pull/1349
[pr1354]: https://github.com/btcsuite/btcwallet/pull/1354
[pr1356]: https://github.com/btcsuite/btcwallet/pull/1356
[pr1358]: https://github.com/btcsuite/btcwallet/pull/1358
[pr1357]: https://github.com/btcsuite/btcwallet/pull/1357
[pr1359]: https://github.com/btcsuite/btcwallet/pull/1359
