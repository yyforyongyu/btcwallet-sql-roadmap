# Task 385 — isolate UnsafeSigner key extraction

Component: **UnsafeSigner raw private-key boundary**.

Dependencies: **[Task 384][t384]**.

## Rationale

`UnsafeSigner.DerivePrivKey` and `GetPrivKeyForAddress` intentionally export
raw private keys. They must remain explicit exceptions rather than forcing
safe Signer operations to expose keys or pretending Vault controls a key after
return.

## Details

- Add the smallest explicitly unsafe Vault extraction operation needed by the
  two maintained UnsafeSigner methods.
- Reuse the semantic locators and backend-local resolution established by
  Task 384; do not reintroduce runtimeCache secret reads or a raw
  general-purpose decrypt method.
- Return an isolated private-key value and document that the caller owns and
  must destroy it. Do not claim post-return lifetime or zeroing by Vault.
- Keep the legacy context-free `PrivKeyForAddress` compatibility wrapper
  compiling by delegating to the same explicit unsafe path; do not otherwise
  expand deprecated wallet APIs.
- Preserve watch-only, locked, not-mine, and imported-XPub failure behavior.
- Do not use this extraction operation from safe Signer or PSBT code.

## Suggested fix

Expose one narrowly named internal extraction capability, implement it in the
maintained Vault backends, and make only UnsafeSigner and the compatibility
wrapper call it. Keep safe callers on the non-exporting operations.

## Verification

- External-package UnsafeSigner tests prove both methods return usable isolated
  keys and document caller destruction.
- Mutation or destruction of one returned key does not alter a later result or
  Vault state.
- Static call-site inspection proves no safe Signer, PSBT, AccountManager, or
  AddressManager path calls the extraction operation.
- Locked, watch-only, not-mine, and imported-XPub vectors preserve existing
  public errors.

## Definition of done

- Raw private-key return is confined to UnsafeSigner and the existing
  compatibility wrapper as a documented caller-owned exception.
- Safe signing remains entirely non-exporting and no general raw decrypt API is
  introduced.

[t384]: ../archive/tasks/384-vault-safe-signing.md
