# Task 382 — move account derivation into the Vault

Component: **AccountManager and account-derivation Vault operation**.

Type: **Refactor**.

Dependencies: **[Task 354][t354] and [Task 390][t390]**.

## Rationale

At `sql-wallet` revision `2ae306dea8c5`, `buildAccountDeriveFn` captures
encrypted root material, not plaintext, but decrypts and derives when the
Store callback runs. That secret work still executes under a Store write and
leaves AccountManager owning operations that ADR 0010 assigns to Vault.

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
  used by the sequential, exact, and custom-scope callers. Preserve their
  existing cursor reads and pass only the typed Vault result's ciphertext and
  public facts to their existing short operation-specific Store CAS.
- Run no Vault operation, derivation, encryption, or caller callback under a
  Store write transaction.
- Zero master, account, branch, serialization, and other task-owned temporary
  material on success, error, and panic. Apply the ADR's immutable-string rule.
- Preserve lock, watch-only, invalid-child, fingerprint, and legacy behavior.
  Migrate the sequential and exact callers from [Task 343][t343] and the
  validated custom-scope caller from [Task 390][t390] without adding a second
  preparation workflow.
- Preserve kvdb's existing sequential creation and unsupported exact/custom
  request results without migrating that backend's secret handling.
- Exclude account-number allocation and exact-account CAS policy from Task 343,
  sequential invalid-child allocation from [Task 460][t460], custom-scope
  schema admission from Task 390,
  supplied XPub/path-bound creation from [Task 391][t391], initial SQL genesis
  from Tasks 329 and 367, and creation validation from Task 365.

## Suggested fix

Replace the callers' raw preparation helper with a Vault-owned account
preparation request and a prepared ciphertext/public result. Keep their
existing cursor read and checked Store CAS phases unchanged. Leave the legacy
path unchanged.

## Verification

- Public AccountManager itests create sequential, exact, and custom-scope
  accounts on SQLite and PostgreSQL with unchanged exported results. Modern
  kvdb covers sequential creation only and proves exact and custom-scope
  requests match `ErrAccountOperationUnsupported` after the permitted read-only
  occupied-name lookup and before Vault preparation or Store mutation.
- Spies prove SQL AccountManager never calls Store secret reads or raw `Decrypt`
  and no Vault call executes while a Store write is open.
- Concurrency tests cover Lock and combined passphrase rotation against account
  preparation without deadlock or obsolete-credential commits.
- Zeroing probes cover success, derivation/encryption error, CAS failure, and
  panic.

## Definition of done

- SQL root-derived account creation obtains prepared secret material through
  one wallet-scoped Vault operation. Sequential, exact, and custom-scope
  callers use it; kvdb's existing implementation and behavior are unchanged.
- Store writes receive ciphertext and public facts only and contain no secret
  work.
- No exact/custom, genesis, validation, reconciliation, or PSBT behavior is
  included.

[t352]: ../archive/tasks/352-kvdb-combined-passphrase-rotation.md
[t354]: 354-vault-owned-secret-resolution.md
[t343]: ../archive/tasks/343-exact-account-creation.md
[t390]: ../archive/tasks/390-custom-scope-account-creation.md
[t391]: 391-path-bound-watch-only-xpub-accounts.md
[t460]: 460-sequential-account-invalid-children.md
