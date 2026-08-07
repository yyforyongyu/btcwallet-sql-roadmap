# Task 354 — move secret resolution behind the Vault

Component: **WalletVault, AccountManager, Signer, and UnsafeSigner**.

Dependencies: **An accepted ADR 0010 amendment, plus [Task 352][t352] while
the modern kvdb backend remains; otherwise [Task 353][t353] removes that
prerequisite with the retired kvdb clauses**.

## Rationale

The signer obtains encrypted account and address rows through `runtimeCache`,
and account creation reads encrypted master-HD material directly from the
Store. ADR 0010 assigns secret persistence routing, typed key resolution, and
secret lifetime to the Vault, but its current returned-key and cache language
does not define the planned callback ownership contract.

## Details

- Add only the semantic account-derivation, account-key, imported-key, and
  script operations required by [Task 336][t336], [Task 343][t343], and
  [Task 347][t347]. Do not redesign every Vault cipher method.
- Amend ADR 0010 first with callback ownership, synchronization, reentrancy,
  zeroing, and the explicit raw-return exceptions retained by this task.
- Keep the Vault wallet-scoped. Callers do not pass a wallet ID or receive
  encrypted Store rows.
- Let Vault implementations load encrypted material through `db.Store` and
  apply legacy cipher provenance internally.
- Remove `AccountDerivationFunc` execution from Store write transactions. Read
  the expected account cursor first, prepare and encrypt account material in the
  Vault, then pass ciphertext and public facts to one short Store CAS. No Vault
  callback, key derivation, or encryption runs under a Store write transaction.
- Expose plaintext keys and scripts only inside callback-scoped operations.
  Hold Vault synchronization for the whole callback so `Lock` cannot zero live
  state during use. Callbacks must not re-enter the Vault. Zero every temporary
  buffer immediately after the callback on success, error, or panic.
- Prefer a byte-oriented extended-key decoder. If upstream parsing requires an
  immutable plaintext string, confine it to the callback, retain no reference,
  document the bounded exception shared with [Task 367][t367] and
  [Task 335][t335], and zero every wipeable source and decoded working copy on
  exit.
- Keep existing `UnsafeSigner` and `PrivKeyForAddress` raw-key returns plus
  `ScriptForOutput` script returns as explicit caller-owned compatibility
  exceptions; do not claim the Vault controls their post-return lifetime.
  Return copies and document caller destruction responsibility.
- Move private-key tweaking inside the Vault callback and zero both the original
  and any replacement key on every exit.
- Replace maintained `PrivKeyTweaker` callbacks with the smallest immutable,
  wallet-owned typed tweak descriptor covering supported operations. Validate
  it before secret lookup and apply it inside the Vault. Leave deprecated wallet
  callback APIs untouched; arbitrary caller code must not run under a
  non-reentrant Vault guard or retain a key.
- Remove raw `Decrypt` from the Wallet-facing Vault interface after typed
  operations migrate. A backend-private primitive may remain only if no Wallet
  caller can obtain its plaintext result.
- Remove `runtimeCache.GetAccountSecret` and `GetAddressSecret` after every
  signer caller moves, and remove direct account-manager master-key reads after
  typed derivation moves.
- Preserve locked, not-mine, imported-xpub, and watch-only behavior.

## Suggested fix

Introduce narrow callback-based Vault selectors for derived account keys,
imported private keys, scripts, and exact account derivation. Move each existing
Store lookup plus decrypt sequence into `WalletVault` and `LegacyWalletVault`,
then delete the cache pass-throughs and direct account-manager lookup. Keep the
Vault guard across typed lookup, typed tweak, use, and zeroing. Replace the
Store derivation callback with Vault preparation plus expected-cursor CAS.

## Verification

No signer or account-manager path reads encrypted secret DTOs directly, the
Vault owns every internal callback-scoped secret lookup and lifetime, and raw
decrypt is no longer Wallet-facing. Focused SQL and kvdb vectors prove zeroing,
lock-versus-callback serialization, tweaker cleanup, documented compatibility
exceptions, and unchanged public behavior. Concurrent account creation and
passphrase rotation never derive inside a Store write, deadlock, or commit
material encrypted under an obsolete credential. Immutable-string parsing
retains no reference beyond the callback and every wipeable copy is cleared.

## Definition of done

- Maintained signing and account creation resolve typed secrets only through
  wallet-scoped Vault operations with callback-bounded ownership and zeroing.
- No Wallet caller receives encrypted Store DTOs or the raw decrypt primitive;
  documented compatibility returns remain caller-owned copies.
- Store writes contain only prepared ciphertext and public facts, with no
  derivation, encryption, or Vault callback under the transaction.

[t335]: 335-kvdb-to-sql-migration.md
[t336]: 336-test-cleanup.md
[t343]: 343-exact-account-creation.md
[t347]: 347-psbt-signing-authorization.md
[t352]: 352-kvdb-combined-passphrase-rotation.md
[t353]: 353-retire-modern-kvdb-backend.md
[t367]: 367-bound-sql-genesis-secrets.md
