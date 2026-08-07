# Task 284: Block Wallet Use During Create Initialization

Status: Archived. Completed.

Archive note: `Manager.Create` now uses a create-in-progress guard and builds a
wallet with `buildWallet` before publishing it with `publish` only after
post-load initialization completes. `Manager.Load` waits via `awaitCreate` while
a create is in flight. Verification passed:

- `go test ./wallet -run 'TestManagerCreateRejectsLoadedWallet|TestManagerCreateRejectsStartedWallet|TestManagerCreateReleasesGuardOnFailure|TestResolveScanTargets|TestStoreScanHorizonsTargeted|TestGetUtxo|TestFilterEligibleOutputsExcludesNonSpendable|TestImportPrivateKeyKVDBSucceeds'`

Source review: `.lgtm/reviews/gpt-review-tasks.md`, finding 1.

## Finding

`Manager.Create` loads and publishes a wallet into the manager cache before the
post-load create initialization finishes. For SQL wallets, that initialization
includes default-account seeding and initial account imports.

Because `Manager.Load` returns cached wallets immediately, a concurrent caller
can observe and start/unlock the wallet while `Create` is still running
`seedDefaultAccounts`. That seeding path unlocks and later re-locks the shared
key vault, so a concurrently started or unlocked wallet can have its vault state
changed underneath it.

## Rationale

The manager cache should only expose wallets that are ready for normal runtime
use. A wallet that has durable rows but has not completed idempotent post-load
initialization is still in a create-only state.

The current flow needs the loaded wallet object because seeding uses wallet
methods and the key vault. However, publishing that object before seeding
completes creates a lifecycle race between create-time initialization and normal
wallet use.

## Detailed Fix Plan

1. Introduce a manager-private create-in-progress guard keyed by wallet name.
   This can be a small map protected by the manager mutex, for example a
   `map[string]chan struct{}` or equivalent condition mechanism.

2. Enter the create-in-progress guard before opening or creating stores for a
   wallet name. Release the guard only after `Create` either publishes a fully
   initialized wallet or fails and cleans up any freshly loaded instance.

3. Make `Load` handle a name that is currently being created. Preferred
   behavior is to wait until the create guard is released, then re-check the
   manager cache and either return the finished wallet or proceed with a normal
   load if create failed before publishing.

4. Split the current `Load` implementation into a private helper that can build
   a `Wallet` without publishing it, plus the public `Load` wrapper that
   publishes fully loaded wallets. `Create` should use the private helper while
   the create guard is held.

5. Run default-account seeding and initial account import replay on the
   unpublicized wallet. If this succeeds, publish the wallet atomically while the
   create guard still prevents other callers from racing in.

6. If post-load initialization fails, call `discardUnstarted` only for the
   freshly loaded create-owned wallet, then release the create guard. No partial
   wallet should remain in the public manager cache.

7. Keep the idempotent retry behavior: durable rows that already exist but lack
   post-load initialization must still be completable by a corrected retry.

## Tests

- Add a test that blocks `seedDefaultAccounts` with a test hook after the wallet
  is constructed but before initialization completes. A concurrent `Load` must
  not return a usable wallet until the create finishes.
- Add a test that concurrent `Start`/`Unlock` cannot run against a wallet while
  `Create` is still seeding default accounts.
- Add a failure-path test proving that a seeding error does not leave the wallet
  in `m.wallets` and that a corrected retry can still complete.
- Re-run existing create retry tests to prove partial-create recovery still
  works.

## Verification

- `go test ./wallet -run 'TestManagerCreate'`
- `go test ./wallet -run 'TestWalletStart|TestWalletUnlock'`
- `go test ./wallet`

## Acceptance Criteria

- A wallet is not visible through `Manager.Load` until `Create` post-load
  initialization has completed.
- Concurrent callers cannot start or unlock the wallet while `Create` is
  seeding default accounts or importing initial accounts.
- Create retries over durable but partially initialized wallets remain
  supported.
- Failed create initialization does not leave an open wallet in the manager
  cache.
