# Task 315 — rebuild the three PRs as the minimal dual-backend itest stack

> **CLOSED 2026-08-02 — archived.** Its requirement set was delivered: both
> co-goals met (zero `walletdb.Update`/`View` in the public wallet package, and
> the Manager itest running on kvdb and SQLite), every DoD 1–13 gate green, and
> the three PRs pushed at `c6425a509` / `df3a02daf` / `960dd02e8` with all 281
> review threads resolved and PR bodies rewritten to the delivered design.
>
> **What did not close with it:** the *commit boundaries*. Task 324 owns the
> re-split of that same delivered tree and the re-push that follows, so the
> heads named throughout this file are the input to 324, not the final ones.
> `opus-report.md` is the surviving rationale record; this file is evidence.

**Status:** implementation reduced by Task 316, validated per commit, and
pushed to `upstream` and `origin`, with every PR `headRefOid` matching.
Delivered OIDs:
#1272 `c6425a509a4365822249c53e14433dace46389f8`,
#1291 `df3a02daf0d865265afef582f9b56c4d094f8334`,
#1292 `960dd02e871987daeacf03b69b0eaef0b46b749b`. DoD 1-13 are met and the
review-thread sweep is complete at 0 unresolved, and all three PR bodies are
rewritten to the delivered design (§316-12.2 step 6). **Open:** green CI at
those exact OIDs. Real-backend address-driven PSBT signing is recorded as
**not delivered and deferred** (DoD 6, `opus-report.md` §5.5). Superseded head
sets — rev-5 `6eae9f519` / `61d2e60c1` / `4d672cbcf`, rev-6 `58d8d9754` /
`b6221ff78` / `70adc46fa`, and the two earlier 2026-08-02 push rounds, the last
being `e851275d3` / `583145370` / `95731a6bc` — are historical; that #1272 head
does not compile and must never be a rollback target. This file is the
**frozen governing requirement set** and the rev-6 input; it is **not** an
executable plan. [[316-reduce-task-315-to-minimal-scope.md]] is the sole active
plan, and §316-1 amends five requirements here: the `manager create wallet`
itest keeps its original create/start/sync shape with no added lifecycle
sequence (§5.5, DoD 6), real-backend address-driven PSBT signing is deferred
(DoD 6, see §316-7), the mutating `scripts/check-each-commit.sh` is replaced by
§316-10.1's non-mutating exact-OID loop (DoD 7), and PostgreSQL is compile
fallout only — no Manager, harness, daemon, behavioral matrix, or completion
gate of its own, while the existing shared DB contract keeps running on it for
CI parity (DoD 8) — and tests assert public behavior rather than private
structure (§1's standing style rule, applied to the Manager tests; §5.1, §5.5,
DoD 6). Everything in `.lgtm/tasks/archive/` is
evidence, not instruction — including
[[archive/314-post-review-completion-of-the-three-prs.md]] and its "AUTHORITATIVE
OWNER DECISIONS", which this task supersedes where §2 says so.

**Base pin:** `sql-wallet` = `2ea76dfc5e85acf84014a21e4813cb23945250fa`.

**Rev-6 input heads** (GPG-signed, *not* the delivered heads):
`impl-secret-store` **`58d8d9754`** (7 commits) · `impl-runtime-store-6`
**`b6221ff78`** (4) · `wallet-default-sqlite` **`70adc46fa`** (1), retained at
`refs/backup/t316-pre/*`. Task 316 rewrote all three in place; the delivered
OIDs are in the status header above and in `opus-report.md` §1. Rev 6
dispositions the three rev-5 reviews (see `opus-report.md` §7.1).

The pre-task heads were `6da45e3f8` / `65d2db689` / `672b9476d`; the first
(unfaithful) rebuild was `cdaae0eb6` / `bc0600698` / `8b44f2d63`. Rev 2 repairs
all 15 findings from gpt's implementation review of that first rebuild — see
`.lgtm/tasks/opus-report.md` §7 for the per-finding fix, the falsification behind
each, and the gate results. Two of gpt's asks were declined with evidence
(`validateInitialAccountsMode` and the watch-only-root check exist verbatim at
base, so removing them would regress rather than defer).

Both §1 deliverables are met: the SQLite itest runs, and the public wallet
package's production files hold **zero** `walletdb.Update/View` outside
`deprecated.go` (**23** at base, per §1's authoritative gate below:
`wallet/db_ops.go` 17, `wallet/signer.go` 4, `wallet/syncer.go` 2).

**Revision 5.** Each revision corrected claims the previous one asserted without
checking; every correction is marked ✗ and was verified in source.

Revision 5's root cause is different from the earlier ones and worth naming: I
adopted **reviewer design preferences over the owner's own recorded rulings**.
`NewKVDBManager`/`NewSQLiteManager` contradicted archived decision 1
(`NewManager(ctx, cfg)`) and decision 4 (`Backend` required) — see §5.1 ✗✗.
Check §2's surviving-rulings table before accepting any reviewer's API shape.

Also in revision 5: `Deprecated:` markers go on **every** kvdb-only public
field, `PubPassphrase` included (§2 — owner ruling, final),
keeping the base cipher API leaves SQL taproot import **broken** without a
`CKTPublic` mapping (§5.2), base's wallet-level `InitialAccounts` post-create
import stays (§5.3), rotation must keep working while locked (§5.4), and §5.1
gains the three contracts it deleted names for without replacing: the kvdb
transaction bridge, the SQLite address deriver, and in-process kvdb
single-wallet admission.

**Two owner rulings landed after the reviews** (2026-07-31): the exact
`ManagerConfig` shape in §5.1, and **one unified Config per component** —
Manager, Wallet, Store, keyvault — recorded as a standing rule in §1 and gated
by DoD 4b.

---

## 1. Goal

Two co-equal deliverables, stated by the owner:

1. **Remove `walletdb.Update`/`walletdb.View` from the new public Wallet
   components.** At base there are exactly **23** such call sites:
   `wallet/db_ops.go` 17, `wallet/signer.go` 4, `wallet/syncer.go` 2.
   Transactions stay legal in `wallet/deprecated.go` and under
   `wallet/internal/`. The gate, verified to return exactly those 23 at base:

   ```bash
   # Exit 1 means no match (pass). Anything above 1 is a git failure and must
   # NOT be mistaken for a clean head — revision 3 had that bug.
   check_tx_boundary() {
       local rev="$1" out status
       if out=$(git grep -n -E 'walletdb\.(Update|View)\(' "$rev" -- \
           ':(top,glob)wallet/*.go' \
           ':(exclude,top)wallet/deprecated.go' \
           ':(exclude,top,glob)wallet/*_test.go'); then
           status=0
       else
           status=$?
       fi
       [ "$status" -eq 1 ] && return 0
       [ "$status" -ne 0 ] && return "$status"
       printf '%s\n' "$out"; return 1
   }
   ```

   The `if` wrapper keeps `set -e` from killing the shell before `status` is
   captured.

   The `glob` magic is required: a bare `wallet/*.go` pathspec matches
   recursively and sweeps in `wallet/internal/db/kvdb`, where transactions
   belong. The check greps the literal package identifier, so also confirm by
   eye that no file import-aliases `walletdb`.

2. **Run the existing Manager itest on both backends:**

   ```text
   make itest chain=btcd db=kvdb   icase=manager
   make itest chain=btcd db=sqlite icase=manager
   ```

**Metric:** total diff from the pinned base — all-file `A+D`, production-Go
`A+D` (`*.go` excluding `_test.go`, generated `sqlc`, and `.pb.go`), and
changed-file count. Not per-commit size.

**Scope rule (owner):** *"we can always fix bugs in another PR."* A pre-existing
bug is not a prerequisite merely because a review noticed it. Only these belong
here: compile-forced changes, regressions **these PRs introduce**, and behaviour
the two commands above actually need.

**Style (owner):** smaller changes and concise comments. Comments state the
non-obvious *why* in a few lines — no essay godocs, no narration of the code.
Tests assert public behaviour, not private structure.

**One Config per component (owner, 2026-07-31).** Every component this stack
touches — Manager, Wallet, Store, keyvault — has **at most one** unified
`Config`-style struct. Never two, never a Config plus loose *configuration*
values, never a backend-specific variant of one component's Config, never a
handle/config wrapper. This is the rule the deleted `OpenStoreParams`,
`StoreHandle`, `testDBConfig`, tagged `DBConfig`, and per-call `Create`/`Load`
database settings each violated (§4). Before adding any configuration struct,
extend the component's existing one or explain why it cannot carry the field.

✗ "At most one", not "exactly one" — an earlier revision 5 draft said the
latter, which is unsatisfiable: `keyvault.NewWalletVault(store, walletID,
watchOnly)` and `NewLegacyWalletVault(db, addrManager)` take **required
dependencies**, not configuration, and have no Config at base. A component that
has no Config at the pinned base does not gain one to satisfy this rule —
wrapping those constructors would add diff and remove no second config source.

The rule governs **construction**. Operation arguments are a different kind and
keep the repo's existing idiom — `UnlockRequest`, `ChangePassphraseRequest`,
`GetAccountSecretQuery`, `GetAddressSecretQuery`, `CreateWalletParams` are
request values, not component configuration, and are not consolidated.
Collapsing requests into component Config would just build one large
optional-field bag.

Per component:

| component | its one config |
| --- | --- |
| Manager | flat `ManagerConfig` (§5.1) — no `KVDBManagerConfig`, `SQLiteManagerConfig`, future `PostgresManagerConfig`, or nested `DBConfig` |
| Wallet | the existing per-wallet `Config`; it gains no backend selection or physical database settings |
| Store backends | one **package-local** config each. `sqlite.Config` and `pg.Config` predate these PRs — do not rewrite them to look alike. The new kvdb adapter gets at most one `kvdb.Config`. Manager translates from `ManagerConfig`; add no public Store config or handle wrapper |
| keyvault | **none new** — its existing constructor arguments already form one unambiguous dependency set. Add no backend-specific Vault config variants |

**What the itest exercises** (verified — one case, `manager create wallet`):
create with `ModeGenSeed` → `Start` → mine 5 blocks → wait for sync → `Stop`. It
never unlocks, signs, derives an address, imports, reloads, or restarts.

---

## 2. What changed since Task 314

The owner re-scoped after 314 was executed. These 314 rulings are **reversed**:

| 314 ruling | now |
| --- | --- |
| Decision 2 — new kvdb wallets are single-encryption, discriminated by the reserved `mgrver` | **Reversed, and abandoned rather than deferred.** kvdb keeps the existing dual format permanently. A new format would mean two migrations (dual→single, single→SQL) for a backend being retired, and needed a new persisted marker because `mgrver` cannot serve as one. Task 311's doctrine becomes historical; reasoning in [[323-kvdb-to-sql-wallet-migration.md]]. |
| Single-passphrase collapse (`PubPassphrase` removed, `PrivatePassphrase` → `Passphrase`) | **Reversed.** Keep base's two fields while kvdb is supported; the collapse happens when kvdb is dropped. SQL uses the private half and rejects a requested public rotation. See ✗ below for where each field lives. |
| Atomic `Create` as the root fix | **Narrowed.** Base `sqlite.Store.CreateWallet` already writes wallet row + secrets + sync state in one `execWrite` (verified), and base kvdb create is already one `walletdb.Update`. Only **account** aggregation was new — deferred to [[317-atomic-wallet-and-account-creation.md]]. |
| ADR 0014 | Deleted with the format arc. |

These 314 rulings **survive**:

- **Manager owns one database** (decisions 1/3) — as a *physical handle*, not a
  bound Store; see §5.1 ✗. This ruling settles ownership; it is not reopened.
- **`Wallet.Stop` is restartable** (decision 1b) — base's shape, plus the one
  forced vault lock in §5.5 ✗.
- **No modern per-call backend selection** (decision 4) — removes
  `Backend`/`DBConfig` from `Create`/`Load`. ✗ Revision 3 said `Config.DB`
  **and** `Config.PubPassphrase` both survive for deprecated construction only.
  Wrong on the second: `Config` (`wallet/wallet.go:171`) is the *modern* config
  `Manager.Load` takes, and its `PubPassphrase` (`:204-205`) is what
  `waddrmgr.Open` needs to decrypt public metadata. Without it a Manager
  could not load a kvdb wallet created with a custom public passphrase. So:
  `Config.DB` is deprecated-only and ignored by the modern Manager;
  **`Config.PubPassphrase` is the modern kvdb Load credential**;
  `CreateWalletParams.PubPassphrase` (`wallet/manager.go:92`) is the kvdb
  Create credential; SQLite ignores both. Add no passphrase to the constructor
  configs and no `LoadParams` — the base fields are already the smallest
  carriers.

  ✗ **Never rename** — `PubPassphraseDeprecated` is 23 references of churn, and
  Go tooling keys on the marker, not the name.

  **Every kvdb-only public field carries a `Deprecated:` marker.** Owner ruling
  (2026-07-31), final — do not re-open, and do not re-argue it from the lint
  side-effect:

  | field | marker |
  | --- | --- |
  | `Config.DB` (compatibility-only; `ManagerConfig.DataSource` replaces it) | **yes** |
  | `ManagerConfig.NoFreelistSync`, `.Timeout` | **yes** |
  | `Config.PubPassphrase`, `CreateWalletParams.PubPassphrase`, `ChangePassphraseRequest.ChangePublic`/`PublicOld`/`PublicNew` | **yes** |

  ```go
  // PubPassphrase is the public passphrase used to open a kvdb wallet. SQL
  // backends ignore it.
  //
  // Deprecated: kvdb is a legacy backend. This field will be removed with kvdb
  // support.
  PubPassphrase []byte
  ```

  The public passphrase is not deprecated *cryptography* inside kvdb — it is
  deprecated **public Wallet API surface**, because only the retiring backend
  consumes it. A caller still needing it to drive kvdb is what the marker
  communicates, not a contradiction of it.

  **The markers cost nothing in lint — add no suppression.** Verified by
  running golangci-lint on a two-package fixture: SA1019 fires on a **selector
  expression** (`c.Old`) but **not** on a keyed composite-literal field
  (`dep.Config{Old: "x"}`). Every external use of these fields is a keyed
  literal — `PubPassphrase` at `bwtest/harness_wallet.go:53,59` and
  `itest/manager_test.go:26,32`, `Config.DB` at `bwtest:43`/`itest:20` (which
  #1291 removes anyway) — and `ChangePublic`/`PublicOld`/`PublicNew` have **no**
  external users at all. So there is no diagnostic to suppress.

  Add the godocs, run `make lint-check`, and add a suppression **only if that
  command produces a real diagnostic**. Do not touch `.golangci.yml`
  preemptively and do not add unused lint directives. If a diagnostic does
  appear, match the repo's existing shape — the narrow staticcheck path
  exclusion at `.golangci.yml:205-211` — never a tree-wide disable, and land it
  in the same commit as the markers.

  Do **not** mark `LegacyWalletVault`, `AddressSecret.ScriptIsSecret`,
  `InsecurePubPassphrase`, or the kvdb `Backend` value — kvdb is a supported
  backend this milestone, and those are domain names or active metadata.
- **Ruling 9 — deprecated files**: forced repair only, no voluntary change.
- **Ruling 13 — the name `LegacyWalletVault`**, now backed by the dual format.

---

## 3. Approach — rebuild, do not patch

All three reviewers recommend recreating #1272 and #1291 from the base rather
than patching. The reason is add-then-delete churn baked into the history:
publication coordination, terminal `Stop`, and partial-create replay were each
**added and then deleted** inside the stack, so a rebuild saves more than
deferral arithmetic suggests. The stack's own historical minimum was **22,176**
(at the insert-only checkpoint) against **22,469** at the current heads.

Most of the keep-list maps onto identifiable existing commits and can be folded
and simplified — but fold to the *end state*. Do not cherry-pick a commit that
introduces an API a later commit removes.

---

## 4. Delete / defer

Nothing below is written in the rebuild.

**Format arc** (deleted for good, not deferred — see §2) — `waddrmgr/format.go`
+ `format_test.go`, `CreateSingleEncryption`, `OpenSingleEncryption`, every
`singleEncryption` branch, the `mgrver` marker, `plaintextCodec`, ADR 0014,
legacy-format refusal, `ErrMigrationRequired`, single-format
`ConvertToWatchingOnly` changes, ADR-0012 canonical-kvdb enforcement, and
watch-only root validation.

**Creation** — `createWalletAggregateOps` (both dialects), `DefaultAccountName`,
`DefaultAccountScopes` and `InitialAccounts` **as fields of
`db.CreateWalletParams`**, the create-time derivation callback,
transaction-local temporary managers, create manifests, partial-create
recovery/replay/auth, the insert-only over-create contract, and the rollback
tests. Also the recovery family in `wallet/manager.go` (~264 lines, measure at
deletion): `recoverPersistedMasterRoot`, `recoverLegacyMasterRoot`,
`recoverRuntimeMasterRoot`, `decryptLegacyMasterRoot`,
`readExistingLegacyEncryptedSeed`, `verifyRootKeyMatchesLegacy`, and
`legacyExisted`.

**Lifecycle** — `startStopMu`, `beginStart`, `isInitialized`, `isStopped`,
`ErrWalletStopped`, stop claims/results/requests, `onStopped`,
`discardUnstarted`, cache eviction, runtime-store close from Wallet, Manager
publication coordination, `awaitCreate`, and the concurrent Start/Stop and
duplicate-Load tests.

**API and config** — `waitForAcceptedResp`, `OpenStoreParams`, `StoreHandle`,
`Manager.kvdbWallet`, `testDBConfig`, the three-backend `DBConfig`, per-call
database settings on `Create`/`Load`, the typed Vault cipher redesign (§7 Q4),
and the nil guards on mandatory Wallet dependencies (but see §5.6 — the
constructor fix comes first). Every one of these is an instance of §1's
one-Config rule being broken; audit the rebuild for any new struct that breaks
it the same way.

Also **`TxDetail.Status` and the wallet-level `TxStatus` alias**: verified,
neither exists at base, so they are never written rather than deferred.
Internal `db.TxStatus` stays.

**Out of milestone** — PostgreSQL *Manager* and daemon harness (the PostgreSQL
**Store** is not deferred — see §5.8), kvdb→SQL migration, concurrent `Load`,
multiple Managers over one database, global chain-sync coordination, and every
pre-existing bug the two commands do not reach.

---

## 5. Work items

### 5.1 ✗ One **physical database** per Manager

Revision 1 said the constructor opens "one Store". Not implementable for
kvdb: `kvdb.NewStore(dbConn, txStore, addrStore)`
(`wallet/internal/db/kvdb/driver.go:33`) requires a live `wtxmgr.TxStore` and
`waddrmgr.AddrStore`, which exist only after `waddrmgr.Create`/`Open`.

**✗✗ One constructor, not two.** Revisions 3 and 4 specified
`NewKVDBManager`/`NewSQLiteManager`. That was `codex`'s design preference and I
adopted it without checking it against the owner's own recorded rulings, which
it contradicts twice — archived `300-INDEX.md:66` decision 1 fixes the shape as
**`NewManager(ctx, cfg) (*Manager, error)`**, and decision 4 requires a
**`Backend` field**, validated once in `NewManager`. Revision 4's "do not add a
`Backend` tag" was therefore wrong on both counts. Two exported constructors
also multiply the public API per backend — PostgreSQL (Task 321) would add a
third — where one config field adds none.

The invariant is **one physical database handle or SQL pool, opened and owned at
Manager construction**:

**Owner-specified shape** (2026-07-31), adopted verbatim:

```go
type DBBackend string

const (
        DBBackendKVDB   DBBackend = "kvdb"
        DBBackendSQLite DBBackend = "sqlite"
)

type ManagerConfig struct {
        Backend        DBBackend
        DataSource     string
        MaxConnections int
        ChainParams    *chaincfg.Params

        // Deprecated: Used only by the legacy KVDB backend.
        NoFreelistSync bool

        // Deprecated: Used only by the legacy KVDB backend.
        Timeout time.Duration
}

func NewManager(ctx context.Context, cfg ManagerConfig) (*Manager, error)
```

`DataSource` rather than a path, because decision 4 requires "the SQLite path or
PostgreSQL DSN" — one field carries a bbolt path, a SQLite path, or a DSN.
`MaxConnections` is the shared SQL pool knob and serves PostgreSQL too.

`NewManager` switches on `Backend` **once**, opens the selected physical
database, and owns it through `Manager.Close`. Implement only the kvdb and
SQLite cases; Task 321 adds `DBBackendPostgres` as a **third switch case** — not
a new constructor and not another config type. Callers get one obvious pair:

```go
mgr, err := wallet.NewManager(ctx, cfg)
if err != nil {
        return err
}
defer mgr.Close()
```

- `NewManager` validates `ChainParams` **once**;
- `Create` and `Load` receive no database or network selection;
- the Manager injects its `ChainParams` into each Wallet;
- per-wallet `Config.ChainParams` stays temporarily as an internal
  compatibility carrier to avoid widespread churn, but callers of the new
  Manager do not set it;
- per-wallet `Config.Chain` **stays** — it is the wallet's chain connection, not
  database configuration.

Validate once in `NewManager`: `Backend` set and supported, `DataSource`
non-empty, `ChainParams` non-nil, plus the existing per-backend checks such as
SQLite's rejection of a negative `MaxConnections`. No defaulting — no
`withDefaults`, no `defaultSQLiteDBName`, no path derived from a legacy kvdb
location.

✗ An earlier revision 5 draft claimed the flat struct leaves "no invalid state
to represent". Not true — it deliberately carries fields that do not apply to
every `Backend`. What it avoids is the rejected
`DBConfig{Backend, KVDB, SQLite, Postgres}`, where per-backend sub-structs could
be populated *simultaneously*. Document the no-op matrix rather than adding
another config or a second validation framework:

| field | kvdb | sqlite (and future postgres) |
| --- | --- | --- |
| `DataSource`, `ChainParams` | used | used |
| `MaxConnections` | ignored | used |
| `NoFreelistSync`, `Timeout` | used | ignored |

These two `Deprecated:` markers follow §2's ruling, which marks every kvdb-only
public field.

- **sqlite**: open one `sqlite.Store` — pool and Store together. Create and Load
  **use that already-open Store**; neither opens another;
- **kvdb**: open an existing bbolt file or create an empty one and own that
  `walletdb.DB`. `Manager.Create` runs the one legacy create transaction, then
  opens the committed waddrmgr/wtxmgr state and constructs the kvdb Store
  adapter; `Manager.Load` opens those managers and constructs it. Modern kvdb
  Load uses the **Manager-owned** handle, never `Config.DB`;
- ✗ **the mutex covers admission, not the whole path.** An earlier draft said to
  hold it "across the sequential check-build-publish", which silently makes
  concurrent `Load` and Create-vs-Load safe — behaviour §4 and §9 defer to
  [[319-manager-and-wallet-lifecycle-hardening.md]]. Caller serializes `Create`
  and `Load`; take the existing mutex only around **cache lookup, the
  in-process kvdb second-name check, and cache publication**, and do *not* hold
  it across backend Create/Open or Wallet assembly. No lock-aware build helpers,
  no publication protocol, no concurrency tests here;
- the Manager overwrites each Wallet's private `ChainParams` with its own before
  validating, so callers do not duplicate it.

**✗ `NewManager` changes signature; it is not deleted.** Base exports
`func NewManager() *Manager` (`wallet/manager.go:199`) — a Manager with no owned
database, which the ownership ruling forbids. It keeps its name and gains
`(ctx, cfg)`. With one constructor there is also no private `newManager` helper
to add — an earlier draft carried one over from the two-constructor design.
**Every caller migrates in the same #1291 commit**, not later:
`bwtest/harness_wallet.go:67` is compiled by ordinary package tests and
`itest/manager_test.go:29` by the `itest`-tagged build that DoD 8 requires at
*every* head, so leaving them for #1292 would leave #1291's head uncompilable.
Migrate them to `Backend: DBBackendKVDB` to preserve base behaviour, and migrate
the 13 `wallet` package test call sites. The deprecated `Loader` does not use
it, so no compatibility wrapper is needed.

✗ **That migration drags the harness's path change into #1291 too.** Revision 5
left path-only harness setup in #1292 while requiring caller migration in
#1291 — impossible: base `HarnessTest` has only `WalletDB walletdb.DB`
(`bwtest/harness.go:44`) and **no `WalletDBPath`**, `setUpWalletDB`
(`harness.go:300-311`) opens `<tempdir>` immediately, and `ManagerConfig`
needs a `DataSource` string that cannot be recovered from an open
`walletdb.DB`. Reopening an inferred path while the base handle is still open
would also collide on the bbolt file lock. So #1291 additionally carries exactly
these input-forced steps, and nothing more:

1. replace `HarnessTest.WalletDB` with `WalletDBPath`;
2. make `setUpWalletDB` allocate the unopened path;
3. migrate `bwtest`/`itest` with fixed `DBBackendKVDB` and that path;
4. register a temporary direct `Manager.Close` cleanup right after
   construction, ordered after Wallet Stop.

§5.7's #1292 then owns only `h.dbType` selection, the Wallet/Manager registries,
centralized teardown, and backend artifact proof — **replacing** the temporary
#1291 cleanup rather than duplicating it.

**The kvdb transaction bridge.** §4 deletes `OpenStoreParams` and `StoreHandle`
but a bridge is still unavoidable: `waddrmgr`/`wtxmgr` `Create`/`Open` need
walletdb buckets, those exist only inside `walletdb.Update`, and putting that
transaction in `wallet/manager.go` fails §1's gate. Name the replacement so an
executor does not reinvent the deleted abstractions — two internal
`wallet/internal/db/kvdb` functions:

✗ An earlier revision 5 draft gave these loose parameters
(`OpenStore(dbConn, pubPassphrase, chainParams)`), which breaks §1's one-Config
rule one section after stating it. They take the adapter's single package-local
config:

```go
// kvdb.Config — the one configuration source for this adapter.
type Config struct {
        DB            walletdb.DB      // already open, owned by the Manager
        ChainParams   *chaincfg.Params
        PubPassphrase []byte
}

func OpenStore(cfg Config) (*Store, *waddrmgr.Manager, error)
func CreateWallet(cfg Config, req CreateWalletRequest) error
```

`CreateWalletRequest` carries only root key, private passphrase, and birthday —
operation data, not a second component configuration.

`OpenStore` runs migrations and opens both legacy managers inside its own
transaction, embeds the transaction store in `Store`, and returns the address
manager only because the Manager must close its live key state before closing
the database. No handle object, no returned close function. Until a Wallet is
successfully published, Create/Load close a locally opened address manager on
every downstream error; bind it to the Manager only after assembly succeeds.

**SQLite needs its address deriver.** `sqlite.Config.DeriveAddress` feeds
`sqlite.Store.deriveAddress` (`store.go:82`), which `NewDerivedAddress` passes
to `db.NewDerivedAddressWithOps`. A nil callback leaves a Manager-created SQL
wallet unable to serve `AddressManager.NewAddress`. The Manager itest never
derives an address, so it would not catch this — but the stated deliverable is a
usable library. `NewManager` installs a small **private** deriver built from the
constructor-owned `ChainParams`. Do not put `DeriveAddress` on `ManagerConfig`
or expose a callback to callers.

**In-process single-wallet admission for kvdb.** §5.1 binds one kvdb address
manager and closes exactly that one, while §4 defers name admission to Task 318.
Both cannot hold for a public `Load`: loading name A binds its manager, then
loading name B misses the `wallets` map and rebinds — overwriting the only close
target or leaking a second live manager. The *persistent* name problem stays
deferred (kvdb stores no wallet name), but the in-process guard is forced by
this ownership shape: after the same-name cache hit check, under the existing
mutex, reject a distinct kvdb name when `len(m.wallets) != 0`. Use the existing
map — no `Manager.kvdbWallet`, no name marker, no set of managers, no persisted
metadata. One test proves the rejected call does not replace the bound manager
-- **asserted publicly** (§316-1 amendment 5): the second name fails both
`Create` and `Load`, and the first name still returns a usable Wallet. Do not
assert `m.kvdbBound` or any other private field; §1's style rule governs.

**✗ Validation: delete one check, do not split.** Revision 3 proposed splitting
`Config.validate()` into backend-independent and legacy halves. Smaller
alternative, and it is sufficient: the only production callers are
`wallet/manager.go:289` (`Load`) and `:409` (`prepareWalletCreation`) — both
become typed-Manager paths, and deprecated construction never calls it. So
simply **remove the `c.DB == nil` check** from the existing `validate`, keep the
Chain / ChainParams / Name / RecoveryWindow checks, and let the Manager
copy its constructor-owned `ChainParams` into the private Wallet config before
validating. `Config.DB` stays only because deprecated Wallet methods reference
it. Adding a second validation function with no production caller is machinery.
Proof stays the same: one test per backend with `Config.DB == nil` and
a nil caller `ChainParams`. Do not paper over this with a dummy DB.

**Close ordering.** `waddrmgr.Manager.Close()` zeroes scoped account keys, locks
private material, and zeroes `cryptoKeyPub`/`masterKeyPub`
(`waddrmgr/manager.go:483-506`). Closing only the `walletdb.DB` leaves that key
state live. So: after the caller has stopped its Wallets, `Manager.Close` closes
the bound kvdb `waddrmgr.Manager` if present, **then** the owned walletdb
handle; SQLite closes its Store. kvdb holds **at most one** bound address
manager — a single private field, not a set — since the kvdb contract is one
Manager and one wallet. No dedup collection, per-wallet close ownership,
idempotency, use-after-close guarantee, or close state machine.

This needs only private kvdb state on the Manager — no `StoreHandle`,
`OpenStoreParams`, backend union, or `kvdbWallet`.

### 5.2 ✗ kvdb create/open on the dual format

Use existing `waddrmgr.Create`/`waddrmgr.Open` with base's public and private
passphrases; create/open wtxmgr in the same core transaction.

Revision 1 claimed `HasScriptKey()` makes script routing format-blind. **Wrong.**
`witnessScriptAddress` persists `isSecretScript` per address
(`waddrmgr/db.go:168-171`, deserialized at `:1551`) and the decrypting key is
chosen from it (`waddrmgr/address.go:956-958`):

```go
cryptoKey := a.manager.rootManager.cryptoKeyScript
if !a.isSecretScript {
        cryptoKey = a.manager.rootManager.cryptoKeyPub
}
```

**✗ But revision 2 overstated it as "per address" across the board.** Legacy
P2SH `scriptAddress` (`address.go:799-802`) has **no such field**, and its
`Script()` unconditionally uses `cryptoKeyScript` (`:844-861`). So:

- kvdb reads the persisted bit for witness/taproot scripts;
- kvdb synthesizes `ScriptIsSecret = true` for P2SH;
- SQL rows are always `true` — SQL ciphertext uses the script key;
- carry it as one internal field, `db.AddressSecret.ScriptIsSecret`;
- new public-Wallet kvdb script imports use the legacy **non-secret** route
  unless an API explicitly asks for secret storage. Document which retained
  import APIs can produce each class.

This is one bit naming which key encrypted one ciphertext — not watch-only
state, identity, or provenance — so it does not touch ADR 0012/0013. Do not add
`AddressInfo.HasAccountNumber`, a new `AddressInfo.HasScript` (base's means
something else), or a wallet-level `ScriptIsSecret`.

**✗ The bit alone does not make SQL script import work.** §7 Q4 keeps base's
generic `Encrypt`/`Decrypt(CryptoKeyType, ...)`, but base
`wallet/address_manager.go:876` encrypts *every* retained `ImportTaprootScript`
under `CKTPublic`, and `WalletVault.selectUnlockedCryptoKey` returns
`errUnsupportedCryptoKeyType` for `CKTPublic`
(`keyvault/vault_cipher.go:81-83`). So the SQL script case DoD 6 requires fails
at **encryption**, before any Store write — `ScriptIsSecret` only helps a later
read. Dropping the typed-cipher commit without noticing this restores a broken
SQL import.

Minimum fix, and it stays a switch case rather than the 542-line redesign:
`WalletVault` maps a `CKTPublic` cipher request to its `cryptoKeyScript` — SQL
has no public crypto key, and this retained import is the only caller that asks
for one. `LegacyWalletVault` keeps forwarding `CKTPublic` to the real kvdb
public key. Persist `ScriptIsSecret = false` for that kvdb import and `true` for
SQL, so each backend's later decryption selects the key it actually used. One
Wallet-level SQL import/read round trip covers it — it fails against base's
unsupported-`CKTPublic` branch. Task 320 still owns replacing the generic API.

### 5.3 SQLite create and load

**Create:** generate secrets with `keyvault.CreateWalletSecrets` → call
`Store.CreateWallet` on the Manager's already-open Store (already
transactional) → use the `WalletInfo` it **returns**; do not re-query for the ID
→ build `keyvault.WalletVault` over that Store.

Return the backend's own duplicate/constraint error for an existing artifact. No
preflight existence check, no new `ErrWalletExists`, no retry.

**Load:** resolve the wallet by name through the Manager's Store, use the
returned authoritative wallet ID and watch-only state, construct `WalletVault`
and the runtime cache, build and publish. A sequential second `Load` returns the
cached pointer; `Create` never does — it calls the backend and lets the
duplicate constraint surface. Preserve base's not-found error. No concurrent
`Load`, no Store reopening, no loser cleanup.

**SQLite automatically creates no default accounts.** Verified — the
`Start`-path accounts read at `wallet/controller.go:328` discards its result by
design ("the read itself surfaces a broken or unreachable store as a startup
error"), SQLite `ListAccounts` returns an empty slice, `AttachAccountBalances`
exits before querying, and recovery init accepts zero accounts. An account-less
SQL wallet starts, scans an empty watch set, syncs, and stops.

✗ Revision 4 said "all seeding is Task 317", which reads as deleting base
behaviour. It is not: base's wallet-level `CreateWalletParams.InitialAccounts`,
its validation, and the post-create `importInitialAccounts` loop for `ModeShell`
all **stay exactly as they are** — DoD 3 already permits that field. What Task
317 owns is moving account creation *inside* the wallet transaction. Do not copy
`InitialAccounts` into `db.CreateWalletParams`, and leave its non-atomic failure
semantics unchanged.

### 5.4 Store/Vault boundary — the walletdb co-goal

Route through `db.Store` / `keyvault.Vault`, deleting `wallet/db_ops.go` once
its last caller moves:

- **lifecycle** — `Unlock`, `Lock`, `ChangePassphrase` through the Vault;
- **signer + PSBT** — account and address secret reads through Store/Vault;
- **sync/recovery** — `verifyBirthday`, `performRuntimeSetup`, `walletBirthday`,
  lookahead from Store account xpubs, scan-batch persistence, sync-tip
  read/write, `Wallet.SyncedTo`.

**✗ Address secrets select by script pubkey, not address ID.** Revision 1 said
to reuse the address-ID lookup. On kvdb that returns the **wrong secret**:
`setAddressID` (`kvdb/addressstore.go:847`) assigns a synthetic ID via
`sortAddressInfos` (`:912-918`), a one-based per-account ordinal that collides
across accounts and moves when the view changes. Use
`GetAddressSecretQuery{WalletID, ScriptPubKey}` — the signer already holds the
output script. SQL queries `(wallet_id, script_pubkey)`, **replacing** the base
address-ID query rather than keeping two; kvdb resolves through its
script-to-address path. Reject an empty script.

**✗ Account secrets need exactly one identity — the account number.** Revision 3
carried optional `AccountID` *and* `AccountNumber` plus XOR validation, and had
SQL path-only signing convert a path to a durable ID via `GetAccount`. All of
that is speculative:

- `ImportAccount` takes an extended **public** key only
  (`wallet/account_manager.go:151-154`), so imported accounts hold no
  account-level private key and never need an account secret;
- wallet-derived accounts — the only spendable kind the public API exposes —
  always have a BIP44 number;
- raw imported private keys are per-address material and go through
  `GetAddressSecret`.

So the `AccountID` route buys nothing here while costing an extra `GetAccount`,
a second SQL query, per-dialect branches, a synthetic kvdb ID mapping, XOR
validation, and four selector-combination tests.

```go
type GetAccountSecretQuery struct {
        WalletID      uint32
        Scope         KeyScope
        AccountNumber uint32
}
```

SQL selects by wallet, scope and number; kvdb uses the same directly. Path-only
APIs (`ECDH`, `SignDigest`, `DerivePrivKey`) already carry the number in their
BIP32 path and need no conversion. Address-driven derived signing uses the
non-nil `AddressInfo.AccountNumber`; an imported-xpub child, whose
`AccountNumber` is nil (`db/data_types.go:388`), returns the existing
no-associated-private-key error **before** any secret query — it must never
fabricate account 0. There is no optional selector, so there is no `Validate`
and no both/neither matrix. `AccountID` stays on `AccountInfo`/`AddressInfo`
where ADR 0013 requires it; it is simply not on this request. If a future API
admits an imported account private key, that PR adds an ID request then.

**Passphrase rotation needs a two-half Vault contract, validated first.** Base
`wallet.ChangePassphraseRequest` (`wallet/controller.go:90`) carries
`ChangePublic`/`PublicOld`/`PublicNew` and `ChangePrivate`/`PrivateOld`/
`PrivateNew` independently, and kvdb changes both in one transaction. Define an
internal `keyvault.ChangePassphraseParams` with both halves.

- `LegacyWalletVault` applies the requested halves in one `walletdb.Update`;
- `WalletVault` **validates the whole request before reading or mutating any
  secret**: if a public change is requested it returns
  `ErrPublicPassphraseUnsupported` and rotates nothing. Applying the private
  half and *then* failing on the public half would leave a partial rotation
  reported as an error;
- never infer intent from an empty value — an empty public passphrase is legal,
  so presence stays explicit. (This answers the owner's open #1272 thread: no,
  `ChangePublic` cannot be inferred from whether `PublicOld` is empty.)

**Rotation works while locked — state it, or the rebuild regresses it.** Base
`walletState.canChangePassphrase` requires only a *started* Wallet, and base
kvdb rotation authenticates `PrivateOld` inside the transaction, so rotating a
locked wallet works today. But base `WalletVault.ChangePassphrase` requires
`unlockedState`; routing through the Vault while keeping that precondition would
be a public behaviour regression. So: **both** Vault implementations rotate
whether locked or unlocked; SQL authenticates `PrivateOld` from the persisted
master-key parameters rather than cached runtime keys; a locked Vault stays
locked and an unlocked one keeps its runtime keys. DoD 6 carries a locked
private-only rotation case per backend.

Do not claim kvdb rotation is atomic *in memory*: waddrmgr can swap live key
state before the outer transaction commits. That is a retained base limitation
— document it, do not fix it here.

Keep the query surface minimal; drop unused selectors and no-op row-conversion
types. Defer targeted-rescan identity redesign, live-watch redesign, deep-reorg
hardening, and unrelated transaction-status changes.

### 5.5 ✗ Start/Stop at base shape, plus one forced vault lock

Caller serializes Start and Stop. Start does runtime setup and starts workers;
Stop cancels, joins, returns to `Stopped`, stays in the Manager cache, and
closes nothing.

**Revision 2's "base-shaped Stop" would introduce a regression.** Base `Stop`
holds no vault — base `wallet/controller.go` has zero `keyVault` references — so
"base locks nothing at Stop" is not the parity argument it looks like. Once
`Unlock` routes through the vault, the vault holds `unlockedState` across a
Stop, and `WalletVault.Unlock` returns `ErrVaultUnlocked` when
`v.unlockedState != nil` (`keyvault/vault_lifecycle.go:46-53`). So Unlock → Stop
→ Start → Unlock fails permanently while the wallet reports itself locked, and
the wallet can never become sign-capable again. The current head already guards
this at `controller.go:390`.

**Ruling:** after workers join successfully and before the transition to
`Stopped`, `Stop` calls `keyVault.Lock()`. It still closes no Store and adds no
concurrency protocol. Broader Stop/Unlock races stay in Task 319.

**✗ Amended by §316-1 and §316-4.5 — the test prescription, not the ruling.**
Revision 5 required "unlock → Stop → Start → unlock on both backends". That
sequence is now proved at the narrowest boundary instead: the adapted base
`TestControllerStop` places the wallet in the unlocked state, requires exactly
one Vault `Lock`, and asserts the stopped state. It is the **sole**
Stop-to-Vault dispatch proof.

There is no Manager-level Stop test: §316-4.5 deletes
`TestManagerStopClosesNoStore` rather than reducing it. `Manager.Load` returns
the cached pointer without touching the Store, and `Wallet` holds no Manager
back-reference, so a post-Stop cached-`Load` assertion has no falsifier this
stack can reach — it composes base `TestControllerStop` with base
`TestManagerLoad_ExistingWallet`. Cache retention stays with those base tests
and the reduced SQLite cached-Load case; `Wallet.Stop` has no Store-close path
and harness teardown owns the one exercised Stop-then-`Manager.Close`
sequence. Add no restart scenario at any level — not to the itest, not as a
stack-added unit test. The forced `keyVault.Lock()` itself is retained
unchanged.

### 5.6 The deprecated-constructor cache (forced repair)

`OpenWithRetry` sets `store`, `keyVault`, `txStore`, `addrStore` — but **not
`cache`**, which the retained signer routing dereferences at
`wallet/account_manager.go:289`, `:346` and `wallet/controller.go:328`. This is
a regression *these PRs* introduce, so ruling 9 permits the repair.

Initialize `cache` in **every** constructor, in the same commit that makes it
mandatory, and delete the `cache == nil` / `keyVault == nil` guards there.
Deleting the guards alone converts a returned error into a panic. This
authorizes no other deprecated work.

### 5.7 ✗ #1292 — real backend selection and one cleanup owner

Replace the current commit entirely (it only edits comments).

**✗ Registration order.** Revision 2 said to register Wallet and Manager
together before `Start` "so a failed Create still releases the pool" — but a
failed `Create` produces no Wallet, so nothing is registered and the open pool
leaks. The order is:

1. construct the Manager for h.dbType (it cleans up its own partial
   resources on failure);
2. **register the Manager immediately**;
3. call `Manager.Create`;
4. register the Wallet immediately on success;
5. call `Wallet.Start`.

The single cleanup path stops every registered Wallet, then closes every unique
Manager exactly once, joining errors rather than returning on the first.
`testing.Cleanup` runs LIFO, which is how today's duplicate Stop arose. Delete
the direct Stop cleanup in `itest/manager_test.go`. Add a failing-Create cleanup
test.

Also: one harness helper switches on `h.dbType` and builds the kvdb or SQLite
Manager from `h.WalletDBPath`; `testCreateWallet` uses it and `CreateEmptyWallet`
follows as a compile-forced user.

`WalletDBPath` and the path-only `setUpWalletDB` already exist by this point —
they land in **#1291**, because the caller migration cannot compile without them
(§5.1 ✗). #1292 does not re-do that; it replaces #1291's temporary direct
`Manager.Close` cleanup with the registry-based teardown above.

**Backend proof** — a `db=sqlite` run must not be able to pass while the test
builds kvdb. Prove it from the artifact (SQLite file header/schema present after
Create; bbolt file for kvdb), not by exposing a production backend tag for the
test's benefit.

No new itest case, no daemon gate, no PostgreSQL harness.

### 5.8 PostgreSQL's compile-forced boundary

The PostgreSQL **Store** is not deferred. Adding account/address secret methods
to `db.Store` forces the PostgreSQL adapter and generated code to compile, and
`wallet/internal/db/itest` keeps it green. #1272 carries the minimum PostgreSQL
secret-query adapter and generated output. Only Manager construction and
daemon/itest selection go to Task 321.

Edit SQL **source** for both dialects, run `make sql`, never hand-edit sqlc
output. Every SQL-source-changing commit contains both dialects' source *and*
generated output, and the commit that first expands `db.Store` also contains the
SQLite, PostgreSQL and kvdb implementations — otherwise CI's per-commit `make
unit` hits an uncompilable interface commit.

---

## 6. Per-PR ownership

| PR | contents |
| --- | --- |
| **#1272** `impl-secret-store`, from `2ea76dfc5` | SQL wallet-secret genesis; minimal account and script-pubkey address-secret Store surfaces with SQLite/PostgreSQL/kvdb adapters; the `ScriptIsSecret` bit; `LegacyWalletVault`; the two-half passphrase contract; lifecycle and signer/PSBT routing; §5.6; focused tests. |
| **#1291** `impl-runtime-store-6`, from rebuilt #1272 | `NewManager(ctx, cfg)` with **all** callers migrated in the same commit (incl. `bwtest` and `itest`); the `Config.DB` validation trim; the owned physical database, kvdb bridge, SQLite deriver, in-process kvdb admission and close ordering (§5.1); dual-format kvdb create/open/bind; SQLite create/load (§5.3); runtime Store migration and sync/recovery routing; `wallet/db_ops.go` deletion; §5.5 Stop. |
| **#1292** `wallet-default-sqlite`, from rebuilt #1291 | §5.7 only — harness selection, backend proof, ordered teardown. No production Wallet changes. |

Within #1272, land Store data and query support before routing production
callers.

✗ Revision 5 also said "within #1291, land Manager construction before changing
itest callers" — which recreates the uncompilable intermediate commit §5.1
exists to prevent. The rule is **atomic**: `NewManager(ctx, cfg)`, the path-only
harness input, every caller migration, and the temporary cleanup land in **one**
#1291 commit. #1292 then changes only final backend selection and centralized
ownership.

Keep `wallet/db_ops.go` alive until #1291 moves its last caller, so #1272's gate
count is non-zero and §1's zero gate applies only at the #1291 and #1292 heads.

Cross-cutting items need a named owning commit: `PubPassphrase` documentation,
`ScriptIsSecret` read/write conversion, every Wallet constructor's cache
initialization, SQL source with its generated output, and each DoD 6 test.

**First execution step — the commit manifest.** Before any branch is rewritten,
inventory `2ea76dfc5..6da45e3f8`, `6da45e3f8..65d2db689`, `65d2db689..672b9476d`
and give **every** old OID exactly one disposition (drop / rework / fold), with
its replacement. Then list each new commit with its owning PR, parent, contents,
and gates. This is produced during execution, not guessed now — but it must
exist and be checked before the first push.

---

## 7. Decisions

Revision 2 left four questions open while writing the rest as if they were
answered. Three are closed here on evidence; one is a genuine preference.

**Q1 — does `Stop` lock the vault? CLOSED: yes.** `codex` and `fable` both said
defer, arguing base parity and that the itest never unlocks. Both premises are
true and the conclusion is still wrong: base has no vault at all, so there is no
parity to preserve, and the vault-routed `Unlock` creates a new permanent
failure mode (§5.5). `gpt` alone caught it. The scope rule admits it as a
regression these PRs introduce.

**Q2 — kvdb store ownership. CLOSED: Manager-owned, via `NewManager(ctx, cfg)`.**
This was never actually open: archived decision 1 fixes both the ownership *and*
the constructor signature, and decision 4 requires the `Backend` field.
`gpt`'s asymmetric caller-owned-kvdb alternative contradicts decision 1;
`codex`'s two typed constructors contradict it too, and revisions 3–4 wrongly
adopted them (see §5.1 ✗✗). Owner rulings win over reviewer preference.

**Q4 — the Vault cipher API. CLOSED: keep base's `CryptoKeyType`
`Encrypt`/`Decrypt`.** `codex` measured the typed-cipher commit `9b07468d9` at
542 `A+D` across 12 files (204 `A+D` across six production files) and it is not
a prerequisite for removing transactions or running the itest. `ScriptIsSecret`
selects `CKTScript` vs `CKTPublic` for kvdb reads; SQL always selects
`CKTScript`. The typed private/script redesign moves to Task 320 with the public
script-import cleanup. (`fable` preferred typed; overruled on the metric. If the
owner wants typed anyway, say so and it becomes binding — but it is an API
improvement, not a requirement.)

**Q3 — does #1292 flip the itest default? CLOSED: keep `db=kvdb`.** My "one
line" argument was wrong — `make itest` supplies its own default
(`Makefile:178`, `-db="$(if $(db),$(db),kvdb)"`) and the header docs at
`:165-169` name kvdb, so a consistent flip touches the Go flag, the Makefile and
the docs. Both DoD-1 commands select explicitly and pass either way, so the flip
buys no capability and is an unrelated delta from base. Closed per this task's
stated fallback; the owner can reopen it with one word, and the change would be
confined to #1292.

(Attribution correction: revision 3 said "all three reviewers" favoured keeping
the default. `fable`'s earlier round favoured the flip. The base-relative
reasoning above stands without the vote count.)

---

## 8. Definition of Done

1. Both `make itest chain=btcd db=kvdb icase=manager` and `db=sqlite` pass with
   `nocache=1`, logs recorded against the final #1292 OID, and §5.7's backend
   proof in place.
2. §1's gate returns no output at the #1291 and #1292 heads; `wallet/db_ops.go`
   deleted; #1272's expected non-zero count stated in advance.
3. **Fail** if the base-relative diff adds `DefaultAccountName`,
   `DefaultAccountScopes` or `InitialAccounts` to `db.CreateWalletParams`; if
   the tree gains a wallet-level `TxStatus` or `TxDetail.Status`; or if
   `waddrmgr/format.go`, `format_test.go` or ADR 0014 exist. **Permit** base
   `mgrver`, `waddrmgr.DefaultAccountName`, wallet-level
   `CreateWalletParams.InitialAccounts`, and internal `db.TxStatus` — do not
   grep bare names repo-wide.
4. `wallet/deprecated.go`'s delta classified line-by-line as compile-forced,
   forced shared-code repair, or gofmt-only — zero improvements.
4b. **One Config per component** (§1), mirroring that table exactly: Manager has
   exactly one `ManagerConfig`; Wallet retains its one `Config`; each Store
   backend has **at most one** package-local config; **keyvault adds none** and
   keeps its dependency constructor arguments. Inventory every config struct the
   diff adds or changes and name the component it configures. Fail only on a
   *second* config source for the same component or backend, a backend-specific
   variant, or a handle/config wrapper. §1's operation request types are
   excluded, and required dependencies passed directly to a constructor are not
   configuration.
5. Every **regression** test for a behavioural fix fails with its fix reverted,
   with command and failure recorded. New feature and compile-contract tests
   need positive and negative coverage but not mutation falsification.
6. Behaviour tests for the co-goal the itest never reaches, each named with its
   package, owning PR, and setup API. "Both backends" means kvdb and SQLite at
   the Wallet level; PostgreSQL conformance comes from the DB integration suite.

   | area | required cases |
   | --- | --- |
   | create/open | kvdb dual-passphrase create and open, including a **custom** public passphrase through `CreateWalletParams.PubPassphrase` then `Config.PubPassphrase` on Load |
   | rotation | kvdb public-only, private-only, combined; SQL private succeeds; SQL public-only and combined **fail before any mutation**, old private passphrase still valid; a **locked** private-only rotation on each backend (§5.4) |
   | lifecycle | vault lock/unlock both backends; the serial Stop-time vault lock proved by the adapted base `TestControllerStop` — unlocked wallet, **exactly one** `Vault.Lock`, stopped state — as the **sole** Stop proof (§5.5 as amended by §316-1/§316-4.5). No Manager-level Stop test: §316-4.5 deletes it as duplicate scope. **No** unlock → Stop → Start → unlock sequence and no broader Start/Stop race at any level |
   | signing | derived-account signing both backends; path-only (`ECDH`/`SignDigest`/`DerivePrivKey`) on both; **mock-backed PSBT Store routing** with a zero and a non-zero account (real-backend, address-driven PSBT signing is **deferred** — the pinned base resolves a persisted scope's coin type against `ChainParams.HDCoinType`, see `opus-report.md` §5.5 and §316-7); an imported-xpub child returns the existing no-associated-private-key error and does not fall back to account 0 — assert the public error, **not** a mock call count, per §1's style rule |
   | scripts | kvdb adapter synthesizes `true` for P2SH; a kvdb taproot script imported through a **retained** public API reports `false` and decrypts with `CKTPublic`, and stays readable while locked. The same import on SQL round-trips through the `CKTPublic`→script-key mapping (§5.2) and reports `true`, using an **unlocked watch-only** SQLite wallet — verified, not conditional: `WalletVault.Encrypt` needs a non-nil `unlockedState`, and `db.NewImportedAddressWithOps` calls `RequireAddressPrivKeyOnSpendable` (`addressstore_newimported.go:73`), whose doc states "Public-only and script-only imports are both rejected", so a spendable SQL wallet refuses this import. (The rejecting helper is `RequireAddressPrivKeyOnSpendable`, **not** `ValidateWatchOnly`, which only guards the opposite direction.) kvdb applies no symmetric check — its legacy rows are grandfathered — so its case stays spendable and locked-readable. Add a secret witness case only if a retained API can create one — no fixtures for pre-existing kvdb wallets, that is Task 323 |
   | imported keys | kvdb imported-private-key signing only. There is **no** modern private-key import — the sole WIF path is `deprecated.go:6549`, and §5.6 forbids extending it — so an equivalent SQLite case is impossible and is not required |
   | deprecated | `OpenWithRetry` with its initialized cache |
   | SQLite | sequential create/load; repeat load returns the cached pointer; a Manager-created SQL wallet derives an account through `AccountManager` and an address through `AddressManager` (proves §5.1's deriver is installed) |
   | kvdb | a `Load` under a second, distinct name is rejected and does not replace the bound address manager (§5.1) — proved **publicly** per §316-1 amendment 5: second-name `Create` and `Load` both fail, and the first name still returns a usable Wallet. No private-field assertion |
   | config | for each `NewManager` `Backend` value, `Create` and `Load` succeed with per-wallet `Config.DB == nil` and `Config.ChainParams == nil` — the Manager injects its constructor-owned chain parameters before validation |

7. Per commit: §316-10.1's non-mutating exact-OID loop over each rebuilt PR
   range — check out each candidate OID detached and run `scripts/check-commit.sh`
   there. Do **not** run `scripts/check-each-commit.sh` on the final refs; it
   rebases, so it validates OIDs that are not the ones delivered. Also
   `git diff --check <parent> <commit>` for every owned commit. CI runs `make
   unit` per commit, so `go build` is insufficient — `_test.go` must compile.
8. Per PR head, from a clean worktree (these targets mutate files — check `git
   status --porcelain` afterwards): `sample-conf-check`, `fmt-check`,
   `tidy-module-check`, `rpc-check`, `sqlc-check`, `sql-format-check`,
   `sql-lint-check`, `go install -v ./...`, `protolint`, `lint-check`, `unit`,
   `unit-cover`, `unit-race`, and `go test -run '^$' -tags='itest dev nolog'
   ./itest`. From #1272 on, also `itest-db` and `itest-db-race` for `db=sqlite`
   and `db=postgres` — these execute the **already shared** DB contract that CI
   runs as a required job over the same matrix
   (`.github/workflows/main.yml:170-196`), so they are parity/hygiene for the
   compile-forced pg Store methods. Per §316-1 amendment 4 they authorize no
   PostgreSQL Manager, harness, daemon, behavioral row, or new acceptance
   criterion.
   Lint is the four Makefile passes — default, `./itest` with
   the `itest` tag, the SQLite DB package set, and the PostgreSQL set with
   `test_db_postgres`.
9. All commits GPG-signed (`%G? = G`); bodies ≤72 columns; comments obey §1's
   style rule.
10. Metrics with recorded commands: base `2ea76dfc5`; *before* = base through
    `672b9476d`; *after* = base through the rebuilt #1292; cumulative **and**
    incremental rows per PR, so moving work between branches cannot disguise
    growth; additions, deletions, `A+D`, changed files.

    **This is a gate, not a report.** The rebuilt stack must improve all three
    base-relative metrics against the old baselines — all-file 132 files /
    `A+D` 22,469, production-Go 56 files / `A+D` 9,162 — or the owner must
    approve a named exception. Separately, every changed production file must
    map to one retained §5 item or a compile-forced generated/interface change;
    an unmapped file comes out of the PRs. Falling under the old total is not
    permission to keep an unmapped feature.
11. Push safety — **the procedure now lives in §316-9.1, §316-12.2 and
    §316-13**; follow it there, because this row's single-remote,
    same-order rollback contradicts the per-remote protocol. Only the
    invariants remain binding here: every authorized `(remote, ref)` update
    uses its own full 40-char `--force-with-lease=<ref>:<old-full-oid>` and
    never plain `--force`; pushes follow #1272 → #1291 → #1292 dependency
    order; rollback follows **reverse** dependency order across every remote
    that changed; and both backup generations —
    `refs/backup/t315-pre/{1272,1291,1292}` for the pre-Task-315 heads and
    `refs/backup/t316-pre/{1272,1291,1292}` for the rev-6 inputs — are created,
    verified, and retained through acceptance.
12. Review threads: after the final heads are pushed and green, re-fetch
    **paginated** at those exact OIDs — do not trust the 23/8/4 snapshot in
    `.lgtm/pivot/t314-thread-dispositions.md`, which is pre-push. Record thread
    ID, URL, PR, path, disposition (removed / retained / rewritten-equivalent /
    deferred / rejected), replacement OID, and reply. Verify each PR's
    `headRefOid` still equals the validated OID before replying, then re-fetch
    and assert zero unresolved.
13. On completion: record the rebuilt heads as full OIDs here, mark this task
    completed, update `INDEX.md` roles, and reclassify the three review files as
    historical evidence of the replaced heads.

---

## 9. Where §4's deferrals went

| task | covers |
| --- | --- |
| [[317-atomic-wallet-and-account-creation.md]] | moving default/imported account seeding **inside** the wallet create transaction. Base's post-create `ModeShell` import stays in 315 (§5.3) |
| [[318-duplicate-create-and-kvdb-admission.md]] | duplicate create; optional restart-stable kvdb name identity. The **in-process** second-name guard is 315 §5.1, not this task |
| [[319-manager-and-wallet-lifecycle-hardening.md]] | concurrent `Load` of one name, concurrent Start/Stop, Manager use-after-close. **Not** kvdb second-name admission (315 §5.1), and not the `awaitCreate`/eviction races — those mechanisms are deleted |
| [[320-adr-0012-enforcement-and-watch-only.md]] | script-only imports; watch-only modes; the typed Vault cipher redesign (§7 Q4) |
| [[321-postgres-manager-and-daemon-harness.md]] | PostgreSQL at the Manager and in the harness (not the Store — §5.8) |
| [[322-shared-chain-reset-coordination.md]] | DB-wide `RollbackToBlock` with several wallets on one database |
| [[323-kvdb-to-sql-wallet-migration.md]] | the terminal plan for kvdb — and why it gets no new on-disk format |

`10001-resumable-per-wallet-rescan.md` and `10002-controller-request-completion.md`
are **reference only, with no scheduled owner**: the rescan redesign deferred out
of §5.4 and the accepted-response cancellation semantics `waitForAcceptedResp`
reached for. Live-watch redesign, deep-reorg hardening and transaction-status
changes are deliberately unplanned — no task owns them.

## 10. Archive references

| file | what it is good for |
| --- | --- |
| `archive/314-post-review-completion-of-the-three-prs.md` | the executed plan and its 13 owner rulings; §2 records which still bind |
| `archive/313-single-encryption-revision-of-the-three-prs.md` | how the single-encryption arc was designed — now deleted, kept as evidence |
| `archive/311-single-encryption-for-all-backends.md` | the doctrine 315 sets aside for kvdb |
| `archive/300-INDEX.md` | the superseded-designs table for tasks 305–313 |
| `archive/deferred-tasks.md` | pre-existing deferral backlog |
| `codex-review-task.md`, `gpt-review-task.md`, `fable-review-task.md` | the reviews this task is built from |
