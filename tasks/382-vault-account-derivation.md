# Task 382 — move account derivation into the Vault

Component: **AccountManager and account-derivation Vault operation**.

Type: **Refactor**.

Dependencies: **[Task 354][t354] and [Task 390][t390]**.

## Rationale

At local `sql-wallet` revision `48e7f4b7e`, `buildAccountDeriveFn` captures
encrypted root material but decrypts and derives inside the account-creation
Store transaction. This is not an existing prepare-then-short-CAS flow.
AccountManager still owns secret work that ADR 0010 assigns to Vault.

That path takes Store resources before Vault's lock, while passphrase rotation
takes the Vault lock before Store access. Validate this source-level concern
with a constrained connection pool; it is not yet a reproduced deadlock. A
minimal demonstrated bug fix need not wait for this ownership refactor.

## Details

- Scope this refactor to SQL. Keep the existing kvdb derivation and credential
  path unchanged; its containment work remains deferred in [Task 352][t352].
- Add only the typed Vault preparation operation needed by the existing
  root-derived AccountManager paths from Tasks 343 and 390. It receives an
  already validated scope and account number without deciding whether
  allocation was sequential, exact, or in a custom scope.
- Let WalletVault resolve its encrypted root/account material and cipher
  provenance without changing LegacyWalletVault.
- Replace the raw maintained decrypt, derivation, and encryption preparation
  used by sequential, exact, and custom-scope callers with Vault-owned secret
  use. Return only ciphertext and public facts to SQL AccountManager.
- Preserve atomic allocation and account creation under concurrent callers.
  Do not assume an existing short CAS or require a new allocation protocol.
  Keep Store/Vault ordering deadlock-free and prevent passphrase rotation from
  allowing an account encrypted with obsolete credentials to commit.
- Zero master, account, branch, serialization, and other task-owned temporary
  material on success, error, and panic. Apply the ADR's immutable-string rule.
- Preserve lock, watch-only, invalid-child, fingerprint, and legacy behavior.
  Migrate the sequential and exact callers from [Task 343][t343] and the
  validated custom-scope caller from [Task 390][t390] without adding a second
  preparation workflow.
- Preserve kvdb's existing sequential creation and unsupported exact/custom
  request results without migrating that backend's secret handling.
- Remove helpers and secret-access seams made unused by this consumer move,
  with their required test/mock updates. Retain any shared seam still needed
  by other consumers or unchanged kvdb behavior; add no legacy adapter.
- Exclude account-number allocation and exact-account CAS policy from Task 343,
  sequential invalid-child allocation from [Task 460][t460], custom-scope
  schema admission from Task 390,
  supplied XPub/path-bound creation from [Task 391][t391], initial SQL genesis
  from Tasks 329 and 367, and creation validation from Task 365.

## Suggested fix

Move the existing secret-use helper behind a narrow Vault operation. Preparation
outside a write is an option only if credential and allocation atomicity remain
intact. Choose the smallest safe boundary under Task 354; leave kvdb unchanged.

## Verification

- Public AccountManager itests create sequential, exact, and custom-scope
  accounts on SQLite and PostgreSQL with unchanged exported results. Modern
  kvdb covers sequential creation only and proves exact and custom-scope
  requests match `ErrAccountOperationUnsupported` after the permitted read-only
  occupied-name lookup and before Vault preparation or Store mutation.
- Spies prove SQL AccountManager never calls Store secret reads or raw `Decrypt`
  and receives no plaintext key material.
- Concurrency tests cover Lock and combined passphrase rotation against account
  creation with a one-connection SQL pool, without deadlock or obsolete-
  credential commits. Concurrent creation preserves allocation and identity.
- Zeroing probes cover success, derivation/encryption error, commit failure,
  and panic. Remove only tests/mocks made obsolete by this consumer move.

## Definition of done

- SQL root-derived account creation obtains prepared secret material through
  one wallet-scoped Vault operation. Sequential, exact, and custom-scope
  callers use it; kvdb's existing implementation and behavior are unchanged.
- SQL AccountManager receives ciphertext and public facts only; creation and
  passphrase rotation retain their atomicity without a Store/Vault deadlock.
- No exact/custom, genesis, validation, reconciliation, or PSBT behavior is
  included.

[t352]: ../archive/tasks/352-kvdb-combined-passphrase-rotation.md
[t354]: 354-vault-owned-secret-resolution.md
[t343]: ../archive/tasks/343-exact-account-creation.md
[t390]: ../archive/tasks/390-custom-scope-account-creation.md
[t391]: 391-path-bound-watch-only-xpub-accounts.md
[t460]: 460-sequential-account-invalid-children.md
