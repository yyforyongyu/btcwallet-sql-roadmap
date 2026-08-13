# Task 382 — move account derivation into the Vault

Component: **AccountManager and account-derivation Vault operation**.

Type: **Refactor**.

Dependencies: **[Task 352][t352], [Task 354][t354], and [Task 390][t390]**.

## Rationale

AccountManager currently loads and decrypts the master HD private key, retains
it in a Store derivation closure, and may execute derivation under a Store
write. This gives Wallet and Store transaction code secret ownership that ADR
0010 assigns to Vault.

## Details

- Add only the typed Vault preparation operation needed by the existing
  root-derived AccountManager paths from Tasks 343 and 390. It receives an
  already validated scope and account number without deciding whether
  allocation was sequential, exact, or in a custom scope.
- Let WalletVault and, while maintained, LegacyWalletVault resolve their own
  encrypted root/account material and cipher provenance. Keep their backend
  implementations isolated behind the same semantic request.
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
- Exclude account-number allocation, exact-account policy, invalid-child and
  CAS sequencing from Task 343, custom-scope schema admission from Task 390,
  supplied XPub/path-bound creation from [Task 391][t391], initial SQL genesis
  from Tasks 329 and 367, creation validation from Task 365, and ambiguous
  genesis reconciliation from Task 366.

## Suggested fix

Replace the callers' raw preparation helper with a Vault-owned account
preparation request and a prepared ciphertext/public result. Keep their
existing cursor read and checked Store CAS phases unchanged. Implement the
legacy path inside LegacyWalletVault without importing SQL row identity.

## Verification

- External-package AccountManager tests create sequential, exact, and custom-
  scope accounts on SQLite and modern kvdb where supported, with unchanged
  public results.
- Spies prove AccountManager never calls Store secret reads or raw `Decrypt`
  and no Vault call executes while a Store write is open.
- Concurrency tests cover Lock and combined passphrase rotation against account
  preparation without deadlock or obsolete-credential commits.
- Zeroing probes cover success, derivation/encryption error, CAS failure, and
  panic.

## Definition of done

- Root-derived account creation obtains prepared secret material only through
  one wallet-scoped Vault operation; the existing sequential, exact, and
  custom-scope callers all use it without changing their public behavior.
- Store writes receive ciphertext and public facts only and contain no secret
  work.
- No exact/custom, genesis, validation, reconciliation, or PSBT behavior is
  included.

[t352]: 352-kvdb-combined-passphrase-rotation.md
[t354]: 354-vault-owned-secret-resolution.md
[t343]: 343-exact-account-creation.md
[t390]: 390-custom-scope-account-creation.md
[t391]: 391-path-bound-watch-only-xpub-accounts.md
