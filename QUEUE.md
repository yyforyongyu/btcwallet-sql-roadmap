# Task dependency phases

The roadmap contains **120 open tasks**. No task in the current Task
329-and-later series has been completed and archived; older archived files are
historical evidence and are not part of this map.

Every open task appears exactly once below. Phase 1 contains active tasks with
no open-task dependency. Every later phase is
`1 + max(phase of each direct open-task dependency)`. Tasks in one phase
have no direct dependency on another task in that phase.

Phase numbers report dependency state. Phase 1 is the current execution focus;
later-phase tasks may be claimed but become active only after their own
dependencies complete and the map is refreshed. Phases impose no global
barrier. Individual task files own scope, dependencies, and implementation
gates.
[ACTIVE.md](ACTIVE.md) separately owns the active-task focus, claims,
in-progress PRs, and coordination state.


## Phase 1 — 41 tasks

| Task | Component | Outcome | Direct dependencies | Non-task gates |
| --- | --- | --- | --- | --- |
| [Task 332][t332] | Manager, AccountManager, and AddressManager | Define watch-only and script-import policy | — | — | | | |
| [Task 334][t334] | SQL wallet architecture documentation | Supersede ADR 0002 with SQL shared-chain ownership | — | — | | | |
| [Task 335][t335] | Legacy migration source snapshot | Validate an immutable legacy wallet source | — | — | | | |
| [Task 339][t339] | TxReader | Define public transaction visibility and status | — | — | | | |
| [Task 341][t341] | wallet-owned account identity types | Define semantic account identity | — | — | | | |
| [Task 342][t342] | wallet configuration | Accept bounded recovery windows | — | — | | | |
| [Task 350][t350] | PostgreSQL Store | Use native pgx for PostgreSQL storage | — | — | | | |
| [Task 352][t352] | LegacyWalletVault and waddrmgr passphrase rotation | Rotate kvdb passphrases atomically | — | — | | | |
| [Task 354][t354] | ADR 0010 secret-use contract | Define Vault secret-operation ownership | — | durable disposition of draft btcwallet PR #1279 before ADR acceptance | | | |
| [Task 355][t355] | TxCreator and the txsizes module | Correct transaction size accounting | — | — | | | |
| [Task 357][t357] | txauthor and TxCreator | Validate transaction output and fee arithmetic | — | — | | | |
| [Task 359][t359] | Manager and SQL Store design | Revalidate database identity design | — | — | | | |
| [Task 368][t368] | Controller | Serialize Wallet startup and shutdown | — | — | | | |
| [Task 369][t369] | Controller | Reject repeated Wallet unlocks without resetting timeout | — | — | | | |
| [Task 378][t378] | AccountManager public error boundary | Translate AccountManager backend errors | — | — | | | |
| [Task 379][t379] | Manager Load public error boundary | Translate Manager load not-found errors | — | — | | | |
| [Task 380][t380] | AddressManager GetAddressInfo error boundary | Translate address lookup errors | — | — | | | |
| [Task 381][t381] | RecoveryState public contract and internal Syncer adapter | Hide recovery adapter identity | — | — | | | |
| [Task 400][t400] | ADR 0004 and ADR 0005 synchronization contract | Separate live watches from historical recovery | — | — | | | |
| [Task 416][t416] | wallet benchmark infrastructure | Build reusable SQLite benchmark fixtures | — | — | | | |
| [Task 421][t421] | WalletVault and SQL wallet-secret persistence | Reconcile SQL passphrase commits | — | — | | | |
| [Task 424][t424] | ADR 0006 transaction and UTXO integrity contract | Define transaction and UTXO integrity ownership | — | — | | | |
| [Task 425][t425] | Wallet transaction-authoring wrapper boundary | Isolate shared transaction authoring from wrapper watch and lease effects | — | — | | | |
| [Task 427][t427] | Controller credential integration tests | Cover remaining Controller credential contracts | — | — | | | |
| [Task 428][t428] | Controller Resync and Rescan integration tests | Cover Controller resync and rescan contracts | — | — | | | |
| [Task 429][t429] | AccountManager mutation integration tests | Cover public account mutation contracts | — | — | | | |
| [Task 430][t430] | AccountManager query integration tests | Cover public account query contracts | — | — | | | |
| [Task 431][t431] | AddressManager allocation and lookup integration tests | Cover address allocation and lookup contracts | — | — | | | |
| [Task 432][t432] | AddressManager import and script integration tests | Cover address import and script contracts | — | — | | | |
| [Task 433][t433] | Wallet sync-state publication and integration-test harness | Enforce the wallet sync test boundary | — | — | | | |
| [Task 434][t434] | TxCreator integration tests | Cover transaction creation contracts | — | — | | | |
| [Task 435][t435] | TxPublisher integration tests | Cover transaction publication contracts | — | — | | | |
| [Task 436][t436] | TxReader integration tests | Cover transaction read contracts | — | — | | | |
| [Task 437][t437] | TxWriter integration tests | Cover transaction label contracts | — | — | | | |
| [Task 438][t438] | PsbtManager packet integration tests | Cover PSBT decoration and combination contracts | — | — | | | |
| [Task 439][t439] | PsbtManager funding integration tests | Cover PSBT funding contracts | — | — | | | |
| [Task 440][t440] | PsbtManager signing integration tests | Cover PSBT signing and finalization contracts | — | — | | | |
| [Task 441][t441] | Signer derivation integration tests | Cover public derivation and ECDH contracts | — | — | | | |
| [Task 442][t442] | Signer signature integration tests | Cover public signing contracts | — | — | | | |
| [Task 443][t443] | UnsafeSigner integration tests | Cover UnsafeSigner extraction contracts | — | — | | | |
| [Task 451][t451] | integration-test CI matrix and chain daemon provisioning | Test supported chain daemon releases | — | — | | | |

## Phase 2 — 25 tasks

| Task | Component | Outcome | Direct dependencies | Non-task gates |
| --- | --- | --- | --- | --- |
| [Task 331][t331] | Manager | Fence Manager admission and Store closure | 368 | — | | | |
| [Task 337][t337] | Wallet public derivation and Signer numeric adapter | Restore SQL imported-XPub public derivation | 341 | — | | | |
| [Task 340][t340] | Manager and SQL Store | Bind SQL databases to schema and network identity | 359 | — | | | |
| [Task 363][t363] | recovery horizon derivation | Bound aggregate recovery derivation work | 342 | — | | | |
| [Task 365][t365] | Manager creation preflight | Validate wallet genesis key identities | 332, 341 | — | | | |
| [Task 373][t373] | Legacy migration account and secret extraction | Extract legacy account and address state | 332, 335 | — | | | |
| [Task 377][t377] | AccountManager public result contract | Make AccountManager results wallet-owned | 341 | — | | | |
| [Task 382][t382] | AccountManager and account-derivation Vault operation | Move account derivation into the Vault | 352, 354 | — | | | |
| [Task 383][t383] | AddressManager ScriptForOutput and script Vault operation | Resolve output scripts through the Vault | 352, 354 | — | | | |
| [Task 387][t387] | ADR 0012 and ADR 0013 account-creation contract | Define custom and path-bound account identity | 332 | — | | | |
| [Task 397][t397] | ADR 0004 targeted-recovery contract | Define semantic-account recovery targets | 341, 400 | — | | | |
| [Task 401][t401] | chain interface and Neutrino/btcd/bitcoind adapters | Add an idempotent live-tip-only chain watch | 400 | — | | | |
| [Task 404][t404] | txauthor input sources and TxCreator source adapters | Validate transaction input-source amounts | 357 | — | | | |
| [Task 417][t417] | wallet Controller and Syncer benchmarks | Measure SQLite start-to-synced performance | 416 | — | | | |
| [Task 418][t418] | wallet UtxoManager benchmarks | Measure SQLite ListUnspent performance | 416 | — | | | |
| [Task 419][t419] | wallet AddressManager benchmarks | Measure SQLite address-read performance | 416 | — | | | |
| [Task 420][t420] | wallet TxReader benchmarks | Measure SQLite transaction-read performance | 416 | — | | | |
| [Task 426][t426] | SQL TxStore ingestion amount validation | Validate persisted transaction output aggregates | 424 | — | | | |
| [Task 444][t444] | public wallet receive scenario | Cover the wallet receive and reload scenario | 429, 430, 431, 433, 436 | — | | | |
| [Task 445][t445] | SQL account signing scenario | Cover SQL account derivation and signing | 429, 431, 441, 442 | — | | | |
| [Task 446][t446] | public transaction spend scenario | Cover transaction creation and publication | 433, 434, 435, 436 | — | | | |
| [Task 447][t447] | public PSBT workflow scenario | Cover the complete PSBT workflow | 433, 435, 436, 438, 439, 440 | — | | | |
| [Task 448][t448] | public Wallet reorganization scenario | Cover synchronization and rollback | 428, 433, 436 | — | | | |
| [Task 449][t449] | multi-Wallet SQL integration scenario | Cover multi-Wallet SQL isolation | 429, 431, 433, 434, 435, 436 | — | | | |
| [Task 450][t450] | bitcoind compatibility integration scenario | Compare public wallet state with bitcoind | 431, 433, 434, 435, 436 | — | | | |

## Phase 3 — 13 tasks

| Task | Component | Outcome | Direct dependencies | Non-task gates |
| --- | --- | --- | --- | --- |
| [Task 330][t330] | Manager | Reject duplicate wallet creation | 331 | — | | | |
| [Task 338][t338] | Controller | Latch accepted SQL passphrase results | 331, 368, 421 | — | | | |
| [Task 345][t345] | SQL account schema and AccountStore persistence | Persist SQL account discovery mode | 340 | — | | | |
| [Task 346][t346] | PsbtManager packet validation and transformation | Preserve PSBT metadata during funding | 357, 404 | — | | | |
| [Task 351][t351] | SQL UTXO and lease schema | Bind SQL UTXOs to wallet tenancy | 340, 424 | — | | | |
| [Task 356][t356] | TxCreator and txauthor | Defer change-script allocation until viable | 355, 357, 404 | released nested-module changes from Tasks 355 and 357 | | | |
| [Task 361][t361] | modern kvdb public derivation adapter | Support kvdb imported-XPub derivation | 337 | — | | | |
| [Task 371][t371] | SQL Store canonical chain state | Serialize canonical chain-frontier mutation | 334, 340 | — | | | |
| [Task 374][t374] | Legacy migration transaction and sync extraction | Extract legacy transaction and sync state | 373 | — | | | |
| [Task 384][t384] | Signer and primitive key-use Vault operations | Route safe signing through the Vault | 337, 352, 354, 383 | — | | | |
| [Task 388][t388] | SQL AccountStore account-creation admission | Reject colliding runtime account XPub identities | 341, 387 | — | | | |
| [Task 409][t409] | SQL schema and TxStore ingestion | Normalize transaction input edges | 340, 424 | — | | | |
| [Task 422][t422] | Manager.Load | Make Manager loading cancellable | 331 | — | | | |

## Phase 4 — 12 tasks

| Task | Component | Outcome | Direct dependencies | Non-task gates |
| --- | --- | --- | --- | --- |
| [Task 329][t329] | Manager SQL genesis and SQL Store | Commit SQL wallet genesis atomically | 332, 337, 345 | — | | | |
| [Task 362][t362] | Controller and LegacyWalletVault | Return a definitive kvdb passphrase result | 338, 352 | — | | | |
| [Task 364][t364] | SQL watch-generation and obligation schema | Persist ordered SQL live-watch obligations | 345, 400 | — | | | |
| [Task 370][t370] | Manager chain coordinator and Wallet processors | Prepare ordered Manager chain delivery | 331, 334, 371 | — | | | |
| [Task 385][t385] | UnsafeSigner raw private-key boundary | Isolate UnsafeSigner key extraction | 384 | — | | | |
| [Task 392][t392] | AccountManager and SQL AccountStore create inputs | Consume discovery mode in existing account creation | 345 | — | | | |
| [Task 393][t393] | AddressManager receiving entry points and AddressStore account lookup | Reject receiving allocation for key-only accounts | 341, 345 | — | | | |
| [Task 394][t394] | AddressStore ownership resolution and transaction ingestion | Exclude key-only locators from wallet ownership | 345 | — | | | |
| [Task 399][t399] | Syncer recovery initialization and account-scoped Store reads | Isolate persisted targets in zero-lookahead recovery | 342, 345 | — | | | |
| [Task 405][t405] | PsbtManager and UTXO Store funding admission | Reserve PSBT funding inputs atomically | 346, 356, 404 | — | | | |
| [Task 410][t410] | Wallet/Store transaction graph admission | Coordinate transaction graph mutations | 334, 409 | — | | | |
| [Task 414][t414] | SQL UTXO schema | Bound persisted SQL UTXO amounts | 351 | — | | | |

## Phase 5 — 9 tasks

| Task | Component | Outcome | Direct dependencies | Non-task gates |
| --- | --- | --- | --- | --- |
| [Task 366][t366] | Manager and SQL Store genesis result | Reconcile ambiguous SQL genesis commits | 329 | — | | | |
| [Task 367][t367] | Manager and WalletVault genesis preparation | Bound SQL genesis secret lifetime | 329 | — | | | |
| [Task 386][t386] | Wallet-facing Vault and runtimeCache secret interfaces | Remove raw Wallet secret seams | 382, 383, 384, 385 | — | | | |
| [Task 395][t395] | AddressManager key allocation and SQL branch-counter mutation | Allocate one durable key-only locator | 341, 393, 394, 399 | — | | | |
| [Task 398][t398] | Controller and Syncer recovery account selection | Accept semantic account recovery targets | 341, 345, 397, 399 | — | | | |
| [Task 402][t402] | Controller/Syncer watch initialization and delivery completion | Replay live-watch obligations through the Syncer | 364, 401 | — | | | |
| [Task 406][t406] | UTXO Store and PsbtManager cleanup | Release PSBT funding leases by exact token | 405 | — | | | |
| [Task 411][t411] | TxStore activation preflight and direct/batch ingestion | Reject children of terminal transactions | 339, 409, 410 | — | | | |
| [Task 413][t413] | SQL UTXO and normalized-input schema | Constrain SQL UTXOs to exact spending inputs | 351, 409, 410 | — | | | |

## Phase 6 — 7 tasks

| Task | Component | Outcome | Direct dependencies | Non-task gates |
| --- | --- | --- | --- | --- |
| [Task 336][t336] | Controller and Wallet passphrase test suite | Remove redundant Wallet passphrase tests | 338, 362, 386 | — | | | |
| [Task 343][t343] | AccountManager root-derived creation and SQL AccountStore CAS | Create next-or-exact root-derived accounts | 332, 377, 382, 388, 392, 393, 394, 398 | — | | | |
| [Task 344][t344] | AddressManager receiving batches and SQL AddressStore mutation | Allocate receiving-address batches atomically | 341, 364, 393, 402 | — | | | |
| [Task 349][t349] | TxStore internal invalidation workflow | Implement idempotent transaction invalidation | 339, 409, 410, 411 | — | | | |
| [Task 358][t358] | Manager.Create | Make Manager creation cancellable | 331, 366, 367 | — | | | |
| [Task 375][t375] | SQLite migration importer | Import one legacy snapshot into SQLite | 329, 374, 411 | — | | | |
| [Task 403][t403] | AddressManager SQL raw-import mutation | Persist raw imports with live-watch obligations | 332, 364, 402 | — | | | |

## Phase 7 — 6 tasks

| Task | Component | Outcome | Direct dependencies | Non-task gates |
| --- | --- | --- | --- | --- |
| [Task 348][t348] | PsbtManager and AddressManager | Watch PSBT change before returning funding | 344, 406, 425 | — | | | |
| [Task 372][t372] | Manager chain coordinator and SQL Store rollback | Prepare atomic multi-Wallet rollback | 349, 370, 371 | — | | | |
| [Task 376][t376] | Manager migration orchestration | Orchestrate and verify legacy conversion | 375 | — | | | |
| [Task 389][t389] | AccountManager and SQL AccountStore account result reader | Reconcile ambiguous runtime account commits | 343 | — | | | |
| [Task 408][t408] | TxCreator and AddressManager | Watch direct transaction change before return | 344, 356, 425 | — | | | |
| [Task 412][t412] | TxWriter public contract, TxPublisher, and transaction graph admission | Expose invalidation with publication ordering | 349 | — | | | |

## Phase 8 — 4 tasks

| Task | Component | Outcome | Direct dependencies | Non-task gates |
| --- | --- | --- | --- | --- |
| [Task 353][t353] | Manager backend selection | Retire kvdb from the modern wallet API | 376 | production conversion evidence, an identified completed consumer-cutover artifact or release proving maintained consumers and deployed wallets no longer select kvdb, and an explicit retirement decision | | | |
| [Task 390][t390] | AccountManager custom scope and SQL key-scope/account mutation | Create root-derived accounts in custom scopes | 343, 387, 389 | — | | | |
| [Task 396][t396] | AddressManager count-one compatibility semantics | Make NewAddress return the oldest unused address | 344, 348, 408 | — | | | |
| [Task 423][t423] | SQL Manager chain source adapter and coordinator activation | Cut over SQL Manager chain notification ownership | 368, 370, 372, 412, 422 | — | | | |

## Phase 9 — 2 tasks

| Task | Component | Outcome | Direct dependencies | Non-task gates |
| --- | --- | --- | --- | --- |
| [Task 347][t347] | PsbtManager and Signer | Authorize explicit PSBT signatures | 346, 384, 390 | — | | | |
| [Task 391][t391] | AccountManager path-bound identity and SQL AccountStore schema | Create exact path-bound watch-only XPub accounts | 332, 337, 387, 388, 389, 390 | — | | | |

## Phase 10 — 1 task

| Task | Component | Outcome | Direct dependencies | Non-task gates |
| --- | --- | --- | --- | --- |
| [Task 407][t407] | PsbtManager FinalizePsbt | Authorize PSBT auto-finalization | 347 | — | | | |
## Conditional route state

This map records the current route in which modern kvdb remains maintained.
If Task 353 retires it first, affected task files replace or remove their
modern-kvdb clauses and dependencies, and every affected downstream phase is
recomputed. A claim does not make that alternate route complete.

## Maintaining the phases

When a task completes or a direct dependency changes, refresh that task and
only the downstream phases affected by the edge. Preserve one row per open task
and keep non-task gates inline with their owning row.

[t329]: tasks/329-atomic-wallet-account-creation.md
[t330]: tasks/330-duplicate-wallet-create.md
[t331]: tasks/331-manager-wallet-lifecycle.md
[t332]: tasks/332-watch-only-script-policy.md
[t334]: tasks/334-shared-chain-state.md
[t335]: tasks/335-kvdb-to-sql-migration.md
[t336]: tasks/336-test-cleanup.md
[t337]: tasks/337-imported-xpub-derivation.md
[t338]: tasks/338-passphrase-result.md
[t339]: tasks/339-transaction-visibility.md
[t340]: tasks/340-durable-database-identity.md
[t341]: tasks/341-public-role-contract.md
[t342]: tasks/342-recovery-window-validation.md
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
[t416]: tasks/416-sqlite-benchmark-fixtures.md
[t417]: tasks/417-sqlite-start-to-synced-benchmark.md
[t418]: tasks/418-sqlite-list-unspent-benchmark.md
[t419]: tasks/419-sqlite-address-read-benchmark.md
[t420]: tasks/420-sqlite-transaction-read-benchmark.md
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
[t444]: tasks/444-wallet-receive-reload-scenario.md
[t445]: tasks/445-sql-account-signing-scenario.md
[t446]: tasks/446-transaction-spend-publish-scenario.md
[t447]: tasks/447-psbt-workflow-scenario.md
[t448]: tasks/448-sync-rollback-scenario.md
[t449]: tasks/449-multi-wallet-sql-isolation-scenario.md
[t450]: tasks/450-bitcoind-compatibility-scenario.md
[t451]: tasks/451-chain-version-itest-matrix.md
