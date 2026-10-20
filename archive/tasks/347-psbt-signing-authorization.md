# Task 347 — authorize existing PSBT signatures

Component: **PsbtManager existing-signature validation**.

Type: **Bug fix**.

Dependencies: **[Task 346][t346]**.

## Rationale

`SignPsbt` can receive existing ECDSA, Taproot script-path, or Taproot key-path
signatures. A malformed or unrelated record must not suppress wallet signing,
survive as trusted output, or trigger private-key access before its transaction
and script commitments are proven.

## Details

- Validate the previous output, script position, derivation metadata, sighash,
  digest, effective public key, and signature before secret lookup or packet
  mutation.
- Bind Taproot script-path records to the exact validated leaf. Validate the
  singleton Taproot key-path record against the authorized output key.
- Preserve valid existing records and Task 346's base-key derivation metadata
  byte-for-byte. Reject malformed, duplicate, mismatched, or invalid records.
- A valid foreign record remains but does not suppress the wallet's own
  signature. A valid record for the wallet's effective key suppresses a
  duplicate.
- Validate a clone and leave the caller packet unchanged on every error.
- Exclude new signature generation, tweak application, callback replacement,
  funding, leasing, finalization, and Vault API changes. [Task 469][t469] owns
  effective-key signing and caller migration.

## Suggested fix

Build one side-effect-free authorization result from each existing signature
record and the validated input context. The private helper shape is flexible;
the observable cutoff before key access and packet mutation is the contract.

## Verification

- Cover valid and invalid ECDSA, Taproot script-path, and Taproot key-path
  records, including wrong keys, outputs, leaves, sighashes, and digests.
- Prove invalid records fail before key lookup and caller mutation.
- Prove valid foreign records remain, matching wallet records suppress a
  duplicate, and all base-key derivation records remain byte-for-byte stable.

## Definition of done

`SignPsbt` trusts or preserves an existing signature only after complete
wallet-owned authorization against the input context, without adding signing,
tweak, or finalization scope.

[t346]: 346-psbt-funding-metadata.md
[t469]: ../../tasks/469-sign-psbt-with-effective-keys.md
