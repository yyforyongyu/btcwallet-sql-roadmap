# Task delivery phases

The roadmap contains **126 tasks**, with **114 remaining open**. Open tasks are
grouped by delivery phase; completed roadmap tasks move to the `Done` table,
which records the PR that completed each task.

Every roadmap task appears exactly once below. The phases are thematic delivery
stages, not computed dependency depths or strict phase barriers. Individual
task files remain the authority for each primary type, every hard dependency
edge, and every implementation gate. A task never becomes implementable merely
because it is listed in an earlier phase.

Phase 1 is the current delivery focus. Do not deliberately start Phase 2 or
later work while Phase 1 remains current, except for already-started work
recorded in [ACTIVE.md](ACTIVE.md) or an explicit owner-approved exception.

`Open dependencies` lists only hard dependencies whose status is `open`.
Completed dependency edges remain recorded in the owning task file and are
intentionally omitted from the table.

The delivery stages are:

1. lnd integration gaps and their open dependency closure;
2. remaining known wallet gaps and bug fixes;
3. integration tests and benchmarks;
4. legacy migration and kvdb retirement; and
5. Signer and Vault hardening.

[ACTIVE.md](ACTIVE.md) separately owns the current focus, claims, in-progress
PRs, and coordination.


## Phase 1 — lnd integration · 53 open

Close the btcwallet-side contracts needed to exercise the wallet from lnd.
This phase contains the direct lnd integration tasks and every open hard
dependency in their transitive closure. Any lnd-repository change requires a
separately accepted task.

| Task | Status | Component | Type | Outcome | Open dependencies |
| --- | --- | --- | --- | --- | --- |
| [Task 329][t329] | open | Manager SQL genesis and SQL Store | Bug fix | Commit SQL wallet genesis atomically | 337, 345 |
| [Task 330][t330] | open | Manager | Bug fix | Reject duplicate wallet creation | 331 |
| [Task 331][t331] | open | Manager | Bug fix | Fence Manager admission and Store closure | 368 |
| [Task 334][t334] | open | SQL wallet architecture documentation | Design | Supersede ADR 0002 with SQL shared-chain ownership | — |
| [Task 337][t337] | open | Wallet and Signer public derivation | Bug fix | Restore SQL imported-XPub public derivation | — |
| [Task 338][t338] | open | Controller | Bug fix | Latch accepted SQL passphrase results | 331, 368, 421 |
| [Task 340][t340] | open | Manager and SQL Store | Bug fix | Bind SQL databases to schema and network identity | 359 |
| [Task 343][t343] | open | AccountManager root-derived creation and SQL AccountStore CAS | Gap fix | Create next-or-exact root-derived accounts | 377, 388, 392, 393, 394, 452 |
| [Task 344][t344] | open | AddressManager receiving batches and SQL AddressStore mutation | Gap fix | Allocate receiving-address batches atomically | 364, 393, 402 |
| [Task 345][t345] | open | SQL account schema and AccountStore persistence | Gap fix | Persist SQL account discovery mode | 340 |
| [Task 346][t346] | open | PsbtManager packet validation and transformation | Bug fix | Preserve PSBT metadata during funding | 357, 404 |
| [Task 347][t347] | open | PsbtManager and Signer | Bug fix | Authorize explicit PSBT signatures | 346, 390 |
| [Task 348][t348] | open | PsbtManager and AddressManager | Bug fix | Watch PSBT change before returning funding | 344, 406, 425 |
| [Task 349][t349] | open | TxStore internal invalidation workflow | Gap fix | Implement idempotent transaction invalidation | 339, 409, 410, 411 |
| [Task 356][t356] | open | TxCreator and `txauthor` | Bug fix | Defer change-script allocation until viable | 355, 357, 404 |
| [Task 357][t357] | open | `txauthor` and TxCreator | Bug fix | Validate transaction output and fee arithmetic | — |
| [Task 358][t358] | open | Manager.Create | Gap fix | Make Manager creation cancellable | 331, 366, 367 |
| [Task 364][t364] | open | SQL watch-generation and obligation schema | Gap fix | Persist ordered SQL live-watch obligations | 345, 400 |
| [Task 366][t366] | open | Manager and SQL Store genesis result | Bug fix | Reconcile ambiguous SQL genesis commits | 329 |
| [Task 367][t367] | open | Manager and WalletVault genesis preparation | Bug fix | Bound SQL genesis secret lifetime | 329 |
| [Task 368][t368] | open | Manager and Wallet runtime lifecycle | Bug fix | Make Manager own Wallet lifecycle | 453 |
| [Task 377][t377] | open | AccountManager public result contract | Gap fix | Make AccountManager results wallet-owned | — |
| [Task 378][t378] | open | AccountManager public error boundary | Bug fix | Translate AccountManager backend errors | — |
| [Task 379][t379] | open | Manager Load public error boundary | Bug fix | Translate Manager load not-found errors | — |
| [Task 380][t380] | open | AddressManager GetAddressInfo error boundary | Bug fix | Translate address lookup errors | — |
| [Task 387][t387] | open | ADR 0012 and ADR 0013 account-creation contract | Design | Define custom and path-bound account identity | — |
| [Task 388][t388] | open | SQL AccountStore account-creation admission | Bug fix | Reject colliding runtime account XPub identities | 387 |
| [Task 389][t389] | open | AccountManager and SQL AccountStore account result reader | Bug fix | Reconcile ambiguous runtime account commits | 343 |
| [Task 390][t390] | open | AccountManager custom scope and SQL key-scope/account mutation | Gap fix | Create root-derived accounts in custom scopes | 343, 387, 389 |
| [Task 391][t391] | open | AccountManager path-bound identity and SQL AccountStore schema | Gap fix | Create exact path-bound watch-only XPub accounts | 337, 387, 388, 389, 390 |
| [Task 392][t392] | open | AccountManager and SQL AccountStore create inputs | Gap fix | Consume discovery mode in existing account creation | 345, 387 |
| [Task 393][t393] | open | AddressManager receiving entry points and AddressStore account lookup | Gap fix | Reject receiving allocation for key-only accounts | 345 |
| [Task 394][t394] | open | AddressStore ownership resolution and transaction ingestion | Gap fix | Exclude key-only locators from wallet ownership | 345 |
| [Task 395][t395] | open | AddressManager key allocation and SQL branch-counter mutation | Gap fix | Allocate one durable key-only locator | 393, 394, 399 |
| [Task 396][t396] | open | AddressManager count-one semantics | Bug fix | Make NewAddress return the oldest unused address | 344, 348, 408 |
| [Task 399][t399] | open | Syncer recovery initialization and account-scoped Store reads | Bug fix | Isolate persisted targets in zero-lookahead recovery | 342, 345 |
| [Task 401][t401] | open | chain interface and Neutrino/btcd/bitcoind adapters | Gap fix | Add an idempotent live-tip-only chain watch | 400 |
| [Task 402][t402] | open | Controller/Syncer watch initialization and delivery completion | Gap fix | Replay live-watch obligations through the Syncer | 364, 401 |
| [Task 404][t404] | open | `txauthor` input sources and TxCreator source adapters | Bug fix | Validate transaction input-source amounts | 357 |
| [Task 405][t405] | open | PsbtManager and UTXO Store funding admission | Gap fix | Reserve PSBT funding inputs atomically | 346, 356, 404 |
| [Task 406][t406] | open | UTXO Store and PsbtManager cleanup | Gap fix | Release PSBT funding leases by exact token | 405 |
| [Task 407][t407] | open | PsbtManager FinalizePsbt | Bug fix | Authorize PSBT auto-finalization | 347 |
| [Task 408][t408] | open | TxCreator and AddressManager | Bug fix | Watch direct transaction change before return | 344, 356, 425 |
| [Task 409][t409] | open | SQL schema and TxStore ingestion | Gap fix | Normalize transaction input edges | 340, 424 |
| [Task 410][t410] | open | Wallet/Store transaction graph admission | Bug fix | Coordinate transaction graph mutations | 334, 409 |
| [Task 411][t411] | open | TxStore activation preflight and direct/batch ingestion | Bug fix | Reject children of terminal transactions | 339, 409, 410 |
| [Task 412][t412] | open | TxWriter public contract, TxPublisher, and transaction graph admission | Gap fix | Expose invalidation with publication ordering | 349 |
| [Task 421][t421] | open | WalletVault and SQL wallet-secret persistence | Bug fix | Reconcile SQL passphrase commits | — |
| [Task 422][t422] | open | Manager.Load | Gap fix | Make Manager loading cancellable | 331, 368, 453 |
| [Task 424][t424] | open | ADR 0006 transaction and UTXO integrity contract | Design | Define transaction and UTXO integrity ownership | — |
| [Task 425][t425] | open | Wallet transaction-authoring wrapper boundary | Refactor | Isolate shared transaction authoring from wrapper watch and lease effects | — |
| [Task 452][t452] | open | Syncer positive-lookahead account admission and enumeration | Gap fix | Exclude key-only accounts from recovery derivation | 345 |
| [Task 453][t453] | open | Manager Wallet cache publication | Bug fix | Serialize Manager Wallet publication | — |

## Phase 2 — remaining wallet gaps and bugs · 19 open

Fix the remaining known gaps and bugs in maintained wallet APIs, storage,
lifecycle, synchronization, transactions, and recovery. Keep their enabling
Design and Refactor work here; schedule other non-Signer/Vault cleanup after
ready Bug fix and Gap fix work.

| Task | Status | Component | Type | Outcome | Open dependencies |
| --- | --- | --- | --- | --- | --- |
| [Task 350][t350] | open | PostgreSQL Store | Refactor | Use native pgx for PostgreSQL storage | 454 |
| [Task 351][t351] | open | SQL UTXO and lease schema | Bug fix | Bind SQL UTXOs to wallet tenancy | 340, 424 |
| [Task 352][t352] | open | LegacyWalletVault and waddrmgr passphrase rotation | Bug fix | Rotate kvdb passphrases atomically | — |
| [Task 361][t361] | open | modern kvdb public derivation adapter | Gap fix | Support kvdb imported-XPub derivation | 337 |
| [Task 362][t362] | open | Controller and LegacyWalletVault | Bug fix | Return a definitive kvdb passphrase result | 338, 352 |
| [Task 363][t363] | open | recovery horizon derivation | Bug fix | Bound aggregate recovery derivation work | 342 |
| [Task 365][t365] | open | Manager creation preflight | Bug fix | Validate wallet genesis key identities | — |
| [Task 370][t370] | open | Manager chain coordinator and Wallet processors | Refactor | Prepare ordered Manager chain delivery | 331, 334, 371 |
| [Task 371][t371] | open | SQL Store canonical chain state | Bug fix | Serialize canonical chain-frontier mutation | 334, 340 |
| [Task 372][t372] | open | Manager chain coordinator and SQL Store rollback | Bug fix | Prepare atomic multi-Wallet rollback | 349, 370, 371 |
| [Task 381][t381] | open | RecoveryState public contract and internal Syncer adapter | Refactor | Hide recovery adapter identity | — |
| [Task 397][t397] | open | ADR 0004 targeted-recovery contract | Design | Define semantic-account recovery targets | 400 |
| [Task 398][t398] | open | Controller and Syncer recovery account selection | Gap fix | Accept semantic account recovery targets | 345, 397, 399, 452 |
| [Task 403][t403] | open | AddressManager SQL raw-import mutation | Bug fix | Persist raw imports with live-watch obligations | 364, 402 |
| [Task 413][t413] | open | SQL UTXO and normalized-input schema | Bug fix | Constrain SQL UTXOs to exact spending inputs | 351, 409, 410 |
| [Task 414][t414] | open | SQL UTXO schema | Bug fix | Bound persisted SQL UTXO amounts | 351 |
| [Task 423][t423] | open | SQL Manager chain source adapter and coordinator activation | Refactor | Cut over SQL Manager chain notification ownership | 368, 370, 372, 412, 422 |
| [Task 426][t426] | open | SQL TxStore ingestion amount validation | Bug fix | Validate persisted transaction output aggregates | 424 |
| [Task 454][t454] | open | PostgreSQL Store transaction runtime | Bug fix | Make PostgreSQL transaction isolation explicit | — |

## Phase 3 — tests and benchmarks · 29 open

Complete the btcwallet component suites, composed public scenarios, chain
daemon compatibility coverage, integration-test CI matrix, and selected
three-backend performance measurements.

| Task | Status | Component | Type | Outcome | Open dependencies |
| --- | --- | --- | --- | --- | --- |
| [Task 369][t369] | open | Controller | Test | Reject repeated Wallet unlocks without resetting timeout | — |
| [Task 416][t416] | open | wallet benchmark infrastructure | Benchmark | Build reusable database benchmark fixtures | — |
| [Task 417][t417] | open | Manager lifecycle and Syncer benchmarks | Benchmark | Measure start-to-synced performance | 368, 416 |
| [Task 418][t418] | open | wallet UtxoManager benchmarks | Benchmark | Measure ListUnspent performance | 416 |
| [Task 419][t419] | open | wallet AddressManager benchmarks | Benchmark | Measure address-read performance | 416 |
| [Task 420][t420] | open | wallet TxReader benchmarks | Benchmark | Measure transaction-read performance | 416 |
| [Task 427][t427] | open | Controller credential integration tests | Test | Cover remaining Controller credential contracts | — |
| [Task 428][t428] | open | Controller Resync and Rescan integration tests | Test | Cover Controller resync and rescan contracts | — |
| [Task 429][t429] | open | AccountManager mutation integration tests | Test | Cover public account mutation contracts | — |
| [Task 430][t430] | open | AccountManager query integration tests | Test | Cover public account query contracts | — |
| [Task 431][t431] | open | AddressManager allocation and lookup integration tests | Test | Cover address allocation and lookup contracts | — |
| [Task 432][t432] | open | AddressManager import and script integration tests | Test | Cover address import and script contracts | — |
| [Task 435][t435] | open | TxPublisher integration tests | Test | Cover transaction publication contracts | — |
| [Task 436][t436] | open | TxReader integration tests | Test | Cover transaction read contracts | — |
| [Task 437][t437] | open | TxWriter integration tests | Test | Cover transaction label contracts | — |
| [Task 438][t438] | open | PsbtManager packet integration tests | Test | Cover PSBT decoration and combination contracts | — |
| [Task 439][t439] | open | PsbtManager funding integration tests | Test | Cover PSBT funding contracts | — |
| [Task 440][t440] | open | PsbtManager signing integration tests | Test | Cover PSBT signing and finalization contracts | — |
| [Task 441][t441] | open | Signer derivation integration tests | Test | Cover public derivation and ECDH contracts | — |
| [Task 442][t442] | open | Signer signature integration tests | Test | Cover public signing contracts | — |
| [Task 443][t443] | open | UnsafeSigner integration tests | Test | Cover UnsafeSigner extraction contracts | — |
| [Task 444][t444] | open | public wallet receive scenario | Test | Cover the wallet receive and reload scenario | 429, 430, 431, 436 |
| [Task 445][t445] | open | public account signing scenario | Test | Cover account derivation and signing | 429, 431, 441, 442 |
| [Task 446][t446] | open | public transaction spend scenario | Test | Cover transaction creation and publication | 435, 436 |
| [Task 447][t447] | open | public PSBT workflow scenario | Test | Cover the complete PSBT workflow | 435, 436, 438, 439, 440 |
| [Task 448][t448] | open | public Wallet reorganization scenario | Test | Cover synchronization and rollback | 428, 436 |
| [Task 449][t449] | open | multi-Wallet SQL integration scenario | Test | Cover multi-Wallet SQL isolation | 429, 431, 435, 436 |
| [Task 450][t450] | open | bitcoind compatibility integration scenario | Test | Compare public wallet state with bitcoind | 431, 435, 436 |
| [Task 451][t451] | open | integration-test CI matrix and chain daemon provisioning | Test | Run itests across supported chain releases | — |

## Phase 4 — migration and retirement · 6 open

Validate and convert legacy wallets only after their wallet and consumer
dependencies are stable, then retire modern kvdb only with the required
cutover evidence.

| Task | Status | Component | Type | Outcome | Open dependencies |
| --- | --- | --- | --- | --- | --- |
| [Task 335][t335] | open | Legacy migration source snapshot | Migration | Validate an immutable legacy wallet source | — |
| [Task 353][t353] | open | Manager backend selection | Migration | Retire kvdb from the modern wallet API | 376 |
| [Task 373][t373] | open | Legacy migration account and secret extraction | Migration | Extract legacy account and address state | 335 |
| [Task 374][t374] | open | Legacy migration transaction and sync extraction | Migration | Extract legacy transaction and sync state | 373 |
| [Task 375][t375] | open | SQLite migration importer | Migration | Import one legacy snapshot into SQLite | 329, 374, 411 |
| [Task 376][t376] | open | Manager migration orchestration | Migration | Orchestrate and verify legacy conversion | 375 |

## Phase 5 — Signer and Vault · 7 open

Define and implement the wallet's typed secret-operation, derivation, signing,
and raw-key ownership boundaries, then remove the superseded secret seams and
dependent tests.

| Task | Status | Component | Type | Outcome | Open dependencies |
| --- | --- | --- | --- | --- | --- |
| [Task 336][t336] | open | Controller and Wallet passphrase test suite | Test | Remove redundant Wallet passphrase tests | 338, 362, 386 |
| [Task 354][t354] | open | ADR 0010 secret-use contract | Design | Define Vault secret-operation ownership | — |
| [Task 382][t382] | open | AccountManager and account-derivation Vault operation | Refactor | Move account derivation into the Vault | 352, 354, 390 |
| [Task 383][t383] | open | AddressManager ScriptForOutput and script Vault operation | Refactor | Resolve output scripts through the Vault | 352, 354 |
| [Task 384][t384] | open | Signer and primitive key-use Vault operations | Refactor | Route safe signing through the Vault | 337, 347, 352, 354, 383 |
| [Task 385][t385] | open | UnsafeSigner raw private-key boundary | Refactor | Isolate UnsafeSigner key extraction | 384 |
| [Task 386][t386] | open | Wallet-facing Vault and runtimeCache secret interfaces | Refactor | Remove raw Wallet secret seams | 382, 383, 384, 385 |

## Conditional route state

The board records the current route in which modern kvdb remains maintained.
If Task 353 retires it first, affected task files replace or remove their
modern-kvdb clauses and dependencies. Their delivery phases do not change
automatically; any move requires an explicit roadmap-priority decision.

## Maintaining the phases

When a task completes, move its row from the phase table to `Done`, preserve
its delivery phase and record its closing PR there, refresh the counts, and
remove its number from remaining `Open dependencies` cells. An explicit
roadmap rephasing may update both open and completed rows so the `Phase` column
continues to reflect the current thematic map. When dependency edges change,
update the owning task and its open-dependency cell without deriving a new
phase. Every roadmap task must remain listed exactly once.

## Done · 12

| Task | Phase | Component | Type | Outcome | Closing PR |
| --- | ---: | --- | --- | --- | --- |
| [Task 332][t332] | 1 | Manager, AccountManager, and AddressManager | Gap fix | Define watch-only and script-import policy | [#1320][pr1320] |
| [Task 333][t333] | 1 | Manager and integration-test harness | Gap fix | Add PostgreSQL Manager and harness support | [#1303][pr1303] |
| [Task 339][t339] | 1 | TxReader | Gap fix | Define public transaction visibility and status | [#1330][pr1330] |
| [Task 341][t341] | 1 | wallet-owned account identity types | Gap fix | Define semantic account identity | [#1313][pr1313] |
| [Task 342][t342] | 1 | wallet configuration | Bug fix | Accept bounded recovery windows | [#1335][pr1335] |
| [Task 355][t355] | 1 | TxCreator and the `txsizes` module | Bug fix | Correct transaction size accounting | [#1338][pr1338] |
| [Task 359][t359] | 1 | Manager and SQL Store design | Design | Revalidate database identity design | [#1319][pr1319] |
| [Task 360][t360] | 1 | ADR 0013 and the modern kvdb adapter contract | Design | Document kvdb adapter identity | [#1308][pr1308] |
| [Task 400][t400] | 1 | ADR 0004 and ADR 0005 synchronization contract | Design | Separate live watches from historical recovery | [#1322][pr1322] |
| [Task 415][t415] | 3 | UtxoManager integration tests | Test | Cover public UTXO contracts end to end | [#1307][pr1307] |
| [Task 433][t433] | 3 | Wallet sync-state publication and integration-test harness | Test | Enforce the wallet sync test boundary | [#1314][pr1314] |
| [Task 434][t434] | 3 | TxCreator integration tests | Test | Cover transaction creation contracts | [#1316][pr1316] |

[t329]: tasks/329-atomic-wallet-account-creation.md
[t330]: tasks/330-duplicate-wallet-create.md
[t331]: tasks/331-manager-wallet-lifecycle.md
[t332]: archive/tasks/332-watch-only-script-policy.md
[t333]: archive/tasks/333-postgres-manager-harness.md
[t334]: tasks/334-shared-chain-state.md
[t335]: tasks/335-kvdb-to-sql-migration.md
[t336]: tasks/336-test-cleanup.md
[t337]: tasks/337-imported-xpub-derivation.md
[t338]: tasks/338-passphrase-result.md
[t339]: tasks/339-transaction-visibility.md
[t340]: tasks/340-durable-database-identity.md
[t341]: archive/tasks/341-public-role-contract.md
[t342]: archive/tasks/342-recovery-window-validation.md
[t343]: tasks/343-exact-account-creation.md
[t344]: tasks/344-bulk-address-allocation.md
[t345]: tasks/345-durable-watch-registration.md
[t346]: tasks/346-psbt-funding-metadata.md
[t347]: tasks/347-psbt-signing-authorization.md
[t348]: tasks/348-watched-change-delivery.md
[t349]: tasks/349-transaction-invalidation.md
[t350]: tasks/350-native-pgx-postgres.md
[t351]: tasks/351-sql-utxo-integrity.md
[t352]: tasks/352-kvdb-combined-passphrase-rotation.md
[t353]: tasks/353-retire-modern-kvdb-backend.md
[t354]: tasks/354-vault-owned-secret-resolution.md
[t355]: tasks/355-transaction-size-accounting.md
[t356]: tasks/356-defer-change-script-allocation.md
[t357]: tasks/357-transaction-amount-validation.md
[t358]: tasks/358-contextual-manager-operations.md
[t359]: tasks/359-revalidate-database-identity-design.md
[t360]: archive/tasks/360-document-kvdb-adapter-identity.md
[t361]: tasks/361-kvdb-imported-xpub-derivation.md
[t362]: tasks/362-kvdb-definitive-passphrase-result.md
[t363]: tasks/363-recovery-scan-work-bound.md
[t364]: tasks/364-durable-watch-registration.md
[t365]: tasks/365-validate-wallet-genesis-keys.md
[t366]: tasks/366-reconcile-sql-genesis-commit.md
[t367]: tasks/367-bound-sql-genesis-secrets.md
[t368]: tasks/368-wallet-startup-shutdown.md
[t369]: tasks/369-repeated-wallet-unlock.md
[t370]: tasks/370-chain-source-delivery.md
[t371]: tasks/371-canonical-chain-frontier.md
[t372]: tasks/372-multi-wallet-rollback.md
[t373]: tasks/373-extract-legacy-account-address-state.md
[t374]: tasks/374-extract-legacy-transaction-sync-state.md
[t375]: tasks/375-import-legacy-snapshot-sqlite.md
[t376]: tasks/376-orchestrate-legacy-conversion.md
[t377]: tasks/377-account-manager-wallet-dto.md
[t378]: tasks/378-account-manager-error-boundary.md
[t379]: tasks/379-manager-load-error-boundary.md
[t380]: tasks/380-address-lookup-error-boundary.md
[t381]: tasks/381-recovery-adapter-identity.md
[t382]: tasks/382-vault-account-derivation.md
[t383]: tasks/383-vault-output-script-resolution.md
[t384]: tasks/384-vault-safe-signing.md
[t385]: tasks/385-unsafe-signer-key-extraction.md
[t386]: tasks/386-remove-raw-wallet-secret-seams.md
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
[t400]: tasks/400-live-watch-semantics-adr.md
[t401]: tasks/401-live-tip-chain-watch.md
[t402]: tasks/402-replay-live-watch-obligations.md
[t403]: tasks/403-raw-import-watch-obligations.md
[t404]: tasks/404-transaction-input-amounts.md
[t405]: tasks/405-psbt-funding-lease-admission.md
[t406]: tasks/406-psbt-funding-lease-release.md
[t407]: tasks/407-psbt-finalization-authorization.md
[t408]: tasks/408-direct-transaction-change-watch.md
[t409]: tasks/409-transaction-input-graph.md
[t410]: tasks/410-transaction-graph-admission.md
[t411]: tasks/411-terminal-parent-rejection.md
[t412]: tasks/412-transaction-publication-admission.md
[t413]: tasks/413-utxo-spend-identity.md
[t414]: tasks/414-utxo-amount-bounds.md
[t415]: archive/tasks/415-utxo-public-integration-tests.md
[t416]: tasks/416-database-benchmark-fixtures.md
[t417]: tasks/417-start-to-synced-benchmark.md
[t418]: tasks/418-list-unspent-benchmark.md
[t419]: tasks/419-address-read-benchmark.md
[t420]: tasks/420-transaction-read-benchmark.md
[t421]: tasks/421-sql-passphrase-commit-reconciliation.md
[t422]: tasks/422-manager-load-cancellation.md
[t423]: tasks/423-manager-chain-source-cutover.md
[t424]: tasks/424-transaction-utxo-integrity-adr.md
[t425]: tasks/425-separate-transaction-authoring-wrappers.md
[t426]: tasks/426-sql-transaction-output-aggregates.md
[t427]: tasks/427-controller-lifecycle-credential-itests.md
[t428]: tasks/428-controller-sync-info-itests.md
[t429]: tasks/429-account-mutation-itests.md
[t430]: tasks/430-account-query-itests.md
[t431]: tasks/431-address-allocation-lookup-itests.md
[t432]: tasks/432-address-import-script-itests.md
[t433]: archive/tasks/433-wallet-sync-test-boundary.md
[t434]: archive/tasks/434-transaction-creation-itests.md
[t435]: tasks/435-transaction-publication-itests.md
[t436]: tasks/436-transaction-read-itests.md
[t437]: tasks/437-transaction-label-itests.md
[t438]: tasks/438-psbt-decoration-combination-itests.md
[t439]: tasks/439-psbt-funding-itests.md
[t440]: tasks/440-psbt-signing-finalization-itests.md
[t441]: tasks/441-signer-derivation-ecdh-itests.md
[t442]: tasks/442-signer-signing-itests.md
[t443]: tasks/443-unsafe-signer-itests.md
[t444]: tasks/444-wallet-receive-reload-scenario.md
[t445]: tasks/445-account-signing-scenario.md
[t446]: tasks/446-transaction-spend-publish-scenario.md
[t447]: tasks/447-psbt-workflow-scenario.md
[t448]: tasks/448-sync-rollback-scenario.md
[t449]: tasks/449-multi-wallet-sql-isolation-scenario.md
[t450]: tasks/450-bitcoind-compatibility-scenario.md
[t451]: tasks/451-chain-version-itest-matrix.md
[t452]: tasks/452-key-only-recovery-derivation.md
[t453]: tasks/453-serialize-manager-wallet-publication.md
[t454]: tasks/454-postgres-transaction-isolation.md
[pr1303]: https://github.com/btcsuite/btcwallet/pull/1303
[pr1307]: https://github.com/btcsuite/btcwallet/pull/1307
[pr1308]: https://github.com/btcsuite/btcwallet/pull/1308
[pr1313]: https://github.com/btcsuite/btcwallet/pull/1313
[pr1314]: https://github.com/btcsuite/btcwallet/pull/1314
[pr1316]: https://github.com/btcsuite/btcwallet/pull/1316
[pr1320]: https://github.com/btcsuite/btcwallet/pull/1320
