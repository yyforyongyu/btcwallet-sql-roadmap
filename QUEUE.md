# Task delivery phases

The numbered roadmap contains **159 tasks**: **75 open**, **61 done**,
**21 superseded**, and **2 deferred**. With **118 ad hoc tasks**, that is
**277 tracked tasks** and **179 completed** overall.

Task files own scope and dependencies; [ACTIVE.md](ACTIVE.md) owns claims and
PRs. Tables show open dependencies only. See [AGENTS.md](AGENTS.md) for task
completion and ad hoc accounting.

Phase 1 is the active focus; all open 2A fixes and 2B verification are tracked
as upcoming work. The completion goal includes both Phase 2 tracks, which
may advance in parallel as dependencies allow. Phase 3 and 4 work starts only
if already underway or owner-approved; neither waits for all of Phase 2.
Hardening starts only after 2A and 2B finish. Placement changes no task scope,
dependency, claim, or PR.

Task 487 owns lnd's Phase 1 compile cutover after the completed pre-claim gap
check, which mapped transaction notifications, standalone signing, online
remote signing, and kvdb caller paths against source. Its compile-only PR
merges into `yyforyongyu/lnd:btcwallet-sql-integration` at a pinned,
build-verified revision. Task 486 consolidates the initial SQL schema before
sql-wallet merges into master. Task 470 retains full runtime and itest
parity as its later lnd-main merge gate; those tests do not gate the
sql-wallet-to-master merge.

## Phase 1 — final SQL schema and lnd compilation · 8 open

Finish the planned SQL account, import, and lease schema changes, then
consolidate them into a clean initial migration for SQLite and PostgreSQL.
Retain the existing migration loader and verify fresh creation and unchanged
reopen. Build lnd against the maintained Wallet without requiring its runtime
itests to pass. Keep already assigned Phase 1 work, including its focused
public tests.

Finish Phase 1 before merging `sql-wallet` into `master`. Later data conversion
targets this consolidated initial schema; it does not require a post-merge
schema upgrade.

The initial port keeps Task 331's non-contextual Create signature and safe
shutdown draining. Contextual creation follows atomic genesis in Phase 3.

Keep the complete prerequisite closure. Task 395 reuses Task 344's durable
child allocation for `AllocateNextKey`; it does not depend on Task 396's
receiving-address semantics. Task 402 establishes btcd/bitcoind watches from
Wallet facts, without a ledger or completion flags. Lost required readiness
stops the Manager; Neutrino fixes and cache optimization remain out of scope.

| Task | Status | Component | Type | Outcome | Open dependencies |
| --- | --- | --- | --- | --- | --- |
| [Task 391][t391] | open | AccountManager and SQL account, address, and UTXO identity shape | Gap fix | Create exact path-bound XPub accounts | 388, 395 |
| [Task 403][t403] | open | AddressManager SQL raw-import mutation | Bug fix | Register raw imports before public success | 402 |
| [Task 475][t475] | open | Wallet transaction notifications | Gap fix | Deliver wallet transaction subscriptions | — |
| [Task 478][t478] | open | UtxoManager lease policy and stores | Gap fix | Retain spend-confirmation leases | — |
| [Task 479][t479] | open | AccountManager custody and Wallet results | Gap fix | Mix imported XPub and signing accounts | — |
| [Task 480][t480] | open | AddressManager raw imports and SQL address ownership | Gap fix | Own raw imports in one signing wallet | 403, 479 |
| [Task 486][t486] | open | SQLite and PostgreSQL schema initialization | Migration | Consolidate initial SQL schema before master merge | 391, 478, 480, 481 |
| [Task 487][t487] | open | lnd Wallet dependency and adapter build | Gap fix | Compile lnd with the maintained Wallet | 391, 395, 396, 475, 478 |

## Phase 2A — bugs and gaps · 22 open

Prioritize correctness failures that block normal supported operation,
including Manager fan-out; then address the remaining known bugs and gaps.
This is a prioritized backlog, not a blanket gate for starting the lnd port.
Tasks 424 and 410 supply the integrity decision and graph correction needed
by Task 454's isolation policy.

| Task | Status | Component | Type | Outcome | Open dependencies |
| --- | --- | --- | --- | --- | --- |
| [Task 330][t330] | open | Manager | Bug fix | Reject duplicate wallet creation | — |
| [Task 338][t338] | open | Controller | Bug fix | Latch accepted passphrase results | — |
| [Task 356][t356] | open | TxCreator, PsbtManager, and Wallet-owned address storage | Bug fix | Defer change-script allocation until viable | — |
| [Task 365][t365] | open | Manager creation preflight | Bug fix | Validate wallet genesis key identities | — |
| [Task 367][t367] | open | Manager and WalletVault genesis preparation | Bug fix | Bound SQL genesis secret lifetime | — |
| [Task 370][t370] | open | Manager chain-event fanout and Wallet processors | Refactor | Fan out ordered Manager chain events | — |
| [Task 372][t372] | open | Manager chain dispatcher and SQL Store rollback | Bug fix | Coordinate multi-Wallet rollback | 370 |
| [Task 389][t389] | open | AccountManager SQL account-creation results | Bug fix | Report uncertain account commits truthfully | — |
| [Task 405][t405] | open | PsbtManager and UTXO Store funding admission | Gap fix | Reserve PSBT funding inputs atomically | — |
| [Task 406][t406] | open | PsbtManager private failure cleanup and UTXO Store | Gap fix | Clean up failed PSBT funding leases | 405 |
| [Task 407][t407] | open | PsbtManager FinalizePsbt | Bug fix | Authorize PSBT auto-finalization | 469 |
| [Task 410][t410] | open | Wallet-owned SQL transaction mutation ordering | Bug fix | Coordinate transaction graph mutations | 424 |
| [Task 423][t423] | open | SQL Manager chain source and dispatcher wiring | Refactor | Cut over Manager chain notification ownership | 370, 372, 402 |
| [Task 424][t424] | open | ADR 0006 transaction and UTXO integrity contract | Design | Define transaction and UTXO integrity ownership | — |
| [Task 454][t454] | open | PostgreSQL Store transaction runtime | Bug fix | Make PostgreSQL transaction isolation explicit | 410 |
| [Task 469][t469] | open | PsbtManager and Signer effective-key signing | Bug fix | Sign PSBT inputs with effective keys | 347 |
| [Task 470][t470] | open | lnd Wallet integration | Gap fix | Complete lnd Wallet runtime parity | 344, 356, 391, 395, 396, 406, 474, 475, 478–484, 487 |
| [Task 474][t474] | open | PsbtManager and TxCreator change registration | Bug fix | Watch authored change before returning | 402 |
| [Task 482][t482] | open | PsbtManager imported-input decoration | Gap fix | Fund imported inputs without known origin | 479 |
| [Task 483][t483] | open | Wallet scan ingestion and TxStore batch | Bug fix | Preserve labels on chain replay | — |
| [Task 484][t484] | open | TxCreator coin eligibility | Bug fix | Align authoring confirmation height | — |

## Phase 2B — verification · 23 open

Integration tests, scenarios, compatibility, and benchmarks run alongside 2A.
Reuse existing component proofs; scenarios own distinct public handoffs.
Final-contract suites retain their required production prerequisites,
including later-phase features.

| Task | Status | Component | Type | Outcome | Open dependencies |
| --- | --- | --- | --- | --- | --- |
| [Task 369][t369] | open | Controller | Test | Reject repeated Wallet unlocks without resetting timeout | — |
| [Task 417][t417] | open | Manager lifecycle and Syncer benchmarks | Benchmark | Measure start-to-synced performance | 471 |
| [Task 419][t419] | open | wallet AddressManager benchmarks | Benchmark | Measure address-read performance | 471 |
| [Task 420][t420] | open | wallet TxReader benchmarks | Benchmark | Measure transaction-read performance | 471 |
| [Task 428][t428] | open | Controller Resync and Rescan integration tests | Test | Cover Controller resync and rescan contracts | — |
| [Task 432][t432] | open | AddressManager import and script integration tests | Test | Cover address import and script contracts | — |
| [Task 438][t438] | open | PsbtManager packet integration tests | Test | Cover PSBT decoration and combination contracts | — |
| [Task 439][t439] | open | PsbtManager funding integration tests | Test | Cover PSBT funding contracts | 356, 406, 474 |
| [Task 440][t440] | open | PsbtManager signing integration tests | Test | Cover PSBT signing and finalization contracts | 407 |
| [Task 444][t444] | open | public wallet receive scenario | Test | Cover new-account receipt and reload | — |
| [Task 445][t445] | open | public account signing scenario | Test | Cover new-account signing identity | — |
| [Task 446][t446] | open | public transaction spend scenario | Test | Cover authored-spend publication and reload | — |
| [Task 447][t447] | open | public PSBT workflow scenario | Test | Cover the complete PSBT workflow | 356, 406, 474 |
| [Task 448][t448] | open | public Wallet reorganization scenario | Test | Cover synchronization and rollback | — |
| [Task 449][t449] | open | multi-Wallet SQL integration scenario | Test | Cover multi-Wallet SQL isolation | 423 |
| [Task 450][t450] | open | bitcoind compatibility integration scenario | Test | Compare public wallet state with bitcoind | — |
| [Task 451][t451] | open | integration-test CI matrix and chain daemon provisioning | Test | Run itests across supported chain releases | — |
| [Task 462][t462] | open | AccountManager mutation-error integration tests | Test | Complete missing account mutation error evidence | 391 |
| [Task 463][t463] | open | UtxoManager path-bound account integration tests | Test | Verify path-bound UTXO account identity | 391 |
| [Task 464][t464] | open | AccountManager NoChainSync query integration tests | Test | Verify true-value lists and path-bound account queries | 391 |
| [Task 466][t466] | open | AddressManager final count-one integration tests | Test | Verify final oldest-unused allocation | 396 |
| [Task 471][t471] | open | wallet UtxoManager benchmarks and shared benchmark setup | Benchmark | Measure ListUnspent on three real backends | — |
| [Task 472][t472] | open | multi-Wallet SQL rescan integration scenario | Test | Isolate historical rescans from live Wallets | 423 |

## Phase 3 — legacy-to-native SQL migration · 11 open

Validate and extract immutable bbolt and legacy SQL sources, import into
native SQLite or PostgreSQL, then expose verified conversion. Atomic genesis
and contextual Create remain prerequisites.
Modern kvdb remains supported for the current and next release.

| Task | Status | Component | Type | Outcome | Open dependencies |
| --- | --- | --- | --- | --- | --- |
| [Task 329][t329] | open | Manager SQL genesis and SQL Store | Bug fix | Commit SQL wallet genesis atomically | — |
| [Task 335][t335] | open | Bbolt migration source snapshot | Migration | Validate an immutable bbolt wallet source | — |
| [Task 358][t358] | open | Manager.Create | Gap fix | Make Manager creation cancellable | 329 |
| [Task 373][t373] | open | Legacy migration wallet, account, and secret extraction | Migration | Extract legacy wallet, account, and secret state | 335 |
| [Task 374][t374] | open | Legacy migration transaction and sync extraction | Migration | Extract legacy transaction and sync state | 473 |
| [Task 375][t375] | open | SQLite migration importer | Migration | Import one legacy snapshot into SQLite | 374, 478, 480, 481 |
| [Task 376][t376] | open | Manager.Create legacy conversion mode | Migration | Convert a legacy wallet through Manager.Create | 358, 375 |
| [Task 473][t473] | open | Legacy migration address and script extraction | Migration | Extract legacy address and script state | 373 |
| [Task 476][t476] | open | Legacy migration source adapter | Migration | Read legacy SQL walletdb sources | 374 |
| [Task 477][t477] | open | PostgreSQL migration importer | Migration | Import a legacy snapshot into PostgreSQL | 374, 478, 480, 481 |
| [Task 485][t485] | open | Manager.Create legacy conversion selection | Migration | Convert legacy SQL wallets through Manager | 376, 476, 477 |

## Phase 4 — optional improvements · 5 open

Additional capabilities, performance refactors, and defensive integrity work.
Task types, scopes, and evidence requirements are unchanged.

| Task | Status | Component | Type | Outcome | Open dependencies |
| --- | --- | --- | --- | --- | --- |
| [Task 350][t350] | open | PostgreSQL Store | Refactor | Use native pgx for PostgreSQL storage | — |
| [Task 397][t397] | open | ADR 0004 targeted-recovery contract | Design | Define semantic-account recovery targets | — |
| [Task 398][t398] | open | Controller and Syncer recovery account selection | Gap fix | Accept semantic account recovery targets | 397 |
| [Task 411][t411] | open | Wallet-owned SQL transaction ingestion | Bug fix | Reject children of terminal transactions | 410 |
| [Task 426][t426] | open | Manager Wallet publication and SQL TxStore amount validation | Bug fix | Validate persisted transaction output aggregates | — |

## Hardening — Signer, Vault, and defensive edges · 7 open

Start only after both 2A and 2B are complete. Existing signing bugs remain in
the core backlog. This category holds later Signer/Vault restructuring and
optional duplicate-Manager and rare derivation-edge protection.

| Task | Status | Component | Type | Outcome | Open dependencies |
| --- | --- | --- | --- | --- | --- |
| [Task 354][t354] | open | ADR 0010 secret-use contract | Design | Define Vault secret-operation ownership | — |
| [Task 382][t382] | open | AccountManager and account-derivation Vault operation | Refactor | Move account derivation into the Vault | 354 |
| [Task 383][t383] | open | AddressManager ScriptForOutput and script Vault operation | Refactor | Resolve output scripts through the Vault | 354 |
| [Task 384][t384] | open | Signer and primitive key-use Vault operations | Refactor | Route safe signing through the Vault | 354, 383, 469 |
| [Task 385][t385] | open | UnsafeSigner raw private-key boundary | Refactor | Isolate UnsafeSigner key extraction | 384 |
| [Task 457][t457] | open | Manager Start and process-local database ownership | Feature | Enforce one process-local Manager per database | — |
| [Task 460][t460] | open | AccountManager sequential creation and SQL AccountStore cursor | Bug fix | Make sequential account allocation invalid-child safe | — |

## Conditional route state

Modern kvdb remains supported for the current release (N) and the next (N+1).
Task 353 is deferred to no earlier than N+2 and still requires verified
migration, consumer cutover, and explicit approval. Migration does not imply
retirement. Keep current kvdb behavior and shared three-backend verification;
do not remove pending kvdb clauses in anticipation of future retirement.

## Maintaining the roadmap

Follow [AGENTS.md](AGENTS.md#completing-a-task) when completing a task. Preserve
its phase, track, or category in Done and list each task exactly once. Older
Done and Superseded placements remain historical; do not rephase them here.

## Deferred · 2

These tasks remain valid but are outside this release roadmap. They are not
done or superseded; reactivation preserves the original task number.

| Task | Type | Outcome | Reactivation gate |
| --- | --- | --- | --- |
| [Task 352][t352] | Bug fix | Contain uncertain kvdb passphrase rotation | Explicit approval for a future-cycle kvdb passphrase-containment effort |
| [Task 353][t353] | Migration | Retire kvdb from the modern wallet API | No earlier than N+2; verified migration, consumer cutover, and explicit retirement approval |

## Superseded · 21

Former phases record the historical map at supersession, not the current
delivery grouping. These tasks have no current delivery phase.

| Task | Former phase | Component | Type | Outcome | Replacement |
| --- | ---: | --- | --- | --- | --- |
| [Task 336][t336] | 5 | Controller and Wallet passphrase test suite | Test | Remove redundant Wallet passphrase tests | Required test updates stay with their production changes in Tasks 382–385 |
| [Task 349][t349] | 3 | TxStore internal invalidation workflow | Gap fix | Implement idempotent transaction invalidation | Existing SQL Store behavior; [Task 412][t412] owns maintained public exposure |
| [Task 364][t364] | 1 | SQL live-watch persistence | Gap fix | Persist ordered SQL live-watch obligations | Accepted fail-stop/startup-reconstruction decision eliminates delivery persistence; [Task 402][t402] retains live registration |
| [Task 381][t381] | 3 | RecoveryState public contract and internal Syncer adapter | Refactor | Hide recovery adapter identity | Accepted decision to retain the current shape until a concrete public need exists |
| [Task 422][t422] | 1 | Manager.Load | Gap fix | Make Manager loading cancellable | [Tasks 331][t331] and [459][t459] |
| [Task 368][t368] | 1 | Maintained Wallet lifecycle public API cutover | Bug fix | Make Wallet lifecycle Manager-internal | [Task 331][t331] |
| [Task 394][t394] | 1 | AddressStore ownership resolution and transaction ingestion | Gap fix | Exclude key-only locators from wallet ownership | Accepted `NoChainSync` ownership decision; [Task 395][t395] owns separate allocation only |
| [Task 351][t351] | 4 | SQL UTXO and lease schema | Bug fix | Bind SQL UTXOs to wallet tenancy | Accepted decision to preserve existing tenancy triggers and scoped writers; no new tenancy representation |
| [Task 361][t361] | 2A | modern kvdb public derivation adapter | Gap fix | Support kvdb imported-XPub derivation | Existing Tasks 337/441 behavior and shared itest eliminate duplicate kvdb work; no new completion credit |
| [Task 362][t362] | 2A | Controller and LegacyWalletVault | Bug fix | Return a definitive kvdb passphrase result | Accepted decision to remove the separate backend delivery layer; shared result handling stays in [Task 338][t338] and kvdb containment in [Task 352][t352] |
| [Task 409][t409] | 4 | SQL TxStore input-graph representation | Refactor | Simplify transaction input graph reads | Existing raw graph reads satisfy current consumers; accepted decision eliminates another representation |
| [Task 413][t413] | 4 | Wallet-owned SQL UTXO spend assignment | Bug fix | Validate exact UTXO spending inputs | Existing exact-outpoint checks and replay regression eliminate duplicate identity work; concurrent ordering stays in [Task 410][t410] |
| [Task 414][t414] | 4 | SQL UTXO amount integrity backstop | Refactor | Bound persisted SQL UTXO amounts | Accepted decision eliminates an optional SQL amount backstop; raw-output admission stays in [Task 426][t426] |
| [Task 416][t416] | 2B | wallet benchmark infrastructure | Benchmark | Provide minimal three-backend benchmark setup | Combined into [Task 471][t471]; shared setup is exercised by the first real measurement |
| [Task 418][t418] | 2B | wallet UtxoManager benchmarks | Benchmark | Measure ListUnspent performance | Combined into [Task 471][t471]; shared setup is exercised by the first real measurement |
| [Task 348][t348] | 2A | PsbtManager and AddressManager | Bug fix | Watch PSBT change before returning funding | [Task 474][t474] consolidates the missing authored-change watch calls |
| [Task 363][t363] | 2A | recovery horizon derivation | Bug fix | Bound aggregate recovery derivation work | Accepted decision retains caller-selected lookahead without a fixed aggregate cap; no replacement |
| [Task 366][t366] | 2A | Manager and SQL Store genesis result | Bug fix | Reconcile ambiguous SQL genesis commits | Accepted decision uses atomic genesis and conservative commit errors, not automatic readback; Tasks 329/358 retain their outcomes |
| [Task 371][t371] | 2A | SQL Store canonical chain state | Bug fix | Serialize canonical chain-frontier mutation | Accepted decision uses existing Wallet checkpoints, not a persisted Manager frontier; Tasks 370/372/423 retain coordination |
| [Task 408][t408] | 2A | TxCreator and AddressManager | Bug fix | Watch direct transaction change before return | [Task 474][t474] consolidates the missing authored-change watch calls |
| [Task 386][t386] | Hardening | Wallet-facing Vault and runtimeCache secret interfaces | Refactor | Remove raw Wallet secret seams | Accepted decision eliminates standalone cleanup; Tasks 382–385 remove their newly unused seams |

## Done · 61

**179 completed: 61 numbered tasks and 118 ad hoc PRs.** Each closing PR counts
once. Ad hoc entries have no task file or retrospective phase; they record
additional delivered work, not progress against a numbered phase gate.

### Numbered tasks

| Task | Phase | Component | Type | Outcome | Closing PR |
| --- | ---: | --- | --- | --- | --- |
| [Task 331][t331] | 1 | Manager lifecycle, public API cutover, and Store ownership | Bug fix | Make Manager the sole lifecycle authority | [#1328][pr1328] |
| [Task 332][t332] | 1 | Manager, AccountManager, and AddressManager | Gap fix | Define watch-only and script-import policy | [#1320][pr1320] |
| [Task 333][t333] | 1 | Manager and integration-test harness | Gap fix | Add PostgreSQL Manager and harness support | [#1303][pr1303] |
| [Task 334][t334] | 1 | SQL wallet architecture documentation | Design | Define aggregate Manager and shared-chain ownership | [#1323][pr1323] |
| [Task 337][t337] | 1 | Wallet and Signer public derivation | Bug fix | Restore SQL imported-XPub public derivation | [#1324][pr1324] |
| [Task 339][t339] | 1 | TxReader | Gap fix | Define public transaction visibility and status | [#1330][pr1330] |
| [Task 340][t340] | 1 | Manager and SQL Store | Bug fix | Bind SQL databases to schema and network identity | [#1345][pr1345] |
| [Task 341][t341] | 1 | wallet-owned account identity types | Gap fix | Define semantic account identity | [#1313][pr1313] |
| [Task 342][t342] | 1 | wallet configuration | Bug fix | Accept bounded recovery windows | [#1335][pr1335] |
| [Task 343][t343] | 1 | AccountManager root-derived creation and SQL AccountStore | Gap fix | Create next-or-exact root-derived accounts | [#1369][pr1369] |
| [Task 344][t344] | 1 | AddressManager receiving batches and SQL AddressStore | Gap fix | Allocate receiving-address batches atomically | [#1379][pr1379] |
| [Task 345][t345] | 1 | SQL account schema and AccountStore persistence | Gap fix | Persist SQL account NoChainSync state | [#1365][pr1365] |
| [Task 346][t346] | 2 | PsbtManager packet validation and transformation | Bug fix | Preserve PSBT metadata during funding | [#1359][pr1359] |
| [Task 347][t347] | 2A | PsbtManager existing-signature validation | Bug fix | Authorize existing PSBT signatures | [#1400][pr1400] |
| [Task 355][t355] | 2 | TxCreator and the `txsizes` module | Bug fix | Correct transaction size accounting | [#1338][pr1338] |
| [Task 357][t357] | 2 | `txauthor` and TxCreator | Bug fix | Validate transaction output and fee arithmetic | [#1348][pr1348] |
| [Task 359][t359] | 1 | Manager and SQL Store design | Design | Revalidate database identity design | [#1319][pr1319] |
| [Task 360][t360] | 1 | ADR 0013 and the modern kvdb adapter contract | Design | Document kvdb adapter identity | [#1308][pr1308] |
| [Task 377][t377] | 1 | AccountManager public result contract | Gap fix | Make AccountManager results wallet-owned | [#1329][pr1329] |
| [Task 378][t378] | 1 | AccountManager public error boundary | Bug fix | Translate AccountManager backend errors | [#1358][pr1358] |
| [Task 379][t379] | 1 | Manager Load public error boundary | Bug fix | Translate Manager load not-found errors | [#1331][pr1331] |
| [Task 380][t380] | 1 | AddressManager GetAddressInfo error boundary | Bug fix | Translate address lookup errors | [#1332][pr1332] |
| [Task 387][t387] | 1 | ADR 0012 and ADR 0013 account-creation contract | Design | Define account identity and chain-sync policy | [#1342][pr1342] |
| [Task 388][t388] | 1 | AccountStore account-creation admission | Bug fix | Reject colliding runtime account XPub identities | [#1394][pr1394] |
| [Task 390][t390] | 1 | AccountManager custom scope and SQL key-scope/account mutation | Gap fix | Create root-derived accounts in custom scopes | [#1371][pr1371] |
| [Task 392][t392] | 1 | AccountManager and SQL AccountStore create inputs | Gap fix | Carry NoChainSync through account creation | [#1366][pr1366] |
| [Task 393][t393] | 1 | AddressManager receiving APIs and AddressStore account lookup | Gap fix | Guard receiving APIs for NoChainSync accounts | [#1367][pr1367] |
| [Task 395][t395] | 1 | AddressManager key allocation and SQL branch-counter mutation | Gap fix | Add AllocateNextKey for durable key allocation | [#1386][pr1386] |
| [Task 396][t396] | 1 | AddressManager count-one semantics | Bug fix | Make NewAddress return the oldest unused address | [#1399][pr1399] |
| [Task 399][t399] | 1 | Syncer recovery initialization and account-scoped Store reads | Bug fix | Isolate persisted targets in zero-lookahead recovery | [#1372][pr1372] |
| [Task 400][t400] | 1 | ADR 0004 and ADR 0005 synchronization contract | Design | Separate live watches from historical recovery | [#1322][pr1322] |
| [Task 401][t401] | 1 | chain interface and Neutrino/btcd/bitcoind adapters | Gap fix | Add an idempotent live-tip-only chain watch | [#1344][pr1344] |
| [Task 402][t402] | 1 | Wallet synchronization and live-watch readiness | Gap fix | Establish live watches from persisted Wallet data | [#1380][pr1380] |
| [Task 404][t404] | 2 | TxCreator and PsbtManager input-source adapters | Bug fix | Validate Wallet transaction input amounts | [#1357][pr1357] |
| [Task 412][t412] | 1 | TxWriter transaction invalidation | Gap fix | Expose transaction invalidation on TxWriter | [#1388][pr1388] |
| [Task 415][t415] | 1 | UtxoManager integration tests | Test | Cover public UTXO contracts end to end | [#1307][pr1307] |
| [Task 421][t421] | 2 | WalletVault and SQL wallet-secret persistence | Bug fix | Reconcile SQL passphrase commits | [#1341][pr1341] |
| [Task 425][t425] | 2 | Wallet transaction-authoring wrapper boundary | Refactor | Isolate shared transaction authoring from wrapper watch and lease effects | [#1327][pr1327] |
| [Task 427][t427] | 1 | Controller credential integration tests | Test | Cover remaining Controller credential contracts | [#1349][pr1349] |
| [Task 429][t429] | 1 | AccountManager mutation integration tests | Test | Cover public account mutation contracts | [#1326][pr1326] |
| [Task 430][t430] | 1 | AccountManager query integration tests | Test | Cover public account query contracts | [#1389][pr1389] |
| [Task 431][t431] | 1 | AddressManager allocation and lookup integration tests | Test | Cover address allocation and lookup contracts | [#1407][pr1407] |
| [Task 433][t433] | 1 | Wallet sync-state publication and integration-test harness | Test | Enforce the wallet sync test boundary | [#1314][pr1314] |
| [Task 434][t434] | 1 | TxCreator integration tests | Test | Cover transaction creation contracts | [#1316][pr1316] |
| [Task 435][t435] | 1 | TxPublisher integration tests | Test | Cover transaction publication contracts | [#1334][pr1334] |
| [Task 436][t436] | 1 | TxReader integration tests | Test | Cover transaction read contracts | [#1346][pr1346] |
| [Task 437][t437] | 1 | TxWriter integration tests | Test | Cover transaction label contracts | [#1347][pr1347] |
| [Task 441][t441] | 1 | Signer derivation integration tests | Test | Cover public derivation and ECDH contracts | [#1374][pr1374] |
| [Task 442][t442] | 1 | Signer signature integration tests | Test | Cover public signing contracts | [#1384][pr1384] |
| [Task 443][t443] | 1 | UnsafeSigner integration tests | Test | Cover UnsafeSigner extraction contracts | [#1383][pr1383] |
| [Task 452][t452] | 1 | Syncer positive-lookahead account admission and enumeration | Gap fix | Exclude NoChainSync accounts from recovery | [#1368][pr1368] |
| [Task 453][t453] | 1 | Manager Wallet cache publication | Bug fix | Serialize Manager Wallet publication | [#1336][pr1336] |
| [Task 455][t455] | 1 | Wallet Controller synchronization information | Bug fix | Report exact wallet synchronization status | [#1363][pr1363] |
| [Task 456][t456] | 1 | ManagerConfig and managed Wallet construction | Refactor | Centralize managed Wallet runtime configuration | [#1353][pr1353] |
| [Task 458][t458] | 1 | Wallet lifecycle state and maintained public Wallet components | Bug fix | Drain Wallet calls during terminal stop | [#1354][pr1354] |
| [Task 459][t459] | 1 | Manager startup Wallet discovery and assembly | Gap fix | Assemble durable Wallets for Manager startup | [#1356][pr1356] |
| [Task 461][t461] | 1 | AddressManager batch integration tests | Test | Cover public address batch contracts | [#1393][pr1393] |
| [Task 465][t465] | 1 | AddressManager AllocateNextKey integration tests | Test | Verify durable key allocation results | [#1392][pr1392] |
| [Task 467][t467] | 1 | wallet public durable-mutation error identity | Gap fix | Define the indeterminate commit identity | [#1355][pr1355] |
| [Task 468][t468] | 1 | AccountManager NoChainSync creation admission | Gap fix | Activate NoChainSync account creation | [#1369][pr1369] |
| [Task 481][t481] | 1 | AccountManager and SQL account address schema | Bug fix | Preserve imported account address schema | [#1409][pr1409] |

### Ad hoc PRs

One PR per row, sorted by PR number. Author is recorded for attribution, not
eligibility. Historical task notes remain evidence, not extra counted rows.

Backfilled on 2026-09-12 from all 153 GitHub PRs marked merged with base
`sql-wallet`, without a date or author filter. The other 40 PRs are already
counted in the numbered table above. This includes work predating the current
numbered roadmap; old archived task references add no separate completion.

| PR | Outcome | Author | Merged (UTC) |
| --- | --- | --- | --- |
| [#1065](https://github.com/btcsuite/btcwallet/pull/1065) | multi: add sqlc support | GustavoStingelin | 2025-10-30 |
| [#1101](https://github.com/btcsuite/btcwallet/pull/1101) | Add the SQL blocks schema | moawnallah | 2025-11-11 |
| [#1111](https://github.com/btcsuite/btcwallet/pull/1111) | wallet: introduce db layer interfaces | yyforyongyu | 2025-11-24 |
| [#1115](https://github.com/btcsuite/btcwallet/pull/1115) | wallet+Makefile: move sql db to wallet/internal | GustavoStingelin | 2025-11-12 |
| [#1121](https://github.com/btcsuite/btcwallet/pull/1121) | wallet: integrate `sqlfluff` metadev helpers | moawnallah | 2025-12-15 |
| [#1125](https://github.com/btcsuite/btcwallet/pull/1125) | wallet: implement WalletStore interface across PG and SQLite | GustavoStingelin | 2025-12-11 |
| [#1131](https://github.com/btcsuite/btcwallet/pull/1131) | wallet: refactor WalletStore interface and add common errors | GustavoStingelin | 2025-11-28 |
| [#1132](https://github.com/btcsuite/btcwallet/pull/1132) | wallet: add store db constructors and safe-casting | GustavoStingelin | 2025-11-28 |
| [#1134](https://github.com/btcsuite/btcwallet/pull/1134) | Implement the SQL blocks store | moawnallah | 2025-12-22 |
| [#1136](https://github.com/btcsuite/btcwallet/pull/1136) | wallet: add sqldb itest | GustavoStingelin | 2025-12-10 |
| [#1138](https://github.com/btcsuite/btcwallet/pull/1138) | wallet: Implement address type store support | GustavoStingelin | 2025-12-17 |
| [#1139](https://github.com/btcsuite/btcwallet/pull/1139) | wallet: rename wallet sync table birthday to birthday_timestamp | GustavoStingelin | 2025-12-16 |
| [#1141](https://github.com/btcsuite/btcwallet/pull/1141) | wallet: fix pg itest "too many clients" | GustavoStingelin | 2025-12-17 |
| [#1142](https://github.com/btcsuite/btcwallet/pull/1142) | wallet: add key scope SQL migrations and queries | GustavoStingelin | 2025-12-22 |
| [#1143](https://github.com/btcsuite/btcwallet/pull/1143) | wallet: add P2A and P2PK address types | GustavoStingelin | 2025-12-18 |
| [#1147](https://github.com/btcsuite/btcwallet/pull/1147) | wallet: add AccountStore implementation | GustavoStingelin | 2026-01-20 |
| [#1158](https://github.com/btcsuite/btcwallet/pull/1158) | multi: add coveralls for database integration tests | GustavoStingelin | 2026-01-22 |
| [#1162](https://github.com/btcsuite/btcwallet/pull/1162) | wallet: implement AddressStore | GustavoStingelin | 2026-02-11 |
| [#1165](https://github.com/btcsuite/btcwallet/pull/1165) | Add design docs for transaction table schema | yyforyongyu | 2026-02-12 |
| [#1166](https://github.com/btcsuite/btcwallet/pull/1166) | docs: add keyvault ADR | GustavoStingelin | 2026-03-06 |
| [#1168](https://github.com/btcsuite/btcwallet/pull/1168) | kvdb: start implementing `Store` interface | yyforyongyu | 2026-02-24 |
| [#1169](https://github.com/btcsuite/btcwallet/pull/1169) | multi: move tools config files back to root | GustavoStingelin | 2026-02-09 |
| [#1170](https://github.com/btcsuite/btcwallet/pull/1170) | docs+README: add Single-Passphrase Encryption Model ADR | GustavoStingelin | 2026-03-03 |
| [#1171](https://github.com/btcsuite/btcwallet/pull/1171) | docs: add XChaCha20-Poly1305 Encryption ADR | GustavoStingelin | 2026-03-02 |
| [#1172](https://github.com/btcsuite/btcwallet/pull/1172) | itest: implement `lnd` style itest framework | yyforyongyu | 2026-02-12 |
| [#1173](https://github.com/btcsuite/btcwallet/pull/1173) | Finish itest framework | yyforyongyu | 2026-02-13 |
| [#1174](https://github.com/btcsuite/btcwallet/pull/1174) | chain: fix flaky `TestPrunedBlockDispatcherQuerySameBlock` | yyforyongyu | 2026-03-30 |
| [#1182](https://github.com/btcsuite/btcwallet/pull/1182) | wallet: add real db constructors out of itest | GustavoStingelin | 2026-03-05 |
| [#1185](https://github.com/btcsuite/btcwallet/pull/1185) | wallet: introduce SQL schema and queries for txs and UTXOs | yyforyongyu | 2026-03-23 |
| [#1186](https://github.com/btcsuite/btcwallet/pull/1186) | wallet: add SQL tx stores | yyforyongyu | 2026-04-07 |
| [#1187](https://github.com/btcsuite/btcwallet/pull/1187) | wallet: add SQL tx replacement and invalidation flows | yyforyongyu | 2026-04-08 |
| [#1188](https://github.com/btcsuite/btcwallet/pull/1188) | Finalize wallet manager's `WalletID` | yyforyongyu | 2026-03-12 |
| [#1189](https://github.com/btcsuite/btcwallet/pull/1189) | wallet: implement store pagination | GustavoStingelin | 2026-04-08 |
| [#1190](https://github.com/btcsuite/btcwallet/pull/1190) | Fix `make tidy-module` | yyforyongyu | 2026-03-16 |
| [#1192](https://github.com/btcsuite/btcwallet/pull/1192) | wallet: fix pg itest connection exhaustion under parallelism | GustavoStingelin | 2026-03-27 |
| [#1193](https://github.com/btcsuite/btcwallet/pull/1193) | wallet: add SQL UTXO store | yyforyongyu | 2026-04-07 |
| [#1195](https://github.com/btcsuite/btcwallet/pull/1195) | bwtest: wait for miner p2p readiness | yyforyongyu | 2026-03-27 |
| [#1196](https://github.com/btcsuite/btcwallet/pull/1196) | wallet: add tx and utxo store coverage | yyforyongyu | 2026-04-10 |
| [#1197](https://github.com/btcsuite/btcwallet/pull/1197) | Update `TxStore` interface and add missing queries | yyforyongyu | 2026-03-31 |
| [#1198](https://github.com/btcsuite/btcwallet/pull/1198) | Clarify integration-test names | GustavoStingelin | 2026-03-30 |
| [#1199](https://github.com/btcsuite/btcwallet/pull/1199) | wallet: simplify postgres itest headroom and add repo guide | yyforyongyu | 2026-03-30 |
| [#1200](https://github.com/btcsuite/btcwallet/pull/1200) | wallet+db: handle SQL backend errors | yyforyongyu | 2026-04-15 |
| [#1201](https://github.com/btcsuite/btcwallet/pull/1201) | wallet: migrate LabelTx to db.Store | yyforyongyu | 2026-04-30 |
| [#1202](https://github.com/btcsuite/btcwallet/pull/1202) | wallet: migrate tx reader to db.Store | yyforyongyu | 2026-05-11 |
| [#1204](https://github.com/btcsuite/btcwallet/pull/1204) | Prep address manager for store migration | yyforyongyu | 2026-04-23 |
| [#1206](https://github.com/btcsuite/btcwallet/pull/1206) | wallet: improve concurrent itests | GustavoStingelin | 2026-04-07 |
| [#1208](https://github.com/btcsuite/btcwallet/pull/1208) | wallet: simplify pagination api | yyforyongyu | 2026-04-14 |
| [#1209](https://github.com/btcsuite/btcwallet/pull/1209) | db: split backend packages and SQL assets | yyforyongyu | 2026-04-13 |
| [#1210](https://github.com/btcsuite/btcwallet/pull/1210) | wallet: fix missing drop statements | GustavoStingelin | 2026-04-14 |
| [#1211](https://github.com/btcsuite/btcwallet/pull/1211) | wallet: move legacy helpers to deprecated | yyforyongyu | 2026-04-28 |
| [#1212](https://github.com/btcsuite/btcwallet/pull/1212) | db: apply runtime read/write helpers | yyforyongyu | 2026-04-21 |
| [#1214](https://github.com/btcsuite/btcwallet/pull/1214) | wallet: migrate address manager to db.Store | yyforyongyu | 2026-05-28 |
| [#1216](https://github.com/btcsuite/btcwallet/pull/1216) | wallet: cleanup import aliases | GustavoStingelin | 2026-04-15 |
| [#1217](https://github.com/btcsuite/btcwallet/pull/1217) | bwtest: pin bitcoind to v1 transport | yyforyongyu | 2026-04-16 |
| [#1219](https://github.com/btcsuite/btcwallet/pull/1219) | wallet: make address scripts unique per wallet | GustavoStingelin | 2026-04-21 |
| [#1220](https://github.com/btcsuite/btcwallet/pull/1220) | wallet: guard nil derived address callbacks | GustavoStingelin | 2026-04-22 |
| [#1221](https://github.com/btcsuite/btcwallet/pull/1221) | wallet: disambiguates watch-only | GustavoStingelin | 2026-05-14 |
| [#1222](https://github.com/btcsuite/btcwallet/pull/1222) | wallet: enforce watch-only invariants in sql | GustavoStingelin | 2026-05-15 |
| [#1223](https://github.com/btcsuite/btcwallet/pull/1223) | db: add SQL tx detail reads | yyforyongyu | 2026-05-11 |
| [#1224](https://github.com/btcsuite/btcwallet/pull/1224) | wallet: store public key in plaintext | GustavoStingelin | 2026-05-15 |
| [#1225](https://github.com/btcsuite/btcwallet/pull/1225) | wallet: replace advisory lock with persisted account counter | GustavoStingelin | 2026-05-15 |
| [#1227](https://github.com/btcsuite/btcwallet/pull/1227) | build: lint tagged itest variants | yyforyongyu | 2026-05-11 |
| [#1228](https://github.com/btcsuite/btcwallet/pull/1228) | db: prepare kvdb store routing | yyforyongyu | 2026-05-14 |
| [#1229](https://github.com/btcsuite/btcwallet/pull/1229) | wallet: migrate wallet metadata to db.Store | yyforyongyu | 2026-06-02 |
| [#1230](https://github.com/btcsuite/btcwallet/pull/1230) | Route TxCreator and TxPublisher through Store | yyforyongyu | 2026-06-10 |
| [#1231](https://github.com/btcsuite/btcwallet/pull/1231) | Prepare AccountManager Store routing | yyforyongyu | 2026-05-18 |
| [#1232](https://github.com/btcsuite/btcwallet/pull/1232) | db: populate AccountInfo balance with SkipBalance opt-out | yyforyongyu | 2026-05-20 |
| [#1233](https://github.com/btcsuite/btcwallet/pull/1233) | multi: use testify mock for derived account ops tests and docs | GustavoStingelin | 2026-05-18 |
| [#1237](https://github.com/btcsuite/btcwallet/pull/1237) | Prepare AddressManager Store routing | yyforyongyu | 2026-05-27 |
| [#1238](https://github.com/btcsuite/btcwallet/pull/1238) | wallet: migrate UTXO store + lease ops to db.Store | yyforyongyu | 2026-06-10 |
| [#1239](https://github.com/btcsuite/btcwallet/pull/1239) | wallet: route account manager through Store + finish kvdb writes | yyforyongyu | 2026-05-25 |
| [#1240](https://github.com/btcsuite/btcwallet/pull/1240) | Prepare AccountStore identity, cache, and filters | yyforyongyu | 2026-05-22 |
| [#1242](https://github.com/btcsuite/btcwallet/pull/1242) | wallet: add wallet secrets read | GustavoStingelin | 2026-05-27 |
| [#1243](https://github.com/btcsuite/btcwallet/pull/1243) | wallet: migrate address-store read path + signer derivation | yyforyongyu | 2026-05-28 |
| [#1245](https://github.com/btcsuite/btcwallet/pull/1245) | wallet: consolidate testify mocks into bwtest/mock | yyforyongyu | 2026-05-28 |
| [#1246](https://github.com/btcsuite/btcwallet/pull/1246) | wallet: enforce wallet-level watch-only invariant (ADR 0012) | yyforyongyu | 2026-06-10 |
| [#1247](https://github.com/btcsuite/btcwallet/pull/1247) | Enrich UtxoInfo and scoped UTXO filtering | yyforyongyu | 2026-06-10 |
| [#1248](https://github.com/btcsuite/btcwallet/pull/1248) | wallet: route tx publisher + tx-store writes through Store | yyforyongyu | 2026-06-15 |
| [#1252](https://github.com/btcsuite/btcwallet/pull/1252) | wallet+AGENTS.md: add agent guidance for wallet subsystems | GustavoStingelin | 2026-06-15 |
| [#1253](https://github.com/btcsuite/btcwallet/pull/1253) | wallet: add account ops workflow | GustavoStingelin | 2026-06-08 |
| [#1256](https://github.com/btcsuite/btcwallet/pull/1256) | db: implement the ApplyScanBatch scan engine | yyforyongyu | 2026-07-09 |
| [#1257](https://github.com/btcsuite/btcwallet/pull/1257) | db: cover ApplyScanBatch conformance and RollbackToBlock atomicity | yyforyongyu | 2026-07-10 |
| [#1258](https://github.com/btcsuite/btcwallet/pull/1258) | Route startup, sync-tip, and rewind through Store | yyforyongyu | 2026-07-12 |
| [#1260](https://github.com/btcsuite/btcwallet/pull/1260) | wallet: split account methods in multiple files | GustavoStingelin | 2026-06-04 |
| [#1261](https://github.com/btcsuite/btcwallet/pull/1261) | wallet: readd create max account number itest | GustavoStingelin | 2026-06-05 |
| [#1262](https://github.com/btcsuite/btcwallet/pull/1262) | db: add synced-block, watched-output, and tx-batch Store methods | yyforyongyu | 2026-07-02 |
| [#1263](https://github.com/btcsuite/btcwallet/pull/1263) | Add scan-batch scaffolding and rollback support | yyforyongyu | 2026-07-02 |
| [#1265](https://github.com/btcsuite/btcwallet/pull/1265) | multi: add keyvault lifecycle boundary | GustavoStingelin | 2026-06-16 |
| [#1267](https://github.com/btcsuite/btcwallet/pull/1267) | wallet+db: fix SQL-wallet review issues | yyforyongyu | 2026-06-18 |
| [#1268](https://github.com/btcsuite/btcwallet/pull/1268) | wallet+db: normalize account address identity | yyforyongyu | 2026-07-02 |
| [#1269](https://github.com/btcsuite/btcwallet/pull/1269) | Add scan-horizon scaffolding and rollback eviction | yyforyongyu | 2026-07-06 |
| [#1270](https://github.com/btcsuite/btcwallet/pull/1270) | Route chain-scan writes and reads through Store | yyforyongyu | 2026-07-12 |
| [#1271](https://github.com/btcsuite/btcwallet/pull/1271) | Resolve rescan identity and require Store in Syncer | yyforyongyu | 2026-07-12 |
| [#1272](https://github.com/btcsuite/btcwallet/pull/1272) | Route account secrets through Store and Vault | yyforyongyu | 2026-08-04 |
| [#1273](https://github.com/btcsuite/btcwallet/pull/1273) | wallet: impl keyvault lifecycle | GustavoStingelin | 2026-07-06 |
| [#1277](https://github.com/btcsuite/btcwallet/pull/1277) | db: align SQL address store with the db-ops interface style | Abdulkbk | 2026-07-07 |
| [#1291](https://github.com/btcsuite/btcwallet/pull/1291) | Finalize SQL Implementation | yyforyongyu | 2026-08-04 |
| [#1292](https://github.com/btcsuite/btcwallet/pull/1292) | Start SQL Integration Tests | yyforyongyu | 2026-08-05 |
| [#1300](https://github.com/btcsuite/btcwallet/pull/1300) | wallet+itest+bwtest: add Manager and Controller itests | GustavoStingelin | 2026-08-06 |
| [#1304](https://github.com/btcsuite/btcwallet/pull/1304) | multi: match nested CI base branches | yyforyongyu | 2026-08-06 |
| [#1306](https://github.com/btcsuite/btcwallet/pull/1306) | Make WatchOnly authoritative during creation | GustavoStingelin | 2026-08-12 |
| [#1309](https://github.com/btcsuite/btcwallet/pull/1309) | itest: fix case filtering | yyforyongyu | 2026-08-11 |
| [#1310](https://github.com/btcsuite/btcwallet/pull/1310) | wallet: silence local goconst lint errors | GustavoStingelin | 2026-08-11 |
| [#1315](https://github.com/btcsuite/btcwallet/pull/1315) | chain: synchronize bitcoind event tests | yyforyongyu | 2026-08-13 |
| [#1325](https://github.com/btcsuite/btcwallet/pull/1325) | itest: document integration test standards | yyforyongyu | 2026-08-19 |
| [#1333](https://github.com/btcsuite/btcwallet/pull/1333) | docs: add ADR authoring template | yyforyongyu | 2026-08-20 |
| [#1361](https://github.com/btcsuite/btcwallet/pull/1361) | wallet: publish a transaction label limit | Abdulkbk | 2026-09-07 |
| [#1362](https://github.com/btcsuite/btcwallet/pull/1362) | wallet+kvdb: clear a transaction label with an empty one | Abdulkbk | 2026-09-09 |
| [#1364](https://github.com/btcsuite/btcwallet/pull/1364) | Reject empty SQL watch-only passphrases | GustavoStingelin | 2026-09-05 |
| [#1373](https://github.com/btcsuite/btcwallet/pull/1373) | wallet: route component requests through admission | yyforyongyu | 2026-09-10 |
| [#1375](https://github.com/btcsuite/btcwallet/pull/1375) | wallet+db: simplify account error boundary | yyforyongyu | 2026-09-12 |
| [#1376](https://github.com/btcsuite/btcwallet/pull/1376) | wallet: reject nil Signer arguments before admission | Abdulkbk | 2026-09-12 |
| [#1377](https://github.com/btcsuite/btcwallet/pull/1377) | bwtest: stop neutrino before closing database | yyforyongyu | 2026-09-11 |
| [#1385](https://github.com/btcsuite/btcwallet/pull/1385) | wallet: classify initial account input errors | yyforyongyu | 2026-09-18 |
| [#1387](https://github.com/btcsuite/btcwallet/pull/1387) | wallet: fix sync readiness and isolate targeted rescan failures | GustavoStingelin | 2026-09-18 |
| [#1395](https://github.com/btcsuite/btcwallet/pull/1395) | wallet: harden psbt funding admission | Abdulkbk | 2026-09-24 |
| [#1396](https://github.com/btcsuite/btcwallet/pull/1396) | multi: delete unmined tx branches instead of retaining them | Abdulkbk | 2026-09-24 |
| [#1402](https://github.com/btcsuite/btcwallet/pull/1402) | wallet+db: make oldest-unused address atomic sql native | GustavoStingelin | 2026-10-15 |

[t329]: tasks/329-atomic-wallet-account-creation.md
[t330]: tasks/330-duplicate-wallet-create.md
[t331]: archive/tasks/331-manager-wallet-lifecycle.md
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
[t343]: archive/tasks/343-exact-account-creation.md
[t344]: tasks/344-bulk-address-allocation.md
[t345]: archive/tasks/345-durable-watch-registration.md
[t346]: archive/tasks/346-psbt-funding-metadata.md
[t347]: tasks/347-psbt-signing-authorization.md
[t348]: archive/tasks/348-watched-change-delivery.md
[t349]: archive/tasks/349-transaction-invalidation.md
[t350]: tasks/350-native-pgx-postgres.md
[t351]: archive/tasks/351-sql-utxo-integrity.md
[t352]: archive/tasks/352-kvdb-combined-passphrase-rotation.md
[t353]: archive/tasks/353-retire-modern-kvdb-backend.md
[t354]: tasks/354-vault-owned-secret-resolution.md
[t355]: archive/tasks/355-transaction-size-accounting.md
[t356]: tasks/356-defer-change-script-allocation.md
[t357]: archive/tasks/357-transaction-amount-validation.md
[t358]: tasks/358-contextual-manager-operations.md
[t359]: archive/tasks/359-revalidate-database-identity-design.md
[t360]: archive/tasks/360-document-kvdb-adapter-identity.md
[t361]: archive/tasks/361-kvdb-imported-xpub-derivation.md
[t362]: archive/tasks/362-kvdb-definitive-passphrase-result.md
[t363]: archive/tasks/363-recovery-scan-work-bound.md
[t364]: archive/tasks/364-durable-watch-registration.md
[t365]: tasks/365-validate-wallet-genesis-keys.md
[t366]: archive/tasks/366-reconcile-sql-genesis-commit.md
[t367]: tasks/367-bound-sql-genesis-secrets.md
[t368]: archive/tasks/368-wallet-startup-shutdown.md
[t369]: tasks/369-repeated-wallet-unlock.md
[t370]: tasks/370-chain-source-delivery.md
[t371]: archive/tasks/371-canonical-chain-frontier.md
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
[t386]: archive/tasks/386-remove-raw-wallet-secret-seams.md
[t387]: archive/tasks/387-account-creation-identity-adr.md
[t388]: tasks/388-runtime-account-xpub-collisions.md
[t389]: tasks/389-runtime-account-commit-reconciliation.md
[t390]: archive/tasks/390-custom-scope-account-creation.md
[t391]: tasks/391-path-bound-watch-only-xpub-accounts.md
[t392]: archive/tasks/392-discovery-mode-account-creation.md
[t393]: archive/tasks/393-key-only-receiving-rejection.md
[t394]: archive/tasks/394-key-only-transaction-ownership.md
[t395]: tasks/395-key-only-locator-allocation.md
[t396]: tasks/396-oldest-unused-address.md
[t397]: tasks/397-semantic-recovery-targets-adr.md
[t398]: tasks/398-semantic-account-recovery-targets.md
[t399]: archive/tasks/399-zero-lookahead-persisted-targets.md
[t400]: archive/tasks/400-live-watch-semantics-adr.md
[t401]: archive/tasks/401-live-tip-chain-watch.md
[t402]: tasks/402-replay-live-watch-obligations.md
[t403]: tasks/403-raw-import-watch-obligations.md
[t404]: archive/tasks/404-transaction-input-amounts.md
[t405]: tasks/405-psbt-funding-lease-admission.md
[t406]: tasks/406-psbt-funding-lease-release.md
[t407]: tasks/407-psbt-finalization-authorization.md
[t408]: archive/tasks/408-direct-transaction-change-watch.md
[t409]: archive/tasks/409-transaction-input-graph.md
[t410]: tasks/410-transaction-graph-admission.md
[t411]: tasks/411-terminal-parent-rejection.md
[t412]: archive/tasks/412-transaction-invalidation.md
[t413]: archive/tasks/413-utxo-spend-identity.md
[t414]: archive/tasks/414-utxo-amount-bounds.md
[t415]: archive/tasks/415-utxo-public-integration-tests.md
[t416]: archive/tasks/416-database-benchmark-fixtures.md
[t417]: tasks/417-start-to-synced-benchmark.md
[t418]: archive/tasks/418-list-unspent-benchmark.md
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
[t430]: archive/tasks/430-account-query-itests.md
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
[t441]: archive/tasks/441-signer-derivation-ecdh-itests.md
[t442]: archive/tasks/442-signer-signing-itests.md
[t443]: archive/tasks/443-unsafe-signer-itests.md
[t444]: tasks/444-wallet-receive-reload-scenario.md
[t445]: tasks/445-account-signing-scenario.md
[t446]: tasks/446-transaction-spend-publish-scenario.md
[t447]: tasks/447-psbt-workflow-scenario.md
[t448]: tasks/448-sync-rollback-scenario.md
[t449]: tasks/449-multi-wallet-sql-isolation-scenario.md
[t450]: tasks/450-bitcoind-compatibility-scenario.md
[t451]: tasks/451-chain-version-itest-matrix.md
[t452]: archive/tasks/452-key-only-recovery-derivation.md
[t453]: archive/tasks/453-serialize-manager-wallet-publication.md
[t454]: tasks/454-postgres-transaction-isolation.md
[t455]: archive/tasks/455-exact-wallet-sync-status.md
[t456]: archive/tasks/456-manager-runtime-configuration.md
[t457]: tasks/457-process-local-manager-lease.md
[t458]: archive/tasks/458-manager-operation-admission.md
[t459]: archive/tasks/459-manager-startup-wallet-assembly.md
[t460]: tasks/460-sequential-account-invalid-children.md
[t461]: tasks/461-address-batch-itests.md
[t462]: tasks/462-account-mutation-error-itests.md
[t463]: tasks/463-path-bound-utxo-identity-itests.md
[t464]: tasks/464-no-chain-sync-account-query-itests.md
[t465]: tasks/465-new-address-result-itests.md
[t466]: tasks/466-oldest-unused-address-itests.md
[t467]: archive/tasks/467-indeterminate-commit-identity.md
[t468]: archive/tasks/468-activate-no-chain-sync-account-creation.md
[t469]: tasks/469-sign-psbt-with-effective-keys.md
[t470]: tasks/470-lnd-wallet-port.md
[t471]: tasks/471-three-backend-list-unspent-benchmark.md
[t472]: tasks/472-multi-wallet-rescan-isolation-scenario.md
[t473]: tasks/473-extract-legacy-address-script-state.md

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
[pr1356]: https://github.com/btcsuite/btcwallet/pull/1356
[pr1363]: https://github.com/btcsuite/btcwallet/pull/1363
[pr1365]: https://github.com/btcsuite/btcwallet/pull/1365
[pr1366]: https://github.com/btcsuite/btcwallet/pull/1366
[pr1367]: https://github.com/btcsuite/btcwallet/pull/1367
[pr1368]: https://github.com/btcsuite/btcwallet/pull/1368
[pr1328]: https://github.com/btcsuite/btcwallet/pull/1328
[pr1369]: https://github.com/btcsuite/btcwallet/pull/1369
[pr1359]: https://github.com/btcsuite/btcwallet/pull/1359
[pr1371]: https://github.com/btcsuite/btcwallet/pull/1371
[pr1372]: https://github.com/btcsuite/btcwallet/pull/1372
[pr1388]: https://github.com/btcsuite/btcwallet/pull/1388
[pr1374]: https://github.com/btcsuite/btcwallet/pull/1374

[t474]: tasks/474-authored-change-watch.md
[pr1383]: https://github.com/btcsuite/btcwallet/pull/1383

[t475]: tasks/475-wallet-transaction-subscription.md
[t476]: tasks/476-read-legacy-sql-walletdb-source.md
[t477]: tasks/477-import-legacy-snapshot-postgres.md
[t478]: tasks/478-retain-spend-confirmation-leases.md
[t479]: tasks/479-mix-imported-xpub-accounts.md
[t480]: tasks/480-own-raw-imports-in-one-wallet.md
[t481]: tasks/481-persist-imported-account-address-schema.md
[t482]: tasks/482-fund-imported-inputs-without-origin.md
[t483]: tasks/483-preserve-labels-on-chain-replay.md
[t484]: tasks/484-align-authoring-confirmation-height.md
[t485]: tasks/485-convert-legacy-sql-wallets-through-manager.md

[pr1384]: https://github.com/btcsuite/btcwallet/pull/1384
[pr1389]: https://github.com/btcsuite/btcwallet/pull/1389

[t486]: tasks/486-freeze-native-sql-schema.md
[t487]: tasks/487-compile-lnd-with-managed-wallet.md
