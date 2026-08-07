# Task 299: Clean PR #1272/#1291 history and close review threads

Status: Completed and pushed. Branches rebuilt and verified, GitHub Actions CI
passed, and all outstanding review threads were replied to and resolved.

## Completion

- `task-299-secret` = `57997901554c825dcca7744aaab1620e47f1d604`
  - 18 signed commits over `sql-wallet`.
  - Final behavior matches the prior PR head, plus the requested signer TODO and
    formatting-only test cleanup.
- `task-299-runtime` = `99030fee69201922886215bc04d55b16207a27c1`
  - 19 signed commits over `task-299-secret`.
  - Implements the durable create-complete marker, backend-safe full reset,
    coordinated shared SQL resets, reference-counted SQL stores, Store-only
    account/benchmark paths, shared Stop result, and strict retry identity
    handling.
- `wallet-default-sqlite` =
  `2684a5712731d30a831025fa2d48549d74c9eac3`
  - Rebased without patch changes over the final runtime-store head.
- Backup refs remain at `backup/task-299-secret-start`,
  `backup/task-299-runtime-start`, and `backup/task-299-sqlite-start`.
- Local, `origin`, and `upstream` refs are updated to the rebuilt heads,
  including `impl-secret-store`, `impl-runtime-store-6`, and
  `wallet-default-sqlite`.

Verification completed:

- Every commit builds and test-compiles.
- `make unit` passed.
- `make unit-race pkg=wallet` passed.
- `make lint-check` passed with zero issues.
- `make sqlc-check` passed.
- SQLite, PostgreSQL, and kvdb/btcd integration checks passed during the runtime
  rebuild.
- PR #1272 GitHub Actions passed:
  `https://github.com/btcsuite/btcwallet/actions/runs/29839394920`.
- PR #1291 final GitHub Actions passed:
  `https://github.com/btcsuite/btcwallet/actions/runs/30029106723`.
- PR #1292 final GitHub Actions passed:
  `https://github.com/btcsuite/btcwallet/actions/runs/30029106856`.
- All commit signatures report good, subjects are under 50 columns, and bodies
  are wrapped to 72 columns.
- All 7 outstanding PR #1272 threads and all 12 outstanding PR #1291 threads
  received evidence-based replies and were resolved. Both PRs now report zero
  unresolved review threads.

This task supersedes active Tasks 296-298, `deferred-tasks.md`, and the prior
GPT review note. Those files are archived alongside the older task history.

## Goal

Finish the two lower upper-stack PRs as reviewable, bisectable commit series:

- PR #1272: `impl-secret-store` -> `sql-wallet`
- PR #1291: `impl-runtime-store-6` -> `impl-secret-store`

Preserve the current end-state behavior, fix the remaining real review bugs,
fold fixes into the commits that introduced the behavior, and resolve every
current unresolved thread only after its code/history change is present.

Do not push or resolve GitHub threads until the rebuilt stack is green and the
user explicitly requests it.

## Initial remote state

### PR #1272

- Head: `9165468d5a0ca658786708a6f13e0267da43e3e8`
- Base: `2ea76dfc5e85acf84014a21e4813cb23945250fa`
- Draft, mergeable, clean merge state, substantive CI green.
- 14 commits, 7 unresolved threads.

### PR #1291

- Head: `f6d677d7cc3b2ee4f6557439475bec03b425b0cd`
- Base: `9165468d5a0ca658786708a6f13e0267da43e3e8`
- Draft, mergeable, clean merge state, substantive CI green.
- 12 commits, 57 changed files, +12,396/-3,823.
- 6 unresolved threads.

## Non-negotiable history rules

- Keep each branch below 20 commits over its immediate base. Target #1291:
  exactly 19 commits after rebuilding.
- Every commit must compile and test-compile independently.
- No review-labeled fixup/WIP commits in the final history.
- No introduce-then-remove APIs when the final dependency can be ordered first.
- SQL source and generated sqlc output stay together in generated-only commits.
- Commit subjects use `subsystem: summary`, stay under 50 characters, and commit
  bodies wrap at 72 columns.
- Go lines stay within 80 columns, counting tabs as 8 spaces.
- Weighted added-line target: ideal <=150, hard <=400. Production counts 1.0,
  tests 0.1, generated/sqlc 0. A forced-atomic exception must be explained in
  the commit body and still contain only one conceptual migration step.
- Sign final commits with `-S`.

## PR #1272 actions

### Address unresolved threads

1. Reorder `4b7fc371` before `b692e8c6`, or fold the final legacy vault
   passphrase adapter into the shim commit. The shim must not introduce an
   unsupported `ChangePassphrase` stub that a later commit removes.
2. Resolve the kvdb `ChangePassphrase` thread as implemented only after the
   reordered history contains the real `Manager.ChangePassphrases` delegation.
3. Resolve the controller `DBPutPassphrase` thread as implemented: the final
   controller calls `w.keyVault.ChangePassphrase`.
4. Add the requested `TODO(yy)` above `deriveStoredAccountChildKey`, noting that
   account-child secret derivation should eventually move behind keyvault.
5. Resolve the error-driven signer-routing thread as implemented: the final
   `derivePathPrivKey` deterministically uses Store + keyvault.
6. Split `1ac536261` (`wallet+db: add account/address secret Go surface`). It
   changes 22 files and adds 1,825 lines across two separate Store surfaces.
7. Reply to the `keyvaultmock` thread with the import-cycle rationale: in-package
   keyvault tests import `bwtest/mock` for Store; keeping Vault in that package
   makes the mock package import keyvault and creates a cycle. The separate
   `keyvaultmock` package is required unless keyvault tests are externalized.

### Split `1ac536261`

Produce focused commits, preserving generated SQL in `e7e1fa240`:

1. `wallet+db: add account secret adapters`
   - shared account-secret selector/validation;
   - pg/sqlite/kvdb adapters;
   - focused unit and DB integration tests.
2. `wallet+db: add address secret adapters`
   - shared address-secret selector/validation;
   - pg/sqlite adapters;
   - focused unit and DB integration tests.
3. `kvdb: expose imported address secrets`
   - kvdb managed-address secret adapter;
   - encrypted key/script semantics and focused tests.

If dependency order requires interface wiring after all adapters, put the
interface addition in the last of these commits rather than creating a temporary
stub.

## PR #1291 correctness fixes

### 1. Preserve genuine partial-create replay

The unresolved `manager.go:1002` comment is valid. Current code treats every
pre-existing wallet row with a missing requested initial account as an
over-create, and its own comment admits it cannot distinguish a genuine partial
create from adding a new account later.

Implement a durable create-complete marker:

- New wallets start incomplete before default/initial account seeding.
- Mark complete only after all post-create initialization succeeds.
- A retry against an incomplete wallet may replay missing initial accounts after
  verifying existing account identity.
- A create against a complete wallet returns `ErrWalletExists` before adding any
  account.
- Existing migrated wallets default to complete.
- Provide equivalent SQL and kvdb semantics and positive/negative tests.

Fold this into the initial-account retry commit; do not add a tip fixup.

### 2. Use a backend-safe full reset

The unresolved `syncer.go:703` comment is valid. The full-reset path passes a
non-zero birthday height to kvdb `RollbackToBlock`, whose fork lookup probes
every absent height down to zero while holding `writeMu` and a walletdb write
transaction.

- Use the explicit height-zero full-reset path, or add a dedicated Store reset
  API with efficient backend implementations.
- Preserve the intended clearing of sync tip, birthday block, transactions, and
  recovery state.
- Add a kvdb regression test with a high non-zero birthday and sparse retained
  history; assert the reset does not perform a linear missing-height walk.
- Fold the change into the syncer/runtime Store commit that introduces the full
  reset behavior.

### 3. Remove the legacy `addrStore` preflight

The `ListAccountsByScope` nil-check thread should not be answered by allowing a
panic: SQL wallets intentionally have no legacy address manager.

- Remove the `w.addrStore.FetchScopedKeyManager` preflight entirely.
- Let `w.store.ListAccounts` own scope validation for both SQL and kvdb.
- Add Store-level invalid-scope validation if needed to preserve the API error.
- Reply that nil is valid for SQL, then resolve after the backend-neutral path
  lands.

### 4. Keep benchmark setup on Store

The `benchmark_helpers_test.go:384` concern is valid directionally. `cfg.DB` is
now a `DBConfig`, but using embedded `w.db` reintroduces direct walletdb access.

- Change `setSyncedToHeight` to call `w.store.UpdateWallet` with the wallet ID and
  target block.
- Keep benchmark helpers backend-neutral and remove the added walletdb-specific
  imports.
- Fold this into the runtime construction/test migration commit.

### 5. Rewrap long Go lines

Rewrap the four reported `manager.go` lines around 929, 933, 999, and 1000.
Run the repository line-length/lint check over every touched Go file.

## PR #1291 history rebuild

The current `ee50c6a9` is not an acceptable atomic commit: 38 files,
+6,954/-3,453, and its body describes configuration, construction, signer,
syncer, recovery identity, test harness migration, and `db_ops.go` deletion.

Retain the focused commits where possible:

- `ecf174850` kvdb lifecycle adapters
- `6429b54ad` birthday margin export
- `3ac4ff398` manager itest config
- `81ee4edf9` shutdown teardown
- `d2837f381` concurrent Load
- `ff8584124` monotonic sync tip
- `f8069e21c` TxStatus export
- `7737d5025` last-good sync tip
- `9ea038f49` generated watch-output query

Replace `ee50c6a9` with seven focused commits:

1. `wallet: add runtime store config`
2. `wallet: add runtime store open helpers`
3. `wallet: build wallets from runtime stores`
4. `wallet: coordinate manager publication`
5. `wallet: create through runtime stores`
6. `wallet: route sync runtime through Store`
7. `wallet: remove migrated db ops`

Placement details:

- Fold the unused legacy encrypted-seed removal into runtime open/create helpers.
- Fold bare-xpub `ModeImportExtKey` rejection into runtime create validation.
- Keep `db_ops.go` deletion separate from behavior changes.
- Move tests with the smallest commit they verify; use a separate test commit only
  when a production commit would exceed the hard weighted cap.

Split `58fdf5e6` into two commits:

1. `wallet: scope targeted store rescans`
2. `wallet: test targeted store rescans`

Refocus `f6d677d7` on initial-account identity and completion semantics only.
Move its unrelated bare-xpub and encrypted-seed changes into the owning runtime
create/open commits above.

This yields 19 commits: 9 retained + 7 runtime-construction splits + 2 targeted
rescan commits + 1 initial-account completion commit.

## Verification

For every rewritten commit:

```bash
go build ./...
go test ./... -run=NONE
```

At final PR heads:

```bash
make unit
make unit-race pkg=wallet
make itest-db db=sqlite
make itest-db db=postgres
make itest chain=btcd db=kvdb
make lint-check
make sqlc-check
```

Also verify:

- `git rev-list --count sql-wallet..impl-secret-store` is below 20.
- `git rev-list --count impl-secret-store..impl-runtime-store-6` is exactly 19
  or lower.
- No commit subject exceeds 50 characters.
- No commit body line exceeds 72 columns.
- No touched Go line exceeds 80 columns.
- Every final commit is signed.
- The rebuilt branch trees match the intended fixed end state.
- All review threads at the final heads are implemented or answered with the
  documented rationale before resolution.
