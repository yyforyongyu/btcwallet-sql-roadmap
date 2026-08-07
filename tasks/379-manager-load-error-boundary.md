# Task 379 — translate Manager load not-found errors

Component: **Manager Load public error boundary**.

Dependencies: **None**.

## Rationale

`Manager.Load` currently returns `db.ErrWalletNotFound` directly, and its
SQLite public-path test asserts the internal sentinel. Wallet absence is a
documented public outcome and needs a wallet-owned contract.

## Details

- Define a wallet-owned `ErrWalletNotFound` and translate SQL and modern-kvdb
  load misses to it once in `Manager.Load` or its backend boundary.
- Preserve wallet-name context while wrapping only the public sentinel.
- Do not change already-loaded Wallet reuse, concurrent-load behavior,
  lifecycle ownership, or backend selection.
- Keep duplicate-create classification in Task 330 and contextual Load
  cancellation in [Task 422][t422].
- Do not inventory AccountManager or AddressManager errors in this task.

## Suggested fix

Add the public sentinel and one narrow load-error mapper, then update the
existing real-SQLite test to assert the public contract.

## Verification

- An external-package real-SQLite test loads a missing wallet and observes
  `errors.Is(err, wallet.ErrWalletNotFound)` with no Wallet result.
- Equivalent supported backend misses map identically.
- A non-not-found backend failure is not reclassified.

## Definition of done

- Missing-wallet Load results expose only the wallet-owned sentinel.
- No Create, cancellation, concurrency, or lifecycle policy changes are
  included.

[t422]: 422-manager-load-cancellation.md
