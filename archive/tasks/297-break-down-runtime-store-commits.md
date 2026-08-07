# Task 297: Break down the runtime-store commits + remove walletdb from public wallet code

Status: Open. Executing. Supersedes the earlier "Manager backend-dispatch
facade / SQL-native" framing — that was a wrong turn; dropped.

## Goal

The public wallet components (`wallet/manager.go`, `wallet/signer.go`,
`wallet/db_ops.go`, …) must not call `walletdb` directly. All persistence goes
through `w.store` (the `db.Store` interface); the `walletdb.View`/`.Update`
transactions live ONLY in the kvdb implementation (`wallet/internal/db/kvdb/`)
behind that interface. If the interface lacks a method the wallet needs, add it.
`account_manager.go` is the model (already 100% `w.store`, zero `walletdb`).

This is also the vehicle for the original job: break the oversized `456727cd4`
(and `98c492392`, `f501e9ce4`) into hygienic commits. **The Manager walletdb
cleanup is folded INTO the `456727cd4` split — not a separate after-the-fact
fix.**

## How we got here

Started to break down `456727cd4` (+4,688). Detours (SQL-native Manager, kvdb
dispatcher facade) were wrong turns — the real goal is walletdb-removal via the
store interface, and kvdb stays (behind `db.Store`). Along the way we also found
a stray WIP commit and an empty commit to clean up.

## Current stack + issues (all LOCAL, not pushed)

- **impl-secret-store** (`7facf0674`): `ac873f2bb` "construct legacy keyvault
  shim" is an **empty cherry-pick** (0 files changed — the change already
  existed via another path) → drop it when rewriting. Also fold the two lint
  fixes already staged in branch `secret-rebuild` (vault_genesis lll →
  `e6388797f`; psbt wsl → tip `7facf0674`).
- **impl-runtime-store-6** (`a0f4017f7`): stray `"WIP task-296 lint fixes
  (temp)"` sits on top of `f501e9ce4` → rebuild the branch off it (baseline
  `f501e9ce4` = `bkp296-r6-tip`).
- Oversized commits to split: `456727cd4` (+4,688), `98c492392` (+579 / ~409
  weighted), `f501e9ce4` (+464 / ~396 weighted). **Leave `d753fd46d`** (+175 —
  over ideal, under hard cap).
- **wallet-default-sqlite** (`b004dce7b`, clean: +47 / +77) → rebase onto the
  rebuilt tip afterward.
- `f501e9ce4` `manager.go` direct walletdb is minimal: only
  `walletdb.ErrDbDoesNotExist` (~line 1339) + the import (fingerprint read is
  already store-routed on impl-runtime-store-5). Route that sentinel through a
  kvdb/store error and drop the import as part of the split.

## Priority

1. Drop `ac873f2bb` (empty) when rewriting impl-secret-store; fold the two lint
   fixes.
2. Rebuild impl-runtime-store-6 off the stray `a0f4017f7` (baseline
   `f501e9ce4`).
3. Split `456727cd4` first — **with the Manager walletdb cleanup folded in**.
4. Split `98c492392`.
5. Split `f501e9ce4`.
6. Leave `d753fd46d`.

## Split: `456727cd4`

- wallet: add runtime store config
- wallet: add runtime store open helpers
- wallet: build wallets from runtime stores
- wallet: coordinate manager create/load publication
- wallet: create wallets through runtime store
- wallet: route controller lifecycle through store
- wallet: route syncer runtime setup through store
- wallet: remove migrated db_ops helpers
- + separate test commits where an impl commit would exceed the cap.

Fold the `manager.go` walletdb cleanup (the `ErrDbDoesNotExist` sentinel + the
`walletdb` import) into the relevant commit above.

## Split: `98c492392`

- wallet: add store recovery derivation seam
- wallet: use store recovery derivation during init
- wallet: test store-backed recovery derivation

## Split: `f501e9ce4`

- wallet: create SQL runtime wallets without sidecar
- wallet: build SQL wallets without legacy stores
- wallet: handle nil legacy managers on SQL wallets
- wallet: test SQL no-sidecar wallets

## Cap / verification

- Per-commit weighted added-line cap: ideal ≤150, hard ≤400 for impl-runtime-store-6 (per maintainer 2026-07-11; ≤200 elsewhere) (generated ×0 /
  tests ×0.5 / prod ×1); deletions uncapped.
- Per-commit `go build` + test-compile (`go test -run=NONE`).
- golangci clean per PR head (impl-secret-store / impl-runtime-store-6 /
  wallet-default-sqlite).
- GPG-signed (`%G? = G`).
- Invariant: the split commits sum to the intended end-state tree (f501e9ce4
  content + the two secret-store lint fixes + the small manager.go
  walletdb-cleanup, minus the empty `ac873f2bb`).
- Rebase `wallet-default-sqlite` onto the rebuilt `impl-runtime-store-6`.

## Backups / WIP branches

`bkp296-r6-tip` (=`f501e9ce4`), `bkp296-secret-tip` (=`7facf0674`); WIP:
`wipfix`, `secret-rebuild`, `runtime-carve`. Worktree
`.worktrees/fix-batch-duplicate`.

## Relation to task 295

This realizes task 295's "remove direct walletdb DB API from top-level
`wallet/`" goal (295 was mis-archived this session; its per-file
residual-walletdb table — syncer/manager/db_ops/signer — is a useful reference).
295's SQL-config / 3-DB-itest scope is separate and later.
