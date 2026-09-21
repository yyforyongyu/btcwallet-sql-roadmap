# Task 384 — route safe signing through the Vault

Component: **Signer and primitive key-use Vault operations**.

Type: **Refactor**.

Dependencies: **[Task 337][t337], [Task 354][t354],
[Task 383][t383], and [Task 469][t469]**.

## Rationale

Safe Signer methods currently receive private keys from Wallet helpers after
runtimeCache secret reads and raw decryption. Private-key derivation, ECDH,
tweaking, and primitive signing can remain inside Vault while Wallet continues
to own transaction and digest policy.

## Details

- Scope the ownership move to SQL. Keep kvdb's existing signing/credential
  path unchanged; its containment work remains deferred in [Task 352][t352].
- Migrate secret use in `ECDH`, `SignDigest`, `ComputeUnlockingScript`, and
  `ComputeRawSig`. `DerivePubKey` already uses public account XPub data and
  stays outside Vault, including Task 337's imported-XPub behavior.
- Reuse existing request identity; add only internal Vault operations needed
  by these consumers. Select account keys by semantic scope and optional
  BIP44 number/name plus branch/index, and raw imports by script pubkey. Never
  use a public or internal cross-backend AccountID abstraction.
- Keep private keys inside Vault for derivation, ECDH, tweak, and signing.
  Return only public keys, fixed-size shared secrets, or signature values.
- Wallet computes and validates transaction digests, sighash semantics,
  scripts, witnesses, and spend policy before invoking primitive Vault signing.
- Consume Tasks 347 and 469's authorization and effective-key signing
  contracts. This later refactor replaces primitive key and tweak ownership
  without changing check ordering, authorization results, or which inputs may
  be signed.
- Consume Task 469's immutable authorization-bound tweak description,
  validate it before secret lookup, and apply it inside Vault. Do not restore
  the removed callback through a wrapper.
- Reuse Task 383 for script material rather than adding a second
  script-decryption route.
- Preserve lock, watch-only, not-mine, imported-XPub public-only, and signature
  serialization behavior.
- Do not add or redefine PSBT authorization from Task 347 or UnsafeSigner raw
  extraction.
- Add no public signing package or parallel request API. Reuse suitable merged
  work from PR #1279 where available, with Task 354 identifying overlap; keep
  any new primitive operation shapes internal to keyvault.
- Remove helpers and secret-access seams made unused by this move, with
  required test/mock updates. Retain shared seams still needed by UnsafeSigner
  or unchanged kvdb behavior; add no replacement legacy adapter.

## Suggested fix

Convert each existing Signer request into a narrow internal Vault locator and
primitive operation after Wallet validation. Move account/address secret
lookup, decryption, derivation, tweak application, and signing under the Vault
operation guard. Keep Wallet assembly unchanged.

## Verification

- Public Signer vectors retain results on all maintained backends. On SQL,
  ECDH, ECDSA, Schnorr, tweaked Schnorr, unlocking scripts, and raw signatures
  expose no private key or encrypted DTO across the Wallet boundary.
- Dependency spies prove invalid requests fail before Vault secret lookup.
- Lock/concurrency and zeroing tests cover success, signer error, tweak error,
  and panic.
- Numberless imported XPub requests remain public-only and cannot fall back to
  account zero. Public derivation continues to work while locked without
  requiring Vault secret access.
- API inspection finds no public backend AccountID locator or duplicate
  unaccepted PR #1279 request package.

## Definition of done

- Every secret-dependent SQL safe Signer operation uses typed Vault operations
  and receives no private key or encrypted Store secret. Public-only
  derivation and kvdb remain unchanged.
- Wallet retains transaction, sighash, witness, and policy ownership.
- No PSBT authorization, UnsafeSigner extraction, or unrelated script/address
  mutation is included.

[t337]: ../archive/tasks/337-imported-xpub-derivation.md
[t352]: ../archive/tasks/352-kvdb-combined-passphrase-rotation.md
[t354]: 354-vault-owned-secret-resolution.md
[t383]: 383-vault-output-script-resolution.md
[t469]: 469-sign-psbt-with-effective-keys.md
