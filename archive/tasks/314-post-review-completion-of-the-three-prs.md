# Task 314 — shrink and finish the three PRs

Status: **EXECUTABLE.** Sole execution plan for #1272, #1291, #1292. Supersedes
[[Task 313]], which delivered the single-encryption arc and is now evidence.

**Revision 5**, after four review rounds by `gpt-5.6`, `codex` and
`claude-fable-5`. Every finding from every round is incorporated; where reviewers
conflicted — including where one reversed itself between rounds — the resolution
is recorded inline.

**Review state: closed.** Round 4 verdicts — `gpt-5.6` **EXECUTABLE** (no
substantive or architectural blockers); `claude-fable-5` **Ready, terminal
round** (no findings); `codex` executable after its findings 1–3, all applied.
No design question is open. The next entry in the review files should be
**post-execution validation against DoD 1–10**, not another plan round.

> **Note on `Wallet.Stop`, because it flipped twice.** 313's R8 chose
> **restartable**. Revision 2 changed it to terminal, citing round-1 `gpt-5.6`
> blocker 3 and `codex` blocker 2, which both asked for "the base-like terminal
> contract". In round 2 **both reviewers reversed themselves** and showed the base
> is restartable — verified here directly. **R8's original choice was correct**;
> revision 2's reversal was the error. §1 now states restartable.

## Objective — the metric, precisely

**Minimise the cumulative diff from `sql-wallet`, pinned at
`2ea76dfc5e85acf84014a21e4813cb23945250fa`.** Recreating the three PRs is
allowed and welcome when the result is smaller.

Revision 1 got this wrong in two ways, both corrected here:

- it called `wallet/db_ops.go`'s **−967** "pure deletion, good", but under a
  textual-diff metric deleted lines *are* diff. **Deletion of base code counts.**
  It is often still the right move — it removes a file from the maintained
  surface — but it is never free.
- it mixed metrics: a production-only contributor table against an all-file DoD.

Report **all three** metrics for every complete candidate:

| metric | baseline |
| --- | --- |
| all-file `A+D` (`git diff --numstat`) | **24,557** (117 files, +18,844 / −5,713) |
| production-Go `A+D` (excl. `_test.go`, `sqlc/`, `.pb.go`) | **9,843** (51 files, +7,419 / −2,424) |
| changed-file count | **117** |

Choose the smallest complete candidate — not merely "smaller than baseline",
which one deleted line satisfies. **No relocating compensation machinery** out of
a named file to satisfy a per-file target.

Biggest production contributors today:

| lines | file | note |
| --- | --- | --- |
| 1305 | `wallet/manager.go` (+1215/−90) | **the target** — mostly compensation for non-atomic creation |
| 967 | `wallet/db_ops.go` (−967) | deletion; counts as diff |
| 857 | `wallet/syncer.go` (+599/−258) | |
| 797 | `waddrmgr/manager.go` (+601/−196) | single-encryption format |
| 665 | `wallet/signer.go` (+413/−252) | |
| 579 | `wallet/controller.go` (+500/−79) | must shrink, not grow |
| 432 | `wallet/runtime_store.go` (+432) | |
| 219 | `wallet/internal/db/kvdb/vault.go` (+219) | `LegacyWalletVault` — kept and renamed, see §5 |
| 194 | `wallet/state.go` (+193/−1) | must shrink, not grow |

Heads (local = origin = upstream, all commits signed), CI **12/13** each,
failing only `fmt-check` (§0):

- #1272 `impl-secret-store` `2bddcc472`
- #1291 `impl-runtime-store-6` `97548d076`
- #1292 `wallet-default-sqlite` `4f0047d85`

## AUTHORITATIVE OWNER DECISIONS

**These are owner rulings, not proposals.** They are the premises this plan is
built on.

> **To reviewers:** analyse whether the plan and the code **comply** with these,
> not whether they are the best choice. If you believe a ruling is wrong, say so
> in one clearly-labelled paragraph as a *scope question* — do not file it as a
> finding, and do not write a remedy that assumes it was reversed. A finding that
> silently contradicts a ruling below will be rejected on that ground alone.
> Rulings marked ⚑ have already been contradicted by a reviewer or by me at least
> once; check those first.

### Architecture

1. ⚑ **Single encryption for every backend, including new kvdb wallets — kvdb
   works exactly like SQL.** One passphrase, one master key. The encryption
   *format* is **not** a difference between backends. New kvdb wallets are created
   with `waddrmgr.CreateSingleEncryption` and opened with `OpenSingleEncryption`
   (`kvdb/lifecycle.go:64`, `:139`), and the vault adapter takes one passphrase.
   The only backend difference is **where the keys are kept** — waddrmgr for
   kvdb, the Store for SQL. Any statement that the modern kvdb path is
   dual-passphrase, "legacy format", or "not canonical format" is **false**;
   `buildWallet`'s comment saying so is a stale defect (§5).
2. **One database per Manager; all wallets share it.** `wallet_id` is the
   isolation and exists for exactly this. Per-wallet database files are rejected:
   they make that column dead weight and are not operable for PostgreSQL. Same
   shape for SQLite and PostgreSQL.
3. **Exactly one Manager.** Assume a single Manager always. No registry, no
   detection, no prevention of a second one.
4. **kvdb supports exactly one wallet — multiple wallets will not be supported,
   to keep things simple** (owner, restated 2026-07-30). It is a legacy system;
   keep the legacy behavior. ADR 0001's multi-wallet commitment is SQL-only. Build
   no kvdb multi-wallet admission, enumeration or `wallet_id` scoping beyond what
   the adapter already carries nominally; a second name is rejected
   deterministically, in memory (§3).
5. ⚑ **Create runs in one database transaction — that is the root fix.** Do not
   add locks, intermediate states or recovery machinery to make half-created
   wallets survivable. A transaction does **not** replace in-memory publication;
   the two are separate problems (§1).
6. ⚑ **Assume a single `Load`.** Concurrent same-name `Load` is **documented as
   unsupported**, not fixed. The race predates us at `sql-wallet` and has zero
   production callers (§1a). Add no singleflight, `creating` map, channel,
   admission counter or construction lock.
7. **No config defaulting.** Delete `withDefaults`. Require `Backend`; require
   the SQLite path or PostgreSQL DSN; validate once in `NewManager` (§4).

### Scope

8. **Minimal means the total diff from `sql-wallet`**, pinned `2ea76dfc5` — not
   the size of each new change. **Recreating the three PRs is welcome** whenever
   the total shrinks. Deleting base code counts as diff.
9. ⚑ **No voluntary deprecated-file changes; forced repair only.** No new code,
   no refactors, no improvements, **no bug fixes** in `wallet/deprecated.go` or
   `wallet/deprecated_*_test.go`. Change is permitted **only** where the compiler
   or our own shared-code changes force it — a break we introduced is ours to
   repair, but a defect that predates us stays. (The old headline said
   "zero-touch", which reads as forbidding even the forced repair the same ruling
   permits.)
   `gpt-5.6` 13 asked for byte-identity and `codex` 6 asked to revert the vault
   line; **this ruling stands over both** — see §9's necessity table, which
   distinguishes forced repair from improvement.
10. **kvdb→SQL migration is out of scope and deferred.** No converter is
    designed, specified or scheduled; mention it only as the intended solution.
    The modern Manager refuses legacy-dual wallets (§8); the deprecated `Loader`
    keeps opening them unchanged.
11. ⚑ **No new kvdb store-secret methods in this round.** Do **not** implement
    kvdb `GetWalletSecrets`/`UpdateWalletSecrets`. Take the **minimal fix** that
    gets the path working (§5).
12. **Scope is the Wallet library and its storage stack** — corrected after all
    three reviewers found the previous wording ("scope is `wallet/`") forbade work
    this plan requires. The boundary is the *library*, not the directory:
    `wallet/`, `waddrmgr/`, SQL source and generated assets, ADR 0014, and the
    **existing** `bwtest`/`itest` harness plus Makefile wiring. **Out of scope:**
    RPC and legacy-RPC API work, any new daemon-itest case, and any new
    daemon-itest gate. `wallet/deprecated.go` changes only where ruling 9 permits
    forced repair.

### Naming

13. **The waddrmgr-backed vault is named `LegacyWalletVault`.** Owner ruling.
    Rename from `LegacyManagerVault` (§5).

## Salvage sources — exact locations

Copy the listed hunks. **Do not cherry-pick the WIP commits.**

| source | location | take | discard |
| --- | --- | --- | --- |
| `deprecated.go` trim (measures +9/−9) | worktree `.worktrees/t308`, HEAD `a8b6db532` | the trim, already folded into its owning commits. **A starting point, not a cap** — §9's predicate fix should reduce it | — |
| legacy-refusal reorder (§8) | **saved** to `.lgtm/pivot/salvage/kvdb-legacy-refusal-reorder.patch` (57 lines) from the uncommitted `wallet/internal/db/kvdb/lifecycle.go` in worktree `.worktrees/t308` @ `a8b6db532` | the whole hunk — refusal moves ahead of `migration.Upgrade`, wtxmgr migrations preserved. Reviewed and confirmed correct; **unbuilt, untested** | — |
| Manager-owned Store | branch `t313/r8-wip` @ `16c9b0aa4` | one Store per Manager; `kvdb.OpenContainer`/`BindWallet` (a walletdb file that opens with no wallet in it — prerequisite for transactional create); deletion of every transient store open; **and its restartable-`Stop` rework** — deleting `stopped`/`ErrWalletStopped`/`onStopped` is exactly the base shape §1 now requires, so R8 was right on this point | coarse mutex, terminal fence, `ErrManagerClosed`, `terminal` flag |

**Already saved** (see the table). Keep the source worktree until the patch is
applied and tested. The hunk existed
in one worktree only and a branch recreation loses it silently.

## 0. Immediate — the CI failure

One blank line inside the import group of
`wallet/internal/db/kvdb/vault_test.go`, added with the `chaincfg/v2` import for
the new watch-only kvdb test. `gofmt` accepts it; `gosimports` removes it.

**Process cause, recorded so it stops recurring:** after the watch-only fixes only
`make lint-check` was re-run and the head was reported clean on that basis. CI's
"Format, compilation and lint check" runs `fmt-check` **and** `sqlc-check` **and**
`lint-check`. The earlier 81-column miss had the same shape.

The lesson is that the three are **coupled** — never report a head clean on one of
them. It is *not* "run all three after every edit": those targets mutate files
before checking cleanliness, so running them on a dirty tree mid-rewrite is unsafe
(`codex` 6). Cadence lives in DoD 9 — narrow tests while iterating, all three
coupled checks once per stable final head.

## 1. The core contract — persistence and identity are separate

All three reviewers found revision 1's central error: it claimed a transactional
Load "subsumes" the Load race. It does not. `Store.GetWallet` already runs in a
read transaction and the race exists anyway.

- A **database transaction** protects persisted state.
- The **existing Manager mutex** protects the in-memory cache.

They are different problems, so atomic Create cannot close the publication one —
but under §1a that problem is *scoped*, not solved by adding machinery.

### 1a. Concurrent `Load` is a documented precondition, not a fix here

Owner ruling: **assume a single `Load`.** Measured before deciding:

- the race is **pre-existing at base**. `sql-wallet`'s `Load` reads the cache
  under `RLock`, builds outside the lock, then blind-writes
  `m.wallets[cfg.Name] = w`. Our stack changes its shape, not its existence;
- `Manager.Load` has **zero production callers** — all nine call sites are tests;
- today's failure mode: both callers miss, both build (each opening its own
  handles via `openLegacyStore` / `runtimeStoreFactory`), both publish, the
  second overwrite wins, and the loser is a live wallet with open handles the
  cache no longer tracks. `onStopped` is already instance-guarded
  (`if m.wallets[name] == w`), so the loser cannot evict the winner. For kvdb the
  second `openLegacyStore` hits bbolt's exclusive file lock and blocks or times
  out instead of yielding two live instances.

So: **replace the `TODO(yy)` at `wallet/manager.go:1033` with a `NOTE`.** A TODO
promises a fix and invites "when?"; the precondition is what we mean:

> `Load` is not safe for concurrent use with the same wallet name. Two racing
> callers both miss the cache and build independent wallets over one database,
> and only the last published is tracked. Call it from one goroutine per wallet.

**No check-then-insert in `publish`** — `gpt-5.6` 2 and `codex` 2, both right, and
revision 2 was inconsistent here: it declared the race unsupported and then
half-fixed it. "Discard your struct" has no defined cleanup policy even with a
shared Store (the loser still owns a timer, context, cancel func, cache, state and
syncer), and implementing one outcome of a race makes it read as supported. Keep
base's check-build-publish shape untouched. Add no singleflight, loser cleanup,
construction lock or concurrent-`Load` test. If a consumer ever needs the
guarantee, per-name singleflight is a separate PR.

The precondition covers `Create`/`Load` too, not just `Load`/`Load` — once
`beginCreate`/`awaitCreate` are deleted, same-name Create-versus-Load has the same
post-commit, pre-publication window. State it once, as the caller contract:

> At most one lifecycle operation per wallet name may be in flight. Callers
> serialize `Create` and `Load` for a name. Concurrent duplicate `Load` is
> unsupported; sequential `Load` returns the cached pointer.

Different wallet names remain independently operable.

The rest of the contract, stated once so nobody rediscovers R8:

- `NewManager(ctx, ManagerConfig)` opens and owns one Store — one SQL pool, or
  one possibly-empty kvdb container.
- **The Manager's existing `sync.RWMutex` is the only lock.** It guards exactly
  three things: cache lookup, publication, and the in-memory kvdb name binding
  (§3). **No eviction** — restartable wallets stay cached, so there is nothing to
  evict. Under §1a it is *not* widened to span wallet construction. Add no second
  lock, map, channel, counter or lifecycle state.
- ⚠ **`Wallet.Stop` is restartable, which *is* the base shape.** Revision 2 said
  "terminal … exactly as at base" and that was **factually backwards** — `gpt-5.6`
  blocker 1 and `codex` blocker 1, both verified independently here: base
  `walletState.toStarting` CASs `lifecycleStopped → lifecycleStarting`, base `Stop`
  ends in `w.state.toStopped()`, and **all eight** terminal symbols
  (`ErrWalletStopped`, `stopped`, `stopRequested`, `stopResult`, `stopClaim`,
  `finishStop`, `onStopped`, `discardUnstarted`) are **absent at `sql-wallet`**.
  The terminal model is stack-added, and it exists only because each wallet owned
  and closed its own Store — a reason Manager-owned Store removes.

  So: `Stop` cancels and joins workers, locks the vault to wipe decrypted
  material, returns to `Stopped`, and **stays in the cache**. It closes nothing
  shared. A later `Load` returns the same pointer; a later `Start` restarts it.
  **Delete** all eight symbols above plus `runtimeStoreClose` and cache eviction.
  Under ruling 8 this is the smaller change: it deletes stack-added machinery
  instead of preserving it, and it shrinks `controller.go`, `state.go`,
  `wallet.go` and `manager.go` at once.

  Start/Stop stay caller-serialized as at base; inherited concurrent Start/Stop
  races are not solved here. Test: one focused `Start → Stop → Start` proving the
  same pointer restarts and the shared Store stays open.
- `Manager.Close()` is synchronous: it closes the owned Store and returns that
  error, after the caller has stopped and quiesced its wallets. That precondition
  is the contract: **no** close fence, `closed` state, `ErrManagerClosed`, drain
  gate, admission counter, `Manager.Stop`/`Unload`, or cancellation protocol —
  **and no `sync.Once` or cached `closeErr`** (`codex` 11). The caller calls it
  exactly once; repeated close and use-after-close are invalid caller use.
- `Manager.Close` performs **no wallet callback** and acquires no lifecycle lock;
  it runs after caller quiescence and closes the Store directly. With no eviction
  callback and no Close-held mutex, there is no re-entry path to design around —
  which is the point of keeping the mutex narrow.

Tests: two wallets sharing one pool; `Stop` not closing it; `Start → Stop →
Start` on the same pointer; one physical Manager close.

## 2. Transactional Create — the main diff reduction

`wallet/manager.go` is +1215/−90 and most of it compensates for non-atomic
creation: `beginCreate`/`awaitCreate` and the `creating` map, the
`legacyExisted`/`runtimeExisted` classification, partial-create replay (14
mentions of "partial", 6 of "replay"), and `discardUnstarted`.

Required shape:

- prepare expensive work **before** the transaction *where the existing API
  allows it* — see the per-backend recipe below — and zero prepared plaintext
  after use;
- one aggregate transaction per backend, stated separately so SQL never grows a
  legacy sidecar (`codex` 8):
  - **kvdb** — one `walletdb.Update` creating the waddrmgr/wtxmgr namespaces,
    canonical waddrmgr state, wtxmgr state, and all wallet/account metadata;
  - **SQL** — one SQL transaction creating the wallet, secrets, default accounts,
    initial imported accounts and sync metadata, with **no** waddrmgr/wtxmgr
    sidecar;
  expose one aggregate backend operation, not a generic transaction handle to
  wallet code;
- transaction-local ops for account writes; never call a public Store method that
  opens a nested transaction;
- open **long-lived** runtime managers only after commit;

### 2c. The kvdb recipe — resolving a real API conflict

`codex` found revision 3's three requirements mutually unsatisfiable for kvdb,
and every premise checks out:

- `CreateSingleEncryption(ns walletdb.ReadWriteBucket, rootKey, passphrase, …)`
  (`waddrmgr/manager.go:2159`) **derives and persists in one call**, so "all
  derivation before the transaction" cannot hold without splitting waddrmgr into
  a new public prepare/persist API — a broad refactor that would cost more diff
  than the transaction it shortens (ruling 8);
- `kvdb.Store.CreateImportedAccount` opens its **own** `walletdb.Update`
  (`accountstore_createimported.go:38`) and drives the in-memory `s.addrStore`,
  so calling it after commit means a second transaction — non-atomic Create;
- `InitialAccounts` are valid for shell wallets, so they must be inside the
  aggregate transaction.

The resolution, which needs no new lock, state or public API:

1. validate params and derive the caller-supplied root **outside** any
   transaction;
2. **SQL** keeps preparing scrypt and encrypted wallet/account material before
   its transaction — its shape already supports that;
3. **kvdb** calls `CreateSingleEncryption` **inside** the one aggregate
   `walletdb.Update`. Atomic *persistence* is the requirement; moving a one-time
   key derivation out of the transaction is not, and no new waddrmgr
   prepare/persist API is mandated for it;
4. for initial accounts, open a **temporary, transaction-local** manager from the
   just-written namespace with `OpenSingleEncryption`, which takes a
   `walletdb.ReadBucket` — and `ReadWriteBucket` embeds `ReadBucket`
   (`walletdb@v1.6.0/interface.go:95-96`), so the same write transaction serves.
   Write accounts through the existing bucket-taking helper
   `putImportedAccount(ns, …)` (`:103`); **no** nested `walletdb.Update`, and no
   duplication of waddrmgr row invariants.

   **Wiring detail** (`codex`, verified): that helper is a method on `*kvdb.Store`
   and reaches the receiver's `addrStore` (`:110`, `:153`, `:175`), so it cannot
   take the temporary manager as an argument. Inside the update callback, bind the
   temporary manager to a **private transaction-local Store adapter** —
   `NewStore(handle.DB, nil, temporaryManager)` (`driver.go:98`; a nil `txStore` is
   already tolerated, `accountstore_helpers.go:104`) — and call the unexported
   helper through that. Defer closing the temporary manager. **Do not** call
   `StoreHandle.BindWallet` or assign the shared handle's `Store`/`AddrStore`/
   `TxStore` fields until after the aggregate transaction commits, so uncommitted
   in-memory state is never reachable. An equivalent private helper taking the
   manager directly is fine if it measures smaller; add no public API and no new
   publication state;
5. discard that temporary manager on **both** commit and rollback. It is never
   published, so a rollback cannot leave mutated live cache state. Only the
   long-lived runtime managers open after commit;
6. place §2b's late-stage fault **after the final initial-account write**, so
   namespace, manager, account, sync and side metadata all prove rollback
   together.

If measurement later shows a private prepared-state refactor is smaller, it stays
a valid candidate — but do not mandate it before measuring.
- **Create is insert-only.** An existing wallet always returns `ErrWalletExists`
  and is never authenticated, replayed or mutated. If commit succeeds but opening
  the runtime fails, a later `Load` finds the complete wallet and another `Create`
  returns `ErrWalletExists`. No compensation state.
- then **delete** all the machinery named above.

**`Load` stays on the existing single Store read.** Once Create is atomic, Load
cannot observe a partial wallet. Add a Store-level aggregate read transaction only
if a demonstrated multi-query snapshot can tear — measure first. Concurrent
same-name `Load` is not a transaction problem at all; see §1a.

### 2a. Over-create currently mutates the wallet and reports success

An existing wallet is rejected only when replay creates zero accounts, and
rootless shell wallets bypass root-key verification because an empty
`MasterPubKey` returns success. A second `Create` with the same name, a
**different passphrase**, and one extra valid initial account inserts that
account, makes `created > 0`, and returns success — leaving a wallet whose secrets
still use the original passphrase.

Falls out of insert-only Create. The regression test must prove absence of
**damage**, not just rejection: `ErrWalletExists`; the original passphrase still
unlocks; the second passphrase does not; the account list is unchanged.

### 2b. Rollback must be proven, not asserted

A function can open one transaction and still omit a namespace, account, secret
or metadata write. Require **one late-stage fault injection per backend** (kvdb, SQLite,
PostgreSQL): assert rollback leaves no wallet, namespace, account, secret or sync
state, and that a clean retry succeeds. A failure at the *final* write proves the
earlier writes roll back with it, so per-stage cases are needed only where a
backend has a genuinely separate transaction boundary (`codex` 7 — every-stage
injection would add hooks and test code that count against ruling 8). Pair it with
a zero-search for post-transaction creation writes. Use table-driven test-only
callbacks; add **no** production stage enum or permanent fault-injection
interface.

## 3. kvdb single-wallet admission

A kvdb database holds one wallet — the legacy layout has a single waddrmgr
namespace with no wallet identity in it. Reject a second name deterministically,
in memory, under the Manager mutex (§1). No persisted wallet-name metadata, no
cross-Manager coordination. Implemented on `t313/r8-wip`; salvage it.

## 4. Delete `withDefaults`

Owner decision. `DBConfig.withDefaults()` has **eight production call sites** (plus its
definition at `runtime_store_config.go:166`), each
re-deriving defensively because no reader knows whether its config was normalised
yet — the tell of an un-normalised value passed around:

```
runtime_store_config.go:80,142   runtime_store.go:93,172
wallet.go:506                    manager.go:801,1076,1402
```

No production caller relies on it: `deprecated.go` sets `DBBackendKVDB`
explicitly and the bwtest harness sets `Backend` explicitly. It serves a caller
that does not exist, and costs:

1. the eight defensive re-derivations;
2. a SQLite path inferred from `filepath.Dir(KVDB.DBPath)` + the constant
   `wallet.sqlite`, so two wallets whose kvdb files share a directory silently
   alias onto one SQLite file from two zero values;
3. coupling the SQL path to a **legacy kvdb path**, meaningless for a pure-SQL
   deployment that has no kvdb file;
4. `defaultSQLiteDBName` and its tests.

Replace with: validate once in `NewManager`; require `Backend`; require the
SQLite path or PostgreSQL DSN. All eight sites become plain field reads.

**Knock-on:** #1292's subject changes from "the library defaults to SQLite" to
"the itests default to SQLite" — the harness and Makefile choose, which is what
they already do. That makes #1292 smaller, and dissolves the "no end-to-end test
of the public default" finding: there is no public default left to test.

**Shared SQLite is not a defect.** Under decision 3, wallets in one Manager share
one file and pool deliberately. Resolve the path once in `ManagerConfig`. No path
registry, no collision detector, no per-wallet filename. Multiple Managers over
one database remain unsupported by caller contract.

## 5. `LegacyWalletVault` — the vault is kept, renamed, and its rationale fixed

Per rulings 1, 11 and 13. **The minimal fix is a rename and a comment
correction — no new kvdb store methods, no custody rework.**

### 5a. Why the vault exists: custody, not format

Revisions 1–2 and my own analysis twice gave the wrong reason. The corrected one,
verified:

- **Format is not the reason.** The modern kvdb wallet is single-encryption
  (ruling 1): `CreateSingleEncryption` / `OpenSingleEncryption`, and the adapter's
  `Unlock(ctx, passphrase)` takes one passphrase. Its layout —
  `mpriv`/`cpriv`/`cscript` + encrypted HD seed — maps **1:1** onto
  `db.WalletSecrets`. So `GetWalletSecrets` *is* implementable for kvdb. There is
  no shape obstacle. (The 5-slot `mpub`/`cpub` mismatch is real but applies **only**
  to the legacy-dual wallets the deprecated loader opens, not to this path.)
- **Custody is the reason.** In kvdb, waddrmgr *is* the address store
  (`buildWallet` sets `addrStore = legacyStore.AddrStore`) and waddrmgr's own
  methods need waddrmgr's own unlocked crypto keys:
  `addressstore.go:834` `addrStore.Decrypt(...)`, `:886` `ImportScript`,
  `:903` `ImportWitnessScript`, `:926`. (`:703` `ImportPublicKey` does not — the
  earlier "five call sites" claim was four.) A `WalletVault` caching its own copy
  in `unlockedState` would leave waddrmgr locked and those calls failing with
  `ErrLocked`: two copies of one key, two lock states, contradicting ADR 0010.

`LegacyWalletVault` is a **view, not a copy** — `struct{ db; mgr }`, no keys and
no lock state of its own, all methods delegating to the waddrmgr that already owns
them. That is what keeps exactly one lock state.

### 5b. The minimal fix

1. **Rename** `LegacyManagerVault` → `LegacyWalletVault` (ruling 13) —
   `kvdb/vault.go`, `buildWallet`, `deprecated.go`, `vault_test.go`. Stack-added
   code, so no base shape is touched. The old name is actively misleading: it is
   the single-encryption vault used by **both** the modern kvdb path and the
   deprecated loader, and only the second use is legacy.
2. **Fix `buildWallet`'s comment**, which is stale on three counts and asserts the
   opposite of what 313 shipped — it is what made me misstate the design twice:
   - "guarded by **two passphrases**" — false, single encryption;
   - "delegating **both halves**" — false, rotation collapsed in 313;
   - "**no canonical-format KVDB wallet is ever created**" — flatly false,
     `lifecycle.go:64` creates exactly that.

   Replace with the custody reason from §5a. This is a correctness fix to a
   comment that lies, so it belongs in the commit that owns the branch, not §10.

### 5c. Measurement, closed

Not a gate. Revision 1's arithmetic was wrong in a way both `gpt-5.6` (blocker 4)
and `codex` (finding 3) caught: the adapter is required by the **deprecated
loader** either way, so its 219 lines never leave and cannot be counted as
deleted; `vault_test.go` is a further 317 lines any comparison must include.

313's R5 ("the modern runtime constructs only the canonical vault") contradicted
owner-approved H3 option (a). **H3 wins; R5 is amended, and [[Task 311]]'s "never
built" statement with it.**

### 5d. Future cleanup — out of scope (ruling 11)

Retiring the vault from the modern path is **not** "implement `GetWalletSecrets`
when the format allows" — the format already allows it. It is **moving key custody
out of waddrmgr**, which requires waddrmgr to stop needing its own unlocked keys:
rework the four sites above, add a waddrmgr import entry point accepting an
already-encrypted blob (removing the vault-encrypt → store-decrypt →
waddrmgr-re-encrypt round trip), then implement kvdb
`GetWalletSecrets`/`UpdateWalletSecrets` with cache coherency after secret
updates, and handle `DeriveAccountKeys`'s dependency on waddrmgr's unlocked
private crypto key. Only then does `buildWallet`'s backend branch disappear and
the modern kvdb path become structurally identical to SQL.

That same dependency is what keeps the vault in `deprecated.go`, so both uses
retire together — after deprecated loading and signing are separated.

## 5e. PR ownership map

Recreating the branches is allowed (ruling 8), so commit ancestry cannot be the
assignment. Without this map an executor can build a correct final stack that
leaves #1272 or #1291 uncompilable alone (`codex` 2).

| PR | owns |
| --- | --- |
| **#1272 `impl-secret-store`** | the single-passphrase `keyvault` contract; the canonical waddrmgr single-encryption format; minimal public/internal secret fields and projections (§9); `LegacyWalletVault` + rename (§5); the public spendable-import sentinel and canonical-kvdb enforcement (§7); standalone watch-only/private-root validation (§9); ADR 0014 corrections; their focused tests and generated SQL |
| **#1291 `impl-runtime-store-6`** | Manager-owned Store and one database per Manager; simple `Manager.Close`; restartable cached wallets (§1); the single-name lifecycle precondition (§1a); atomic insert-only Create for both backends incl. §2c; one-wallet kvdb admission (§3); `wallet.ErrMigrationRequired` mapping and refusal order (§8); `withDefaults` deletion and required Manager config (§4); rollback and lifecycle tests |
| **#1292 `wallet-default-sqlite`** | **removal** of the library-default behaviour and its commit; retains only the existing harness/Makefile choice that defaults itests to SQLite. No new daemon-itest case or gate |

Every intermediate head must compile and pass its own applicable checks. Fold
cross-cutting hygiene into the earliest commit introducing the affected code.

## 6. Preserve what 313 delivered

A smaller rebuild must not silently lose the watch-only fixes. These are DoD
invariants with named tests:

- watch-only unlock exposes the script key and **skips absent private account
  keys** — `TestSingleEncryptionWatchOnlyUnlocksWithAccounts`;
- the one watch-only passphrase re-wraps master and script material —
  `TestSingleEncryptionWatchOnlyRotatesPassphrase`;
- script bodies are never written under the pass-through metadata codec —
  `TestSingleEncryptionScriptsNeverPlaintext`;
- `ConvertToWatchingOnly` leaves a canonical wallet openable —
  `TestSingleEncryptionConvertToWatchingOnlyReopens`.

## 7. Canonical kvdb must actually enforce ADR 0012

Both `gpt-5.6` (7) and `codex` (4) noted revision 1 asked only for an ADR edit, so
an executor could satisfy it without fixing behaviour. Verified: kvdb's
`NewImportedAddress` rejects private-key input and then accepts script-only input
without checking wallet mode; `RequireAddressPrivKeyOnSpendable` is called only
from the shared **SQL** path (`addressstore_newimported.go:73`), and the code
documents the exemption.

Production requirement:

- spendable **canonical** kvdb rejects public-only and script-only imports with
  `ErrSpendableWalletNeedsAddressPrivKey`;
- watch-only kvdb keeps accepting them;
- **gate on format, not on backend** (`codex` 3). In the kvdb imported-address
  transaction, read the namespace's `mgrver` via `waddrmgr.IsSingleEncryption` and
  apply `RequireAddressPrivKeyOnSpendable` **only** to the canonical format.
  Applying it indiscriminately would change legacy-dual behaviour, which ruling 10
  forbids. This mirrors the SQL store-boundary validation and adds no field.
- tests on the **two real public methods** — `AddressManager.ImportPublicKey` and
  `ImportTaprootScript` (verified: `address_manager.go:183`; there is no
  witness-script method on the public interface) — spendable rejects via
  `errors.Is`, watch-only accepts;
- **then** correct ADR 0014, stating that it narrows ADR 0012's kvdb
  grandfathering to legacy/deprecated wallets. Do not imply private-key import is
  being added.

Whether transitional kvdb supports private-key import at all is a separate scope
choice — do not silently imply this task implements it.

## 8. `wallet.ErrMigrationRequired`

Production exports only `kvdb.ErrLegacyWalletFormat`, under `wallet/internal/`, so
no external caller can `errors.Is` against it. Callers must distinguish "no wallet
yet" from "needs converting" from "corrupt".

- **ordering** — done, uncommitted, in the salvage table: refusal now precedes
  `migration.Upgrade`, so an old wallet can no longer report a migration failure
  instead of the stable answer;
- **export** — add the sentinel in package `wallet` and map it at the boundary.

## 9. Remaining correctness and doc items

- **#1272 standalone safety.** At its head, `WatchOnly=true` with a generated or
  imported private root reaches `CreateSingleEncryption`, which infers watch-only
  from a nil root; #1291 adds the rejection. Move that validation and its tests
  into #1272.
- **Secret projections — accounts *and* addresses.** `AccountSecret` is one field,
  but `AccountSecretRow` and both SQL queries still fetch six discarded
  identity/public projections. `codex` 4 found the same defect in the address
  half, verified here: all four `GetAddressSecret` / `GetAddressSecretByScriptPubKey`
  queries (both dialects) select `w.is_watch_only AS wallet_is_watch_only` with
  **zero** readers in either adapter, the `INNER JOIN wallets AS w` exists *only*
  to fetch it (the `WHERE` filters on `a.*`), and the doc comment still describes
  the deleted `ScriptIsSecret` key-selection design. Drop the column, the comment
  and the now-unused join in both dialects, then regenerate with `make sql`. Keep
  the ordinary address-info `WalletIsWatchOnly` and internal `HasScript`
  projections — those have production readers. This is open thread `U-9mi`.
- **`ErrSpendableWalletNeedsAddressPrivKey` is unreachable** (`codex` 5, verified):
  it lives in `wallet/internal/db/interface.go`, so lnd cannot `errors.Is` the
  refusal that §7 makes a public guarantee — the same defect §8 fixes for the
  migration error. Expose a `wallet`-package sentinel, preferably aliasing the
  internal one so there is a single error identity, and document the refusal on
  the two methods that actually carry no private key —
  `AddressManager.ImportPublicKey` and `ImportTaprootScript`. There is **no**
  witness-script method on the public interface; an earlier revision invented one.
  Add no mode or presence fields.
- **`storeSyncedTo`'s predicate is inverted** (`wallet.go:506`). It asks
  `== DBBackendKVDB`, so an *unset* backend reads as SQL and a deprecated-loaded
  kvdb wallet takes the SQL path — which is the only reason `deprecated.go` needs
  its pinned `cfg` line. Deleting `withDefaults` does **not** fix this: unset
  `Backend` is `""`, still `!= DBBackendKVDB`. Invert it to *explicitly SQL*,
  correct anyway under ruling 7 where a required `Backend` means an unset one is
  by definition not a Manager-created wallet. Those three lines then leave
  `deprecated.go` entirely, fixed in a file we already touch.
- **`deprecated.go` necessity table.** Record line by line what forces each edit,
  and classify it — the ruling-9 distinction is *forced repair* versus
  *improvement*:

  | lines | edit | forced by | class |
  | --- | --- | --- | --- |
  | 2 | `w.cfg.DB.BeginReadTx()` → `w.db.BeginReadTx()` | `Config.DB` retyped `walletdb.DB` → `DBConfig` | compile-forced |
  | 1+1 | `keyvault.NewWalletVault` → `kvdb.NewLegacyWalletVault`, and the freed import | our stack made `w.keyVault` load-bearing. At base there is **one** behavioural use, in `account_manager.go` — a single narrow account-derivation path, and **none** in `controller.go`/`manager.go`/`signer.go`. At head there are **nine**, spanning unlock, lock, passphrase change, derivation and signing across all four files. `WalletVault.Unlock` needs `GetWalletSecrets`, which kvdb does not implement (base *and* head, identically), so keeping the base line leaves a deprecated kvdb wallet unable to unlock or sign | **forced repair** — the break is ours; `codex` 6 correctly identifies it as not compile-forced, but its remedy ships strictly worse than base |
  | 3 | pinned `cfg: Config{DB: DBConfig{Backend: DBBackendKVDB}}` + comment | `storeSyncedTo`'s inverted predicate | **avoidable — delete**, fix the predicate instead |
  | — | field realignment around the collapsed constructor call | gofmt | churn, not behavior |

  **Measure the final delta; assume no cap** (`codex` 5/6). The t308 `+9/−9` hunk
  is a salvage *starting point*, not the expected result — with the predicate fixed
  it should be about +6/−9. The gate is qualitative, not numeric: **every** changed
  line is compile-forced, forced shared-code repair, or gofmt-only, and **zero**
  lines are improvements.
- **ADR 0014 contradicts itself.** Line 30 says "SQL keeps distinct private and
  script data keys" (correct — `EncryptedCryptoScriptKey`); line 112's
  reconciliation says SQL has no separate script-key row. Replace :112 with: the
  change removes the *caller-visible* key class, not the stored key — SQL still
  persists a script data key as envelope internals, as §1 describes. Then
  re-verify the ADR-0012 paragraph against §7.

## 10. Hygiene

1. Introduce-then-remove of `resolvePrivKey` / `resolveDerivedPathPrivKey` — fold
   the end state into the introducing commit.
2. Three commit bodies describe rotation APIs their patches no longer contain,
   fallout from 313's rotation collapse: the code was reworded, the messages were
   not.
3. Eight commit-body lines at 73 columns (limit 72).
4. Orphan comment fragments from the field deletions:
   `wallet/signer.go:~860` — a stale pre-typed-vault fragment sits directly above
   the correct comment; delete the fragment. Same treatment for the twin at
   `wallet/internal/db/pg/addressstore_common.go:31-35`.
5. `TestControllerStop_LocksVaultWhenUnlocked` — underscore in a test name.
6. Dropped doc comment on the rewritten `TestHandleLockReq_Errors`.
7. `bwtest/mock/addr_store.go` — `Encrypt`/`Decrypt` doc comments still claim
   "implements keyvault.Vault", untrue since the typed-methods change.
8. `RescanTarget` landed inside `wallet: route sync runtime through Store` rather
   than its own commit.
9. **`deprecated.go` necessity table** — now in §9, since it is correctness
   rather than hygiene. It classifies the **final measured** delta into the two
   permitted classes (compile-forced, forced shared-code repair) plus gofmt-only
   churn, with zero improvements. No fixed line cap.

## 11. Task authority cleanup — **DONE**

`300-INDEX.md` rule 1 named 314 while its decision table still declared 313/R8's
coarse mutex, close fence and restartable `Stop` decided, `deprecated.go`
byte-identical, and the waddrmgr vault never rebuilt. Rewritten:

- rule 1 now states the Create-only transaction, one-DB-per-Manager and no
  defaulting, and that a transaction does **not** subsume publication (§1a);
- decisions 1/1b rewritten to §1's **restartable** `Stop` and simple `Close`; new 1c
  (no new lock), 3 (one DB per Manager) and 4 (delete `withDefaults`);
- the superseded table's waddrmgr-vault row rewritten to §5, and the
  coarse-mutex row corrected;
- the `deprecated.go` owner ruling refined to compile-forced `+9/−9`;
- branch state refreshed to the real heads, thread counts, pagination trap and
  the uncommitted-salvage warning;
- 313 marked historical.

[[Task 311]]'s status and references now point at 314 (313 = historical
evidence), and its vault bullet is **restated as the final rule** rather than an
assertion plus amendment — `codex` 9's point that a binding file must not carry
both a rule and its reversal.

Round-3 additions, after all three reviewers found residue:

- Index decision 1c no longer promises the removed check-and-insert;
- the superseded-`Close` row now resolves to 314's simple caller-quiesced
  `Manager.Close`, not 313 R8's terminal fence;
- `withDefaults` count corrected to eight; the salvage patch described as saved
  but unbuilt; the fixed `+9/−9` deprecated cap removed in favour of measurement;
- [[Task 310]] marked **HISTORICAL, not binding** — its byte-identical standing
  rule forbade the forced repair ruling 9 permits, so it can no longer claim
  independent authority.

## 12. Threads — disposition, not count

35 unresolved today: #1272 **23**, #1291 **8**, #1292 **4**. A hardcoded count
lets new threads be ignored and old ones blanket-resolved, so build a **paginated
disposition table keyed by thread ID**: path, finding, `fix` / `moot` /
`push back`, owning commit, evidence.

> **Pagination trap.** #1272 has 111 threads and #1291 has 127, so one
> `reviewThreads(first:100)` query reports #1291 as **zero** unresolved when it
> has eight. Paginate and reconcile `totalCount` against unique IDs.

**#1291's eight do not all "die" from one mechanism** — `gpt-5.6` 3 and `codex` 3.
`codex` supplied the exact split; use it as the template for the other two PRs:

| thread | finding | disposition |
| --- | --- | --- |
| `U-uXH`, `U-7dV` | partial-create races | **moot** — atomic, insert-only Create (§2) |
| `U1V9E` | per-wallet store open/close window | **moot** — Manager-owned Store, and `Stop` neither closes it nor evicts (§1) |
| `U1VsY`, `U-uBn` | concurrent same-name `Load` | **push back** — inherited, pre-existing at base, unsupported by caller contract (§1a, ruling 6). Not claimed as fixed |
| `U-uBD` | shared global-chain rollback race | **push back / defer** — one shared pool does **not** coordinate two syncers mutating the shared `blocks` table. Multi-wallet chain sync and reorg coordination are not delivered by these PRs. Belongs to the deferred chain-sync work; **do not reactivate** Task 307 or Task 1000 here, and add no registry or global write gate |
| `U-202`, `U-7oM` | commit hygiene | **fix** by folding the commits |

Never claim the shared-chain race is fixed by pool sharing.

Reply and resolve only once the fix is pushed and green.

## Definition of Done

1. **All three metrics reported** against pinned `2ea76dfc5`, before and after,
   and the chosen candidate is the smallest complete one — not merely under
   24,557 / 9,843 / 117. `wallet/manager.go`, `wallet/controller.go` and
   `wallet/state.go` all reduced, with no compensation machinery relocated
   elsewhere.
2. **Zero added definitions or uses relative to `sql-wallet`** — measured by diff,
   not by bare-token grep (`codex` 7: base legitimately contains `lifecycleStopped`
   and "stopped" prose, so a literal search for `stopped` is both impossible and
   dangerous) — for: `beginCreate`, `awaitCreate`, the `creating` map,
   `legacyExisted`, `runtimeExisted`, `discardUnstarted`, partial-create replay,
   `withDefaults`, `LegacyManagerVault`, `ErrWalletStopped`, `stopRequested`,
   `stopResult`, `stopClaim`, `finishStop`, `onStopped`, `runtimeStoreClose`, and
   the stack-added terminal `stopped` **field declaration** specifically. Every lifecycle state still present relative to
   `sql-wallet` has a recorded justification.
3. §0 fixed; **all** checks GitHub reports are green at each of the three final
   SHAs.
4. §1's contract implemented and tested, including §1a's `NOTE` replacing the
   `TODO(yy)` at `wallet/manager.go:1033`; §2 landed with 2a and 2b; §3, §4, §7,
   §8, §9 done.
5. §5 closed as written — vault kept, **renamed `LegacyWalletVault`** with zero
   remaining `LegacyManagerVault` hits, `buildWallet`'s three false claims
   replaced by the custody reason, and 311 restated. **Zero** kvdb
   `GetWalletSecrets`/`UpdateWalletSecrets` implementation (ruling 11).
6. §6's four named tests present and passing.
7. §9's `deprecated.go` necessity table complete; §10's hygiene list complete.
   (§11 is already done.)
8. §12: disposition table complete and **zero unresolved threads** at the three
   final pushed SHAs.
9. **Verification cadence** (`codex` 12 — the full matrix after every edit costs
   more than it catches). *While iterating:* the narrowest relevant package/backend
   test. *Once per final head, after the stack is stable:* per-commit
   `go vet ./...`; `go vet` under tags `<none>`, `itest`, `kvdb_itest`,
   `sqlite_itest`, `postgres_itest`, `db_itest`; the **unit and DB-backend
   suites** (not a daemon-itest run — that boundary stands under ruling 12);
   `-race` on touched packages; and all three of `make fmt-check`,
   `make sqlc-check`, `make lint-check`. The three format/lint checks stay coupled
   — §0 exists because only `lint-check` was re-run.
10. Commits GPG-signed (`%G? = G`); `wallet/interface.go` untouched. **`deprecated.go`
    measured, not assumed** — every line classified in §9's necessity table as
    compile-forced or forced repair, with **zero** lines classed as improvement, and
    the pinned-`cfg` three gone via the predicate fix. **Preparation-only boundary
    holds: no new daemon-itest case and no daemon-itest gate.**
