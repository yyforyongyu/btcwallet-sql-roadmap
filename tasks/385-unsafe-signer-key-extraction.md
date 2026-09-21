# Task 385 — isolate UnsafeSigner key extraction

Component: **UnsafeSigner raw private-key boundary**.

Type: **Refactor**.

Dependencies: **[Task 384][t384]**.

## Rationale

`UnsafeSigner.DerivePrivKey` and `GetPrivKeyForAddress` intentionally export
raw private keys. They must remain explicit exceptions rather than forcing
safe Signer operations to expose keys or pretending Vault controls a key after
return.

## Details

- Follow Task 384's SQL-only ownership move; leave kvdb key extraction
  unchanged rather than importing deferred credential work.
- Add the smallest explicitly unsafe Vault extraction operation needed by the
  two maintained UnsafeSigner methods.
- Reuse the semantic locators and backend-local resolution established by
  Task 384; do not reintroduce runtimeCache secret reads or a raw
  general-purpose decrypt method.
- Return an isolated private-key value and document that the caller owns and
  must destroy it. Do not claim post-return lifetime or zeroing by Vault.
- Limit caller changes to the two maintained UnsafeSigner methods. The
  context-free `PrivKeyForAddress` wrapper serves deprecated callers and is
  outside this task; its removal is not required.
- Preserve watch-only, locked, not-mine, and imported-XPub failure behavior.
- Do not use this extraction operation from safe Signer or PSBT code.

## Suggested fix

Expose one narrowly named internal extraction capability, implement it in the
SQL Vault backend, and make only SQL UnsafeSigner call it. Keep safe callers
on the non-exporting operations without expanding into deprecated callers.

## Verification

- Public UnsafeSigner itests prove both methods return usable isolated keys and
  document caller destruction.
- Mutation or destruction of one returned key does not alter a later result or
  Vault state.
- Static call-site inspection proves no safe Signer, PSBT, AccountManager, or
  AddressManager path calls the extraction operation.
- Locked, watch-only, not-mine, and imported-XPub vectors preserve existing
  public errors.

## Definition of done

- Raw private-key return is confined to UnsafeSigner as a documented caller-
  owned exception.
- SQL safe signing remains entirely non-exporting, kvdb stays unchanged, and
  no general raw decrypt API is introduced.

[t384]: 384-vault-safe-signing.md
