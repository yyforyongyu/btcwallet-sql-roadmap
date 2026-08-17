# Task 354 — define Vault secret-operation ownership

Component: **ADR 0010 secret-use contract**.

Type: **Design**.

Dependencies: **None**, plus durable disposition of draft btcwallet PR #1279
before ADR acceptance.

## Rationale

ADR 0010 assigns secret lifetime and typed key operations to keyvault but does
not define operation synchronization, reentrancy, plaintext ownership, raw
return exceptions, or the relationship between primitive signing and the
public Signer. PR #1279 proposes one answer but is unaccepted and exposes a
backend AccountID.

## Details

- Amend ADR 0010 before implementation to define operation-scoped secret
  ownership, synchronization against `Lock`, reentrancy, and zeroing on
  success, error, and panic.
- Prefer typed Vault operations that perform key derivation, ECDH, tweaking,
  and primitive signing without returning private material to safe Wallet
  callers. Wallet retains transaction, PSBT, script-commitment, sighash, and
  witness policy.
- Permit an internal callback only when no narrower typed operation can express
  the use. Do not run arbitrary caller code under a non-reentrant Vault guard.
- Keep the Vault wallet-scoped. Selectors use semantic scope/name or optional
  BIP44 number, derivation path, or script pubkey; they never expose or accept
  SQL row IDs, kvdb adapter IDs, or wallet IDs.
- Define `UnsafeSigner` private-key returns and
  `AddressManager.ScriptForOutput` script returns as explicit caller-owned copy
  exceptions. Do not claim Vault controls their lifetime after return.
- Define the bounded immutable-string parsing exception: retain no reference
  and zero every wipeable source and working copy, without claiming immutable
  string memory is wipeable.
- State that account derivation/encryption occurs outside Store write
  transactions and only prepared ciphertext/public facts enter an
  operation-specific CAS.
- Record PR #1279's exact disposition. Reject its proposed public AccountID
  locator. If an accepted revision is consumed, record the exact merge commit
  and delete duplicate roadmap surface.
- Keep SQL genesis secret lifetime in Task 367, PSBT authorization in Task 347,
  and redundant test cleanup in Task 336.

## Suggested fix

Add one focused ADR section describing typed operation ownership, lock/use
serialization, raw-return exceptions, semantic selectors, and consumer
responsibilities. Include a short PR #1279 overlap table and an explicit
accepted/superseded/non-overlap outcome.

## Verification

- Review the amendment independently against current AccountManager, Signer,
  UnsafeSigner, ScriptForOutput, WalletVault, LegacyWalletVault, and Store
  transaction paths.
- Confirm no accepted type contains a backend ID and no safe operation returns
  a private key.
- Confirm every planned plaintext value has an owner and an exit-path rule.
- Confirm PR #1279 is referenced by exact accepted commit only if merged.

## Definition of done

- ADR 0010 specifies the complete secret-use ownership contract needed by the
  four consumer tasks.
- PR #1279 overlap is removed, accepted by exact commit, or explicitly frozen
  as non-overlapping.
- The PR changes no Go code and does not absorb genesis, PSBT authorization, or
  test cleanup.
