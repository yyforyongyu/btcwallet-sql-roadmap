# Archived tasks

This directory preserves planning history for traceability. Its contents are
reference material, not active instructions. Use the root
[README.md](../README.md) for navigation, [QUEUE.md](../QUEUE.md) for the
delivery phase map, and [tasks/](../tasks/) for executable task
specifications.

## Deferred

- [Task 352](tasks/352-kvdb-combined-passphrase-rotation.md) was deferred on
  2026-09-21 as kvdb-only work, not completed or obsolete. Reactivation needs
  explicit approval for a future-cycle passphrase-containment effort.

- [Task 353](tasks/353-retire-modern-kvdb-backend.md) was deferred on
  2026-09-05, not completed or superseded. Modern kvdb remains supported for
  the current and next release; retirement is no earlier than two releases
  out and still requires verified migration, consumer cutover, and an explicit
  decision. Reactivate Task 353 rather than allocating a replacement number.

## History
- [Task 464](tasks/464-no-chain-sync-account-query-itests.md) completed on 2026-10-29.
- [Task 466](tasks/466-oldest-unused-address-itests.md) completed on 2026-10-29.
- [Task 446](tasks/446-transaction-spend-publish-scenario.md) completed on 2026-10-28.
- [Task 489](tasks/489-passphrase-shutdown-context.md) completed on 2026-10-28.
- [Task 488](tasks/488-passphrase-request-ownership.md) completed on 2026-10-28.
- [Task 454](tasks/454-postgres-transaction-isolation.md) completed on 2026-10-28.
- [Task 478](tasks/478-retain-spend-confirmation-leases.md) completed on 2026-10-28.
- [Task 405](tasks/405-psbt-funding-lease-admission.md) completed on 2026-10-26.
- [Task 483](tasks/483-preserve-labels-on-chain-replay.md) completed on 2026-10-23.
- [Task 365](tasks/365-validate-wallet-genesis-keys.md) completed on 2026-10-23.
- [Task 330](tasks/330-duplicate-wallet-create.md) completed on 2026-10-23.
- [Task 475](tasks/475-wallet-transaction-subscription.md) completed on 2026-10-23.
- [Task 484](tasks/484-align-authoring-confirmation-height.md) completed on 2026-10-22.
- [Task 389](tasks/389-runtime-account-commit-reconciliation.md) completed on 2026-10-22.
- [Task 367](tasks/367-bound-sql-genesis-secrets.md) completed on 2026-10-21.

- [Task 474](tasks/474-authored-change-watch.md) completed through
  [PR #1419](https://github.com/btcsuite/btcwallet/pull/1419), merged into
  `sql-wallet` on 2026-10-19 (UTC).

- [Task 424](tasks/424-transaction-utxo-integrity-adr.md) was superseded on
  2026-10-20. A separate integrity ADR is no longer required; [Task 410][t410]
  owns ordering evidence and correction, while [Task 454][t454] retains its
  dependency on 410. PR #1343 closed unmerged; this adds no completion.

- [Task 480](tasks/480-own-raw-imports-in-one-wallet.md) completed on 2026-10-20.
- [Task 391](tasks/391-path-bound-watch-only-xpub-accounts.md) completed on 2026-10-20.

- [Task 338](tasks/338-passphrase-result.md) was superseded on 2026-10-20.
  Caller cancellation may end observation after acceptance with an unknown
  mutation outcome; unconditional waiting is no longer required.
  [Task 488](../tasks/488-passphrase-request-ownership.md) owns request buffers
  and [Task 489](../tasks/489-passphrase-shutdown-context.md) owns shutdown
  context. PR #1421 closed unmerged; this adds no completion.

- [Task 486](tasks/486-freeze-native-sql-schema.md) was superseded on
  2026-10-19 without a replacement. Separate SQL sources and the existing
  migration loader remain; schema changes stay with Tasks 391, 478, 480, and
  481, and fresh creation and unchanged reopen remain pre-merge checks.

- [Task 479](tasks/479-mix-imported-xpub-accounts.md) completed on 2026-10-16.
- [Task 403](tasks/403-raw-import-watch-obligations.md) completed on 2026-10-16.
- [Task 431](tasks/431-address-allocation-lookup-itests.md) completed on 2026-10-16.
- [Task 481](tasks/481-persist-imported-account-address-schema.md) completed on 2026-10-15.
- [Task 388](tasks/388-runtime-account-xpub-collisions.md) completed on 2026-10-14.
- [Task 347](tasks/347-psbt-signing-authorization.md) completed on 2026-10-13.
- [Task 396](tasks/396-oldest-unused-address.md) completed on 2026-10-06.
- [Task 465](tasks/465-new-address-result-itests.md) completed on 2026-10-01.
- [Task 461](tasks/461-address-batch-itests.md) completed on 2026-10-01.
- [Task 395](tasks/395-key-only-locator-allocation.md) completed on 2026-09-30.
- [Task 344](tasks/344-bulk-address-allocation.md) completed on 2026-09-24.
- [Task 402](tasks/402-replay-live-watch-obligations.md) completed on 2026-09-24.
- [Task 442](tasks/442-signer-signing-itests.md) completed on 2026-09-22.
- [Task 430](tasks/430-account-query-itests.md) completed on 2026-09-22.

- [Task 386](tasks/386-remove-raw-wallet-secret-seams.md) was superseded on
  2026-09-21 without a replacement task. Each of Tasks 382–385 removes its
  newly unused secret seams and required tests/mocks during the consumer move;
  unchanged kvdb and deprecated callers remain supported. This is not done.

- [Task 348](tasks/348-watched-change-delivery.md) and
  [Task 408](tasks/408-direct-transaction-change-watch.md) were superseded on
  2026-09-21 by [Task 474](tasks/474-authored-change-watch.md), which owns
  the missing authored-change watch calls. Neither is complete.
- [Task 363](tasks/363-recovery-scan-work-bound.md),
  [Task 366](tasks/366-reconcile-sql-genesis-commit.md), and
  [Task 371](tasks/371-canonical-chain-frontier.md) were superseded on
  2026-09-21 without replacements. Their archived decisions remove the fixed
  aggregate recovery cap, automatic genesis reconciliation, and extra
  persisted Manager frontier; they are not completion credit.

- [Task 416](tasks/416-database-benchmark-fixtures.md) and
  [Task 418](tasks/418-list-unspent-benchmark.md) were superseded on 2026-09-19
  by [Task 471](../tasks/471-three-backend-list-unspent-benchmark.md), combining
  minimal shared setup with its first real measurement. Neither is complete.
  Tasks 449 and 373 retain their numbers after their rescan and address/script
  outcomes were split into [Task 472][split-rescan] and [Task 473][split-address],
  respectively.
- [Task 443](tasks/443-unsafe-signer-itests.md) completed on 2026-09-19.

- [Task 351](tasks/351-sql-utxo-integrity.md) was superseded on
  2026-09-18; its archived decision records the evidence and retained owner.
- [Task 361](tasks/361-kvdb-imported-xpub-derivation.md) was superseded on
  2026-09-18; its archived decision records the evidence and retained owner.
- [Task 362](tasks/362-kvdb-definitive-passphrase-result.md) was superseded on
  2026-09-18; its archived decision records the evidence and retained owner.
- [Task 409](tasks/409-transaction-input-graph.md) was superseded on
  2026-09-18; its archived decision records the evidence and retained owner.
- [Task 413](tasks/413-utxo-spend-identity.md) was superseded on
  2026-09-18; its archived decision records the evidence and retained owner.
- [Task 414](tasks/414-utxo-amount-bounds.md) was superseded on
  2026-09-18; its archived decision records the evidence and retained owner.

- [Task 412](tasks/412-transaction-invalidation.md) completed on 2026-09-17.
- [Task 399](tasks/399-zero-lookahead-persisted-targets.md) completed on 2026-09-15.
- [Task 441](tasks/441-signer-derivation-ecdh-itests.md) completed on 2026-09-15.
- [Task 331](tasks/331-manager-wallet-lifecycle.md) completed on 2026-09-15.
- [Task 468](tasks/468-activate-no-chain-sync-account-creation.md) completed on 2026-09-15.
- [Task 390](tasks/390-custom-scope-account-creation.md) completed on 2026-09-15.
- [Task 346](tasks/346-psbt-funding-metadata.md) completed on 2026-09-15.
- [Task 343](tasks/343-exact-account-creation.md) completed on 2026-09-15.
- [Task 455](tasks/455-exact-wallet-sync-status.md) completed on 2026-09-12.
- [Task 452](tasks/452-key-only-recovery-derivation.md) completed on 2026-09-12.
- [Task 393](tasks/393-key-only-receiving-rejection.md) completed on 2026-09-11.
- [Task 392](tasks/392-discovery-mode-account-creation.md) completed on 2026-09-11.
- [Task 345](tasks/345-durable-watch-registration.md) completed on 2026-09-11.
- [Task 459](tasks/459-manager-startup-wallet-assembly.md) completed on 2026-09-10.

- [Task 364](tasks/364-durable-watch-registration.md) was superseded on
  2026-09-10: fail-stop and reconstruction eliminate watch-delivery persistence.
  [Task 402](tasks/402-replay-live-watch-obligations.md) retains registration
  from existing Wallet facts.
- [Task 378](tasks/378-account-manager-error-boundary.md) completed on 2026-09-10.
- [Task 458](tasks/458-manager-operation-admission.md) completed on 2026-09-08.
- [Task 404](tasks/404-transaction-input-amounts.md) completed on 2026-09-08.
- [Task 357](tasks/357-transaction-amount-validation.md) completed on 2026-09-05.

- [Task 427](tasks/427-controller-lifecycle-credential-itests.md) completed
  through [PR #1349](https://github.com/btcsuite/btcwallet/pull/1349) on
  2026-09-04 (UTC). Task 331 owns later lifecycle caller migration.
- [Task 349](tasks/349-transaction-invalidation.md) was superseded on
  2026-09-05 because SQL already has the required internal branch invalidation;
  Task 412 owns only the remaining maintained `TxWriter` exposure.
- [Task 381](tasks/381-recovery-adapter-identity.md) was superseded on
  2026-09-05 because no maintained public consumer needs the identity-only
  refactor; the current shape remains until a concrete need is demonstrated.
- [Task 336](tasks/336-test-cleanup.md) was superseded on 2026-09-05 because
  deleting duplicate tests has no independent outcome. Required test updates
  stay with their production changes, now in Tasks 382–385 rather than the
  superseded standalone cleanup Task 386.
- [Task 368](tasks/368-wallet-startup-shutdown.md) was superseded on
  2026-09-05 when its public lifecycle cutover was absorbed into
  [Task 331](tasks/331-manager-wallet-lifecycle.md) as part of one atomic
  Manager lifecycle outcome; PR #1328 was retargeted to Task 331.
- [Task 340](tasks/340-durable-database-identity.md) completed on 2026-09-05.
- [Task 456](tasks/456-manager-runtime-configuration.md) completed on 2026-09-04.
- [Task 401](tasks/401-live-tip-chain-watch.md) completed on 2026-09-03.
- [Task 387](tasks/387-account-creation-identity-adr.md) completed on 2026-09-03.
- [Task 437](tasks/437-transaction-label-itests.md) completed on 2026-09-03.
- [Task 467](tasks/467-indeterminate-commit-identity.md) completed on 2026-09-02.
- [Task 334](tasks/334-shared-chain-state.md) completed on 2026-09-02.
- [Task 421](tasks/421-sql-passphrase-commit-reconciliation.md) completed on 2026-08-31.
- [Task 394](tasks/394-key-only-transaction-ownership.md) was superseded before
  implementation on 2026-08-30 when `NoChainSync` was defined as chain-sync
  policy rather than an ownership rule, eliminating the old outcome; Task 395
  owns only the separate managed-child allocation.
- [Task 436](tasks/436-transaction-read-itests.md) completed on 2026-08-30.
- [Task 422](tasks/422-manager-load-cancellation.md) was superseded before
  implementation on 2026-08-28 when the roadmap adopted Task 368's planned
  removal of public Manager loading; Task 459 owns aggregate startup assembly.
- [Task 429](tasks/429-account-mutation-itests.md) completed on 2026-08-26.
- [Task 379](tasks/379-manager-load-error-boundary.md) completed on 2026-08-25.
- [Task 380](tasks/380-address-lookup-error-boundary.md) completed on 2026-08-25.
- [Task 337](tasks/337-imported-xpub-derivation.md) completed on 2026-08-25.
- [Task 453](tasks/453-serialize-manager-wallet-publication.md) completed on 2026-08-25.
- [Task 425](tasks/425-separate-transaction-authoring-wrappers.md) completed on 2026-08-22.
- [Task 377](tasks/377-account-manager-wallet-dto.md) completed on 2026-08-21.
- [Task 435](tasks/435-transaction-publication-itests.md) completed on 2026-08-21.
- [Task 339](tasks/339-transaction-visibility.md) completed on 2026-08-21.
- [Task 355](tasks/355-transaction-size-accounting.md) completed on 2026-08-20.
- [Task 400](tasks/400-live-watch-semantics-adr.md) completed on 2026-08-20.
- [Task 359](tasks/359-revalidate-database-identity-design.md) completed on 2026-08-20.
- [Task 342](tasks/342-recovery-window-validation.md) completed on 2026-08-18.
- [Task 434](tasks/434-transaction-creation-itests.md) completed on 2026-08-15.
- [Task 332](tasks/332-watch-only-script-policy.md) completed on 2026-08-15.
- [Task 433](tasks/433-wallet-sync-test-boundary.md) completed on 2026-08-13.
- [Task 341](tasks/341-public-role-contract.md) completed on 2026-08-13.
- [Task 360](tasks/360-document-kvdb-adapter-identity.md) completed on 2026-08-12.
- [Task 415](tasks/415-utxo-public-integration-tests.md) completed on 2026-08-11.
- [Task 333](tasks/333-postgres-manager-harness.md) completed on 2026-08-10.

- [Tasks 278-328](tasks/) were consolidated and deduplicated on 2026-08-07.
  [Tasks 329-358](../tasks/) replace them.
- [Task 1000](tasks/1000-complete-btcwallet-sql-role-readiness.md) was archived
  on 2026-08-07 as reference analysis. The [queue](../QUEUE.md) and
  [active tasks](../tasks/) replace it.
- [Task 10001](tasks/10001-resumable-per-wallet-rescan.md) was archived on
  2026-08-07. Its decision remains a scope rule; it has no executable
  replacement.
- [Task 10002](tasks/10002-controller-request-completion.md) was archived on
  2026-08-07. Its finding was absorbed into
  [Task 338](tasks/338-passphrase-result.md), later superseded on 2026-10-20.
  The accepted decision does not restore a result registry.

[split-rescan]: ../tasks/472-multi-wallet-rescan-isolation-scenario.md
[split-address]: ../tasks/473-extract-legacy-address-script-state.md

[t410]: ../tasks/410-transaction-graph-admission.md
[t454]: ../tasks/454-postgres-transaction-isolation.md
