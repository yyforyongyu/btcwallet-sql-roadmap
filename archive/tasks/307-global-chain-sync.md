# Task 307 — Manager-owned global chain sync (deferred milestone)

Status: **retired as duplicate reference.** Not scheduled, not an owner of any
current limitation, and no branches or cascades come from it.

[[Task 1000]] already describes this work — a database canonical frontier
(`:1133-1166`), durable participant arbitration and takeover (`:1167-1221`),
targeted canonical ceilings (`:1222-1245`), and DB-wide rollback plus
`ResetChain` in its PR 5 (`:3522-3535`). Keeping this file as a second future
owner would mean two competing plans for one invariant, so it is reference
material: read it for the verified analysis, not as a plan of record. 1000
itself stays untouched and unscheduled.

## Why It Is Deferred

Three reasons, and the third is the owner's:

1. **It needs branches outside the three-PR scope.** An earlier version of this
   file mapped its series onto `sql-wallet` plus two new branches, and required
   a second cascade of #1272/#1291/#1292. That is more than the round allows.
2. **[[Task 1000]] calls this class of work forward hardening.** That task's
   milestone *"assumes one btcwallet process per wallet database"* and lists
   simultaneous processes sharing one role schema — arbitration, takeover, owner
   replacement, multi-owner fencing — as forward hardening rather than
   milestone-critical. This task is that work.
3. **The lock question is unresolved and deserves an explicit answer.** The Go
   `WriteGate` never merges ([[Task 308]] cuts below it), but this task replaces
   it with `chain_state` as a *"database-wide chain-write lock"* acquired as
   every chain transaction's first statement. That is cross-process and enforced
   by the database rather than in-process — but it is still a lock, and the
   owner rejected the last one. Decide that on its own terms, not as a side
   effect.

## What Deferring Costs, Precisely

The [[Task 308]] cut leaves per-wallet syncers in place, so the
rollback-versus-sync race stays **open**. That race needs **two wallets on one
database**. So deferring this task costs **multi-wallet on a shared database**,
not single-wallet correctness — which is why #1291's known-limitations list says
single-wallet reorg remains self-serialized inside that wallet's syncer, and why
multi-wallet reorg tests wait for this task.

## Open Questions To Settle Before It Is Scheduled

Carried from review, unresolved:

- **Wallet admission has no executable protocol.** Onboarding is decided as
canonical rollback, but `Manager.Create` has only `Birthday time.Time`
(`manager.go:151`); the height is discovered later in `verifyBirthday`
(`controller.go:973`). Either resolve the birthday block before the creation
transaction — which needs a cancellable `Manager.Create`, and it has no context
— or roll back to boundary 1 and discover later. - **Required accounting has no
durable completion model.** `chain_state` as specified holds only height, hash
and revision, so a second Manager reports `Info.Synced = true` while the first
rebuilds. Two designs are open and neither is in the DoD. - **`Resync`**
currently keeps a deprecation cycle for callers the library scope excludes.

## What This File Is Good For

Two things, neither of them ownership:

1. **The verified analysis.** The shared-cursor diagnosis, the `WriteGate`
   autopsy (in-process contention shaping, never cross-process correctness), the
   boundary semantics of `RollbackToBlock`, and the NULL-cursor migration rule
   are all checked against the code and should not be re-derived.
2. **The unresolved lock question.** Any future multi-wallet design has to
   answer whether a **database-level** chain-write lock is acceptable when the
   in-process one was rejected. That question is open, and nothing should be
   scheduled around it until it is answered.

Keep single-wallet SQL correctness separate from all of this — one atomic
apply-and-tip transaction, expected-tip checks, a targeted scan that cannot move
the tip — so a single-wallet fix never waits on the multi-wallet lock decision.

---

# Archived Design (non-executable)

Everything below is verified against the code and is the intended design for the
multi-wallet milestone — but its **branch map, series shape, verification matrix
and Definition of Done are void** while the task is unscheduled. In particular
it maps work onto `sql-wallet` and two new branches and recascades the current
stack, which the three-PR scope forbids. Do not create branches from it. Revisit
once the row-lock and admission questions above are answered.

## The Model

Owner's directives:

1. All wallets share one chain, one block height, one rollback. A rollback rolls
   back every wallet.
2. Sync is global: either all wallets are synced or none is. No per-wallet sync
   tip, no per-wallet synced flag, and no operation advances one wallet's chain
   view without advancing all.
3. A wallet's birthday/recovery progress is accounting discovery, not chain
   sync, and must not be modeled as a sync tip.

Global `blocks` is therefore **correct and stays** — it *is* the shared on-chain
state. ADR 0006's global-block decision is affirmed, not superseded.

## 1. The Chain Position Becomes A Singleton

Today the position is per wallet (`000002_wallets.up.sql:141`), which is what
contradicts directive 2:

```sql
CREATE TABLE chain_state (
    -- Same singleton idiom as Task 306's database_meta: SQLite has no real
    -- BOOLEAN (the keyword only carries NUMERIC affinity), so both singletons
    -- use the INTEGER form rather than diverging per table.
    id INTEGER PRIMARY KEY CHECK (id = 1),
    synced_height INTEGER REFERENCES blocks (block_height)
        ON DELETE RESTRICT,
    synced_hash BYTEA CHECK (
        synced_hash IS NULL OR length(synced_hash) = 32
    ),
    chain_revision BIGINT NOT NULL DEFAULT 0,
    updated_at TIMESTAMP NOT NULL
);
```

SQLite uses `BLOB` for `synced_hash`. The length check carries an `IS NULL` arm
because the column is legitimately null before the first sync, but it must exist
— every other hash column in the schema is length-checked.

`chain_state` is mutable by design and therefore deliberately **outside** Task
306's immutability rule, which covers only the `database_meta` singleton and
`blocks` height 0.

**"At most one row" is not enough — it must be exactly one, and coherent.** The
DDL above bounds the row count but never guarantees the row *exists*, and
nothing stops a `DELETE`. That matters more here than anywhere else in the
schema: if the row is absent, the first-statement `SELECT ... FOR UPDATE` or
conditional `UPDATE ... RETURNING` that §4 makes the chain-write lock **affects
no row**, so the revision fence and the wallet-admission protocol silently stop
existing rather than failing. Same class as an `INSERT ... SELECT` that quietly
inserts nothing.

Required, all four:

- **seed the row in the migration** that creates the table, so it exists from
  the first open;
- **reject `DELETE`** at the database boundary (rule or trigger, both backends),
  while still permitting controlled `UPDATE`;
- **the tip pair is all-NULL or all-non-NULL** — `synced_height` and
  `synced_hash` must not disagree;
- **a non-NULL hash must be the hash at that height**: either derive it from
  `blocks` instead of storing it, or enforce a composite
  `(synced_height, synced_hash)` foreign key against
  `blocks (block_height, header_hash)`.

  **Price the FK arm before choosing it.** Verified: `blocks` has
  `block_height INTEGER PRIMARY KEY` and a *separate*
  `header_hash BYTEA NOT NULL UNIQUE` (`000001_blocks.up.sql:10`, `:13`). In
  both dialects a foreign key must reference a unique constraint on **exactly**
  the referenced column set, and two single-column constraints do not qualify —
  so the FK arm silently requires adding `UNIQUE (block_height, header_hash)` to
  `blocks` in both backends. **Prefer deriving the hash**: it cannot drift, and
  it adds no column and no index.

Tests: deleting the row is rejected; a mismatched height/hash pair is rejected;
an empty singleton fails loudly rather than degrading the lock to a no-op.

`synced_height` leaves `wallet_sync_states`. What remains there is birthday and
recovery metadata only — rename the table to say so (`wallet_recovery`) so no
future reader mistakes it for a second sync position. State the final shape
explicitly in the implementing commit; do not leave it to inference.

### Migration mechanics

State this rather than leaving it implied, mirroring Task 306's
migration-ownership framework:

- while the stack is **unmerged**, split the cursor out of Task 306's renumbered
  `000003` by **editing those migrations in place** — legal on this feature
  branch and the reason 306 deliberately left the cursor there rather than
  pre-splitting it;
- once the stack has **merged**, this becomes a forward migration that adds
  `chain_state`, reconciles the existing per-wallet cursors into it, and drops
  `synced_height`.

  **"Backfill from the existing rows" is underspecified, because Task 306
  deliberately leaves those cursors free to diverge.** Two wallets may sit at
  different heights, and accounting above the chosen global tip must not survive
  as though it were synced. So the migration needs a deterministic rule plus the
  matching correction in one transaction.

  **NULL dominates — an earlier draft's "minimum non-NULL" rule was wrong.** In
  this schema NULL means the wallet has synced *nothing*
  (`000002_wallets.up.sql:139-146`), and Task 306 creates every wallet that way.
  So if wallet A is NULL and wallet B sits at height 100, taking the minimum
  non-NULL value publishes global tip 100 while A has no accounting through it —
  violating all-or-none the moment the migration commits. Rolling back to
  boundary 101 changes nothing, and dropping the column then destroys the
  evidence that A was behind.

  The rule, by population:

  - **any cursor NULL** → the safe baseline for the whole database: global
    rollback to **boundary 1** (genesis retained) plus a global unsynced state,
    letting normal recovery re-derive everything. Do not salvage a partial tip.
  - **all cursors non-NULL** → the **minimum** as the global tip, plus a global
    rollback to `min + 1` so every wallet's accounting matches the published
    tip.
  - **no wallets at all** → the same unsynced/genesis baseline as the NULL case.

  Test all three populations, and assert the **resulting tip and accounting** —
  not merely that the migration completed — before the old columns are dropped.

Either way both backends renumber identically, and `.sql` plus regenerated sqlc
land in one commit with the adapters needed to compile.

**This task owns the same migration-ownership problem Task 306 solved, and must
not re-open it.** "Edit 306's renumbered `000003` in place" is only safe in the
branch that *owns* those migrations — `sql-wallet` — and by the time this task
starts, 306's Definition of Done has already cascaded and pushed #1272, #1291
and #1292 onto it. So:

- **the schema change lands in `sql-wallet`**, not in a branch above it. Editing
  it higher up re-creates exactly the defect 306 warns about: a database that
  already recorded that version will not re-run a modified body.
- **that makes every dependent head stale a second time**, which is a cost to
  plan for, not a surprise: re-cascade the owned stack (#1272 → #1291 → #1292),
  re-verify, and re-push, exactly as 306 does.
- **notify the external dependents again** — @GustavoStingelin (#1244, #1279,
  #1300) and @EricGrill (#1154), plus #1110's subscribers — with the new base
  SHA. They are not a completion gate.
- **state the complete renumbered sequence** this task produces, the same way
  306 does, rather than describing the delta.

Map the three PRs of the series below to branches explicitly; "three reviewable
PRs" without branch assignment is not an executable plan.

## 2. `chain_revision` Is Global, And It Is Required

**This is the mechanism that makes deleting the coordination safe, so it is not
optional.** The race it closes survives every other change:

syncer B reads tip H+5 and prepares a batch; syncer A rolls back to H and, being
a reorg, re-inserts a *different* block at H+5 (the `blocks` table is
height-keyed, so a reorg is DELETE-then-INSERT at the same height); B commits.
Every foreign key is satisfied — and B has attributed its transactions to the
wrong block. A block-FK check cannot catch this because the height still exists.

So:

- capture `chain_revision` before any chain or network work begins;
- compare-and-set it in **every** chain-derived write: `ApplyScanBatch`,
  `ApplyTxBatch`, tip advance, rollback, manual rewind, birthday verification;
- increment it whenever persisted chain state is invalidated;
- a stale revision produces **zero** side effects.

Prove it across two independently opened stores. There is one revision to
capture and compare, which is strictly simpler than the per-wallet version that
was originally proposed.

## 3. Chain Ownership Moves From Wallet To Manager

Verified: `manager.go:1500` builds one syncer per wallet
(`newSyncer(cfg, ..., w.id)`). Under directive 2 that is both redundant work and
the source of the race above.

- one chain-sync owner per Manager/database;
- a connected block is **one** Manager-prepared store transaction: insert the
  global block, apply every wallet's accounting changes, advance tip + revision;
- a rollback is the reverse across every wallet in one transaction;
- wallets subscribe as accounting consumers and read the same height and status;
- if any wallet needs catch-up, or any chain operation fails, the single global
  status is unsynced. It becomes synced only once every wallet's accounting is
  current at the canonical tip.

## 4. Rollback Becomes The Canonical Global Operation

Keep the existing DB-wide `RollbackToBlock(height)` — do **not** replace it with
a per-wallet primitive. The NOTE at `wallet/internal/db/interface.go:506-508`
("a DB-wide operation affecting all wallets") stops being a defect and becomes
the documented contract.

One transaction:

1. one `UPDATE` per affected transaction row, detaching `block_height`,
   `confirmed_order` and coinbase status **together** (Task 306 establishes this
   discipline and the `RESTRICT` action that makes it mandatory);
2. clamp every wallet's remaining block references — including
   `birthday_height`, which is also `RESTRICT` (`000002:165`);
3. update `chain_state` and bump `chain_revision`;
4. delete the block rows **at and above** the boundary `B` (`>= B`), which
   is what the contract below establishes — not "above the target".

### `height` is a boundary, not a retained tip — say so and keep it

**Verified, and it corrects an error carried through two earlier rounds of this
plan.** The existing contract removes blocks **at and above** `height`:
`block_height >= sqlc.arg('rollback_height')`
(`pg/queries/transactions.sql:342`, with `synced_height >= …` and
`birthday_height >= …` at `:366`, `:377`, `:389`, `:393`). So:

- **a genesis-only reset is `RollbackToBlock(1)`, not `(0)`.** An earlier draft
  here said `(0)` on the reasoning that "genesis is height 0" — under boundary
  semantics that would try to **delete genesis**, which Task 306 makes
  immutable, so the reset would fail rather than reset. (An earlier review round
  asserted the opposite — that `1` was "off by one even on its own terms" — and
  this task accepted it; that reviewer has since confirmed the claim was itself
  the off-by-one. The query text settles it: `1` is correct, and
  `txstore_common.go:1604` already calls `RollbackToBlockWithOps(ctx, 1, ops)`
  for the existing full reset.)
- **do not silently redefine the parameter.** Two options, and this task picks
  the second: rename the operation to express retained-tip semantics everywhere,
  **or** keep the established boundary meaning and document it. Keep it — the
  `>=` form is already threaded through every rollback query and both dialects,
  and renaming buys nothing but churn. State the meaning in the interface godoc
  so no caller has to infer it.
- **[[Task 10001]]'s clamp is expressed against this contract**, not against the
  method name: for a rollback with boundary `B`, the retained tip is `B-1`, so a
  job clamps to at most `B`. That is the same arithmetic as "at most H+1" with
  `H = B-1`, but it must be written in terms of the parameter the API actually
  takes.

`ResetChain`, `ResetChainOps`/`ResetChainWithOps`, both SQL implementations and
the dead kvdb one (`kvdb/txstore.go:1430`) are deleted — a full reset is a
global rollback to boundary 1, retaining the genesis row that Task 306's
bootstrap guarantees exists.

### Why `WriteGate` Goes, And What Must Land With It

The owner asked directly whether this task removes `WriteGate`. It does — and
the gate is weaker than it looks, so this is a deletion rather than a trade.
Verified at the current tip:

- **`RollbackToBlock` already runs in one `sql.Tx`**
  (`{sqlite,pg}/txstore_rollback.go:115` -> `execWriteUnlocked` ->
  `dbruntime.Write`), and its own doc says "atomically removes every block at or
  above the provided height". Database isolation already serializes it against
  concurrent writes; the gate adds nothing there.
- **The gate is process-local.** It is a `semaphore.Weighted` field on each
  backend's `Store` (`RLock` at `{sqlite,pg}/runtime.go:55`; `Lock` at
  `txstore_rollback.go:108` and `:128`). Two processes against one PostgreSQL
  database hold two independent gates, so the exclusion is vacuous in precisely
  the deployment the SQL backend exists to enable.
- **It does not close the race that matters.** A batch *prepared before* the
  rollback and *committed after* it meets no gate — the gate was released long
  before that commit. Closing that requires the global revision fence (§2).

What the gate genuinely provides is in-process **contention shaping**: an
ordinary write queues FIFO behind a pending rewind instead of reaching the
database and failing on the `RESTRICT` foreign key or a serialization conflict.
That is a behaviour difference, not an invariant.

**But a generic retry is the wrong replacement, and an earlier draft of this
section prescribed one.** "Retry on serialization or foreign-key conflict" does
not work: a stale batch that violated the block FK will usually violate the same
FK on retry, and a deterministic constraint failure must never be retried
blindly. Worse, checking the revision *late* in a transaction can deadlock
against a rollback that has already locked other rows.

**Replacement: make the `chain_state` singleton the database-wide
chain-write lock.**

1. Every chain-dependent transaction acquires that row as its **first SQL
   action** and verifies the revision it was prepared against.
2. PostgreSQL: `FOR SHARE` for non-invalidating accounting writes, `FOR UPDATE`
   for tip-changing writes and rollback — ordinary writers stay concurrent while
   rollback waits. SQLite: a conditional no-op `UPDATE ... RETURNING` (or an
   equivalent immediate write transaction) both takes the single-writer lock and
   compares the revision.
3. The transaction holds that lock while applying all accounting and block
   effects; tip advance or rollback updates tip and revision before commit.
4. A mismatch returns a stable `ErrStaleChainRevision` with **zero** effects.
   The caller discards and re-prepares against current chain state — it does not
   retry the same batch.
5. **Only** transient serialization, deadlock and SQLite-busy errors get a
   bounded, context-aware retry of the *whole* transaction from a fresh read.
   Constraint errors and ambiguous commits are never blindly retried.

This is stronger than the gate — it holds across processes — without putting a
global lock around unrelated writes.

**Audit every path that enters `execWrite`, not just block writes.** Any
operation whose validity depends on rollback-mutated accounting or on the
transaction graph must use this protocol: `ApplyScanBatch`, `ApplyTxBatch`,
connected-block apply, rollback, manual rewind, birthday verification and
required onboarding work are mandatory, and `CreateTx` is included if it can
race rollback's descendant invalidation. A wallet-local write *proven*
independent of chain state may skip the row lock — prove it, do not assume it.
Enumerate every write touching `blocks`, `synced_height`/`birthday_height`, or
`transactions.block_height`; anything that turns out not to be chain-derived is
a residual gap the gate was silently covering.

**Delete `WriteGate` only after two-store tests** prove that rollback versus
each write path either serializes or returns `ErrStaleChainRevision`, with no FK
leak, deadlock, partial side effect, or stale commit.

### Wallet Admission Must Join The Same Protocol

A connected-block transaction that applies accounting "to every wallet" has to
decide *which* wallets, and that enumeration races `Manager.Create` — especially
with two Managers on one database. The hole: a block transaction snapshots
wallets A and B, another process commits wallet C, and the first transaction
then advances `chain_state`. The database now publishes a globally synced cursor
while C's accounting was never applied. The revision fence does not catch this,
because C's creation invalidated nothing.

So:

- **a connected-block transaction locks `chain_state` before enumerating the
  wallet set**, and holds it through the apply and the tip advance;
- **wallet creation participates in the same row-lock protocol**: its
  transaction inserts the complete wallet *and* atomically moves global state to
  the required unsynced/recovery boundary **before** the wallet is published.

**Onboarding — decided, because leaving it open blocks the series.** While
`blocks` rows above the new wallet's birthday still exist, "the shared cursor
rewinds" is not by itself executable. Two candidates: replay the retained block
rows for the new wallet only, or roll the chain back and let normal sync
re-apply for everyone.

**Decision: canonical rollback.** Replay would need a second, parallel apply
path that walks retained blocks for one wallet while the cursor stays put —
precisely the per-wallet chain view this task exists to remove. Rollback reuses
the one apply path and the one cursor.

**But creation does not have a birthday *height* to roll back to.** Verified:
`CreateWalletParams` carries only `Birthday time.Time` (`manager.go:151`), and
the height is discovered much later, in `Wallet.Start` → `verifyBirthday`
(`controller.go:973`) → `locateBirthdayBlock` (`:1001`), which queries the
chain. So the admission transaction as decided above cannot execute. Pick one:

- **resolve the birthday block before the creation transaction** — capture
  `chain_revision` *before* that network work, then lock `chain_state` and
  either reject or repeat a resolution that went stale. This makes creation
  depend on
  chain availability, and it needs a **cancellable `Manager.Create`**, which
  today takes no context; or
- **preserve offline creation** by conservatively rolling back to **boundary 1**
  at admission and discovering the birthday later, accepting a deeper rescan
  than strictly necessary.

State the chain-unavailable behaviour either way, and test a reorg landing
between the birthday lookup and admission. §5's birthday-discovery flow must be
reconciled with whichever protocol is chosen — the two cannot each own the
birthday.

**If the first option is chosen, the boundary is `birthday_height + 1`, not
`birthday_height`.** Under the `>= B` contract, rolling back to the birthday
height would delete the birthday block itself and clamp the wallet's own
`birthday_height` reference at that same height.
The birthday block must be retained, so the boundary is one above it. Test the
off-by-one directly: after onboarding, the birthday block still exists and
`birthday_height` still resolves.

The cost is the one recorded in the ADR requirement below: every wallet rescans
from the new wallet's birthday.

**That cost currently has no owner, and [[Task 10001]] is not it.** An earlier
draft pointed there, but 10001 explicitly puts onboarding out of scope and says
any future optimization must be a *distinct required-accounting job owned by
this task's global state* — the right boundary, since onboarding must drive
global
sync false while an optional rescan must not. So either accept the cost
permanently and drop the "may later be reduced" claim, or file a numbered task
for the required-accounting optimization. Do not leave it pointing at a task
that disclaims it.

Test both commit orders across two stores, and inject failure between the wallet
insert and the global-state change. There must never be a committed wallet
omitted from a cursor-advancing batch, nor a visible synced state with
incomplete accounting.

## 5. Resync Is Redefined; Rescan Is Not

- **`Resync`** (`controller.go:848`) today calls wallet-scoped `RewindWallet`
  and changes that wallet's synced height (`syncer.go:878-896`). That is
  incompatible with all-or-none sync. Redefine it as rebuilding one wallet's
  accounting projection from a requested global block while global sync status
  is false for all wallets; it must never delete or rewind canonical blocks.
  Once the projection reaches the global tip, the Manager may mark everything
  synced.

  **Decided: add a new operation and deprecate `Resync`; do not silently change
  it.** `Resync`'s accepted contract is public (ADR 0004: it rewinds the global
  watermark and rescans all addresses), so redefining the same name to mean
  "rebuild one wallet's projection" would change behaviour under callers without
  a compile-time signal.

  **Two details that "deprecate pointing at it" left ambiguous, now settled:**

  - **`Resync` keeps its current behaviour until it is removed** — a global
    rollback plus all-wallet rescan, expressed against the new shared cursor. It
    does **not** delegate to the new operation: delegation *is* the silent
    behaviour change this decision rejects, and a deprecation comment is not a
    compile-time signal. If keeping it working proves impossible under the new
    ownership, it returns a stable error from a named release boundary — stated,
    not discovered.
  - **the new operation does not go on the exported `Controller` interface.**
    Adding a method there breaks every external implementation at compile time
    (`controller.go:120-158`) — the same compatibility class Task 305's T7
    avoids for `waddrmgr.AddrStore`. Put it on the concrete wallet or a local
    extension interface, unless an exported-interface break is a deliberate,
    coordinated decision.

  Name the operation explicitly, and add compatibility tests for **both** an old
  `Resync` caller and a new-operation caller.
  **A durable "required accounting outstanding" state is missing, and an
  in-memory Manager status cannot substitute.** `chain_state` as specified holds
  only height, hash, revision and timestamp. So while Manager A rebuilds one
  wallet's projection, Manager B — on the same database, which this task
  explicitly supports — still reads a tip-height cursor, reports
  `Info.Synced = true`, may advance the chain, and may serve operations against
  an incomplete projection. A crash erases A's only record that work remained.
  [[Task 10001]] confirms required accounting needs a global false status
  distinct from an optional rescan, and deliberately does not implement it.

  Two acceptable resolutions; pick one:
  - make projection replacement **one atomic transaction** that keeps the old
    complete projection visible until commit; or
  - add a **durable required-accounting state** to `chain_state` with
    revision-fenced admission, progress, completion, crash recovery and
    cross-Manager observation.

  Either way add a two-Manager test covering both a read and a cursor-advancing
  write while a rebuild is incomplete.

- **`Rescan`** (`controller.go:860`) already documents itself as scanning
  "without rewinding the wallet's global synchronization state". Preserve that
  meaning exactly. It is a wallet-local accounting job over immutable global
  blocks. Its progress must never drive `Controller.Info.Synced` or confirmation
  math. Durable rescan progress is [[Task 10001]], after this task.
- **Birthday discovery** (`controller.go:1001-1023`) calls `locateBirthdayBlock`
  over the chain and then writes `SyncedTo` **and** `BirthdayBlock` through
  `UpdateWallet`. That per-wallet sync-tip write violates directive 3 and must
  become a recovery-metadata write only, under the revision CAS.

## 6. Legacy Boundary

kvdb stores keep their own block and rollback state, so they cannot participate
in the SQL transaction that guarantees all-wallet atomicity. Keep legacy loading
in a separate compatibility mode and **prohibit mixed SQL/kvdb live sets**
within one Manager sync domain.

## Series Shape

**The branch map and migration numbering this task called mandatory — supplied
here, because without them the plan is by its own words "not executable".**

Final numbering after this task (both dialects identically). `chain_state` sits
immediately after `blocks`, since it references it; `wallet_recovery` replaces
`wallet_sync_states` in place:

```
000001_database_meta
000002_blocks               (+ genesis seed, from Task 306)
000003_chain_state          NEW: singleton cursor + revision, FK -> blocks
000004_wallets              wallets, wallet_secrets, wallet_recovery
000005_address_types
000006_key_scopes
000007_accounts
000008_addresses
000009_transactions         + transaction_inputs
000010_utxos
000011_tx_replacements
000012_utxo_leases
```

Branch map:

| PR | Branch | Base |
| --- | --- | --- |
| 1 | `sql-wallet` | schema owner |
| 2 | `chain-owner-manager` | top of the owned stack |
| 3 | `retire-reset-machinery` | PR 2 |

- **PR 1** — `chain_state`, the renumbering above, the confirmation/sync query
  rewrite, and the `wallet_sync_states` -> `wallet_recovery` rename.
- **PR 2** — chain ownership from `Wallet` to `Manager`: the single
  block-connect transaction, the rollback transaction, and the wallet-admission
  protocol.
- **PR 3** — `Resync`/birthday rework, then the deletions: coordinator,
  `WriteGate`, `ResetChain`, `ResetChainOps`/`ResetChainWithOps`.

**PR 1 lands in `sql-wallet` and therefore forces the second cascade** —
recascade #1272 -> #1291 -> #1292, re-verify, re-push, and re-notify the
external dependents, exactly as Task 306's DoD does. PRs 2 and 3 are
runtime-only and stack normally above the owned stack, so they cost no further
cascade.

Three reviewable PRs, not one commit:

1. Add `chain_state` (height, hash, revision); rewrite confirmation and sync
   queries against it; rename `wallet_sync_states` to recovery metadata. Schema
   plus regenerated sqlc in their own commits, per the standing rule.
2. Move chain ownership and block batches from `Wallet` to `Manager`; single
   block-connect transaction; global sync status.
3. Redefine `Resync`/birthday discovery as accounting jobs, then delete
   `chainResetCoordinator`, `requestFullReset`, `WriteGate`, `ResetChain` and
   the cross-wallet cancellation, **in the same series** that replaces them.

Deleting step 3's machinery before steps 1-2 land is exactly the ordering error
this task exists to prevent.

## Verification

- **all-or-none**: after any batch apply, rollback, or rewind, the global cursor
  is the only sync position and every wallet observes it — across two
  independently opened stores;
- **no per-wallet sync representation** survives anywhere in schema or Go;
- **global rollback atomicity**: one rollback rewinds every wallet's
  transactions, the cursor, and the block rows in one transaction; no wallet is
  left ahead of the shared tip;
- **the reorg-at-same-height race**: a batch prepared before a rollback that
  re-inserts a different block at the same height is rejected by the revision
  CAS with zero side effects — this test must fail if the CAS is removed;
- **rollback versus every write path, across two stores**: each either
  serializes on the `chain_state` row lock or returns `ErrStaleChainRevision`,
  with no FK leak, deadlock, partial side effect, or stale commit. This is the
  gate on deleting `WriteGate`;
- **only transient errors retry**: an injected serialization/deadlock/busy error
  retries the whole transaction from a fresh read; an injected constraint error
  does **not** retry;
- **wallet admission versus a cursor-advancing batch**, both commit orders,
  across two stores, plus an injected failure between the wallet insert and the
  global-state change: no committed wallet is ever omitted from a
  cursor-advancing batch, and no synced state is ever visible with incomplete
  accounting;
- **stale revision on every chain-derived path**, birthday verification
  included;
- **`Rescan` does not move the cursor** and does not affect `Info.Synced`;
- mixed SQL/kvdb live sets are refused;
- both backends behave identically.

## Definition Of Done

1. `chain_state` is a singleton holding tip, hash and `chain_revision`; no
   per-wallet sync tip exists in either backend.
2. `chain_revision` is CAS-checked on every chain-derived write, with a test
   that fails if the CAS is dropped.
3. One chain-sync owner per Manager; a connected block and a rollback are each
   one store transaction spanning all wallets.
4. `RollbackToBlock` is the canonical rollback; its DB-wide contract **and its
   boundary semantics** (`>= height`, so a genesis-only reset is boundary 1) are
   documented as intended behaviour in the interface godoc.
5. `Resync` and birthday discovery no longer write a per-wallet sync tip;
   `Rescan` semantics are unchanged.
6. `chainResetCoordinator`, `requestFullReset`, `WriteGate`, `ResetChain` and
   `ResetChainOps`/`ResetChainWithOps` are **confirmed never merged** (they sit
   above [[Task 308]]'s `1047a9b91` cut), and the `chain_state` row-lock
   protocol is in place regardless — it makes concurrent chain writes safe, not
   the absence of the gate: first-statement lock acquisition,
   `FOR SHARE`/`FOR UPDATE` (PG) or conditional `UPDATE ... RETURNING` (SQLite),
   `ErrStaleChainRevision` with zero effects, and bounded retry restricted to
   transient serialization/deadlock/busy errors. The audit enumerating every
   write that touches `blocks`, `synced_height`/`birthday_height` or
   `transactions.block_height` is recorded, with each proven chain-derived or
   explicitly exempted.
7. Wallet admission is atomic with global chain state: a connected-block
   transaction locks `chain_state` before enumerating wallets, and creation
   moves global state to the unsynced/recovery boundary before publishing.
   Onboarding is implemented as the **decided** mechanism — canonical rollback
   to boundary `birthday_height + 1`, not replay — with a test proving the
   birthday block survives and `birthday_height` still resolves.
8. Mixed SQL/kvdb live sets are prohibited with a stable error.
9. The superseding ADR records single-owner sync, the global revision fence,
   that ADR 0006's global-blocks decision is **affirmed**, and — explicitly —
   **the accepted onboarding cost**: a wallet added to a database already at tip
   cannot catch up privately, so its older birthday forces the shared cursor to
   rewind and every wallet to rescan together. Record it as **accepted**, not
   as deferred to [[Task 10001]] — that task disclaims onboarding. If the cost
   is to be reduced later, a numbered task must own it.
10. The branch map and final migration numbering above are followed, and PR 1's
    landing in `sql-wallet` is accompanied by the recascade, re-verification,
    re-push and re-notification of the owned stack and external dependents.
11. The superseding ADR also names the superseded portions of **two further
    accepted ADRs**, and the ADR index reflects those relationships — otherwise
    the repo retains accepted, contradictory descriptions of sync ownership and
    `Resync`:
    - **0002 `controller-syncer-architecture`** (`:18-35`): the Syncer is
      Wallet-owned and "manages the database state for synchronization". This
      task moves chain ownership to the Manager and makes that state global.
    - **0004 `targeted-rescan-vs-rewind`** (`:18-35`): `Resync` "**Rewinds**
      the global `SyncedTo` watermark" and re-scans all addresses. This task
      replaces that operation rather than redefining it, per §5's decision.
