# Task delivery phases

The roadmap contains **141 tasks**, with **96 remaining open**, **34 done**,
**7 superseded**, and **1 deferred**. Open tasks are grouped by delivery phase;
completed tasks move to `Done`, while superseded tasks retain their identity
and replacement in `Superseded`. Approved future-cycle work is in `Deferred`.

Every roadmap task appears exactly once below. Phases are thematic delivery
stages, not dependency depths or strict barriers. Task files own primary type,
hard dependencies, and implementation gates. Earlier placement never waives
a dependency; later feature verification can follow its production prerequisite.

Phase 1 is the current focus. Do not deliberately start later-phase work while
it remains current, except for already-started work in [ACTIVE.md](ACTIVE.md)
or an explicit owner-approved exception. Rephasing preserves existing claims
and PRs; it does not stop their implementation.

`Open dependencies` lists only hard dependencies whose status is `open`.
Completed edges remain in the owning task file and are omitted here.

The delivery stages are:

1. API stabilization and basic public integration tests;
2. critical correctness, scenario integration tests, and benchmarks;
3. kvdb-to-SQL migration;
4. remaining wallet gaps and bugs; and
5. Signer and Vault hardening.

Finish Phase 1's API implementation and basic tests before starting the main
lnd port PR. The port can then advance in parallel with Phase 2. Known critical
defects and relevant scenarios gate merging or enabling the affected adapter
paths; benchmarks are measurement, not an integration gate. Do not wait for
every later feature, refactor, or hardening task to begin the port.

Phase 3 owns verified conversion, not kvdb removal. It needs its explicit
prerequisites and verification of the paths migration uses, not every optional
scenario in Phase 2 or every bug in Phase 4. An lnd-repository implementation
still requires separate authorization; these tasks remain btcwallet-owned.

[ACTIVE.md](ACTIVE.md) owns current focus, claims, and in-progress PRs.

## Phase 1 — API stabilization and basic itests · 22 open

Close the initial lnd adapter's contract and runtime gaps: aggregate Manager
lifecycle and contextual creation, purpose-1017 key families, final address
allocation, transaction invalidation, and truthful synchronization status.
Basic component tests prove these contracts and existing key/signing APIs.

Keep the complete prerequisite closure. Task 395 reuses Task 344's durable
child allocation for `AllocateNextKey`; it does not depend on Task 396's
receiving-address semantics. Task 402 establishes btcd/bitcoind watches from
Wallet facts, without a ledger or completion flags. Lost required readiness
stops the Manager; Neutrino fixes and cache optimization remain out of scope.

| Task | Status | Component | Type | Outcome | Open dependencies |
| --- | --- | --- | --- | --- | --- |
| [Task 331][t331] | open | Manager lifecycle, public API cutover, and Store ownership | Bug fix | Make Manager the sole lifecycle authority | 459 |
| [Task 343][t343] | open | AccountManager root-derived creation and SQL AccountStore | Gap fix | Create next-or-exact root-derived accounts | 392 |
| [Task 344][t344] | open | AddressManager receiving batches and SQL AddressStore | Gap fix | Allocate receiving-address batches atomically | 392, 393, 402 |
| [Task 358][t358] | open | Manager.Create | Gap fix | Make Manager creation cancellable | 331 |
| [Task 390][t390] | open | AccountManager custom scope and SQL key-scope/account mutation | Gap fix | Create root-derived accounts in custom scopes | 343, 468 |
| [Task 393][t393] | open | AddressManager receiving APIs and AddressStore account lookup | Gap fix | Guard receiving APIs for NoChainSync accounts | 345, 392 |
| [Task 395][t395] | open | AddressManager key allocation and SQL branch-counter mutation | Gap fix | Add AllocateNextKey for durable key allocation | 344, 390, 393, 399, 452 |
| [Task 396][t396] | open | AddressManager count-one semantics | Bug fix | Make NewAddress return the oldest unused address | 344 |
| [Task 399][t399] | open | Syncer recovery initialization and account-scoped Store reads | Bug fix | Isolate persisted targets in zero-lookahead recovery | 345 |
| [Task 402][t402] | open | Wallet synchronization and live-watch readiness | Gap fix | Establish live watches from persisted Wallet data | 331, 345 |
| [Task 412][t412] | open | TxWriter transaction invalidation | Gap fix | Expose transaction invalidation on TxWriter | — |
| [Task 430][t430] | open | AccountManager query integration tests | Test | Cover public account query contracts | — |
| [Task 431][t431] | open | AddressManager allocation and lookup integration tests | Test | Cover address allocation and lookup contracts | — |
| [Task 441][t441] | open | Signer derivation integration tests | Test | Cover public derivation and ECDH contracts | — |
| [Task 442][t442] | open | Signer signature integration tests | Test | Cover public signing contracts | — |
| [Task 443][t443] | open | UnsafeSigner integration tests | Test | Cover UnsafeSigner extraction contracts | — |
| [Task 452][t452] | open | Syncer positive-lookahead account admission and enumeration | Gap fix | Exclude NoChainSync accounts from recovery | 345 |
| [Task 455][t455] | open | Wallet Controller synchronization information | Bug fix | Report exact wallet synchronization status | — |
| [Task 461][t461] | open | AddressManager batch integration tests | Test | Cover public address batch contracts | 343, 344, 468 |
| [Task 465][t465] | open | AddressManager AllocateNextKey integration tests | Test | Verify durable key allocation results | 395, 431 |
| [Task 466][t466] | open | AddressManager final count-one integration tests | Test | Verify final oldest-unused allocation | 396, 431 |
| [Task 468][t468] | open | AccountManager NoChainSync creation admission | Gap fix | Activate NoChainSync account creation | 343, 393, 452 |

## Phase 2 — critical correctness and verification · 23 open

Task 346 is the remaining open critical transaction/PSBT defect; preserve its
owner and PR. Tasks 357 and 404 are completed history. Complete scenario
coverage, remaining component cases, daemon compatibility, and benchmarks.

This is not a blanket migration gate. Tasks 439, 447, and 462–464 also verify
optional later features and retain their real Phase 4 prerequisites; those
cases follow feature implementation without pulling it into Phase 1. A known
defect still gates any adapter or migration path that would actually expose it.

| Task | Status | Component | Type | Outcome | Open dependencies |
| --- | --- | --- | --- | --- | --- |
| [Task 346][t346] | open | PsbtManager packet validation and transformation | Bug fix | Preserve PSBT metadata during funding | — |
| [Task 369][t369] | open | Controller | Test | Reject repeated Wallet unlocks without resetting timeout | — |
| [Task 416][t416] | open | wallet benchmark infrastructure | Benchmark | Provide minimal three-backend benchmark setup | 331 |
| [Task 417][t417] | open | Manager lifecycle and Syncer benchmarks | Benchmark | Measure start-to-synced performance | 331, 416, 455 |
| [Task 418][t418] | open | wallet UtxoManager benchmarks | Benchmark | Measure ListUnspent performance | 416 |
| [Task 419][t419] | open | wallet AddressManager benchmarks | Benchmark | Measure address-read performance | 416 |
| [Task 420][t420] | open | wallet TxReader benchmarks | Benchmark | Measure transaction-read performance | 416 |
| [Task 428][t428] | open | Controller Resync and Rescan integration tests | Test | Cover Controller resync and rescan contracts | 331, 455 |
| [Task 432][t432] | open | AddressManager import and script integration tests | Test | Cover address import and script contracts | — |
| [Task 438][t438] | open | PsbtManager packet integration tests | Test | Cover PSBT decoration and combination contracts | — |
| [Task 439][t439] | open | PsbtManager funding integration tests | Test | Cover PSBT funding contracts | 348 |
| [Task 440][t440] | open | PsbtManager signing integration tests | Test | Cover PSBT signing and finalization contracts | — |
| [Task 444][t444] | open | public wallet receive scenario | Test | Cover the wallet receive and reload scenario | 331, 430, 431 |
| [Task 445][t445] | open | public account signing scenario | Test | Cover account derivation and signing | 331, 431, 441, 442 |
| [Task 446][t446] | open | public transaction spend scenario | Test | Cover transaction creation and publication | 331 |
| [Task 447][t447] | open | public PSBT workflow scenario | Test | Cover the complete PSBT workflow | 331, 438, 439, 440 |
| [Task 448][t448] | open | public Wallet reorganization scenario | Test | Cover synchronization and rollback | 331 |
| [Task 449][t449] | open | multi-Wallet SQL integration scenario | Test | Cover multi-Wallet SQL isolation | 331, 431, 455 |
| [Task 450][t450] | open | bitcoind compatibility integration scenario | Test | Compare public wallet state with bitcoind | 431 |
| [Task 451][t451] | open | integration-test CI matrix and chain daemon provisioning | Test | Run itests across supported chain releases | — |
| [Task 462][t462] | open | AccountManager mutation-error integration tests | Test | Cover public account mutation errors | 391 |
| [Task 463][t463] | open | UtxoManager path-bound account integration tests | Test | Verify path-bound UTXO account identity | 391 |
| [Task 464][t464] | open | AccountManager NoChainSync query integration tests | Test | Verify NoChainSync account queries | 391, 430 |

## Phase 3 — kvdb-to-SQL migration · 5 open

Validate one immutable source, extract its account/address and transaction/sync
state, import it into SQLite, and expose conversion through Manager.Create.
Tasks 335 and 373–376 are one migration sequence, not a backend-retirement
program. Keep modern kvdb usable for the current and next release.

| Task | Status | Component | Type | Outcome | Open dependencies |
| --- | --- | --- | --- | --- | --- |
| [Task 335][t335] | open | Legacy migration source snapshot | Migration | Validate an immutable legacy wallet source | — |
| [Task 373][t373] | open | Legacy migration account, address, and secret extraction | Migration | Extract legacy account and address state | 335 |
| [Task 374][t374] | open | Legacy migration transaction and sync extraction | Migration | Extract legacy transaction and sync state | 373 |
| [Task 375][t375] | open | SQLite migration importer | Migration | Import one legacy snapshot into SQLite | 374 |
| [Task 376][t376] | open | Manager.Create legacy conversion mode | Migration | Convert a legacy wallet through Manager.Create | 331, 358, 375 |

## Phase 4 — remaining wallet gaps and bugs · 40 open

Fix remaining gaps and bugs without treating them as blanket prerequisites for
the initial lnd port or conversion. Claims and PRs remain owned. If concrete
evidence makes one necessary for an enabled path, record that gate or rephase
it explicitly; a component name alone does not make it an integration blocker.

| Task | Status | Component | Type | Outcome | Open dependencies |
| --- | --- | --- | --- | --- | --- |
| [Task 329][t329] | open | Manager SQL genesis and SQL Store | Bug fix | Commit SQL wallet genesis atomically | 345 |
| [Task 330][t330] | open | Manager | Bug fix | Reject duplicate wallet creation | 331 |
| [Task 338][t338] | open | Controller | Bug fix | Latch accepted SQL passphrase results | 331 |
| [Task 347][t347] | open | PsbtManager existing-signature validation | Bug fix | Authorize existing PSBT signatures | 346 |
| [Task 348][t348] | open | PsbtManager and AddressManager | Bug fix | Watch PSBT change before returning funding | 344, 356, 402, 406 |
| [Task 350][t350] | open | PostgreSQL Store | Refactor | Use native pgx for PostgreSQL storage | 454 |
| [Task 351][t351] | open | SQL UTXO and lease schema | Bug fix | Bind SQL UTXOs to wallet tenancy | 424 |
| [Task 352][t352] | open | Controller and LegacyWalletVault | Bug fix | Contain uncertain kvdb passphrase rotation | 331 |
| [Task 356][t356] | open | TxCreator, PsbtManager, and Wallet-owned address storage | Bug fix | Defer change-script allocation until viable | 344 |
| [Task 361][t361] | open | modern kvdb public derivation adapter | Gap fix | Support kvdb imported-XPub derivation | — |
| [Task 362][t362] | open | Controller and LegacyWalletVault | Bug fix | Return a definitive kvdb passphrase result | 338, 352 |
| [Task 363][t363] | open | recovery horizon derivation | Bug fix | Bound aggregate recovery derivation work | — |
| [Task 365][t365] | open | Manager creation preflight | Bug fix | Validate wallet genesis key identities | — |
| [Task 366][t366] | open | Manager and SQL Store genesis result | Bug fix | Reconcile ambiguous SQL genesis commits | 329, 358 |
| [Task 367][t367] | open | Manager and WalletVault genesis preparation | Bug fix | Bound SQL genesis secret lifetime | 329 |
| [Task 370][t370] | open | Manager chain-event fanout and Wallet processors | Refactor | Fan out ordered Manager chain events | 331 |
| [Task 371][t371] | open | SQL Store canonical chain state | Bug fix | Serialize canonical chain-frontier mutation | — |
| [Task 372][t372] | open | Manager chain dispatcher and SQL Store rollback | Bug fix | Coordinate multi-Wallet rollback | 370, 371 |
| [Task 388][t388] | open | AccountStore account-creation admission | Bug fix | Reject colliding runtime account XPub identities | 343 |
| [Task 389][t389] | open | AccountManager and SQL AccountStore account result reader | Bug fix | Reconcile ambiguous runtime account commits | 343, 390 |
| [Task 391][t391] | open | AccountManager and SQL account, address, and UTXO identity shape | Gap fix | Create exact path-bound XPub accounts | 388, 389, 390, 392, 395 |
| [Task 397][t397] | open | ADR 0004 targeted-recovery contract | Design | Define semantic-account recovery targets | — |
| [Task 398][t398] | open | Controller and Syncer recovery account selection | Gap fix | Accept semantic account recovery targets | 345, 397, 399, 452 |
| [Task 403][t403] | open | AddressManager SQL raw-import mutation | Bug fix | Register raw imports before public success | 402 |
| [Task 405][t405] | open | PsbtManager and UTXO Store funding admission | Gap fix | Reserve PSBT funding inputs atomically | 346, 356 |
| [Task 406][t406] | open | PsbtManager private failure cleanup and UTXO Store | Gap fix | Clean up failed PSBT funding leases | 405 |
| [Task 407][t407] | open | PsbtManager FinalizePsbt | Bug fix | Authorize PSBT auto-finalization | 469 |
| [Task 408][t408] | open | TxCreator and AddressManager | Bug fix | Watch direct transaction change before return | 344, 356, 402 |
| [Task 409][t409] | open | SQL TxStore input-graph representation | Refactor | Simplify transaction input graph reads | 424 |
| [Task 410][t410] | open | Wallet-owned SQL transaction mutation ordering | Bug fix | Coordinate transaction graph mutations | 424 |
| [Task 411][t411] | open | Wallet-owned SQL transaction ingestion | Bug fix | Reject children of terminal transactions | 410 |
| [Task 413][t413] | open | Wallet-owned SQL UTXO spend assignment | Bug fix | Validate exact UTXO spending inputs | 351, 410 |
| [Task 414][t414] | open | SQL UTXO amount integrity backstop | Refactor | Bound persisted SQL UTXO amounts | 424 |
| [Task 423][t423] | open | SQL Manager chain source and dispatcher wiring | Refactor | Cut over Manager chain notification ownership | 370, 371, 372, 402, 457 |
| [Task 424][t424] | open | ADR 0006 transaction and UTXO integrity contract | Design | Define transaction and UTXO integrity ownership | — |
| [Task 426][t426] | open | SQL TxStore ingestion amount validation | Bug fix | Validate persisted transaction output aggregates | 424 |
| [Task 454][t454] | open | PostgreSQL Store transaction runtime | Bug fix | Make PostgreSQL transaction isolation explicit | — |
| [Task 457][t457] | open | Manager Start and process-local database ownership | Gap fix | Enforce one process-local Manager per database | 331 |
| [Task 460][t460] | open | AccountManager sequential creation and SQL AccountStore cursor CAS | Bug fix | Make sequential account allocation invalid-child safe | 389 |
| [Task 469][t469] | open | PsbtManager and Signer effective-key signing | Bug fix | Sign PSBT inputs with effective keys | 347, 390 |

## Phase 5 — Signer and Vault · 6 open

Refine secret-operation, derivation, signing, and raw-key ownership boundaries,
then remove superseded secret seams. Existing signing APIs do not wait for
these refactors. A critical signing defect belongs with the affected delivery
path, not here merely because it touches Signer or Vault.

| Task | Status | Component | Type | Outcome | Open dependencies |
| --- | --- | --- | --- | --- | --- |
| [Task 354][t354] | open | ADR 0010 secret-use contract | Design | Define Vault secret-operation ownership | — |
| [Task 382][t382] | open | AccountManager and account-derivation Vault operation | Refactor | Move account derivation into the Vault | 352, 354, 390 |
| [Task 383][t383] | open | AddressManager ScriptForOutput and script Vault operation | Refactor | Resolve output scripts through the Vault | 352, 354 |
| [Task 384][t384] | open | Signer and primitive key-use Vault operations | Refactor | Route safe signing through the Vault | 352, 354, 383, 469 |
| [Task 385][t385] | open | UnsafeSigner raw private-key boundary | Refactor | Isolate UnsafeSigner key extraction | 384 |
| [Task 386][t386] | open | Wallet-facing Vault and runtimeCache secret interfaces | Refactor | Remove raw Wallet secret seams | 382, 383, 384, 385 |

## Conditional route state

Modern kvdb remains supported for the current release (N) and the next (N+1).
Task 353 is deferred to no earlier than N+2 and still requires verified
migration, consumer cutover, and explicit approval. Migration does not imply
retirement. Keep current kvdb behavior and shared three-backend verification;
do not remove pending kvdb clauses in anticipation of future retirement.

## Maintaining the phases

When a task completes, move its row from the phase table to `Done`, preserve
its delivery phase and record its closing PR there, refresh the counts, and
remove its number from remaining `Open dependencies` cells. An explicit
roadmap rephasing may update both open and completed rows so the `Phase` column
continues to reflect the current thematic map. When dependency edges change,
update the owning task and its open-dependency cell without deriving a new
phase. Every roadmap task must remain listed exactly once.

## Deferred · 1

These tasks remain valid but are outside this release roadmap. They are not
done or superseded; reactivation preserves the original task number.

| Task | Type | Outcome | Reactivation gate |
| --- | --- | --- | --- |
| [Task 353][t353] | Migration | Retire kvdb from the modern wallet API | No earlier than N+2; verified migration, consumer cutover, and explicit retirement approval |

## Superseded · 7

Former phases record the historical map at supersession, not the current
five-phase assignment. These tasks have no current delivery phase.

| Task | Former phase | Component | Type | Outcome | Replacement |
| --- | ---: | --- | --- | --- | --- |
| [Task 336][t336] | 5 | Controller and Wallet passphrase test suite | Test | Remove redundant Wallet passphrase tests | Required test updates remain inside [Task 386][t386] |
| [Task 349][t349] | 3 | TxStore internal invalidation workflow | Gap fix | Implement idempotent transaction invalidation | Existing SQL Store behavior; [Task 412][t412] owns maintained public exposure |
| [Task 364][t364] | 1 | SQL live-watch persistence | Gap fix | Persist ordered SQL live-watch obligations | Accepted fail-stop/startup-reconstruction decision eliminates delivery persistence; [Task 402][t402] retains live registration |
| [Task 381][t381] | 3 | RecoveryState public contract and internal Syncer adapter | Refactor | Hide recovery adapter identity | Accepted decision to retain the current shape until a concrete public need exists |
| [Task 422][t422] | 1 | Manager.Load | Gap fix | Make Manager loading cancellable | [Tasks 331][t331] and [459][t459] |
| [Task 368][t368] | 1 | Maintained Wallet lifecycle public API cutover | Bug fix | Make Wallet lifecycle Manager-internal | [Task 331][t331] |
| [Task 394][t394] | 1 | AddressStore ownership resolution and transaction ingestion | Gap fix | Exclude key-only locators from wallet ownership | Accepted `NoChainSync` ownership decision; [Task 395][t395] owns separate allocation only |

## Done · 37

| Task | Phase | Component | Type | Outcome | Closing PR |
| --- | ---: | --- | --- | --- | --- |
| [Task 332][t332] | 1 | Manager, AccountManager, and AddressManager | Gap fix | Define watch-only and script-import policy | [#1320][pr1320] |
| [Task 333][t333] | 1 | Manager and integration-test harness | Gap fix | Add PostgreSQL Manager and harness support | [#1303][pr1303] |
| [Task 334][t334] | 1 | SQL wallet architecture documentation | Design | Define aggregate Manager and shared-chain ownership | [#1323][pr1323] |
| [Task 337][t337] | 1 | Wallet and Signer public derivation | Bug fix | Restore SQL imported-XPub public derivation | [#1324][pr1324] |
| [Task 339][t339] | 1 | TxReader | Gap fix | Define public transaction visibility and status | [#1330][pr1330] |
| [Task 340][t340] | 1 | Manager and SQL Store | Bug fix | Bind SQL databases to schema and network identity | [#1345][pr1345] |
| [Task 341][t341] | 1 | wallet-owned account identity types | Gap fix | Define semantic account identity | [#1313][pr1313] |
| [Task 342][t342] | 1 | wallet configuration | Bug fix | Accept bounded recovery windows | [#1335][pr1335] |
| [Task 345][t345] | 1 | SQL account schema and AccountStore persistence | Gap fix | Persist SQL account NoChainSync state | [#1365][pr1365] |
| [Task 355][t355] | 2 | TxCreator and the `txsizes` module | Bug fix | Correct transaction size accounting | [#1338][pr1338] |
| [Task 357][t357] | 2 | `txauthor` and TxCreator | Bug fix | Validate transaction output and fee arithmetic | [#1348][pr1348] |
| [Task 359][t359] | 1 | Manager and SQL Store design | Design | Revalidate database identity design | [#1319][pr1319] |
| [Task 360][t360] | 1 | ADR 0013 and the modern kvdb adapter contract | Design | Document kvdb adapter identity | [#1308][pr1308] |
| [Task 377][t377] | 1 | AccountManager public result contract | Gap fix | Make AccountManager results wallet-owned | [#1329][pr1329] |
| [Task 378][t378] | 1 | AccountManager public error boundary | Bug fix | Translate AccountManager backend errors | [#1358][pr1358] |
| [Task 379][t379] | 1 | Manager Load public error boundary | Bug fix | Translate Manager load not-found errors | [#1331][pr1331] |
| [Task 380][t380] | 1 | AddressManager GetAddressInfo error boundary | Bug fix | Translate address lookup errors | [#1332][pr1332] |
| [Task 387][t387] | 1 | ADR 0012 and ADR 0013 account-creation contract | Design | Define account identity and chain-sync policy | [#1342][pr1342] |
| [Task 392][t392] | 1 | AccountManager and SQL AccountStore create inputs | Gap fix | Carry NoChainSync through account creation | [#1366][pr1366] |
| [Task 400][t400] | 1 | ADR 0004 and ADR 0005 synchronization contract | Design | Separate live watches from historical recovery | [#1322][pr1322] |
| [Task 401][t401] | 1 | chain interface and Neutrino/btcd/bitcoind adapters | Gap fix | Add an idempotent live-tip-only chain watch | [#1344][pr1344] |
| [Task 404][t404] | 2 | TxCreator and PsbtManager input-source adapters | Bug fix | Validate Wallet transaction input amounts | [#1357][pr1357] |
| [Task 415][t415] | 1 | UtxoManager integration tests | Test | Cover public UTXO contracts end to end | [#1307][pr1307] |
| [Task 421][t421] | 2 | WalletVault and SQL wallet-secret persistence | Bug fix | Reconcile SQL passphrase commits | [#1341][pr1341] |
| [Task 425][t425] | 2 | Wallet transaction-authoring wrapper boundary | Refactor | Isolate shared transaction authoring from wrapper watch and lease effects | [#1327][pr1327] |
| [Task 427][t427] | 1 | Controller credential integration tests | Test | Cover remaining Controller credential contracts | [#1349][pr1349] |
| [Task 429][t429] | 1 | AccountManager mutation integration tests | Test | Cover public account mutation contracts | [#1326][pr1326] |
| [Task 433][t433] | 1 | Wallet sync-state publication and integration-test harness | Test | Enforce the wallet sync test boundary | [#1314][pr1314] |
| [Task 434][t434] | 1 | TxCreator integration tests | Test | Cover transaction creation contracts | [#1316][pr1316] |
| [Task 435][t435] | 1 | TxPublisher integration tests | Test | Cover transaction publication contracts | [#1334][pr1334] |
| [Task 436][t436] | 1 | TxReader integration tests | Test | Cover transaction read contracts | [#1346][pr1346] |
| [Task 437][t437] | 1 | TxWriter integration tests | Test | Cover transaction label contracts | [#1347][pr1347] |
| [Task 453][t453] | 1 | Manager Wallet cache publication | Bug fix | Serialize Manager Wallet publication | [#1336][pr1336] |
| [Task 456][t456] | 1 | ManagerConfig and managed Wallet construction | Refactor | Centralize managed Wallet runtime configuration | [#1353][pr1353] |
| [Task 458][t458] | 1 | Wallet lifecycle state and maintained public Wallet components | Bug fix | Drain Wallet calls during terminal stop | [#1354][pr1354] |
| [Task 459][t459] | 1 | Manager startup Wallet discovery and assembly | Gap fix | Assemble durable Wallets for Manager startup | [#1356][pr1356] |
| [Task 467][t467] | 1 | wallet public durable-mutation error identity | Gap fix | Define the indeterminate commit identity | [#1355][pr1355] |

[t329]: tasks/329-atomic-wallet-account-creation.md
[t330]: tasks/330-duplicate-wallet-create.md
[t331]: tasks/331-manager-wallet-lifecycle.md
[t332]: archive/tasks/332-watch-only-script-policy.md
[t333]: archive/tasks/333-postgres-manager-harness.md
[t334]: archive/tasks/334-shared-chain-state.md
[t335]: tasks/335-kvdb-to-sql-migration.md
[t336]: archive/tasks/336-test-cleanup.md
[t337]: archive/tasks/337-imported-xpub-derivation.md
[t338]: tasks/338-passphrase-result.md
[t339]: archive/tasks/339-transaction-visibility.md
[t340]: archive/tasks/340-durable-database-identity.md
[t341]: archive/tasks/341-public-role-contract.md
[t342]: archive/tasks/342-recovery-window-validation.md
[t343]: tasks/343-exact-account-creation.md
[t344]: tasks/344-bulk-address-allocation.md
[t345]: tasks/345-durable-watch-registration.md
[t346]: tasks/346-psbt-funding-metadata.md
[t347]: tasks/347-psbt-signing-authorization.md
[t348]: tasks/348-watched-change-delivery.md
[t349]: archive/tasks/349-transaction-invalidation.md
[t350]: tasks/350-native-pgx-postgres.md
[t351]: tasks/351-sql-utxo-integrity.md
[t352]: tasks/352-kvdb-combined-passphrase-rotation.md
[t353]: archive/tasks/353-retire-modern-kvdb-backend.md
[t354]: tasks/354-vault-owned-secret-resolution.md
[t355]: archive/tasks/355-transaction-size-accounting.md
[t356]: tasks/356-defer-change-script-allocation.md
[t357]: archive/tasks/357-transaction-amount-validation.md
[t358]: tasks/358-contextual-manager-operations.md
[t359]: archive/tasks/359-revalidate-database-identity-design.md
[t360]: archive/tasks/360-document-kvdb-adapter-identity.md
[t361]: tasks/361-kvdb-imported-xpub-derivation.md
[t362]: tasks/362-kvdb-definitive-passphrase-result.md
[t363]: tasks/363-recovery-scan-work-bound.md
[t364]: archive/tasks/364-durable-watch-registration.md
[t365]: tasks/365-validate-wallet-genesis-keys.md
[t366]: tasks/366-reconcile-sql-genesis-commit.md
[t367]: tasks/367-bound-sql-genesis-secrets.md
[t368]: archive/tasks/368-wallet-startup-shutdown.md
[t369]: tasks/369-repeated-wallet-unlock.md
[t370]: tasks/370-chain-source-delivery.md
[t371]: tasks/371-canonical-chain-frontier.md
[t372]: tasks/372-multi-wallet-rollback.md
[t373]: tasks/373-extract-legacy-account-address-state.md
[t374]: tasks/374-extract-legacy-transaction-sync-state.md
[t375]: tasks/375-import-legacy-snapshot-sqlite.md
[t376]: tasks/376-orchestrate-legacy-conversion.md
[t377]: archive/tasks/377-account-manager-wallet-dto.md
[t378]: archive/tasks/378-account-manager-error-boundary.md
[t379]: archive/tasks/379-manager-load-error-boundary.md
[t380]: archive/tasks/380-address-lookup-error-boundary.md
[t381]: archive/tasks/381-recovery-adapter-identity.md
[t382]: tasks/382-vault-account-derivation.md
[t383]: tasks/383-vault-output-script-resolution.md
[t384]: tasks/384-vault-safe-signing.md
[t385]: tasks/385-unsafe-signer-key-extraction.md
[t386]: tasks/386-remove-raw-wallet-secret-seams.md
[t387]: archive/tasks/387-account-creation-identity-adr.md
[t388]: tasks/388-runtime-account-xpub-collisions.md
[t389]: tasks/389-runtime-account-commit-reconciliation.md
[t390]: tasks/390-custom-scope-account-creation.md
[t391]: tasks/391-path-bound-watch-only-xpub-accounts.md
[t392]: tasks/392-discovery-mode-account-creation.md
[t393]: tasks/393-key-only-receiving-rejection.md
[t394]: archive/tasks/394-key-only-transaction-ownership.md
[t395]: tasks/395-key-only-locator-allocation.md
[t396]: tasks/396-oldest-unused-address.md
[t397]: tasks/397-semantic-recovery-targets-adr.md
[t398]: tasks/398-semantic-account-recovery-targets.md
[t399]: tasks/399-zero-lookahead-persisted-targets.md
[t400]: archive/tasks/400-live-watch-semantics-adr.md
[t401]: archive/tasks/401-live-tip-chain-watch.md
[t402]: tasks/402-replay-live-watch-obligations.md
[t403]: tasks/403-raw-import-watch-obligations.md
[t404]: archive/tasks/404-transaction-input-amounts.md
[t405]: tasks/405-psbt-funding-lease-admission.md
[t406]: tasks/406-psbt-funding-lease-release.md
[t407]: tasks/407-psbt-finalization-authorization.md
[t408]: tasks/408-direct-transaction-change-watch.md
[t409]: tasks/409-transaction-input-graph.md
[t410]: tasks/410-transaction-graph-admission.md
[t411]: tasks/411-terminal-parent-rejection.md
[t412]: tasks/412-transaction-invalidation.md
[t413]: tasks/413-utxo-spend-identity.md
[t414]: tasks/414-utxo-amount-bounds.md
[t415]: archive/tasks/415-utxo-public-integration-tests.md
[t416]: tasks/416-database-benchmark-fixtures.md
[t417]: tasks/417-start-to-synced-benchmark.md
[t418]: tasks/418-list-unspent-benchmark.md
[t419]: tasks/419-address-read-benchmark.md
[t420]: tasks/420-transaction-read-benchmark.md
[t421]: archive/tasks/421-sql-passphrase-commit-reconciliation.md
[t422]: archive/tasks/422-manager-load-cancellation.md
[t423]: tasks/423-manager-chain-source-cutover.md
[t424]: tasks/424-transaction-utxo-integrity-adr.md
[t425]: archive/tasks/425-separate-transaction-authoring-wrappers.md
[t426]: tasks/426-sql-transaction-output-aggregates.md
[t427]: archive/tasks/427-controller-lifecycle-credential-itests.md
[t428]: tasks/428-controller-sync-info-itests.md
[t429]: archive/tasks/429-account-mutation-itests.md
[t430]: tasks/430-account-query-itests.md
[t431]: tasks/431-address-allocation-lookup-itests.md
[t432]: tasks/432-address-import-script-itests.md
[t433]: archive/tasks/433-wallet-sync-test-boundary.md
[t434]: archive/tasks/434-transaction-creation-itests.md
[t435]: archive/tasks/435-transaction-publication-itests.md
[t436]: archive/tasks/436-transaction-read-itests.md
[t437]: archive/tasks/437-transaction-label-itests.md
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
[t453]: archive/tasks/453-serialize-manager-wallet-publication.md
[t454]: tasks/454-postgres-transaction-isolation.md
[t455]: tasks/455-exact-wallet-sync-status.md
[t456]: archive/tasks/456-manager-runtime-configuration.md
[t457]: tasks/457-process-local-manager-lease.md
[t458]: archive/tasks/458-manager-operation-admission.md
[t459]: tasks/459-manager-startup-wallet-assembly.md
[t460]: tasks/460-sequential-account-invalid-children.md
[t461]: tasks/461-address-batch-itests.md
[t462]: tasks/462-account-mutation-error-itests.md
[t463]: tasks/463-path-bound-utxo-identity-itests.md
[t464]: tasks/464-no-chain-sync-account-query-itests.md
[t465]: tasks/465-new-address-result-itests.md
[t466]: tasks/466-oldest-unused-address-itests.md
[t467]: archive/tasks/467-indeterminate-commit-identity.md
[t468]: tasks/468-activate-no-chain-sync-account-creation.md
[t469]: tasks/469-sign-psbt-with-effective-keys.md
[pr1303]: https://github.com/btcsuite/btcwallet/pull/1303
[pr1307]: https://github.com/btcsuite/btcwallet/pull/1307
[pr1308]: https://github.com/btcsuite/btcwallet/pull/1308
[pr1313]: https://github.com/btcsuite/btcwallet/pull/1313
[pr1314]: https://github.com/btcsuite/btcwallet/pull/1314
[pr1316]: https://github.com/btcsuite/btcwallet/pull/1316
[pr1319]: https://github.com/btcsuite/btcwallet/pull/1319
[pr1320]: https://github.com/btcsuite/btcwallet/pull/1320
[pr1322]: https://github.com/btcsuite/btcwallet/pull/1322
[pr1323]: https://github.com/btcsuite/btcwallet/pull/1323
[pr1324]: https://github.com/btcsuite/btcwallet/pull/1324
[pr1326]: https://github.com/btcsuite/btcwallet/pull/1326
[pr1327]: https://github.com/btcsuite/btcwallet/pull/1327
[pr1329]: https://github.com/btcsuite/btcwallet/pull/1329
[pr1330]: https://github.com/btcsuite/btcwallet/pull/1330
[pr1331]: https://github.com/btcsuite/btcwallet/pull/1331
[pr1332]: https://github.com/btcsuite/btcwallet/pull/1332
[pr1334]: https://github.com/btcsuite/btcwallet/pull/1334
[pr1335]: https://github.com/btcsuite/btcwallet/pull/1335
[pr1336]: https://github.com/btcsuite/btcwallet/pull/1336
[pr1338]: https://github.com/btcsuite/btcwallet/pull/1338
[pr1341]: https://github.com/btcsuite/btcwallet/pull/1341
[pr1342]: https://github.com/btcsuite/btcwallet/pull/1342
[pr1344]: https://github.com/btcsuite/btcwallet/pull/1344
[pr1345]: https://github.com/btcsuite/btcwallet/pull/1345
[pr1346]: https://github.com/btcsuite/btcwallet/pull/1346
[pr1347]: https://github.com/btcsuite/btcwallet/pull/1347
[pr1348]: https://github.com/btcsuite/btcwallet/pull/1348
[pr1349]: https://github.com/btcsuite/btcwallet/pull/1349
[pr1353]: https://github.com/btcsuite/btcwallet/pull/1353
[pr1354]: https://github.com/btcsuite/btcwallet/pull/1354
[pr1355]: https://github.com/btcsuite/btcwallet/pull/1355
[pr1357]: https://github.com/btcsuite/btcwallet/pull/1357
[pr1358]: https://github.com/btcsuite/btcwallet/pull/1358
