# btcwallet SQL wallet roadmap

This private repository coordinates the remaining SQL wallet work. This file
defines the canonical priority and dependency order, [ACTIVE.md](ACTIVE.md)
records current ownership, and [tasks/](tasks/) contains the executable task
specifications.

All earlier task files are preserved under [archive/](archive/) as evidence,
not instruction. This sequence begins after archived [Task 328][t328]. The
next free task number is 359.

## Canonical backlog

| Task | Scope | [Assignee](ACTIVE.md) |
| --- | --- | --- |
| [329][t329] | atomic creation | — |
| [330][t330] | duplicate-create semantics | — |
| [331][t331] | lifecycle concurrency | — |
| [332][t332] | watch-only and script-import policy | [YY](ACTIVE.md#claimed-tasks) |
| [333][t333] | PostgreSQL Manager and harness | [YY](ACTIVE.md#claimed-tasks) |
| [334][t334] | shared canonical chain and rollback | — |
| [335][t335] | atomic kvdb-to-SQL conversion | — |
| [336][t336] | redundant Wallet tests | — |
| [337][t337] | imported-xpub public derivation | — |
| [338][t338] | definitive passphrase-change result | — |
| [339][t339] | public transaction visibility/status | — |
| [340][t340] | SQL schema/network identity | — |
| [341][t341] | wallet-owned public role types/errors | — |
| [342][t342] | recovery-window bounds | — |
| [343][t343] | exact/custom/path-bound accounts | — |
| [344][t344] | atomic receiving/key allocation | — |
| [345][t345] | discovery/watch registration | — |
| [346][t346] | metadata-preserving PSBT funding | — |
| [347][t347] | PSBT signing authorization | — |
| [348][t348] | watched change before funded results | — |
| [349][t349] | invalidation/input graph | — |
| [350][t350] | native pgx PostgreSQL cleanup | — |
| [351][t351] | declarative SQL UTXO integrity | — |
| [352][t352] | atomic kvdb rotation | — |
| [353][t353] | retire modern kvdb support | — |
| [354][t354] | Vault-owned secret resolution | — |
| [355][t355] | transaction size accounting | — |
| [356][t356] | viable change allocation | — |
| [357][t357] | checked monetary aggregates | — |
| [358][t358] | cancellable Manager operations | — |

## Execution rule and recommended order

Each branch and PR addresses exactly one task file. Do not combine two tasks
because they are in the same dependency group or touch nearby code. A dotted
milestone such as [`337.sql-selector`][t337] is one narrowly scoped PR belonging only
to its named task; completing that milestone does not authorize the rest of the
task in the same PR.

"Parallel" below means different developers may take different ready items.
One developer still works on one item at a time, and every item remains its own
reviewable PR.

### Phase 1 — enable PostgreSQL itests

1. **[Task 333][t333] — PostgreSQL Manager and harness.** Do this first. It is
   the only task needed to make the existing and future component itests
   selectable with `db=postgres`. [Task 350][t350]'s native-pgx cleanup is not
   a prerequisite.

Do not mix a bug exposed by the PostgreSQL run into [Task 333][t333] unless the
bug is in its backend-selection or harness plumbing. Record other defects
under their own task and keep the PostgreSQL-enabling PR focused.

### Phase 2 — make creation atomic

After [Task 333][t333], finish the prerequisites below. Items on the same line
can run in parallel as separate PRs:

1. **Parallel policy and design:** [Task 332][t332] (watch-only creation
   policy), [Task 340][t340]'s [`340.identity-revalidation`][t340] gate, and
   [Task 341][t341]'s [`341.adr0013-adapter-identity`][t341] gate.
2. **Parallel foundations:** [Task 340][t340] after its revalidation is
   accepted, and [Task 341][t341] after its ADR 0013 adapter-identity amendment
   is accepted. Each implementation remains separate from its design review
   and other tasks.
3. **Parallel milestones:** [`337.sql-selector`][t337] after [Task 341][t341], and
   [`345.sql-discovery`][t345] after [Task 340][t340].
4. **[Task 329][t329] — atomic wallet and account creation** after
   [Task 332][t332] and both milestones.

Only the two named milestones are prerequisites here. Do not pull all of
[Task 337][t337] or [Task 345][t345] into the atomic-creation PR.

### Phase 3 — remaining independent foundations

Once [Task 329][t329] is complete, the following foundation lanes may be
developed in parallel after satisfying any gate named on the item, each in its
own PR:

- [Task 331][t331] — Manager wallet lifecycle concurrency.
- [Task 339][t339] — public transaction visibility and status.
- [Task 350][t350] — native pgx PostgreSQL cleanup.
- [Task 352][t352] — atomic legacy-kvdb passphrase rotation.
- [Task 355][t355] — transaction size accounting.
- [Task 357][t357] — checked transaction amount arithmetic.
- [Task 335][t335] — kvdb-to-SQL migration, because its Phase 2 prerequisites are now
  complete.
- The remaining work in [Task 337][t337] — imported-XPub public derivation.
- The remaining work in [Task 345][t345] after accepted ADR 0004 and ADR 0005
  live-watch amendments — durable watch registration.

### Phase 4 — first dependent group

These may run in parallel once their individual predecessors are complete:

- [Task 330][t330] after [Task 331][t331] and [Task 333][t333] —
  duplicate-create semantics.
- [Task 334][t334] after [Task 333][t333] and [Task 340][t340] plus an accepted
  ADR 0002 replacement — shared canonical chain and rollback.
- [`338.sql-definitive`][t338] after [Task 331][t331] — definitive SQL
  passphrase results.
- [`342.config-bound`][t342] after [Task 341][t341] — recovery-window
  configuration bound.
- [Task 346][t346] after [Task 357][t357] — metadata-preserving PSBT funding.
- [Task 354][t354] after an accepted ADR 0010 amendment and either
  [Task 352][t352] while modern kvdb remains or [Task 353][t353] removing that
  prerequisite — Vault-owned secret resolution.
- [Task 356][t356] after [Task 355][t355] and [Task 357][t357] — defer change
  allocation until viable.
- [Task 358][t358] after [Task 329][t329] and [Task 331][t331] — cancellable
  Manager operations.
- [Task 353][t353] only after [Task 335][t335], verified production conversion
  evidence, and an explicit consumer-cutover decision — retire modern kvdb
  support.

### Phase 5 — second dependent group

These may run in parallel once their individual predecessors are complete:

- [Task 338][t338] after [`338.sql-definitive`][t338] and either
  [Task 352][t352] while modern kvdb remains or [Task 353][t353] removes its
  kvdb clause — complete definitive passphrase results across the surviving
  backends.
- [Task 342][t342] after [Task 337][t337] and [Task 345][t345],
  [`342.config-bound`][t342], and an accepted ADR 0004 semantic-account amendment —
  recovery-window validation and scan behavior.
- [Task 343][t343] after [Task 332][t332], [Task 337][t337],
  [Task 340][t340], [Task 345][t345], and [Task 354][t354] plus accepted ADR
  0012 and ADR 0013 account amendments — exact/custom/path-bound account
  creation.
- [Task 344][t344] after [`337.sql-selector`][t337] and [Task 345][t345] —
  atomic bulk receiving/key allocation.
- [Task 348][t348] after [Task 345][t345], [Task 346][t346],
  [Task 355][t355], and [Task 356][t356] — watched change delivery.
- [Task 349][t349] after [Task 334][t334] and [Task 339][t339] plus an accepted
  ADR 0006 invalidation amendment — transaction invalidation and input graph.

### Phase 6 — final dependents and cleanup

- [Task 347][t347] after [Task 343][t343] and [Task 346][t346] — PSBT signing
  authorization.
- [Task 351][t351] after [Task 349][t349] plus an accepted ADR 0006
  UTXO-integrity amendment — declarative SQL UTXO integrity.
- [Task 336][t336] after [Task 338][t338] and [Task 354][t354] — remove
  redundant Wallet tests only after the replacement boundaries are stable.

The numbered phases express the recommended priority, not a requirement to
serialize unrelated work across developers. Within a phase, follow the explicit
predecessors and keep every PR limited to one task.

## Dependencies

Only the following ordering constraints are currently established:

- [Task 331][t331] and [Task 333][t333] → [Task 330][t330].
- Accepted [`340.identity-revalidation`][t340] → [Task 340][t340].
- Accepted [`341.adr0013-adapter-identity`][t341] → [Task 341][t341].
- [Task 340][t340] → [`345.sql-discovery`][t345].
- [Task 340][t340], [`345.sql-discovery`][t345], and accepted ADR 0004/0005 live-watch
  amendments → [Task 345][t345].
- [Task 341][t341] → [`337.sql-selector`][t337] → [Task 337][t337].
- [Task 332][t332], [`337.sql-selector`][t337], and [`345.sql-discovery`][t345] →
  [Task 329][t329].
- [Task 329][t329] and [Task 331][t331] → [Task 358][t358].
- [Task 332][t332], [Task 340][t340], [`337.sql-selector`][t337], and
  [`345.sql-discovery`][t345] → [Task 335][t335].
- [Task 335][t335], production conversion evidence, and a consumer cutover
  decision → [Task 353][t353].
- [Task 333][t333], [Task 340][t340], and an accepted ADR 0002 replacement →
  [Task 334][t334].
- [Task 334][t334], [Task 339][t339], and an accepted ADR 0006 invalidation
  amendment → [Task 349][t349].
- [Task 340][t340], [Task 349][t349], and an accepted ADR 0006 UTXO-integrity
  amendment → [Task 351][t351].
- [Task 341][t341] → [`342.config-bound`][t342].
- [Task 337][t337], [Task 345][t345], [`342.config-bound`][t342], and an accepted ADR
  0004 target amendment → [Task 342][t342].
- [`337.sql-selector`][t337] and [Task 345][t345] → [Task 344][t344].
- [Task 332][t332], [Task 337][t337], [Task 340][t340],
  [Task 345][t345], [Task 354][t354], and accepted ADR 0012/0013 amendments →
  [Task 343][t343].
- [Task 357][t357] → [Task 346][t346].
- [Task 343][t343] and [Task 346][t346] → [Task 347][t347].
- [Task 331][t331] → [`338.sql-definitive`][t338].
- [`338.sql-definitive`][t338], and either [Task 352][t352] while modern kvdb remains
  or [Task 353][t353] retiring it → [Task 338][t338].
- An accepted ADR 0010 amendment, and either [Task 352][t352] while modern kvdb
  remains or [Task 353][t353] retiring it → [Task 354][t354].
- [Task 338][t338] and [Task 354][t354] → [Task 336][t336].
- [Task 355][t355] and [Task 357][t357] → [Task 356][t356].
- [Task 345][t345], [Task 346][t346], [Task 355][t355], and
  [Task 356][t356] → [Task 348][t348].

[Tasks](tasks/) not joined by this graph remain independently schedulable.
[Task 331][t331] owns Manager admission before [Task 330][t330] adds its
duplicate-create fast path. [Task 330][t330] still keeps persistence atomicity
separate from [Task 329][t329].

[Task 333][t333] is intentionally independent: it enables the existing itest
source on PostgreSQL for early bug discovery. [Task 330][t330],
[Task 334][t334], and [Task 340][t340] own any duplicate, shared-chain, or
durable-identity defects that run exposes; they do not block the harness
wiring.

[Task 353][t353] is not ready merely because [Task 335][t335] is complete.
Scheduling it also requires verified production conversion evidence and an
explicit consumer cutover decision. If modern kvdb remains,
[Task 352][t352] precedes complete [Task 338][t338] and [Task 354][t354]. If
[Task 353][t353] retires modern kvdb first, both tasks drop their legacy
clauses. [Task 354][t354] still requires its accepted ADR 0010 amendment.

Accepted ADR changes shown in the graph are scheduling artifacts, not prose
reminders. A complete task may not start code that contradicts its current ADR
until the named replacement or amendment is accepted. The dotted milestones
retain only the dependencies stated below and are exempt from later behavioral
ADR gates where explicitly noted.

The dotted nodes are schedulable, reviewable milestones rather than prose-only
partial dependencies:

- [`340.identity-revalidation`][t340] is [Task 340][t340]'s accepted pre-code
  design record. It rechecks the minimal identity tuple, custom-signet input,
  preflight/adoption order, and database-native first-open exclusion against
  the current branches.
  It contains no Store implementation and confirms that [Task 1000][t1000]'s lock,
  manifest, and generic recovery machinery remain out of scope.
- [`341.adr0013-adapter-identity`][t341] is [Task 341][t341]'s accepted ADR 0013
  amendment. It documents kvdb's internal scope-local adapter identity without
  exposing it as a portable account ID and contains no public role
  implementation.
- [`337.sql-selector`][t337] lands the wallet-owned exactly-one account selector,
  SQLite numberless-XPub public derivation, and collision/not-found tests. Its
  acceptance gate is a tagged or merged artifact that [Task 329][t329] and
  [Task 335][t335] can build against without modern-kvdb acceptance.
- [`345.sql-discovery`][t345] lands the SQL discovery-mode schema, complete
  preexisting-account backfill, immutability proof, and backend-neutral mode
  contract. Its acceptance gate is regenerated SQL plus reopen tests on both
  SQL dialects. It does not wait for the ADR 0004 and ADR 0005 live-watch
  amendments; watch-delivery completion does.
- [`342.config-bound`][t342] lands `wallet.MaxRecoveryWindow = 100_000`, accepts
  recovery windows from zero through that bound unchanged, and rejects the
  first excess value before Store or chain access. It contains no recovery
  derivation, target-selection, or scan change, so it does not wait for the ADR
  0004 semantic-target amendment; the rest of [Task 342][t342] follows
  [Task 337][t337] and [Task 345][t345] plus that accepted amendment.
- [`338.sql-definitive`][t338] lands the accepted-request result latch, shutdown join,
  and ambiguous-commit reconciliation for SQL `ChangePassphrase`. It depends
  only on [Task 331][t331], contains no kvdb or waddrmgr rotation, and gates the lnd SQL
  password path. [Task 352][t352] remains a prerequisite for complete
  [Task 338][t338]'s kvdb result only while modern kvdb remains;
  [Task 353][t353] removes that clause.

[Task 329][t329] and [Task 335][t335] consume
[`337.sql-selector`][t337] and [`345.sql-discovery`][t345]; neither requires the
remainder of [Task 337][t337] or [Task 345][t345]. [Task 342][t342] consumes
[`342.config-bound`][t342] before its scan changes. The lnd SQL password path
consumes [`338.sql-definitive`][t338] without scheduling [Task 352][t352].
Modern-kvdb clauses may remain pending or be removed by [Task 353][t353] and do
not block the SQL milestone consumers.

## Scope rules

- Keep future PRs small and behavior-owned. A task is not permission to execute
  neighboring tasks.
- `wallet/` is the maintained library surface. Avoid voluntary changes to the
  deprecated wallet and RPC layers.
- kvdb remains legacy, dual-encryption, and single-wallet. [Task 335][t335] provides the
  converter, and [Task 353][t353] later retires kvdb from the modern Manager surface.
  Do not add a new kvdb encryption format.
- Do not revive create manifests, partial-state replay, generic write locks, or
  Manager publication state machines. Use existing database transactions and
  the smallest operation-specific compare-and-set or reconciliation needed.
- Do not add daemon itests unless a task explicitly requires integration
  evidence. Prefer focused unit and Store conformance tests.
- PostgreSQL is not a prerequisite for SQLite-focused tasks unless a shared
  Store contract changes and must continue to compile.

## Deduplication decisions

- Archived [Task 1000][t1000] is reference analysis, not one executable
  mega-task. Its surviving gaps are represented by [Task 332][t332],
  [Task 334][t334], and [Task 339][t339]–[Task 349][t349].
- Archived [Task 10001][t10001] keeps its default decision: restart a rescan
  from scratch; durable rescan jobs are not planned.
- Archived [Task 10002][t10002] is absorbed by the narrow [Task 338][t338]. A
  generic Controller operation registry is not planned.
- Ambiguous SQL commits are handled by the mutation-owning task. There is no
  standalone generic journal/quarantine task.
- Complete SQL-role evidence is a final gate after relevant tasks land, not a
  separate implementation task.
- Archived single-encryption [Task 311][t311] and [Task 313][t313] are
  abandoned. [Task 335][t335] is the only planned data transition away from
  legacy kvdb encryption; [Task 353][t353] is the later modern-API removal
  step.
- The archived DBVault task is stale; the delivered design uses `WalletVault`.
- The archived combined-rotation finding is preserved as narrow [Task 352][t352]. It is
  not part of watch-only policy, Controller result delivery, or migration.
- [Task 354][t354] owns the ADR 0010 correction before [Task 336][t336] removes
  redundant Wallet tests and [Task 343][t343] and [Task 347][t347] consume typed
  secrets. [Task 355][t355] owns concrete `txsizes` defects,
  [Task 357][t357] owns input, output, and fee-target arithmetic, and
  [Task 356][t356] owns deferred change allocation before [Task 348][t348]
  returns funded results. [Task 346][t346] and [Task 356][t356] consume
  [Task 357][t357] rather than duplicating checks.
- [Task 358][t358] alone replaces the context-free Manager `Create` and `Load` APIs.
  It follows [Task 329][t329]'s definitive genesis reconciliation and [Task 331][t331]'s
  admission contract; neither prerequisite grows a partial context adapter.
- [Task 353][t353] removes every implemented or pending modern-kvdb clause after
  migration and consumer cutover, using a task-by-task retirement ledger. Do
  not implement temporary kvdb behavior merely to delete it in the retirement
  stack.

[t311]: archive/tasks/311-single-encryption-for-all-backends.md
[t313]: archive/tasks/313-single-encryption-revision-of-the-three-prs.md
[t328]: archive/tasks/328-transaction-detail-validity-policy.md
[t329]: tasks/329-atomic-wallet-account-creation.md
[t330]: tasks/330-duplicate-wallet-create.md
[t331]: tasks/331-manager-wallet-lifecycle.md
[t332]: tasks/332-watch-only-script-policy.md
[t333]: tasks/333-postgres-manager-harness.md
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
[t1000]: archive/tasks/1000-complete-btcwallet-sql-role-readiness.md
[t10001]: archive/tasks/10001-resumable-per-wallet-rescan.md
[t10002]: archive/tasks/10002-controller-request-completion.md
