# Task 473 — extract legacy address and script state

Component: **Legacy migration address and script extraction**.

Type: **Migration**.

Dependencies: **[Task 373][t373]**.

## Rationale

Legacy address records contain derivation, usedness, raw-script, and secret
provenance facts that the modern kvdb public projection cannot fully expose
for an exact SQL import. This decoder boundary was split from Task 373 on
2026-09-19; it consumes that task's normalized wallet/account identities and
secret preparation rather than duplicating them.

## Details

- Read only [Task 335][t335]'s immutable source, joining derived addresses to
  Task 373's normalized account identities from that same snapshot.
- Preserve derived paths, raw scripts, usedness, ownership, and custody facts.
  Map raw imports to accountless SQL addresses without synthetic legacy IDs.
- Reject malformed records, contradictory ownership, unsupported custody, or
  unrepresentable address/script state rather than changing its meaning.
- Decrypt each address/script secret according to its recorded legacy key
  class and prepare ciphertext under the independently supplied destination
  passphrase. Require a private legacy credential only for spendable source
  material; reuse Task 373's narrow secret-preparation mechanism.
- Bound allocations and destroy plaintext on success, error, cancellation,
  and panic. Publish only a complete bounded snapshot of normalized public
  facts and ciphertext, never a partial result or plaintext.
- Keep implementation in Wallet-owned migration code and consume existing
  legacy readers and formats unchanged.
- Exclude wallet/account decoding, root/account-secret extraction, transaction
  and sync extraction, SQL writes, custody transformation, public APIs, and
  in-place kvdb passphrase rotation.

## Suggested fix

Extend migration-only reading with the address/script decoder, using the
already-normalized account identities and secret preparation. Private reader
and snapshot shapes remain flexible; add no generic extraction framework.

## Verification

- Derived, imported-XPub-derived, and raw-script fixtures retain every
  supported path, ownership, script, usedness, and custody fact.
- Address/script key-class fixtures decrypt and re-encrypt correctly for the
  destination credential without modifying the source.
- Malformed state, ownership conflicts, and unsupported custody fail before
  destination access and publish no partial snapshot.
- Cleanup probes prove bounded plaintext ownership on every exit, including
  cancellation and panic, reusing the account-stage cleanup mechanism.

## Definition of done

A supported immutable source yields a complete ciphertext-only normalized
address/script snapshot joined to Task 373's account identities, ready for
[Task 374][t374] and the atomic importer without source mutation or changed
custody semantics.

[t335]: 335-kvdb-to-sql-migration.md
[t373]: 373-extract-legacy-account-address-state.md
[t374]: 374-extract-legacy-transaction-sync-state.md
