# Task 380 — translate address lookup errors

Component: **AddressManager GetAddressInfo error boundary**.

Type: **Bug fix**.

Dependencies: **None**.

## Rationale

`AddressManager.GetAddressInfo` promises an error for an unknown wallet
address but currently returns `db.ErrAddressNotFound` directly.

## Details

- Define a wallet-owned `ErrAddressNotFound` and translate SQL and legacy
  address misses at `GetAddressInfo`.
- Preserve address context while wrapping only the public sentinel.
- Keep unrelated address DTO redesign outside this error-boundary task. Update
  maintained callers and mocks directly if the final error contract requires a
  compile-time change; do not add a legacy adapter.
- Do not change address creation/import policy, notification behavior,
  derivation, script resolution, or signing.
- Do not absorb `ScriptForOutput` secret lifetime; that belongs to
  Task 383.

## Suggested fix

Add one address-lookup mapper at the Wallet boundary and update focused tests
to use the wallet sentinel instead of Store or waddrmgr errors.

## Verification

- Public AddressManager itests on kvdb, SQLite, and PostgreSQL prove an unknown
  address satisfies `errors.Is(err, wallet.ErrAddressNotFound)`.
- Invalid address encoding and unexpected Store errors are not reclassified as
  not-found.
- The itest imports only the wallet-owned sentinel and no Store error or legacy
  wrapper.

## Definition of done

- `GetAddressInfo` owns its public not-found contract in wallet.
- No DTO, address mutation, derivation, or secret-resolution behavior changes.
