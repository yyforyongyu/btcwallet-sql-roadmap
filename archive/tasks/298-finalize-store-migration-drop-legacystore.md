# Task 298 — Finalize store migration: complete kvdb `Store`/`Vault`, then dissolve `w.legacyStore` + delete `db_ops.go`

Revised per GPT review (`.lgtm/tasks/gpt-review-task.md`, verified against code).
Round-2 re-review folded the passphrase-rotation **Blocker** into 1c. Round-3
re-review declared the task-plan clean ("No blocking task-plan issue remains");
its only leftover — precise `GetAddressSecretQuery` selector validation — is
folded into 1b. Two user decisions this round are applied: syncer identity keys
by the raw store `AccountID`, not the BIP44 number (2c), and
`wallet-default-sqlite` keeps the sqlite default (see triage §⚖️). The
review-comment triage below is verified against code at the three branch heads
(2026-07-13). Round-4 folded the imported-account signer selector into 1a/2a.
Round-5 caught that the kvdb-contained snapshot/restore was not implementable (the
`waddrmgr.Manager` key state it needs is unexported) — the passphrase decision was
re-made to a combined `waddrmgr.Manager.ChangePassphrases` (1c); Round-6 tightened
that method to defer the in-memory swap to `ns.Tx().OnCommit` (atomic even if the
commit itself fails). All forks resolved — see **Decisions locked** below.

End-state: wallet runtime code uses `w.store` for persistence and `w.keyVault`
for auth/encryption/passphrase; the kvdb backend hides all `walletdb`/`waddrmgr`/
`wtxmgr` detail inside `wallet/internal/db/kvdb`; no `w.legacyStore` field; no
`wallet/db_ops.go`. `deprecated.go` stays kvdb (out of scope, WONTFIX).

Branch heads (draft PRs, CI-green): `impl-secret-store`=`fbc57ec50` (#1272),
`impl-runtime-store-6`=`b23830bf2` (#1291), `wallet-default-sqlite`=`1f88969ae`
(#1292).

## Decisions locked (ready for one-go implementation)
1. **Scope:** fix ALL 15 standalone review bugs in this stack. Lifecycle/concurrency
   fixes get their OWN commits (controller shutdown 496/503/505/416, `manager.go:966`
   concurrent-Load, `syncer.go:426` rescan-tip); the rest fold into their owning commit.
2. **Passphrase (1c):** add a combined `waddrmgr.Manager.ChangePassphrases` (validate
   all old → stage all → write all rows → swap memory on commit via
   `ns.Tx().OnCommit`); atomic even if `Commit()` fails. Additive `waddrmgr` public
   API — the kvdb-only snapshot was rejected (Round-5: `Manager` key state is
   unexported). `LegacyManagerVault.ChangePassphrase` maps params → one call.
3. **Script secrets (2b/1b):** included now — reroute `ScriptForOutput` +
   `ManagedScriptAddress` secrets through `GetAddressSecret`+`Decrypt(CKTScript)`, with
   P2SH/P2WSH/taproot tests.
4. **Imported-account identity:** `GetAccountSecret` gains an `AccountID` selector;
   signer resolves via `AddressInfo.AccountID` (1a/2a). Syncer keys recovery state by
   raw `AccountID`, never `AccountNumber` (2c).
5. **SQLite default kept** in `wallet-default-sqlite`; migration/gating + release-note
   are a pre-ship follow-up (out of stack).
6. **Tier-2 option-picks (defaults):** tx_reader → `wallet.TxStatus` alias+constants;
   `manager.go:806` → reject bare-xpub `ModeImportExtKey`; `syncer.go:426` →
   monotonic-height guard on `updateSyncTip`; `wallet.go:509` → surface error / keep
   last-valid tip; `vault_genesis.go` → merge into `vault.go`(+`vault_test.go`).

## CORRECTED premise (my earlier "pure reroute" was wrong)
This is NOT reroute-then-delete. Three interface methods the reroute depends on
are **stubs**, so deleting `legacyStore` first breaks KVDB:
- `kvdb.Store.GetAccountSecret` → returns `db.ErrAccountSecretUnavailable`
  (`accountstore_getaccountsecret.go:19`) — KVDB derived signing currently only
  works via `legacyStore.ResolveDerivedPrivKey`.
- `kvdb.Store.GetAddressSecret` → `notImplemented` — imported private-key signing
  currently only works via `waddrmgr.ManagedPubKeyAddress.PrivKey()`.
- `LegacyManagerVault.ChangePassphrase` → `errLegacyVaultChangePassphrase`
  (`vault.go:112`) — KVDB passphrase change currently only works via
  `DBPutPassphrase`.
Also `LegacyManagerVault.Unlock` (`vault.go:35`) calls `waddrmgr.Unlock`
unconditionally, violating the `Vault.Unlock` contract (must return
`ErrVaultUnlocked` without validating when already unlocked).

## Ordered shape: IMPLEMENT → REROUTE → TEARDOWN

### Phase 1 — complete kvdb `Store` + `Vault` (mostly `impl-secret-store`)
**1a. `kvdb.Store.GetAccountSecret` real adapter (Finding 1).**
**Add `AccountID *uint32` to `GetAccountSecretQuery`** (Round-4): require exactly
one selector among `AccountID`/`Name`/`AccountNumber`; prefer `AccountID` (SQL →
`accounts.id`; kvdb → the durable walletdb account number) so imported accounts
with NULL `account_number` are addressable. Resolve scoped manager from
`query.Scope`; resolve the account by that selector; read persisted encrypted
account priv key (do NOT
re-derive on read — must be stable and work while locked); return
`db.AccountSecret{PublicKey, AccountName, AccountNumber, MasterKeyFingerprint,
EncryptedPrivateKey}`. Watch-only/imported-xpub → public key, nil
`EncryptedPrivateKey` (not `ErrAccountSecretUnavailable`). Absent → `ErrAccountNotFound`.
waddrmgr support (prefer explicit method over widening `AccountProperties`):
```go
AccountSecret(ns walletdb.ReadBucket, account uint32) (
    pubKey *hdkeychain.ExtendedKey, encryptedPrivKey []byte,
    fingerprint uint32, name string, imported bool, err error)
```

**1b. `db.Store.GetAddressSecret` query shape + kvdb/SQL impls (Finding 4).**
KVDB address IDs are synthetic → current `AddressID`-only query is insufficient.
New shape (require exactly one selector):
```go
type GetAddressSecretQuery struct {
    WalletID     uint32
    AddressID    *uint32
    ScriptPubKey []byte
}
```
Selector validation (Round-3 review): `hasAddressID := AddressID != nil;
hasScriptPubKey := len(ScriptPubKey) > 0; if hasAddressID == hasScriptPubKey {
return db.ErrInvalidQuery }` — reject BOTH selectors and NEITHER (incl. nil/empty
`ScriptPubKey`). Tests: id-only, spk-only, neither, both, empty-spk.
- SQL: `AddressID` → existing query; `ScriptPubKey` → join `addresses`↔
  `address_secrets` by wallet+scriptpubkey; `ErrAddressNotFound` /
  `ErrSecretNotFound`.
- KVDB: `addressFromScript` → `addrMgr.Address(ns, addr)`; `ManagedPubKeyAddress`
  → encrypted priv key; `ManagedScriptAddress` → encrypted script; address exists
  but no private/script secret → `db.ErrSecretNotFound` (no successful empty-secret
  result). waddrmgr support: a kvdb-facing `AddressSecret`/`ManagedAddressSecret`
  adapter (avoid adding `EncryptedPrivKey()` to the public `ManagedPubKeyAddress`
  interface).
- Regenerate sqlc + mocks; SQL + generated in their own commit.

**1c. `keyvault.Vault.ChangePassphrase` → request object + impls (Finding 2).**
Change interface from private-only to:
```go
type ChangePassphraseParams struct {
    ChangePublic bool; PublicOld, PublicNew []byte
    ChangePrivate bool; PrivateOld, PrivateNew []byte
}
ChangePassphrase(ctx, params ChangePassphraseParams) error
```
- `LegacyManagerVault` (**Blocker** — do NOT copy the two-call pattern): calling
  `waddrmgr.Manager.ChangePassphrase` once for public then once for private in one
  `walletdb.Update` can leave manager runtime state diverged from a rolled-back DB
  — it mutates in-memory master-key state after each DB write, so if the second
  call fails the DB rolls back but memory does not. A kvdb-only snapshot/restore is
  impossible: the state (`masterKeyPub`, `masterKeyPriv`, crypto keys, salts,
  hashed-passphrase) is unexported on `waddrmgr.Manager` and unreachable from
  `package kvdb`. **DECISION: add a combined `waddrmgr.Manager.ChangePassphrases(ns,
  ...)` method** that (1) validates ALL requested old passphrases first, (2) stages
  all new master keys + encrypted crypto-key blobs in locals, (3) writes all DB rows
  in the caller's bucket, (4) registers the in-memory `Manager` swap via
  `ns.Tx().OnCommit(...)` so it runs ONLY after the enclosing `walletdb.Update`
  actually commits — a callback error OR a `Commit()` error → `OnCommit` never fires
  → memory stays unchanged (Round-6: swapping after the callback returns but before
  commit still diverges if the commit fails; `ns.Tx().OnCommit` mirrors the existing
  `scoped_manager.go:1414` pattern, so no signature change is needed). Additive/
  backward-compatible `waddrmgr` public API (lnd unaffected); land it as its own
  `waddrmgr:` commit ahead of the adapter. `LegacyManagerVault.ChangePassphrase` then just maps
  `ChangePassphraseParams` → one `ChangePassphrases` call inside a `walletdb.Update`
  (no duplicated `DBPutPassphrase` behavior).
- `WalletVault`: reject `ChangePublic` (typed unsupported error) unless SQL public
  passphrase added; `!ChangePrivate` → nil; fetch `GetWalletSecrets`, decrypt with
  `PrivateOld` to validate, rotate `MasterPrivParams`/`EncryptedCryptoPrivKey`/
  `EncryptedCryptoScriptKey` with `PrivateNew`; leave `EncryptedMasterHdPrivKey`
  (encrypted by crypto key, not passphrase); **preserve original lock state inside
  the vault** (no controller lock/unlock gymnastics — supersedes my earlier F2/F6
  controller-sync fix).

**1d. `LegacyManagerVault.Unlock` contract fix (Finding 3).**
`if !v.mgr.IsLocked() { return keyvault.ErrVaultUnlocked }` before the
`walletdb.View`/`v.mgr.Unlock`.

### Phase 2 — reroute wallet (signer/syncer/controller)
**2a. Derived-key signing → store+keyvault (Finding 5).** Build path from
`BIP32Path`/`AddressInfo` → `w.store.GetAccountSecret` → `w.keyVault.Decrypt(
CKTPrivate, ...)` → parse account xprv, derive branch/index → leaf priv key.
Watch-only → `ErrWatchOnlyAccount`; absent → `ErrAccountNotInStore`. Removes the
`waddrmgr`-cache + `legacyStore.ResolveDerivedPrivKey` branches. **Query by
`AddressInfo.AccountID`** (already present on the address row) for derived AND
imported-xpub children — not `AccountNumber` (NULL for imported); fall back to
`AccountNumber` only where a call site genuinely has only a BIP44 path (this is
what actually fixes signer.go:1006).

**2b. Imported-key signing → `GetAddressSecret`+`Decrypt` (Findings 4,6).**
`GetPrivKeyForAddress` / `ComputeUnlockingScript`: script pubkey →
`w.store.GetAddress` → derivation-metadata? derive via account secret : fetch
`w.store.GetAddressSecret` (by script pubkey) + decrypt; imported w/o priv key
(`db.ErrSecretNotFound`) → `ErrNoAssocPrivateKey`. Do NOT fall back to
`PrivKeyForAddress`/`ManagedPubKeyAddress.PrivKey()`. `PrivKeyForAddress`
(no-ctx) becomes a thin wrapper over the store-based helper (no `walletdb` tx).

**DECISION: include script-secret migration now.** Reroute `ScriptForOutput` (and
any `ManagedScriptAddress.Script()` path) through the store: fetch `EncryptedScript`
with `w.store.GetAddressSecret`, decrypt with `w.keyVault.Decrypt(CKTScript, ...)`,
decode taproot scripts with `waddrmgr.DecodeTaprootScript`, and add P2SH, P2WSH, and
taproot-script tests. (So §1b's KVDB `ManagedScriptAddress` → encrypted-script branch
is in scope, not optional.)

**2c. Syncer identity via raw store `AccountID` (Finding 7; DECISION).** Today
recovery/scan state is keyed by a bare `uint32` that MIXES two identity spaces —
derived accounts contribute their BIP44 `AccountNumber`, imported accounts a
synthetic store id — so a store `AccountID` equal to a BIP44 number in the same
scope silently overwrites another account's deriver/name/horizons
(syncer.go:2330, verified REAL). **Decision: use the raw store `AccountID` as the
single recovery identity for ALL accounts; never mix with / fall back to
`AccountNumber`** (that stays a derivation-only concept). Concretely:
- SQL → `AccountID` = `accounts.id`; KVDB → set `AccountID` to its unique walletdb
  account number in `kvdb.loadAccountInfo` (+ update the `db.AccountInfo.AccountID`
  doc to state it is the durable per-account store identity).
- Key EVERY recovery-state map (`accountNames`, `accountIDs`, `accountDerivers`,
  `branchStates`, horizons) by `AccountID` only.
- `resolveScanTarget`: imported → passthrough; else resolve the incoming target to
  its `AccountID` via the store (`GetAccount`/`ListAccounts`) at the boundary, then
  key state by that `AccountID`. Do NOT key by or fall back to the target's
  `AccountNumber`.
- Replace `storeAccountRecoveryNumber` + delete `lookupStoreAccountNumber`; removes
  `s.legacyStore.ResolveScanTargets` + `LookupAccountNumber`.

**2d. Passphrase → controller calls `w.keyVault.ChangePassphrase(params)`** only.

### Phase 3 — teardown
- Delete `w.legacyStore`/`s.legacyStore` field (construction keeps a LOCAL
  `StoreHandle` for bootstrap).
- Delete kvdb escape-hatch methods + exported package-func twins:
  `DeriveManagedAddress`, `LoadManagedAddress`, `ResolveDerivedPrivKey`,
  `(*Store).LookupAccountNumber`, `(*Store).ResolveScanTargets`.
- **Unexport `LoadStore` → `loadStore`** (Finding 8 — it IS used by
  `CreateStore`/`OpenStore`; NOT dead). Keep `CreateStore`/`OpenStore` exported
  (wallet bootstrap calls them). Doc `StoreHandle` + `closeFn` (Finding: driver.go docs).
- Delete `wallet/db_ops.go` (only after 1c/2d land — Finding 9); relocate
  `db_ops_test.go` cases to keyvault/kvdb/controller tests; move
  `TestCreateLegacyWallet`/`TestLoadLegacyWallet` under kvdb if they test kvdb
  lifecycle.

## Branch split (Finding 10 — interface/query/mocks touch multiple layers)
- `impl-secret-store`: keyvault contract fixes (1c,1d), `LegacyManagerVault`
  passphrase, `kvdb.GetAccountSecret` (1a), `GetAddressSecret` query+kvdb/SQL
  (1b), signer derived + imported reroute (2a,2b), controller passphrase (2d),
  KVDB+SQL signing/passphrase tests. (All the `db.Store`/kvdb/SQL/mock/signer
  changes live here, below irs6.)
- `impl-runtime-store-6`: syncer identity/`AccountID` (2c), remove
  `s.legacyStore`/`w.legacyStore`, delete kvdb escape-hatch helpers, unexport
  `loadStore`, delete `db_ops.go`.
- `wallet-default-sqlite`: default flip only; rebased after the lower branches.
  **Decision: KEEP sqlite as the default here** (the flip is this branch's whole
  purpose). The unset-`Backend` data-abandonment objection (runtime_store_config.go:166)
  is accepted for this stack; existing-caller kvdb→sqlite migration is a separate
  follow-up, not an in-stack blocker (triage §⚖️).

## Test plan
- kvdb `GetAccountSecret`: encrypted priv material for derived account; nil for
  watch-only/imported-xpub.
- kvdb + SQL derived signing through `GetAccountSecret`+`Decrypt` with
  `legacyStore` nil/removed.
- imported-key: `GetPrivKeyForAddress`/`ComputeUnlockingScript` use
  `GetAddressSecret`, not `ManagedPubKeyAddress.PrivKey`.
- imported account with NULL SQL `account_number`: signing resolves via
  `GetAccountSecret` by `AccountID` (or name), not a BIP44-number selector.
- `LegacyManagerVault.ChangePassphrase` (via `waddrmgr.Manager.ChangePassphrases`):
  public-only / private-only / both / wrong-old / lock-state preserved / mid-op
  failure AND post-callback commit failure (commit-failing walletdb hook) both leave
  BOTH the DB and in-memory master-key state unchanged (`masterKeyPub`/`masterKeyPriv`,
  crypto keys, salts, hashed passphrase).
- `WalletVault.ChangePassphrase`: locked & unlocked rotation, wrong-old, public
  rejection, re-unlock with new passphrase.
- syncer: imported-xpub target resolved without `legacyStore` via `AccountID`; plus
  a collision test where a store `AccountID` equals a different BIP44 `AccountNumber`
  in the same scope (must not overwrite the other account's recovery state).
- targeted pkg tests: `wallet`, `wallet/internal/keyvault`,
  `wallet/internal/db/kvdb`; affected signer/controller/syncer tests; `-race`;
  lint `--new-from-rev`.

## Review comments — full triage (verified against code at branch heads 2026-07-13)
Each site read at `impl-secret-store`=fbc57ec50 (#1272) /
`impl-runtime-store-6`=b23830bf2 (#1291) / `wallet-default-sqlite`=1f88969ae
(#1292). Legend: ✅ fixed by Phases 1–3 · 🔴 standalone bug to fold into the
owning commit · 📝 commit hygiene · ⚖️ decision applied · ⚪ stale/no-op.

### ✅ Subsumed by Phases 1–3 (resolve the thread when the phase lands)
- #1272 signer.go:914 & signer.go:567 — "put this in `keyvault`?" → store/keyvault
  reroute (2a/2b).
- #1272 signer.go:1006 — imported account rows have NULL `account_number`, so the
  by-number-only lookup can't find imported secrets and returns `ErrAccountNotInStore`
  before deriving children → §1a `AccountID` selector on `GetAccountSecretQuery` + §2a
  signer resolves via `AddressInfo.AccountID`. (NOT §2c — that is syncer-only.)
- #1272 controller.go:781 — `ChangePrivate` only rotates waddrmgr via `DBPutPassphrase`,
  so a Store-backed `WalletVault` keeps the old credential while reporting success →
  §1c/2d.
- #1272 vault.go:49 — `waddrmgr.Unlock` can lock an already-unlocked manager on a wrong
  passphrase, violating the `Vault.Unlock` contract → §1d.
- #1272 signer_test.go:1032 & signer.go:1040 — the Store-secret success path (vault
  `Decrypt` → child derive) is untested (only negative branches covered) → test plan.
- #1291 kvdb/signer.go:59 & driver.go:153 — `DeriveManagedAddress`/`LoadManagedAddress`/
  `ResolveDerivedPrivKey`/`LoadStore` exported → deleted (Phase 3) / unexported (§Finding 8).
- #1291 driver.go:69 — doc `StoreHandle`/`closeFn` → Phase-3 driver.go pass.
- #1291 db_ops.go:950 — deletion in its own commit → Phase 3.
- #1291 signer.go:1292 — "use the `Store` interface, and why in irs6 not impl-secret-store?"
  → §2b + moved into impl-secret-store (Finding 10).
- #1291 signer.go:542 — "use the new `w.store`?" → §2a.
- #1291 db_ops.go:40 — on a wrong `PrivateOld`, `changeStorePassphrase` returns after
  `Lock()`+failed `Unlock` before the `wasLocked` restore, leaving a previously-unlocked
  store wallet locked (controller then forces `toLocked()`), diverging from the kvdb path
  → §1c deletes this path; `WalletVault.ChangePassphrase` preserves lock state internally.

### 🔴 Standalone verified bugs — ALL in scope this stack (DECISION)
Fix all in-stack. **Lifecycle/concurrency fixes get their OWN commits** (controller
shutdown 496/503/505/416, `manager.go:966` concurrent-Load, `syncer.go:426`
rescan-tip); the rest fold into their owning commit.
Wallet create/import (`wallet/manager.go`, #1291):
- manager.go:938 [REAL] — `initialAccountAlreadyImported` compares only `info.PublicKey`;
  a retry keeping the xpub but changing `MasterKeyFingerprint`/`AddrType` is accepted with
  stale metadata. Compare every persisted account field.
- manager.go:806 [PARTIAL] — `ModeImportExtKey` + a bare xpub (watch-only) yields a
  zero-account shell that `CreateAccount` can't populate (spendable-xpub is already rejected
  up front). Reject the mode for a bare xpub, or import the xpub as an account.
- manager.go:966 [REAL] — `Load`→`awaitCreate` never enters `creating`, so two concurrent
  `Load`s each `buildWallet`+`publish` (last writer wins; self-documented in `onStopped`).
  Guard build+publish per wallet name.
- manager.go:732 [REAL] — `createLegacyStore`'s `encryptedSeed` is discarded (`_`) yet still
  computed via `legacyEncryptedSeed` on the kvdb-only fresh-create branch (a real
  `GetEncryptedHDSeed` whose error aborts create — dead work). Drop the seed return + the two
  `legacyEncryptedSeed` reads.
Controller shutdown (`wallet/controller.go` + `state.go`, #1291) — 3 live in `finishStop`:
- controller.go:505 [REAL] — `finishStop` logs+discards the runtime/legacy store-close error;
  `Stop` still returns nil and `onStopped` evicts the wallet. Propagate through stop completion.
- controller.go:503 [REAL] — `finishStop` never `keyVault.Lock()`s nor closes `addrStore`, so
  an unlocked terminal wallet retains decrypted key material. Clear both key managers before
  closing stores.
- controller.go:496 [REAL] — `Stopped` is published (`toStopped`) before store close + cache
  eviction, so `waitForStopped`/`Load` can observe a done shutdown and hand back the
  still-closing instance. Transition only after teardown completes.
- controller.go:416 [REAL] — `requestStop` returns false/false when already `Stopping`, so a
  2nd `Stop` returns before terminal, breaking the blocking contract. Wait for `Stopped`.
Syncer (`wallet/syncer.go`, #1291):
- syncer.go:2655 [REAL] — a targeted rescan only scopes horizons; `storeScanAddresses` and
  `storeScanUnspent` still seed RecoveryState wallet-wide, so it can match+persist unrelated
  account activity. Thread `resolved` targets through those loaders (or filter off-target).
- syncer.go:426 [REAL] — `Rescan` queues block-connected notifications that `advanceChainSync`
  doesn't consume, and `updateSyncTip` writes `SyncedTo` with no monotonic-height guard → a
  stale replay regresses the tip. Add a registration-only watch, or guard `updateSyncTip`.
SQL correctness (#1291):
- wallet.go:509 [REAL] — SQL `SyncedTo`/`storeSyncedTo` maps `GetWallet`/block-decode errors
  to height −1 reported as success (addrStore==nil for SQL). Preserve last-valid tip or
  surface the error.
- runtime_store.go:306 [REAL] — a spendable SQL wallet reaches `CreateWalletSecrets` with an
  empty private passphrase that legacy `waddrmgr.Create` rejects (`ErrEmptyPassphrase`). Guard
  a non-empty priv passphrase before creating secrets (pairs with vault_genesis.go:66).
API surface (#1291):
- tx_reader.go:147 [REAL] — public `TxDetail.Status` is the internal type
  `wallet/internal/db.TxStatus`; external callers (lnd) can't name it or its `TxStatus*`
  constants. Re-export `wallet.TxStatus` + constants (or surface the status as a string).
Keyvault genesis (`wallet/internal/keyvault/vault_genesis.go`, #1272):
- vault_genesis.go:66 [REAL] — `CreateWalletSecrets(watchOnly=false)` only guards
  `hdRootKey==nil`; a neutered xpub is persisted as master-priv (spendable-but-broken, can't
  derive hardened children). Reject `!hdRootKey.IsPrivate()`.
itest harness (`bwtest/harness_wallet.go`, #1292):
- harness_wallet.go:47 [REAL] — `db=postgres` supplies only `KVDB.DBPath`, so `Manager.Create`
  fails on missing `DB.Postgres.DSN`. Wire a Postgres DSN/config or reject the db value up front.
- harness_wallet.go:45 [REAL] — itest flag + `make itest` default `dbType=kvdb`; given the
  sqlite-default decision, make itest exercise sqlite by default and require explicit `db=kvdb`.

### ⚖️ Decisions applied this round
- **Syncer identity — raw `AccountID`, no mixing (syncer.go:2330, #1291).** REAL collision
  confirmed (a store `AccountID` can equal a BIP44 number in the same scope, overwriting
  another account's deriver/horizons). Decision: key recovery state by the raw store
  `AccountID` for ALL accounts; never mix with / fall back to `AccountNumber`. Applied in §2c.
- **SQLite default kept (runtime_store_config.go:166, #1292).** REAL (unset-`Backend` callers,
  incl. lnd, get a fresh empty `wallet.sqlite` next to `wallet.db`, abandoning kvdb data with
  no migration). Decision: KEEP the sqlite default in `wallet-default-sqlite`; the objection is
  accepted for this stack. Pre-ship follow-up (out of this stack, per Round-4): an explicit
  migration/gating path AND a release-note callout for unset-`Backend` callers.

### 📝 Commit hygiene (reword/reshape pass; no behavior change)
- #1272 vault.go:1 — strip the `(cherry picked from …)` trailers stack-wide.
- #1272 vault.go:13 — rewrap commit bodies over 72 columns (2839c1ef, e9055d85, 3b7cdbf3,
  93fdada7, 3ad432b5, ec9460bb, ee5ea23d, fbc57ec5).
- #1272 controller.go:703 — commit bodies describe timeout/refresh/import APIs absent from
  their patches (2839c1ef, ec9460bb, fbc57ec5); rewrite bodies to match the actual changes.
- #1272 psbt_manager.go:1058 — fold the `fbc57ec5` review-labeled fix commit into originating
  commits (no standalone tip fix-commit).
- #1291 lifecycle.go:1 — de7afc2e/a363c3c5 have empty bodies; add ≤72-col bodies.
- #1291 waddrmgr/manager.go:51 — split `BirthdaySafetyMargin` into its own `waddrmgr:` commit.
- #1292 runtime_store_config.go:165 — 1f88969a has no body despite flipping the default; add one.
- #1292 signer_test.go:1060 — fold the SQL-only `DerivePubKey` fallback test into `fbc57ec5`
  (its originating commit in impl-secret-store), not the sqlite-default commit.
- #1272 vault_genesis.go:1 — file placement (merge into `vault.go`/`vault_test.go`?) — your
  call, low stakes.

### ⚪ Stale (resolve as already-addressed; no code action)
- #1291 manager.go:821 — over-create is already guarded: `beginCreate` rejects a cached wallet
  with `ErrWalletExists`, and `preCreated && created==0` rejects a durable-but-complete replay.

## Sequencing guard
Do NOT delete `w.legacyStore` / `db_ops.go` / kvdb helpers until Phase 1 methods
are implemented AND Phase 2 reroute lands green. Do NOT resolve GitHub threads
until each item is implemented & landed green.
