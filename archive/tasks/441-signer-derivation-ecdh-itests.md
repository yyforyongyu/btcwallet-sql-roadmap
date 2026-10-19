# Task 441 — cover public derivation and ECDH contracts

Component: **Signer derivation integration tests**.

Type: **Test**.

Dependencies: **None**.

## Rationale

[Issue #1250][issue1250] includes `Signer.DerivePubKey` and `Signer.ECDH`, but
neither has an integration-test task using real kvdb, SQLite, and PostgreSQL
Wallets. These operations share public BIP32 path resolution and
key-availability boundaries, so one focused lane can prove their maintained
contract end to end.

## Details

- Exercise `DerivePubKey` and `ECDH` through a real Store-backed Wallet.
- Register one shared public contract for kvdb, SQLite, and PostgreSQL. Do not
  skip kvdb inside shared cases. Restrict backend-specific code to setup and
  teardown; require identical public assertions and stable error identities.
- Cover supported account paths, external and internal branches, invalid or
  unknown paths, and locked or watch-only states where defined.
- Construct `DerivePubKey` inputs with the exported selector and request from
  [Task 337][t337]. Exercise name and numbered selectors, every exported
  request field, and numberless imported-XPub derivation on SQLite and
  PostgreSQL.
- Verify ECDH against an independently generated peer key in both directions
  and reject nil or unsupported inputs without exposing secret material.
- Reopen the same Wallet and prove public derivation and the shared secret are
  stable for the same durable identity.
- Task 337 owns the SQL imported-XPub behavior asserted in a separate SQL
  case here. Modern kvdb lacks that behavior today; this is a tracked gap, not
  a permanent structural inability. [Task 361][t361] owns the kvdb acceptance
  fix and its focused public imported-XPub regression in the same PR. Keep
  the common derivation/ECDH cases three-backend and do not skip kvdb in them.
  Do not block this baseline on Task 361 or add Vault routing from Task 384.

## Suggested fix

Add one derivation-focused itest source using deterministic accounts, BIP32
paths, and an independent peer key. Compare only public keys and fixed-size
shared secrets, and never log or persist private material.

## Verification

- Run the focused Signer derivation cases on kvdb, SQLite, and PostgreSQL.
- Verify stable results after reopen and complete secret-safe teardown.
- Exercise every exported derivation-request field through the imported
  `wallet` package rather than a separate `package wallet_test` file.

## Definition of done

- Both methods have representative path, locked, watch-only,
  invalid-input, and durable-reopen coverage.
- ECDH results are independently verified without Store or private-key reads.
- Shared cases contain no backend skip branch.
- SQL imported-XPub behavior is covered here; Task 361 explicitly owns the
  remaining kvdb behavior and public regression.

[issue1250]: https://github.com/btcsuite/btcwallet/issues/1250
[t337]: 337-imported-xpub-derivation.md
[t361]: 361-kvdb-imported-xpub-derivation.md
