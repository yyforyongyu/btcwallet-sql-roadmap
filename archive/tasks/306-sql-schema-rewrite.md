# Task 306 — SQL schema rewrite (RETIRED — reference analysis)

Status: **retired reference analysis.** Do not execute this file. Its findings
are verified and worth keeping; its plan is void. It is **not** folded into
[[Task 1000]] — that task is untouched reference, and several of this file's
items have no owner at all yet.

## Why It Is Retired

Two independent reasons:

1. **It requires a branch outside the three-PR scope.** Items 1 and 3-6 rewrite
   migrations `000001`-`000010`, which are owned by `sql-wallet` — not #1272,
   #1291 or #1292. Editing them from above `sql-wallet` re-creates exactly the
   migration-ownership defect this file itself warns about.
2. **[[Task 1000]] already describes most of it.** Its purpose is the same goal
   ("the btcwallet-owned work required for a fresh lnd node to use the
   role-based SQLite or PostgreSQL wallet") and its ledger already covers these
   areas. It is **reference only** — not worked on in this round and not edited
   — so the `BWR-` ids below are pointers for whoever writes the follow-ups, not
   assignments.

## Where Each Item Went

| This file | Owner now |
| --- | --- |
| Item 1 — `database_meta`, bootstrap, genesis | future PR (cf. `BWR-000`) |
| Item 2 — atomic creation, ambiguous commit | future PR (cf. `BWR-013`) |
| Item 3 — `block_height` `SET NULL` -> `RESTRICT` | unowned future PR |
| Item 4 — `transaction_inputs`, descendant CTE | unowned future PR |
| Item 5 — `utxos.wallet_id`, composite FKs | unowned future PR |
| Item 6 — delete `coin_pub_key`/`key_scope_secrets` | unowned future PR |
| Manager ownership, `Close` semantics | future PR (cf. `BWR-001`) |
| Chain-cursor diagnosis (premise item 2) | unowned (307 is retired) |

**One place 1000 is better and this file was wrong.** For the PostgreSQL
bootstrap, this file prescribed pinning a `*sql.Conn` across the whole protocol
— which deadlocks when `MaxConnections = 1`, a valid configuration. 1000 already
prescribes a **connection-bound migration executor** with the advisory-lock
connection held **outside** `MaxConnections` (`:505`, `:592`, `:3165`). Use
1000's design.

**What still needs carrying across:** the analysis below is verified against the
code and should not be re-derived. In particular the `SET NULL` detach defect
(item 3), the spend-pointer-versus-outpoint problem (item 5), the `NULL`-cursor
migration rule, and the `spent_input_index` decision are findings, not opinions.
Note that the `spent_input_index` drop was decided but **not carried through**
this file's own index/wire-width/clear/verification sections — resolve that when
the ledger entries are written.

---

# Archived Analysis (non-executable)

Everything below is the original plan. Its **findings** are verified against the
code and should not be re-derived — in particular the `SET NULL` detach defect,
the spend-pointer-versus-outpoint problem, the NULL-cursor migration rule, and
the `spent_input_index` decision. Its **execution order, branch partition,
verification plan and 18-item Definition of Done are void**: they target
`sql-wallet`, assume machinery that never merges, and carry legacy/kvdb
acceptance gates the library scope excludes. Do not execute any of it.

## Premise

Much of #1291 is compensating for two wrong persistence boundaries rather than
fixing them:

1. **Wallet creation commits before initialization is complete.**
2. **One shared fact — the chain cursor — is stored N times, once per wallet.**
   Diagnosis only: the fix is [[Task 307]]. This task must not touch chain
   ownership or delete the coordination that currently guards it.

Every mechanism below is downstream of one of those. The stack should delete the
compensation by moving the boundary, not add more state to manage it.

Because the stack is unmerged, **rewrite the existing migrations** rather than
layering compatibility migrations on top.

## Critical: Migration Ownership Decides Where This Task Lands

Verified: **10 of the 11 migrations already exist at `sql-wallet`.** Only
`000011_wallet_create_complete` was added by this stack (in `wallet+db:
regenerate runtime queries`). So migrations 000001-000010 are owned by a branch
*below* `sql-wallet`.

Rewriting them "in #1291" is unsafe. If the lower PRs merge first, any database
that already recorded those versions will not rerun a modified body, so the
rewrite silently does not apply. Two options:

- **Rewrite the lower schema-owning branch and cascade every dependent PR.**
- Add forward migrations instead — which contradicts this task's premise and
  keeps the band-aid columns in history.

**Decision: rewrite the lower branch, and that branch is `sql-wallet` itself.**
Verified: the migrations arrive via `594bb0d4a sql/pg: move sql assets`, an
ancestor of `sql-wallet`; and none of it is on `upstream/master` yet, so a
rewrite is still possible rather than requiring forward migrations.

**Correction to an earlier claim here: `sql-wallet` is not headless.** It is
the head of open draft PR **#1110** ("[draft]: Introduce SQL schema for
wallet", base `interface-wallet`), and **four** open PRs base on it — #1272,
#1279, #1244 and **#1154** (@EricGrill, sqlc scripts documentation), which the
earlier cascade list omitted entirely. Consequences, not blockers: the rewrite
lands inside #1110's diff automatically, so that is where schema reviewers will
encounter it and its subscribers must be told rather than surprised; and
@EricGrill joins the notify list.

So the rewrite lands as a change to `sql-wallet` — either directly or as a PR
into it — and everything above cascades:

```
sql-wallet (schema rewritten)
  -> #1272 impl-secret-store          rebase; re-verify; re-push
  -> #1291 impl-runtime-store-6       rebase + rewrite (this task's Go work)
  -> #1292 wallet-default-sqlite      rebase; re-verify
  -> #1300 itests/very-first-itests           @GustavoStingelin — not ours
  -> #1279 sqldb/add-keyvault-sig-interface   @GustavoStingelin — not ours
  -> #1244 sqldb/ll-linter                    @GustavoStingelin — not ours
  -> #1154 docs/sqlc-scripts-documentation    @EricGrill        — not ours
(sql-wallet is itself the head of draft #1110)
```

**Owned scope is `sql-wallet` + #1272 + #1291 + #1292 only.** #1244, #1279 and
#1300 are all @GustavoStingelin's and #1154 is @EricGrill's (verified via
`gh pr list`). For those: notify the owner with the base SHA and the breaking
changes, and record the divergence here. Do not rebase them, and do not gate
this task's completion on them. #1110's subscribers get the same notice, since
the rewrite appears in its diff.

**Migration and generated-code commit shape**, so each commit is buildable:
schema change and its regenerated sqlc output land together per entity — the
repo's standing rule that `.sql` plus generated `.sql.go` form one atomic
commit.
**The renumbered sequence, stated in full.** Today's `000002_wallets.up.sql`
creates `wallets`, `wallet_secrets` *and* `wallet_sync_states`, and
`wallet_sync_states` already references `blocks` (`:160`, `:165`). Once blocks
stays where it is for this task — the cursor hoist is Task 307 — the split is
narrower than an earlier draft claimed:

```
000001_database_meta      metadata + genesis binding (item 1)
000002_blocks             global, height-keyed (shape unchanged)
000003_wallets            wallets, wallet_secrets, wallet_sync_states
000004_address_types
000005_key_scopes
000006_accounts
000007_addresses
000008_transactions       + transaction_inputs (item 4)
000009_utxos              FK -> transaction_inputs (item 5)
000010_tx_replacements
000011_utxo_leases
```

`blocks` moves ahead of `wallets` so the genesis row is seeded before any wallet
exists (item 1); `wallet_sync_states` can stay with `wallets` because it still
references only `wallets` and `blocks`, both of which now precede it. Old
`000011_wallet_create_complete` disappears (item 2). Both backends must renumber
identically. Task 307 splits the cursor out of `000003` later; this task does
not pre-split it.

Consequence for Task 305: it may proceed now against the current base, but its
resulting head — and #1291/#1292 — must later be **rebased onto the rewritten
schema base**. Record that as expected, not as a surprise.

## Root Causes

| Current machinery | Underlying schema / design problem |
| --- | --- |
| `CreateComplete`, `CreateManifest` | creation split across transactions |
| `ResetChain` † | genesis not guaranteed; history can start at birthday |
| `chainResetCoordinator` † | one wallet's syncer deletes shared block rows |
| `WriteGate` † | DB-wide rollback lacks database-level isolation |
| large runtime-store registry | DB ownership and network binding in memory |
| Go descendant-graph traversal | inputs exist only inside `raw_tx` |
| UTXO wallet-consistency triggers | `utxos` omits `wallet_id` |
| key-scope read/create/re-read | `ON CONFLICT DO NOTHING` ignores schemas |

† **Never merged — see [[Task 308]].** The owner cut #1291 at `1047a9b91`, below
the commits that introduce all three, so there is nothing here to freeze or
delete: the rows record the *design* problems, and the code never lands. Any
delete-inventory entry for them becomes **"confirm never merged"**. Item 3 and
DoD 6 read accordingly.

All eight were verified against the schema before writing this task; the
supporting evidence is inline below.

## 1. Durable Database Network Metadata

**ADR 0001 requires this and it was never implemented.** It says so three
times — `0001-multi-wallet-architecture.md:24` ("a `meta` table in each database
will bind it to a specific network upon creation. On every startup, the daemon
will verify this binding"), `:30` ("a critical backstop"), and `:59` as an
implementation step. Verified: **no such table exists** in either backend's
migrations. Instead #1291 keeps network identity and DB ownership in
`runtime_store_registry.go` memory.

```sql
-- PostgreSQL; SQLite uses BLOB for genesis_hash.
CREATE TABLE database_meta (
    id INTEGER PRIMARY KEY CHECK (id = 1),

    -- BIGINT, not INTEGER: mainnet magic 0xd9b4bef9 is 3652501241, which
    -- overflows a signed PostgreSQL INTEGER.
    network_magic BIGINT NOT NULL
        CHECK (network_magic BETWEEN 0 AND 4294967295),

    genesis_hash BYTEA NOT NULL CHECK (length(genesis_hash) = 32),
    genesis_timestamp BIGINT NOT NULL
);
```

**This needs an executable protocol, not just an ordering preference.** Both
stores currently run every migration in one `migrate.Up`, so "verify first" is
not expressible today. Specify:

1. **Migration locking — concrete mechanism per backend.** Verified: the
   SQLite migrate driver's `Lock()` is `isLocked.CompareAndSwap` (a plain
   in-process atomic; zero cross-process protection), while the PostgreSQL
   driver's `Lock()` does take `pg_advisory_lock` on a pinned `*sql.Conn`.
   Both `ApplyMigrations` helpers run migrations through `*sql.DB`.

   - **SQLite: an OS file lock, not a write transaction.** A write transaction
     cannot work here: `ApplyMigrations(db)` runs on a *different pooled
     connection* of the same `*sql.DB`, and SQLite admits one writer, so
     holding a bootstrap write transaction across it deadlocks against our own
     migrator. Take a real advisory file lock on a sidecar file beside the
     database and hold it across bootstrap *and* both migration passes.

     **Not a bare `x/sys/unix.Flock` import — this repo builds on Windows**
     (`internal/legacy/rename/rename_windows.go` alongside `rename_unix.go`,
     `bwtest/rlimit_unix.go`). Define a small context-aware file-lock behind
     build tags, or take a maintained cross-platform dependency. Do not solve
     startup serialization by dropping platform support. The `SIGKILL` test is
     likewise Unix-only; supported non-Unix platforms need equivalent
     process-termination coverage.

     **Do not copy `chain/port/port.go:acquireLockFile`** — an earlier draft
     cited it as a flock precedent and that was wrong. It opens with
     `os.O_CREATE|os.O_EXCL` (`:183-185`) and its own doc concedes the failure
     mode: "If the process is killed before the lock file is removed, this
     function will timeout and panic. In that case, the lock file must be
     manually removed." There is no `flock` call anywhere in the tree. An
     O_EXCL sidecar is not released on process death, so copying it would give
     the bootstrap a stale-lock deadlock in which one crashed process blocks
     every future open. `flock` is kernel-released on death; that is the whole
     reason to use it. Canonicalize the path (so alias paths cannot both hold
     "the" lock), return errors rather than panicking, honour context
     cancellation, and **wait** on contention rather than failing fast — a
     concurrent same-network opener is a normal startup race, not an error.
     `busy_timeout` stays as a within-process backstop, not as the
     mutual-exclusion mechanism.

     Tests: cancellation while waiting, release on holder `SIGKILL`, and two
     alias paths to the same database.
   - **PostgreSQL: our own advisory lock on our own pinned conn.** The
     migrator's lock covers only the migrator's session, so it does not protect
     the metadata read/write around it. Take `pg_advisory_lock` on a
     `*sql.Conn` this protocol pins, hold it for the whole protocol, and release
     it with an explicit `pg_advisory_unlock` rather than relying on connection
     return. Use a lock ID distinct from golang-migrate's (which derives from
     database/schema/table names) so nesting the migrator's lock inside ours
     cannot self-deadlock.

     **Pool starvation is a real hang here, and a distinct lock ID does not
     prevent it.** Verified: `MaxConnections = 1` is a *valid* configuration —
     `pg/config.go` rejects only `MaxConnections < 0` — and
     `pg/store.go` calls `SetMaxOpenConns(maxConns)` **before**
     `pg.ApplyMigrations(dbConn)`. So a bootstrap that pins one `*sql.Conn` for
     the whole protocol and then runs the migrator through the same `*sql.DB`
     consumes the only slot and the migrator waits forever: `NewManager` hangs
     on a configuration users may legitimately set.

     Resolve it explicitly, one of two ways, and say which:
     - run the whole protocol **on the pinned connection**, including the
       migrations (golang-migrate's postgres driver has a `WithConnection`
       constructor taking a `*sql.Conn`, so this is expressible); or
     - **raise and validate a minimum pool size** before taking the lock, and
       reject configurations below it at validation time rather than deadlocking
       at open time.

     Either way add a `MaxConnections = 1` bootstrap test with a **bounded**
     context, so a regression fails instead of hanging CI.
2. **Bootstrap step — version-aware, never a bare target.** Both helpers call
   `m.Up()` (`sqlite/migrations.go:45`, `pg/migrations.go:47`), so a meta-only
   step needs `Steps(1)` or `Migrate(1)`. **`Migrate(target)` is bidirectional**
   — its own doc reads "looks at the currently active migration version, then
   migrates either up or down to the specified version" — so calling
   `Migrate(1)` on a database already at version 11 **migrates down and drops
   every table**. Read the current version first; run the meta-only step only
   when the schema is below it; never pass a target below the current version;
   and define dirty-flag handling. State explicitly that verify-then-migrate
   runs on **every** open, not only on first creation.
3. **Atomic first-writer binding.** `INSERT ... ON CONFLICT DO NOTHING`
   followed by a read, inside the lock: the first writer binds the network,
   every later opener verifies. A losing writer must *verify*, never overwrite.
4. **Verify, then continue.** Only after the binding matches does the process
   apply the remaining migrations. A wrong-network process therefore mutates
   nothing — it fails holding the lock, before any other migration runs.
5. **Immutable metadata — exactly two things.** The `database_meta` singleton
   row and `blocks` at height 0 (genesis is height 0,
   `000001_blocks.up.sql:6`) must reject `UPDATE` and `DELETE` at the database
   boundary, not merely a mismatched `INSERT`. Rules or triggers are acceptable
   here because immutability cannot be expressed as a constraint — but they
   must exist in **both** backends. Nothing else is covered: Task 307's
   `chain_state` is mutable by design and deliberately outside this rule.
6. **Seed genesis — but `blocks` does not exist yet at step 2.** An earlier
   draft said "insert genesis immediately after binding `database_meta`", which
   is unexecutable: `database_meta` is migration 000001 and `blocks` is 000002,
   so at binding time there is no table to insert into.

   **Make migration 000002 seed genesis itself, reading `database_meta`.** Then
   a recorded 000002 can never exist without genesis, and no ordering discipline
   has to be remembered by callers. The stages become:
   migrate to 1 → bind and verify → migrate to 2 (which seeds genesis from the
   verified binding) → migrate 3 onward. Define crash recovery between each
   stage, and verify genesis on **every** open, not only the first.

   **The seeding statement must fail loudly on an unbound database, so use the
   scalar-subquery form, not `INSERT ... SELECT`.** An
   `INSERT ... SELECT ... FROM database_meta` inserts **zero rows and returns no
   error** when `database_meta` is empty, so any path that runs a plain `m.Up()`
   without the staged bootstrap — today's `ApplyMigrations` callers, tests,
   future tooling — would record 000002 with no genesis and no complaint,
   destroying the invariant this design rests on. Write it as values instead:

   ```sql
   INSERT INTO blocks (block_height, header_hash, block_timestamp)
   VALUES (
       0,
       (SELECT genesis_hash FROM database_meta),
       (SELECT genesis_timestamp FROM database_meta)
   );
   ```

   An empty `database_meta` then yields NULL and the `NOT NULL` constraints
   abort the migration. Same form in both dialects. "Verify genesis on every
   open" then confirms the invariant rather than papering over its absence.

   This is also why `blocks` precedes `wallets` in the renumbered sequence.

Tests: concurrent first opens with different networks (exactly one wins, the
other fails and mutates nothing); update and delete attempts on both
`database_meta` and genesis.

Database configuration then belongs to **Manager**, not to each wallet's
`Config`: one Manager owns one database runtime and pool for the configured
network, and wallets carry only wallet-specific configuration.

Removes most of the registry's identity canonicalization, network memory,
security comparison, and reset ownership.

## 2. Atomic Wallet Creation

Prepare everything **before** opening the transaction: validate mode and root
key; generate or import the root; generate wallet encryption material; derive
and encrypt all default accounts; normalize shell `InitialAccounts`; validate
names, scopes, address schemas, networks and account keys.

Then one transaction:

```
INSERT wallet
INSERT wallet secrets
INSERT wallet sync state       -- complete row, synced_height NULL (see below)
INSERT key scopes
INSERT default/imported accounts
INSERT account secrets
COMMIT
```

Multiple statements on one `sql.Tx`, **not** one giant statement — PostgreSQL
supports data-modifying CTEs and SQLite does not, so a single-statement
multi-table insert would diverge between backends. The plumbing already exists:
`execWrite` wraps its closure in a `*sql.Tx` via `dbruntime.Write`.

No genesis-block insert appears here: the genesis row is database-level and is
written by the bootstrap (item 1), so wallet creation never touches `blocks`.

**The sync-state row is the *transitional* shape, not the post-[[Task 307]]
one.** This task deliberately keeps `wallet_sync_states.synced_height` and the
per-wallet syncers (item 3, DoD 6), so creation must still insert a **complete
`wallet_sync_states` row with a NULL `synced_height`**, which the retained
queries and syncers then advance. An earlier draft justified the birthday-only
insert with "the cursor is shared" — the cursor is *not* shared during this
task, and following that reasoning here would either introduce `wallet_recovery`
early or omit state the frozen per-wallet sync path still requires.

The store receives a complete prepared aggregate. Only ciphertext and public
metadata cross into the `db` package; temporary encryption keys are zeroed after
account preparation.

**Removes:** `create_complete`, `create_manifest`, migration 000011,
`SetWalletCreateComplete`, `ErrWalletIncomplete`, `ErrCreateManifestMismatch`,
create replay and retry authentication, `recoverRuntimeMasterRoot`,
default-account retry matching, initial-account retry matching,
`Wallet.createComplete`, `Wallet.createManifest`.

A failed transaction leaves no wallet; a crash after commit leaves a complete
wallet `Manager.Load` can open.

The one unavoidable uncertainty is an ambiguous commit transport failure,
already modeled by `runtime.ErrAmbiguousTxCommit`. **Do not reconcile it by
wallet name alone**: if the original transaction actually rolled back and
another process then created the same name, a name lookup returns the *other*
wallet and reports success.

**Decision: return `ErrAmbiguousTxCommit` unchanged — but the Manager must
reconcile before returning it, because the caller cannot.**

An earlier draft here said "let the caller reconcile explicitly". That is unsafe
for `ModeGenSeed`: the Manager generates the seed and root key *internally* and
returns neither on error, so the caller has no way to tell its own committed
wallet from a same-named wallet another process created. Name-only
reconciliation would silently adopt someone else's wallet.

So on an ambiguous commit the Manager re-reads and compares the **complete
immutable aggregate it prepared** (root key material, key scopes, account
identities) — not the name. Only if that comparison is itself impossible does
the error surface. When it does surface it must be a **public**
`wallet.ErrAmbiguousTxCommit`, not only the internal `runtime` sentinel, since
callers cannot act on a sentinel they cannot name. Do not add another marker
field.

**Prerequisite (former Task 305 item S1): `Manager` stops creating kvdb
wallets**; loading them continues. Note the boundary is the **constructor**, not
`Create`: `NewManager` accepts only SQLite/PostgreSQL configuration, so there is
no `DBBackendKVDB` for `Create` to refuse (see the Manager-ownership section and
DoD 4). An earlier draft of this line said `Create` "refuses `DBBackendKVDB`",
which contradicted that decision. The standalone
daemon is unaffected — it creates through `loader.CreateNewWallet`
(`walletsetup.go:198`), not through `Manager`. This deletes only the
**creation and retry** helpers, verified to exist at these positions:
`createLegacyStore` (declared `manager.go:689`), `verifyRootKeyMatchesLegacy`
(declared `:805`, called `:766`), `recoverLegacyMasterRoot` (declared `:1865`,
called `:1850`), `decryptLegacyMasterRoot` (declared `:2004`, called `:1908`),
and `recoverPersistedMasterRoot`'s kvdb branch. Positions are **stack-tip**
(`d98e82bab`); re-verify at execution time. An earlier draft also
listed `legacyEncryptedSeed` and `readExistingLegacyEncryptedSeed`; **neither
exists anywhere in the tree** — do not go hunting for them. It removes the
waddrmgr transaction boundaries that make atomic creation impossible today.

**`openLegacyStore` must be retained.** An earlier draft listed it for deletion,
which contradicts "loading continues": `buildWallet` calls it on **every**
Manager kvdb load (`manager.go:1410`). All legacy open, load and vault plumbing
stays.

Consequence to handle: `bwtest/harness_wallet.go:113` creates via
`manager.Create`, so kvdb itest fixtures must be created through the deprecated
`Loader` and then loaded through `Manager`. That is the intended
**compatibility** shape for testing — not the standalone daemon's production
shape, which uses `Loader` for both create and load.

## 3. Chain History Stays Global; The Detach Action Is Broken Today

**Design constraint (owner's):** all wallets share one chain and one block
height. A rollback rolls back every wallet. Either all wallets are synced or
none is. Wallets are accounting boundaries over one on-chain state.

Global `blocks` is therefore **correct and stays**. An earlier draft of this
task replaced it with per-wallet `wallet_blocks`, per-wallet genesis rows,
per-wallet rollback and a per-wallet `chain_revision`; all three reviews
rejected that as the opposite of the target model, and it is withdrawn. ADR
0006's global-block decision is **affirmed**, not superseded.

**The rest of the chain work is [[Task 307]], not this task.** Making sync
genuinely all-or-none means hoisting the position into a singleton, adding a
global revision fence, and moving chain ownership from `Wallet` to `Manager` —
44 production files and 39 query sites per dialect, because `synced_height` is
joined into confirmation math (`accounts.sql:402-413`). Deleting
`chainResetCoordinator`, `WriteGate` or `ResetChain` **here** would remove the
races' only guard while per-wallet syncers still exist. Treat them as frozen
transitional code for the duration of this task: do not extend them, do not
patch them, do not delete them.

### The one chain-adjacent defect this task does fix

`transactions.block_height` declares `ON DELETE SET NULL`
(`000007_transactions.up.sql:41`) — and that action can never succeed on a
confirmed row. Verified in the same file:

- `:102-104` `check_confirmed_order_matches_block` requires `block_height` and
  `confirmed_order` to be NULL or non-NULL **together**;
- `:97` forces `tx_status = 1` (confirmed) while `block_height` is set;
- `:115-118` constrains coinbase rows to confirmed-with-block or
  orphaned-without.

`SET NULL` updates `block_height` alone, so for any confirmed row it produces
NULL block with non-NULL `confirmed_order` — a CHECK violation, and the block
delete fails. The referential action is unusable in exactly the situation it
exists for, and it converts any missed application-side detach into a runtime
error instead of the graceful degradation its comment promises.

**Fix, at the schema level:** change the action to `ON DELETE RESTRICT` and make
the detach explicit — **one `UPDATE` per affected transaction row**, clearing
`block_height` and `confirmed_order` together and setting only *coinbase* rows
to orphaned. Separate statements violate the CHECK in between, so this is one
statement by construction, not by preference.

**Non-coinbase rows keep `published` — they must not be demoted to pending.** An
earlier draft of this item said "non-coinbase to pending per `:97`", which
misread the constraint and contradicted the lifecycle. `:97`
(`check_confirmed_published`) is `block_height IS NULL OR tx_status = 1`: it
constrains rows *with* a block to be published and says nothing about block-less
rows, so a detached row may legally stay published. ADR 0006 §2.3 is explicit —
"**Effect (non-coinbase):** Regular transactions become unconfirmed (but remain
`published`)" — and it is semantically right: `pending` (`tx_status = 0`) is the
local-only pre-broadcast state (`check_coinbase_not_pending` confirms that
reading), and a reorged transaction was certainly broadcast. Demoting it would
lose that fact.

Coinbase is the exception, per `check_coinbase_confirmation_state` (`:115-118`):
a coinbase row must be either block-with-published or block-less-with-orphaned,
so a detached coinbase becomes `orphaned` (`4`) in the same `UPDATE`. This also
replaces ADR 0006 §2.3's recommendation to do that rewrite in a trigger — the
explicit `UPDATE` under `RESTRICT` makes the trigger unnecessary.

This is independent of sync ownership and survives Task 307 verbatim — 307's
global rollback is built on it.

## 4. Normalize Transaction Dependencies

Verified: there is **no** `transaction_inputs` table, and `transactions.raw_tx
BYTEA NOT NULL` is the only record of inputs. So rollback loads and decodes
every unmined transaction in Go and walks the graph repeatedly —
`CollectDescendantTxIDs` (`txstore_common.go:1036`,
`txstore_invalidateunmined_common.go:118`) and `BuildUnminedTxRecords`
(`txstore_common.go:1676`).

```sql
-- PostgreSQL; SQLite uses BLOB for prev_tx_hash. wallet_id and spending_tx_id
-- are BIGINT to match wallets.id and transactions.id (both BIGSERIAL).
CREATE TABLE transaction_inputs (
    wallet_id BIGINT NOT NULL,
    spending_tx_id BIGINT NOT NULL,
    input_index BIGINT NOT NULL,
    prev_tx_hash BYTEA NOT NULL CHECK (length(prev_tx_hash) = 32),
    prev_output_index BIGINT NOT NULL,

    PRIMARY KEY (wallet_id, spending_tx_id, input_index),

    -- Model the full unsigned 32-bit range these wire fields carry.
    CHECK (input_index BETWEEN 0 AND 4294967295),
    CHECK (prev_output_index BETWEEN 0 AND 4294967295),

    -- One transaction may not reference the same previous outpoint twice.
    UNIQUE (wallet_id, spending_tx_id, prev_tx_hash, prev_output_index),

    -- CASCADE, not RESTRICT: input rows are composition-owned by their
    -- transaction, so RESTRICT would stop the existing DeleteTx from deleting
    -- rows it owns. Child-spender leaf checks stay explicit in the delete path.
    FOREIGN KEY (wallet_id, spending_tx_id)
        REFERENCES transactions(wallet_id, id)
        ON DELETE CASCADE
);

CREATE INDEX idx_transaction_inputs_parent
ON transaction_inputs(wallet_id, prev_tx_hash, prev_output_index);
```

Persist **every** non-coinbase input, not only currently recognized wallet
spends, so a recursive CTE can discover descendants without deserializing raw
transactions. `raw_tx` stays for wire APIs and signing but stops being the
relational dependency index.

**The recursive query has requirements of its own**, and getting these wrong
silently changes behavior:

- filter to **active unmined** rows — a historical or abandoned spender is not a
  descendant to invalidate;
- exclude the roots themselves, or include them, deliberately and consistently
  with what the current Go walk does;
- deduplicate safely — a diamond dependency must not yield a row twice, and the
  CTE must terminate on cycles even though the tx graph should be acyclic;
- replace **every** raw dependency and leaf scan, not just the rollback path.
  `InvalidateUnminedTx` and the child/leaf reconciliation paths use the same
  traversal today.

**Removes:** `UnminedTxRecord`, `BuildUnminedTxRecords`,
`CollectDescendantTxIDs`, fixed-point raw-transaction scans, backend
`ListUnminedTxRecords` adapters, and several child/leaf reconciliation paths.

### The dual-write invariant, stated

`raw_tx` stays for the wire APIs while `transaction_inputs` becomes the
dependency index — two representations of the same fact. Read-equivalence and
delete-cascade are specified above; the **write** side must be too, or
descendant
discovery silently under-invalidates:

**Every transaction insert or update maintains its input rows in the same
`sql.Tx`.** No path may write `raw_tx` without writing `transaction_inputs`, and
no path may write one outside the other's transaction.

Verification: on a randomized fixture, the inputs decoded from `raw_tx` and the
rows in `transaction_inputs` agree exactly, on both backends.

## 5. UTXO Integrity By Constraint

Verified: `utxos` has no `wallet_id`; it references `addresses(id)` and
`transactions(id)` and enforces tenant consistency with a PL/pgSQL trigger,
`assert_utxo_wallet_consistency()` (`000008_utxos.up.sql:69`), which does three
lookups to compare wallet ids. Being a trigger function, it is also PG-specific,
so integrity is enforced differently per backend.

Add `wallet_id` to `utxos`. Repeating the tenant key in a multi-tenant
relational schema is worthwhile because it permits declarative composite keys:

```sql
FOREIGN KEY (wallet_id, tx_id)          REFERENCES transactions(wallet_id, id)
FOREIGN KEY (wallet_id, address_id)     REFERENCES addresses(wallet_id, id)
FOREIGN KEY (wallet_id, spent_by_tx_id) REFERENCES transactions(wallet_id, id)
```

plus a partial uniqueness constraint for the spending input:

```sql
CREATE UNIQUE INDEX uidx_utxo_spending_input
ON utxos(wallet_id, spent_by_tx_id, spent_input_index)
WHERE spent_by_tx_id IS NOT NULL;
```

Replaces the triggers with ordinary foreign keys and makes that class of
corruption structurally impossible.

**Leases need the same treatment.** Adding `utxos.wallet_id` does not by itself
remove the separate lease wallet-consistency triggers. Add `UNIQUE (wallet_id,
id)` on `utxos` and give leases a composite reference:

```sql
FOREIGN KEY (wallet_id, utxo_id)
    REFERENCES utxos(wallet_id, id)
    ON DELETE CASCADE
```

**The spend-pointer FK is mandatory — and it must tie to the outpoint, not just
to the input index.** An earlier draft referenced
`transaction_inputs(wallet_id, spending_tx_id, input_index)`. That proves the
named input row *exists*; it does **not** prove that input spends *this* UTXO.
Counterexample: transaction `T` has input 0 spending outpoint `B` and input 1
spending outpoint `A`. UTXO `A` may then point at `(T, 0)` — the foreign key and
the partial unique index both pass, while the spend attribution is false.
Balance, unspend and descendant invalidation would all operate on the wrong
credit, silently.

So the spend edge must match the UTXO's **creating outpoint**:

- add `prev_tx_id BIGINT` to `transaction_inputs` — the resolved local id of the
  previous transaction, `NULL` when the wallet does not know it — alongside the
  existing `prev_tx_hash`/`prev_output_index`;
- add `UNIQUE (wallet_id, spending_tx_id, prev_tx_id, prev_output_index)`;
- reference it from the utxo as
  `(wallet_id, spent_by_tx_id, tx_id, output_index)`, where `tx_id` and
  `output_index` are the utxo's own creation outpoint.

**Decided, so the rest of this task can be made consistent: drop
`spent_input_index`.** A second copy of the same fact is exactly what allowed
the mismatch, and keeping it would require a wider foreign key to stop the two
disagreeing. Dropping it means the partial unique index, the wire-width rewrite,
the explicit pointer-clear and the verification list must all be restated in
terms of `spent_by_tx_id` plus the utxo's own `(tx_id, output_index)` — carry
the decision through; do not leave those sections naming a removed column.

**`prev_tx_id` needs the rest of its own definition, or it just relocates the
lie.** As sketched it had no foreign key, no `ON DELETE` action, and nothing
tying it to the transaction whose `tx_hash` equals `prev_tx_hash` — so a corrupt
row could name any `prev_tx_id`, satisfy the utxo reference, and still have a
serialized input spending a different outpoint. Required:

- a foreign key to `transactions` with an explicit `ON DELETE` action (RESTRICT,
  consistent with the rest of item 5);
- **hash/ID coherence**: `prev_tx_id` must identify the transaction whose
  `tx_hash` **is** `prev_tx_hash`. Enforce it at the boundary — a
  `UNIQUE (wallet_id, id, tx_hash)` on `transactions` plus a composite reference
  is the declarative option; a trigger is not, since item 5 exists to remove
  those;
- **late-parent enrichment**: define what happens when the parent transaction
  becomes known *after* the input row is written — `prev_tx_id` starts NULL and
  is filled in, and no utxo spend pointer may reference that input until it is.
  State the ordering rule and test a parent learned after its spender.

Item 4 makes this possible; taking item 4 without it leaves the weaker invariant
in place for no reason.

Test the corruption directly: a referenced transaction and input index that are
both valid **but spend a different outpoint** must be rejected. The
"non-existent input" test does not catch this.

**Fix the UTXO wire-index types while the table is being rewritten.**
`000008_utxos.up.sql` still declares `output_index INTEGER NOT NULL CHECK
(output_index >= 0)` (`:11`) and `spent_input_index INTEGER` (`:24`). PostgreSQL
`INTEGER` is signed 32-bit, so the upper half of the `uint32` wire range is
unrepresentable — and the new `transaction_inputs` columns are correctly
`BIGINT` with unsigned-32-bit checks, so leaving these as `INTEGER` also makes
the mandatory spend-pointer FK straddle two integer widths. Both become
`BIGINT CHECK (... BETWEEN 0 AND 4294967295)`.

**The composite-FK targets mostly already exist — cite them, don't re-add
them.** Verified: `uidx_transactions_wallet_id_id UNIQUE (wallet_id, id)`
(pg `000007:19`, sqlite `:67`) and `uidx_addresses_id_wallet` (pg `000006:58`,
sqlite `:60`) are already present, so references to
`transactions(wallet_id, id)` and `addresses(wallet_id, id)` are legal as soon
as the referencing columns exist. Only `utxos` needs a new
`UNIQUE (wallet_id, id)`, which this item already adds.

**Every new reference states its `ON DELETE` action explicitly:**

| Reference | Action |
| --- | --- |
| `utxos(wallet_id, tx_id)` -> transactions | RESTRICT |
| `utxos(wallet_id, address_id)` -> addresses | RESTRICT |
| `utxos(wallet_id, spent_by_tx_id)` -> transactions | RESTRICT |
| `utxos` spend pointer -> `transaction_inputs` | RESTRICT |
| leases `(wallet_id, utxo_id)` -> utxos | CASCADE |

**No `SET NULL` on any of these — it repeats the block-FK defect.** Both spend
references are composite and include `wallet_id`, which is `NOT NULL`; a
column-list-free `ON DELETE SET NULL` nulls *every* column of the reference and
so violates that constraint at delete time. PG's column-list form
(`SET NULL (col, ...)`) is PG-15-only and SQLite has no equivalent at all, so
"pointers only" is not expressible portably.

**Decision: `RESTRICT`, with the pointer cleared explicitly before the delete.**
Unspending is already an application step, so the deleting transaction issues
one `UPDATE` clearing `spent_by_tx_id` and `spent_input_index` together — the
same single-statement discipline as the block detach, for the same reason: a
half-cleared pointer violates the partial unique index. `RESTRICT` then turns
any missed clear into a loud failure instead of silent tenant-key corruption.

## 6. Key Scopes: Simplify, Don't Accommodate

Verified: `CreateKeyScope` ends in `ON CONFLICT (wallet_id, purpose, coin_type)
DO NOTHING` (`pg/queries/key_scopes.sql:13`) and the caller then accepts the
existing row without proving its address schema matches.

**But the better fix is to remove the fields, not to design equality for them.**
Also verified: production writes `CoinPubKey: nil` in **both** backends
(`{pg,sqlite}/keyscopestore_ensurekeyscope.go:91`), so `coin_pub_key` is always
NULL in practice; and `key_scope_secrets` has **no production writer**. It also
appears in generated sqlc code (`wallet/internal/sql/*/sqlc/*`) and itest
fixtures (`wallet/internal/db/itest/fixtures_*_test.go`); those generated
queries and fixtures are deleted with the table.

**Decision: delete `coin_pub_key` and `key_scope_secrets`, unconditionally.**
The schema is unmerged and neither has a production writer; designing null-safe
comparison and enrich-once semantics for a column nothing writes is
accommodation, not a fix. If a consumer appears later it arrives with its own
migration and its own justification — it is not a reason to keep dead columns
through this rewrite.

With those gone, the upsert compares only what exists: one atomic statement
returning a row when the internal and external address types match, and
`ErrKeyScopeMetadataMismatch` otherwise — named for scope metadata generally,
not for the address schema, so it stays accurate if the compared set grows.

**It must be reachable by the callers who receive it.** Verified: the store
lives under `wallet/internal/db`, which external packages cannot import, and
`Wallet.ImportAccount` returns non-`waddrmgr.ManagerError` store errors
unchanged (`account_manager.go:467-477`). An external caller would therefore get
a sentinel it cannot name in `errors.Is`. Expose or translate it as
`wallet.ErrKeyScopeMetadataMismatch` — the same treatment F9 gives the
passphrase sentinels in [[Task 305]] — and test `errors.Is` from an **external**
package's account import, not only in-module.

Removes the read/create/re-read workflow and the create-time in-memory schema
compensation.

## What Should Remain

Not every #1291 mechanism is compensation:

- per-wallet construction singleflight — still needed, to prevent duplicate live
  `Wallet` objects and goroutines;
- keyvault encryption — application logic;
- syncer lifecycle and cancellation — application logic;
- SQL transaction error classification and ambiguous-commit handling —
  necessary;
- connection-pool ownership — necessary, but explicit at Manager level rather
  than hidden in a large per-wallet registry.

### Manager ownership — decided

Today `Config.DB DBConfig` is per-wallet (`wallet.go:172`) and `NewManager()`
takes nothing, which is exactly why `runtimeStoreRegistry` has to key and
reference-count pools. Inverting that removes the registry rather than
reorganizing it:

- **Constructor: `NewManager(ctx context.Context, cfg ManagerConfig)
  (*Manager, error)`.** It opens the one SQL runtime eagerly and runs the
  bootstrap protocol (item 1) once per Manager, so it returns an error.

  Two corrections over an earlier `NewManager(db DBConfig)` draft:
  - **`DBConfig` carries no network identity**, but the bootstrap must bind
    network magic and genesis. So the parameter is a `ManagerConfig` holding
    database settings *and* `ChainParams`, not a bare `DBConfig`.
  - **`context.Context` comes first**, because the constructor now acquires
    locks and runs migrations — blocking work that must be cancellable, per the
    repo's own convention.

  Using a config struct also means chain ownership can later add the shared
  chain client ([[Task 307]]) without another constructor break.
- **Relocate the frozen reset coordinator; do not orphan it.** The registry
  lease is what currently hands each syncer its coordinator:
  `syncer.reset = runtimeStore.reset` and `runtimeStore.reset.register(syncer)`
  (`manager.go:1500-1504`). Deleting `runtimeStoreRegistry` while this task
  *freezes* `chainResetCoordinator` would either break the build or silently
  drop the behaviour DoD 6 promises to preserve. So the Manager holds the one
  existing coordinator alongside its one SQL pool, and keeps wiring it to each
  syncer exactly as the lease did. This is **relocation, not new reset
  machinery** — Task 307 deletes the coordinator and the per-wallet syncers
  together.
- **One network authority, enforced.** `ManagerConfig` now carries
  `ChainParams`, but per-wallet `Config.ChainParams` still exists
  (`wallet.go:181`, validated `:221`) and this task does not remove it. Until
  Task 307 does, **every `Create` and `Load` must reject a per-wallet
  `ChainParams` that disagrees with the Manager's bound network, before opening
  any wallet state** — otherwise a wallet can derive addresses or query a chain
  against parameters the database binding contradicts. Record the exact
  comparison (network magic plus genesis hash, matching `database_meta`) and
  test both the match and mismatch paths.
- **Make the invalid state unrepresentable, rather than refusing it later.**
  `NewManager` **validates that its database configuration is SQLite or
  PostgreSQL** and fails otherwise; `LoadLegacy` is the only entry point that
  accepts legacy configuration. Then `Create` is SQL-only *by construction* and
  needs no `DBBackendKVDB` branch to refuse — the refusal moves to the one place
  that can enforce it once. This is a better answer to thread `UgVTm` ("stop
  supporting creating kvdb wallet") than a runtime check inside `Create`, and
  DoD 4's `make itest db=kvdb` requirement is unaffected, because those fixtures
  are built through the `Loader`, not `Manager.Create`.
- **Consequence to state, not discover: SQL becomes mandatory to construct a
  `Manager`.** A kvdb-only caller still pays for opening and migrating a SQL
  database. Acceptable given #1292 defaults to SQLite, but it is a real
  trade-off and belongs in the constructor's godoc. It is also why legacy
  loading gets its own path below rather than sharing this one.
- **`DBConfig` leaves per-wallet `Config`.** `Config` keeps only wallet-scoped
  fields (`Name`, `Chain`, `ChainParams`, `PubPassphrase`, …). With one pool per
  Manager there is no identity to key on and nothing to refcount, so
  `runtimeStoreRegistry`, `runtimeStoreOptions`/`Security`, `pgRouteIdentity`
  and `pgSessionDigest` all go with it.
- **Runtime lifetime is the Manager's, not the wallet's.** Consequence to
  implement deliberately: `evictAndCloseStore` (`controller.go:692`) currently
  closes the wallet's *owned* runtime store. For SQL it must stop doing that and
  only evict the cache entry plus release its lease; the pool closes in
  `Manager.Close`. kvdb handles stay per-wallet and are still closed there,
  matching today's "KVDB stores bypass it" comment.
- **`Manager.Close(ctx)` needs atomic admission, not just a liveness check.**
  Reading `len(m.wallets)` and then closing the pool races `Create`, `Load` and
  in-flight construction — a wallet can be admitted between the check and the
  close. Add an explicit `open`/`closing`/`closed` state guarded by the **same**
  lock that admits wallet construction (`m.creating` already serializes per
  name; this is the manager-wide equivalent). `Close` must:
  - reject new `Create`/`Load` with a stable `ErrManagerClosed`;
  - account for in-flight reservations rather than only published wallets;
  - refuse while wallets are live, naming them, rather than force-stopping —
    `Wallet.Stop` already owns teardown and a force-stop would duplicate that
    ownership;
  - be idempotent;
  - and have **decided** cancellation semantics, not "to be defined". Leaving
    them open would recreate exactly the ambiguous-completion class [[Task 305]]
    exists to fix — a cancelled context returning while the pool is still open,
    or reporting failure while the close proceeds. The rule mirrors 305's
    acceptance boundary: **before** the open->closing transition succeeds,
    context cancellation may abort with no state change; **after** it succeeds,
    `Close` runs to one definitive result and reports it, and caller
    cancellation must not abandon the accepted lifecycle mutation. Test `Close`
    against cancellation with an acceptance latch, and two concurrent `Close`
    calls.
- **Legacy load is a separate entry point: `(*Manager).LoadLegacy(cfg Config,
  legacy LegacyDBConfig)`.** A kvdb wallet's file location is per-wallet and not
  a property of the shared SQL runtime. It is load-only — kvdb creation is gone
  (item 2) — so this path never bootstraps or migrates.

## Execution Order

Note on ordering: genesis is database-level, seeded by the bootstrap (item 1),
which is why `blocks` precedes `wallets`. Wallet creation never touches
`blocks`.

This task's home is the **lower schema-owning branch**, not #1291 — see the
migration-ownership section.

1. Durable database network metadata plus seeded genesis, bootstrapped and
   verified ahead of the other migrations (item 1).
2. `transactions.block_height` becomes `RESTRICT`, with the single-`UPDATE`
   detach (item 3).
3. Atomic SQL creation; remove migration 000011 (item 2, with its
   `Manager.Create` prerequisite).
4. Normalized transaction inputs, the dual-write invariant, and recursive
   descendant queries (item 4).
5. `wallet_id` on UTXOs and leases; replace triggers with composite foreign keys
   (item 5).
6. Key-scope simplification (item 6).
7. `Manager` owns one database runtime, per the API design above.
8. Cascade the **owned** stack onto the rewritten schema base: #1272, then
   #1291, then #1292. Notify @GustavoStingelin about #1244/#1279/#1300 and
   @EricGrill about #1154 with the new base SHA and the breaking changes; do not
   rebase them.

Chain ownership, the shared cursor, the revision fence and the
coordinator/gate/`ResetChain` deletions are **[[Task 307]]**, deliberately not
steps here.

### Buildable Branch And Commit Partition

"Migration plus regenerated sqlc" is not by itself a buildable slice — the Go
callers of a changed query break in the same commit. Each entity's change is
partitioned as:

| Slice | Lands in | Rule |
| --- | --- | --- |
| schema + queries + sqlc + adapters | `sql-wallet` | per entity, both DBs |
| Store contracts + other adapters | `sql-wallet` | with their package tests |
| runtime workflows, mocks, callers | #1291 | compiles against the above |
| Task 305 adaptations | #1272 | passphrase surface only |
| default/harness wiring | #1292 | `bwtest` fixtures, itest config |

**Correction to an earlier draft of this table, which did not compile.** It put
schema plus regenerated sqlc in one commit and *all* consuming Go in the next,
while also requiring every commit to pass `go vet`. That is impossible:
dropping `coin_pub_key`, adding `utxos.wallet_id` and changing query parameters
change the generated Go API immediately, so the existing adapters stop compiling
in the very commit that regenerates them.

The standing rule requires `.sql` and generated output to be **together**; it
does not require direct consumers to be **delayed**. So schema, query source,
generated output, and every adapter needed for compilation land in one commit
per entity. Split further only where the earlier commit still compiles and its
package tests pass. Verify per commit with `go vet ./...` (which compiles
`_test.go`), not `go build`.

## ADR And Documentation

Write a **superseding ADR** rather than editing 0006 in place, and update the
transaction and reorg developer docs to match. Be precise about what it does and
does not reverse:

- **Affirms** ADR 0006's global-blocks decision, recording the shared-chain
  model as the reason. An earlier draft of this task listed global blocks among
  the decisions being superseded; that was wrong and is withdrawn.
- **Supersedes** only its unnormalized-transaction-inputs, no-`wallet_id`-on-
  utxos, and trigger-backed-integrity decisions.
- **Resolves ADR 0006's own self-contradiction**: `:106` states
  `utxos.wallet_id` is `NOT NULL` while `:163` states it is intentionally not
  stored. Item 5 settles it in favour of `:106`; say so, since the document
  currently supports both readings.
- **Records the `ON DELETE SET NULL` defect** (item 3) as a schema bug fixed at
  the schema level, not as a policy change.

[[Task 307]] writes the sync half of the story — single-owner chain sync, the
global revision fence, and the all-or-none contract. Do not pre-empt it here.

## Verification

Beyond the usual per-commit and per-head checks:

- `make sql` regenerates cleanly for **both** dialects, with no diff;
- both DB integration suites pass (`itest-db db=sqlite`, `db=postgres`);
- every migration rolls back and reapplies;
- **the detach action**: deleting a block whose transactions are still
  confirmed fails with `RESTRICT` rather than corrupting them, and the explicit
  single-`UPDATE` detach then lets the delete succeed — non-coinbase rows land
  **published** — not pending — and coinbase rows orphaned. This test must fail
  both if the action is put back to `SET NULL` and if a detached non-coinbase
  row is demoted to pending;
- **genesis is seeded and immutable**: a fresh database has `blocks` height 0
  before any wallet exists, and `UPDATE`/`DELETE` on it are rejected;
- **migration 000002 fails on an unbound `database_meta`**: run it directly,
  without the staged bootstrap, against an empty metadata table and require the
  migration to error. This is the test that distinguishes the correct
  scalar-subquery seed from an `INSERT ... SELECT`, which would insert zero rows
  and report success — run it on both backends;
- **atomic-create failure injection**: a forced failure at each insert step
  leaves no wallet row behind;
- foreign-key violations are rejected for every new composite constraint, on
  both backends;
- recursive descendant discovery returns the same set the Go walk did, on a
  fixture with multi-level unmined chains;
- **kvdb loading still works**, including `openLegacyStore`, vault delegation,
  and rotation;
- **wrong network mutates nothing**: a mismatched open fails with the schema
  untouched;
- **metadata and genesis immutability**: `UPDATE` and `DELETE` rejected on both;
- **existing chain behaviour is unchanged**: the reset coordinator, `WriteGate`
  and `ResetChain` still pass their current tests. This task freezes them; a
  behavioural diff here means scope has leaked in from [[Task 307]];
- **the dual-write invariant**: on a randomized fixture, inputs decoded from
  `raw_tx` and rows in `transaction_inputs` agree exactly, on both backends;
- **bootstrap never down-migrates**: opening a database already at the final
  version runs no downward step and drops nothing;
- **bootstrap lock survives holder death**: `SIGKILL` the holder mid-bootstrap
  and the next open proceeds; plus cancellation while waiting, and two alias
  paths to one database;
- **active versus historical descendants**: an abandoned or mined spender is not
  returned as a descendant to invalidate;
- **input cascade**: `DeleteTx` removes its own input rows;
- **spend-pointer FK**: a utxo cannot name a non-existent input;
- **concurrent key-scope mismatch**: two processes racing an upsert with
  different address types — one wins, the other gets the mismatch error;
- **ambiguous creation**: an injected ambiguous commit is reconciled against
  the prepared aggregate, and only an unreconcilable one surfaces the public
  `wallet.ErrAmbiguousTxCommit`; either way no partial wallet remains, and a
  same-named wallet from another process is **never** adopted;
- **Manager lifecycle**: `Close` refuses live wallets, rejects new `Create`/
  `Load` with `ErrManagerClosed`, is idempotent, and races a concurrent
  `Create` without admitting it after the close began;
- **legacy load** works through `LoadLegacy` alongside SQL wallets exactly as it
  does today. Prohibiting **mixed** SQL/kvdb live sets is [[Task 307]]'s (its
  DoD 7): the rationale — kvdb cannot join the all-wallet atomic sync
  transaction — does not exist until 307, mixed sets are legal today, and
  enforcing it here would breach this task's own freeze on chain ownership.

## Inherited Items From Task 305 — Decided

- **itest wiring**: fold the itest configuration change into the commit that
  changes the API. Do **not** relax the per-commit tagged vet — the exception
  existed only because the two were split.
- **`kvdb.IsMissingStore`**: remove. Its only production caller
  (`manager.go:1882`, inside `recoverLegacyMasterRoot`) disappears with the
  create path.
- **`(*kvdb.Store).DB`**: remove. It has **zero callers** — `git log -S` finds
  none across the whole stack, so the "deprecated passphrase shim" its doc names
  never used it. The `store.DB()` calls in `wallet/internal/db/itest` are a
  *different* method: `(*sqlite.Store).DB` / `(*pg.Store).DB` returning
  `*sql.DB`, declared in the build-tagged `itest.go` files. Those stay.
- **kvdb surfaces**: remove the creation surfaces (`CreateLegacyWalletParams`,
  `CreateStore`); retain every open/load surface, `openLegacyStore` included.
  Their only production caller is `manager.go:693-702`. The deprecated `Loader`
  creates through `waddrmgr.Create`/`wtxmgr.Create` directly
  (`deprecated.go:152-160`, `:255`), so removing them cannot break it.
- **the C4 comment at `wallet/manager.go:1461-1472`.** Task 305's F8 routes it
  here because C4 landed in #27, a #1291 commit (verified: `8f03737f0`
  introduces the text). After 305's F0 revert, its claim that rotation stays on
  `Wallet.ChangePassphrases` becomes false. Nothing else owns fixing it, so this
  task does — and since the vault-selection reasoning around it also changes,
  restate the whole `isKVDB` comment block rather than patching one sentence.
- **deprecated files need no change.** `wallet/deprecated.go`'s only import from
  the new tree is the kvdb package, and it uses just `kvdb.NewStore` and
  `kvdb.NewLegacyManagerVault` (`:7264`, `:7270`) — both retained open-path
  surfaces. `wtxmgr/deprecated.go` is pure walletdb. No schema, runtime, or
  `internal/sql` symbol reaches either file, so items 1-6 cannot touch them.
  They are *read* as the retained kvdb creation path, not edited. Task 305 does
  edit `deprecated.go:2864-2911` — that is its `ChangePassphrases` guard, and is
  consistent with the standing rule, which forbids making these files
  SQL-*aware*, not editing them at all.
- **the old kvdb invariant test**: replace `manager_kvdb_format_test.go` with
  `Manager.Create` **rejection** coverage plus a `Loader`-created,
  `Manager`-loaded case. Task 305 owns the new rotation test; this task owns
  removing the old one.
- **Threads — this task owns all six**, re-verified against a *paginated* sweep
  (`reviewThreads(first:100)` alone hides #1291's, which sit past node 100):
  13 unresolved overall = 7 on #1272 (Task 305's) + 5 on #1291 + 1 on #1292.
  - `UgVTm` (`kvdb/lifecycle.go:17`, "stop supporting creating kvdb wallet") and
    `UgPWI` (`kvdb/driver.go:35`, "why do we need this thin method?") are
    answered by deleting the code they ask about — resolve when the deletion
    lands, not before.
  - `UgRON` (`kvdb/driver.go:119`, `Store.DB`), `UfRns`
    (`syncer.go:220`, dead `scanTarget.AccountName` whose comment also claims
    name-matching while filtering is `(KeyScope, AccountID)`) and `UfSB5`
    (`manager.go:1162`, `InitialAccounts` documented as reserved but active for
    `ModeShell`) are schema-independent and can be fixed directly.
  - **`UfRny` (#1292, `bwtest/harness_wallet.go:127`)** — the sixth. Verified:
    `manager.Create` succeeds, then `require.NoError(h, w.Start(...))` exits the
    test *before* `h.RegisterWallet(w)`, so a startup failure leaks the
    Manager-opened store past `stopActiveWallets`. Fix: register immediately
    after `Create`, before `Start`, in both `CreateEmptyWallet` and
    `testCreateWallet`. The existing comment about not adding a second Stop
    cleanup stays correct — registration *is* the teardown handoff.

## Definition Of Done

1. `database_meta` exists, is verified at open, and a network mismatch fails at
   the database boundary with nothing mutated. The canonical genesis block is
   seeded into `blocks` at bootstrap. Both are immutable in both backends;
   registry identity/network/security comparison shrinks accordingly.
2. The bootstrap protocol is implemented as specified: a **cross-platform**
   advisory file lock for SQLite (build-tagged or a maintained dependency;
   path-canonicalized, context-aware, waits on contention, released on process
   death) and an explicitly unlocked `pg_advisory_lock` on a pinned `*sql.Conn`
   for PostgreSQL; version-aware meta-only bootstrap that can never
   down-migrate; atomic first-writer binding; verify-then-migrate on **every**
   open; and migration 000002 seeds genesis from `database_meta` **using the
   scalar-subquery `VALUES` form**, so an unbound `database_meta` aborts the
   migration on a `NOT NULL` violation rather than recording 000002 with no
   genesis row. An `INSERT ... SELECT` satisfies the words "seeds genesis" while
   silently inserting nothing, so the form is part of the requirement.
3. Wallet creation is one transaction that inserts a **complete
   `wallet_sync_states` row with NULL `synced_height`** — the transitional shape
   the frozen per-wallet syncers still require — verified by the retained
   queries advancing it. `create_complete`, `create_manifest` and migration
   000011 are gone, along with every retry-authentication path listed in
   item 2. An ambiguous commit is reconciled against the prepared aggregate,
   never by name, and any residual ambiguity surfaces as public
   `wallet.ErrAmbiguousTxCommit`.
4. kvdb creation is unrepresentable, not merely refused: `NewManager` accepts
   only SQLite or PostgreSQL configuration, `LoadLegacy` is the sole legacy
   entry point, and `Create` therefore carries no kvdb branch. The legacy
   create/recovery group is deleted; `make itest db=kvdb` passes with
   `Loader`-created fixtures.
5. `transactions.block_height` is `ON DELETE RESTRICT` and every detach is a
   single `UPDATE` clearing `block_height` and `confirmed_order` together,
   **preserving `published` for non-coinbase rows** and orphaning only coinbase
   rows. `blocks` stays global and height-keyed.
6. **Chain ownership is untouched.** `chainResetCoordinator`, `WriteGate`,
   `ResetChain` and the per-wallet syncers are neither extended nor deleted
   here; their existing tests still pass. The coordinator is **relocated** from
   the deleted registry lease to the Manager, with identical wiring. Mixed
   SQL/kvdb live sets stay legal here. That work is all [[Task 307]].
7. Descendant discovery is a recursive CTE over `transaction_inputs`; no
   rollback path deserializes `raw_tx`; and every transaction write maintains
   its input rows in the same `sql.Tx`.
8. `utxos` carries `wallet_id` with composite foreign keys and the partial
   unique index; the consistency triggers are gone from both backends; both
   UTXO wire-index columns are `BIGINT` with an unsigned-32-bit range check.
9. Every new reference states its `ON DELETE` action, and none of the spend
   references uses `SET NULL`; the lease composite FK exists; the utxo spend
   pointer ties to the utxo's own creating outpoint, not to an input index;
   `spent_input_index` is gone and every section that referenced it is
   restated. New uniqueness lands on `utxos (wallet_id, id)`,
   `transaction_inputs (wallet_id, spending_tx_id, prev_tx_id,
   prev_output_index)` and `transactions (wallet_id, id, tx_hash)`; the
   `transactions`/`addresses` pair constraints are cited as already present.
10. `coin_pub_key` and `key_scope_secrets` are deleted along with their
    generated queries and fixtures, and key-scope creation is one atomic upsert
    returning `ErrKeyScopeMetadataMismatch` on divergence — reachable as
    `wallet.ErrKeyScopeMetadataMismatch`, with an `errors.Is` test from an
    external package.
11. `Manager` database ownership is implemented as decided:
    `NewManager(ctx, ManagerConfig)` carrying database settings **and**
    `ChainParams`, `DBConfig` off per-wallet `Config`, runtime lifetime owned by
    the Manager (SQL pool no longer closed by `evictAndCloseStore`), `Close`
    with atomic admission and `ErrManagerClosed`, and `LoadLegacy` as the
    separate legacy-load path. `Close`'s **cancellation semantics** are
    implemented as decided — cancellation may abort before the open->closing
    transition, never after — with an acceptance-latch test and a concurrent-
    `Close` test. The mandatory-SQL trade-off is documented, and
    every `Create`/`Load` rejects a per-wallet `ChainParams` that disagrees with
    the Manager's bound network before opening wallet state.
12. Migrations are **rewritten**, not layered, in the renumbered sequence named
    above, identically in both backends: no compatibility migration adds or
    drops a column this task also introduces.
13. Both backends implement every new constraint identically — no PG-only
    trigger standing in for a constraint SQLite lacks.
14. `openLegacyStore` and the rest of the legacy open/load/vault plumbing still
    exist, and a Manager-loaded kvdb wallet still loads, rotates and unlocks.
15. Every slice in the branch-partition table landed where the table says —
    schema, queries, generated output and the adapters needed to compile in one
    commit per entity — and every commit passes `go vet ./...` on its own.
16. The rewrite landed in `sql-wallet`, and the **owned** stack (#1272, #1291,
    #1292) is cascaded onto it and green. @GustavoStingelin (#1244/#1279/#1300),
    @EricGrill (#1154) and #1110's subscribers are notified, and the divergence
    is recorded here; none of them is a completion gate.
17. The superseding ADR **affirms** ADR 0006's global blocks, supersedes only
    its unnormalized-inputs, no-`wallet_id`-on-utxos and trigger-integrity
    decisions, and resolves 0006's `:106`-versus-`:163` self-contradiction.
18. All six descendant threads (`UgVTm`, `UgPWI`, `UgRON`, `UfRns`, `UfSB5`,
    `UfRny`) replied to and resolved, plus the C4 comment block restated.
