# Task delivery phases

The roadmap contains **120 open tasks**. Completed tasks stay listed with
Status `done`; their files live under `archive/tasks/`.

Every task appears exactly once below. These are thematic delivery stages, not
computed dependency depths or strict phase barriers. Individual task files
remain the authority for every hard dependency edge and implementation gate.
A task never becomes implementable merely because it is listed in an earlier
phase.

Phase 1 is the current delivery focus. Do not deliberately start Phase 2 or
later work while Phase 1 remains current, except for already-started work
recorded in [ACTIVE.md](ACTIVE.md) or an explicit owner-approved exception.

`Open dependencies` lists only hard dependencies whose status is `open`.
Completed dependency edges remain recorded in the owning task file and are
intentionally omitted from the table.

The delivery stages are:

1. known wallet gaps and bug fixes;
2. integration tests and benchmarks;
3. Signer and Vault hardening;
4. integration with lnd; and
5. legacy migration and kvdb retirement.

[ACTIVE.md](ACTIVE.md) separately owns the current focus, claims, in-progress
PRs, and coordination.


## Phase 1 — wallet gaps and bugs · 66 open · 1 done

Stabilize the maintained wallet APIs, storage, lifecycle, synchronization,
transaction, PSBT, and recovery behavior with focused fixes.

| Task | Status | Component | Outcome | Open dependencies |
| --- | --- | --- | --- | --- |
| [Task 329][t329] | open | Manager SQL genesis and SQL Store | Commit SQL wallet genesis atomically | 332, 337, 345 |
| [Task 330][t330] | open | Manager | Reject duplicate wallet creation | 331 |
| [Task 331][t331] | open | Manager | Fence Manager admission and Store closure | 368 |
| [Task 332][t332] | open | Manager, AccountManager, and AddressManager | Define watch-only and script-import policy | — |
| [Task 334][t334] | open | SQL wallet architecture documentation | Supersede ADR 0002 with SQL shared-chain ownership | — |
| [Task 337][t337] | open | Wallet public derivation and Signer numeric adapter | Restore SQL imported-XPub public derivation | 341 |
| [Task 338][t338] | open | Controller | Latch accepted SQL passphrase results | 331, 368, 421 |
| [Task 339][t339] | open | TxReader | Define public transaction visibility and status | — |
| [Task 340][t340] | open | Manager and SQL Store | Bind SQL databases to schema and network identity | 359 |
| [Task 341][t341] | open | wallet-owned account identity types | Define semantic account identity | — |
| [Task 342][t342] | open | wallet configuration | Accept bounded recovery windows | — |
| [Task 345][t345] | open | SQL account schema and AccountStore persistence | Persist SQL account discovery mode | 340 |
| [Task 346][t346] | open | PsbtManager packet validation and transformation | Preserve PSBT metadata during funding | 357, 404 |
| [Task 349][t349] | open | TxStore internal invalidation workflow | Implement idempotent transaction invalidation | 339, 409, 410, 411 |
| [Task 350][t350] | open | PostgreSQL Store | Use native pgx for PostgreSQL storage | — |
| [Task 351][t351] | open | SQL UTXO and lease schema | Bind SQL UTXOs to wallet tenancy | 340, 424 |
| [Task 352][t352] | open | LegacyWalletVault and waddrmgr passphrase rotation | Rotate kvdb passphrases atomically | — |
| [Task 355][t355] | open | TxCreator and the txsizes module | Correct transaction size accounting | — |
| [Task 356][t356] | open | TxCreator and txauthor | Defer change-script allocation until viable | 355, 357, 404 |
| [Task 357][t357] | open | txauthor and TxCreator | Validate transaction output and fee arithmetic | — |
| [Task 358][t358] | open | Manager.Create | Make Manager creation cancellable | 331, 366, 367 |
| [Task 359][t359] | open | Manager and SQL Store design | Revalidate database identity design | — |
| [Task 360][t360] | done | ADR 0013 and the modern kvdb adapter contract | Document kvdb adapter identity | — |
| [Task 361][t361] | open | modern kvdb public derivation adapter | Support kvdb imported-XPub derivation | 337 |
| [Task 362][t362] | open | Controller and LegacyWalletVault | Return a definitive kvdb passphrase result | 338, 352 |
| [Task 363][t363] | open | recovery horizon derivation | Bound aggregate recovery derivation work | 342 |
| [Task 364][t364] | open | SQL watch-generation and obligation schema | Persist ordered SQL live-watch obligations | 345, 400 |
| [Task 365][t365] | open | Manager creation preflight | Validate wallet genesis key identities | 332, 341 |
| [Task 366][t366] | open | Manager and SQL Store genesis result | Reconcile ambiguous SQL genesis commits | 329 |
| [Task 367][t367] | open | Manager and WalletVault genesis preparation | Bound SQL genesis secret lifetime | 329 |
| [Task 368][t368] | open | Controller | Serialize Wallet startup and shutdown | — |
| [Task 369][t369] | open | Controller | Reject repeated Wallet unlocks without resetting timeout | — |
| [Task 370][t370] | open | Manager chain coordinator and Wallet processors | Prepare ordered Manager chain delivery | 331, 334, 371 |
| [Task 371][t371] | open | SQL Store canonical chain state | Serialize canonical chain-frontier mutation | 334, 340 |
| [Task 372][t372] | open | Manager chain coordinator and SQL Store rollback | Prepare atomic multi-Wallet rollback | 349, 370, 371 |
| [Task 377][t377] | open | AccountManager public result contract | Make AccountManager results wallet-owned | 341 |
| [Task 378][t378] | open | AccountManager public error boundary | Translate AccountManager backend errors | — |
| [Task 379][t379] | open | Manager Load public error boundary | Translate Manager load not-found errors | — |
| [Task 380][t380] | open | AddressManager GetAddressInfo error boundary | Translate address lookup errors | — |
| [Task 381][t381] | open | RecoveryState public contract and internal Syncer adapter | Hide recovery adapter identity | — |
| [Task 387][t387] | open | ADR 0012 and ADR 0013 account-creation contract | Define custom and path-bound account identity | 332 |
| [Task 388][t388] | open | SQL AccountStore account-creation admission | Reject colliding runtime account XPub identities | 341, 387 |
| [Task 392][t392] | open | AccountManager and SQL AccountStore create inputs | Consume discovery mode in existing account creation | 345 |
| [Task 393][t393] | open | AddressManager receiving entry points and AddressStore account lookup | Reject receiving allocation for key-only accounts | 341, 345 |
| [Task 394][t394] | open | AddressStore ownership resolution and transaction ingestion | Exclude key-only locators from wallet ownership | 345 |
| [Task 397][t397] | open | ADR 0004 targeted-recovery contract | Define semantic-account recovery targets | 341, 400 |
| [Task 398][t398] | open | Controller and Syncer recovery account selection | Accept semantic account recovery targets | 341, 345, 397, 399 |
| [Task 399][t399] | open | Syncer recovery initialization and account-scoped Store reads | Isolate persisted targets in zero-lookahead recovery | 342, 345 |
| [Task 400][t400] | open | ADR 0004 and ADR 0005 synchronization contract | Separate live watches from historical recovery | — |
| [Task 401][t401] | open | chain interface and Neutrino/btcd/bitcoind adapters | Add an idempotent live-tip-only chain watch | 400 |
| [Task 402][t402] | open | Controller/Syncer watch initialization and delivery completion | Replay live-watch obligations through the Syncer | 364, 401 |
| [Task 403][t403] | open | AddressManager SQL raw-import mutation | Persist raw imports with live-watch obligations | 332, 364, 402 |
| [Task 404][t404] | open | txauthor input sources and TxCreator source adapters | Validate transaction input-source amounts | 357 |
| [Task 405][t405] | open | PsbtManager and UTXO Store funding admission | Reserve PSBT funding inputs atomically | 346, 356, 404 |
| [Task 406][t406] | open | UTXO Store and PsbtManager cleanup | Release PSBT funding leases by exact token | 405 |
| [Task 409][t409] | open | SQL schema and TxStore ingestion | Normalize transaction input edges | 340, 424 |
| [Task 410][t410] | open | Wallet/Store transaction graph admission | Coordinate transaction graph mutations | 334, 409 |
| [Task 411][t411] | open | TxStore activation preflight and direct/batch ingestion | Reject children of terminal transactions | 339, 409, 410 |
| [Task 412][t412] | open | TxWriter public contract, TxPublisher, and transaction graph admission | Expose invalidation with publication ordering | 349 |
| [Task 413][t413] | open | SQL UTXO and normalized-input schema | Constrain SQL UTXOs to exact spending inputs | 351, 409, 410 |
| [Task 414][t414] | open | SQL UTXO schema | Bound persisted SQL UTXO amounts | 351 |
| [Task 421][t421] | open | WalletVault and SQL wallet-secret persistence | Reconcile SQL passphrase commits | — |
| [Task 422][t422] | open | Manager.Load | Make Manager loading cancellable | 331 |
| [Task 423][t423] | open | SQL Manager chain source adapter and coordinator activation | Cut over SQL Manager chain notification ownership | 368, 370, 372, 412, 422 |
| [Task 424][t424] | open | ADR 0006 transaction and UTXO integrity contract | Define transaction and UTXO integrity ownership | — |
| [Task 425][t425] | open | Wallet transaction-authoring wrapper boundary | Isolate shared transaction authoring from wrapper watch and lease effects | — |
| [Task 426][t426] | open | SQL TxStore ingestion amount validation | Validate persisted transaction output aggregates | 424 |

## Phase 2 — tests and benchmarks · 30 open · 2 done

Complete the btcwallet component suites, composed public scenarios, chain
daemon compatibility coverage, integration-test CI matrix, and selected
three-backend performance measurements.

| Task | Status | Component | Outcome | Open dependencies |
| --- | --- | --- | --- | --- |
| [Task 333][t333] | done | Manager and integration-test harness | Add PostgreSQL Manager and harness support | — |
| [Task 415][t415] | done | UtxoManager integration tests | Cover public UTXO contracts end to end | — |
| [Task 416][t416] | open | wallet benchmark infrastructure | Build reusable database benchmark fixtures | — |
| [Task 417][t417] | open | wallet Controller and Syncer benchmarks | Measure start-to-synced performance | 416 |
| [Task 418][t418] | open | wallet UtxoManager benchmarks | Measure ListUnspent performance | 416 |
| [Task 419][t419] | open | wallet AddressManager benchmarks | Measure address-read performance | 416 |
| [Task 420][t420] | open | wallet TxReader benchmarks | Measure transaction-read performance | 416 |
| [Task 427][t427] | open | Controller credential integration tests | Cover remaining Controller credential contracts | — |
| [Task 428][t428] | open | Controller Resync and Rescan integration tests | Cover Controller resync and rescan contracts | — |
| [Task 429][t429] | open | AccountManager mutation integration tests | Cover public account mutation contracts | — |
| [Task 430][t430] | open | AccountManager query integration tests | Cover public account query contracts | — |
| [Task 431][t431] | open | AddressManager allocation and lookup integration tests | Cover address allocation and lookup contracts | — |
| [Task 432][t432] | open | AddressManager import and script integration tests | Cover address import and script contracts | — |
| [Task 433][t433] | open | Wallet sync-state publication and integration-test harness | Enforce the wallet sync test boundary | — |
| [Task 434][t434] | open | TxCreator integration tests | Cover transaction creation contracts | — |
| [Task 435][t435] | open | TxPublisher integration tests | Cover transaction publication contracts | — |
| [Task 436][t436] | open | TxReader integration tests | Cover transaction read contracts | — |
| [Task 437][t437] | open | TxWriter integration tests | Cover transaction label contracts | — |
| [Task 438][t438] | open | PsbtManager packet integration tests | Cover PSBT decoration and combination contracts | — |
| [Task 439][t439] | open | PsbtManager funding integration tests | Cover PSBT funding contracts | — |
| [Task 440][t440] | open | PsbtManager signing integration tests | Cover PSBT signing and finalization contracts | — |
| [Task 441][t441] | open | Signer derivation integration tests | Cover public derivation and ECDH contracts | — |
| [Task 442][t442] | open | Signer signature integration tests | Cover public signing contracts | — |
| [Task 443][t443] | open | UnsafeSigner integration tests | Cover UnsafeSigner extraction contracts | — |
| [Task 444][t444] | open | public wallet receive scenario | Cover the wallet receive and reload scenario | 429, 430, 431, 433, 436 |
| [Task 445][t445] | open | public account signing scenario | Cover account derivation and signing | 429, 431, 441, 442 |
| [Task 446][t446] | open | public transaction spend scenario | Cover transaction creation and publication | 433, 434, 435, 436 |
| [Task 447][t447] | open | public PSBT workflow scenario | Cover the complete PSBT workflow | 433, 435, 436, 438, 439, 440 |
| [Task 448][t448] | open | public Wallet reorganization scenario | Cover synchronization and rollback | 428, 433, 436 |
| [Task 449][t449] | open | multi-Wallet SQL integration scenario | Cover multi-Wallet SQL isolation | 429, 431, 433, 434, 435, 436 |
| [Task 450][t450] | open | bitcoind compatibility integration scenario | Compare public wallet state with bitcoind | 431, 433, 434, 435, 436 |
| [Task 451][t451] | open | integration-test CI matrix and chain daemon provisioning | Test supported chain daemon releases | — |

## Phase 3 — Signer and Vault · 7 open

Define and implement the wallet's typed secret-operation, derivation, signing,
and raw-key ownership boundaries, then remove the superseded secret seams and
dependent tests.

| Task | Status | Component | Outcome | Open dependencies |
| --- | --- | --- | --- | --- |
| [Task 336][t336] | open | Controller and Wallet passphrase test suite | Remove redundant Wallet passphrase tests | 338, 362, 386 |
| [Task 354][t354] | open | ADR 0010 secret-use contract | Define Vault secret-operation ownership | — |
| [Task 382][t382] | open | AccountManager and account-derivation Vault operation | Move account derivation into the Vault | 352, 354 |
| [Task 383][t383] | open | AddressManager ScriptForOutput and script Vault operation | Resolve output scripts through the Vault | 352, 354 |
| [Task 384][t384] | open | Signer and primitive key-use Vault operations | Route safe signing through the Vault | 337, 352, 354, 383 |
| [Task 385][t385] | open | UnsafeSigner raw private-key boundary | Isolate UnsafeSigner key extraction | 384 |
| [Task 386][t386] | open | Wallet-facing Vault and runtimeCache secret interfaces | Remove raw Wallet secret seams | 382, 383, 384, 385 |

## Phase 4 — lnd integration · 11 open

Land the btcwallet-side public APIs explicitly required by lnd: exact and
custom accounts, atomic address batches, durable key locators, and their final
PSBT and address semantics. Any lnd-repository change, including companion
import-wallet routing, requires a separately accepted task in this phase.

| Task | Status | Component | Outcome | Open dependencies |
| --- | --- | --- | --- | --- |
| [Task 343][t343] | open | AccountManager root-derived creation and SQL AccountStore CAS | Create next-or-exact root-derived accounts | 332, 377, 382, 388, 392, 393, 394, 398 |
| [Task 344][t344] | open | AddressManager receiving batches and SQL AddressStore mutation | Allocate receiving-address batches atomically | 341, 364, 393, 402 |
| [Task 347][t347] | open | PsbtManager and Signer | Authorize explicit PSBT signatures | 346, 384, 390 |
| [Task 348][t348] | open | PsbtManager and AddressManager | Watch PSBT change before returning funding | 344, 406, 425 |
| [Task 389][t389] | open | AccountManager and SQL AccountStore account result reader | Reconcile ambiguous runtime account commits | 343 |
| [Task 390][t390] | open | AccountManager custom scope and SQL key-scope/account mutation | Create root-derived accounts in custom scopes | 343, 387, 389 |
| [Task 391][t391] | open | AccountManager path-bound identity and SQL AccountStore schema | Create exact path-bound watch-only XPub accounts | 332, 337, 387, 388, 389, 390 |
| [Task 395][t395] | open | AddressManager key allocation and SQL branch-counter mutation | Allocate one durable key-only locator | 341, 393, 394, 399 |
| [Task 396][t396] | open | AddressManager count-one compatibility semantics | Make NewAddress return the oldest unused address | 344, 348, 408 |
| [Task 407][t407] | open | PsbtManager FinalizePsbt | Authorize PSBT auto-finalization | 347 |
| [Task 408][t408] | open | TxCreator and AddressManager | Watch direct transaction change before return | 344, 356, 425 |

## Phase 5 — migration and retirement · 6 open

Validate and convert legacy wallets only after the preceding wallet and
consumer contracts are stable, then retire modern kvdb only with the required
cutover evidence.

| Task | Status | Component | Outcome | Open dependencies |
| --- | --- | --- | --- | --- |
| [Task 335][t335] | open | Legacy migration source snapshot | Validate an immutable legacy wallet source | — |
| [Task 353][t353] | open | Manager backend selection | Retire kvdb from the modern wallet API | 376 |
| [Task 373][t373] | open | Legacy migration account and secret extraction | Extract legacy account and address state | 332, 335 |
| [Task 374][t374] | open | Legacy migration transaction and sync extraction | Extract legacy transaction and sync state | 373 |
| [Task 375][t375] | open | SQLite migration importer | Import one legacy snapshot into SQLite | 329, 374, 411 |
| [Task 376][t376] | open | Manager migration orchestration | Orchestrate and verify legacy conversion | 375 |

## Conditional route state

The board records the current route in which modern kvdb remains maintained.
If Task 353 retires it first, affected task files replace or remove their
modern-kvdb clauses and dependencies. Their delivery phases do not change
automatically; any move requires an explicit roadmap-priority decision.

## Maintaining the phases

When a task completes, preserve its row and delivery phase, change only its
`Status` to `done`, refresh the open and done counts, and remove its number
from remaining `Open dependencies` cells. When dependency edges change,
update the owning task and its open-dependency cell without deriving a new
phase. Every open task must remain listed exactly once.

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
[t445]: tasks/445-account-signing-scenario.md
[t446]: tasks/446-transaction-spend-publish-scenario.md
[t447]: tasks/447-psbt-workflow-scenario.md
[t448]: tasks/448-sync-rollback-scenario.md
[t449]: tasks/449-multi-wallet-sql-isolation-scenario.md
[t450]: tasks/450-bitcoind-compatibility-scenario.md
[t451]: tasks/451-chain-version-itest-matrix.md
[t333]: archive/tasks/333-postgres-manager-harness.md
[t360]: archive/tasks/360-document-kvdb-adapter-identity.md
[t415]: archive/tasks/415-utxo-public-integration-tests.md
