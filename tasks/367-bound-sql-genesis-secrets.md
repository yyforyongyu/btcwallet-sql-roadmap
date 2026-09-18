# Task 367 — bound SQL genesis secret lifetime

Component: **Manager and WalletVault genesis preparation**.

Type: **Bug fix**.

Dependencies: **None**.

## Rationale

At `sql-wallet` revision `2ae306dea8c5`, `genRootKey` and
`Manager.createWallet` do not clear their generated seed and owned master
key. `createWalletSecrets` already defers cleanup of its cipher keys and
serialized root. Close the remaining Manager-owned lifetime gap on the
current SQL path; it does not require Task 329's atomic-genesis redesign.

## Details

- Clear Manager-generated seed bytes and internally derived master keys on
  every exit, establishing cleanup when ownership begins.
- Borrow caller-provided seed and XPrv synchronously without retaining,
  mutating, or destroying them. `hdkeychain.NewMaster` does not retain its
  seed. Do not use `CloneWithVersion` as an ownership copy: its backing slices
  can alias and zeroing it can destroy caller material.
- Reuse WalletVault's existing cleanup and ciphertext/public persistence
  boundary. Do not add a future account-preparation plan or duplicate the
  runtime derivation cleanup owned by Task 382.
- Keep modern kvdb, general Vault API redesign, reconciliation, key-validation
  policy, public cancellation, and caller-owned key destruction out of scope.

## Suggested fix

Defer destruction immediately after obtaining each Manager-owned seed/master.
Distinguish borrowed roots from derived owned roots in the current creation
flow; reuse existing Vault cleanup without a new ownership abstraction.

## Verification

- Focused observations prove generated seed and owned master cleanup on
  success and representative later validation, Vault, or Store failure, with
  deferred cleanup also surviving panic.
- Imported-seed and imported-XPrv requests leave caller material unchanged
  and usable after success and failure; derived masters are still cleared.
- Reuse existing WalletVault cleanup tests rather than duplicating them or
  requiring new preparation/test infrastructure.

## Definition of done

The current SQL creation path bounds every remaining Manager-owned seed and
master lifetime while preserving borrowed inputs and existing Vault cleanup.
No dependency on a future genesis representation or unrelated backend work is
introduced.
