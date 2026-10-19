# Task 296: SQLite-native secrets + runtime backend decoupling

Status: Planning (2026-07-10). Concrete execution plan for the top-3 upper
branches, reflecting the current reconcile state and the decision to make the
SQLite backend truly SQL-native. Refines the planning in
[294](294-reorganize-runtime-store-prep-sqlite-last.md) (branch reorg)
and [295](295-finalize-walletdb-removal-then-sqlite-config.md) (finalize
walletdb removal + SQL config).

## Update (2026-07-10)

The rt6 Manager design changed: see **task 297** — the runtime Manager becomes
SQL-native only (the legacy-kvdb path duplicated the existing `wallet.Loader`).
Do 297 before carving rt6.

## Goal

Finish the three upper branches so a **SQLite-backed wallet is truly SQL-native
— walletdb-free at the FILE level, not just the source level**:

- wallet secrets live in SQL (`wallet_secrets`), generated at create via the
  genesis crypto, and unlocked via `WalletVault` reading the store;
- **no walletdb file and no walletdb-backed `waddrmgr.Manager`** is created for a
  SQL wallet (pure `waddrmgr` helper APIs are still allowed — see the precision
  note below);
- the public managers (signer / account / address / utxo / tx) already route
  through `db.Store` / adapters / keyvault (source-level walletdb-free).

KVDB stays the default in the prep branch; the SQLite default flip is the final
branch only.

**Precision — walletdb-free != waddrmgr-package-free.** SQL backends are
walletdb-free, not `waddrmgr`-package-free. They may use `waddrmgr` as a pure
in-memory helper library for HD derivation, address-schema parity, and address
encoding, but must not use a walletdb-backed `waddrmgr.Manager` or any `waddrmgr`
DB operations. All SQL-backed persistence — wallet secrets, account metadata,
addresses, UTXOs, transactions, sync state, and scan horizons — must flow
through `db.Store`.

## Branch structure (Route B — the 5 lower PRs stay UNTOUCHED)

    impl-secret-store (#1272)      secret primitives: keyvault, GetAccountSecret,
      |                            store-backed key derivation, + wallet-secret genesis
      +-- impl-runtime-store-6     runtime wiring: DBConfig backend selection,
      |                            runtime-store factory, SQL-native decoupling
      |                            (DEFAULT = KVDB)
            +-- wallet-default-sqlite   flip DBConfig zero-value -> SQLite

## Why B, not the interim MVP (A)

The reconcile initially brought the orphan's MVP: `LegacyManagerVault` for BOTH
backends + placeholder `wallet_secrets`. That is a **hybrid** — because the
create path builds a legacy `waddrmgr` for *every* wallet, an MVP "sqlite"
wallet still carries a walletdb file for its secrets (`LegacyManagerVault` wraps
it). It runs, but it is walletdb-free only at the source level.

That hybrid existed **only because SQL-native secret genesis did not exist**.
That gap is now closed (`keyvault.CreateWalletSecrets`, done + tested), so the
sqlite backend can and must hold its own secrets — no waddrmgr sidecar. That is
B.

## Done so far

- **impl-secret-store** (tip `7facf0674`): added `keyvault.CreateWalletSecrets`
  (spendable + watch-only genesis mirroring `waddrmgr.Create`'s snacl scheme:
  `NewSecretKey` master key, fresh `GenerateCryptoKey` script/priv keys,
  `masterKey.Encrypt` each, `cryptoKeyPriv.Encrypt(rootKey.String())`) with 4
  passing tests (spendable roundtrip incl. HD-root parity, watch-only,
  wrong-passphrase -> `ErrInvalidPassphrase`, spendable-requires-root). Commit
  `e6388797f`, positioned after `wallet: route unlock and lock through
  keyvault`. All commits signed; builds. This is #1272's new content
  (force-push only at the end).
- **impl-runtime-store-6** (WIP branch `irs6-recon` @ `456727cd4` in
  `.worktrees/fix-batch-duplicate`): Group 1 = 4 kvdb adapter commits
  (`lifecycle`/`signer`/`sync`/`driver`). Group 2 = ONE provisional "reconcile"
  commit (DBConfig backend selection, runtime-store factory that constructs
  `sqlite.NewStore`/`pg.NewStore`/kvdb, kvdb-routed signer/lifecycle/syncer,
  `waddrmgr` scope-name + birthday-margin additions, test updates). `./wallet/`
  builds + all tests pass. **Currently the MVP hybrid — to be upgraded to B and
  split into clean commits.**

## B work plan (impl-runtime-store-6)

**Scope note.** "SQL backend" means any non-kvdb backend — **sqlite AND
postgres**. The decoupling is gated on `Backend != DBBackendKVDB`, so one code
path covers both; there is no sqlite-only branch. **sqlite** is the primary
acceptance-tested backend and the default-flip target; **postgres** rides the
identical path and is verified via the pg itest. Both SQL backends become
**legacy-store-free**: no walletdb file and no walletdb-backed `waddrmgr.Manager`
are created or opened. The wallet may still call small `waddrmgr` helper APIs for
deterministic, in-memory derivation and address encoding from SQL-loaded
xpub/account metadata.

1. **Recovery / address-lookahead derivation for SQL backends — reuse the
   wallet's OWN store-native encoder; no waddrmgr, no new export.** The wallet
   already derives + encodes typed addresses from an account xpub in
   `deriveAddressData` (`address_manager.go:318`): parse `AccountPubKey` ->
   `deriveChildKey`(branch) -> `deriveChildKey`(index) ->
   `addresstype.ToWallet(AddrType).AddrFromPubKeyBytes` -> `PayToAddrScript`.
   Pure in-memory, no walletdb, no waddrmgr — the **same path that generates real
   SQL addresses**, so lookahead is consistent by construction (no *waddrmgr*-
   parity test needed — same code path — but the helper + adapter are still
   unit-tested; see below).
   - **Extract it off the `Wallet` receiver.** `deriveAddressData` is a `*Wallet`
     method today; recovery/syncer must not need a `Wallet`. Extract its pure
     body into a **package-level helper** (or an injected derivation func) —
     e.g. `deriveStoreAddress(params db.AddressDerivationParams, chainParams
     *chaincfg.Params) -> (addr address.Address, script, pubKey []byte, err)` —
     and make `w.deriveAddressData` a thin wrapper over it. (Also surface the
     `address.Address` it computes internally at `:364`; recovery caches it for
     reverse lookup, and the current return only gives script+pubkey.)
   - **Seam.** Recovery's `BranchRecoveryState` (`recovery.go:713/852`) derives
     via `waddrmgr.AccountStore`; narrow that field to a minimal
     `DeriveAddr(account, branch, index) -> (addr, script, err)` interface with
     two impls: the existing `waddrmgr.AccountStore` (kvdb), and a thin SQL
     adapter that builds `db.AddressDerivationParams` and calls the extracted
     helper. `GetBranchState` (`recovery.go:227-231`, already nil-`addrMgr`
     tolerant) builds the SQL adapter from `AccountProperties.AccountPubKey` +
     `AddrSchema` (fields already on `waddrmgr.AccountProperties`) once
     `storeAccountProperties` (`syncer.go:2232-2241`) **populates both**
     `AccountProperties.AccountPubKey` (parsed from `info.PublicKey`) and
     `AccountProperties.AddrSchema` (from `info.AddrSchema`) — today it sets
     neither, and the SQL adapter needs both (xpub to derive, schema to pick the
     address type).
   - **Schema-type bridge (decide explicitly).** `AccountProperties.AddrSchema`
     is `*waddrmgr.ScopeAddrSchema`, but the helper wants `db.AddressType` /
     `db.AddressDerivationParams`. **Chosen: (b) convert in the SQL adapter** —
     map `waddrmgr.ScopeAddrSchema -> db.AddressType` per branch (the mirror of
     the `db -> waddrmgr` mapping `storeAccountProperties` already performs), the
     same local `waddrmgr -> db` conversion the adapter already does for `Scope`
     and the xpub. Alternative (a): carry `db.ScopeAddrSchema` alongside
     recovery's account props to avoid the round-trip (more invasive to
     recovery's init plumbing).
   - **Focused unit tests (still required — "no parity test" != "no tests").**
     Cover (i) the extracted `deriveStoreAddress` helper: a known account xpub +
     branch/index derives the expected address + script + pubkey across
     P2PKH / P2WKH / NP2WKH / P2TR; and (ii) the SQL recovery adapter: its
     `DeriveAddr` yields the same address/script the SQL address-generation path
     produces for the same `(account, branch, index)`, and the
     `waddrmgr.ScopeAddrSchema -> db.AddressType` conversion maps every scope
     correctly. These are package-level tests (no `Wallet`, no walletdb).
2. **Wire the genesis into create** — for SQL backends, populate `wallet_secrets`
   from `CreateWalletSecrets(privPass, rootKey, watchOnly)` (replaces the
   placeholder). Account seed/import already encrypt via `w.keyVault`, so account
   secrets stay consistent with `wallet_secrets` under `WalletVault`.
3. **Backend-selected keyvault** in `buildWallet` (+ `deprecated.go`): SQL
   backends -> `keyvault.NewWalletVault(store, walletID, watchOnly)`; kvdb ->
   `kvdb.NewLegacyManagerVault(...)`.
4. **Skip the legacy store for SQL backends** — `createLegacyStore` /
   `openLegacyStore` run for kvdb only. SQL wallets create/open no walletdb file
   and no walletdb-backed `waddrmgr.Manager`. Any `waddrmgr` usage in SQL mode is
   limited to pure helper code with inputs loaded from `db.Store`.
5. **Decouple from `waddrmgr`/`wtxmgr`** — with no legacy store, `w.addrStore`
   (`*waddrmgr.Manager`), `w.txStore` (`*wtxmgr.Store`), `w.legacyStore` are nil
   for SQL backends. Route the prod files that still call them through `db.Store`:
   signer legacy path (`fetchManagedPubKeyAddress`, `PrivKeyForAddress`), syncer,
   recovery (step 1), utxo/tx/account managers, notifications, and the
   create/load path. Backend-aware reads for master fingerprint / watch-only from
   the store.
6. **Load path** (`DBLoadWallet` / `buildWallet`) backend-aware: build the store
   without the waddrmgr for SQL backends.

## Acceptance criteria

- **Full sqlite roundtrip**: create -> **stop -> load the existing wallet** ->
  unlock (`WalletVault`) -> derive -> sign -> import -> **rescan/recovery**. The
  stop+load leg is essential — a create-only test would miss a lazy legacy-store
  open in `DBLoadWallet` / `buildWallet`; the rescan leg proves the store-backed
  lookahead seam (step 1) works without a walletdb-backed `waddrmgr.Manager`.
  This is the concrete "the SQL
  backend is actually used" gate.
- **Explicit no-legacy-file assertion**: the test asserts the kvdb/walletdb file
  does **not** exist on disk for a sqlite wallet **both after create and after
  load/rescan** — not merely that operations succeed.
- kvdb path unchanged and green — verified via `./wallet/` and
  `./wallet/internal/db/kvdb` package/unit tests (kvdb is NOT covered by the DB
  itests).
- SQL backends (sqlite + pg) verified via `wallet/internal/db/itest`.
- Per-commit `go build`/`go vet`; golangci clean per PR head; GPG-signed
  (`%G? = G`).

## Commit hygiene

Split the provisional Group-2 commit into logical, per-commit-compiling groups
(kvdb adapters [done] / `DBConfig` + config / runtime-store factory / recovery
derivation seam / create path / signer+syncer routing / backend-selected
keyvault + genesis wiring / waddrmgr decoupling / tests). Follow the repo
commit-hygiene rule: weighted added-line limit **150** (generated 0% / tests
50% / prod 100%); SQL + generated in their own commits; review-fixes folded into
introducing commits; no introduce-then-remove. Any commit that must exceed 150
is an **explicit, marked exception** with a one-line justification in the body —
the only expected one is the forced-atomic `DBConfig` type change, which ripples
to every `cfg.DB` user and cannot compile when split.

## Final branch (wallet-default-sqlite)

Flip `DBConfig` zero-value `Backend` KVDB -> SQLite + default-path tests. This is
the ONLY behavior flip.

## References / current state

- Genesis: `keyvault.CreateWalletSecrets` (`impl-secret-store` @ `e6388797f`).
- Orphan reference (the fuller integration to reconcile from):
  `impl-runtime-store-6` = `a3f0546ef`, `wallet-default-sqlite` = `b38a0e1dc`;
  backups on tags `bkp-txinputs-6` / `bkp-txinputs-wds` + the pushed orphan refs.
- `tx_inputs` / confirmed-conflicts feature is DROPPED (backed up), not carried.
- SHAs: `impl-secret-store` = `7facf0674` (reshaped, +genesis); WIP
  `irs6-recon` = `456727cd4` (Group 1 + provisional Group 2, green MVP-hybrid).
- Working worktree: `.worktrees/fix-batch-duplicate` on `irs6-recon`.
- No push until the whole stack is done + verified. Then push ONLY the three
  rewritten upper branches — `impl-secret-store`, `impl-runtime-store-6`,
  `wallet-default-sqlite` — to BOTH remotes (`origin` = yyforyongyu, `upstream` =
  btcsuite), each with `--force-with-lease=<branch>:<expected-remote-sha>` (per
  branch, per remote; capture the current remote SHAs first). The 5 lower PRs
  (`prep-runtime-3` .. `impl-runtime-store-5`) are UNTOUCHED and NOT pushed.
