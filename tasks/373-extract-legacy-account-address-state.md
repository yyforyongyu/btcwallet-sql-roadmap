# Task 373 — extract legacy account and address state

Component: **Legacy migration account, address, and secret extraction**.

Type: **Migration**.

Dependencies: **[Task 332][t332], [Task 335][t335], and
[Task 387][t387]**.

## Rationale

The modern kvdb adapter omits legacy identity and encryption facts needed for
an exact SQL import. Imported XPub accounts, raw scripts, and per-record secret
provenance cannot be reconstructed safely from its public projection.

## Details

- Read only Task 335's immutable snapshot and produce one bounded internal
  account/address snapshot.
- Preserve scopes, ordinary accounts, names, schemas, counters, derived paths,
  imported XPub identity, raw scripts, usedness, and custody facts.
- Map true imported XPub accounts to numberless SQL accounts and raw imports to
  accountless addresses. Do not persist synthetic legacy identifiers.
- Set `NoChainSync=false`; legacy kvdb cannot represent that independent
  policy.
- Reject identity collisions, unsupported custody, or another legacy shape
  that cannot be represented without changing meaning.
- Decrypt each secret according to its recorded legacy key class and prepare
  ciphertext for the independently supplied SQL passphrase. Require a private
  legacy credential only for spendable source material.
- Bound and destroy migration plaintext on success, error, cancellation, and
  panic. Return only ciphertext and normalized public facts.
- Keep implementation in Wallet-owned migration code and legacy packages
  unchanged.
- Exclude transaction and sync extraction, SQL writes, custody transformation,
  public APIs, and in-place kvdb passphrase rotation.

## Suggested fix

Use a migration-specific raw legacy reader and a narrow Wallet-owned secret
preparation boundary. Private snapshot and decoder shapes are not prescribed.

## Verification

- Spendable, watch-only, imported-XPub, and raw-script fixtures preserve every
  supported identity, schema, counter, path, usedness, and custody fact.
- Per-record key-class fixtures decrypt and re-encrypt correctly with the
  destination credential.
- Unsupported custody, identity collisions, and malformed state fail before
  destination access.
- Cleanup probes demonstrate bounded plaintext ownership on every exit.

## Definition of done

A supported legacy source yields one ciphertext-only normalized account and
address snapshot without mutating the source or changing its identity and
custody semantics.

[t332]: ../archive/tasks/332-watch-only-script-policy.md
[t335]: 335-kvdb-to-sql-migration.md
[t387]: ../archive/tasks/387-account-creation-identity-adr.md
