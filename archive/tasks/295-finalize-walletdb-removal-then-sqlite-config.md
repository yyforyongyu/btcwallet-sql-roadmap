# Task 295: Finalize kvdb→Store migration (remove walletdb in the prep), then SQL config + 3-DB itests

Status: SUPERSEDED by tasks 296 + 297 (generic DBConfig landed in impl-runtime-store-6, not the final branch; Manager is now SQL-native per 297). Archived 2026-07-10. Original planning below. Reworks the task-294 branches (`impl-secret-store` #1272,
`impl-runtime-store-6`, `wallet-default-sqlite`). Task 294 used **Route B**
(continuation branches above the green lower stack) **+ option A** (config in the
prep). This task moves the generic backend config/SQL to the final branch and finishes
the walletdb removal.

## Goal
1. **Prep `impl-runtime-store-6`** — finish the kvdb adapters needed to
   remove non-deprecated top-level direct `walletdb` **DB API** usage
   (`walletdb.View`/`.Update`/`.DB`/`.ReadTx`/`.ErrDb…`) from `wallet/`. (`wallet/log.go`'s
   `walletdb/migration` import is not a DB API use and stays.) Keep a kvdb-only
   config/open shape; no generic backend selection, no SQL config. (kvdb `UpdateWalletSecrets` stays stubbed; DBVault/SQL secret rotation is deferred — see the #1272 section.)
2. **`impl-secret-store` (#1272)** — migrate the passphrase change (secret material) off
   the `db_ops.go` shim into a kvdb-only seam (option b; NOT the `keyvault.Vault` interface).
3. **Final `wallet-default-sqlite`** — new generic `DBConfig`/`DBBackend` + selection +
   sqlite default + itests on kvdb / sqlite / postgres.

## Part 1 — Prep: remove direct walletdb DB API from top-level wallet/

| file | residual walletdb | action |
|------|-------------------|--------|
| `syncer.go` | `walletdb.View` in `lookupStoreAccountNumber` (~2274) | move resolution into the kvdb adapter — see "syncer" (covers BOTH scan paths) |
| `interface.go` | `Database() walletdb.DB` + legacy types | **rename → `interface_deprecated.go`**, never touch |
| `notifications.go` | `walletdb.ReadTx` helpers | **rename → `notifications_deprecated.go`**, never touch (lnd-facing) |
| `manager.go` | `walletdb.ErrDbDoesNotExist` (from opening the kvdb file pre-Store) | surface a kvdb/open-store sentinel to wallet code — NOT a generic Store error (minor) |

`signer.go` is already clean; `wallet.go:374` is a comment only (reword/remove it).

### Config: prep stays kvdb-only (was Medium)
The prep does NOT remove config — impl-runtime-store-6 already uses `Config.DB`, `cfg.DB.KVDB`,
and `openLegacyStore` to open kvdb. **Naming choice:** the prep uses a kvdb-only **`KVDBConfig`**
(no `Backend`/SQLite/Postgres fields); the **final** introduces the generic **`DBConfig`**
(`Backend DBBackend` + `KVDB` + `SQLite` + `Postgres`) wrapping it and adding selection. So the
prep never carries `DBBackend`; the final's flip changes `Config.DB` from the kvdb-only shape to
the generic `DBConfig`.

### syncer — non-masked recovery number (RESOLVED: kvdb-only, ADR 0013)
`lookupStoreAccountNumber` (the `walletdb.View`, syncer.go:2274/2282) is the imported-xpub
fallback inside `storeAccountRecoveryNumber` (2248), which serves BOTH:
- the **targeted** path (a `fallbackNumber` arrives from `resolveScanTargets`), and
- the **untargeted / full-scan** path (no fallback → `lookupStoreAccountNumber`, via
  syncer.go:2225 → 2263).

So moving the read only into `kvdb.ResolveScanTargets` is INCOMPLETE — untargeted
imported-xpub recovery still has no non-masked source. Move the resolution into a **kvdb
adapter method that BOTH paths call** (kvdb runs the `walletdb.View` internally; syncer.go
calls the Store, no direct walletdb).
**RESOLVED -> kvdb-only** (per ADR 0013, docs/developer/adr/0013-normalized-account-address-identity.md). **Premise correction:** SQL imported-xpub accounts have `account_number` **NULL** (ADR 0013 requires it), so `AccountInfo.AccountNumber == nil` is NOT masking a hidden SQL number — there is simply no SQL number to return. The concrete number `RecoveryState` needs is a legacy **waddrmgr derivation artifact**, so kvdb/waddrmgr owns it. SQL scan persistence already resolves horizons by **`AccountID`**; the numeric account is only needed while `RecoveryState` derives/lookahead through the legacy address manager.

**Shape:** add a kvdb adapter method (e.g. `ResolveRecoveryAccountNumber`) on the legacy store; the syncer calls it **through `s.legacyStore` — including when the runtime store is SQL**, because RecoveryState derivation is legacy-waddrmgr for BOTH backends (the legacy address manager is retained for derivation until SQL-native recovery lands). It serves both the targeted and untargeted scan paths. **Do NOT** add a shared `db.Store` method unless SQL later stores or derives a real recovery number independently. SQL's own `account_id`-based recovery horizon is a future follow-up (deferred with the SQL work; `AccountInfo.AccountID` already exposed).

### Done-when (real DB API calls only, deprecated files excluded)
`git grep -nE 'walletdb\.[A-Z]' -- wallet/ | grep -vE '/internal/|deprecated\.go|_test\.go'`
is empty. This checks direct walletdb **DB API** usage (not literal `walletdb` — so it ignores
`wallet/log.go`'s `walletdb/migration` import and prose like `wallet.go:374`). `deprecated\.go`
excludes `deprecated.go` AND the renamed `interface_deprecated.go` / `notifications_deprecated.go`;
`wallet/internal/...` is the kvdb backend (keeps walletdb).

## impl-secret-store (#1272) — DBPutPassphrase migration (DECIDED: option b — kvdb-only seam)
`DBPutPassphrase` (db_ops.go:201) is secret material → in #1272, move it out of top-level
`wallet/db_ops.go` into kvdb/keyvault-adjacent code, reusing the existing atomic `walletdb.Update`
+ `waddrmgr.ChangePassphrase` behavior. **Do NOT** add `ChangePassphrase` to the `keyvault.Vault`
interface, and **do NOT** implement or touch `DBVault`.

**Seam (option b).** Since the method is NOT on `keyvault.Vault`, the controller cannot call it via
`w.keyVault`. Add a **separate narrow seam injected into the Wallet**:
- a small wallet-defined interface (e.g. `passphraseChanger` with one `ChangePassphrase` method) and
  a new `Wallet` field, injected at construction (manager.go:348 + deprecated.go:7217) alongside
  `keyVault`;
- implemented by kvdb-adjacent code (a concrete method on `LegacyManagerVault`, or a small sibling
  type in `wallet/internal/db/kvdb/`) running the atomic `walletdb.Update` +
  `waddrmgr.ChangePassphrase` moved from `db_ops.go:201`;
- **params must avoid the wallet→kvdb import cycle**: use primitives or a kvdb-level params type,
  NOT `wallet.ChangePassphraseRequest` (a kvdb impl referencing it would import `wallet`);
- `handleChangePassphraseReq` (controller.go:772) calls the seam instead of `DBPutPassphrase`
  (:782); keep the `RefreshPrivatePassphrase(PrivateNew)` call at :792 (no-op in kvdb) as-is.

**Why not the alternatives:** NOT `db.Store.UpdateWalletSecrets` (blob-persist, not rotation, and
kvdb stubs it); NOT the `keyvault.Vault` interface (would force a `DBVault` stub + imply SQL
semantics now).

**DEFERRED follow-up (see deferred-tasks.md):** DBVault stays the never-wired placeholder (all
`TODO(gus)` stubs); implementing DBVault + SQL-backed passphrase rotation is a separate task, NOT
295.

**Tests:** add a kvdb-package test for the new passphrase seam/adapter (atomic
`waddrmgr.ChangePassphrase` behavior), and update `controller_test.go`'s ChangePassphrase tests to
expect the injected seam (instead of `addrStore.ChangePassphrase` via `DBPutPassphrase`); move or
delete `db_ops_test.go:167 TestDBPutPassphrase` with the method. Also update the test wallet builder `createTestWalletWithMocks` + `mockWalletDeps` (common_test.go) to inject the new seam field (mirroring the existing `keyVault: mockVault` injection).

**Do NOT delete db_ops.go in #1272** — it still holds `DBCreateWallet` (:35, called by
manager.go:241), `DBLoadWallet`, and many syncer `DB*` methods. #1272 only moves `DBPutPassphrase`
out; db_ops.go is deleted in the **prep**, where routing has stripped it to just `DBPutPassphrase`.

## Part 2 — Final (wallet-default-sqlite)
- New generic `DBConfig`/`DBBackend` (kvdb / sqlite / postgres) + open/select wrapper + sqlite
  default (the old task 282). This wraps the prep's kvdb-only `KVDBConfig`.
- itests across all three backends.
- a68 SQL `signer_test.go` (already split here in task 294).

### Create-correctness split — RESOLVED (per-commit / per-hunk)
Rule: kvdb-applicable -> prep; SQL-runtime-row-specific -> final; per-HUNK for mixed commits. (All
sit above the config-driven wrapper `createRuntimeWallet`, so any fix kept in the prep is
re-expressed against the prep's kvdb-direct create path.)

**Clean -> prep:**
- `065dd1c69` close Stop/Start lifecycle race.
- `aeb76571c` prevalidate duplicate initial accounts.

**Clean -> final:**
- `3607729e8` tx_inputs schema/queries; `8e648ffc9` detect confirmed conflicts; `5b7bcee0d` conflict itests.
- `a28a17614` default-account seeding idempotency — **final, NOT prep**: the helper is for spendable
  SQL wallets; kvdb already seeds default accounts in `waddrmgr.Create`.

**Split per-hunk:**
- `833271f8c` reject Create on existing wallet: kvdb existing-wallet rejection -> prep; SQL
  `runtimeExisted` pieces -> final.
- `f7f3d5ff7`: partial-wallet cleanup on post-Load failure -> prep; SQL birthday safety margin -> final.
- `00a1eb98d`: make Stop terminal -> prep; recover SQL master seed on create-retry -> final.
- `efcd06f9b` (4-way): generic manager concurrency/cache correctness -> prep; initial-account
  replay/idempotency -> prep; SQL retry consistency (seed-mismatch / root-key verify,
  `recoverLegacyMasterRoot`, SQL runtime birthday reuse) -> final; SQL default-account
  count/replay/idempotency -> final.

**Also (per the plan, not create-correctness):** the generic config + wrapper + SQL support
(`8e4637f17` runtime store config, `38df5d312` select/open wrapper, `84d8007af` support SQL
backends) move to the **final**; the kvdb adapters + routing (bottom 8) stay in the **prep**;
imported-account identity/schema (`77437a875`, `3dcb418bf`) are prep candidates — confirm per-hunk
at impl.

## Findings (verified 2026-07-01)
- `notifications.go`: deprecated-driven + lnd-facing (`lnwallet/btcwallet/btcwallet.go`
  `SubscribeTransactions` → `wallet.NtfnServer.TransactionNotifications()`). Keep; rename.
- `db_ops.go`: @ #1272 has 26 funcs (DBCreateWallet/DBLoadWallet/DBPutPassphrase + syncer DB*);
  @ prep only `DBPutPassphrase`. `LegacyManagerVault` (vault.go:13) holds walletdb.DB +
  waddrmgr.Manager. `kvdb.UpdateWalletSecrets` is a stub (walletstore.go:460).
- `keyvault.Vault` implementers: `LegacyManagerVault` (kvdb, wired) + `DBVault` (SQL, never wired, all `TODO(gus)` stubs). 295 uses a kvdb-only seam and does NOT touch either the interface or DBVault.
- `wallet/log.go:12` imports `walletdb/migration` — not a DB API use; excluded by the check.
- `syncer.go`: both scan paths need the non-masked number.

## Sequencing
`#1272` sits below the prep, so its passphrase migration will re-cascade
`impl-runtime-store-6` + `wallet-default-sqlite` (new SHAs) — expected, bounded.

## Cross-ref
- Task 294 (archived): the Route-B + option-A reorg that produced the current branches.
- Task 282 (archived): sqlite-default flip — folded into Part 2.
- ADR 0013 (docs/developer/adr/0013-normalized-account-address-identity.md): SQL keys account/address identity — incl. imported-xpub scan horizons — by immutable `account_id`; kvdb keeps legacy waddrmgr numbers. Resolves DECIDE #1 (syncer) = kvdb-only.


## Progress (2026-07-01)

- **Phase 1 — #1272 passphrase seam (option b): DONE + verified.** Scratch branch
  `wip-295-secret` = `f37afd447` (off `impl-secret-store` 6a008f64b, GPG-signed).
  - `keyvault.ChangePassphraseParams` (cycle-free param type).
  - `kvdb.LegacyManagerVault.ChangePassphrase` — concrete method, **off** `keyvault.Vault`;
    reuses the atomic `walletdb.Update` + `waddrmgr.ChangePassphrase`.
  - wallet `passphraseChanger` seam interface + Wallet field; same LegacyManagerVault
    injected (manager.go + deprecated.go); `handleChangePassphraseReq` calls the seam;
    `RefreshPrivatePassphrase` kept. `DBPutPassphrase` removed from db_ops.go (file kept —
    still has DBCreateWallet/DBLoadWallet/etc.).
  - `walletmock.PassphraseChanger` + kvdb `vault_test.go` (2 tests) + controller-test +
    common_test.go rewire. `go build ./...` OK; `./wallet` + kvdb + keyvault tests green.
  - Real branches UNTOUCHED; not pushed.

- **Phases 2-4 — PENDING** (large multi-branch restructure; each re-cascades upward):
  - Phase 2 (prep): rename interface.go/notifications.go -> `*_deprecated.go`; manager
    kvdb/open-store sentinel; syncer `ResolveRecoveryAccountNumber` via `s.legacyStore`;
    `KVDBConfig` restructure (generic DBConfig -> final); delete the now-empty db_ops.go.
  - Phase 3: per-hunk create-correctness split (4 mixed commits; SQL halves -> final).
  - Phase 4 (final): generic DBConfig/DBBackend + selection + sqlite default + moved
    commits + 3-DB itests (kvdb/sqlite/postgres).


## Progress — Stage 1 (2026-07-01, cont.)

- **Phase 1 seam re-based cleanly**: amended to leave DBPutPassphrase dead in #1272 (removed in
  prep) so the prep rebase onto the seam has no db_ops.go conflict. `wip-295-secret` = 6589b30b9.
- **`wip-295-prep`** = the 23-commit prep rebased onto the seam + Stage-1 commits, all GPG-signed,
  `./wallet` + kvdb + keyvault green:
  - `36cac864a` delete db_ops.go (the migrated DBPutPassphrase shim).
  - `9e7e3deb4` rename interface.go/notifications.go -> `*_deprecated.go`.
  - `1a08fefab` kvdb `ResolveRecoveryAccountNumber` + syncer rewired (drops syncer walletdb.View).
  - `1153672a2` kvdb `ErrStoreNotExist` open-store sentinel (drops manager walletdb ref).
  - **Done-when MET**: `git grep 'walletdb\.[A-Z]' wallet/` non-internal/non-deprecated/non-test is EMPTY.

- **REMAINING (the structural core + final):**
  - Phase 2-config: KVDBConfig restructure — the prep still carries the generic
    DBConfig/DBBackend/SQLite/Postgres + the switch-based runtime_store.go wrapper + SQL support
    (rebased in from the old prep). Move those to the final; prep keeps kvdb-only.
  - Phase 3: create-correctness per-hunk split (SQL halves -> final).
  - Phase 4 (final): generic config + sqlite flip + 3-DB itests (kvdb/sqlite; postgres where infra).


## Cleanup note (2026-07-03)
The `wip-295-secret` + `wip-295-prep` scratch branches holding the verified Phase-1 (passphrase seam) + Stage-1 (walletdb removal) work were **deleted** in the repo-wide branch cleanup (kept only the 12-branch lineage). That implementation is unrecoverable (unreachable, will be GC'd) — the plan above stands; re-implement from it if resumed. Also: fix-import-schema + prep-runtime-1 + prep-runtime-2 are now merged into sql-wallet.
