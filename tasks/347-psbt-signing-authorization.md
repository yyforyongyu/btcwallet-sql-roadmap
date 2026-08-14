# Task 347 — authorize explicit PSBT signatures

Component: **PsbtManager and Signer**.

Type: **Bug fix**.

Dependencies: **[Task 346][t346] and [Task 390][t390]**.

## Rationale

Retain the current explicit SignPsbt authorization
contract: validate previous outputs and script commitments; infer spend type
from validated scripts; validate scope/account/branch/index, fingerprints,
derivation uniqueness, sighash forms, and exact persisted key identity before
secret lookup or clone mutation. After key lookup, apply an authorized tweak
exactly once and validate the effective key against the script commitment
before signing.

## Details

Permit spendable custom-scope signing only after Task 390 establishes matching
private account material. Path-bound external XPub accounts remain watch-only.
Complete every callback-independent authorization check before invoking the
maintained Signer. Preserve the existing
`InputTweakers`/`PrivKeyTweaker` compatibility seam by passing an
authorization-bound wrapper to the Signer. After the Signer looks up the base
key, that wrapper invokes the caller's tweak exactly once, validates the
effective public key against the authorized script commitment, and returns the
key only on a match. Reject callback-independent failures before secret lookup
or any tweak callback runs, and reject callback or effective-key failures
before signing or clone mutation.

Emit each successful signature under its validated effective-key identity:
use the effective compressed key in ECDSA `PartialSig.PubKey` and the effective
x-only key in Taproot script-path `TaprootScriptSpendSig.XOnlyPubKey`. For a
Taproot key-path spend, validate the effective internal key, apply the BIP341
key tweak after the caller tweak, validate the resulting output key, and write
only the signature bytes to `TaprootKeySpendSig`.

Keep Task 346's original `Bip32Derivation` and `TaprootBip32Derivation`
records byte-for-byte unchanged. They continue to identify the persisted base
key/path and must never be rewritten to the effective-key identity, including
across subsequent signing passes.

Before key lookup or any tweak callback, authorize every admitted ECDSA
`PartialSig` and Taproot script-path `TaprootScriptSpendSig`. Validate its
sighash form, bind its effective compressed or x-only key to the validated
previous output and script position, recompute the digest from that output and
script, and cryptographically verify the signature. Taproot script-path
records must also bind to the exact validated leaf hash. Preserve only fully
authorized records; reject any malformed, mismatched, or invalid record before
secret access, callback execution, signing, or clone mutation. After the
callback, compare only these already-verified record identities with the
wallet's validated effective key and leaf. A match suppresses signing and
prevents a duplicate PSBT map key. A verified foreign record remains preserved
but does not suppress the wallet's signature. Do not compare signature-record
keys against the persisted base-key derivation identity.

For Taproot key-path, treat the non-empty `TaprootKeySpendSig` field as the
singleton existing-signature record. After callback-independent transaction,
prevout, script, ownership, and sighash validation, verify that signature
against the authorized Taproot output key and digest. Preserve a valid record
byte-for-byte and treat the input as already signed before key lookup or tweak
callback execution; reject a malformed or invalid record before either side
effect.

[Task 384](384-vault-safe-signing.md) later moves primitive key use and tweak
application behind typed Vault operations without changing this task's
authorization contract. Do not define that Vault API or typed tweak descriptor
here.

Exclusions: FinalizePsbt auto-signing, funding, leasing, metadata merging,
and Vault interface invention.

## Suggested fix

Centralize a side-effect-free per-input authorization result
containing the validated previous output, spend form, persisted account/path,
sighash, script commitment, authorized tweak, and verified existing-signature
identities. Bind the result into a validating compatibility wrapper passed to
the maintained Signer. Once the callback-independent checks find no valid
existing Taproot key-path signature, let the Signer resolve the base key,
invoke the caller's tweak once, validate the effective key, and compare its
public identity only with the preverified records. A matching record suppresses
signing; otherwise permit the Signer to produce a signature. Mutate only a
packet clone while copying the base-key derivation metadata unchanged.

## Verification

Standard/custom-scope, multisig, SegWit, and Taproot vectors
sign only one authorized identity on a clone. Forged or ambiguous metadata,
invalid branches/children/accounts/sighashes, and missing-output
`SIGHASH_SINGLE` fail before key lookup, tweak callback, and caller mutation.
Tweak errors and effective-key/script mismatches fail after exactly one
callback but before signing or caller mutation.
Successful ECDSA and Taproot script-path records contain the effective
compressed and x-only identities respectively. Taproot key-path vectors
validate the caller-tweaked internal key and BIP341 output key while recording
only signature bytes.
Repeated signing passes retain every original `Bip32Derivation` and
`TaprootBip32Derivation` record byte-for-byte while adding signature records
under the appropriate effective-key identity. A second pass detects the
existing effective-key record, verifies its sighash, recomputed digest,
previous-output/script commitment, signature, and Taproot leaf where
applicable, then emits no additional signature. Corrupt signatures, wrong
sighashes or digests, mismatched effective keys, outputs, script positions, or
leaves fail before key lookup, callback execution, signing, or caller mutation.
Valid foreign records are preserved without suppressing the wallet's
signature; a verified matching record is compared with the callback-derived
effective identity and suppresses signing after exactly one callback.
Repeated Taproot key-path signing validates and preserves the singleton
`TaprootKeySpendSig` byte-for-byte, preserves base-key derivations, and performs
no key lookup or tweak callback on the second pass.

## Definition of done

SignPsbt emits signatures only after one complete
wallet-owned authorization decision and never mutates the caller packet on
error. Every emitted record uses the validated effective-key identity required
by its PSBT signature form without changing the persisted-base-key derivation
metadata. Every preserved signature record is cryptographically authorized
against the validated transaction context. No signing pass emits a duplicate
effective-key record or replaces a validated Taproot key-path signature;
FinalizePsbt behavior remains owned by the follow-up task.

[t346]: 346-psbt-funding-metadata.md
[t390]: 390-custom-scope-account-creation.md
