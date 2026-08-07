# Task 304 — Single-passphrase canonical vault, kvdb keeps dual

Status: **DONE**, executed with Task 303 in one cascade and pushed.
Heads `7bf664f83` (#1272) / `17638808c` (#1291) / `d98e82bab` (#1292), all
13/13 CI green.

C1-C7 all landed. One plan correction found during execution: **C4's target was
#27, not #17** — the `isKVDB` branch it comments does not exist until #27
introduces it, so the comment and the invariant test live there. C6's failure
mode was confirmed by falsification to be the nil-pointer panic the review
described, not a hang.

Heads at design time: `7df7ca70c` (#1272) / `f15eb8fe3` (#1291) /
`1f4bf7b52` (#1292), all 13/13 green. **#1300 (`GustavoStingelin`) is based on
`wallet-default-sqlite`**, so any rewrite forces a rebase on someone else's
open PR — coordinate, and do 303 + 304 together so it happens once.

## Decision

The canonical `keyvault.Vault` API is **single-passphrase**. The legacy
waddrmgr dual-passphrase scheme stays behind the legacy `Wallet` APIs and the
`LegacyManagerVault` adapter, which is **temporary format compatibility**, not
a second encryption model.

Vault selection is by backend — `isKVDB`:

```
kvdb → LegacyManagerVault (legacy waddrmgr format, dual passphrase)
SQL  → WalletVault        (canonical format, single passphrase)
```

In principle a database backend and an encryption format are independent
choices, and kvdb could store the canonical envelope. **This plan does not
take that route.** kvdb keeps the legacy dual-passphrase format, and existing
kvdb wallets are moved forward by a **kvdb → SQLite migration** rather than by
converting their envelope in place. Under that plan kvdb and legacy format are
equivalent by construction, so `isKVDB` is a correct discriminator rather than
a proxy for something finer.

The practical consequences:

- No canonical-format kvdb wallet ever exists, so `kvdb.Store` never needs to
  serve a wallet whose secrets sit outside waddrmgr's envelope.
- kvdb passphrase changes stay dual, on the deprecated path, permanently — no
  new-API parity is owed to kvdb wallets.
- `LegacyManagerVault` is removed when kvdb support is retired after
  migration, not when a format check replaces the backend check.

## Ordering

1. Finish the keyvault and route the wallet through it — the current stack
   (#1272 routes signer and lifecycle through `keyvault.Vault`).
2. SQL: single passphrase. kvdb: dual passphrase via the deprecated path.
   Select the implementation by backend. This is Task 304's C1-C7.
3. Once the new wallet is done, build the **kvdb → SQLite migration**. Only
   after no kvdb wallet remains in service does `LegacyManagerVault` go away.

Migration is out of scope here and gets its own task; nothing in C1-C7 should
pre-build for it.

## Invariant For This Phase

**Every kvdb wallet created during this phase must use the legacy waddrmgr
encryption format.** Do not create a canonical single-passphrase kvdb wallet
while the vault is selected solely through `isKVDB` — that would make the
check ambiguous, since `isKVDB` would no longer imply legacy format, and a
wallet could be handed the wrong vault. If a canonical-format kvdb wallet ever
becomes desirable, the discriminator must move to the persisted format
*first*.

This is why the create path staying on `recoverLegacyMasterRoot` for kvdb
(`manager.go:1841`) is correct here rather than something to fix.

**The invariant needs a test, not just a comment.** Add a focused `Manager`
create-and-load test asserting that a kvdb-configured wallet receives
`LegacyManagerVault` and resolves its master root through
`recoverLegacyMasterRoot`, never through canonical `GetWalletSecrets`. Comments
and broad CI cannot catch a future change that quietly creates a
canonical-format kvdb wallet; this test can.

## Interaction With Task 303

C1/C2 edit #12's params and tests; Task 303's V4 also touches #12's genesis
tests. Resolving them in separate passes would conflict, which is the main
reason to execute 303 and 304 together.

**Locked-rotation facts (verified).** `#6` introduces **no** locked-rotation
assertion, so there is nothing to remove from it:

- `TestWalletVaultChangePassphraseLockedRequiresUnlockedState` is **base code**
  (`sql-wallet:wallet/internal/keyvault/vault_lifecycle_test.go:241`).
- #6 `3e1317c84` adds only `vault_genesis.go` and `vault_genesis_test.go`.
- #12 therefore changes *pre-existing base behavior* — a deliberate behavior
  change, not a reversal of something this PR introduced.

(An earlier draft of both tasks claimed otherwise; that claim is retracted.)

That behavior change is intended and consistent with C1: rotation must work
while locked, which is why `OldPassphrase` is retained. It needs a clear
justification in #12's body, but it is **not** a hygiene violation and nothing
should be deleted from #6 on its account.

## Why: The Legacy Key Model

waddrmgr has two independent passphrase-derived master keys
(`waddrmgr/manager.go:374-403`):

```
public passphrase  → masterKeyPub  → cryptoKeyPub     (public data)
private passphrase → masterKeyPriv → cryptoKeyPriv    (HD root, privkeys)
                                   → cryptoKeyScript  (scripts)
```

`lock()` zeroes the private side but deliberately keeps `cryptoKeyPub`
(`manager.go:484`), so *which key a script is encrypted under decides whether
it is readable while locked*. That is the entire reason
`db.AddressSecret.ScriptIsSecret` is persisted and must be read rather than
inferred: it names the key, and guessing fails the ciphertext's auth tag.

A store-backed wallet has no second key hierarchy. Its script key is, as in
waddrmgr, a randomly generated crypto key *wrapped* by a master key derived
from the passphrase — the passphrase does not derive the script key itself.
What differs is that there is only one such hierarchy, so every operation
refuses while locked.

## Why: The Canonical Format Is Already Single-Passphrase

The base branch's own vault interface took one passphrase
(`sql-wallet:wallet/internal/keyvault/keyvault.go:51`):

```go
ChangePassphrase(ctx context.Context, newPassphrase []byte) error
```

Commit #12 `7cf158ab5 keyvault: complete passphrase lifecycle` widened it to
the two-passphrase `ChangePassphraseParams` struct
(`wallet/internal/keyvault/keyvault.go:25-49`). So narrowing is **folding back
our own widening**, which keeps it inside #1272's scope rather than making it
a base change.

`WalletVault` uses the same primitives as waddrmgr — `snacl.SecretKey`,
scrypt params, `snacl.CryptoKey` — and `db.WalletSecrets`
(`wallet/internal/db/data_types.go:379-394`) mirrors waddrmgr's private side:

| `db.WalletSecrets` | waddrmgr |
| --- | --- |
| `MasterPrivParams` (salt, scrypt N/R/P) | `masterKeyPriv` |
| `EncryptedCryptoPrivKey` | `cryptoKeyPrivEncrypted` |
| `EncryptedCryptoScriptKey` | `cryptoKeyScriptEncrypted` |
| `EncryptedMasterHdPrivKey` | encrypted HD root |

The format is not what forces two implementations.

## Why: Two Vaults Exist Because waddrmgr Stays Live

**As it stands today, before C3**, `LegacyManagerVault` is pure delegation —
`Unlock`, `Lock`, `IsLocked`, `Encrypt`, `Decrypt` and `ChangePassphrase` all
forward to `v.mgr.*` (`wallet/internal/db/kvdb/vault.go:34-144`). C3 changes
only the last of these, to return `ErrPassphraseChangeUnsupported`; the other
five keep delegating, and that is the point. Delegation is forced because
waddrmgr owns lock state and other code drives it directly. If `WalletVault`
independently unlocked the same wallet:

- new Controller operations would see the vault unlocked while legacy
  waddrmgr operations still saw `m.locked == true`;
- locking either component would not clear the other's secrets;
- a passphrase change through one would leave the other's cached master keys,
  encrypted crypto-key blobs, and `hashedPrivPassphrase` stale.

`LegacyManagerVault` is best understood as an **anti-corruption adapter around
the legacy crypto owner**, correctly located at the vault boundary. Crypto
must not move into `kvdb.Store`: `db.Store` is a persistence interface with no
key material and no lifecycle, and `Unlock`/`Encrypt` would make it a crypto
engine.

## Scale Of The Legacy Coupling (measured, not estimated)

`deprecated.go` has 97 `w.addrStore.*` calls, but most are not secret
operations:

| Count | Method |
| --- | --- |
| 22 | `FetchScopedKeyManager` |
| 16 | `SyncedTo` |
| 11 | `Address` |
| 8 | `AddrAccount` |
| 6 | `SetSyncedTo` |

The directly lifecycle-bearing subset is **7 calls**: 3 `ChangePassphrase`,
2 `IsLocked`, 1 `Lock`, 1 `Unlock`. There are also indirect secret paths via
`ManagedAddress.PrivKey`, `Script`, and scoped managers.

`rpc/` contains **no** direct `addrStore.` references; it reaches waddrmgr
transitively through legacy `Wallet` APIs.

So a future migration need not replace all 97 sites — changing only
secret-bearing paths, or injecting the canonical vault into waddrmgr's
cryptographic operations, are both viable.

## Why *Existing kvdb* lnd Is Unaffected

Scope note: this section establishes only that lnd keeps working for
legacy-format wallets. Migrated SQL wallets are a separate, unsolved problem —
see "lnd And Migrated SQL Wallets" below.

lnd changes passphrases through the **concrete** `*wallet.Wallet`, not through
any interface:

```
walletunlocker.ChangePassword (lnd walletunlocker/service.go:750)
  → loader.OpenExistingWallet(publicPw, false)   → *wallet.Wallet
  → w.ChangePassphrases(publicPw, new, privatePw, new)   (service.go:824)
     → wallet/deprecated.go:2860 → w.changePassphrases chan
        → walletLocker() (deprecated.go:2728)
           → w.addrStore.ChangePassphrase(ns, …) x2, one walletdb.Update
              (pre-C5 shape; C5 replaces this with the atomic call)
```

`Loader.OpenExistingWallet` returns `*Wallet` (`deprecated.go:7753`) and calls
`StartDeprecated()` (`:7810`), which starts `walletLocker()` (`:1553`). The
whole path is self-contained and kvdb-only. It is the single lnd call site —
`grep` across lnd finds only `walletunlocker/service.go:824`.

Consequences:

- **Do not rename** `Wallet.ChangePassphrases`. It is absent from
  `wallet.Interface`, so a `…Deprecated` suffix looks free when reading
  `interface.go` but breaks lnd at compile time.
- lnd's `BtcWallet` holds `base.Interface`
  (`lnd/lnwallet/btcwallet/btcwallet.go:88`) and therefore *cannot* change a
  passphrase at all. Passphrase changes were never part of that contract.

## Why "pub == priv" Cannot Be Assumed

lnd's main path does create wallets with one value for both
(`lnd/config_builder.go:1507`: `CreateNewWallet(password, password, …)`), but
three real paths differ:

| Path | public | private |
| --- | --- | --- |
| lnd `--noseedbackup` (`btcwallet.go:146`) | `"public"` | `cfg.PrivatePass` |
| lnd blank current pw (`service.go:777`) | `"public"` | `"hello"` |
| btcwallet daemon (`walletsetup.go:184`) | `"public"` | user-supplied |

And the failure is not graceful: `loadManager` derives `masterKeyPub` from the
supplied public passphrase and returns `ErrWrongPassphrase`
(`waddrmgr/manager.go:2199`). Guess wrong and the wallet does not open.

This is also why rotating **only** the private half on a legacy wallet is
unsafe: pub would stay at the old value, and lnd's next unlock passes the new
password as the public passphrase. On legacy format it is both-or-neither,
which is why rotation stays on the legacy API.

## Changes — Fold Into #1272

C1-C4 and C7 all fold into existing commits — C1/C2 into #12, C3 into #13, and
C4 plus C7 into #17. C5 and C6 touch pre-existing legacy code and land in the
widened #10; see the note at the end of this section.

### C1 — narrow the canonical params (#12 `7cf158ab5`)

Narrow `keyvault.ChangePassphraseParams` to exactly:

```go
type ChangePassphraseParams struct {
	OldPassphrase []byte
	NewPassphrase []byte
}
```

The old passphrase is retained deliberately: a locked `WalletVault` holds no
decrypted keys, so rotation must re-derive the old master key from the persisted
`MasterPrivParams` and unwrap the existing crypto keys before re-wrapping them.
That is the direct reason, independent of any caller.

(Current lnd does *not* call the canonical API — it uses the legacy path — so it
is not the justification. It becomes the consumer once SQL lnd integration
lands; see "lnd And Migrated SQL Wallets" below.)

**The Controller adaptation belongs to #17, not #12 — see C7.** The type and
`WalletVault` narrow in #12; the Controller does not construct these params
until #17 `742cd735d wallet: route lifecycle through vault`. At #12's tree the
handler still calls `w.DBPutPassphrase` (`controller.go:790` at that commit), so
editing the Controller here would introduce code ahead of its owning change and
break standalone verification of #12.

C7 records the Controller work. For reference, the construction it must adapt
(present only from #17 onward) is `wallet/controller.go:1279-1285`:

```go
params := keyvault.ChangePassphraseParams{
	ChangePublic:  req.req.ChangePublic,
	PublicOld:     req.req.PublicOld,
	PublicNew:     req.req.PublicNew,
	ChangePrivate: req.req.ChangePrivate,
	PrivateOld:    req.req.PrivateOld,
	PrivateNew:    req.req.PrivateNew,
}
```

### C7 — adapt the Controller handler (#17 `742cd735d`)

`handleChangePassphraseReq` gains the vault call in #17, so that is where it
must be written against the narrowed params. It must:

- reject `ChangePublic` with `keyvault.ErrPassphraseChangeUnsupported`, not
  ignore it;
- map `PrivateOld`/`PrivateNew` onto `OldPassphrase`/`NewPassphrase`;
- define the `ChangePrivate == false` case explicitly. `WalletVault` currently
  returns nil early ("nothing to rotate"); that no-op semantics moves to the
  handler once the flag leaves the params;
- update the Controller tests and the comments that describe two passphrases.

Narrowing `Controller.ChangePassphraseRequest` itself stays deferred (it is base
code), but this handler adaptation is required — without it #17 does not
compile against the narrowed type.

### C2 — delete the now-unreachable branch (#12 `7cf158ab5`)

Remove `WalletVault`'s `ChangePublic` rejection and
`errPublicPassphraseUnsupported`. Both become dead once the field is gone.

### C3 — kvdb vault refuses the canonical call (#13 `2abd0107b`)

`LegacyManagerVault.ChangePassphrase` returns an explicit unsupported error
instead of mapping onto waddrmgr.

Define a sentinel — `keyvault.ErrPassphraseChangeUnsupported` — and wrap it
with a message naming `Wallet.ChangePassphrases`. Callers must be able to
classify it with `errors.Is` rather than matching error text, and the
Controller's rejection of `ChangePublic` (see C1) should use the same sentinel.
Require `errors.Is` assertions in the tests for both the vault and the
Controller path. Keeping
the method (rather than removing it from `keyvault.Vault`) keeps the interface
uniform, so no call site needs a type assertion.

`ChangePassphrase` **stays on the `Vault` interface**; only the kvdb
implementation refuses it. (Lifting rotation out of the interface was
considered and rejected — it would force type assertions at the call sites and
the Definition Of Done assumes the sentinel-returning method.) The requirement
this satisfies: the new Controller returns unsupported for a kvdb passphrase
change rather than exposing dual-passphrase parameters, and legacy callers keep
using the deprecated methods directly.

### C4 — document the selection (#17 `742cd735d`)

Document the `isKVDB` branch in `buildWallet` (`manager.go:1463-1470`) as
temporary, using this wording:

```go
// KVDB wallets use the legacy waddrmgr encryption format until the
// KVDB-to-SQL wallet migration is available.
```

The branch is intentional for this phase, and it disappears with kvdb support
after the migration — a later reader must not mistake it for an accident.

### C5 — retarget the legacy handler (#10 `40e55d573`)

Point `deprecated.go:2728` at the atomic `waddrmgr.ChangePassphrases` instead
of calling the single-passphrase form twice. Requires adding
`ChangePassphrases` to the `AddrStore` interface — `waddrmgr/interface.go:139`
carries only the single form — and updating `bwtest/mock/addr_store.go`.

### C6 — fail instead of panicking or hanging (#10, widened)

Guard `Wallet.ChangePassphrases` (`deprecated.go:2860`). There are **two**
distinct failure modes, and the earlier draft of this task described only the
second:

1. **Panic.** `Wallet` embeds `*walletDeprecated` as a pointer
   (`wallet/wallet.go:343`), populated only by the legacy constructor
   (`deprecated.go:7231`). A wallet built by the modern `Manager` leaves it
   nil, so `w.changePassphrases` dereferences a nil embedded pointer and
   panics.
2. **Block.** A legacy wallet whose channels exist but whose `walletLocker()`
   is not running sends on an unserviced channel and blocks forever.

The guard must cover nil `walletDeprecated`, never-started, and stopped, and it
must be race-safe: a pre-send boolean check is insufficient on its own, because
shutdown can win between the check and the send. Select on the quit channel
alongside the send.

Return a **classifiable sentinel**, not a bare string — e.g.
`ErrLegacyWalletUnavailable`, for "this wallet has no running legacy runtime".
Callers must be able to distinguish it from a rotation failure with
`errors.Is`.

Tests are required for all four cases: modern wallet with nil
`walletDeprecated`; legacy wallet never started; stopped legacy wallet; and
concurrent shutdown racing the send.

C6 lands in the same widened #10 as C5, since that commit already owns the
legacy handler change. This is no longer optional or alternatively homed.

### Resolved — commit #10 `40e55d573` is retained and retargeted

C3 removes `kvdb/vault.go:144`, the **only** production caller of
`waddrmgr.Manager.ChangePassphrases`. Commit #10 adds **496 lines** to
`waddrmgr/manager.go` (246 of them non-comment code), and commit #11
`c8dda325a waddrmgr: expose stored secrets` carries its tests —
`manager_secret_test.go` calls `ChangePassphrases` at eight sites.

**Decision: keep #10 and retarget the legacy handler at it, folded into #10
itself.** #10's subject widens to cover both halves, e.g.
`waddrmgr+wallet: rotate passphrases atomically`. Rationale:

- No uncalled API exists at any commit, so the hygiene rule is satisfied
  without a separate base-fixes commit.
- The deletion path is more expensive than it looks: dropping #10 also
  requires removing or restructuring #11's eight `ChangePassphrases` tests.
- It fixes a real bug. The old `ChangePassphrase` swaps `masterKeyPub`
  (`manager.go:1157`) and `masterKeyPriv` (`:1130`) **inline**, so when the
  private call fails, `walletdb.Update` rolls back the database but not
  memory — persisted and cached state diverge until restart. The atomic version
  stages writes and swaps on `OnCommit`.

This supersedes the earlier ambiguity between "retarget in #10" and "land as a
separate legacy fix": it is **one** commit, #10.

### On extending `AddrStore` for C5

`waddrmgr/interface.go:38+` already documents that `AddrStore` is a leaky,
oversized interface that should be refactored, so adding `ChangePassphrases`
to it makes an acknowledged problem worse.

It is still the chosen route, because a narrow legacy-passphrase subset
interface in the `wallet` package was ruled out previously for this codebase
(subset interfaces over `AddrStore` were rejected in favour of using
`waddrmgr.AddrStore` directly). Record that reasoning in #10's body so the
tradeoff is visible: extending the interface is the minimal compatibility
choice given that constraint, not an oversight.

## lnd And Migrated SQL Wallets — Not A Blocking Gap

An earlier draft of this task called SQL lnd password changes a blocking gap.
Verification retracted that: lnd cannot reach a migrated SQL wallet through the
legacy entry point at all, so it must adopt the new `Manager`/`Controller` API
to use one — and `Controller.ChangePassphrase` already serves SQL wallets with a
single passphrase.

- `wallet.Loader` has no DB-backend awareness whatsoever.
- `Loader.OpenExistingWallet` opens the bolt database directly
  (`walletdb.Open` on `WalletDBName`, `deprecated.go:7770`).
- The legacy constructor hardwires `kvdb.NewStore` and
  `kvdb.NewLegacyManagerVault` (`deprecated.go:7223-7229`), pinning kvdb
  explicitly because an unset backend now defaults to sqlite.

So "once migrated, lnd uses the new wallet" is forced by construction, not an
assumption — and passphrase rotation arrives with that adoption rather than
needing its own mechanism. A façade over `Wallet.ChangePassphrases` would sit on
an entry point that cannot open the wallet, so it is not worth building.

Two migration-time notes, recorded here so they are not lost — neither affects
this stack:

- **No new interface is needed for rotation.** `Controller` already declares
  `ChangePassphrase(ctx, req)` (`wallet/controller.go:133`), so lnd can use it
  directly. `wallet.Interface` declaring no passphrase method is only why lnd
  currently reaches for the concrete `*wallet.Wallet`.
- **The migration keeps the private passphrase.** It is moot for lnd's main
  path, where pub == priv already, but `--noseedbackup` wallets and btcwallet's
  own daemon differ; the private one protects the key material the canonical
  envelope carries, and the public one is discarded.

## Deferred — Not This Stack

- **kvdb → SQLite migration** (own task, after the new wallet is done). It
  must record that lnd has to adopt the new `Manager`/`Controller` API to use a
  migrated wallet at all, plus the two migration-time notes above. This is what
  retires the legacy format. Agreed outline:
  1. Open and unlock the legacy wallet with its public *and* private
     passphrases.
  2. Extract the decrypted canonical wallet material.
  3. Create SQLite wallet secrets under the single selected passphrase.
  4. Copy accounts, addresses, scripts, transactions, and sync state.
  5. Validate the migrated wallet before cutting over.
  6. Retain the original kvdb database as a rollback backup until verification
     succeeds.

  Once migration support is complete and the legacy runtime is retired, remove:
  `LegacyManagerVault`; the `isKVDB` vault-selection branch; dual-passphrase
  support; the waddrmgr lock-state dependency; and the stubbed secret methods
  at `kvdb/walletstore.go:522,529`.
  - Note for whoever writes it: new kvdb wallets are created in the legacy
    envelope today (`recoverPersistedMasterRoot` → `recoverLegacyMasterRoot`
    at `manager.go:1841`), which is correct under this plan — the migration
    moves wallets to SQLite rather than converting kvdb envelopes in place.
  - Also note `kvdb.Store` is itself an adapter over a live waddrmgr: it holds
    `s.addrStore` and routes through `FetchScopedKeyManager` and
    `waddrmgr.NamespaceKey` (`kvdb/addressstore.go:59-78`). The migration reads
    through that adapter rather than needing a native kvdb secret path.
- **Narrowing `Controller.ChangePassphraseRequest`** (`controller.go:106-118`)
  — base code, needs a bottom base-fixes commit. Until then the controller
  should reject `ChangePublic` explicitly rather than ignore it silently.
- **Add the legacy method to `wallet.Interface`** marked `Deprecated:`, without
  renaming, so the contract stops depending on the concrete type.
- **Consolidate the four duplicated backend gates** (`manager.go:1466/1470`,
  `:1841`, `deprecated.go:7229`) into one resolved-once vault.

## Do Not Do

- **Do not add `MasterPubParams` / `EncryptedCryptoPubKey` to canonical
  `WalletSecrets`.** That imports waddrmgr's dual-key scheme into the new model
  permanently, and it is only required if the canonical vault must operate
  directly on the legacy envelope without migration.
- **Do not assume pub == priv** for legacy wallets (see the table above).
- **Do not move crypto into `kvdb.Store`.**
- **Do not treat `UpdateWalletSecrets` for legacy format as mechanical.** A
  generic store could write the legacy buckets, but waddrmgr caches master
  keys, encrypted crypto-key blobs, `privPassphraseSalt`,
  `hashedPrivPassphrase`, and lock state; writing underneath a live manager
  leaves all of it stale. `GetWalletSecrets` is mostly serialization;
  `UpdateWalletSecrets` is not.

## Verification

Per rewritten commit: `GOWORK=off go vet ./...` plus an itest-tagged vet
(`go build` misses `_test.go`), and `%G? = G`.

At each of the three PR heads: four golangci passes (default, `./itest`,
sqlite DB-itest, postgres DB-itest) **pinned to SHAs, not branch names** — a
branch-name run can race a fixup commit and lint the wrong tree; an
80-column audit at **tab width 8** (`lll` runs at width 4 and cannot see that
class); `git diff --check`; `make fmt-check`; `make sqlc-check`.

Branch level: `GOWORK=off go test ./wallet/...` plus `-race` on the
concurrency tests. C5 requires a regression test that a failed private rotation
leaves neither persisted nor cached public state changed — unconditional, since
C5 is decided.

## Definition Of Done

1. `keyvault.ChangePassphraseParams` carries only `OldPassphrase` and
   `NewPassphrase`; no public passphrase field survives in the canonical API.
2. `WalletVault` has no public-passphrase branch; `LegacyManagerVault`
   returns a clear unsupported error naming the legacy API.
3. **Existing kvdb lnd still works**, verified reproducibly rather than
   assumed. lnd's `go.mod` pins a released btcwallet, so a plain `go build`
   proves nothing about this stack. Use a **temporary external `go.work`**
   listing both modules — not an ad hoc `replace` in either `go.mod`, which the
   repo's guidance discourages and which is easy to commit by accident. Build
   lnd, then run both password-change tests:

   ```
   go test ./walletunlocker \
     -run 'TestChangeWalletPassword(NewRootKey|Stateless)$'
   ```

   Both matter to C5, because they exercise different passphrase shapes:
   `NewRootKey` the normal equal public/private case, `Stateless` the unequal
   `"public"`/`"hello"` default path. Delete the workspace afterward and record
   the exact commands.
   `ChangePassphrases` must keep its name and its behavior for legacy-format
   wallets.
   This is *not* the same as SQL lnd password changes, which no item here
   delivers — those arrive with lnd's adoption of the new API. Do not read a
   green stack as evidence about migrated wallets.
4. `waddrmgr.Manager.ChangePassphrases` has a production caller, and the
   divergence bug has a regression test proving a failed private rotation
   leaves neither persisted nor cached public state changed.
5. C6's guard returns `ErrLegacyWalletUnavailable`, with `errors.Is` tests for
   all four cases: nil `walletDeprecated`, never started, stopped, and
   concurrent shutdown racing the send.
6. The `isKVDB` selection carries the temporary-format comment verbatim, and no
   kvdb wallet is created in the canonical format, so `isKVDB` still implies
   legacy format everywhere it is used — proven by the invariant test below,
   not by inspection.
7. A focused `Manager` create-and-load test proves a kvdb wallet selects
   `LegacyManagerVault` and takes the legacy root-recovery path
   (`recoverLegacyMasterRoot`), never canonical `GetWalletSecrets`.
8. All three PR heads CI green with four clean lint passes; every commit
   compiles and passes vet standalone.
