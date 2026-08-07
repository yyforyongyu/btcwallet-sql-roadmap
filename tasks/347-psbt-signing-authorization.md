# Task 347 — complete PSBT signing authorization

Component: **PsbtManager and Signer**.

Dependencies: **[Task 343][t343], [Task 346][t346], and [Task 354][t354]**.

## Rationale

Signing applies network-global coin-type assumptions and does not uniformly
prove that persisted account identity, scripts, UTXOs, derivations, and Taproot
metadata authorize the requested signature.

## Details

- Resolve persisted wallet scope and account identity, including custom scopes.
- Verify witness/non-witness UTXOs and every script commitment.
- Infer the spend type from the validated previous output plus redeem, witness,
  and Taproot scripts. Do not select signing behavior from BIP purpose.
- Accept only branch 0 or 1 and a non-hardened child index before Store or Vault
  access.
- Reject an account component above `db.MaxAccountNumber` before Store or Vault
  access for both BIP32 and Taproot derivations.
- Preserve absent versus present-zero fingerprints.
- Normalize effective sighash by spend version.
- Define a closed set of valid base sighashes and flags for legacy, SegWit v0,
  and Taproot. Reject unknown bits, invalid base types, and illegal flag
  combinations before key lookup.
- Reject base `SIGHASH_SINGLE` for every supported spend version when the input
  has no corresponding output, before key lookup or packet mutation.
- Require exactly one unambiguous owned derivation for the key being signed;
  reject conflicting or multiple wallet-owned candidates.
- Apply ownership and tweaker authorization before decoration, finalization, or
  any other wallet mutation of the cloned packet.
- Restrict `FinalizePsbt` auto-signing to untweaked keys. Inputs requiring a
  tweaker must already carry an authorized signature before finalization.
- Keep arbitrary numberless XPub-only private signing unavailable. Permit a
  path-bound external XPub only after [Task 343][t343] proves matching private account
  material through the Vault.

## Suggested fix

Consume [Task 346][t346]'s side-effect-free validator and clone, then centralize signing
authorization, including the effective tweaked key, before mutation. Derive the
exact owned key through the Vault, sign the clone, and execute finalized scripts
before publishing the result.

## Verification

Standard, custom-scope, multisig, and Taproot vectors sign only when ownership
and commitments match; forged metadata produces no signature or caller
mutation. Missing-output `SIGHASH_SINGLE` and unsigned tweaked finalization fail
before key access. A numberless imported XPub cannot fall back to private
signing, while an authorized path-bound account can sign after [Task 343][t343].
Invalid purpose-independent spend shapes, branches, children, and sighash
encodings fail before Store or Vault access. Account-number boundary tests
accept `MaxAccountNumber` and reject the next value without secret lookup.

## Definition of done

- PSBT signing authorizes exactly one persisted wallet identity, validated
  previous output, script commitment, derivation, sighash, and tweak before
  packet mutation or secret lookup.
- Forged or ambiguous metadata produces no signature, secret access, or caller
  mutation, while supported standard and custom accounts remain signable.
- Numberless imported XPub accounts remain public-only unless a path-bound
  account proves matching private material through the Vault.

[t343]: 343-exact-account-creation.md
[t346]: 346-psbt-funding-metadata.md
[t354]: 354-vault-owned-secret-resolution.md
