# Task 382 — move account derivation into the Vault

Component: **AccountManager and account-derivation Vault operation**.

Dependencies: **[Task 352][t352] and [Task 354][t354]**.

## Rationale

AccountManager currently loads and decrypts the master HD private key, retains
it in a Store derivation closure, and may execute derivation under a Store
write. This gives Wallet and Store transaction code secret ownership that ADR
0010 assigns to Vault.

## Details

- Add only the typed Vault preparation operation needed by root-derived
  AccountManager account creation. It receives an already validated scope and
  account number without deciding whether allocation was sequential or exact.
- Let WalletVault and, while maintained, LegacyWalletVault resolve their own
  encrypted root/account material and cipher provenance. Keep their backend
  implementations isolated behind the same semantic request.
- Read the expected account cursor before mutation, derive and encrypt account
  material in Vault, then pass only ciphertext and public facts to one short
  operation-specific Store CAS.
- Run no Vault operation, derivation, encryption, or caller callback under a
  Store write transaction.
- Zero master, account, branch, serialization, and other task-owned temporary
  material on success, error, and panic. Apply the ADR's immutable-string rule.
- Preserve lock, watch-only, invalid-child, fingerprint, and legacy behavior.
  Migrate the existing sequential caller; [Task 343][t343] later consumes the
  same operation for exact root-derived accounts and [Task 390][t390] for
  validated custom scopes.
- Exclude account-number allocation and exact-account policy from Task 343,
  custom-scope schema admission from Task 390, supplied XPub/path-bound
  creation from [Task 391][t391], initial SQL genesis from Tasks 329 and 367,
  creation validation from Task 365, and ambiguous genesis reconciliation from
  Task 366.

## Suggested fix

Replace `buildAccountDeriveFn` with a Vault-owned account preparation request
and a prepared ciphertext/public result. Split cursor read, Vault preparation,
and checked Store CAS into explicit phases. Implement the legacy path inside
LegacyWalletVault without importing SQL row identity.

## Verification

- External-package AccountManager tests create an ordinary account on SQLite
  and modern kvdb with unchanged public results.
- Spies prove AccountManager never calls Store secret reads or raw `Decrypt`
  and no Vault call executes while a Store write is open.
- Concurrency tests cover Lock and combined passphrase rotation against account
  preparation without deadlock or obsolete-credential commits.
- Zeroing probes cover success, derivation/encryption error, CAS failure, and
  panic.

## Definition of done

- Root-derived account creation obtains prepared secret material only through
  one wallet-scoped Vault operation; the current sequential caller uses it and
  later exact/custom callers can consume it without a new secret path.
- Store writes receive ciphertext and public facts only and contain no secret
  work.
- No exact/custom, genesis, validation, reconciliation, or PSBT behavior is
  included.

[t352]: 352-kvdb-combined-passphrase-rotation.md
[t354]: 354-vault-owned-secret-resolution.md
[t343]: 343-exact-account-creation.md
[t390]: 390-custom-scope-account-creation.md
[t391]: 391-path-bound-watch-only-xpub-accounts.md
