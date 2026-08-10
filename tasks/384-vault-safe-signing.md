# Task 384 — route safe signing through the Vault

Component: **Signer and primitive key-use Vault operations**.

Dependencies: **[Task 337][t337], [Task 352][t352], [Task 354][t354], and
[Task 383][t383]**.

## Rationale

Safe Signer methods currently receive private keys from Wallet helpers after
runtimeCache secret reads and raw decryption. Private-key derivation, ECDH,
tweaking, and primitive signing can remain inside Vault while Wallet continues
to own transaction and digest policy.

## Details

- Migrate maintained safe `Signer` methods: `DerivePubKey`, `ECDH`,
  `SignDigest`, `ComputeUnlockingScript`, and `ComputeRawSig`.
- Consume Task 337's final selector-bearing public derivation request and
  checked numeric `Signer.DerivePubKey` adapter. Task 337 owns those public
  shapes and adapter semantics; this task owns only moving their key resolution
  and derivation body into the Vault and must not redefine either contract.
- Add only the semantic key locators and typed Vault operations exercised by
  those concrete methods. Select account keys by semantic scope and optional
  BIP44 number/name plus branch/index, and raw imports by script pubkey. Never
  use a public or internal cross-backend AccountID abstraction.
- Keep private keys inside Vault for derivation, ECDH, tweak, and signing.
  Return only public keys, fixed-size shared secrets, or signature values.
- Wallet computes and validates transaction digests, sighash semantics,
  scripts, witnesses, and spend policy before invoking primitive Vault signing.
- Replace maintained private-key callback tweakers with the smallest immutable
  typed tweak descriptor. Validate it before secret lookup and apply it inside
  Vault. Leave deprecated callback APIs unchanged.
- Reuse Task 383 for script material rather than adding a second
  script-decryption route.
- Preserve lock, watch-only, not-mine, imported-XPub public-only, and signature
  serialization behavior.
- Do not add PSBT authorization from Task 347 or UnsafeSigner raw extraction.
- Do not add the proposed public `wallet/signing` package or digest request
  types from PR #1279 unless Task 354 records an exact accepted merge commit.
  If no artifact is accepted, keep primitive request types internal to
  keyvault and introduce them with this concrete consumer.

## Suggested fix

Convert each existing Signer request into a narrow internal Vault locator and
primitive operation after Wallet validation. Move account/address secret
lookup, decryption, derivation, tweak application, and signing under the Vault
operation guard. Keep Wallet assembly unchanged.

## Verification

- External-package Signer vectors cover public derivation, ECDH, ECDSA,
  Schnorr, tweaked Schnorr, unlocking scripts, and raw signatures without a
  private key or encrypted DTO crossing the mock boundary.
- Dependency spies prove invalid requests fail before Vault secret lookup.
- Lock/concurrency and zeroing tests cover success, signer error, tweak error,
  and panic.
- Numberless imported XPub requests remain public-only and cannot fall back to
  account zero.
- API inspection finds no public backend AccountID locator or duplicate
  unaccepted PR #1279 request package.

## Definition of done

- Every maintained safe Signer operation uses typed Vault key operations and
  receives no private key or encrypted Store secret.
- Wallet retains transaction, sighash, witness, and policy ownership.
- No PSBT authorization, UnsafeSigner extraction, or unrelated script/address
  mutation is included.

[t337]: 337-imported-xpub-derivation.md
[t352]: 352-kvdb-combined-passphrase-rotation.md
[t354]: 354-vault-owned-secret-resolution.md
[t383]: 383-vault-output-script-resolution.md
