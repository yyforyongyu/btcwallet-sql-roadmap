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
- On maintained modern kvdb, route only its existing sequential root-derived
  subset through LegacyWalletVault. Preserve Task 378's canonical
  `ErrAccountOperationUnsupported` for Task 343 exact requests and Task 390
  custom scopes after Task 392's required read-only occupied-name lookup but
  before Vault preparation or Store mutation.
- Exclude account-number allocation and exact-account CAS policy from Task 343,
  sequential invalid-child allocation from [Task 460][t460], custom-scope
  schema admission from Task 390,
  supplied XPub/path-bound creation from [Task 391][t391], initial SQL genesis
  from Tasks 329 and 367, creation validation from Task 365, and ambiguous
  genesis reconciliation from Task 366.
- Keep implementation local to existing AccountManager, WalletVault,
  LegacyWalletVault, source files, and colocated tests. Add no parallel
  interface, arbitrary helper, unrelated production file, or standalone test
  file; justify any unavoidable new file against the repository's current
  layout in the implementation PR.
- Document non-obvious code blocks with the contract, rationale, invariant, or
  consequence they preserve; do not restate the code.
- Keep unit tests clearly structured as Arrange, Act, and Assert.
- Interface mocks, when needed, embed `mock.Mock`. Set `On(...).Return(...)`
  expectations during Arrange with the narrowest correct cardinality (`Once()`
  for one required call). During Assert, call `AssertExpectations(t)` directly;
  do not duplicate call assertions or wrap the result in `require.True`.

## Suggested fix

Replace the callers' raw preparation helper with a Vault-owned account
preparation request and a prepared ciphertext/public result. Keep their
existing cursor read and checked Store CAS phases unchanged. Implement the
legacy path inside LegacyWalletVault without importing SQL row identity.

## Verification

- Public AccountManager itests create sequential, exact, and custom-scope
  accounts on SQLite and PostgreSQL with unchanged exported results. Modern
  kvdb covers sequential creation only and proves exact and custom-scope
  requests match `ErrAccountOperationUnsupported` after the permitted read-only
  occupied-name lookup and before Vault preparation or Store mutation.
- Spies prove AccountManager never calls Store secret reads or raw `Decrypt`
  and no Vault call executes while a Store write is open.
- Concurrency tests cover Lock and combined passphrase rotation against account
  preparation without deadlock or obsolete-credential commits.
- Zeroing probes cover success, derivation/encryption error, CAS failure, and
  panic.

## Definition of done

- Root-derived account creation obtains prepared secret material only through
  one wallet-scoped Vault operation. SQL sequential, exact, and custom-scope
  callers use it, while maintained modern kvdb uses it only for its supported
  sequential subset without changing public behavior.
- Store writes receive ciphertext and public facts only and contain no secret
  work.
- No exact/custom, genesis, validation, reconciliation, or PSBT behavior is
  included.

[t352]: 352-kvdb-combined-passphrase-rotation.md
[t354]: 354-vault-owned-secret-resolution.md
[t343]: 343-exact-account-creation.md
[t390]: 390-custom-scope-account-creation.md
[t391]: 391-path-bound-watch-only-xpub-accounts.md
[t460]: 460-sequential-account-invalid-children.md
