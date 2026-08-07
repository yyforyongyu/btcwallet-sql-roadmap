# Task 285: Reject Loaded Wallet Create Before Store Open

Status: Archived. Completed.

Archive note: `Manager.Create` now calls `beginCreate` before store open/create
work. `beginCreate` rejects an already cached wallet with `ErrWalletExists` and
waits for any in-flight create of the same name instead of opening stores
concurrently. Verification passed:

- `go test ./wallet -run 'TestManagerCreateRejectsLoadedWallet|TestManagerCreateRejectsStartedWallet|TestManagerCreateReleasesGuardOnFailure|TestResolveScanTargets|TestStoreScanHorizonsTargeted|TestGetUtxo|TestFilterEligibleOutputsExcludesNonSpendable|TestImportPrivateKeyKVDBSucceeds'`

Source review: `.lgtm/reviews/gpt-review-tasks.md`, finding 2.

## Finding

`Manager.Create` can open or create the legacy store before checking whether the
same manager already has the wallet loaded. When the wallet is already loaded,
that can block or fail on the bbolt database lock before the method reaches the
logic that would classify the operation as `ErrWalletExists`.

## Rationale

A duplicate create against a live wallet managed by the same `Manager` is not a
partial-create retry. It is an over-create of an already usable wallet and
should fail quickly and predictably without touching the underlying stores.

Store I/O before the cache check also makes behavior backend-dependent: kvdb can
hit a database lock while SQL backends may continue farther into runtime row
handling.

## Detailed Fix Plan

1. After validating `Config` and `CreateWalletParams`, check the manager cache
   for `cfg.Name` before calling `createLegacyStore` or opening any runtime
   store.

2. If the wallet name is present in `m.wallets`, return
   `fmt.Errorf("%w: %q", ErrWalletExists, cfg.Name)` immediately.

3. Coordinate this with task 284's create-in-progress guard. The guard should
   distinguish these cases:

   - already loaded in cache: return `ErrWalletExists` immediately;
   - create in progress for the same name: wait or return a deterministic
     exists/in-progress error, but do not open the stores concurrently;
   - durable rows exist on disk but no live wallet is cached: continue into the
     existing partial-create retry logic.

4. Preserve corrected retry behavior for durable partial creates. The early cache
   check must only reject live managed wallets, not wallets that merely exist on
   disk.

5. Ensure `discardUnstarted` is never called on an already cached wallet from
   this path.

## Tests

- Create and keep a wallet loaded in one `Manager`; call `Create` again on the
  same manager with matching params. Assert `ErrWalletExists` and that the
  original wallet remains usable.
- Repeat the same test with the wallet started, and verify the started wallet's
  stores are not closed by the failed duplicate create.
- Add a regression that the second create does not attempt to open the bbolt
  database while the first wallet's store handles are still open. This can be
  observed by requiring `ErrWalletExists` rather than a backend lock/open error.
- Keep the existing fresh-manager duplicate-create tests for durable fully
  created wallets.

## Verification

- `go test ./wallet -run 'TestManagerCreateRejectsExisting'`
- `go test ./wallet -run 'TestManagerCreateRetry'`
- `go test ./wallet`

## Acceptance Criteria

- Duplicate `Create` against a wallet already loaded by the same manager returns
  `ErrWalletExists` before any store open/create work.
- The existing live wallet remains registered and usable after the failed
  duplicate create.
- Partial-create retries that require store access are still allowed when no
  live wallet is cached.
