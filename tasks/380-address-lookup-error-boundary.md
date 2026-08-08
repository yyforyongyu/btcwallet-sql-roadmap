# Task 380 — translate address lookup errors

Component: **AddressManager GetAddressInfo error boundary**.

Dependencies: **None**.

## Rationale

`AddressManager.GetAddressInfo` promises an error for an unknown wallet
address but currently returns `db.ErrAddressNotFound` directly.

## Details

- Define a wallet-owned `ErrAddressNotFound` and translate SQL and legacy
  address misses at `GetAddressInfo`.
- Preserve address context while wrapping only the public sentinel.
- Keep existing wallet-owned address DTOs unchanged; they do not leak internal
  DB types.
- Do not change address creation/import policy, notification behavior,
  derivation, script resolution, or signing.
- Do not absorb `ScriptForOutput` secret lifetime; that belongs to
  Task 383.

## Suggested fix

Add one address-lookup mapper at the Wallet boundary and update focused tests
to use the wallet sentinel instead of Store or waddrmgr errors.

## Verification

- External-package SQL and modern-kvdb tests prove an unknown address satisfies
  `errors.Is(err, wallet.ErrAddressNotFound)`.
- Invalid address encoding and unexpected Store errors are not reclassified as
  not-found.
- Existing AddressManager result types remain source-compatible.

## Definition of done

- `GetAddressInfo` owns its public not-found contract in wallet.
- No DTO, address mutation, derivation, or secret-resolution behavior changes.
