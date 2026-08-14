# Task 373 — extract legacy account and address state

Component: **Legacy migration account and secret extraction**.

Type: **Migration**.

Dependencies: **[Task 332][t332] and [Task 335][t335]**.

## Rationale

Legacy waddrmgr represents ordinary accounts, imported-XPub accounts, and raw
imports through different scoped buckets and a synthetic imported account.
Its encrypted scripts also select their protecting crypto key per record. The
public kvdb Store projection loses some of this identity, and copying encrypted
rows directly would not produce SQL's single-passphrase Vault format.

## Details

- Read only Task 335's private legacy snapshot. Do not access the original
  source path and do not open or mutate a SQL destination.
- Produce one bounded internal snapshot of master/public roots, scopes,
  ordinary accounts, names, schemas, encrypted account material, external and
  internal counters, persisted child paths, imported-XPub identity, raw
  scripts, optional fingerprints, sticky legacy usedness, and custody facts.
- Map the synthetic imported-address pseudo-account to accountless SQL raw
  addresses. Map each true imported XPub to a numberless SQL account without
  copying a synthetic waddrmgr account number.
- Preserve every ordinary scope and derived account. Normalize each scope's
  next numbered account cursor from legacy allocation state and the greatest
  ordinary account with checked arithmetic; synthetic and numberless accounts
  cannot advance it.
- Normalize every legacy account and raw import to immutable `watch-chain`.
  Legacy kvdb has no key-only mode, so do not infer one.
- Consume Task 332's accepted custody policy. Refuse mixed spendable/watch-only
  material, root-neutered or other unrepresentable custody, conflicting
  per-account schema overrides, and any shape that would require splitting a
  wallet or transforming custody.
- Compare normalized account-XPub identities and script identities before
  mutation. Merge only semantically identical all-raw duplicates. Refuse
  latent overlapping-XPub branches, raw-versus-derived collisions,
  derived-versus-derived collisions, or conflicting raw duplicates.
- Accept a required destination SQL passphrase independently from the legacy
  credentials. Decrypt each legacy script with its persisted crypto-key class
  and re-encrypt every target secret with that destination passphrase through
  a migration-specific WalletVault preparation boundary.
- Require and validate a borrowed legacy private credential only when custody
  facts prove that spendable source secrets must be decrypted. Watch-only
  extraction uses the required legacy public credential and recorded
  public/script key provenance without requesting or validating a nonexistent
  legacy private passphrase. Task 352 remains the only owner of in-place kvdb
  rotation.
- Give every plaintext root, account key, private key, script, decoder buffer,
  and temporary serialization one explicit owner and one deferred destruction
  path covering success, refusal, cancellation, panic, and later extraction
  failure.
- Prefer byte-oriented extended-key decoding. If upstream parsing forces an
  immutable plaintext string, confine it to the migration Vault call, retain no
  reference, document the bounded exception, and never claim the immutable
  string itself was wiped.
- Return only ciphertext and normalized public facts. Keep the snapshot,
  refusal reasons, Vault preparation types, and decoder interfaces internal.
- Exclude transaction and sync extraction, usedness-versus-credit proof, SQL
  writes, Task 367's ordinary-genesis cleanup, Vault API redesign, watch-only
  wallet splitting, custody transformation, and public orchestration.

## Suggested fix

Add a migration-only raw namespace reader over Task 335's snapshot and a narrow
WalletVault preparation helper. Walk scopes and imported buckets directly,
normalize them into SQL-shaped internal facts, decrypt each secret with its
recorded provenance, and re-encrypt it with the distinct destination SQL
passphrase before allowing ciphertext to leave the helper. Reject
unrepresentable identity or custody before constructing a destination Store.

## Verification

- Spendable and watch-only fixtures preserve roots, every ordinary scope and
  account, account names, schemas, optional fingerprints, child paths,
  counters, and normalized next-account cursors.
- Imported-XPub fixtures become numberless accounts and raw imported scripts
  become accountless addresses; neither synthetic identity moves a numbered
  cursor.
- Every migrated account and raw import is `watch-chain`, with no inferred
  key-only state.
- Per-record public-key and script-key ciphertext fixtures decrypt and
  re-encrypt correctly. Using one key class for the whole wallet fails the
  focused vector.
- A spendable source requires its legacy private credential, and a missing or
  wrong value returns the typed credential refusal without destination access
  or retained plaintext.
- A watch-only source converts with no legacy private credential, rejects an
  invalid destination SQL passphrase under the Vault policy, and produces
  ciphertext protected by the independently supplied destination passphrase.
- Mixed custody, root-neutered custody, conflicting schema overrides,
  overlapping equal XPub payloads, and each script-collision class return their
  typed refusal before destination access.
- Cleanup probes cover success, each refusal, cancellation, Vault error, and
  injected panic. Every wipeable value is zero and no immutable parser string
  is retained.

## Definition of done

- A supported legacy source yields one bounded, ciphertext-only normalized
  account/address snapshot with truthful SQL identity and custody.
- Every unsupported account, script, schema, or custody shape is refused before
  destination access without changing the source.
- Migration plaintext is bounded and destroyed on every exit by this task,
  independently of Task 367.
- The PR contains no transaction extraction, SQL mutation, public converter,
  kvdb rotation, wallet splitting, or general Vault redesign.

[t332]: 332-watch-only-script-policy.md
[t335]: 335-kvdb-to-sql-migration.md
