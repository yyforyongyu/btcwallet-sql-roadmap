# Task 347 — authorize explicit PSBT signatures

Component: **PsbtManager and Signer**.

Dependencies: **[Task 346][t346], [Task 384][t384], and [Task 390][t390]**.

## Rationale

Retain the current explicit SignPsbt authorization
contract: validate previous outputs and script commitments; infer spend type
from validated scripts; validate scope/account/branch/index, fingerprints,
derivation uniqueness, sighash forms, tweakers, and exact persisted key
identity before secret lookup or clone mutation.

## Details

Permit spendable custom-scope signing only after Task 390 establishes matching
private account material. Path-bound external XPub accounts remain watch-only.
Consume Task 384's safe Vault-signing contract rather than defining another
Vault API; Task 354 and the PR #1279 disposition are transitive through it.

Exclusions: FinalizePsbt auto-signing, funding, leasing, metadata merging,
and Vault interface invention.

## Suggested fix

Centralize a side-effect-free per-input authorization result
containing the validated previous output, spend form, persisted account/path,
sighash, and authorized tweak, then sign and mutate only a packet clone.

## Verification

Standard/custom-scope, multisig, SegWit, and Taproot vectors
sign only one authorized identity on a clone. Forged or ambiguous metadata,
invalid branches/children/accounts/sighashes, and missing-output
`SIGHASH_SINGLE` fail before key lookup and caller mutation.

## Definition of done

SignPsbt emits signatures only after one complete
wallet-owned authorization decision and never mutates the caller packet on
error; FinalizePsbt behavior remains owned by the follow-up task.

[t346]: 346-psbt-funding-metadata.md
[t384]: ../archive/tasks/384-vault-safe-signing.md
[t390]: 390-custom-scope-account-creation.md
