# Index — which 3xx task actually executes

Read this first. All three reviewers (`gpt`, `fable`, `codex`) independently
flagged the same hazard: nine task files read as nine cumulative plans, and an
executor can follow a superseded design. They are a **replacement chain**, not a
backlog.

## The rule

0. **[[Task 314]]'s "AUTHORITATIVE OWNER DECISIONS" section is the top of the
   authority chain.** Thirteen numbered owner rulings live there. Everything in
   this index and in every 3xx file is subordinate to them. **Reviewers: analyse
   compliance with those rulings, not their merit** — a finding that contradicts
   one is rejected on that ground; raise it as a labelled scope question instead.
   Rulings flagged ⚑ there have already been contradicted at least once.
1. **Execute [[Task 314]] only.** It is the sole execution plan for #1272,
   #1291 and #1292. It supersedes [[Task 313]], which delivered the
   single-encryption arc and is now evidence.

   314's objective is different in kind: minimise the **total diff from
   `sql-wallet`** (pinned `2ea76dfc5`), not the size of each new change.
   Recreating the three PRs is allowed if the result is smaller. Owner decisions
   set its shape — one passphrase for both backends (kvdb conversion deferred),
   **Create** inside one database transaction as the root fix **instead of** more
   locks or intermediate states, one database per Manager shared by all its
   wallets, and no config defaulting. The transaction decision supersedes 313's
   R8 apparatus wherever R8 existed to make partial state survivable.

   It does **not** subsume in-memory publication: a transaction protects
   persisted state, the Manager's existing mutex protects object identity. Under
   314 §1a concurrent `Load` of one name is a **documented precondition**, not
   fixed here — the race is pre-existing at `sql-wallet` and `Manager.Load` has
   zero production callers.
2. **Tasks 305–313 are evidence**: verified findings, reader counts, thread
   inventories. An old instruction executes **only when Task 314 explicitly says
   to carry it forward or implement it**. Citations, evidence, and
   moot/reversed dispositions never reactivate an old instruction.
3. **[[Task 311]] is doctrine**, incorporated by 313 and still binding. Not
   independently executable.
4. **Task 1000 and 10001/10002 are read-only reference.** Not owners, not
   scheduled.

## Superseded designs — do not implement

| Where | Superseded design | Replaced by |
| --- | --- | --- |
| **Task 308 "Revision 2"** (from `# Revision 2` onward, incl. "Revision 2 Additions") | Creating new kvdb wallets with a constant `InsecurePubPassphrase` | 311 + 313 §A: that still builds waddrmgr's two master keys and `CKTPublic`/`CKTPrivate`/`CKTScript`. A constant credential changes user interaction, not the format. Use a single-encryption format discriminated by `mgrver`. |
| Task 308 Revision 2 | Coarse lifecycle mutex **while keeping** wallet-owned Store handles and close-on-`Stop` | 314 §1: Manager owns the Store and `Wallet.Stop` closes nothing — but there is **no coarse lifecycle mutex**. The existing `sync.RWMutex` guards the `wallets` map only |
| Task 308 Revision 2 | Atomic aggregate creation scoped to SQL, "kvdb creation is already atomic" | 313 §C: kvdb `Store.CreateWallet` is `notImplemented`; the atomic contract must cover every canonical backend |
| Task 313, earlier revision | An explicit new encryption-format marker column | 311 + 313 §A: reuse waddrmgr's existing `mgrver`; add no field, registry, manifest, completion marker or write lock |
| Task 311, earlier revision | Watch-only conversion makes *script bodies* plaintext | 311 (corrected): only public **metadata** goes plaintext; script bodies stay encrypted under the canonical script key, as canonical SQL already does |
| Task 313, earlier revision | Amend ADR 0012 to permit spendable script-only imports | 313 V1 (reversed): **enforce** ADR 0012. A script is a spending *condition*, not the signing material to satisfy it |
| Task 309 candidate 2 | Retype `AccountSecret.AccountNumber` to `*uint32` | 313 V5: delete it and four sibling fields — zero production readers |
| Task 309 candidate 5 | Keep public `HasScript` because it is load-bearing | 313 V3: the **internal** projection is load-bearing; the public field is write-only |
| Task 312 §A | Fix the incomplete locked-state wipe | Moot — 313 deletes the dual-rotation machinery containing it |
| Task 313, earlier revision | `Manager.Close` snapshots and stops wallets; async `Close(ctx)` with a cancellation/join protocol and an in-flight admission counter | **314 §1** (not 313 R8, whose terminal fence is also superseded): `Close()` is simple and synchronous — it closes the owned Store and returns that error, after the **caller** has stopped and quiesced its wallets. No fence, `closed` state, `ErrManagerClosed`, drain gate, admission counter, `sync.Once`, cached `closeErr`, wallet callback or lifecycle lock |
| Task 313 R5 / 311:175 | `LegacyManagerVault` is never built; the modern runtime constructs only the canonical vault | **Amended by 314 §5.** The premise stands (nothing at `sql-wallet` constructs it), the conclusion does not. Renamed **`LegacyWalletVault`** (owner ruling 13). It is the **single-encryption, waddrmgr-backed** implementation of `keyvault.Vault` — one passphrase, typed methods, no `CryptoKeyType`. It exists because in kvdb waddrmgr holds the keys and must stay unlocked for its own script imports (**custody**), *not* because the modern kvdb format is dual-passphrase — it is not (ruling 1). Owner-approved 313 H3 option (a) wins over R5. Retiring it needs key custody moved out of waddrmgr; kvdb store-secret methods are **explicitly out of scope** (ruling 11) |
| Task 312 §C / §D | Add `ScriptIsSecret` and legacy-vault TODOs | Moot — the field is deleted. (The "canonical vault only" half is amended above.) |
| Any "vault selection keys on `mgrver`" phrasing | Reads as a runtime branch choosing between vaults | 313 R5/R10: **Manager admission** keys on the kvdb namespace `mgrver`; legacy versions are refused with `wallet.ErrMigrationRequired`. (The canonical kvdb path *does* construct `LegacyWalletVault` — see 314 §5) |
| Task 310 | Relocate T7's rotator out of `deprecated.go`; byte-identical zero-touch as an independently binding standing rule | **[[Task 310]] is HISTORICAL and not binding.** The relocation is moot (never written in the rebuild), and its byte-identical rule is superseded by **314 ruling 9** — *no voluntary deprecated-file changes; forced repair only* — which permits the compile- and shared-code-forced edits 310 forbade. Ruling 9 is the only active deprecated-file rule |

## Decisions — all closed

| # | Decision | Status |
| --- | --- | --- |
| 1 | **Store ownership** | **DECIDED — Manager-owned.** `NewManager(ctx, cfg) (*Manager, error)` opens and owns one database: one SQL pool, or one possibly-empty kvdb container. All of that Manager's wallets share it. `Manager.Close()` is **simple and synchronous** — it closes the Store once, after the caller has stopped and quiesced its wallets. That precondition is the contract: **no** terminal fence, `closed` state, `ErrManagerClosed`, drain gate, admission counter, `Manager.Stop`/`Unload` or cancellation protocol. Caller-owned Store rejected (below). 314 §1, superseding 313 §C-bis + R8 |
| 1b | **`Wallet.Stop` shape** | **DECIDED — restartable, which is the base shape.** Stop cancels and joins workers, locks the vault to wipe decrypted material, returns to `Stopped`, and **stays in the Manager cache**; it closes nothing shared. A later `Load` returns the **same** pointer and a later `Start` restarts it. **Delete** the stack-added terminal machinery: `ErrWalletStopped`, `stopped`, `stopRequested`, `stopResult`, `stopClaim`, `finishStop`, `onStopped`, `discardUnstarted`, `runtimeStoreClose`, cache eviction — all **absent at `sql-wallet`** (verified). **This restores 313 R8's original choice.** 314 rev 2 briefly said "terminal, exactly as at base", citing round-1 `gpt-5.6`/`codex`; in round 2 both reviewers reversed themselves and proved base is restartable. Start/Stop stay caller-serialized as at base. 314 §1 |
| 1c | **Locking** | **DECIDED — no new lock.** The Manager's existing `sync.RWMutex` guards the `wallets` map and the in-memory kvdb name binding. It is **not** widened to span wallet construction, and no `creating` map, channel, await protocol or counter is added. Concurrent same-name `Load` — and same-name `Create`/`Load` — are **externally serialized by the caller**. `Load` keeps base's check-build-publish shape: **no** check-and-insert, loser cleanup, singleflight or concurrent-`Load` test. There is no eviction, and `Manager.Close` takes no lifecycle lock and makes no wallet callback. 314 §1/§1a |
| 2 | **kvdb single-encryption format shape** | **DECIDED — option (a)**, approved explicitly (owner, 2026-07-30). A distinct credential-aware waddrmgr create/open format writing the **reserved next namespace `mgrver`**; legacy create/open and the migration list untouched; the credential-free migration path never advances a wallet to it. Not a thin adapter: watch-only waddrmgr wallets have no `masterKeyPriv`/`cryptoKeyScript` and reject lock/unlock, so the format has real work. 313 H3 — **delivered**, at the current heads |
| 3 | **One database per Manager** | **DECIDED** (owner, 2026-07-30). One SQLite file / one PostgreSQL database per Manager, shared by every wallet it owns, with `wallet_id` distinguishing them — that column exists for exactly this. Per-wallet database files are rejected: they would make `wallet_id` dead weight and are not operable for PostgreSQL. Same shape for both SQL backends. Multiple Managers over one database remain unsupported by caller contract. 314 §4 |
| 4 | **No config defaulting** | **DECIDED — delete `withDefaults`** (owner, 2026-07-30). `Backend` is required; the SQLite path or PostgreSQL DSN is required; validate once in `NewManager`. No default derived from a legacy kvdb path, no `defaultSQLiteDBName`, no re-derivation at the **eight** production call sites. #1292's subject becomes "the **itests** default to SQLite". 314 §4 |

**No decisions remain open.** [[Task 314]] is the plan; **313 is historical** —
the sections below marked 313 are evidence for how a decision was reached, not
instructions. Includes the single-Manager assumption (owner: assume there is
always one manager — no registry, no detection, no prevention).

**Also settled this round** (every residual question closed):

- **Watch-only kvdb mirrors SQL** — one passphrase (possibly empty) → one master
  key → one script key; no private key. `Lock`/`Unlock` begin working on
  watch-only kvdb managers. Not a new design; it is what
  `keyvault/vault_genesis.go` already does. 313 H3
- **Vault exposes typed methods** — `EncryptPrivate`/`EncryptScript`; no
  `CryptoKeyType` on the canonical contract. Collapsing to a *single* secret key
  is a recorded follow-up, out of scope. 313 R3
- **ADR 0014 is small and extends ADR 0009** rather than restating it; ADR 0012
  needs no edit. 313 R7
- **kvdb name binding is in-memory only**, not persisted and not restart-stable.
  313 R9
- **The three-PR split survives**; #1292 keeps its two logical commits. 313 R6

**Not a decision:** ADR 0014 is execution work **inside #1272** (313 R7), not a
thing to approve.

**Rejected, recorded so it is not revisited:** a caller-owned public Store
(`OpenStore`/`NewManager(store)`). `db.Store` and the backend constructors are
internal, so caller ownership needs a new public opaque Store API plus
constructors — public surface deferred rather than forced into these three PRs.
Caller-supplied pools can be added deliberately later if lnd needs to share one.

## Implementation traps recorded (second review round)

Verified against the code; see 313 "Implementation requirements" and 311.

- **`mgrver` cannot be an ordinary migration.** `migration.Upgrade` runs
  credential-free *before* `waddrmgr.Open` (`kvdb/lifecycle.go:98` vs `:103`) and
  the engine stamps the version itself
  (`walletdb/migration/manager.go:152`). Appending the single-encryption version
  would mark legacy-dual wallets converted **without rewriting ciphertext**;
  bumping `latestMgrVersion` would stamp new versions onto dual layouts.
- **`CryptoKeyType` still sits on `keyvault.Vault.Encrypt`/`Decrypt`** — deleting
  `ScriptIsSecret` is not enough to clear the canonical boundary.
- **kvdb `Store.CreateWallet` is `notImplemented`** and creation currently builds
  waddrmgr/wtxmgr before any Store exists, so atomic kvdb creation needs a Store
  constructible over an uninitialized walletdb.
- **The modern passphrase API change must be named explicitly** (`PubPassphrase`
  removal, `PrivatePassphrase` → `Passphrase`), or an executor may legitimately
  leave the creation structs dual-shaped.
- **No daemon-itest gate**: package/backend coverage only, per the
  preparation-only scope.

## Settled by the owner

- kvdb creation **stays**; new kvdb wallets are single-encryption.
- 311's Option **A** — public metadata plaintext, private keys and full script
  bodies encrypted.
- **Migration is out of scope and deferred.** No converter is designed,
  specified or scheduled. It is mentioned only as the intended solution for
  existing wallets. Legacy-dual wallets are refused by the modern `Manager`; the
  deprecated `Loader` keeps opening them unchanged, so nothing needs converting
  for this work to land.
- **No voluntary deprecated-file changes; forced repair only.** No new code, no bug
  fixes, no refactors in `wallet/deprecated.go` or `wallet/deprecated_*_test.go`;
  the known F2 defects there are permanent WONTFIX. `interface.go`'s aggregate
  `Interface` ends byte-identical. **Refined** (owner, 2026-07-30): minimal
  change to `deprecated.go` is acceptable where the compiler forces it —
  unnecessary change is not — a break **we** introduced is ours to repair, a defect
  that predates us stays. **No fixed cap**: measure the final delta and classify
  every changed line in 314 §9's necessity table as compile-forced, forced
  shared-code repair, or gofmt-only, with zero improvements. `gpt-5.6` 13 asked for byte-identity instead; the
  owner ruling stands. See [[Task 310]].
- **kvdb supports exactly one wallet** (owner, 2026-07-30). It is a legacy
  system; keep the legacy behavior — one wallet per kvdb database/file. ADR
  0001's multi-wallet commitment applies to the SQL schema only. Build no kvdb
  multi-wallet admission, enumeration, or `wallet_id` scoping beyond what the
  adapter already carries nominally. A second name/alias must be **rejected
  deterministically** — today's adapter accepts aliases silently, because
  `GetWallet` does not validate the name (`kvdb/walletstore.go:67`) and
  `ListWallets`/`IterWallets` are `notImplemented`. See 313 R9.
- **Revise-first** rather than merge-then-converge.

## Plan state

[[Task 314]] is at **revision 5**, reviews **closed** — `gpt-5.6` EXECUTABLE,
`claude-fable-5` Ready (terminal round), `codex` executable with its findings
applied. No design question is open and no further plan round is needed. The next
review entry should be post-execution validation against 314's DoD 1–10.

Execution order: §0 (the `fmt-check` blank line) → salvage reconciliation →
§1/§2 rebuild → §3–§9 → hygiene → threads → the three-metric before/after report
against pinned `2ea76dfc5`.

## Current branch state

`impl-secret-store` `2bddcc472` · `impl-runtime-store-6` `97548d076` ·
`wallet-default-sqlite` `4f0047d85` — local = origin = upstream, all signed.
CI **12/13** on each: only "Format, compilation and lint check" fails, on one
blank line in `kvdb/vault_test.go`'s import group (314 §0).

Unresolved review threads: **23 / 8 / 4**. #1272 has 111 threads and #1291 has
127, so a single `reviewThreads(first:100)` query reports #1291 as **zero** —
paginate and reconcile `totalCount` against unique IDs.

Salvage: the kvdb legacy-refusal reorder is **saved** at
`.lgtm/pivot/salvage/kvdb-legacy-refusal-reorder.patch` (57 lines, reviewed
correct, **unbuilt and untested**), taken from `.worktrees/t308` @ `a8b6db532`.
Keep that worktree until the patch is applied and tested. Store/container work is on `t313/r8-wip` `16c9b0aa4`.

Backups: `refs/backup/t313-green/*`, `refs/backup/t308/*` (pre-rewrite
originals).
