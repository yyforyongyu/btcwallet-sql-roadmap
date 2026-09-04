# Task 469 — sign PSBT inputs with effective keys

Component: **PsbtManager and Signer effective-key signing**.

Type: **Bug fix**.

Dependencies: **[Task 347][t347] and [Task 390][t390]**.

## Rationale

After existing records are authorized, `SignPsbt` must bind each new signature
to the key that actually satisfies the validated script. The current callback
tweak seam can mutate authorization after validation and can confuse the
persisted base key with its effective signing key.

## Details

- Complete all secret-independent account, path, script, sighash, and tweak
  authorization before key lookup.
- Replace the mutable tweak callback with the smallest immutable description
  required by maintained callers and mocks. Do not keep a compatibility
  wrapper solely for the old callback.
- Apply the authorized tweak exactly once after resolving the base key and
  verify that the effective key matches the script commitment before signing.
- Emit ECDSA and Taproot script-path records under their effective key
  identities. For Taproot key-path spends, validate the internal and output
  keys and emit only the signature bytes.
- Preserve the base-key derivation records and the existing-signature behavior
  from Task 347. Keep caller packets unchanged on error.
- Exclude FinalizePsbt, funding, leasing, metadata merging, and new Vault APIs.

## Suggested fix

Carry one immutable authorization result from PsbtManager to the maintained
Signer. Its private representation is flexible if callers cannot alter the
authorized tweak and every emitted signature is bound to the validated
effective key.

## Verification

- Cover untweaked and tweaked ECDSA, SegWit, custom-scope, and Taproot inputs.
- Prove invalid metadata fails before key lookup; tweak and effective-key
  mismatches fail before signing or caller mutation.
- Prove each tweak runs once, emitted records use the effective identity, and
  a second pass preserves base derivations and emits no duplicate.
- Compile every maintained caller and mock against the final immutable input.

## Definition of done

`SignPsbt` emits only signatures whose effective keys satisfy the authorized
input, and maintained signing retains no mutable tweak callback or
compatibility wrapper.

[t347]: 347-psbt-signing-authorization.md
[t390]: 390-custom-scope-account-creation.md
