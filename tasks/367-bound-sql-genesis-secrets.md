# Task 367 — bound SQL genesis secret lifetime

Component: **Manager and WalletVault genesis preparation**.

Dependencies: **[Task 329][t329]**.

## Rationale

SQL genesis preparation can retain generated seed bytes, internally derived
masters, account working keys, or temporary private-key serializations beyond
their required use. Caller-owned seed and XPrv inputs are borrowed
synchronously, so the ciphertext-only transaction boundary in Task 329 needs
both one owner for destroying preparatory plaintext and an explicit rule that
borrowed inputs are never retained or mutated.

## Details

- Keep generated seed bytes in wipeable Manager-owned storage and zero them on
  success, validation failure, preparation failure, write failure, and panic.
- Treat caller-provided seed and XPrv inputs as borrowed. Validate and use them
  synchronously without cloning, retaining, mutating, or destroying them.
  `hdkeychain.NewMaster` does not retain the seed, and the imported `RootKey`
  is not cached or mutated by genesis preparation.
- Do not use `CloneWithVersion` as an ownership copy for an imported XPrv. Its
  backing key and chain-code slices can alias the caller's key, so zeroing the
  apparent clone can destroy caller-owned material.
- Treat every internally derived master extended key as task-owned and arrange
  its destruction immediately after derivation so every later exit clears it.
- Bound plaintext account private keys and temporary serializations inside the
  WalletVault preparation operation. Clear every wipeable source, decoded
  working copy, and temporary `[]byte(acctPriv.String())` serialization on
  success, error, and panic.
- Keep only ciphertext and public derivation facts in the SQL genesis plan.
  No plaintext root or account serialization may enter the Store transaction.
- Prefer byte-oriented extended-key parsing. If the upstream API requires an
  immutable plaintext string, confine it to the Vault call, retain no
  reference, document the bounded exception, and test every wipeable copy.
- Keep this task on the maintained SQL genesis path. Do not add modern-kvdb
  cleanup, general Vault API redesign, reconciliation, validation policy,
  public cancellation, or caller-owned key destruction.

## Suggested fix

Give each Manager- and Vault-owned plaintext value an explicit owner and a
single deferred destruction path established at acquisition time. Borrow
caller-owned seed and private-root inputs only for synchronous use, pass only
ciphertext and public facts out of WalletVault, and add narrow observation
seams for deterministic cleanup tests.

## Verification

- Focused probes observe zeroing after successful preparation, validation and
  Vault errors, definite Store failures, and injected panics.
- Generated-seed flows clear the Manager-owned seed and derived master on every
  exit.
- Imported-seed flows prove the borrowed input remains unchanged and usable
  after successful and failed preparation while the derived master is cleared.
- Imported-XPrv flows prove the borrowed key remains unchanged and usable
  after successful and failed preparation, with no clone-and-zero path.
- Initial-account fixtures prove private working keys and wipeable
  serializations are cleared while the persisted plan contains only ciphertext
  and public facts.
- Immutable-string parsing retains no reference beyond the bounded Vault call;
  the exception is documented without claiming that immutable memory is
  wipeable.
- Tests and code inspection confirm no modern-kvdb or general Vault API work is
  included.

## Definition of done

- Every Manager- or WalletVault-owned SQL genesis plaintext value has one
  bounded lifetime and is destroyed on all exits.
- Caller-owned seed and XPrv inputs are borrowed synchronously and never
  cloned, retained, mutated, or destroyed.
- The SQL persistence plan contains only ciphertext and public facts.
- Any unavoidable immutable-string exposure is confined and documented, and
  the PR contains no unrelated backend or Vault redesign.

[t329]: 329-atomic-wallet-account-creation.md
