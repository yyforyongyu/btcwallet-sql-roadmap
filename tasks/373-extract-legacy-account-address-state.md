# Task 373 — extract legacy wallet, account, and secret state

Component: **Legacy migration wallet, account, and secret extraction**.

Type: **Migration**.

Dependencies: **[Task 332][t332], [Task 335][t335], and
[Task 387][t387]**.

## Rationale

The modern kvdb adapter omits legacy identity and encryption facts needed for
an exact SQL import. Imported XPub identity and per-record root/account
secret provenance cannot be reconstructed safely from its public projection.

On 2026-09-19, address and raw-script decoding moved to [Task 473][t473]. This
task retains wallet/account identity and secret preparation as an independently
reviewable extraction stage.

## Details

- Read only Task 335's immutable snapshot and produce one bounded internal
  wallet/account snapshot.
- Preserve wallet identity, scopes, ordinary accounts, names, schemas, counters,
  imported XPub identity, and custody facts.
- Map true imported XPub accounts to numberless SQL accounts. Do not persist
  synthetic legacy identifiers; Task 473 owns address paths and raw imports.
- Preserve explicit scope/account-number identity for path-bound watch-only
  accounts. Until [Task 391][t391]'s runtime support is available, refuse that
  source profile rather than silently flattening it into a numberless import.
  Ordinary supported profiles do not wait for Task 391.
- Set `NoChainSync=false`; legacy kvdb cannot represent that independent
  policy.
- Reject identity collisions, unsupported custody, or another legacy shape
  that cannot be represented without changing meaning.
- Decrypt each root/account secret using its recorded legacy key class and
  prepare ciphertext for the independently supplied SQL passphrase. Require
  a private legacy credential only for spendable source material.
- Bound and destroy migration plaintext on success, error, cancellation, and
  panic. Return only ciphertext and normalized public facts.
- Keep implementation in Wallet-owned migration code and legacy packages
  unchanged.
- Exclude address/script decoding and their secrets, transaction and sync
  extraction, SQL writes, custody transformation, public APIs, and in-place
  kvdb passphrase rotation.

## Suggested fix

Use a migration-specific raw legacy reader and a narrow Wallet-owned secret
preparation boundary reusable by Task 473. Consume existing legacy readers
and Wallet cryptography without prescribing private snapshot/decoder types
or prebuilding a generic extraction framework.

## Verification

- Spendable, watch-only, and imported-XPub fixtures preserve every supported
  wallet/account identity, schema, counter, and custody fact.
- Root/account key-class fixtures decrypt and re-encrypt correctly with the
  destination credential.
- Unsupported custody, identity collisions, and malformed state fail before
  destination access.
- Path-bound watch-only fixtures retain their real account number when
  supported, or are explicitly refused without destination mutation.
- Cleanup probes demonstrate bounded plaintext ownership on every exit.

## Definition of done

A supported legacy source yields one ciphertext-only normalized wallet and
account snapshot without mutating the source or changing identity and custody
semantics. Address/script extraction remains independently owned by Task 473.

[t332]: ../archive/tasks/332-watch-only-script-policy.md
[t335]: 335-kvdb-to-sql-migration.md
[t387]: ../archive/tasks/387-account-creation-identity-adr.md
[t391]: ../archive/tasks/391-path-bound-watch-only-xpub-accounts.md
[t473]: 473-extract-legacy-address-script-state.md
